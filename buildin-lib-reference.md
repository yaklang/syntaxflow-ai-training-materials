# SyntaxFlow Buildin Lib 参考手册

本文档详细说明 `common/syntaxflow/sfbuildin/buildin` 中各类语言支持的 lib 规则，以及各 lib 的 `<include>` 使用场景。

## 概述

- **lib**：在规则 `desc` 中通过 `lib: "xxx"` 声明，供其他规则通过 `<include('xxx')>` 引用
- **用途**：lib 多为可复用的 **source（污点源）**、**sink（汇点）**、**filter（过滤函数）** 定义，组合后用于数据流检测
- **路径**：buildin 目录按语言划分：`golang/lib/`、`java/lib/`、`php/lib/`、`c/lib/`

### ⚠️ include 必须带 `as $var`

`<include('lib-name')>` **必须**紧跟 `as $var`，将引用结果绑定到变量，否则后续无法使用该引用：

```
// ❌ 错误：$gin 未定义
<include('golang-gin-context')>
$gin.Context as $context;

// ✅ 正确
<include('golang-gin-context')> as $gin;
$gin.Context as $context;
$context.Query(* as $param) as $source;
```

---

## 1. Golang 语言 Lib

### 1.1 用户输入 / HTTP 相关

| Include 名称 | 文件路径 | 用途 | 使用场景 |
|-------------|---------|------|----------|
| `golang-user-input` | `golang/lib/golang-user-input.sf` | 汇总 HTTP/Gin 等用户输入来源 | 作为 **source**，用于 SSRF、SQL 注入、命令注入、XSS、路径穿越等漏洞检测。内部已 include `golang-http-sink`、`golang-gin-context` |
| `golang-gin-context` | `golang/lib/golang-gin-context.sf` | Gin 框架的 `gin.Context` 及其 Query/PostForm 等参数获取 | 作为 **source 或 sink**：XSS（反射输出）、开放重定向、CORS、CSRF 等。检测 `c.Query()`、`c.PostForm()` 等用户可控数据 |
| `golang-http-sink` | `golang/lib/http/golang-http-sink.sf` | HTTP 响应写入点（`w.Write`、`c.JSON` 等） | 作为 **sink**：XSS、信息泄露。内部 include `golang-http-gin`、`golang-http-net` |
| `golang-http-gin` | `golang/lib/http/golang-http-handlefunc-gin.sf` | Gin 的 HTTP 输出 | 被 `golang-http-sink` 使用 |
| `golang-http-net` | `golang/lib/http/golang-http-handlefunc-net.sf` | 标准库 `net/http` 的 ResponseWriter | 被 `golang-http-sink` 使用 |
| `golang-http-source` | `golang/lib/http/golang-http-source.sf` | HTTP 请求参数获取（如 `r.URL.Query().Get`） | 作为 HTTP 输入 source |

### 1.2 数据库相关

| Include 名称 | 文件路径 | 用途 | 使用场景 |
|-------------|---------|------|----------|
| `golang-database-sink` | `golang/lib/sqldb/golang-database-sink.sf` | 汇总所有 Go 数据库执行点 | 作为 **sink**：SQL 注入、信息泄露。内部 include gorm、pop、reform、database/sql、sqlx |
| `golang-database-sql` | `golang/lib/sqldb/golang-database-sql.sf` | 标准库 `database/sql` | SQL 注入检测 |
| `golang-database-gorm` | `golang/lib/sqldb/golang-database-gorm.sf` | GORM ORM 库 | SQL 注入、Gin 上下文 SQL |
| `golang-database-sqlx` | `golang/lib/sqldb/golang-database-sqlx.sf` | sqlx 扩展库 | SQL 注入 |
| `golang-database-pop` | `golang/lib/sqldb/golang-database-pop.sf` | gobuffalo/pop ORM | SQL 注入 |
| `golang-database-reform` | `golang/lib/sqldb/golang-database-reform.sf` | hedonist/reform ORM | SQL 注入 |

### 1.3 文件操作相关

