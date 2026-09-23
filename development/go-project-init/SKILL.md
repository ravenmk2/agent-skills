---
name: go-project-init
description: Use when creating, initializing, or scaffolding a Go project, or when adding engineering tooling to an existing Go project.
---

# Go 项目创建与工程化最佳实践

通用约定，覆盖三种项目形态：**库（Library）**、**命令行工具（CLI）**、**服务端应用（Server）**。面向个人/小团队项目：低维护成本、低误报、克制。

## 工作流

创建新项目时按序执行；为存量项目补配置时直接查文末「按需参考」。

1. **确认形态与模块路径**：库 / CLI / Server 三选一；模块路径与仓库地址一致（如 `module github.com/user/project`）——`go install`、pkg.go.dev 文档页、徽章示例中的路径均以此为准
2. **搭建目录骨架**：按本文「目录结构」，用「按项目形态取舍」表裁剪
3. **创建仓库基础文件**：`.gitignore`、`.gitattributes`、`AGENTS.md`，见本文「仓库基础文件」
4. **可选特性**：embed 前端构建产物 → 读 `references/web-embed.md`；使用 SQLite → 读 `references/cgo-sqlite.md`
5. **lint 与 CI**：读 `references/ci-lint.md`
6. **发布**：goreleaser、容器镜像、本地构建脚本 → 读 `references/release.md`（库项目通常跳过）
7. **README**：按本文「README 徽章」写最小 README

## 目录结构

### 原则

