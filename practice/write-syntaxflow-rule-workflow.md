# write_syntaxflow_rule 工作流与 check-syntaxflow-syntax

本文档说明 AI 在 write_syntaxflow_rule 专注模式下生成、验证 SyntaxFlow 规则的完整流程，以及 `check-syntaxflow-syntax` 工具返回值含义。

## 1. 工作流概览

```
用户需求 → load_capability(write_syntaxflow_rule)
         → init: 分析需求、Grep/RAG 搜索 initial_rule_samples
         → write_rule / modify_rule（含 <|GEN_RULE_...|> 代码块）
         → 语法自动验证（若有错误 → Feedback → modify_rule）
         → 【必须】check-syntaxflow-syntax(path=规则路径)
         → 若有正例：传入 sample_code、filename、language，得到 matched=true 后方可 directly_answer
         → directly_answer（展示规则，系统自动替换为磁盘真实内容）
```

## 2. check-syntaxflow-syntax 工具

### 参数

| 参数 | 必填 | 说明 |
|------|------|------|
| path | 与 syntaxflow-code 二选一 | .sf 规则文件路径 |
| syntaxflow-code | 与 path 二选一 | 规则内容字符串 |
| sample_code | 有正例时必传 | 漏洞样例完整代码 |
| filename | 推荐 | 虚拟文件名，如 vuln.go、Main.java |
| language | 有正例时必传 | golang、java、php、c、javascript、yak、python |

### 返回值

**语法错误时**：
- `passed: false`, `syntax_error: true`, `errors`: 错误详情

**语法通过且无样例**：
- `passed: true`, `message`: 语法检查通过

**语法通过且正例自检 matched**：
- `passed: true`, `matched: true`, `sample_verified: true`
- `alert_count`, `alert_details`：各 alert 变量匹配数量
- `query_results_full`：完整查询结果（含变量、位置、源代码上下文）

**正例自检 matched=false**：
- `matched: false`, `result_vars_diagnostic`：各 alert 变量匹配数量（0 表示数据流未贯通）
- `suggestion`：修复建议

## 3. result_vars_diagnostic 解读

当 matched=false 时，工具返回各变量的匹配数量，用于定位问题。**推荐按固定顺序排查**（详见 `rule-debugging-strategy.md` 第 1 节）：

1. **用户输入是否匹配** → source/include 是否覆盖样例的输入方式
2. **危险函数是否匹配** → sink 是否匹配样例的危险调用
3. **topdef 是否贯通** → `#->` 溯源是否能从 sink 参数到 source
4. **过滤函数是否过严** → dataflow exclude 等是否误排安全路径

- **数量为 0**：数据流未到达该变量，即链在此处断开，按上序从前往后检查。
- **数量 > 0 但 totalAlert=0**：数据流到达了部分变量，但 source→sink 链路未贯通，检查 alert 变量是否覆盖完整

修改规则后：先通过语法验证 → 再重新调用 check-syntaxflow-syntax 并传样例参数。

## 4. 与 ssa_query 的互补

- **ssa_query**：内联规则，即时查询已编译 SSA，适合探索性分析
- **write_syntaxflow_rule**：生成完整 .sf 文件，可持久化、含测试用例、供 IRify 等平台使用

两者共享 SyntaxFlow 运算符与数据流逻辑，区别在于规则结构（desc、alert、file://）。

## 5. 遇到 desc 语法错误时

linter 或 check-syntaxflow-syntax 报 `missing ')'`、`mismatched input ','` 时，通常是 desc 格式问题：

- **必须** 使用 `fieldName: value`，冒号不可省略
- 字段间用**换行**分隔，**禁止**逗号
- 键名不加引号，无尾逗号

**处理流程**：查阅 `desc-errors-troubleshooting.md` 或 `desc-syntax-quick-reference.md`，按正确格式一次性修正，避免多次无效 modify_rule 触发 spin 退出。

## 6. 迭代与 spin 注意事项

- **modify_rule 多次无效**：当多次 modify_rule 报同类错误（如 desc 格式）时，系统可能触发 spin 检测并强制退出，此时控制权回到 default 循环，其 directly_answer 不做 sf_verify_matched 校验，**输出可能未经验证**。
- **规避**：遇到语法错误时，先查阅格式文档，一次性修正；迭代失败时参考 `rule-debugging-strategy.md` 执行止损。

## 7. 参考

- `ai-operator-quick-reference.md`：运算符速查
- `desc-syntax-quick-reference.md`：desc 格式
- `desc-errors-troubleshooting.md`：desc 错误排错
- `ai-common-mistakes.md`：AI 常见错误与规避
- `buildin-lib-reference.md`：include 选择
