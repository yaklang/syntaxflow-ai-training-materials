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