| Include 名称 | 文件路径 | 用途 | 使用场景 |
|-------------|---------|------|----------|
| `golang-file-path` | `golang/lib/golang-file-path.sf` | 文件路径处理（`path/filepath`） | 路径穿越、文件上传、权限检查 |
| `golang-os-sink` | `golang/lib/golang-os-file.sf` | `os` 包文件操作（Open、Create 等） | 路径穿越、文件上传 sink |
| `golang-file-read-path-sink` | `golang/lib/read/golang-file-read-path-sink.sf` | 文件路径作为读取参数 | 路径穿越检测，内部 include bufio、ioutil、os |
| `golang-file-write-path-sink` | `golang/lib/write/golang-file-write-path-sink.sf` | 文件路径作为写入参数 | 任意文件写入检测 |
| `golang-file-read-sink` | `golang/lib/read/golang-file-read-sink.sf` | 文件读取相关 API | 被路径穿越规则使用 |
| `golang-file-write-sink` | `golang/lib/write/golang-file-write-sink.sf` | 文件写入相关 API | 文件上传、XSS 写文件 |

### 1.4 命令执行 / 网络 / 其他

| Include 名称 | 文件路径 | 用途 | 使用场景 |
|-------------|---------|------|----------|
| `golang-os-exec` | `golang/lib/golang-os-exec.sf` | `os/exec` 命令执行 | **sink**：命令注入（与 `golang-user-input` 组合） |
| `golang-ldap-sink` | `golang/lib/golang-ldap-sink.sf` | LDAP 连接与查询 | **sink**：LDAP 注入、认证绕过 |
| `golang-xml-sink` | `golang/lib/golang-xml-sink.sf` | `encoding/xml` 解析 | **sink**：XXE 漏洞 |
| `golang-ftp-sink` | `golang/lib/golang-ftp-sink.sf` | FTP 客户端库 | 硬编码密码、FTP 相关风险 |
| `golang-fmt-print` | `golang/lib/golang-fmt-print.sf` | `fmt` 打印函数 | 信息泄露、日志相关 |

---

## 2. Java 语言 Lib

### 2.1 用户输入 Source

| Include 名称 | 文件路径 | 用途 | 使用场景 |
|-------------|---------|------|----------|
| `java-servlet-param` | `java/lib/user-input-http-source/java-servlet-params.sf` | Servlet `HttpServletRequest` 参数（getParameter、getInputStream、getSession） | **source**：SQL 注入、XSS、命令注入、反序列化、SSRF、路径穿越、日志伪造等几乎所有 Web 漏洞 |
| `java-spring-mvc-param` | `java/lib/user-input-http-source/java-spring-mvc-params.sf` | Spring MVC 控制器方法参数（@RequestParam、@RequestBody 等） | **source**：同 java-servlet-param，与 Spring 项目搭配使用。可加 `?{<typeName>?{have: String}}` 等过滤 |
| `java-spring-mvc-param` (MultipartFile) | 同上 | 限定 `MultipartFile` 类型 | 文件上传漏洞：`?{<typeName>?{have: MultipartFile}}` |
| `java-net-socket-read` | `java/lib/net/java-net-socket-input.sf` | TCP Socket 数据接收 | **source**：资源消耗、DDoS 类检测 |
| `java-spring-param` | 部分规则引用 | Spring 通用参数 | 与 java-servlet-param 类似场景 |

### 2.2 HTTP / 网络 Sink（SSRF）

| Include 名称 | 文件路径 | 用途 | 使用场景 |
|-------------|---------|------|----------|
| `java-http-sink` | `java/lib/http/java-http-sink.sf` | 汇总 Java HTTP 请求执行点 | **sink**：SSRF。内部 include OkHttp、Apache HttpClient、URL.openConnection、RestTemplate、ImageIO.read(URL)、Druid 等 |
| `java-okhttpclient-request-execute` | `java/lib/http/java-okhttpclient-request-execute.sf` | OkHttp 请求 | 被 java-http-sink 使用 |
| `java-apache-commons-httpclient` | `java/lib/http/java-apache-commons-httpclient.sf` | Apache Commons HttpClient | 被 java-http-sink 使用 |
| `java-net-url-connect` | `java/lib/http/java-net-url-connect.sf` | `URL.openConnection` / `openStream` | 被 java-http-sink 使用 |
| `java-image-io-read-url` | `java/lib/http/java-image-io-read-url.sf` | `ImageIO.read(URL)` | 被 java-http-sink 使用 |
| `java-spring-rest-template-request-params` | `java/lib/spring/java-spring-framework-resttemplate-request.sf` | RestTemplate 请求 | 被 java-http-sink 使用 |
| `java-spring-multipartfile-transferTo-target` | `java/lib/spring/java-spring-boot-multipartfile-params.sf` | MultipartFile.transferTo | 文件上传落地 sink |

### 2.3 命令执行 Sink

