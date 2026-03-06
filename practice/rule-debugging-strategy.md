# SyntaxFlow 规则调试策略：matched=false 时的思路

当 check-syntaxflow-syntax 返回 matched=false 时，按以下策略排查，避免无效迭代。

> **快速解决方案**：若需按场景快速定位修复，见 [rule-self-check-solutions.md](./rule-self-check-solutions.md)。

## 1. 自检未通过时的处理思路（推荐顺序）

按固定顺序逐层排查，避免盲目修改：

| 顺序 | 排查项 | 检查内容 | 对应修复 |
|------|--------|----------|----------|
| 1 | **用户输入是否匹配** | source 模式（include 或 .methodName）是否能匹配样例中的输入获取方式 | 换 include、改方法名（如 c.Query、r.FormValue、getParameter） |
| 2 | **危险函数是否匹配** | sink 模式是否能匹配样例中的危险调用（如 exec、Execute、Exec、parse） | 调整 .MethodName、slice 参数下标、include 的 sink lib |
| 3 | **危险函数的 topdef 是否匹配** | sink 参数的 `#->` 溯源能否到达 source；链是否贯通 | 检查 `$param #-> * & $source` 或 `$source #-> $sink` 的写法；result_vars_diagnostic 中首个为 0 的变量即断点 |
| 4 | **topdef 数据流中是否有过滤函数** | 路径是否被 filter 误判为安全而排除 | 检查 dataflow(exclude=)、`$all - $filtered` 等；若过滤过严，放宽或移除 exclude |

结合 `result_vars_diagnostic` 与 `diagnostic_hint`：变量链中首个匹配数为 0 的变量即为断点，按上表对应层排查。

### 案例 1：用户输入未匹配（$input:0）

**现象**：`result_vars_diagnostic` 显示 `$input:0 → $sink:1 → $vuln:0`，链首为 0。

**样例**：使用 Gin 的 `c.Query("q")` 获取参数。

**错误**：规则用了 `golang-http-source`（匹配 net/http 的 `r.URL.Query`），未覆盖 Gin 的 `c.Query`。

**修复**：改用 `golang-gin-context` 或显式写 `*.Query(* #-> as $param)`。

```syntaxflow
// 错误：Gin 样例用 golang-http-source 无法匹配 c.Query
<include('golang-http-source')> as $input;

// 正确：Gin 需用 golang-gin-context
<include('golang-gin-context')> as $input;
```

### 案例 2：危险函数未匹配（$sink:0）

**现象**：`$input:1 → $sink:0 → $vuln:0`，sink 为 0。

**样例**：`db.QueryRow("SELECT ... " + id)`，即 `*sql.DB` 的 `QueryRow`。

**错误**：规则只匹配了 `$sink.Query`，而 `golang-database-sql` 中可能是 `*.Query` 或 `*.QueryRow`，未覆盖 `QueryRow`。

**修复**：补充 sink 方法，如 `$sink.QueryRow as $func; $sink.Query as $func`。

```syntaxflow
// 错误：只匹配 Query
$sink.Query as $func;

// 正确：Query 与 QueryRow 都要匹配
$sink.QueryRow as $func;
$sink.Query as $func;
*.QueryRow as $func;
*.Query as $func;
```

### 案例 3：topdef 未贯通（$vuln:0 但 $sink>0, $input>0）

**现象**：`$input:1 → $sink:1 → $vuln:0`，source 和 sink 都匹配，但最终 vuln 为 0。

**样例**：Gin XSS，`c.HTML(..., gin.H{"Query": query})`，`query` 来自 `c.Query("q")`。

**错误**：规则取了 `c.HTML` 的第 2 个参数（模板名），而用户数据在第 3 个参数。或 `$param #->` 的 until/include 条件过严，导致链断裂。

**修复**：用 `*<slice(index=3)>` 取第 3 个参数；检查 `#{until:...}`、`#{include:...}` 是否与样例数据流一致。

