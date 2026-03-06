# SyntaxFlow AIKB

SyntaxFlow 知识库，用于 RAG 语义搜索。支持 SyntaxFlow 语法、运算符、NativeCall、规则编写模式等内容的检索。内容来源包括 ssa.to 文档及 **syntaxflow-cookbook.pdf**。

## 目录结构

```
syntaxflow-aikb/
├── README.md              # 本文件
├── buildin-lib-reference.md  # buildin lib 参考手册（各语言 lib 详解与 include 使用场景）
├── basic-syntax/          # 核心语法文档
│   ├── README.md
│   ├── intro.md
│   ├── quick-start.md
│   ├── rule-intro.md
│   ├── intro-and-desc.md
│   ├── ai-operator-quick-reference.md    # AI 速查：运算符、漏洞模式（ssa_query 风格）
│   ├── cookbook-summary.md   # Cookbook 摘要
│   ├── sf-search.md
│   ├── sf-filter.md
│   ├── sf-dataflow.md
│   ├── sf-func-call.md
│   ├── sf-dot-call-chain.md
│   ├── sf-variable.md
│   ├── sf-calc.md
│   ├── sf-nativecall.md
│   ├── sf-sca.md
│   ├── sf-file-filter.md
│   ├── nativecall-demos.md
│   └── advanced-analyzing-dataflow.md
├── practice/              # 实践示例
│   ├── README.md
│   ├── rce-detection.md
│   ├── xxe-detection.md
│   ├── dataflow-filter.md
│   ├── sql-injection-path-sensitive.md   # Cookbook 路径敏感 SQL 注入
│   ├── rule-self-check-solutions.md      # 规则自检未通过时的解决方案
│   ├── write-syntaxflow-rule-workflow.md # write_syntaxflow_rule 工作流、check-syntaxflow-syntax
│   ├── ai-common-mistakes.md             # AI 常见错误与规避
│   └── desc-errors-troubleshooting.md    # desc 语法错误排错
├── rule-examples/         # 规则示例片段（Markdown）
│   ├── README.md
│   └── samples.md
├── examples-rule/         # 语法样例规则 (.sf，按 Cookbook 组织)
│   ├── README.md
│   └── 01-*.sf ~ 24-*.sf
├── awesome-rule/          # 漏洞与安全规则库（CWE 分类）
│   ├── README.md
│   ├── java/ golang/ php/ c/ general/
│   └── *.sf
└── scripts/               # 构建和维护脚本
    ├── README.md
    ├── build-syntaxflow-aikb-rag.yak
    ├── merge-in-one-text.yak
    └── update-rag.yak
```

各子目录均有 `README.md`，说明其用途与内容。

## 使用说明

### 构建 RAG 索引

从 `basic-syntax/`、`practice/`、`rule-examples/` 收集 .md 文件，调用 AI 生成搜索问题，构建语义索引：

```bash
yak scripts/build-syntaxflow-aikb-rag.yak --output syntaxflow-aikb.rag
```

或指定 yaklang 仓库路径运行（若脚本不在 syntaxflow-aikb 根目录）：

```bash
cd syntaxflow-aikb
yak scripts/build-syntaxflow-aikb-rag.yak --output syntaxflow-aikb.rag
```

- **输入**：`basic-syntax/`、`practice/`、`rule-examples/` 下的 .md 文件
- **集合**：`yaklang-syntaxflow-aikb`
- **输出**：`syntaxflow-aikb.rag` 或 `syntaxflow-aikb-{YAK_VERSION}.rag`

### 打包为 ZIP

将文档打包为 ZIP，用于训练或增量更新：

```bash
yak scripts/merge-in-one-text.yak --output syntaxflow-aikb.zip
```

- **目标目录**：`basic-syntax`、`practice`、`rule-examples`、`examples-rule`
- **扩展名**：`.md`、`.sf`

### 增量更新 RAG

基于现有 RAG + 差异 ZIP 包增量更新（ZIP 中仅含变更的 .md 文件）：

```bash
yak scripts/update-rag.yak --rag-file old.rag --diff-zip syntaxflow-aikb.zip --output new.rag
```

### 导入向量数据库

将生成的 .rag 导入 yaklang 向量库以供语义检索：

```yak
// 在 Yak 中执行
err = rag.Import("syntaxflow-aikb.rag", rag.importName("yaklang-syntaxflow-aikb"))
```

或使用「导入默认知识库」插件，传入 `--rag-file-path`、`--rag-name`、`--rag-serial-version-uid`。

## 参考来源

- **ssa.to** 官网 syntaxflow-guide
- **syntaxflow-cookbook.pdf**（`ssa.to-master/static/pdf/syntaxflow-cookbook.pdf`）

## SyntaxFlow 核心概念

- **`#->`** / **`-->`**：数据流追踪（定义链 / 使用链）
- **`?{}`**：条件过滤
- **`as`**：结果命名
- **`<include>`**：包含外部规则
- **`desc`** / **`check`** / **`alert`**：规则结构
- **NativeCall**：`<getReturns>`、`<dataflow>`、`<typeName>` 等

## AI 规则生成（write_syntaxflow_rule）

本库为 yaklang 的 `write_syntaxflow_rule` ReAct Loop 提供 RAG/Grep 检索素材，与 MCP `ssa_query` 工具互补。参考 **irify-sast-skill** 的编写策略与示例：

- **ai-operator-quick-reference.md**：ssa_query 风格运算符速查，便于 AI 快速生成完整 .sf 规则
- **nativecall-quick-reference.md**：常用 NativeCall 速查（&lt;include&gt;、&lt;typeName&gt;、&lt;dataflow&gt; 等）
- **writing-strategies.md**：用户意图、Engine-First、Source-Sink-Filter、主动安全洞察
- **rule-examples/samples.md**：通用模式（Common Patterns）及示例片段
- **rule-self-check-solutions.md**：规则自检未通过时的解决方案（语法 / matched 分场景）
- **write-syntaxflow-rule-workflow.md**：check-syntaxflow-syntax 工具、result_vars_diagnostic 解读
- **rule-debugging-strategy.md**：matched=false 时的调试思路
- **ai-common-mistakes.md**：AI 常见错误（desc 格式、spin 绕开验证、迭代失控）
- **desc-errors-troubleshooting.md**：desc 语法错误排错、修复检查清单
- **desc-syntax-quick-reference.md**：desc 元数据与测试用例格式

更新本库后需重新执行 `merge-in-one-text.yak` 与 `build-syntaxflow-aikb-rag.yak` 以更新 ZIP 与 RAG 索引。

## 重要：语法约束

- 规则以 `desc()` 开头，**禁止** `rule()`、`pattern()`、`call()`、`with_param()`、`without()` 等非 SyntaxFlow 语法
- 详见 `basic-syntax/syntax-anti-patterns.md`

## 校验

- 规则示例：`yak sf --program test rule.sf`
- 构建输出：`yak scripts/build-syntaxflow-aikb-rag.yak --output /tmp/syntaxflow-aikb.rag`
- ZIP：`yak scripts/merge-in-one-text.yak --output /tmp/syntaxflow-aikb.zip`
