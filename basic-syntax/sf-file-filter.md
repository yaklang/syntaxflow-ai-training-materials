# 11. 文件过滤：检测文件内容

**SyntaxFlow** 能够直接通过正则、XPath、JSONPath 等方式对文件内容进行过滤和审计，从而进行更精确的代码审计和安全分析。

## 语法规则

```antlr
fileFilterContentStatement
    : '${' fileFilterContentInput '}' '.' fileFilterContentMethod (As refVariable)?
    ;

fileFilterContentInput: fileName | regexpLiteral;
fileFilterContentMethod: Identifier '(' fileFilterContentMethodParam? ')';
```

## 具体案例解析

### 1. 使用正则表达式过滤

```syntaxflow
${*}.re("\bya29\.[0-9A-Za-z\-_]+") as $google_oauth_access_token
```

- `${*}` 匹配所有文件（支持 Glob）
- `.re(...)` 正则表达式过滤

### 2. 使用 JSONPath 过滤

```syntaxflow
${*.json}.jsonPath("$.users[*].name") as $userNames
```

### 3. 使用 XPath 过滤 XML

```syntaxflow
${*.xml}.xpath("//user/name") as $xmlUserNames
```

### 4. XPath 过滤 JSON/YAML

```syntaxflow
${*.json}.json("$.auths.*.email") as $result

${*.yml}.xpath(<<<XPATH
//*[contains(lower-case(local-name()), 'passwd') or contains(lower-case(local-name()), 'password')]
XPATH) as $result
```

## 注意事项

- 确保文件路径正确，支持通配符和相对路径
- 正则注意转义字符
- 大文件或复杂条件时注意性能
- 对过滤结果进行适当存储以便后续分析
