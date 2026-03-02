# SyntaxFlow NativeCall 速查

NativeCall 是 SyntaxFlow 的扩展机制，通过 `<name(args)>` 语法调用内置函数，用于高级 SSA IR 分析。参考 irify-sast-skill 的 nativecall-reference。

## 语法

```
<nativeCallName(arg1, key="value", ...)>
```

## 常用 NativeCall 速查表

| NativeCall | 用途 | 示例 |
|------------|------|------|
| `<include>` | 引入 lib 规则 | `<include('golang-user-input')> as $input` |
| `<typeName>` | 短类型名 | `$vals?{<typeName>?{have:'String'}}` |
| `<fullTypeName>` | 全限定类型名 | `$val?{<fullTypeName>?{have:'javax.servlet'}}` |
| `<name>` | 获取名称 | `<getCallee><name>?{have:toString}` |
| `<getReturns>` | 函数返回值 | `$entryMethods<getReturns>` |
| `<getFormalParams>` | 函数形参 | `$start<getFormalParams>?{opcode: param && !have: this}` |
| `<getFunc>` | 所在函数 | `$sink?{<getFunc><getCurrentBlueprint>}` |
| `<getCall>` | 调用点 | `$entry.Open<getCall>` |
| `<getCallee>` | 被调函数 | `$params<getCallee>?{<name>?{have:toString}}` |
| `<getObject>` | 父对象 | `.readObject?{<typeName>?{have:'XMLDecoder'}}<getObject>` |
| `<getMembers>` | 对象成员 | `Controller.__ref__<getMembers>?{.annotation.*Mapping}` |
| `<slice>` | 按索引取值 | `*<slice(index=1)>`、`*<slice(start=1)>` |
| `<dataflow>` | 数据流过滤 | `$all<dataflow(include=\`* & $filter\`)>` |
| `<mybatisSink>` | MyBatis `${}` 注入点 | `<mybatisSink> as $sink` |
| `<freeMarkerSink>` | FreeMarker 模板注入 | `<freeMarkerSink> as $sink` |
| `<javaUnescapeOutput>` | JSP/Thymeleaf 未转义输出 | `<javaUnescapeOutput> as $sink` |
| `<versionIn>` | 版本范围检查 | `$ver?{version_in:(0.1.0,1.3.0]}` |
| `<sourceCode>` | 获取源码 | `bb1<sourceCode(context=3)>` |

## 典型用法

### include 组合

```syntaxflow
<include('java-spring-param')> as $source;
<include("java-http-sink")> as $sink;

$sink #{
    include: `<self> & $source`,
    exclude: `<self>?{opcode:call}?{!<self> & $source}?{!<self> & $sink}`,
}->as $mid;
alert $mid for { message: "SSRF", level: mid };
```

### 类型过滤

```syntaxflow
$vals?{<typeName>?{have:'String'}} as $strings
$result?{<typeName>?{!any: Long,Integer,Boolean,Double}} as $nonPrimitive
```

### 参数提取

```syntaxflow
.parse(*<slice(index=1)> as $firstArg)
*sql*.append(*<slice(start=1)> as $params)
```

### dataflow 过滤

```syntaxflow
$all<dataflow(include=`* & $filter`)> as $filtered
$all - $filtered as $unfiltered
```

### SCA 版本检查

```syntaxflow
__dependency__.*fastjson.version as $ver;
$ver?{version_in:(0.1.0,1.2.83]} as $vuln
```

## 参考

- [buildin-lib-reference.md](../buildin-lib-reference.md)：各语言 include 详解
- [sf-nativecall.md](./sf-nativecall.md)：NativeCall 机制说明
