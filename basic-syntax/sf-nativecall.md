# 9. 原生扩展：NativeCall 机制

**NativeCall**（原生调用）是 SyntaxFlow 的预封装函数，允许用户在规则中执行各种高级操作，无需编写复杂的逻辑。

## NativeCall 语法定义

```
<nativeCallName(arg1, argName="value", ...)>
```

- `<`：标记开始
- `nativeCallName`：函数名称
- `(...)` 或 `{...}`：参数
- `>`：标记结束

## NativeCall 函数列表

| 函数名称 | 描述 |
|----------|------|
| `<getReturns>` | 获取函数或方法的返回值 |
| `<getFormalParams>` | 检索函数或方法的形式参数 |
| `<getFunc>` | 查找包含特定指令的当前函数 |
| `<getCall>` | 获取值的调用指令 |
| `<getCaller>` | 查找包含特定值的调用指令 |
| `<searchFunc>` | 搜索对某个函数的调用 |
| `<getObject>` | 获取与值关联的对象 |
| `<getMembers>` | 获取对象或类的成员变量或方法 |
| `<getSiblings>` | 检索兄弟节点或值 |
| `<typeName>` | 获取值的类型名称 |
| `<fullTypeName>` | 获取完整类型名称 |
| `<name>` | 获取函数、方法、变量或类型的名称 |
| `<string>` | 将值转换为字符串表示 |
| `<include>` | 包含外部 SyntaxFlow 规则 |
| `<eval>` | 评估字符串形式的 SyntaxFlow 规则 |
| `<fuzztag>` | 评估 Yaklang fuzztag 模板 |
| `<show>` | 输出值（调试用） |
| `<slice>` | 从值中提取切片，如 `<slice(index=1)>` |
| `<regexp>` | 正则表达式匹配或提取 |
| `<const>` | 搜索常量值 |
| `<versionIn>` | 判断依赖版本是否在区间内 |
| `<dataflow>` | 获取数据流信息（在 `-->` 或 `#->` 之后使用） |
| `<mybatisSink>` | 识别 MyBatis Sink |
| `<freeMarkerSink>` | 识别 FreeMarker Sink |

## 常用示例

```syntaxflow
<include('java-spring-param')> as $source;
*<slice(index=1)> as $arg1
$value<typeName> as $type
```
