# basic-syntax：SyntaxFlow 核心语法文档

本目录包含 SyntaxFlow 语言的核心语法说明，内容主要来自 ssa.to 官方文档及 syntaxflow-cookbook.pdf。

## 文档列表

| 文件 | 内容 |
|------|------|
| intro.md | SyntaxFlow 简介、支持特性 |
| quick-start.md | 快速上手 |
| rule-intro.md | 规则文件结构、desc 内置字段 |
| intro-and-desc.md | desc 语句详解 |
| cookbook-summary.md | Cookbook 摘要（集合运算、lib/include、dataflow） |
| sf-search.md | 词法搜索：变量名、方法名、glob、正则、常量 |
| sf-filter.md | ?{} 条件过滤、集合运算 |
| sf-dataflow.md | 数据流 Use-Def 链 ->, -->, #>, #-> |
| sf-func-call.md | 函数调用参数审计 |
| sf-dot-call-chain.md | 链式调用、递归链 ... |
| sf-variable.md | 变量与中间结果 as |
| sf-calc.md | 计算表达式 |
| sf-nativecall.md | NativeCall 机制与函数列表 |
| sf-sca.md | SCA 依赖分析 |
| sf-file-filter.md | 文件过滤 |
| nativecall-demos.md | NativeCall 常见用法与示例 |
| advanced-analyzing-dataflow.md | 数据流路径敏感分析、<dataflow> |
| syntax-anti-patterns.md | **禁止使用的语法**（rule、pattern、call、with_param、without） |

## 阅读顺序建议

1. **入门**：intro → quick-start → rule-intro  
2. **语句**：intro-and-desc → sf-search → sf-filter → sf-dataflow  
3. **进阶**：sf-func-call → sf-dot-call-chain → sf-nativecall  
4. **实战**：nativecall-demos → advanced-analyzing-dataflow → cookbook-summary  
