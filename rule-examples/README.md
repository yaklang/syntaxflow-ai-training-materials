# rule-examples：规则示例片段

本目录提供按主题整理的 SyntaxFlow 规则片段，便于查阅和复用。

## 内容

- **samples.md**：汇总多种场景的规则代码，包括  
  - XXE（DocumentBuilderFactory）  
  - URL 请求、本地文件写入  
  - ProcessBuilder RCE、GroovyShell eval  
  - SQL 注入 + dataflow 过滤  
  - include 组合规则  
  - 依赖版本检测  
  - 递归链 `...`  
  - **通用模式（Common Patterns）**：Source-Sink-Filter、Include+Exclude 遍历、变量并集/差集、类型过滤、深度限制、SCA 版本检查（参考 irify-sast-skill）  

## 与 examples-rule 的区别

- **rule-examples/samples.md**：Markdown 中的代码片段，侧重说明和教学  
- **examples-rule/*.sf**：可执行的 .sf 规则文件，按语法点编号  

两者互补：samples.md 便于快速浏览，examples-rule 便于实际运行和调试。
