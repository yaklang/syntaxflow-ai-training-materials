# 规则自检未通过时的解决方案

当 `check-syntaxflow-syntax` 返回 **passed=false** 或 **matched=false** 时，按本指南快速定位并修复。

## 1. 自检失败类型速查

| 返回特征 | 失败类型 | 解决方案文档 |
|----------|----------|--------------|
| `syntax_error: true`, `errors` 非空 | **语法错误** | 第 2 节 |
| `matched: false`, `result_vars_diagnostic` 存在 | **正例自检未通过** | 第 3 节 |

---

## 2. 语法错误时的解决方案

### 2.1 快速决策

| 错误关键词 | 原因 | 修复动作 |
|------------|------|----------|
| `missing ')'`、`mismatched input ','` | desc 格式错误 | 查阅 [desc-errors-troubleshooting.md](./desc-errors-troubleshooting.md)，按 `key: value`、禁止逗号修正 |
| `mismatched input ':'` + heredoc `<<<` | heredoc 结束符有前导空格 | 结束标识符**单独一行、行首无空格**，参考 golang-reflected-xss-gin-context.sf |
| `expecting`、`unexpected token` | 键名引号、尾逗号等 | 去掉 `"title"` 引号、尾逗号；字段换行分隔 |
| `$var 未定义`、`undefined variable` | include 未写 `as $var` | `<include('xxx')> as $gin` 必须带 `as $var`，见 [ai-common-mistakes.md](./ai-common-mistakes.md) §4 |

### 2.2 修复流程

1. **获取完整错误**：调用 check-syntaxflow-syntax，查看 `errors` 中的行号与上下文
2. **按类型查文档**：desc 相关 → `desc-errors-troubleshooting.md`、`desc-syntax-quick-reference.md`
3. **一次性修正**：避免同类错误反复 modify_rule，防止触发 spin 退出

---

## 3. 正例自检 matched=false 时的解决方案

### 3.1 根据 result_vars_diagnostic 快速定位

**原则**：变量链中**首个匹配数为 0 的变量**即为断点，按顺序排查。

| 断点变量 | 可能原因 | 解决方案 |
|----------|----------|----------|
| **链首**（$input、$source、$gin 等） | include 不匹配样例框架；source 模式错误 | 换 include（如 Gin→golang-gin-context）；或直接写 `.Query(* #-> as $param)` |
| **链中**（$sink） | 危险方法名与样例不符；include 的 sink lib 未覆盖 | 补充方法（如 QueryRow、Exec）；用 `*.QueryRow`、`*.Exec` 或对应 include |
| **链尾**（$vuln） | `#->` 溯源未贯通；参数下标取错 | 检查 `$param #-> * & $source`；用 `*<slice(index=N)>` 取正确参数 |

### 3.2 分场景解决方案

#### 场景 A：$source 或 $input 为 0

**现象**：`result_vars_diagnostic` 中链首变量为 0。

**解决步骤**：

1. **确认样例如何获取输入**：`c.Query` / `r.FormValue` / `getParameter` / `$_GET` 等
2. **选择或更换 include**：
   - Gin: `golang-gin-context`
   - net/http: `golang-http-source`
   - Spring: `java-spring-mvc-param`
3. **若 include 仍为 0**：读取 lib 文件（如 `awesome-rule/golang/lib/golang-gin-context.sf`），按 lib 内部分步模式拆分验证，见 [rule-debugging-strategy.md](./rule-debugging-strategy.md) 案例 6

#### 场景 B：$sink 为 0

**现象**：source 有匹配，sink 为 0。

**解决步骤**：

1. **确认样例中的危险调用**：`exec`、`Execute`、`Exec`、`QueryRow`、`parse` 等
2. **调整 sink 模式**：补充未覆盖的方法，如 `$sink.QueryRow as $func`、`*.Execute(* #-> as $x)`
3. **参考同类型规则**：如 SSTI→golang-template-ssti.sf、SQL→golang-database-gin-context-sql.sf

#### 场景 C：source、sink 均 >0，但 $vuln 或 alert 变量为 0

**现象**：数据流未从 sink 参数贯通到 source。

**解决步骤**：

1. **检查参数下标**：如 `c.HTML` 第 3 个参数才是数据，用 `*<slice(index=3)>` 而非 index=2
2. **检查 #-> 连接**：`$param #-> * & $source` 或 `$source #-> $sink` 的 until/include 条件是否过严
3. **最小化验证**：先写 `.Execute(* #-> as $x); alert $x` 仅验证 sink→参数，通过后再加 source

#### 场景 D：样例确认有漏洞，但 matched 仍为 false

**现象**：逻辑上应命中，实际未命中。

**可能原因**：filter 过严，误排安全路径。

**解决步骤**：

1. 检查 `dataflow(exclude=)`、`$all - $filtered` 等条件
2. 放宽 exclude，或增加对自定义 sanitize 的识别
3. **临时去掉 filter**：只做 source→sink 贯通验证，确认通过后再加回 filter

### 3.3 复合模式拆分验证（$source:0 且难以定位时）

当 `$gin.Context.Query(* as $param) as $source` 整体为 0 时，拆成多步分别验证：

```syntaxflow
// 拆成 3 步，分别看 result_vars_diagnostic 中 $context、$param、$source
<include('golang-gin-context')> as $gin;
$gin.Context as $context;
$context.Query(* #-> as $param) as $source;
```

**首个为 0 的变量**即断点：`$context:0`→include 或 `.Context` 问题；`$param:0`→`.Query` 写法问题；`$source:0`→`#->` 条件问题。

### 3.4 迭代失败时的止损

若已 modify_rule 多次仍未通过：

1. **停止叠加**：不要再加 ?{}、exclude、新 include
2. **回归参考规则**：从 initial_rule_samples 复制同类型规则（如 SSTI→golang-template-ssti.sf），仅改方法名以适配样例
3. **最小化重试**：删除复杂部分，只保留 sink 匹配，确认 matched 后再逐步加回

---

## 4. include 选择速查（自检时常用）

| 样例特征 | Source Lib | Sink Lib |
|----------|------------|----------|
| Gin: c.Query, c.PostForm | golang-gin-context | golang-http-sink |
| net/http: r.URL.Query | golang-http-source | golang-http-sink |
| template.Execute | - | 直接 `.Execute(* #->)` |
| db.Exec, sql.Open | golang-user-input | golang-database-sink |
| Spring: @RequestParam | java-spring-mvc-param | java-http-sink |
| Servlet: getParameter | java-servlet-param | java-runtime-exec-sink |

---

## 5. 参考文档

| 文档 | 用途 |
|------|------|
| [rule-debugging-strategy.md](./rule-debugging-strategy.md) | matched=false 详细案例与策略 |
| [desc-errors-troubleshooting.md](./desc-errors-troubleshooting.md) | desc 语法错误排错 |
| [desc-syntax-quick-reference.md](../basic-syntax/desc-syntax-quick-reference.md) | desc 正确格式 |
| [ai-common-mistakes.md](./ai-common-mistakes.md) | 常见错误与规避 |
| [write-syntaxflow-rule-workflow.md](./write-syntaxflow-rule-workflow.md) | 工作流与工具说明 |
| [buildin-lib-reference.md](../buildin-lib-reference.md) | include 与 lib 选择 |
