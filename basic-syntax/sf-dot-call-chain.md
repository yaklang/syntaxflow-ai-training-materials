# 5. 链式调用：递归检查成员和调用

**深度遍历链式调用**（Deep Chain Invocation Tracing）是一种递归检查链式调用的技术，允许快速追踪到某个特定方法调用，无论这个调用是直接发生还是通过多层链式方法调用间接发生的。

## 语法定义

在 **SyntaxFlow** 中，递归检查链式调用通过 `...` 实现：

```antlr
filterItem
    | '...' lines? nameFilter    # DeepChainFilter
    ;
```

- **`...`**：从当前位置开始，向下递归追踪，直到找到指定的方法或函数
- **nameFilter**：指定目标方法或函数的名称

## 示例说明

### 传统方法（完整书写调用链）

```syntaxflow
DocumentBuilderFactory.newInstance().newDocumentBuilder().parse();
```

### 递归链式调用方法（简化）

```syntaxflow
DocumentBuilderFactory...parse();
```

- `DocumentBuilderFactory`：起始类名
- `...`：递归追踪所有可能的链式调用，直到找到 `parse()` 方法
- `parse()`：目标方法名

SyntaxFlow 将自动忽略中间的 `newInstance()` 和 `newDocumentBuilder()` 调用，直接定位到 `parse()` 方法。

## 实战应用

### 追踪 XML 解析器的配置和使用

```syntaxflow
DocumentBuilderFactory...newInstance() as $factory;
$factory.setFeature(* as $features);
$factory...parse(* as $source);
```

**完整查询：**

```syntaxflow
DocumentBuilderFactory...newInstance() as $factory;
$factory.setFeature(* as $features);
$factory...parse(* as $source);
```

## 重要性与优势

1. **简化规则编写**：通过 `...` 省略中间层调用细节
2. **提升审计效率**：快速定位目标方法
3. **增强可维护性**：规则简洁明了
4. **提高审计覆盖率**：递归确保无论多少层链式调用都能准确捕捉