```syntaxflow
// 错误：HTML 第 3 个参数才是数据，index=2 会取错
$sink.HTML(*<slice(index=2)> #-> as $param)

// 正确：第 3 个参数（0-based 的 index=2 或 1-based 的 3，按实现）
$sink.HTML(*<slice(index=3)> #-> as $param)
```

### 案例 4：过滤函数过严（有漏洞但被排除）

**现象**：样例确认有漏洞，规则应命中，但 matched=false。

**样例**：用户输入经自定义 `sanitize()` 后到 SQL，规则用 `mysqli_real_escape_string` 作为 filter，只排除了 mysqli 转义，未识别自定义 sanitize。

**错误**：`$sink - $filtered` 中 `$filtered` 的 dataflow 条件过窄，把未经过 mysqli 但经过其他过滤的路径也当成漏洞，或反过来把应报的路径误排除。

**修复**：放宽 dataflow 的 exclude；或增加对自定义 sanitize 的识别；或在不确定时先去掉 filter，只做 source→sink 贯通验证。

```syntaxflow
// 过严：只认识 mysqli_real_escape_string
$sink<dataflow(<<<CODE
*?{opcode:call && <getCaller><name>?{have: mysqli_real_escape_string}} as $__next__
CODE)> as $filtered;

// 可考虑：增加其他已知安全函数，或先注释掉 filter 验证贯通
```

### 参考规则文件（按漏洞类型）

| 漏洞类型 | 规则文件 | 说明 |
|----------|----------|------|
| Gin 反射型 XSS | golang-reflected-xss-gin-context.sf | source=gin-context，sink=HTML 第3参，filter=HTMLEscapeString |
| Golang 命令注入 | golang-command-injection.sf | include source + sink，交集匹配 |
| Golang SQL 注入 | golang-database-gin-context-sql.sf | Query/QueryRow 参数溯源，until Sprintf/add |
| Golang SSTI | golang-template-ssti.sf | template.Execute 参数溯源，include/exclude 配置 |

### 案例 5：source 未匹配时拆分模式逐项验证

**现象**：`$source:0`，链首为 0，无法确定是整体模式错误还是其中某一段不匹配。

**策略**：将复合模式拆成多步，分别验证每一步是否能匹配，从而定位断点。

以 `$gin.Context.Query(* as $param) as $source` 为例，若 $source 为 0，可拆分为：

| 步骤 | 拆分写法 | 验证目标 |
|------|----------|----------|
| 1 | `$gin.Context as $context` | 检查 $context 能否匹配 gin.Context 类型/实例 |
| 2 | `$context.Query as $query` 或 `$context.Query(* #-> as $param)` | 检查 .Query 方法能否匹配 |
| 3 | 分别调用 check-syntaxflow-syntax，看 result_vars_diagnostic 中 $context、$query 的匹配数 | 首个为 0 的变量即断点 |

**示例：Gin 用户输入拆分**

```syntaxflow
// 原始：一次性匹配 $gin.Context.Query(* as $param)，若 $source:0 无法定位
<include('golang-gin-context')> as $gin;
$gin.Context.Query(* as $param) as $source;

// 拆分验证：先验证 Context 能否匹配
<include('golang-gin-context')> as $gin;
$gin.Context as $context;
$context.Query(* #-> as $param) as $source;
// 分别观察 $context、$param、$source 的匹配数
alert $source for { ... };
```

若 `$context:0`，则问题在 `$gin.Context`（如 include 输出结构、类型名等）；若 `$context:1` 且 `$param:0`，则问题在 `.Query` 的写法或参数捕获。参考 golang-gin-context.sf：该 lib 使用 `*.Query(* #-> as $param)` 和 `gin.Context as $param` 等分步模式。

**通用原则**：复合链 `A.B.C` 匹配失败时，拆成 `A as $a`、`$a.B as $b`、`$b.C as $c`，逐段检查 result_vars_diagnostic，定位首个为 0 的变量。

### 案例 6：include 未匹配时手动查看 lib 内容并拆分

**现象**：`<include('xxx')> as $var` 后 `$var:0`，或链首变量来自 include 且为 0，难以确定 include 是否选错或模式写法有问题。

