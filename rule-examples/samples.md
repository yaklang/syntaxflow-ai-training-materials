# SyntaxFlow 规则示例片段

按场景整理的规则代码片段，便于快速查阅和复用。所有片段均为 **可执行 SyntaxFlow 语法**。

## XXE（DocumentBuilderFactory）

```syntaxflow
desc(title: "Check XXE", type: vuln, level: high)
DocumentBuilderFactory.newInstance()?{!((.setFeature) || (.setXIncludeAware))} as $entry;
$entry.*Builder().parse(* #-> as $source);
alert $source for { message: "XXE Detected", level: high };
```

## 方法调用与参数捕获

```syntaxflow
desc(title: "Audit parse args")
.parse(* as $source);
$source #-> as $root;
alert $root for { message: "parse 参数来源", level: info };
```

## 数据流 + 过滤（Sanitizer 感知）

```syntaxflow
desc(title: "sql-injection-unsanitized")
_GET.* as $params;
$params --> * as $sink;
$sink<dataflow(<<<CODE
*?{opcode: call && <getCaller><name>?{have: mysqli_real_escape_string}} as $__next__
CODE)> as $filtered;
$sink - $filtered as $vuln;
alert $vuln;
```

## include 组合

```syntaxflow
<include('php-param')> as $source;
<include('php-http-sink')> as $sink;
$sink #{include:`<self> & $source`, exclude:`...`}-> as $mid;
alert $mid for { message: "SSRF", risk: ssrf };
```

## 递归链 `...`

```syntaxflow
DocumentBuilderFactory...parse()
```

## 集合运算

- 交集：`$a & $b as $v`
- 差集：`$sink - $filtered as $vuln`
- 并集：`$a + $b as $v`

## 常量搜索

- `g"a*"` glob
- `r"regex"` 正则
- `e<<<CODE password CODE` 精确匹配

## 带测试用例的 desc 结构

规则应有两个 `desc`：第一个为元数据，第二个为测试用例（`file://`、`safefile://`）。详见 [intro-and-desc.md](../basic-syntax/intro-and-desc.md)。

```syntaxflow
desc(title: "...", type: audit, ...);
// 规则体：include、dataflow、alert
desc(lang: golang, alert_high: 1,
    'file://vuln.go': <<<UNSAFE ... UNSAFE,
    'safefile://safe.go': <<<SAFE ... SAFE);
```

---

## 通用模式（Common Patterns）

### desc 格式（必读，避免语法错误）

**正确格式**：每行 `fieldName: value`，冒号不可省略；字段间换行分隔，**禁止逗号**。

```syntaxflow
desc(
	title: "规则标题"
	type: vuln
	level: high
	cwe: "CWE-78"
)
```

**错误示例**（AI 易犯）：
- `desc(title "x")` → 缺冒号，应为 `title: "x"`
- `desc(title: "x", type: vuln)` → 禁止逗号，应换行分隔
- `desc("title": "x")` → 键名不加引号

详见 `basic-syntax/desc-syntax-quick-reference.md`、`practice/desc-errors-troubleshooting.md`。

### Pattern 1：Source-Sink-Filter 三层检测

```syntaxflow
<include('java-spring-mvc-param')> as $source;
<include("java-common-filter")>() as $filter;
<mybatisSink> as $sink;

$sink #{until: `* & $source`}-> as $all
$all<dataflow(include=`* & $filter`)> as $filtered
$all - $filtered as $unfiltered

alert $filtered for { level: "mid", message: "有 filter" };
alert $unfiltered for { level: "high", message: "无 filter" };
```

### Pattern 2：Include + Exclude 遍历

```syntaxflow
$sink #{
    include: `<self> & $source`,
    exclude: `<self>?{opcode:call}?{!<self> & $source}?{!<self> & $sink}`,
}-> as $result;
```

### Pattern 3：变量并集与差集

```syntaxflow
$a + $b as $merged;     // 并集
$all - $safe as $vuln;  // 差集：排除安全路径
```

### Pattern 4：类型过滤

```syntaxflow
$result?{<typeName>?{!any: Long,Integer,Boolean,Double}} as $nonPrimitive
$val?{<fullTypeName>?{have:'javax.servlet'}} as $servletTypes
```

### Pattern 5：深度限制追踪

```syntaxflow
$sink #{depth: 5}-> as $source     // 最多 5 层
$val #{
  until: `* & $source`,            // 到达 source 即停止
}-> as $reachable
```

### Pattern 6：SCA 依赖版本检查

```syntaxflow
__dependency__.*fastjson.version as $ver;
$ver?{version_in:(0.1.0,1.2.83]} as $vuln
alert $vuln for { level: "high", title: "fastjson 漏洞版本" };
```
