# awesome-rule：漏洞与安全规则库

本目录包含按 CWE 分类的 SyntaxFlow 安全规则，覆盖 Java、Go、PHP、C 等多语言漏洞检测。

## 目录结构

| 目录 | 说明 |
|------|------|
| java/ | Java 规则（CWE、SCA、lib、components） |
| golang/ | Go 规则 |
| php/ | PHP 规则 |
| c/ | C 规则 |
| general/ | 通用规则 |
| buildin-rule-test/ | 内置规则测试用例 |

## 规则类型

- **CWE-xxx/**：按 CWE 编号分类的漏洞规则（如 CWE-89 SQL 注入、CWE-611 XXE）
- **lib/**：可被 `<include('lib-name')>` 引用的库规则（source/sink/filter）
- **sca/**：依赖与组件安全检查（如 fastjson、shiro）
- **components/**：特定组件规则（log4j、JWT、Quartz 等）

## 使用方式

规则需配合 yaklang SSA 编译后的程序使用：

```bash
yak ssa -t <project_path> --program <program_name>
yak sf --program <program_name> <rule.sf>
```

含 `<include>` 的规则会引用同仓库中声明了 `lib` 的规则。
