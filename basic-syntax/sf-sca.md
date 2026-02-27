# 10. SCA: 依赖版本信息检测

**SyntaxFlow** 能够解析依赖包的版本信息，并允许用户编写规则来分析这些版本信息，有效识别存在已知漏洞的依赖包。

## 获取依赖信息

SyntaxFlow 通过内置变量 `__dependency__` 存储解析后的依赖信息：

```syntaxflow
// 获取依赖名以 fastjson 结尾的依赖版本
__dependency__.*fastjson.version as $ver;

// 获取依赖所在的依赖文件
__dependency__.*fastjson.filename as $file;
```

## 筛选依赖版本：version_in

### 语法

- **version_in**：版本区间筛选关键字
- **versionInterval**：`()` 表示不包括边界，`[]` 表示包括边界
- 多个区间用 `||` 连接表示并集

### 示例

```syntaxflow
// 检查版本是否在 1 < version <= 2 范围内
$version in (1,2]

// 检查版本是否在 1.0.0 < version <= 2.0.0 范围内
$version in (1.0.0,2.0.0]

// 多区间
$version in [1.1,1.3] || [2.2,2.3] || [3.2,3.3]

// 使用 ?{} 过滤
$version ?{version_in:(1,2]}
$version ?{version_in:(1.0.0,2.0.0]}
$version ?{version_in:[1.1,1.3] || [2.2,2.3]}
```

## 实战案例：检测 fastjson 漏洞版本

```syntaxflow
__dependency__.*fastjson.version as $ver;
$ver?{version_in:(0.1.0,1.3.0]} as $vulnVersion
```

用于检测存在漏洞的 fastjson 版本范围。
