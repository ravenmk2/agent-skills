---
name: create-skills-with-scripts
description: MUST use BEFORE any task that creates, edits, refactors, debugs, reviews, or deletes executable scripts bundled in an agent skill, or wraps one-off uvx/npx commands into a skill.
---

# Create Skills with Scripts

为「带可执行脚本/命令的技能」编写脚本时遵循本规范；通用技能解剖（frontmatter、触发词、progressive disclosure、迭代流程）遵循 skill-creator，本文不重复。

**脚本的第一用户是 Agent，不是人**：Agent 读 `--help` 了解用法、解析 stdout 获取结果、依据 stderr 与退出状态决策、失败后重试、受 token 预算约束。本规范所有细则都是这一事实的推论。

## 何时配脚本

脚本的价值不是省去重复输入，而是把「已验证的行为」固化为可调用的工具，替代 Agent 每次现场生成代码。三类情形应固化：

| 情形 | 说明与示例 |
|---|---|
| 确定性、重复出现的操作 | 格式化、批量重命名、索引构建——写一次，消除每次的变异 |
| 依赖专业库或格式知识的复杂操作 | 生成 PPTX、解析 DOCX、数据分析等——现场生成的代码容易在格式与 API 细节处出错，测试过的脚本把正确性一次锁定 |
| 多步骤、难以一次调对的命令序列 | 用几个 flag 就能调对的一次性命令（`uvx`/`npx`）直接写进 SKILL.md；再复杂就固化 |

需要判断、一次性、随上下文变化的操作不固化，直接在 SKILL.md 写指令。

## 细则

### 可发现 Discoverable

在 SKILL.md 设 Scripts 小节，用表格列出每个脚本的命令、用途、示例，路径以技能根目录为基准：

```markdown
## Scripts

| 命令 | 用途 | 示例 |
|---|---|---|
| `scripts/run.sh` | 构建索引（含运行时预检） | `bash scripts/run.sh ./src` |
| `scripts/query.py` | 查询索引 | `uv run scripts/query.py "keyword" --format json` |
```

`--help` 是 Agent 了解脚本的第一入口（失败后也会先跑它），必须覆盖用法、参数与可直接运行的示例，同时保持精简——它也占用 Agent 的上下文窗口：

```python
parser = argparse.ArgumentParser(
    description="Build the search index.",
    epilog="examples:\n  uv run scripts/build.py ./src --format json",
    formatter_class=argparse.RawDescriptionHelpFormatter,
)
```

外部依赖（如 uv、Node.js 22+）在 SKILL.md 中显式声明，与脚本运行时的预检（见「可诊断」）形成双保险。

### 可复现 Reproducible

一次性命令用 `uvx` / `npx` 运行，并锁定精确版本，不用 `@latest`：

```bash
uvx ruff@0.8.0 check .
uvx black@24.10.0 .
npx prettier@3.3.3 --check .
```

Python 脚本携带 PEP 723 内联元数据，用 `uv run` 执行，不要直接调 `python`：

```bash
uv init --script scripts/foo.py          # create a script with inline metadata
uv add --script scripts/foo.py requests  # add a dependency; never hand-edit the block
uv lock --script scripts/foo.py          # optional: lockfile for full reproducibility
uv run scripts/foo.py --help             # auto-provisions the environment, then runs
```

`uvx` / `uv run` 首次执行需联网解析依赖。目标环境可能离线时，在 SKILL.md 中交代这一失败模式。

### 可解析 Machine-readable

- stdout 只承载结果数据；进度、警告、诊断一律走 stderr。Agent 才能安全地管道 stdout，或与 `jq`、`grep` 等标准工具组合。
- 结果默认输出 JSON；人类可读表格仅在显式 `--format table` 时输出。
- 「查询无结果」输出空结果集（如 `[]`）并以 0 退出——对 Agent 它是有效答案，不是失败。

### 可诊断 Diagnosable

错误信息写到 stderr，统一三段式；FIX 给出可直接复制执行的命令：

```text
ERROR: config file .tool.toml not found
CAUSE: resolved root directory is /abs/path/project (from --root)
FIX:   run: uv run scripts/init.py --root /abs/path/project
```

退出状态按失败类型区分，并在 `--help` 中列出对照，Agent 才能据此选择重试策略：