| Include 名称 | 文件路径 | 用途 | 使用场景 |
|-------------|---------|------|----------|
| `java-runtime-exec-sink` | `java/lib/command-exec-sink/java-runtime-exec.sf` | `Runtime.getRuntime().exec()` | **sink**：命令注入 |
| `java-process-builder-sink` | `java/lib/command-exec-sink/java-process-builder.sf` | `ProcessBuilder` | **sink**：命令注入 |
| `java-command-exec-sink` | `java/lib/command-exec-sink/java-command-exec-misc.sf` | 第三方命令行执行（NuProcess、ProcessExecutor、CommandLine 等） | **sink**：命令注入，内部 include java-process-builder-sink |

### 2.4 代码执行 / 脚本 Sink

| Include 名称 | 文件路径 | 用途 | 使用场景 |
|-------------|---------|------|----------|
| `java-groovy-lang-shell-sink` | `java/lib/code-exec-sink/java-groovy-lang-shell-sink.sf` | GroovyShell 代码执行 | **sink**：Groovy 代码注入 |
| `java-js-sink` | `java/lib/code-exec-sink/java-script-manager-eval.sf` | ScriptEngineManager.eval | **sink**：脚本注入 |

### 2.5 文件操作 Sink

| Include 名称 | 文件路径 | 用途 | 使用场景 |
|-------------|---------|------|----------|
| `java-write-filename-sink` | `java/lib/file-operator/java-file-write-filename-params.sf` | 文件写入时文件名参数（File、FileOutputStream、Files.write、RandomAccessFile 等） | **sink**：路径穿越、任意文件写入 |
| `java-read-filename-sink` | `java/lib/file-operator/java-file-read-filename-params.sf` | 文件读取时文件名参数 | **sink**：路径穿越 |
| `java-delete-filename-sink` | `java/lib/file-operator/java-file-delete-filename-params.sf` | 文件删除 API | 任意文件删除 |

### 2.6 Filter（过滤 / 转义）

| Include 名称 | 文件路径 | 用途 | 使用场景 |
|-------------|---------|------|----------|
| `java-escape-method` | `java/lib/filter/common/java-escape-method.sf` | 转义方法（HTML、XML 等） | **filter**：用于排除已转义的 XSS 路径 |
| `java-common-filter` | `java/lib/filter/common/java-common-filter.sf` | 常见过滤方法 | **filter**：SQL 注入等污点分析中的 sanitizer |
| `java-filter-hostname-prefix` | `java/lib/filter/ssrf/java-filter-hostname-prefix.sf` | 主机名前缀过滤 | SSRF 白名单相关 |
| `is-contain-sanitizer` | `java/lib/filter/java-is-contain-sanitizer.sf` | 含 contain 的过滤判断 | 过滤逻辑 |

### 2.7 日志 / 其他

| Include 名称 | 文件路径 | 用途 | 使用场景 |
|-------------|---------|------|----------|
| `java-log-record` | `java/lib/log/java-log-record.sf` | 日志记录 API | **sink**：日志伪造（Log Forging） |

---

## 3. PHP 语言 Lib

### 3.1 用户输入 Source

| Include 名称 | 文件路径 | 用途 | 使用场景 |
|-------------|---------|------|----------|
| `php-param` | `php/lib/php-custom-param.sf` | `$_GET`、`$_POST`、`$_REQUEST`、`$_COOKIE` | **source**：SQL 注入、XSS、文件包含、命令注入、路径穿越、反序列化等 |
| `php-tp-all-extern-variable-param-source` | `php/lib/php-tp-param.sf` | ThinkPHP `input()`、`I()` 等参数获取 | **source**：ThinkPHP 项目中的用户输入，常与 `php-param` 一起使用 |

### 3.2 输出 / XSS 相关

| Include 名称 | 文件路径 | 用途 | 使用场景 |
|-------------|---------|------|----------|
| `php-xss-method` | `php/lib/php-custom-xss.sf` | XSS 相关输出方法（echo、print、printf 等） | **sink**：XSS、信息泄露 |

### 3.3 文件操作

| Include 名称 | 文件路径 | 用途 | 使用场景 |
|-------------|---------|------|----------|
| `php-file-read` | `php/lib/php-file-read.sf` | 文件读取函数（file_get_contents、fopen、readfile 等） | 路径穿越、本地文件包含 |
| `php-file-write` | `php/lib/php-file-write.sf` | 文件写入函数 | 文件上传、任意文件写入 |
| `php-file-unlink` | `php/lib/php-file-unlink.sf` | 文件删除 `unlink` | 任意文件删除 |

