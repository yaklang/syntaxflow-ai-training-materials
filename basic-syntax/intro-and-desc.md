# 1. `desc` 语句：准备并编写基础描述

在学习编写 SyntaxFlow 规则之前，为了方便用户理解使用，我们使用 XXE 这个漏洞来进行教学。用户可以在手动实现对这个漏洞的分析检测过程中，掌握 SyntaxFlow 的编写技术。

## 准备要审计的代码

我们将存在 XXE 漏洞的 Java 代码保存为 `XXE.java`。这段代码位于一个使用 Spring Framework 构建的 Web 应用中，定义了一个处理 XML 数据的控制器。控制器中的 `one` 方法用来处理通过 HTTP 请求传递的 XML 字符串。漏洞集中在以下三行：

```java
DocumentBuilder documentBuilder = DocumentBuilderFactory.newInstance().newDocumentBuilder();
InputStream stream = new ByteArrayInputStream(xmlStr.getBytes("UTF-8"));
org.w3c.dom.Document doc = documentBuilder.parse(stream);
```

在这三行中，最重要的是 `documentBuilder.parse(stream)`。审计可以先从这个地方开始。

## 编译源码

进入 `XXE.java` 所在的目录，执行以下命令即可编译：

```shell
yak ssa -t . --program xxe
```

编译完成后，看到 `finished compiling` 则说明编译完成了。

## 编写描述

创建 `xxe.sf` 文件，并写入：

```sf
desc(
    title: "Check XXE Vulnerability in DocumentBuilderFactory",
)
```

添加描述有助于在结果输出时包含对结果的解读信息。

### SPEC：语句 `desc`

`desc` 语句用于为规则提供描述性的文本。此语句可以包含一条或多条描述项，这些项可以单独列出或配对（键和值）。

**语法结构：**

- 使用圆括号 `()` 或花括号 `{}` 封装描述项
- 描述项格式：`key: value`，用冒号分隔
- 支持 Heredoc 语法 `<<<TEXT ... TEXT` 提供多行文本

### 示例

```sf
desc(
    title:"Audit Unsafe Database Connection Configurations",
    title_zh:"审计不安全的数据库连接配置",
    type:Config,
    risk:"信息泄露",
    desc:<<<TEXT
    该规则用于审计数据库连接配置是否存在不安全的设置。
TEXT
    solution:<<<TEXT
    为了提高安全性，建议使用加密的连接字符串。
TEXT
)
```

### 优化后的 XXE 描述

```sf
desc(
    title: "Check XXE Vulnerability in DocumentBuilderFactory",
    title_zh: "检查 DocumentBuilderFactory 中的 XXE 漏洞",
    type:Vulnerability,
    risk:XXE,
    desc: <<<TEXT
    该规则旨在检查 Java 中 DocumentBuilderFactory 的使用是否存在 XXE 漏洞。
TEXT
    solution: <<<TEXT
    建议在创建 DocumentBuilderFactory 实例时，确保未调用 setFeature、setXIncludeAware 或 setExpandEntityReferences 方法。
TEXT
)
```

## 编写审计语句

在代码审计过程中，找到特定变量和方法的调用位置是识别潜在漏洞的关键步骤。

### 搜索特定的变量名

```sf
documentBuilder;
```

这条规则会匹配代码中所有 `documentBuilder` 变量的实例。

### 搜索方法调用

```sf
.parse;
```

点 `.` 表示方法调用，后面跟随的方法名。

### 组合使用

```sf
documentBuilder.parse;
```

将只匹配在 `documentBuilder` 变量上调用的 `parse` 方法。
