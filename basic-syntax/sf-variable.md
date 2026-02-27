# 4. 变量：存储和追踪审计结果

在复杂的代码审计过程中，审计员往往需要追踪和分析多个步骤中的中间结果。通过将这些中间结果暂存为变量，可以大大简化分析过程。**SyntaxFlow** 提供了使用 `as` 关键字将审计结果存储为变量的功能。

## 语法结构

- **RefFilterExpr**：`refVariable filterItem* (as refVariable)?`
- **PureFilterExpr**：`filterExpr (as refVariable)?`

## 使用 `as` 关键字

`as` 关键字用于将某个过滤表达式或操作的结果存储到一个变量中，便于后续的引用和操作。

## 示例说明

### 示例 1：捕获函数调用参数

```syntaxflow
.parse(* as $params);
```

### 示例 2：追踪数据库查询语句

```syntaxflow
.createStatement().executeQuery(* as $sql);
```

### 示例 3：多步骤审计

```syntaxflow
DocumentBuilderFactory.newInstance() as $factory;
$factory.setFeature(* as $features);
$factory.newDocumentBuilder().parse(* as $source);
```

### 示例 4：分析数据流中的中间变量

```syntaxflow
request.getParameter(* as $input);
$input?{<typeName>?{have: filter}} as $safeInput;
alert $safeInput for { msg: "user safe input" };
```

## 重要性与优势

1. **模块化**：将复杂任务分解成多个简单步骤
2. **重用性**：变量可在多个表达式中重复使用
3. **清晰性**：明确标记关键审计点
4. **数据流追踪**：利用 SSA 架构确保追踪准确性
