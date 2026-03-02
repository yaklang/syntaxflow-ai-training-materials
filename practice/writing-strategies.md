# SyntaxFlow 规则编写策略

参考 irify-sast-skill 的 Engine-First 工作流与用户意图跟踪，适用于 ssa_query 内联查询与 write_syntaxflow_rule 完整规则编写。

## 1. 遵循用户意图（Critical）

**不要**在用户未明确要求时自动构造 source→sink 漏洞规则。

| 用户问题 | 规则类型 | 示例 |
|----------|----------|------|
| "找用户输入" | **Source-Only** | 列出所有输入端点，不含 sink |
| "找 SQL 注入" | **Source→Sink** | 污点追踪 $input #-> $sink |
| "这个值流到哪" | **正向追踪** | 使用 `-->` 追踪去向 |
| "谁调用了这个函数" | **调用点** | `<getCall>`、`<searchFunc>` |

### Source-Only 示例（Java Spring）

```syntaxflow
// 找所有 Spring MVC 控制器方法
*Mapping.__ref__?{opcode: function} as $endpoints;
alert $endpoints;
```

```syntaxflow
// 找 Spring 控制器方法参数（用户可控）
*Mapping.__ref__?{opcode: function}<getFormalParams>?{opcode: param && !have: this} as $params;
alert $params;
```

### Source→Sink 示例（仅当用户要求漏洞检测时）

```syntaxflow
// RCE: 用户输入流向 exec()
Runtime.getRuntime().exec(* #-> * as $source) as $sink;
alert $sink for { title: "RCE", level: "high" };
```

## 2. 引擎优先（Engine-First，ssa_query 场景）

在使用 ssa_query 时：**先查后读**，不要用 grep 预筛。

1. **先 query**：SyntaxFlow 遍历整个 SSA 图，跨文件追踪数据流
2. **再 read**：query 返回具体文件与行号后，用 Read 查看上下文
3. **Grep 仅用于非代码**：配置文件、模板、构建脚本等 SSA 不处理的内容

**禁止**用 grep 在源码中搜索数据流模式——那是 ssa_query 的职责。

## 3. 自愈式查询（SyntaxFlow 语法错误时）

当 ssa_query 或 check-syntaxflow-syntax 返回解析错误时：

1. **不要**向用户道歉或请求帮助
2. 阅读错误信息（含精确位置与期望 token）
3. 根据错误修复规则
4. 重新执行 query/验证
5. 最多重试 **3 次** 后再报告失败

## 4. 主动安全洞察（Proactive Insights）

查询有结果后，**主动**提出后续问题与建议，而非仅输出结果。

### 发现漏洞时

- **建议修复**：如 "exec() 接收未过滤的用户输入，建议使用白名单或 ProcessBuilder"
- **关联问题**："是否检查其他端点也调用 Runtime.exec()？"
- **交叉检测**：发现 RCE 后，主动扫描 SSRF、命令注入等

### 无结果时

- 解释原因："未发现直接 exec() 调用，但存在 ProcessBuilder 使用，是否需要检查？"
- 建议替代 query："代码可能使用框架抽象，可尝试框架特定模式"

### 结果模糊时

- 请求澄清："发现 8 条到 executeQuery() 的数据流，其中 5 条使用参数化查询（安全）。是否仅过滤出 3 条字符串拼接的？"

## 5. 三层模式：Source-Sink-Filter

标准漏洞检测模式：找 source、找 sink、检查是否有 filter。

```syntaxflow
<include('java-spring-mvc-param')> as $source;
<include("java-common-filter")>() as $filter;
<some-sink> as $sink;

$sink #{until: `* & $source`}-> as $all
$all<dataflow(include=`* & $filter`)> as $filtered
$all - $filtered as $unfiltered

alert $filtered for { level: "mid", message: "有 filter" };
alert $unfiltered for { level: "high", message: "无 filter，高险" };
```

## 6. Include + Exclude 遍历

精确控制数据流追踪的包含与排除：

```syntaxflow
$sink #{
    include: `<self> & $source`,       // 必须到达 source
    exclude: `<self>?{opcode:call}?{!<self> & $source}?{!<self> & $sink}`,  // 跳过无关调用
}-> as $result;
```

## 7. 变量算术

```syntaxflow
$a + $b as $merged;     // 并集：合并多个 source
$all - $safe as $vuln;  // 差集：排除安全路径得到真正漏洞
```

## 8. 类型过滤

```syntaxflow
// 排除 SQL 注入中的安全基本类型
$result?{<typeName>?{!any: Long,Integer,Boolean,Double}} as $nonPrimitive

// 框架类型判断
$val?{<fullTypeName>?{have:'javax.servlet'}} as $servletTypes
```

## 9. 编写顺序（write_syntaxflow_rule）

1. **读懂样例**：明确 sink 危险调用、source 用户输入
2. **最小 sink**：先写 `.dangerousMethod(* #-> as $x); alert $x` 验证
3. **加 source**：用 include 或 .methodName 匹配输入来源
4. **连数据流**：`$source #-> $sink as $vuln`
5. **可选 filter**：排除 sanitizer 路径，分级告警

## 10. 参考

- [rule-debugging-strategy.md](./rule-debugging-strategy.md)：matched=false 时的调试
- [ai-operator-quick-reference.md](../basic-syntax/ai-operator-quick-reference.md)：运算符速查
- [nativecall-quick-reference.md](../basic-syntax/nativecall-quick-reference.md)：NativeCall 速查
