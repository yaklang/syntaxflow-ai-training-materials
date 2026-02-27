# SyntaxFlow AIKB

SyntaxFlow 知识库，用于 RAG 语义搜索。支持 SyntaxFlow 语法、运算符、NativeCall、规则编写模式等内容的检索。内容来源包括 ssa.to 文档及 **syntaxflow-cookbook.pdf**。

## 目录结构

```
syntaxflow-aikb/
├── README.md              # 本文件
├── basic-syntax/          # 核心语法文档
│   ├── README.md
│   ├── intro.md
│   ├── quick-start.md
│   ├── rule-intro.md
│   ├── intro-and-desc.md
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
│   └── sql-injection-path-sensitive.md   # Cookbook 路径敏感 SQL 注入
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

## 重要：语法约束

- 规则以 `desc()` 开头，**禁止** `rule()`、`pattern()`、`call()`、`with_param()`、`without()` 等非 SyntaxFlow 语法
- 详见 `basic-syntax/syntax-anti-patterns.md`

## 校验

- 规则示例：`yak sf --program test rule.sf`
- 构建输出：`yak scripts/build-syntaxflow-aikb-rag.yak --output /tmp/syntaxflow-aikb.rag`
- ZIP：`yak scripts/merge-in-one-text.yak --output /tmp/syntaxflow-aikb.zip`
