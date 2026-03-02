# SyntaxFlow 规则编写策略

参考 irify-sast-skill 的 Engine-First 工作流与用户意图跟踪，适用于 ssa_query 内联查询与 write_syntaxflow_rule 完整规则编写。

## 1. 核心编写思路（漏洞检测）

漏洞规则的本质是验证**用户输入是否经数据流到达危险函数**。编写时可按下述思路拆解：

1. **定位用户输入（source）**：找到用户可控输入的位置（如 `c.Query`、`r.FormValue`、`getParameter`、`@RequestParam` 等）
2. **定位危险函数（sink）**：找到危险调用点（如 `exec`、`Execute`、`Exec`、`parse`、`open` 等）
3. **在危险函数的参数中追踪 topdef**：用 `#->` 从 sink 的参数向前溯源到其定义链起点（topdef）
4. **匹配 topdef 与用户输入**：检查 topdef 数据流是否与 source 相交，即 `$sink_param #-> * & $source`

典型模式：`$sink.dangerousMethod(* #-> as $param); $param #-> * & $source` 或 `$source #-> $sink` 等价表达。

### 案例 1：Gin 反射型 XSS（参考 golang-reflected-xss-gin-context.sf）

**样例**：`c.Query("q")` 获取输入，`c.HTML(..., gin.H{"Query": query})` 直接渲染。

| 步骤 | 对应规则写法 | 说明 |
|------|--------------|------|
| source | `<include('golang-gin-context')> as $sink` | gin 上下文即 source 来源，该 lib 匹配 `*.Query`、`*.PostForm` 等 |
| sink | `$sink.HTML(*<slice(index=3)> #-> as $param)` | 危险函数为 `c.HTML`，第 3 个参数为数据 map，用 slice 取参 |
| topdef | `$param #->` 溯源 | 追踪 `gin.H{"Query": query}` 中 `query` 的来源 |
| filter | `$param.HTMLEscapeString(* #-> as $safe); $param - $safe as $output` | 排除经 `HTMLEscapeString` 转义的路径 |
| 匹配 | `$output?{!opcode:make} as $target; $target.Query as $mid` | 最终要求溯源链中有 `.Query`（用户输入） |

```syntaxflow
<include('golang-gin-context')> as $sink;
$sink.HTML(*<slice(index=3)> #-> as $param)
$param.HTMLEscapeString(* #-> as $safe)
$param - $safe as $output
$output?{!opcode:make} as $target
$target.Query as $mid
alert $mid for { ... };
```

### 案例 2：Golang 命令注入（参考 golang-command-injection.sf）

**样例**：`r.URL.Query().Get("cmd")` → `exec.Command("sh", "-c", cmd)`。

| 步骤 | 对应规则写法 | 说明 |
|------|--------------|------|
| source | `<include('golang-user-input')> as $input` | 匹配 net/http、Gin 等用户输入 |
| sink | `<include('golang-os-exec')> as $sink` | 匹配 exec.Command、Command 等 |
| 匹配 | `$sink & $input as $high` | 取交集：sink 的 topdef 与 source 相交 |

```syntaxflow
<include('golang-os-exec')> as $sink;
<include('golang-user-input')> as $input;
$sink & $input as $high;
alert $high for { ... };
```

### 案例 3：Golang SQL 注入 + 非参数化（参考 golang-database-gin-context-sql.sf）

**样例**：`ctx.Query("id")` → `db.Query("SELECT ... " + id)`。

| 步骤 | 对应规则写法 | 说明 |
|------|--------------|------|
| source | `<include('golang-user-input')> as $input` | 用户输入 |
| sink | `*.Query(* as $param)` / `*.QueryRow(* as $param)` | SQL 查询方法 |
| topdef | `$param #{ until: `/Sprintf/` , until: `*?{opcode:add}` }-> as $unsafe` | 溯源直到 Sprintf 或 add，即字符串拼接 |
| 匹配 | `$unsafe<getCall> *?{*#{until: "* & $input"}-> } as $high` | 调用点前的数据流需到达 $input |

```syntaxflow
<include('golang-database-sql')> as $sink;
<include('golang-user-input')> as $input;
$func(* as $param);
$param #{ until: `/Sprintf/`, until: `*?{opcode:add}` }-> as $unsafe;
$unsafe<getCall> *?{*#{until: "* & $input"}-> } as $high;
alert $high for { ... };
```

### 案例 4：Golang SSTI（参考 golang-template-ssti.sf）

**样例**：`r.URL.Query().Get("name")` 拼接到模板字符串 → `template.Execute(w, ...)`。

| 步骤 | 对应规则写法 | 说明 |
|------|--------------|------|
| sink | `$tmpl.Execute(* #-> as $target)` | Execute 第 1 个参数为数据，用 #-> 溯源 |
| topdef | `$target #{ include: "* & $new", exclude: `*?{opcode:const}` }-> as $high` | 溯源需经过 $new（模板构造），排除纯常量 |

```syntaxflow
$tmpl.Execute(* #-> as $target);
$target #{
    include: "* & $new",
    exclude: `*?{opcode:const}`,
}-> as $high;
alert $high for { ... };
```

### 案例 5：数据流过滤（排除 sanitizer，参考 dataflow-filter.md）

**样例**：`$_GET['x']` → SQL 查询，路径中若有 `mysqli_real_escape_string` 则视为安全。

```syntaxflow
_GET.* as $params;
$params --> * as $sink;
$sink<dataflow(<<<CODE
*?{opcode:call && <getCaller><name>?{have: mysqli_real_escape_string}} as $__next__
CODE)> as $filtered;
$sink - $filtered as $vuln;
alert $vuln;
```

