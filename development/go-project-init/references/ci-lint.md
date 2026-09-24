# lint 与 CI：golangci-lint + GitHub Actions Test

> 加载条件：为 Go 项目配置 golangci-lint，或搭建/调整 GitHub Actions 测试 CI 时阅读。
> release 流程不在本文，见 `release.md`。

## golangci-lint

### golangci-lint 原则

- 面向个人/小团队项目，默认集保持宽松，低误报优先；误报成本会侵蚀遵守意愿
- 其余 linter 按项目实际需要逐个补充

### golangci-lint 配置

```yaml
# .golangci.yml
version: "2"

linters:
  enable:
    - misspell     # 英文拼写
    - unconvert    # 冗余类型转换

formatters:
  enable:
    - gofmt
    - goimports
```

- v2 默认启用 `errcheck`、`govet`、`ineffassign`、`staticcheck`、`unused`，本身是低误报高价值的基线；`enable` 在默认集之上**增补**（除非显式设置 `linters.default: none`）
- 显式增补的四个均为低成本选择：`unconvert` 几乎零误报；`misspell` 偶发误报生僻词，可按需配置排除；`gofmt`/`goimports` 统一格式，消除 diff 噪音
- `timeout`、`issues` 等运行参数不配置，使用默认值
- 可选增补（按需）：`gosec`（安全扫描，纯工具类项目误报偏多）、`revive`（风格检查，规则多且部分有争议）

### 集成方式

- **CI**：lint 作为 job 并入 Test Workflow（配置见下文 GitHub Actions 一节），与 test job 并行，不单独建 Workflow 文件
- **版本**：CI 中固定小版本（如 `v2.13`，示例会过期，取 [golangci-lint releases](https://github.com/golangci/golangci-lint/releases) 当前最新小版本），升级由人主动进行，不用 latest
- **版本耦合硬约束**：golangci-lint 的**构建 Go 版本必须 ≥ `go.mod` 的目标 Go 版本**，否则 lint job 直接失败（旧版工具链无法解析新版语言特性与标准库 API）。升级 Go 工具链（`go.mod` 的 `go` 指令）时必须同步检查并升级 CI 固定的 lint 版本，两者作为一次变更提交
- **本地**：Makefile 提供 `lint` 目标：

```makefile
lint:
	golangci-lint run
```

## GitHub Actions

CI 通用约定与 Test Workflow；release 统一由 goreleaser 负责（见 `release.md`）。

### 通用约定

- Go 版本：通过 `go-version-file: go.mod` 从 `go.mod` 读取，不在 Workflow 中硬编码
- 依赖缓存：使用 `actions/setup-go` 内置缓存（默认启用），不手动配置 `actions/cache`
- 触发方式：允许 `workflow_dispatch` 手动触发；release 默认由 tag `v*.*.*`（语义化版本）触发
- 工具版本固定策略因工具而异：golangci-lint 固定**小版本**（其小版本可能新增检查项导致既有代码 CI 变红，升级应受控）；goreleaser 固定**大版本** `~> v2`（行为已由配置中的 `version: 2` schema 锁定，小版本升级风险低）

### Test Workflow

```yaml
# .github/workflows/test.yml
name: Test

on:
  push:
    branches: [main, master, develop, dev]
  pull_request:
  workflow_dispatch:

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7

      - uses: actions/setup-go@v7
        with:
          go-version-file: go.mod

      - uses: golangci/golangci-lint-action@v9
        with:
          version: v2.13

  test:
    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]
    runs-on: ${{ matrix.os }}
    env:
      CGO_ENABLED: 0
    steps:
      - uses: actions/checkout@v7

      - uses: actions/setup-go@v7
        with:
          go-version-file: go.mod

      - run: go vet ./...

      - run: go test ./...
```

- push 触发显式列出主干分支（main / master / develop / dev）而非放开所有分支：覆盖默认分支仍为 master 的存量仓库、以及直接 push develop/dev 主干的工作流，同时避免与 `pull_request` 重复触发导致同一次提交双跑；feature 分支的验证统一走 PR
- 平台矩阵默认三系统（ubuntu / windows / macos），**由具体项目按需裁剪**——例如只支持特定系统的工具可只保留对应平台
- 存在 `web/` 目录（embed 场景）时，lint 与 test 两个 job 在编译前都需创建 `web/dist` 占位文件（golangci-lint 同样会做类型检查），步骤见 `web-embed.md`
- test job 固定 `CGO_ENABLED=0`：**测你所发**——release 产物为纯 Go 构建（见 `release.md`），CI 即验证同一配置；矩阵行为因此一致（否则各 runner 是否预装 C 工具链会决定走哪个 SQLite 驱动，行为差异表象为平台问题、根因是驱动不同），同时省去 mattn 的 C 编译耗时与 gcc 依赖。CGO 驱动路径由本地开发自然覆盖；启用 CGO release 变体的项目再按需增加专门的 CGO 测试 job（驱动取舍见 `cgo-sqlite.md`）
- `go vet` 先于测试执行，静态问题快速失败；它与 lint job 中的 `govet` 存在重叠，是有意保留：`go vet` 在矩阵各平台上运行，可覆盖平台条件编译的代码路径，lint 只在 ubuntu 运行一次
- 数据竞争敏感的项目可加 `-race`：`-race` 依赖 CGO，需将该 job 改回 `CGO_ENABLED=1` 并确保 runner 具备 C 工具链（windows 需 gcc）
