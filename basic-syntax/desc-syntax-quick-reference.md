# SyntaxFlow desc 块语法速查（官方格式）

本文档提供 SyntaxFlow 官方 `desc` 块语法样例，用于修复 desc 元数据语法错误。**在修改规则前，务必先查阅本文档或检索类似样例，验证正确格式后再修改。**

关键词：desc 语法、desc 元数据、SyntaxFlow desc 块、file://、alert_high、规则结构

---

## 1. 第一个 desc：元数据块（必须）

规则**开头**的 `desc` 仅包含元数据，**禁止**在其中写入 `file://`、`UNSAFE`、`SAFE` 等测试用例。

### 正确格式

```syntaxflow
desc(
    title: "规则标题"
    type: audit
    level: high
    risk: "漏洞类型"
    desc: <<<TEXT
漏洞描述正文...
TEXT
    solution: <<<TEXT
修复建议...
TEXT
    reference: <<<TEXT
参考链接
TEXT
    title_zh: "中文标题"
    rule_id: "可选UUID"
)
```
**注意**：heredoc 结束符（TEXT、DESC、SOLUTION 等）必须**行首无空格**。错误：`    TEXT`。正确：换行后紧跟 `TEXT`。

### 字段说明

| 字段 | 类型 | 说明 |
|------|------|------|
| title | string | 规则英文标题 |
| title_zh | string | 规则中文标题（可选） |
| type | enum | audit / Vulnerability / Config 等 |
| level | enum | high / mid / low / info |
| risk | string | 漏洞类型，如 "ssti"、"xss" |
| desc | heredoc | 漏洞描述，使用 `<<<TEXT ... TEXT` |
| solution | heredoc | 修复建议 |
| reference | heredoc | 参考链接 |
| rule_id | string | 可选 UUID |

### 常见错误（AI 易犯）

- ❌ 在第一个 desc 中写 `'file://xxx': <<<UNSAFE` —— 测试用例必须在**第二个 desc**
- ❌ 使用 `desc: "单行"` 包裹多行文本 —— 多行应用 heredoc `<<<TEXT ... TEXT`
- ❌ **缺少冒号**：`title "xxx"`、`type vuln` 错误。必须写 `title: "xxx"`、`type: vuln`（冒号不可省略）
- ❌ **用逗号分隔字段**：`title: "x", type: audit,` 易触发 `mismatched input ',' expecting <EOF>`。多行 desc 中字段间用**换行**分隔，不要用逗号
- ❌ **heredoc 结束符有前导空格**：`    TEXT`（4 空格+TEXT）不会被识别，heredoc 不闭合，易触发 `mismatched input ':' expecting <EOF>`。结束标识符必须**行首无空格**
- ✅ 正确格式：每行 `fieldName: value`，字段间换行；heredoc 结束符行首无空格。参考 golang-reflected-xss-gin-context.sf

---

## 2. 第二个 desc：测试用例块（可选）

当规则需要嵌入测试样例时，在**规则末尾**使用**第二个独立的 desc**。`file://` 和 `safefile://` **必须**放在此块中。

### 正确格式

```syntaxflow
desc(title: "...", type: audit, level: high, ...);

<include('golang-gin-context')> as $ctx;
$ctx.Query("q") #-> as $source;
... as $sink;
alert $sink for { message: "...", level: high };

desc(lang: golang, alert_high: 1,
    'file://unsafe.go': <<<UNSAFE
package main
func vuln() { ... }
UNSAFE
    ,
    'safefile://safe.go': <<<SAFE
package main
func safe() { ... }
SAFE
);
```

### 字段说明

| 字段 | 说明 |
|------|------|
| lang | 语言：golang、java、php、c 等 |
| alert_high | 高危用例预期命中数（1 表示应命中 1 个） |
| alert_mid | 中危用例预期命中数 |
| 'file://xxx.go' | 不安全样例，内容用 `<<<UNSAFE ... UNSAFE` 包裹 |
| 'safefile://xxx.go' | 安全样例，内容用 `<<<SAFE ... SAFE` 包裹 |

---

## 3. 完整结构示例（参考 golang-template-ssti.sf）

```
1. desc(元数据)           ← 第一个 desc，仅 title、type、level、risk、desc、solution、reference
2. 规则体                 ← include、数据流、alert
3. desc(lang, alert_*, file://)  ← 第二个 desc，测试用例
```

---

## 4. 修改规则前的自检清单

1. 参考本库中已有规则模板（如 golang-template-ssti.sf、golang-reflected-xss-gin-context.sf）的 desc 结构。
2. 在未确认正确 desc 语法格式前，不要继续修改规则文件。
3. 修改后必须调用 `check-syntaxflow-syntax` 验证，若有漏洞样例则传入 path、sample_code、filename、language 进行正例自检。

---

## 5. 参考规则文件

- `golang-template-ssti.sf`：完整 desc 元数据 + 第二个 desc 测试用例
- `golang-reflected-xss-gin-context.sf`：Gin XSS + file:// / safefile:// 结构
- `intro-and-desc.md`：desc 语句详细说明
