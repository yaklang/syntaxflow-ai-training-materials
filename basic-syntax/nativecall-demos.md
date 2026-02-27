# 常见 NativeCall 以及用例

NativeCall 是 SyntaxFlow 提供的一种机制，允许用户在规则中调用内置函数来实现复杂的分析功能。

## include

包含外部规则，实现规则复用。在 `desc` 中声明 `lib` 字段后，其他规则可用 `<include('lib-name')>` 调用：

```syntaxflow
<include('java-spring-param')> as $source;
<include("java-http-sink")> as $sink;
```

## typeName / fullTypeName

获取值的类型名。`typeName` 包含简单名和全名；`fullTypeName` 为全限定名。

```syntaxflow
JSON.parse<typeName()> as $name
JSON.parse<fullTypeName()> as $name
```

## getReturns / getFormalParams

- `getReturns`：获取函数返回值
- `getFormalParams`：获取函数形参

```syntaxflow
HHHHH <getReturns> as $sink;
HHHHH <getFormalParams>?{opcode: param && !have: this} as $params;
```

## getFunc / getCall / getCallee

- `getFunc`：获取指令所在的函数
- `getCall`：获取指令的调用
- `getCallee`：获取被调用的函数

```syntaxflow
aArgs<getCall><getCallee><name> as $sink
$params<getCallee>?{<name>?{have:toString}}<getObject>.append(,* as $appendParams)
```

## getObject / getMembers

- `getObject`：获取成员的父对象
- `getMembers`：获取对象的所有成员

```syntaxflow
.b<getObject>.c as $sink
Controller.__ref__<getMembers>?{.annotation.*Mapping} as $entryMethods
```

## slice

对参数或值进行切片：

```syntaxflow
.parse(*<slice(index=1)> as $arg1)
.parse(*<slice(start=2)> as $args)
```

## regexp

正则匹配和分组提取：

```syntaxflow
.annotation.Select.value<regexp(\$\{\s*(\w+)\s*\}, group=1)> as $entry
```

## dataflow

在 `-->` 或 `#->` 之后使用，对数据流路径进行过滤检查：

```syntaxflow
$sink<dataflow(<<<CODE
*?{opcode: call && <getCaller><name>?{have: mysqli_real_escape_string}} as $__next__
CODE)> as $filtered;
```

## const

搜索常量值，支持 `g`(glob)、`r`(regexp)、`e`(exact)：

```syntaxflow
<const(g="127*")> as $output
<const(r="^\\d+\\.\\d+\\.\\d+\\.\\d+$")> as $ipAddresses
```

## versionIn

检查依赖版本是否在区间内：

```syntaxflow
$ver?{version_in:(0.1.0,1.3.0]} as $vulnVersion
$ver in (0.1.0,1.3.0] as $vulnVersion
```
