# practice：SyntaxFlow 实践示例

本目录包含按漏洞类型拆分的实践示例，帮助理解如何用 SyntaxFlow 编写检测规则。

## 文档列表

| 文件 | 内容 |
|------|------|
| rce-detection.md | ProcessBuilder / Runtime.exec RCE 检测 |
| xxe-detection.md | DocumentBuilder.parse XXE 检测 |
| dataflow-filter.md | 数据流 + 过滤函数（如 mysqli_real_escape_string） |
| sql-injection-path-sensitive.md | 路径敏感 SQL 注入（Cookbook 多级告警示例） |

## 使用方式

1. 阅读文档中的规则说明  
2. 准备对应语言的测试代码（如 Java、PHP）  
3. 使用 `yak ssa` 编译后，用 `yak sf` 执行规则  

示例：

```bash
# Java 项目
yak ssa -t . --program test
yak sf --program test rce.sf

# PHP 项目
yak ssa -l php -t . -p dataflow
yak sf -p dataflow rule.sf
```
