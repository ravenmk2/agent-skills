---
name: better-skills
description: Best practices for agent skills. Use when user wants to create, edit, review, or optimize a skill or SKILL.md, when verifying a newly drafted or generated skill.
---

# Better Skills

编写与审查 Agent 技能时的最佳实践与红线。

## Description 与触发时机

**description 是触发器，不是文档。** 它决定技能会不会被加载——读者是做匹配决策的 Agent，不是人。

写法要点：

- **用用户的原话做触发词**：用户发起任务时怎么说（`create, edit, optimize a skill`）就怎么写；不用抽象概括（`working on`）——抽象对人友好，对匹配致命。
- **覆盖「决策点之前」**：用户还没动手、正在考虑要不要做（`may contain executable scripts`）时也该触发；但不超过技能内容能服务的边界，避免错载。
- **避免隐含阶段限定的词**：`BEFORE`、`after using X` 会让 Agent 推断「任务已开始 / 未走该路径则不适用」。
- **不点名其他技能**：划界靠触发词错位（各自独占不同的场景词），不靠互相引用——被引用的技能可能并不存在。
- **祈使 / 强制语气作用有限**：先被选中，强制力才有意义；语义对齐优先于语气强度。

硬约束：

- `description`：1–1024 字符，非空，写「何时用」为主。
- `name`：1–64 字符，仅小写字母、数字、连字符；不以连字符开头或结尾；无连续连字符；必须与目录同名。

## 红线

违反即错、无需判断的硬约束：

- frontmatter 只写 `name` 和 `description`；其余字段（license、compatibility 等）在有特殊需求时由用户决定。
- SKILL.md 正文 ≤500 行（约 5000 tokens）；超出部分下沉到 `references/`，并在 SKILL.md 留指针。
- 不解释 Agent 已知的常识（什么是 PDF、HTTP 如何工作）——每段内容自问：「没有这条，Agent 会做错吗？」不会就删。
- Lack of Surprise：技能内容不得超出 description 声称的意图；不编写误导性内容、恶意代码或任何可能危害系统安全的东西。
- 引用技能内文件用相对路径（以技能根目录为基准），且指针必须带明确的加载条件——「当 API 返回非 200 时读 `references/api-errors.md`」，而非「详见 references/」；引用链保持一层，不做深层嵌套。

## 最佳实践

### 范围与详略

- **切割内聚单元**：一个技能封装一个能与其他技能组合的内聚工作单元。过窄会迫使一次任务加载多个技能（开销与指令冲突），过宽则难以精确触发。
- **适度细节**：简洁的分步指导 + 可运行示例，优于百科全书式文档。当你发现自己在覆盖每一个边缘情况时，把多数交给 Agent 自己的判断。
- **教方法，不给答案**：技能应教会 Agent「如何接近一类问题」（读 schema → 按外键约定 join → 按需聚合），而非某个具体实例的产物（join 这两张表、过滤这个值）。输出模板、硬约束可以具体，方法必须可泛化。

### 控制的分寸

- **specificity 匹配 fragility**：操作脆弱、一致性要紧、顺序必须固定时，写死精确指令；多种做法都成立、任务容忍差异时给自由度——此时解释 why 比刚性指令更有效，理解目的的 Agent 能做出更好的情境判断。
- **给默认值，不给菜单**：多个工具都能用时，选定一个默认并简注备选（「用 pdfplumber；扫描件改用 pdf2image + pytesseract」），而不是罗列四五个并列选项。
- **解释 why 优于堆 MUST**：满篇大写 ALWAYS / NEVER 是黄灯——尽量重构为解释背后的理由。

### 常用模式

按需选用，不必全用：

- **Gotchas 清单**：环境特有、违背合理假设的事实（软删除表要带 `WHERE deleted_at IS NULL`）。放在 SKILL.md 让 Agent 提前读到；每次纠正 Agent 的错误后，把纠正沉淀进来。
- **输出模板**：需要固定输出格式时给具体模板，比散文描述可靠；长模板放 `assets/` 按需加载。
- **多步清单**：步骤有依赖或验证门时，用显式 checklist 防跳步。

## 详细参考

- 技能包含可执行脚本 / 命令时，读 [references/script-engineering.md](references/script-engineering.md)