**策略**：若 include 找不到/不匹配，尝试**手动查看 lib 文件内容**，按 lib 内部结构拆分并逐段测试。

1. **查找 lib 文件路径**：在 `syntaxflow-ai-training-materials/awesome-rule` 中按 lib 名搜索 `.sf` 文件，路径多为 `{语言}/lib/` 或其子目录
   - 如 `golang-gin-context` → `awesome-rule/golang/lib/golang-gin-context.sf`
   - 如 `java-spring-mvc-param` → `awesome-rule/java/lib/user-input-http-source/java-spring-mvc-params.sf`
   - 如 `php-param` → `awesome-rule/php/lib/php-custom-param.sf`（含 `lib: 'php-param'`）

2. **读取 lib 内容**：查看 lib 内部使用的具体模式（如 `*.Query(* #-> as $param)`、`gin.Context as $param`、`*.PostForm(* #-> as $param)` 等）。

3. **按 lib 结构拆分**：根据 lib 中的分步写法，将当前规则拆成与 lib 一致的多步模式，分别调用 check-syntaxflow-syntax 观察各变量匹配数，定位断点。

4. **示例**：`<include('golang-gin-context')> as $gin` 后 `$gin:0` 或 `$source:0` 时，读取 `awesome-rule/golang/lib/golang-gin-context.sf`，可见其使用 `*.Query(* #-> as $param)`、`*.PostForm(* #-> as $param)`、`gin.Context as $param` 等。据此拆分：先写 `gin.Context as $context` 或 `*.Query(* #-> as $param)` 单独验证，通过后再组合。

## 2. 理解返回的变量链

工具返回 `result_vars_diagnostic`（所有变量及其匹配数量）和 `diagnostic_hint`（断点解读）。

示例：`$input:1 → $sink:0 → $vuln:0`

- **$sink 为 0**：sink 模式未匹配样例。检查：1) 样例中的危险方法名是否与规则一致；2) include 的 sink lib 是否覆盖该 API
- **$input 为 0**：source 未匹配。检查：1) 样例如何获取用户输入（c.Query / r.FormValue）；2) include 是否选对框架（Gin / net.http）

## 3. 编写策略：从 sink 倒推

1. **定位 sink**：在样例中找出危险调用，如 `exec(cmd)`、`db.Exec(sql)`、`template.Execute(tmpl, data)`
2. **写最小规则**：仅匹配 sink 点，如 `.Execute(* #-> as $x); alert $x`
3. **验证通过后再扩展**：加上 source、include，形成 `$input #-> $sink`

## 4. 迭代失败时的止损

若已 modify_rule 多次仍未通过：

- **停止叠加**：不要再加 ?{}、exclude、新 include
- **回归参考规则**：从 initial_rule_samples 中复制与漏洞类型相同的规则（如 SSTI→golang-template-ssti.sf），仅改方法名以适配样例
- **最小化重试**：删除复杂部分，只保留 sink 匹配，确认能 matched 后再逐步加回

## 5. include 选择速查

| 样例特征 | Source Lib | Sink Lib |
|----------|------------|----------|
| Gin: c.Query, c.PostForm | golang-gin-context | golang-http-sink |
| net/http: r.URL.Query | golang-http-source | golang-http-sink |
| template.Execute, fmt.Sprintf | - | 直接 .Execute(* #->) |
| db.Exec, sql.Open | golang-user-input | golang-database-sink |
| Spring: @RequestParam | java-spring-mvc-param | java-http-sink |
| Servlet: getParameter | java-servlet-param | java-runtime-exec-sink |

## 6. 常见断点原因

| 断点变量 | 可能原因 | 修复方向 |
|----------|----------|----------|
| 链首（$input 等） | include 不匹配框架/语言 | 换 include 或直接用 .methodName |
| 链中（$sink 等） | 方法名/包路径与样例不符 | 用 .MethodName 或 `*method*` 精确匹配 |
| 链尾（$vuln） | #-> 连接断裂或 alert 变量写错 | 检查 $a #-> $b 的 $b 是否为 sink |
