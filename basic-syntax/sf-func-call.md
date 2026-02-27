# 3. 函数调用：审计调用和参数

在代码审计和安全分析中，精准地识别和审查函数调用及其参数至关重要。**SyntaxFlow** 提供了强大的功能，使得这一任务变得高效和简便。

## 语法定义

```antlr
filterItem
    | '(' lines? actualParam? ')'    # FunctionCallFilter
    ;

actualParam
    : singleParam                   # AllParam
    | actualParamFilter+ singleParam?   # EveryParam
    ;

singleParam: ( '#>' | '#{...}' )? filterStatement ;
```

## 具体案例解析

### 1. 搜索函数调用

```syntaxflow
.parse();
```

匹配所有调用 `.parse` 方法的地方。

### 2. 审计所有参数

```syntaxflow
.parse(* as $source);
```

- `*` 表示匹配任何参数
- `as $source` 将匹配到的参数赋予变量名

### 3. 审计特定位置的参数

```syntaxflow
.parse(* as $arg1, * as $otherArgs);
```

- 逗号前面为第一个参数
- 逗号后面匹配第一个参数后面的所有参数

```syntaxflow
.parse(* as $arg1, * as $arg2, * as $otherArgs);
```

### 4. 使用 NativeCall slice 审计参数

```syntaxflow
.parse(*<slice(index=1)> as $arg1)   // 获取第一个参数
.parse(*<slice(index=5)> as $arg2)   // 获取第五个参数
.parse(*<slice(start=2)> as $args)   // 获取第二个参数开始的所有参数
```

**提示**：逗号适用于简单的参数分隔或需要 `?{}` 过滤的场景；`slice` 更适合精确定位参数位置。

### 5. 实战案例

**DocumentBuilderFactory.parse：**

```syntaxflow
DocumentBuilderFactory.newInstance().newDocumentBuilder().parse(* as $source);
```

**Runtime.getRuntime().exec：**

```syntaxflow
Runtime.getRuntime().exec(* as $cmd);
```

## 实战注意事项

- 避免指定具体的参数名，使用 `*` 或正则提升通用性
- 利用 SSA 架构，赋值不会中断数据流
- 追踪完整的方法调用链
