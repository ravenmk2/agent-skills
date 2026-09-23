# 发布：goreleaser + 容器镜像 + 本地构建脚本

> 加载条件：为 Go 项目配置 release 流程（多平台二进制、changelog、GHCR 容器镜像）或本地多平台构建脚本时阅读。
> 测试 CI 不在本文，见 `ci-lint.md`。

release 构建统一使用 [goreleaser](https://goreleaser.com/)：多平台多架构交叉编译、归档、checksums、changelog 开箱即用，声明式配置维护成本低。

## goreleaser 配置

```yaml
# .goreleaser.yml
version: 2

builds:
  - id: <appname>
    main: ./cmd/<appname>
    binary: <appname>
    goos: [linux, darwin, windows]
    goarch: [amd64, arm64]
    env: [CGO_ENABLED=0]
    ldflags:
      - -s -w -X main.version={{.Version}}

archives:
  - formats: [tar.gz]
    format_overrides:
      - goos: windows
        formats: [zip]
    name_template: >-
      {{ .ProjectName }}_{{ .Version }}_{{ .Os }}_{{ .Arch }}

checksum:
  name_template: checksums.txt

changelog:
  sort: asc
  filters:
    exclude:
      - "^docs:"
      - "^test:"
      - "^chore:"
      - "^ci:"
      - "^style:"
      - "^build:"
      - "Merge pull request"
  groups:
    - title: Features
      regexp: "^feat(\\(.+\\))?:.*$"
      order: 0
    - title: Bug Fixes
      regexp: "^fix(\\(.+\\))?:.*$"
      order: 1
```

- `CGO_ENABLED=0`：纯 Go 交叉编译基线，单 runner 即可产出全平台产物；涉及 SQLite 时的驱动取舍见 `cgo-sqlite.md`
- 压缩格式约定：windows 用 `.zip`，其余平台用 `.tar.gz`，由 `format_overrides` 实现
- 产物命名：`<name>_<version>_<os>_<arch>`，version 不含 `v` 前缀
- changelog 基于 Conventional Commits 解析 commit（个人开发通常不走 PR 流程，commit 即信息源）；只分 Features / Bug Fixes 两组，未匹配的 commit 落入 Others；`docs`/`test`/`chore`/`ci`/`style`/`build` 及合并提交一律排除；不生成仓库内的 `CHANGELOG.md` 文件

## 版本注入

- release 由 tag 触发时，goreleaser 取 git tag 去掉 `v` 前缀作为 `{{.Version}}`（tag `v1.2.3` → `1.2.3`），经 `-X main.version={{.Version}}` 在链接期写入：

```go
package main

var version = "dev" // 未注入时的默认值
```

- 本地开发构建默认得到 `dev`；`scripts/build.sh` 内置相同的注入机制（见下文「本地构建脚本」），其版本由 git 命令生成：

```bash
git describe --tags --always --dirty
```

- tag 后无新提交输出 `v1.2.3`；有新提交输出 `v1.2.3-5-gabc1234`；工作区脏追加 `-dirty`
- 兜底手段（零 ldflags）：Go 1.18+ 可用 `runtime/debug.ReadBuildInfo()` 读取二进制内嵌的 VCS 信息（`vcs.revision`、`vcs.time`、`vcs.modified`）

## Release Workflow

```yaml
# .github/workflows/release.yml
name: Release

on:
  push:
    tags: ["v*.*.*"]
  workflow_dispatch:

permissions:
  contents: write

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - uses: actions/setup-go@v5
        with:
          go-version-file: go.mod

      - uses: goreleaser/goreleaser-action@v6
        with:
          distribution: goreleaser
          version: "~> v2"
          args: release --clean
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

- 触发：tag `v*.*.*`（语义化版本）为主，保留 `workflow_dispatch` 手动触发
- `fetch-depth: 0`：goreleaser 需要完整 git 历史计算 changelog 与版本
- embed 场景下，goreleaser 构建前需先完成前端构建（`web/dist` 为真实产物，见 `web-embed.md`）；此时 Release Workflow 中需增加前端工具链（如 Node.js）的安装与构建步骤，按具体项目补充
- 服务端形态需发布容器镜像时，在此基础上追加 GHCR 登录与 buildx 准备步骤，见下文「容器镜像发布」

## 容器镜像发布（服务端形态）

服务端应用通常还需发布容器镜像至 GHCR（GitHub Packages 容器仓库，`ghcr.io`）。沿用 goreleaser 统一构建：`dockers` 将已交叉编译好的各架构二进制打包为镜像，`docker_manifests` 合成多架构 manifest 推送。**镜像内二进制与 release 归档产物同源同构建**，不引入第二套构建流程；也不建议在独立 job 中用多阶段 Dockerfile 重新编译（版本注入需重复维护，arm64 走 QEMU 模拟编译慢一个数量级）。

### .goreleaser.yml 追加配置

```yaml
dockers:
  - id: <appname>-amd64
    goarch: amd64
    use: buildx
    dockerfile: Dockerfile
    image_templates:
      - ghcr.io/<owner>/<project>:{{ .Version }}-amd64
    build_flag_templates:
      - --platform=linux/amd64
      - --label=org.opencontainers.image.source=https://github.com/<owner>/<project>
  - id: <appname>-arm64
    goarch: arm64
    use: buildx
    dockerfile: Dockerfile
    image_templates:
      - ghcr.io/<owner>/<project>:{{ .Version }}-arm64
    build_flag_templates:
      - --platform=linux/arm64
      - --label=org.opencontainers.image.source=https://github.com/<owner>/<project>

docker_manifests:
  - name_template: ghcr.io/<owner>/<project>:{{ .Version }}
    image_templates:
      - ghcr.io/<owner>/<project>:{{ .Version }}-amd64
      - ghcr.io/<owner>/<project>:{{ .Version }}-arm64
  - name_template: ghcr.io/<owner>/<project>:latest
    image_templates:
      - ghcr.io/<owner>/<project>:{{ .Version }}-amd64
      - ghcr.io/<owner>/<project>:{{ .Version }}-arm64
```

- `goarch` 按架构过滤 `builds` 产物，goreleaser 将对应二进制放入镜像构建上下文根目录，Dockerfile 直接 `COPY` 即可
- 镜像 tag 默认集：`{{ .Version }}`（如 `1.2.3`，与归档命名一致、不含 `v` 前缀）+ `latest`；`-amd64` / `-arm64` 单架构 tag 是合成 manifest 的中间产物，也可用于定点拉取
- `org.opencontainers.image.source` label：将 GHCR 包与源代码仓库关联，包页面展示 README 并继承仓库可见性
- GHCR 镜像名必须全小写：owner 或仓库名含大写字母时需手动转为小写，goreleaser 不会自动处理
- 前提：`builds` 的 `goos`/`goarch` 已覆盖 linux/amd64 与 linux/arm64（默认配置即满足）

### Dockerfile

```dockerfile
FROM alpine:3

RUN apk add --no-cache ca-certificates

COPY <appname> /usr/local/bin/<appname>

ENTRYPOINT ["<appname>"]
```

- 基础镜像默认 `alpine` 或 `gcr.io/distroless/static`：发布产物是 `CGO_ENABLED=0` 静态二进制，无运行时依赖；需要 shell 便于调试排障时可换 `debian:bookworm-slim` 等小体积镜像。不推荐 `scratch` 作默认——无 CA 证书，HTTPS 出站（调外部 API、webhook）直接失败
- `ca-certificates`：HTTPS 出站所必需；distroless/static 已内置，选用时可省去 `RUN apk add`
- `COPY` 源路径相对于 goreleaser 构建上下文根（二进制所在目录），不是仓库根
- 若项目启用了 CGO release 变体（见 `cgo-sqlite.md`），linux 产物动态链接 glibc，`alpine`（musl）与 distroless/static 均无法运行，基础镜像必须换 `debian:bookworm-slim` 等 glibc 系镜像

### Release Workflow 追加步骤

```yaml
permissions:
  contents: write
  packages: write   # 推送 GHCR
```

在 release job 的 steps 中、goreleaser 步骤之前追加：

```yaml
      - uses: docker/setup-qemu-action@v3       # 注册 binfmt，支持 arm64 镜像构建
      - uses: docker/setup-buildx-action@v3     # dockers 的 use: buildx 的前提

      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
```

- GHCR 认证使用 `GITHUB_TOKEN` 配合 `packages: write` 权限，无需创建 PAT
- QEMU 仅注册 binfmt：Go 交叉编译在宿主机完成，镜像构建只做 `COPY`；alpine 的 `apk add` 等少量 RUN 在 arm64 下走模拟执行，开销可忽略。选用 distroless/static（Dockerfile 无 RUN）则完全无模拟开销

## 配置验证

```bash
goreleaser check                   # 校验配置文件
goreleaser release --snapshot --clean   # 本地快照构建，不推送，验证产物完整性
```

- 快照模式跳过 `dockers` / `docker_manifests` 段（不构建、不推送镜像），镜像构建只能在真实 release 流程中验证

## 本地构建脚本

`scripts/build.sh`：本地多平台构建到 `dist/`，版本注入机制与 goreleaser 一致；面向本地验证与自用安装，正式发布仍走 goreleaser（压缩、checksums、changelog）。

```bash
#!/usr/bin/env bash
# scripts/build.sh — 本地多平台构建
set -euo pipefail

APP=<appname>
DIST=dist
VERSION=$(git describe --tags --always --dirty 2>/dev/null || echo dev)
LDFLAGS="-s -w -X main.version=${VERSION}"

platforms=(
  linux/amd64 linux/arm64
  darwin/amd64 darwin/arm64
  windows/amd64 windows/arm64
)

# embed 场景：确保 web/dist 存在且非空（占位文件不纳入版本管理，见
# web-embed.md）；若前端已构建则跳过，不覆盖真实产物
if [[ -d web && ! -e web/dist/index.html ]]; then
  mkdir -p web/dist
  printf '<!-- placeholder: run frontend build to replace -->\n' > web/dist/index.html
fi

# --install：仅构建当前平台并安装到 ~/.local/bin（CLI 类项目）
if [[ "${1:-}" == "--install" ]]; then
  os=$(go env GOOS)
  bin=$APP
  [[ $os == windows ]] && bin=${APP}.exe
  mkdir -p "${HOME}/.local/bin"
  CGO_ENABLED=0 go build -ldflags "$LDFLAGS" -o "${HOME}/.local/bin/${bin}" "./cmd/${APP}"
  echo "installed ${bin} to ~/.local/bin"
  exit 0
fi

mkdir -p "$DIST"
for p in "${platforms[@]}"; do
  os=${p%/*}
  arch=${p#*/}
  out="${DIST}/${APP}_${os}_${arch}"
  [[ $os == windows ]] && out="${out}.exe"
  echo "building ${os}/${arch} -> ${out}"
  CGO_ENABLED=0 GOOS=$os GOARCH=$arch go build -ldflags "$LDFLAGS" -o "$out" "./cmd/${APP}"
done
```

- `--install` 面向 CLI 类项目：只构建当前平台并安装到 `~/.local/bin/`（需自行确保该目录在 `PATH` 中）
- 跨平台执行：Windows 无内置 `make`，且 Unix 命令依赖类 Unix 环境——本地构建统一入口为 `scripts/build.sh`，Windows 下经 Git Bash 执行；Makefile 仅作可选的便捷封装（如 `lint` 目标，见 `ci-lint.md`），不作为必需入口
