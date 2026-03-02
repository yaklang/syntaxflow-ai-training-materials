# SyntaxFlow 规则调试策略：matched=false 时的思路

当 check-syntaxflow-syntax 返回 matched=false 时，按以下策略排查，避免无效迭代。

## 1. 理解返回的变量链

工具返回 `result_vars_diagnostic`（所有变量及其匹配数量）和 `diagnostic_hint`（断点解读）。

示例：`$input:1 → $sink:0 → $vuln:0`

- **$sink 为 0**：sink 模式未匹配样例。检查：1) 样例中的危险方法名是否与规则一致；2) include 的 sink lib 是否覆盖该 API
- **$input 为 0**：source 未匹配。检查：1) 样例如何获取用户输入（c.Query / r.FormValue）；2) include 是否选对框架（Gin / net.http）

## 2. 编写策略：从 sink 倒推

1. **定位 sink**：在样例中找出危险调用，如 `exec(cmd)`、`db.Exec(sql)`、`template.Execute(tmpl, data)`
2. **写最小规则**：仅匹配 sink 点，如 `.Execute(* #-> as $x); alert $x`
3. **验证通过后再扩展**：加上 source、include，形成 `$input #-> $sink`

## 3. 迭代失败时的止损

若已 modify_rule 多次仍未通过：

- **停止叠加**：不要再加 ?{}、exclude、新 include
- **回归参考规则**：从 initial_rule_samples 中复制与漏洞类型相同的规则（如 SSTI→golang-template-ssti.sf），仅改方法名以适配样例
- **最小化重试**：删除复杂部分，只保留 sink 匹配，确认能 matched 后再逐步加回

## 4. include 选择速查

| 样例特征 | Source Lib | Sink Lib |
|----------|------------|----------|
| Gin: c.Query, c.PostForm | golang-gin-context | golang-http-sink |
| net/http: r.URL.Query | golang-http-source | golang-http-sink |
| template.Execute, fmt.Sprintf | - | 直接 .Execute(* #->) |
| db.Exec, sql.Open | golang-user-input | golang-database-sink |
| Spring: @RequestParam | java-spring-mvc-param | java-http-sink |
| Servlet: getParameter | java-servlet-param | java-runtime-exec-sink |

## 5. 常见断点原因

| 断点变量 | 可能原因 | 修复方向 |
|----------|----------|----------|
| 链首（$input 等） | include 不匹配框架/语言 | 换 include 或直接用 .methodName |
| 链中（$sink 等） | 方法名/包路径与样例不符 | 用 .MethodName 或 `*method*` 精确匹配 |
| 链尾（$vuln） | #-> 连接断裂或 alert 变量写错 | 检查 $a #-> $b 的 $b 是否为 sink |
