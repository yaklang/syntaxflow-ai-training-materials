# 7. 过滤器：`?{...}` 筛选审计结果

通过使用 `?{...}` 结构，您可以过滤掉不需要的审计值或排除那些没有漏洞的值，从而提升审计的精准度和效率。

## 简介

**分析值的筛选过滤**（Analysis Value Filtering）允许审计员通过条件表达式精确控制哪些数据流继续参与进一步的审计分析。**SyntaxFlow** 提供了 `?{...}` 结构，使得这种筛选变得简单而强大。

## 分析值筛选过滤语法定义

### 运算符概览

| 运算符类型 | 描述 | 示例 |
|-----------|------|------|
| 嵌套语句 | 确定方法成员或属性等嵌套语句的执行是否存在 | `$vars?{.setFeature} as $new` |
| 逻辑非 `!` | 排除特定操作，用于否定条件 | `$vars?{!((.setFeature) \|\| (.setXIncludeAware))} as $new` |
| 逻辑与 `&&` | 同时满足多个条件 | `$vars?{(.setFeature) && (.setXIncludeAware)} as $new` |
| 逻辑或 `\|\|` | 满足任一条件 | `$vars?{(.setFeature) \|\| (.setXIncludeAware)} as $new` |
| Opcode : | 检查特定类型的操作 | `$vars?{opcode: 'call', 'phi'} as $new` |
| Have : | 检查是否包含指定字符串 | `$vars?{have: 'abc'} as $new` |
| HaveAny : | 检查是否包含任一指定字符串 | `$vars?{any: 'abc', 'def'} as $new` |
| VersionIn : | 检查依赖版本是否在某个版本区间 | `$vars?{version_in:(1.2.3,2.3.4]} as $vulnVersion` |

### 语法结构

- **`?{conditionExpression}`**：用于定义过滤条件
- **conditionExpression**：支持嵌套、操作码检查、字符串包含、逻辑与/或等

## 使用实例：审计 DocumentBuilderFactory 防止 XXE

```syntaxflow
desc(
    "Title": "审计因为未设置 setFeature 等安全策略造成的XXE漏洞",
    "Fix": "修复方案：需要用户设置 setFeature / setXIncludeAware / setExpandEntityReferences 等安全配置"
)

$factories = DocumentBuilderFactory.newInstance();

// 过滤出未设置安全属性的 DocumentBuilderFactory 实例
$factories?{!((.setFeature) || (.setXIncludeAware) || (.setExpandEntityReferences))} as $entry;

// 追踪 $entry 中所有 Builder 对象的 parse 方法调用
$entry.*Builder().parse(* #-> as $source;

check $source then "XXE Attack" else "XXE Safe";
```

**解释：**

- `?{!((.setFeature) || (.setXIncludeAware) || (.setExpandEntityReferences))}` 过滤出未调用安全配置方法的实例
- `#->` 追踪 parse 参数的顶级定义

## 高级特性：根据参数筛选方法 `?(...)`

`?(...)` 语法允许在方法调用中直接对参数进行筛选：

```syntaxflow
method?(*?{...}) as $result
```

示例（XXE 规则）：

```syntaxflow
XMLReader.setFeature?( ,*?{=="http://xml.org/sax/features/external-general-entities",?{==false}) as $result
```

- `*?{==...}` 表示第一个参数需要等于指定值
- `?{==false}` 表示第二个参数需要等于 false

## 集合运算（Cookbook）

SyntaxFlow 支持集合运算符，用于组合或排除审计结果：

| 运算符 | 含义 | 示例 |
|--------|------|------|
| `&` | 交集 | `$callPoint & $filteredCall as $vuln` |
| `-` | 差集 | `$sink - $filtered as $vuln` |
| `+` | 并集 | `$paramDirectly + $paramIndirectly as $vuln` |

- **差集**常用于排除已过滤路径，如 `$sink - $filtered` 得到未经过滤的漏洞点。
- **交集**用于找出同时满足两个条件的结果。
- **并集**用于合并来自不同源的结果。

## 最佳实践

1. **明确目标方法或变量**：确保准确无误
2. **合理设置追踪深度**：避免过度追踪导致性能问题
3. **结合其他过滤条件**：提升精准度
4. **模块化编写规则**：提升可读性和可维护性
5. **定期审查和更新规则**：保持审计有效性