### 3.4 命令执行 / 过滤

| Include 名称 | 文件路径 | 用途 | 使用场景 |
|-------------|---------|------|----------|
| `php-os-exec` | `php/lib/php-exec-function.sf` | 命令执行函数（exec、system、passthru、shell_exec 等） | **sink**：命令注入 |
| `php-filter-function` | `php/lib/php-custom-filter.sf` | 过滤/转义函数（htmlspecialchars、mysqli_real_escape_string 等） | **filter**：排除经过安全处理的数据流路径（SQL 注入、XSS） |

---

## 4. C 语言 Lib

| Include 名称 | 文件路径 | 用途 | 使用场景 |
|-------------|---------|------|----------|
| `c-user-input` | `c/lib/c-user-input.sf` | 用户输入函数（main 参数、getenv、fgets、gets、scanf、read、recv 等） | **source**：SQL 注入、命令注入、XSS、SSRF、路径穿越、信息泄露、认证 bypass |
| `c-file-path` | `c/lib/c-file-path.sf` | 文件路径参数（fopen、open、creat、stat、chmod、unlink、rename 等） | **sink**：路径穿越 |

---

## 5. 典型 include 组合模式

### 5.1 Source → Sink 数据流

```syntaxflow
<include('golang-user-input')> as $input;
<include('golang-http-sink')> as $sink;
$input #-> $sink as $vuln;
alert $vuln for { ... };
```

适用于：SSRF、XSS、开放重定向等。

### 5.2 Source → Sink + Filter 排除

```syntaxflow
<include('php-param')> as $params;
<include('php-tp-all-extern-variable-param-source')> as $params;
<include('php-filter-function')> as $filter;
$params #-> * as $sink;
$sink #{exclude: `...`}-> as $vuln;
alert $vuln for { ... };
```

适用于：SQL 注入（排除经过 mysqli_real_escape_string 等处理的路径）。完整语法参考 `basic-syntax/advanced-analyzing-dataflow.md` 与 `practice/sql-injection-path-sensitive.md`。

### 5.3 Java 多 Source 组合

```syntaxflow
<include('java-servlet-param')> as $source;
<include('java-spring-mvc-param')> as $source;
<include('java-http-sink')> as $sink;
$source #-> $sink as $vuln;
alert $vuln for { ... };
```

适用于：SSRF、命令注入等，同时覆盖 Servlet 与 Spring MVC 项目。

### 5.4 路径穿越

```syntaxflow
<include('c-user-input')> as $user_input;
<include('c-file-path')> as $file_path;
$user_input #-> $file_path as $vuln;
alert $vuln for { ... };
```

### 5.5 数据库 + 用户输入（SQL 注入）

```syntaxflow
<include('golang-database-sink')> as $sink;
<include('golang-user-input')> as $input;
$input #-> $sink as $vuln;
alert $vuln for { ... };
```

---

## 6. Include 命名规则与解析

- **include 名称**：通常对应规则 `desc` 中的 `lib` 字段，如 `lib: "golang-user-input"` → `<include('golang-user-input')>`
- **文件命名**：多数为 `{lib名称}.sf` 或 `{lib名称}s.sf`（如 `java-servlet-params.sf` 对应 `java-servlet-param`）
- **解析方式**：SyntaxFlow 在 buildin 目录中按 lib 名称查找对应规则文件并内联其内容

---

## 7. 编写规则时如何选择 Lib

| 漏洞类型 | 推荐 Source Lib | 推荐 Sink Lib | 可选 Filter |
|---------|----------------|---------------|-------------|
| SQL 注入 | golang-user-input / java-servlet-param / java-spring-mvc-param / php-param | golang-database-sink / java-jdbc-* | php-filter-function / java-common-filter |
| XSS | 同上 | golang-http-sink / golang-gin-context / php-xss-method | java-escape-method / php-filter-function |
| 命令注入 | 同上 | golang-os-exec / java-runtime-exec-sink / java-command-exec-sink / php-os-exec | - |
| SSRF | 同上 | golang-http-sink / java-http-sink | java-filter-hostname-prefix |
| 路径穿越 | 同上 | golang-file-read-path-sink、golang-file-write-path-sink / java-*-filename-sink / c-file-path | - |
| XXE | golang-user-input / java-*-param | golang-xml-sink | - |
| 反序列化 | java-servlet-param / java-spring-mvc-param / php-param | 特定反序列化 API | php-filter-function |
| 日志伪造 | java-servlet-param / java-spring-mvc-param | java-log-record | - |
