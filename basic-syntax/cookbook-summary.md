# SyntaxFlow Cookbook 摘要

本文件提取自 syntaxflow-cookbook.pdf，补充规则编写、desc 字段、集合运算、依赖版本检测、数据流路径敏感分析等核心内容。

## 规则文件结构

- **desc**：规则描述（推荐必填）
- **审计语句**：核心，定义代码模式与行为
- **check**：根据结果断言输出
- **alert**：报告漏洞或重点变量

## desc 内置字段

| 字段 | 说明 | 示例 |
|------|------|------|
| title | 规则标题 | title: "Check XXE" |
| title_zh | 中文标题 | title_zh: "检测XXE漏洞" |
| type | 规则类型 | type: vuln / audit / config / security |
| level | 风险等级 | level: critical / high / middle / low / info |
| risk | 风险类型 | risk: rce / xxe / sqli / ssrf / ... |
| lib | 作为库被 include | lib: "java-spring-param" |
| desc/description | 详细说明（建议 heredoc） | desc: <<<TEXT ... TEXT |
| cve | CVE 编号 | cve: "CVE-2024-xxx" |

type 简写：vuln(v)、audit(a)、config(c)、security(s)。

## 递归链式调用 `...`

跳过中间调用，直接匹配目标方法：

```syntaxflow
DocumentBuilderFactory...parse()
```

等价于完整链 `DocumentBuilderFactory.newInstance().newDocumentBuilder().parse()`。

## 常量搜索

- `g"a*"`：glob 匹配
- `e<<<CODE password CODE`：精确匹配
- `r"regex"`：正则匹配

## 集合运算

- **交集** `&`：`$callPoint & $filteredCall as $vuln`
- **差集** `-`：`$sink - $filtered as $vuln`
- **并集** `+`：`$paramDirectly + $paramIndirectly as $vuln`

## 依赖版本检测

```syntaxflow
__dependency__.*fastjson.version as $ver;
$ver ?{version_in:(1.2.3,2.3.4]} as $vulnVersion
```

版本区间：`(1,2]` 表示 1 < version <= 2，`[`/`]` 包含，`(`/`)` 不包含。

## lib 与 include

在 desc 中声明 `lib: "java-spring-param"` 后，其他规则可调用：

```syntaxflow
<include('java-spring-param')> as $source;
<include("java-http-sink")> as $sink;
$sink #{include:`<self> & $source`, exclude:`...`}-> as $mid;
alert $mid for { message: "SSRF", risk: ssrf, level: mid };
```

## 数据流路径敏感：`<dataflow>`

在 `-->` 或 `#->` 之后使用，检查路径是否经过特定函数（如过滤函数）：

```syntaxflow
_GET.* as $params;
$params --> * as $sink;
$sink<dataflow(<<<CODE
*?{opcode: call && <getCaller><name>?{have: mysqli_real_escape_string}} as $__next__
CODE)> as $filtered;
$sink - $filtered as $vuln;
alert $vuln;
```

- `$__next__`：满足条件的路径点；若非空，路径被保留
- `<<<CODE ... CODE`：HereDoc 多行规则片段

## typeName 与 fullTypeName

- `typeName`：简单名 + 全限定名
- `fullTypeName`：全限定名（含版本等）

```syntaxflow
JSON.parse<typeName()> as $name
JSON.parse<fullTypeName()> as $name
```
