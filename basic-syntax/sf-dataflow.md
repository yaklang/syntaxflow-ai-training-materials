# 6. 数据流运算：追踪定义和使用

在静态代码分析和安全审计中，**Use-Def 链**（UD 链）和 **Def-Use 链**（DU 链）的追踪是至关重要的技术。**SyntaxFlow** 支持对 UD 链和 DU 链的追踪，充分利用 SSA 形式的优势，实现对数据流的精确追踪。

## 简介

- **Use-Def 链（UD 链）**：描述一个变量的使用位置与其定义位置之间的关系
- **Def-Use 链（DU 链）**：描述一个变量的定义位置与其后续使用位置之间的关系

通过追踪这些链，审计员可以精准地识别数据在代码中的流向，从而发现潜在的安全漏洞（SQL 注入、命令注入、XSS 等）。

## Use-Def 链追踪语法定义

### 运算符概览

1. **`->` 和 `-->`（使用链追踪）**
   - `->`：追踪到下一个使用该变量或函数的地方（一级）
   - `-->`：追踪直到使用链的结束（Bottom-Use 分析）

2. **`#>` 和 `#->`（定义链追踪）**
   - `#>`：追踪到变量或函数的直接定义点
   - `#->`：追踪直到定义链的起点（Top-Def 分析）

3. **`-{...}->` 和 `#{...}->`（定制配置）**
   - `-{depth: 5}->`：限制追踪深度为 5 层
   - `-{include:...}->`：只保留符合配置项的结果
   - `-{exclude:...}->`：排除符合配置项的结果
   - `#{until:...}->`：追踪直到满足 until 条件

### 语法结构

```antlr
filterItem
    | '->'       # NextFilter
    | '#>'       # DefFilter
    | '-->'      # DeepNextFilter
    | '-{...}->' # DeepNextConfigFilter
    | '#->'      # TopDefFilter
    | '#{...}->' # TopDefConfigFilter
```

## 使用实例

### 案例一：ProcessBuilder RCE

```syntaxflow
desc(title: "rce")

ProcessBuilder(* as $cmd) as $builder
$builder.start() as $execBuilder

check $execBuilder then "fine" else "rce 2 SyntaxFlow error"
alert $execBuilder for {
    message: "发现潜在的远程代码执行漏洞，未正确处理 ProcessBuilder 的参数。",
    risk: rce,
    level: high,
}
```

- `ProcessBuilder(* as $cmd)` 捕获构造函数的所有参数
- `$builder.start()` 追踪 start 方法调用

### 案例二：GroovyShell 代码执行

```syntaxflow
desc(title: "groovy shell eval")

GroovyShell().evaluate(* as $cmd)
$cmd #-> * as $target

check $target then "fine" else "not found groovyShell.evaluate parameter"
```

- `$cmd #-> *` 追踪参数的定义来源直到最初的定义点

## 高级特性：数据流分析配置项

### 1. 追踪深度限制：`depth`

```syntaxflow
$sink #{depth: 5}-> $source;
$source -{depth: 3}-> $sink;
```

### 2. 使用 hook 记录追踪过程中的值

```syntaxflow
$req -{
  hook: `*.getParameter() as $indirectParam`
}->;
```

### 3. include 和 exclude

- **include**：只保留满足条件的路径
- **exclude**：排除满足条件的路径

### 4. until：追踪终止条件

```syntaxflow
$sink #{
    until: "* & $source"
}-> as $controlled_source_site
```

## 最佳实践

1. 明确目标方法或变量
2. 合理设置追踪深度
3. 结合其他过滤条件
4. 模块化编写规则
5. 定期审查和更新规则
