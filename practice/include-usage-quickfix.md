# include 用法速记：必须带 `as $var`

## 常见 AI 错误

```syntaxflow
// ❌ 错误：$gin 未定义，后续 $gin.Context 无法使用
<include('golang-gin-context')>

$gin.Context as $context;
$context.Query(* as $param) as $source;
```

## 正确写法

```syntaxflow
// ✅ 正确：include 必须紧跟 as $var
<include('golang-gin-context')> as $gin;

$gin.Context as $context;
$context.Query(* as $param) as $source;
```

## 规则

- `<include('lib-name')>` **必须**紧跟 `as $var`，将 lib 结果绑定到变量
- 后续才能使用 `$var.Context`、`$var.Query` 等
- 变量名可按语义选择：`as $gin`、`as $ctx`、`as $source` 等

## 其他 lib 示例

```syntaxflow
<include('golang-user-input')> as $input;
<include('golang-http-sink')> as $sink;
<include('java-spring-mvc-param')> as $source;
```
