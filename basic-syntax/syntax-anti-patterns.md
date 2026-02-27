# SyntaxFlow 语法反模式（禁止使用）

本文档明确列出 **SyntaxFlow 不支持的语法**，避免与 Semgrep、CodeQL 等其它规则引擎混淆。

## 禁止使用的顶层结构

- **`rule("name")`** — SyntaxFlow 没有 `rule()`。规则文件直接以 `desc()` 开头。
- **`pattern(...)`** — 不存在。使用审计表达式直接编写模式。

## 禁止使用的伪 API

以下语法 **不是** SyntaxFlow 语法，请勿使用：

- **`call("regex|pattern")`** — 不存在。用 `.methodName` 或 glob `*redirect*` 匹配方法。
- **`with_param(source("..."), key("..."))`** — 不存在。用 `* as $var` 或 `*<slice(index=N)>` 捕获参数。
- **`without(pattern("..."))`** — 不存在。用集合差 `$sink - $filtered` 或 `<dataflow(exclude=...)>` 排除。

## 正确替代方式

| 错误示例 | 正确 SyntaxFlow |
|----------|-----------------|
| `rule("my-rule")` | 直接 `desc(...)` 开头 |
| `call("redirect\|location")` | `.Redirect` 或 `*redirect*` 或 `.header`（按语言选） |
| `with_param(source("request"), key("url"))` | `* as $param` 配合 `#->` 数据流追踪 |
| 字符串中 `"a\|b\|c"` 表示多选 | 分别写多条规则，或用 glob `*redirect*`、正则 `/(redirect\|location)/` |

## 规则结构参考

```syntaxflow
desc(
	title: "规则标题"
	type: audit
	level: info
)
// 审计表达式：具体方法名、变量名、#-> 数据流
.someMethod(* #-> as $source);
alert $source for { message: "风险", level: high };
```
