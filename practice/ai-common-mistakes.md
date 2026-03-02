# AI 常见错误与规避

本文档总结 AI 在 write_syntaxflow_rule 流程中 **高频犯错的模式**，以及对应的正确做法，用于训练与提示增强。

## 1. desc 元数据语法错误（最高频）

### 错误形态

AI 常将 JSON/JavaScript 写法带入 desc，导致 parser 报错：

| 错误示例 | 问题 | 正确写法 |
|----------|------|----------|
| `desc(title "检测RCE")` | 缺少冒号 | `desc(title: "检测RCE")` |
| `desc(type vuln, level high)` | 冒号缺失；误用逗号 | `desc(type: vuln, level: high)` 或换行分隔 |
| `desc(title: "x", type: "vuln",)` | 尾逗号（JSON 风格） | 去掉尾逗号 |
| `desc("title": "x")` | 键名加引号 | `desc(title: "x")` |

**根本原因**：SyntaxFlow desc 使用 `fieldName: value`，冒号不可省略；字段间用换行分隔，**禁止逗号**。

### 正确格式速记

```syntaxflow
desc(
	title: "规则标题"
	type: vuln
	level: high
	cwe: "CWE-78"
)
```

- 每行 `key: value`
- 冒号与值之间可有空格
- 字段间换行分隔，不用逗号

详见 `desc-syntax-quick-reference.md` 与 `desc-errors-troubleshooting.md`。

---

## 2. 未验证就输出（spin 导致绕开验证）

### 现象

规则被直接输出给用户，但 **未通过 check-syntaxflow-syntax**，或 matched=false 未被修复。

### 根因

- **spin 检测**：当 AI 多次重复 `modify_rule`（如 13 次以上）且无明显进展时，系统会强制退出当前循环。
- 控制权回到 default 循环，其 `directly_answer` **不做** `sf_verify_matched` 校验。
- 因此 spin 退出后的输出 **可能未经验证**。

### 规避

1. **避免无效 modify_rule 循环**：若 linter/check 持续报同类型错误（如 desc 格式），应一次性按规范修正，而非反复小改。
2. **遇到 desc 错误**：立即查阅 desc 格式文档，按 `fieldName: value`、禁止逗号等规则修正。
3. **迭代失败时**：参考 `rule-debugging-strategy.md` 执行止损（回归参考规则、最小化重试）。

---

## 3. 迭代循环失控（误认为超限）

### 现象

AI 认为“迭代超过限制”而停止。

### 实际情况

- 单循环内部有 `iterationCount > maxIterations` 检查。
- **多数情况下**，实际触发的是 **spin 强制退出**（多次重复 modify_rule），而非达到 max iterations。
- 两者表现类似，但 spin 更常见。

### 规避

- 减少无效 modify_rule 次数：每次修改应有实质性变化。
- 遇到 desc/语法错误时，**一次性修正**，避免同一错误反复触发修改。

---

## 4. 其它禁止语法（与 Semgrep/CodeQL 混淆）

| 禁止用法 | 正确替代 |
|----------|----------|
| `rule("name")` | 直接 `desc(...)` 开头 |
| `flow()`、`source=`、`sink=` | 用 `#->`、`* as $var`、include |
| `call("regex")` | `.methodName` 或 glob `*method*` |
| 字段间用逗号 | 换行分隔 |

详见 `syntax-anti-patterns.md`。

---

## 5. 不主动验证即继续 modify_rule

### 现象

AI 在 modify_rule 后收到语法错误反馈，直接进行下一次 modify_rule，**未调用** check-syntaxflow-syntax 验证。导致：
- 无法获取带行号的完整错误信息，定位困难
- 陷入盲目修改循环（spin）
- 有正例时无法完成 matched 自检

### 根因

- 系统在 write_rule/modify_rule 后会自动执行语法检查并 Feedback，AI 可能误认为“已有验证结果”而不再调用工具
- reactive 提示“下一步：必须调用 check-syntaxflow-syntax”仅在**无反馈**时显示，有错误时 AI 看不到该提示

### 规避

- **每次** write_rule 或 modify_rule 后，**必须**立即调用 check-syntaxflow-syntax 验证
- 有语法错误时：先调用验证获取完整错误（含行号），再根据结果 modify_rule
- 语法通过时：若有正例，传入 sample_code、filename、language 做 matched 自检
- **禁止**在未调用验证工具的情况下进行下一次 modify_rule 或 directly_answer

---

## 6. 工具调用注意

- **check-syntaxflow-syntax**：有正例时 **必须** 传入 `sample_code`、`filename`、`language`，否则无法得到 matched 结果。
- **write_rule vs modify_rule**：初次生成用 `write_rule`；后续修正用 `modify_rule`，不要用 write_rule 覆盖已有文件导致结构丢失。

---

## 参考

- `desc-syntax-quick-reference.md`：desc 格式速查
- `desc-errors-troubleshooting.md`：desc 语法错误排错
- `syntax-anti-patterns.md`：禁止语法
- `rule-debugging-strategy.md`：matched=false 调试
- `write-syntaxflow-rule-workflow.md`：完整工作流
