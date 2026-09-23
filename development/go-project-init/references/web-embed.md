# Web 前端代码嵌入（embed）

> 加载条件：服务端应用需将前端构建产物嵌入 Go 二进制、实现单文件分发时阅读。
> 涉及 CI 占位文件步骤与 release 前端构建时，配合 `ci-lint.md`、`release.md` 阅读。

## 前端代码嵌入原则

- 使用标准库 `embed` 包（Go 1.16+），**统一嵌入，不做开发/生产构建标签分支**；开发调试由前端工具自身的 dev server + 后端代理解决
- 前端源码与构建产物都在 `web/` 下，构建输出至 `web/dist/`，直接嵌入，不复制、不移动产物
- 不绑定具体前端构建工具，只约定目录结构与路由行为

## 目录约定

```txt
web/
├── embed.go     # embed 声明所在包
├── dist/        # 前端构建产物（构建工具输出目录）
│   └── ...
└── src/         # 前端源码
```

- `//go:embed` 只能嵌入当前包目录及子目录，因此 embed 声明必须位于 `web/` 下的 Go 文件中
- `embed` 要求被嵌入目录在编译时非空，但**占位文件不提交到仓库**：前端构建工具默认清空并重写输出目录，任何提交在 `dist/` 内的文件构建后都会被覆盖或删除，导致工作区污染——`git status` 挂无关变更、`git describe --dirty` 让本地构建版本号带上 `-dirty` 后缀、并可能误提交构建产物。因此 `web/dist/` 整体忽略，占位文件由构建入口按需创建：
  - `scripts/build.sh` 内置创建逻辑（见 `release.md`「本地构建脚本」）
  - CI 中编译 Go 代码（`go vet` / `go test`）前创建一次，注意用 `shell: bash` 保证三系统 runner 一致：

    ```yaml
    - name: Ensure web/dist placeholder
      shell: bash
      run: mkdir -p web/dist && touch web/dist/index.html
    ```

  - 本地直接编译或检查（`go build` / `go vet` / `golangci-lint run`）前手动执行一次 `mkdir -p web/dist && touch web/dist/index.html`

## embed 声明

```go
// web/embed.go
package web

import (
	"embed"
	"io/fs"
)

//go:embed all:dist
var distFS embed.FS

// DistFS 返回前端构建产物，路径前缀 dist/ 已剥离。
func DistFS() fs.FS {
	sub, err := fs.Sub(distFS, "dist")
	if err != nil {
		panic(err)
	}
	return sub
}
```

- `all:` 前缀：同时嵌入以 `_`、`.` 开头的文件（部分构建工具的产物或缓存文件默认会被 `embed` 排除）
- 通过 `fs.Sub` 剥离 `dist/` 前缀，服务端代码拿到的是产物根目录视图

## 路由约定：SPA 回退

前端使用 history 模式路由时，非静态资源路径必须回退到 `index.html`，由前端路由接管：

```go
func SPAHandler(dist fs.FS) http.Handler {
	fileServer := http.FileServerFS(dist)
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		p := strings.TrimPrefix(r.URL.Path, "/")
		if f, err := dist.Open(p); err == nil {
			st, statErr := f.Stat()
			f.Close()
			if statErr == nil && !st.IsDir() {
				fileServer.ServeHTTP(w, r)
				return
			}
		}
		// 路径不存在或为目录：回退 index.html，交由前端路由
		r.URL.Path = "/"
		fileServer.ServeHTTP(w, r)
	})
}
```

- 对目录路径也要回退：直接放行会让 FileServer 暴露嵌入目录的列表页
- API 路由应先于 SPA handler 注册，避免被回退逻辑吞掉
- 使用 `http.FileServerFS`（Go 1.22+）直接从 `fs.FS` 提供静态文件
