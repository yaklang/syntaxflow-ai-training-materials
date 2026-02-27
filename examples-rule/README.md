# examples-rule：语法样例规则

本目录包含从 [syntaxflow-cookbook.pdf](https://ssa.to/cookbook) 提取的各语法样例，按编号组织，用于学习 SyntaxFlow 语法与规则写法。

## 文件列表

| 文件 | 语法点 |
|------|--------|
| 01-desc-statement.sf | desc 语句：title, type, level, risk, lib |
| 02-search-variable-method.sf | 词法搜索：变量名、方法名 |
| 03-search-glob-regexp.sf | 模糊搜索：glob、正则 |
| 04-search-constant.sf | 常量搜索：g/r/e 前缀 |
| 05-filter-condition.sf | ?{} 条件过滤 |
| 06-filter-set-operations.sf | 集合运算 &, -, + |
| 07-func-call-params.sf | 函数调用参数 .parse(* as $x) |
| 08-as-variable.sf | as 中间变量 |
| 09-recursive-chain.sf | 递归链 ... |
| 10-dataflow-usdef.sf | 数据流 ->, -->, #>, #-> |
| 11-nativecall-include.sf | NativeCall include |
| 12-nativecall-typename.sf | typeName, fullTypeName |
| 13-nativecall-dataflow.sf | dataflow 路径过滤 |
| 14-check-alert.sf | check, alert |
| 15-java-xxe.sf | Java XXE 规则 |
| 16-java-url-open.sf | Java URL 请求 |
| 17-java-files-write.sf | Java 文件写入 |
| 18-java-processbuilder-rce.sf | ProcessBuilder RCE |
| 19-java-groovyshell-eval.sf | GroovyShell eval |
| 20-php-sql-inject-dataflow.sf | PHP SQL 注入 + dataflow |
| 21-dependency-version.sf | 依赖版本 version_in |
| 22-include-compose.sf | include 组合规则 |
| 23-dataflow-config.sf | 数据流配置 depth, until |
| 24-xxe-factories.sf | XXE $factories 写法 |
| 25-open-redirect-golang-minimal.sf | 开放重定向最小示例（Golang gin） |

## 使用

规则需在对应语言环境下运行：

```bash
# Java 项目
yak ssa -t . --program xxe
yak sf --program xxe 15-java-xxe.sf

# PHP 项目
yak ssa -l php -t . -p dataflow
yak sf -p dataflow 20-php-sql-inject-dataflow.sf
```

**注意**：含 `<include>` 的规则（11、22、23）依赖 yaklang 内置或 awesome-rule 中的 lib 规则。
