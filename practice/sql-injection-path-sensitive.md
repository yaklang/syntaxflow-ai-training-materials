# Practice: SQL 注入路径敏感检测（Cookbook 示例）

## 目标

使用 `<dataflow>` 做路径敏感分析，区分直接 SQL 注入、经过滤的注入、疑似过滤但未识别过滤函数等不同风险等级。

## 规则概览

来自 syntaxflow-cookbook.pdf 的 MySQL 注入规则，在单条规则中覆盖多种注入模式。

## 完整规则

```syntaxflow
desc(
    title: "mysql inject",
    type: audit,
    level: low,
)
<include('php-param')> as $params;
<include('php-filter-function')> as $filter;
mysql_query(* as $query);
$query #{
    until: `* & $params`,
}-> as $root;

$root?{!<dataflow(<<<CODE
*?{opcode: call} as $__next__;
CODE)>} as $result;
alert $result for {
    title: "Direct mysql injection",
    title_zh: "直接的mysql注入不经过任何过滤",
    type: 'vuln',
    level: 'high',
};

$root?{<dataflow(<<<CODE
*?{opcode: call && <self> & $filter} as $__next__;
CODE)>} as $filter_result;
alert $filter_result for {
    title: 'Filtered sql injection, filter function detected',
    title_zh: '经过过滤的sql注入，检测到过滤函数',
    type: 'low',
    level: 'low'
};

$root?{<dataflow(<<<CODE
*?{opcode: call && !<self> & $filter} as $__next__;
CODE)>} as $seem_filter;
alert $seem_filter for {
    title: 'Filtered sql injection, but no filter function detected',
    title_zh: '经过过滤的sql注入，但未检测到过滤函数',
    type: 'mid',
    level: 'mid'
};
```

## 逻辑说明

1. **参数与过滤**：`<include('php-param')>` 作为 source，`<include('php-filter-function')>` 作为已知过滤函数集合。
2. **Sink 与回溯**：`mysql_query(* as $query)` 定位 SQL 调用，`$query #{until: * & $params}-> as $root` 沿定义链回溯到用户输入。
3. **直接注入**：`$root?{!<dataflow(...)>}` 表示路径上无任何 `call` 节点，即未经过函数处理，判定为直接注入，高风险。
4. **有过滤函数**：`<dataflow>` 中 `opcode: call && <self> & $filter` 表示路径经过已知过滤函数，风险降低。
5. **疑似过滤但无识别**：`opcode: call && !<self> & $filter` 表示路径有调用，但不在已知过滤集合，中等风险。

## 关键点

- `until`：回溯到满足条件的点为止
- `$__next__`：在 `<dataflow>` 块内标记“符合条件的路径点”
- `!<dataflow(...)>`：路径上不存在满足 dataflow 块条件的点
