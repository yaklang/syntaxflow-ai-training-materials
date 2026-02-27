# SyntaxFlow 规则文件介绍

在使用 SyntaxFlow 技术过程中，理解规则文件（`.sf` 文件）的结构至关重要。这些文件包含特定的语句和表达式，用于定义如何在代码中搜索特定模式和行为。本章节将通过几个实际案例来展示规则文件的编写方法，并解释每个组成部分的功能和作用。

在后面的叙述中，我们通常将 SyntaxFlow 规则简称为 **SF 文件** 或 **SF 规则**。

### 1. 通用的规则文件架构

SF 规则文件的结构通常遵循以下模式：

- **描述性说明 `desc`**：提供规则的概览和目的。
- **审计语句**：定义特定的代码模式或行为来捕捉和分析。
  - **过滤器**：通过条件表达式过滤和选择代码的特定部分。
  - **变量命名 `as`**：用于后续引用的结果命名。
- **条件检查 `check`**：根据审计语句的结果来断言和输出相应的信息。
- **漏洞输出 `alert`**：通知报告生成器或前端(IRify)需要重点关注的或者存在漏洞的变量信息。

> **注意**：虽然 `desc` 和 `check` 并非必须，但我们强烈建议在编写规则时包含这两个语句。在上述所有的内容中，**审计语句** 是最核心的部分。

---

### 2. 规则文件案例与解读

#### **XXE 漏洞检测规则 (`xxe.sf`)**

```plaintext
desc(
    title: "Check XXE Vulnerability in DocumentBuilderFactory",
    title_zh: "检查 DocumentBuilderFactory 中的 XXE 漏洞",
    risk:XXE,
    type:Vulnerability,
    desc: <<<TEXT
    该规则旨在检查 Java 中 DocumentBuilderFactory 的使用是否存在 XXE 漏洞。XXE（XML External Entity）漏洞可能导致敏感信息泄露或远程代码执行。通过确保 DocumentBuilderFactory 实例未设置危险特性，可以防止此类攻击。
TEXT
    solution: <<<TEXT
    为了防止 XXE 漏洞，建议在创建 DocumentBuilderFactory 实例时，确保未调用 setFeature、setXIncludeAware 或 setExpandEntityReferences 方法。这样可以避免解析外部实体，从而降低 XXE 攻击的风险。
TEXT
)

DocumentBuilderFactory.newInstance()?{!((.setFeature) || (.setXIncludeAware) || (.setExpandEntityReferences))} as $entry;
$entry.*Builder().parse(* #-> as $source);

check $source then "XXE Attack" else "XXE Safe";
alert $source for {
    message:"XXE Attack Detected",
    level: high,
}
```

**解读：**

1. **desc 语句** 描述规则基础信息。其中 `<<<TEXT` 为 heredoc 语法。
2. **审计表达式** 中的 `?{}` 结构用于确保没有调用 `setFeature`、`setXIncludeAware` 或 `setExpandEntityReferences` 方法。`#->` 运算符追踪从 `parse` 方法传入的参数的顶级定义。
3. **check 语句** 基于 `$source` 的存在与否来判定是否可能存在 XXE 攻击。
4. **alert 语句** 输出结果，风险等级为 high。

#### **URL 请求检测规则 (`url-open-connection.sf`)**

```plaintext
URL(* #-> * as $source).openConnection().getResponseMessage() as $sink;

check $sink then "Request From URL" else "No Response From URL";
alert $sink for{
    message:"Request From URL Detected",
    level:info,
}
```

- 此规则追踪 `URL` 对象创建到发起网络请求的完整流程。
- **`$source`** 表示 `URL` 的来源，**`$sink`** 表示从该 `URL` 获得的响应。

#### **本地文件写入检测规则 (`local-file-write.sf`)**

```plaintext
Files.write(, * #-> as $params) as $sink;
$params?{.getBytes()} as $source;

check $source then "Local Files Writer" else "No Files Written";

alert $source for{
    message:"Local Files Write Detected",
    level:info,
}
```

- 此规则检查 `Files.write` 方法的调用，并追踪写入操作中使用的参数。
- 通过检查是否调用了 `getBytes` 方法来确认是否有数据被写入。

---

## 3. desc 内置字段（Cookbook）

| 字段 | 说明 | 示例 |
|------|------|------|
| title | 规则标题 | title: "Check XXE" |
| title_zh | 中文标题 | title_zh: "检测XXE漏洞" |
| type | vuln/audit/config/security | type: vuln |
| level | critical/high/middle/low/info | level: high |
| risk | rce/xxe/sqli/ssrf 等 | risk: rce |
| lib | 作为库供其他规则 include | lib: "java-spring-param" |
| desc | 详细说明（建议 heredoc） | desc: <<<TEXT ... TEXT |

type 简写：vuln(v)、audit(a)、config(c)、security(s)。

---

## 附录：常用运算符和语法

- **`#->`**：追踪变量的定义（Use-Def 链）。
- **`?{}`**：条件过滤，确保满足特定的条件。
- **`as`**：为匹配的结果命名，便于后续引用。
- **`...`**：递归链式调用，如 `DocumentBuilderFactory...parse()`。
