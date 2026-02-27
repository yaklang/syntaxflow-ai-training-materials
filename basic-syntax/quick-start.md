# SyntaxFlow 快速入门指南

## 前置条件

在开始使用 SyntaxFlow 之前，请确保您的系统上已安装 **Yaklang** 环境。设置此环境的最简单方法是使用 Yaklang 的预编译环境：

```bash
bash <(curl -sS -L http://oss.yaklang.io/install-latest-yak.sh)
```

安装完成后，运行以下命令验证版本：

```bash
yak version
```

要使用 SyntaxFlow 的最新功能，建议使用 **Yaklang 版本 1.3.7 或更高版本**。

虽然安装基本的执行环境是必要的，但拥有编程概念的基础知识也将大有裨益。熟悉 **PHP**、**JavaScript** 或 **Java** 等语言将有助于理解我们将要探讨的示例。

---

## 开始使用

假设您对 **SSA（静态单赋值形式）** 和 **SyntaxFlow** 的语法都不熟悉，让我们从最基础的开始。

### 步骤 1：克隆项目仓库

首先，将 `syntaxflow-zero-to-hero` 仓库克隆到您的本地机器上，并进入 `lesson-1-hello-world` 目录：

```bash
git clone https://github.com/yaklang/syntaxflow-zero-to-hero
cd syntaxflow-zero-to-hero/lesson-1-hello-world/
```

### 步骤 2：编译 Hello World 程序

与传统的编程语言中编写 `println("Hello World")` 不同，**SyntaxFlow** 的操作方式有所不同。要使用 SyntaxFlow，我们需要将要分析的代码编译成特定的 **SSA 格式**。

在 `lesson-1-hello-world` 目录下，执行以下命令：

```bash
yak ssa -t . --program lesson1
```

**注意：** `--program lesson1` 选项将您的程序命名为 `lesson1`。在后续的命令中，您将引用此程序名称，因此请记住它。

执行后，您应看到显示编译过程的输出。消息 `finished compiling..., results: ...` 确认编译已成功完成。

#### 理解源码

我们正在分析的源代码被有意设计得简单，以展示 SyntaxFlow 的功能。此代码片段表示一个可能根据用户输入执行系统命令的 Java 控制器。

---

## 执行 SyntaxFlow 规则

现在我们已经编译了程序，接下来我们将执行一个 **SyntaxFlow 规则** 来分析它。规则文件内容如下：

```plaintext
desc(title: "这是 SyntaxFlow 的 Hello World 示例，简单而有效！")

Runtime.getRuntime().exec(* #-> * as $source) as $sink;

check $source then "找到命令执行参数来源（依赖）" else "未找到参数来源";
check $sink then "找到命令执行点" else "未检测到命令执行";
```

此规则使用 SyntaxFlow 的查询语言来检测 Java 代码中的命令执行漏洞。它的重点是：

1. 识别命令执行的 **源头**（用户输入）。
2. 定位命令执行发生的 **位置**（汇点）。

此规则的关键行是：

```plaintext
Runtime.getRuntime().exec(* #-> * as $source) as $sink;
```

### 理解 `#->` 运算符

- `#->` 运算符用于追踪一个值的 **支配者**。
- 它回答的问题是：**"是什么影响了这个特定的值？"**
- 它遵循 **Use-Def 链**，帮助分析一个值的来源和使用方式。
- 这对于理解数据流和识别潜在的安全问题至关重要。

### 运行 SyntaxFlow 规则

要执行规则，请运行：

```bash
yak sf --program lesson1 lesson-1.sf
```

确保 `--program` 参数与您在编译期间使用的名称（`lesson1`）一致。

---

## 总结

1. **将代码编译为 SSA 形式：** 使用 `yak ssa -t ./project-path --program name`
2. **编写 SyntaxFlow 规则：** 创建包含特定查询和检查的 `rule.sf` 文件
3. **执行 SyntaxFlow 规则：** 运行 `yak sf --program name rule.sf`

### 关键要点

- **SyntaxFlow** 是用于静态代码分析的强大工具，特别适用于安全审计。
- 理解像 `#->` 这样的运算符对于追踪数据流和识别漏洞至关重要。
- 该过程包括编译代码、编写规则和分析结果。
