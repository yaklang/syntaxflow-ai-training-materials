# 2. 搜索定位：搜索符号或方法名

在进行代码审计或安全分析时，快速准确地定位感兴趣的代码片段至关重要。本文将介绍如何利用 **SyntaxFlow** 的词法与符号搜索功能，从变量名和方法名入手，实现高效的代码审计。

## 词法与符号搜索简介

**SyntaxFlow** 提供强大的词法与符号搜索功能，允许用户直接定位特定的变量名、方法名或函数名。这在代码审计和安全分析中尤为重要。

### 语法规则

```antlr
filterItemFirst
    : nameFilter                           # NamedFilter
    | '.' lines? nameFilter                # FieldCallFilter
    ;

nameFilter: '*' | identifier | regexpLiteral;
```

## 具体案例解析

### 1. 搜索特定的变量名

```syntaxflow
documentBuilder;
```

这条规则将匹配代码中所有 `documentBuilder` 变量的实例。

### 2. 搜索方法调用

```syntaxflow
.parse;
```

通过 `.` 前缀，指定搜索的是方法或属性名，而非变量名。

### 3. 结合变量名和方法调用

```syntaxflow
documentBuilder.parse;
```

此查询将定位所有在 `documentBuilder` 对象上调用 `parse` 方法的代码位置。

### 4. 使用正则表达式进行模糊搜索

查找所有以 `get` 开头的方法调用：

```syntaxflow
.get*;
```

或更精确的正则：

```syntaxflow
/(get[A-Z].*)/;
```

### 5. 使用 Glob 格式

```syntaxflow
*config*;
```

此规则将匹配所有包含 `config` 的标识符。

### 6. 搜索常量

常量搜索支持三种格式：`regexp`、`glob`、`exact`。使用前缀 `r`、`g`、`e` 来指定格式：

```syntaxflow
g"a*"

e<<<CODE
password
CODE
```

### 7. 全局搜索

在过滤语法 `?{}` 之前添加 `*` 前缀表示全局搜索：

```syntaxflow
*?{opcode:const} as $const
*?{have:"documentBuilder"} as $docBuilder
*?{have:/^[1-9]\d*$/} as $number
*?{have:parse*} as $parseMethods
```

## 实战中注意事项

### 审计通用代码时避免具体指明参数名

编写 SyntaxFlow 规则时，尽量避免指定具体的参数名。参数命名可能因项目或技术栈不同而有所变化。使用通用的匹配模式（如 `*` 或正则表达式）能够提升规则的灵活性。

### 赋值语句不会中断数据流

在 SSA（静态单赋值）的形式下，赋值语句不会中断数据流。每个变量在生命周期内只被赋值一次，即使存在看似重新赋值的操作，实际会引入新的变量。