`$sink - $filtered` 得到**未经过滤**的路径，即真正漏洞。

当 matched=false 时，按上述「用户输入→危险函数→topdef→过滤函数」顺序排查，详见 [rule-debugging-strategy.md](./rule-debugging-strategy.md) 第 1 节及案例。

## 2. 遵循用户意图（Critical）

**不要**在用户未明确要求时自动构造 source→sink 漏洞规则。

| 用户问题 | 规则类型 | 示例 |
|----------|----------|------|
| "找用户输入" | **Source-Only** | 列出所有输入端点，不含 sink |
| "找 SQL 注入" | **Source→Sink** | 污点追踪 $input #-> $sink |
| "这个值流到哪" | **正向追踪** | 使用 `-->` 追踪去向 |
| "谁调用了这个函数" | **调用点** | `<getCall>`、`<searchFunc>` |

### Source-Only 示例（Java Spring）

```syntaxflow
// 找所有 Spring MVC 控制器方法
*Mapping.__ref__?{opcode: function} as $endpoints;
alert $endpoints;
```

```syntaxflow
// 找 Spring 控制器方法参数（用户可控）
*Mapping.__ref__?{opcode: function}<getFormalParams>?{opcode: param && !have: this} as $params;
alert $params;
```

### Source→Sink 示例（仅当用户要求漏洞检测时）

```syntaxflow
// RCE: 用户输入流向 exec()
Runtime.getRuntime().exec(* #-> * as $source) as $sink;
alert $sink for { title: "RCE", level: "high" };
```

## 3. 引擎优先（Engine-First，ssa_query 场景）

在使用 ssa_query 时：**先查后读**，不要用 grep 预筛。

1. **先 query**：SyntaxFlow 遍历整个 SSA 图，跨文件追踪数据流
2. **再 read**：query 返回具体文件与行号后，用 Read 查看上下文
3. **Grep 仅用于非代码**：配置文件、模板、构建脚本等 SSA 不处理的内容

**禁止**用 grep 在源码中搜索数据流模式——那是 ssa_query 的职责。

## 4. 自愈式查询（SyntaxFlow 语法错误时）

当 ssa_query 或 check-syntaxflow-syntax 返回解析错误时：

1. **不要**向用户道歉或请求帮助
2. 阅读错误信息（含精确位置与期望 token）
3. 根据错误修复规则
4. 重新执行 query/验证
5. 最多重试 **3 次** 后再报告失败

## 5. 主动安全洞察（Proactive Insights）

查询有结果后，**主动**提出后续问题与建议，而非仅输出结果。

### 发现漏洞时

- **建议修复**：如 "exec() 接收未过滤的用户输入，建议使用白名单或 ProcessBuilder"
- **关联问题**："是否检查其他端点也调用 Runtime.exec()？"
- **交叉检测**：发现 RCE 后，主动扫描 SSRF、命令注入等

### 无结果时

- 解释原因："未发现直接 exec() 调用，但存在 ProcessBuilder 使用，是否需要检查？"
- 建议替代 query："代码可能使用框架抽象，可尝试框架特定模式"

### 结果模糊时

- 请求澄清："发现 8 条到 executeQuery() 的数据流，其中 5 条使用参数化查询（安全）。是否仅过滤出 3 条字符串拼接的？"

## 6. 三层模式：Source-Sink-Filter

标准漏洞检测模式：找 source、找 sink、检查是否有 filter。

```syntaxflow
<include('java-spring-mvc-param')> as $source;
<include("java-common-filter")>() as $filter;
<some-sink> as $sink;

$sink #{until: `* & $source`}-> as $all
$all<dataflow(include=`* & $filter`)> as $filtered
$all - $filtered as $unfiltered

alert $filtered for { level: "mid", message: "有 filter" };
alert $unfiltered for { level: "high", message: "无 filter，高险" };
```

## 7. Include + Exclude 遍历

精确控制数据流追踪的包含与排除：

```syntaxflow
$sink #{
    include: `<self> & $source`,       // 必须到达 source
    exclude: `<self>?{opcode:call}?{!<self> & $source}?{!<self> & $sink}`,  // 跳过无关调用
}-> as $result;
```

## 8. 变量算术

```syntaxflow
$a + $b as $merged;     // 并集：合并多个 source
$all - $safe as $vuln;  // 差集：排除安全路径得到真正漏洞
```

## 9. 类型过滤

```syntaxflow
// 排除 SQL 注入中的安全基本类型
$result?{<typeName>?{!any: Long,Integer,Boolean,Double}} as $nonPrimitive

// 框架类型判断
$val?{<fullTypeName>?{have:'javax.servlet'}} as $servletTypes
```

## 10. 编写顺序（write_syntaxflow_rule）

1. **读懂样例**：明确 sink 危险调用、source 用户输入
2. **最小 sink**：先写 `.dangerousMethod(* #-> as $x); alert $x` 验证
3. **加 source**：用 include 或 .methodName 匹配输入来源
4. **连数据流**：`$source #-> $sink as $vuln`
5. **可选 filter**：排除 sanitizer 路径，分级告警

## 11. 参考

- [rule-debugging-strategy.md](./rule-debugging-strategy.md)：matched=false 时的调试
- [ai-operator-quick-reference.md](../basic-syntax/ai-operator-quick-reference.md)：运算符速查
- [nativecall-quick-reference.md](../basic-syntax/nativecall-quick-reference.md)：NativeCall 速查
