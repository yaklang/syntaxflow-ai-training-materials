# SyntaxFlow AI 速查：运算符与完整规则结构

本文档采用 ssa_query 风格，为 AI 生成完整 .sf 规则提供** concise 运算符速查**与**典型漏洞检测模式**。适用于 write_syntaxflow_rule、RAG 检索、工具描述增强。

## 1. 核心运算符

| 运算符 | 含义 | 示例 |
|--------|------|------|
| `#->` | 数据流溯源（Use-Def，值从哪来） | `$source #-> * as $root` |
| `-->` | 数据流跟踪（Def-Use，值流到哪去） | `$param --> * as $sink` |
| `?{}` | 条件过滤 | `* ?{opcode: call}` |
| `as $var` | 捕获到变量 | `Runtime.exec(* as $cmd)` |
| `alert $var` | 标记为告警 | `alert $sink for { message: "...", level: high }` |
| `<include('lib')>` | 引入 buildin 库 | `<include('golang-user-input')> as $input` |
| `...` | 递归链（任意深度调用） | `DocumentBuilderFactory...parse()` |
| `*` | 通配 | `* as $any`、`.get*` 方法名 glob |

## 2. 典型漏洞检测模式（单行逻辑）

| 漏洞类型 | 规则逻辑 | 说明 |
|----------|----------|------|
| 命令注入 | `Runtime.getRuntime().exec(* #-> * as $source) as $sink` | exec 参数溯源 |
| SQL 注入 | `*sql*.append(*<slice(start=1)> as $params)` | SQL 追加参数 |
| XSS | `<include('golang-gin-context')> as $ctx; $ctx.Query(*) #-> <include('golang-http-sink')>` | 用户输入 → 输出 |
| SSRF | `<include('golang-user-input')> #-> <include('golang-http-sink')> as $vuln` | 输入 → HTTP 请求 |
| XXE | `.DocumentBuilderFactory.newDocumentBuilder()(* #-> as $src)` | parse 参数溯源 |
| 路径穿越 | `<include('c-user-input')> #-> <include('c-file-path')> as $vuln` | 输入 → 文件路径 |

## 3. 完整 .sf 规则结构（write_syntaxflow_rule 必须）

```
1. desc(元数据)           ← title、type、level、risk、desc、solution、reference
2. 规则体                 ← include、数据流 #->、alert
3. desc(lang, alert_*, file://)  ← 可选，测试用例（第二个 desc）
```

### 最小有效规则

```syntaxflow
desc(title: "Check XXE", type: vuln, level: high, risk: xxe);
.DocumentBuilderFactory.newDocumentBuilder()(* #-> as $src);
alert $src for { message: "XXE in DocumentBuilderFactory", level: high };
```

### 带 include 的数据流规则

```syntaxflow
desc(title: "SSRF", type: vuln, level: high, risk: ssrf);
<include('golang-user-input')> as $input;
<include('golang-http-sink')> as $sink;
$input #-> $sink as $vuln;
alert $vuln for { message: "SSRF", level: high };
```

### 带测试用例（第二个 desc）

```syntaxflow
desc(title: "...", type: audit, level: high);
// 规则体...
desc(lang: golang, alert_high: 1,
    'file://vuln.go': <<<UNSAFE
package main
func vuln() { ... }
UNSAFE
);
```

## 4. 禁止语法（勿与 Semgrep/CodeQL 混淆）

- `rule()`、`pattern()`、`call()`、`with_param()`、`without()`、`testcase()` — **不存在**
- 数据流用 `#->`，禁止误用 `-->` 表示溯源
- 仅含 desc() 的规则**无效**，必须有数据流 + alert

## 5. 与 ssa_query 的区别

| 场景 | ssa_query（MCP） | write_syntaxflow_rule |
|------|------------------|------------------------|
| 输入 | 内联规则字符串 | 完整 .sf 文件 |
| 结构 | 可仅写 `*.exec(* #-> *)` | 必须 desc() 开头 + alert |
| 测试用例 | 无 | 第二个 desc 中 file://、UNSAFE |

ssa_query 的规则逻辑可直接复用到 .sf 规则体，只需补全 desc() 与 alert 结构。

## 6. 常见 Lib 组合

| 漏洞 | Source Lib | Sink Lib |
|------|------------|----------|
| SQL 注入 | golang-user-input / java-servlet-param / php-param | golang-database-sink |
| XSS | golang-gin-context / java-servlet-param | golang-http-sink / php-xss-method |
| 命令注入 | golang-user-input / java-servlet-param | golang-os-exec / java-runtime-exec-sink |
| SSRF | 同上 | golang-http-sink / java-http-sink |
| 路径穿越 | 同上 | golang-file-read-path-sink / c-file-path |

详见 `buildin-lib-reference.md`。