- `0` = 成功，**包括幂等命中**（"already exists, skipping"），保证 Agent 可安全重试；
- `2` = 用法/参数错误（argparse 默认行为，零成本）；
- 其余失败类型（未找到、前置条件不满足、环境缺失等）各自分配固定退出码——具体号码自行定义，业界无统一标准，但必须文档化。

其余细则：

- 输入 fail-fast：启动即校验参数合法性与路径存在性，报错中回显解析后的绝对路径。
- 前置环境检查：依赖的外部命令（git、node 等）缺失时报错并附安装指引，不要让底层 traceback 泄漏给 Agent。
- 网络请求与子进程调用设置 timeout，防止挂死 Agent 回合。

### 安全可控 Safe

- 非交互：不从 stdin 读入（除非明确设计为管道模式）；禁用分页器（`git --no-pager`、`GIT_PAGER=cat`）；底层工具有确认提示时传 `--yes` 或设 `CI=true`。
- 按操作可逆性分级：

| 操作类型 | 默认行为 |
|---|---|
| 只读 | 直接执行 |
| 可逆写（create-if-not-exists） | 直接执行并报告结果 |
| 不可逆 / 批量变更 | 默认 dry-run 输出预览，`--confirm` / `--force` 才执行 |

- 幂等：重复执行产生同一结果；目标已存在时报告 "skipping" 并以 0 退出。

### 经济 Token-efficient

输出体量由调用方（Agent）控制，脚本只提供默认值与开关，不主动替 Agent 写文件（本条针对「结果输出」，不涉及以生成产物为职责的脚本）：

- 默认限量：列表/批量输出默认截断（如前 50 条）。截断时在 **stderr** 提示总量与获取完整输出的方式——提示不进 stdout，保证 stdout 始终可解析、可重定向：

```text
NOTE: truncated at 50 of 1,832 items.
      more: --limit 2000 | --offset 50 | --full
      offline processing: uv run scripts/lint.py ./src --full > report.json
```

- Agent 通过 `--limit N` / `--offset N` 分页，或 `--full` 一次取全。
- JSON 输出用 envelope 携带截断元数据，截断后仍是合法 JSON：

```json
{"items": ["..."], "total": 1832, "truncated": true}
```

- 需要落盘处理时由 Agent 自行重定向，再用 Read 分页、grep 过滤。

背景：多数 Agent 运行环境对工具输出有硬性截断（约 10–30K 字符），超限信息会静默丢失——默认限量是保护，不是限制。

## 路径约定

Agent 调用脚本时 cwd 是用户项目目录且随场景变化，脚本不能对 cwd 有任何假设：

| 路径类型 | 约定 |
|---|---|
| 脚本自有资源（同目录模板、配置） | 用 `Path(__file__).parent` 定位 |
| 操作目标（用户项目的文件） | 由参数显式传入；尽早 resolve 为绝对路径，在输出与报错中回显 |
| 调用侧 | SKILL.md 中的路径以技能根目录为基准书写；Agent 调用时解析为绝对路径（技能目录位置因安装方式而异） |

可以提供「默认操作 cwd」的便利（`black .` 式惯例），但必须在输出中回显实际操作的绝对路径。

## 语言与运行时

业务逻辑一律用 Python 编写。Bash 只承担 launcher 角色：预检运行时 → 缺失则引导安装 → `exec` 委托。它填补 PEP 723 的自举缺口——uv 缺失时报错发生在 shell 层，Python 脚本没有机会自检：

```bash
#!/usr/bin/env bash
# scripts/run.sh — preflight + delegate; no business logic here
set -euo pipefail

if ! command -v uv >/dev/null 2>&1; then
  cat >&2 <<'EOF'
ERROR: uv not found
CAUSE: this script relies on uv to run PEP 723 scripts
FIX:   install uv and retry: curl -LsSf https://astral.sh/uv/install.sh | sh
EOF
  exit 1
fi

exec uv run "$(dirname "$0")/main.py" "$@"
```

这个范例同时示范了三段式错误、脚本相对路径定位（`$(dirname "$0")`）与参数转发（`"$@"`）。

## 验证仪式

交付前实测三遍，缺一不可：

1. `--help`——确认用法说明与示例完整、可直接运行；
2. happy path——确认 stdout 是干净的结构化结果，无多余噪音；
3. 典型错误路径——确认错误信息为三段式，且 FIX 可行动。
