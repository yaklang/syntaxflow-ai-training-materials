# desc 语法错误排错指南

当 check-syntaxflow-syntax 或 linter 报 `missing ')'`、`mismatched input ','` 等与 desc 相关的错误时，按本指南快速定位并修复。

## 1. 常见错误消息与对应问题

| 错误消息（示例） | 可能原因 | 修复方向 |
|------------------|----------|----------|
| `missing ')'` | desc 内格式错误导致括号不匹配；或字段缺冒号 | 检查每行是否 `key: value`，补全冒号 |
| `mismatched input ','` | 字段间用了逗号 | 将逗号改为换行，或删除逗号 |
| `unexpected token` | 键名加引号、尾逗号等 | 去掉 `"title"` 引号、尾逗号 |

## 2. 错误示例与正确写法

### ❌ 错误：缺少冒号

```syntaxflow
desc(title "检测Java命令执行漏洞", type vuln, level high)
```

**问题**：`title`、`type`、`level` 后缺冒号；且用了逗号分隔。

### ✅ 正确

```syntaxflow
desc(
	title: "检测Java命令执行漏洞"
	type: vuln
	level: high
)
```

### ❌ 错误：逗号分隔字段

```syntaxflow
desc(title: "规则", type: audit, level: info)
```

**问题**：SyntaxFlow desc **禁止**用逗号分隔字段。

### ✅ 正确

```syntaxflow
desc(
	title: "规则"
	type: audit
	level: info
)
```

### ❌ 错误：键名加引号

```syntaxflow
desc("title": "规则", "type": "audit")
```

**问题**：键名不应加引号；且禁止逗号。

### ✅ 正确

```syntaxflow
desc(
	title: "规则"
	type: audit
)
```

## 3. 修复检查清单

遇到 desc 相关语法错误时，依次检查：

1. [ ] 每个字段是否为 `fieldName: value`，**冒号不可省略**
2. [ ] 字段之间是否仅用**换行**分隔，**无逗号**
3. [ ] 键名（如 title、type、level）是否**无引号**
4. [ ] 是否有**尾逗号**（最后一行后的 `,`）— 若有则删除
5. [ ] 括号是否配对：`desc(` 与 `)` 一一对应

## 4. 参考

- `basic-syntax/desc-syntax-quick-reference.md`：desc 格式速查
- `basic-syntax/intro-and-desc.md`：desc 完整说明
- `basic-syntax/syntax-anti-patterns.md`：禁止语法
