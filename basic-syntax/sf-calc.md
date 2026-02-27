# 8. 集合运算：交集 `&`，差集 `-`，并集 `+`

在静态代码分析和安全审计中，**集合运算**用于处理和分析代码中的数据流。**SyntaxFlow** 定义了一套集合运算的运算符。

## 运算符概览

| 运算符 | 描述 | 示例 |
|--------|------|------|
| **交集 `&`** | 找出两个集合中共有的元素 | `$callPoint & $filteredCall as $vuln` |
| **差集 `-`** | 从一个集合中移除存在于另一个集合中的元素 | `$callPoint - $filteredCall as $vuln` |
| **并集 `+`** | 合并两个集合的元素 | `$paramDirectly + $paramIndirectly as $vuln` |

## 语法结构

```antlr
filterItem
    | '+' refVariable    # MergeRefFilter
    | '-' refVariable    # RemoveRefFilter
    | '&' refVariable    # IntersectionRefFilter
    ;
```

## 使用实例

### 交集运算符 `&`

```syntaxflow
$callPoint & $filteredCall as $vuln;
```

找出两个分析过程中共同的安全漏洞点。

### 差集运算符 `-`

```syntaxflow
$callPoint - $filteredCall as $vuln;
```

识别潜在的未过滤或未验证的危险数据流。

### 并集运算符 `+`

```syntaxflow
$paramDirectly + $paramIndirectly as $vuln;
```

合并来自不同源的数据点，进行更全面的分析。

## 最佳实践

1. 明确集合的来源和内容
2. 组合使用不同的运算符
3. 分模块编写规则
4. 定期优化和测试规则
