# SQLite 驱动与 CGO

> 加载条件：项目使用 SQLite 需选择驱动，或需评估 CGO 对交叉编译、静态二进制、容器镜像的影响时阅读。
> CI 与 release 中 `CGO_ENABLED=0` 的具体配置见 `ci-lint.md`、`release.md`。

## 两条驱动路线

| 驱动                          | 类型  | `database/sql` 驱动名 | 特点                                                                   |
| ----------------------------- | ----- | --------------------- | ---------------------------------------------------------------------- |
| `github.com/mattn/go-sqlite3` | CGO   | `sqlite3`             | 直接绑定官方 C 源码，性能最好；编译需 C 工具链，无法单 runner 交叉编译 |
| `modernc.org/sqlite`          | 纯 Go | `sqlite`              | C 源码自动转译，`CGO_ENABLED=0` 可编译；二进制更大、性能略低           |

## 默认策略：纯 Go 驱动

- **开发、测试、发布一律默认纯 Go 驱动**：保持全平台交叉编译能力与静态二进制可移植性，CI 维持单 runner 基线
- CGO 驱动的性能优化路线保留为可选项，见下文

## 自动切换机制

利用官方内置的 `cgo` build constraint（`go help buildconstraint`，CGO 启用时自动满足），业务代码对驱动无感知：

```go
// driver_cgo.go
//go:build cgo

package store

import _ "github.com/mattn/go-sqlite3"

const driverName = "sqlite3"
```

```go
// driver_nocgo.go
//go:build !cgo

package store

import _ "modernc.org/sqlite"

const driverName = "sqlite"
```

- 本机有 C 工具链时开发构建自动使用 CGO 驱动；CI 与发布构建固定 `CGO_ENABLED=0`（见 `ci-lint.md`、`release.md`），发布产物为纯 Go 版本
- 打开数据库统一使用 `sql.Open(driverName, dsn)`

## DSN 推荐写法

两驱动的私有 DSN 参数不同（mattn 的 `_journal_mode=` 等 vs modernc 的参数风格），公共交集是 `_pragma=` 形式（mattn/go-sqlite3 自 v1.14 起支持），统一使用：

```txt
file:app.db?_pragma=journal_mode(WAL)&_pragma=busy_timeout(5000)&_pragma=foreign_keys(1)
```

## 切换驱动后的注意事项

- 两驱动对 SQLite 扩展功能（如部分 FTS、虚拟表）支持存在差异，切换后需回归测试
- CGO release 变体的代价（仅当确有重 DB 负载时启用）：
  - 无法单 runner 交叉编译：darwin 产物需 macos runner 原生构建，windows 需 mingw-w64，release 需拆分为各平台原生构建再合并产物（goreleaser split/merge 流程）
  - linux 产物动态链接 glibc，失去静态二进制可移植性（Alpine/musl、`scratch` 镜像不可用；基础镜像选择见 `release.md`「Dockerfile」）
  - 本文只给原则，完整配置按具体项目落地