- 遵循社区约定，参考 [golang-standards/project-layout](https://github.com/golang-standards/project-layout)，仅采用被广泛实践的部分（该仓库非权威标准）
- 平铺优先，目录随规模演化，不提前引入用不到的分层

### 通用结构

```txt
project/
├── cmd/
│   └── <appname>/
│       └── main.go        # 程序入口，只做组装与启动，不含业务逻辑
├── internal/              # 应用私有代码，禁止外部模块导入（编译器强制）
├── pkg/                   # 允许外部模块导入的公共代码
├── web/                   # 前端源码（embed 场景，服务端应用可选）
├── scripts/
│   └── build.sh           # 本地多平台构建脚本
├── .github/workflows/     # GitHub Actions
├── .gitignore
├── .gitattributes         # Git 文件属性（行尾、二进制、diff 等）
├── .ignore                # ripgrep 等检索工具的忽略规则
├── .golangci.yml          # lint 配置
├── .goreleaser.yml        # release 配置
├── Dockerfile             # 容器镜像构建（服务端形态发布镜像时需要）
├── AGENTS.md              # 面向 AI 编码助手的项目说明
├── go.mod
├── go.sum
├── LICENSE                # 许可证（GitHub 据此探测，README 徽章依赖）
├── Makefile               # 可选的便捷封装，非必需入口
└── README.md
```

### 关键约定

- **`cmd/`**：统一使用，即使只有一个可执行文件。每个子目录对应一个可执行程序，目录名即二进制名；支持 `go install github.com/user/project/cmd/<appname>@latest`
- **`internal/`**：应用代码默认放这里，明确表达"非公共 API"；Go 编译器禁止其他模块导入 `internal/` 下的包
- **`pkg/`**：确实需要对外暴露的公共 API 才放这里；库项目的对外代码可直接放根目录或 `pkg/`
- **`internal/`、`pkg/` 内部的分层方式不做规约**，由具体项目按业务域决定
- **模块路径与仓库地址一致**（如 `module github.com/user/project`）：`go install`、pkg.go.dev 文档页、徽章示例中的路径均以此为准

### 按项目形态取舍

| 目录         | 库                   | CLI  | 服务端应用         |
| ------------ | -------------------- | ---- | ------------------ |
| `cmd/`       | 不需要               | 需要 | 需要               |
| `internal/`  | 可选（隐藏实现细节） | 需要 | 需要               |
| `pkg/`       | 对外 API 主体        | 可选 | 可选               |
| `web/`       | —                    | —    | embed 前端时使用   |
| `Dockerfile` | —                    | —    | 发布容器镜像时需要 |

库项目的典型形态是对外代码直接在根目录（如 `github.com/user/project` 即包本身），或以命名子包组织（如标准库风格）。

### 避免的目录命名

- `src/`、`lib/`：Go 无此概念
- `util/`、`common/`、`helper/`：垃圾桶式目录，应按业务域命名

## 仓库基础文件

### 默认 .gitignore

```gitignore
.vscode/
.idea/
.agents/

/dist/
/web/dist/
```

- `.vscode/`、`.idea/`、`.agents/`：编辑器与 AI 工具的本地配置，不入库
- `/dist/`：`scripts/build.sh` 的本地构建输出目录
- `/web/dist/`：前端构建产物整体忽略，包括 embed 占位文件（不纳入版本管理的原因见 `references/web-embed.md`）

### 默认 .gitattributes

```gitattributes
* text=auto

*.go  text eol=lf
*.sh  text eol=lf
*.ps1 text eol=lf

*.bat text eol=crlf
*.cmd text eol=crlf
```

- `* text=auto`：git 自动判断文本文件，仓库内统一按 LF 存储、按平台习惯检出，消除各开发者 `core.autocrlf` 设置差异带来的行尾噪音
- `*.go` / `*.sh` 强制 LF 存储与检出：`.sh` 一旦被 CRLF 污染，在 Git Bash / WSL / Linux 下执行会报 `\r` 相关错误；Go 源码 `gofmt` 输出即 LF
- `*.bat` / `*.cmd` 强制 CRLF：cmd.exe 解析器对 LF-only 文件有实际兼容问题，`goto`/标签跳转、括号代码块会失效或行为异常
- `*.ps1` 指定 LF：PowerShell 解析器（5.1 与 7+）对 LF/CRLF 均兼容，行尾不影响执行，统一 LF 与仓库整体约定一致；需要注意的是**编码**而非行尾——Windows PowerShell 5.1 将无 BOM 文件按 ANSI 解读，`.ps1` 含非 ASCII 字符（如中文注释）时应保存为 UTF-8 with BOM
- 对行尾敏感的 shell 脚本依赖此约定才能跨平台可靠执行（配合 `references/release.md` 的本地构建脚本）

### AGENTS.md 最小示例

面向 AI 编码助手的项目说明，保持最小：标题固定、一句话项目说明、目录结构、文档索引。

```markdown
# AI Agents 工作规范

<一句话项目说明：这个项目是什么、做什么>

## 目录结构

project/
├── cmd/<appname>/   # 程序入口
├── internal/        # 应用私有代码
├── web/             # 前端源码与构建产物
└── scripts/         # 辅助脚本

## 文档索引

文档变化时同步更新

- <其他文档路径>：<一句话说明>
```

## README 徽章

### 徽章原则

- 克制：徽章传递项目活跃度与质量信号，过多反而稀释信息
- LICENSE 文件放仓库根目录（GitHub 约定位置，License 徽章依赖它探测）
- 风格不强制：知名项目（[cobra](https://github.com/spf13/cobra)、[gin](https://github.com/gin-gonic/gin) 等）普遍使用 shields.io 默认 flat 样式、不统一样式参数；保持默认即可

### 最小 README 示例

结构固定为：主标题 → 徽章 → 简介。

```markdown
# project

[![Test](https://github.com/user/project/actions/workflows/test.yml/badge.svg)](https://github.com/user/project/actions/workflows/test.yml)
[![Release](https://img.shields.io/github/v/release/user/project)](https://github.com/user/project/releases)
[![License](https://img.shields.io/github/license/user/project)](LICENSE)

一句话简介：这个项目是什么、解决什么问题。
```

### 徽章集与排序

排序按读者关心度从左到右：健康状态 → 版本 → 使用要求 → 法律信息。

**通用（三种形态）**：Test → Release → License

**库项目追加**：pkg.go.dev 排最前（API 文档是库的第一入口），Go Version 排 License 之前

```txt
Library:    pkg.go.dev → Test → Release → Go Version → License
CLI/Server: Test → Release → License
```

- Go Version 徽章内容取自 `go.mod` 的 `go` 指令，向库使用者传达最低版本要求，仅库项目纳入；CLI/服务端用户消费二进制，不感知 Go 版本

### 徽章示例

```markdown
<!-- 通用 -->
[![Test](https://github.com/user/project/actions/workflows/test.yml/badge.svg)](https://github.com/user/project/actions/workflows/test.yml)
[![Release](https://img.shields.io/github/v/release/user/project)](https://github.com/user/project/releases)
[![License](https://img.shields.io/github/license/user/project)](LICENSE)

<!-- 库项目追加 -->
[![Go Reference](https://pkg.go.dev/badge/github.com/user/project.svg)](https://pkg.go.dev/github.com/user/project)
[![Go Version](https://img.shields.io/github/go-mod/go-version/user/project)](go.mod)
```

### 不纳入

- **Go Report Card**：已于 2026 年 7 月停服（[官方公告](https://github.com/gojp/goreportcard)），徽章链接已失效，存量项目应清理——这也是徽章腐化的典型案例，选择徽章时应优先官方/一手来源
- 覆盖率（Codecov 等，依赖外部服务，小项目噪音大）、下载量、star 数、装饰性徽章

## 跨主题硬约定

- **CGO 基线**：开发、测试、发布一律 `CGO_ENABLED=0` 纯 Go 构建，保持全平台交叉编译能力与静态二进制可移植性；CGO 是显式 opt-in（SQLite 场景见 `references/cgo-sqlite.md`）
- **版本注入**：统一 `-ldflags "-s -w -X main.version=<ver>"`，代码中 `var version = "dev"` 兜底；release 取 git tag 去 `v` 前缀，本地构建用 `git describe --tags --always --dirty`
- **工具版本固定**：golangci-lint 固定小版本（如 `v2.5`）、goreleaser 固定大版本（`~> v2`），不用 latest
- **CI Go 版本**：用 `go-version-file: go.mod` 从 `go.mod` 读取，不在 workflow 中硬编码
- **release 统一走 goreleaser**：tag `v*.*.*` 触发；changelog 从 Conventional Commits 生成，不在仓库内维护 CHANGELOG.md

## 按需参考

| 文件                        | 加载条件                                                       |
| --------------------------- | -------------------------------------------------------------- |
| `references/web-embed.md`   | 服务端应用要将前端构建产物 embed 进二进制、单文件分发时        |
| `references/cgo-sqlite.md`  | 项目使用 SQLite，需选择驱动或处理 CGO 与交叉编译问题时         |
| `references/ci-lint.md`     | 配置 golangci-lint 或 GitHub Actions 测试 CI 时                |
| `references/release.md`     | 配置 goreleaser 发布、GHCR 容器镜像或本地多平台构建脚本时      |
