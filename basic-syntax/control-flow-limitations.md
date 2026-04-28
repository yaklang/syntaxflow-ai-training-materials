# SyntaxFlow 控制流分析与 CFG Native（early-return guard）

面向规则与引擎维护的**单一文档**：Case B 动机、与纯数据流/规则裁剪的边界，以及 **`getCfg`/`cfg*`/`reachabilityGuard`** 的职责、参数与可执行示例。实现细节以 `common/yak/ssaapi` 源码与 `sf_native_call.go` 中 `nc_desc` 为准。

---

## 1. 问题概要

### Case A / Case B

#### Case A：分支内调用

```go
if check(input) {
    vulnerability(input)
}
```

#### Case B：early-return guard

```go
if check(input) {
    return
}
vulnerability(input)
```

Case B 里 sink 隐式满足 `!check(input)`；SyntaxFlow 主攻数据流，条件多经 `phi`，**否定 + 截断**易丢，导致误报/漏报。

### 现状与缺口（简述）

- **SSA**：控制流合流依赖 `phi`，从 `phi` 回溯条件不等于「沿某条执行路径」的证明。
- **规则**：`until`/`exclude`/`include` 只裁搜索空间，不等于路径可达性或 must-check。
- **缺口**：需可查询的 **支配 / 后支配 / 可达性 / early-return guard**，以及 **`reachabilityGuard`（mustExecute）**；不是 SMT。

### 内置 Golang 规则为何仍受累

仅用数据流裁剪时：`until`/`exclude`、`$input & $sink`、检查集合差集等，在 **guard 之后才 sink**、**检查只在分支**、**guard 后才拼接** 时仍易误判。典型规则路径（可作对照）：`sfbuildin/buildin/golang` 下 `cwe-79` Beego XSS、`cwe-89` GORM SQL、`cwe-863` filepath、`cwe-942` Beego CORS。**TopDef**：`include` vs `include_reachable`（需配合 `$锚点<getCfg>`），差异见 `TestSF_Config_TopDefReachable`。

---

## 2. CFG Native 参考手册

本节为 **CFG native 的唯一主参考**： **`getCfg` 与全部 `cfg*`**、`reachabilityGuard` 的职责、一览表、**分项示例**、关联 `dataflow`/`TopDef`、单测与源码位置。**以本节正文与 `sf_native_call.go` 中 `nc_desc`、实现源码为准（重复说明已删）。**

### 调用顺序

1. **`getCfg`**：把任意命中点（调用点、参数 value 等）解析为 **`CfgCtxValue`**（同一函数内的 func / block / inst）。
2. **`cfgDominates` / `cfgPostDominates` / `cfgReachable` / `cfgReachPath`**：在 **链上**对 cfg 调用；通常需要 **`target="$已 getCfg 的变量"`**。
3. **`cfgGuards`**：链上 cfg 表示 **sink 块**；无 config；无命中时 native **报错**。
4. **`cfgCondition` / `cfgConditionValues` / `cfgBlock` / `cfgInst`**：仅依赖当前链上 cfg；其中 `cfgCondition*` 在无可用条件/无 value 时可能 **报错**（见实现）。

### 实现位置（`common/yak/ssaapi`）

| 主题 | 文件 |
|------|------|
| 注册与 `nc_desc` | `sf_native_call.go` |
| `getCfg` 与全部 `cfg*`、`mapCfgCtxValues`、`mapCfgCtxAgainstTarget`、`reachabilityGuard`（mustExecute）及 if 条件常量折叠相关 BFS | `sf_cfg_native.go` |
| 支配 / 后支配（`dominates` / `postDominates` 等） | `sf_cfg_dom.go` |
| 可达性、最短路径串 | `sf_cfg_reach.go` |
| 块条件摘要、`summaryToCondString`、`appendResolvedCondValues` | `sf_cfg_condition.go` |
| Guard 辅助（loop exit/latch 等） | `sf_cfg_support.go` |
| `CfgCtxValue` | `sf_cfgctx_value.go` |
| `GuardPredicateValue` | `sf_cfg_guard_value.go` |

### Native 一览表

| 名称 | 链上角色 | 输出 | Config |
|------|----------|------|--------|
| `getCfg` | 任意命中 value | `CfgCtxValue` | 无 |
| `cfgBlock` | cfg | 字符串 `func=… block=…` | 无 |
| `cfgInst` | cfg | 字符串含 `inst=` | 无 |
| `cfgCondition` | cfg | 条件摘要字符串 | 无 |
| `cfgConditionValues` | cfg | 条件相关 **Value** 列表 | 无 |
| `cfgDominates` | **终点**（从入口走来） | bool | **`target="$…"`** 必填：必经点 |
| `cfgPostDominates` | **起点**（向出口走去） | bool | **`target="$…"`** 必填 |
| `cfgReachable` | 起点 | bool | **`target`**；可选 **`icfg`**、`max_depth`、`max_nodes` |
| `cfgReachPath` | 起点 | 路径串或空 | 同 `cfgReachable` |
| `cfgGuards` | **sink** | `GuardPredicateValue` | 无 |
| `reachabilityGuard` | 管道可有锚点 value；**必填** `target="$…"` | `true` / `false` 或多条条件 SSA value | **`mode=mustExecute`**（当前仅支持） |

**`target`**：`target="$a"` 或 `target="a"`；帧内变量须展开后含至少一个 **`getCfg`** 得到的 cfg。

### 语义备忘（易错）

- **`cfgDominates`**：图论上 **`target` 支配链上「当前点」** → 应写 **`$sinkCfg<cfgDominates(target="$checkCfg")>`**（链上为 sink，target 为检查点）。
- **`cfgPostDominates`**：**`target` 后支配「当前点」**。
- **`cfgReachable` / `cfgReachPath`**：默认过程内；`icfg=true` 启用最小 ICFG（见 `sf_cfg_reach.go` 与 `TestNativeCall_cfgReachable`）。

### 各 Native：文字说明与示例

**公共约定**：`target="$xxxCfg"`（或等价写法）指向**已通过 `getCfg` 绑定**的另一个 cfg。**支配**读作「必经（从入口走来）」，**后支配**读作「通向出口必经之路」；参见上文 **「语义备忘（易错）」**。**不是** SMT 路径可满足性证明。

#### `getCfg`

把任意 SSA 命中点落到**同一函数**的 `(func, block, inst)`；所有 `cfg*` 都链在此之后。

```sf
println(* #-> as $arg)
$arg<getCfg()> as $cfg
```

#### `cfgBlock`

输出 **`func=` / `block=`** 定位串，用于告警 evidence / 快照对齐。

```sf
$d<getCfg()> as $cfg
$cfg<cfgBlock()> as $blk
```

#### `cfgInst`

在 `cfgBlock` 基础上增加 **`inst=`** 粒度（字符串）。

```sf
$d<getCfg()> as $cfg
$cfg<cfgInst()> as $ins
```

#### `cfgCondition`

当前锚点**所在基本块**上的条件摘要（结构化证据；不等于整条路径上条件合取）。

```sf
$sinkCall<getCfg()> as $sinkCfg
$sinkCfg<cfgCondition()> as $condText
```

#### `cfgConditionValues`

该块参与分支条件的 **SSA value 列表**，便于与 dataflow 命中的 `$input`/`$sink` 做集合级对照。

```sf
$sinkCfg<cfgConditionValues()> as $condVals
```

#### `cfgDominates`

- **语义**：链上 cfg 视为从入口走来时的**终点**；为真 ⟺ **`target` 支配**链上终点（必经点）。
- **易错**：不是「当前点支配 `target`」。**must-check**（入口→sink 是否必经检查）应写：`$sinkCfg<cfgDominates(target="$checkCfg")>`。

```sf
$checkCall<getCfg()> as $checkCfg
$sinkCall<getCfg()> as $sinkCfg
$sinkCfg<cfgDominates(target="$checkCfg")> as $mustPassCheck
```

#### `cfgPostDominates`

- **语义**：链上点为**起点**（向任一出口）；为真 ⟺ **`target` 后支配**该起点。
- 典型：汇合前调用点到出口是否必经某块 / sink。

```sf
$callCall<getCfg()> as $callCfg
$sinkCall<getCfg()> as $sinkCfg
$callCfg<cfgPostDominates(target="$sinkCfg")> as $mustAlongAllExitingPaths
```

#### `cfgReachable`

是否存在从链上锚点到 `target` 的**控制流连边**（BFS）。默认**仅过程内**；`icfg=true` 打开最小跨函数边（Call→ callee 入口、callee exit→ caller 后继等，`sf_cfg_reach.go`）。

```sf
$caller<getCfg()> as $callerCfg
$calleeSink<getCfg()> as $sinkCfg
$callerCfg<cfgReachable(target="$sinkCfg")> as $sameFunc
$callerCfg<cfgReachable(target="$sinkCfg", icfg=true, max_depth=4, max_nodes=8000)> as $icfg
```

#### `cfgReachPath`

与 `cfgReachable`**同一套搜索**；不可达为空串；否则为最短 **`name[f=funcId,b=blockId] -> ...`** 证据链。

```sf
$a<getCfg()> as $from
$b<getCfg()> as $to
$from<cfgReachPath(target="$to", icfg=true)> as $path
```

#### `cfgGuards`

- **语义**：枚举相对 **sink 块** 的 **early-return/panic/break/continue** 一类 **if 两分枝**守卫（`GuardPredicateValue`）。
- **前提**：链上 cfg 必须为 **sink 锚点的 `getCfg`**；**无 guard 时 native 报错**（规则需 `?{...}` 容错或前置条件）。

```sf
println(* #-> as $arg)
$arg<getCfg()> as $sinkCfg
$sinkCfg<cfgGuards()> as $guards
```

#### `reachabilityGuard`（`mode=mustExecute`）

在**管道与 `target` 同函数**、且 `target` 能 `getCfg` 的前提下，判断从函数**入口**出发、沿**任意一条能到出口的执行路径**，是否都**必经** `target` 所在块（启发式，**非** SMT）。**跨 Program、跨函数或缺乏入口上下文**时多为 `false`。

```sf
cc as $to
a as $from
$from<reachabilityGuard(target="$to", mode="mustExecute")> as $mustExecuteCC
```

#### `mustExecute` 与可常数折叠的 `if`

若 `if` 条件在 SSA 上可被常数折叠，则 `reachabilityGuard` / `mustExecute` 对「是否必经某块」的判断往往与直觉更一致；不可测条件、`switch`/`loop` 等仍为保守近似，**不是** SAT。

### NativeCall（Go）与 SyntaxFlow（速查）

| 常量（Go） | SyntaxFlow |
|---|---|
| `NativeCall_GetCFG` | `getCfg` |
| `NativeCall_CFGGuards` | `cfgGuards` |
| `NativeCall_CFGDominates` | `cfgDominates` |
| `NativeCall_CFGPostDom` | `cfgPostDominates` |
| `NativeCall_CFGReachable` | `cfgReachable` |
| `NativeCall_CFGReachPath` | `cfgReachPath` |
| `NativeCall_CFGCondition` | `cfgCondition` |
| `NativeCall_CFGConditionValues` | `cfgConditionValues` |
| `NativeCall_CFGBlockInfo` | `cfgBlock` |
| `NativeCall_CFGInstInfo` | `cfgInst` |
| `NativeCall_ReachabilityGuard` | `reachabilityGuard` |

### 与 `dataflow` / TopDef 配套的 CFG（非 `cfg*` 本体）

- **`<dataflow(..., only_reachable="$锚点Cfg")>`**：对 dataflow **结果后置**或对路径枚举邻居做 **`cfgReachable` 裁剪**（`post` vs `path`/`strict`，见引擎实现）。
- **`$rec * #{ include_reachable: \`...\` }`**：TopDef 递归里先按 **`cfgReachable`** 过滤候选再走数据流上行；常与 `$x<getCfg> as $锚点Cfg` 配伍。快照与边界见 `TestSF_Config_TopDefReachable`。

### 单测与文档维护

- `common/yak/ssaapi/test/yak/cfg_native_call_test.go`：**`TestNativeCall_*`**（含 `cfgReach*`、`cfgGuards`、**`TestNativeCall_reachabilityGuard`** 及 if 常量相关子场景）。
- `common/yak/ssaapi/test/syntaxflow/recursive_config_test.go`：**`TestSF_Config_TopDefReachable`**（`include` / `include_reachable` / 组合键）。

语义与参数以 **`sf_native_call.go` 内 `nc_desc`** 与本仓库实现为准；改 native **须同步本文与 `nc_desc`**。

---

## 附录：内置规则改进清单（P0–P2）

本节列出若干与**控制流可达性 / evidence** 强相关的内置 `.sf` 规则，供后续迭代时对照 `cfg*` 与 `only_reachable`。

### P0（立即优化）

- `common/syntaxflow/sfbuildin/buildin/golang/cwe-89-sql-injection/golang-gorm-sql.sf`
  - 主要问题：字符串拼接类路径在不可达分支上仍被计入候选。
  - 优先动作：统一引入 `<dataflow(only_reachable="$sinkCfg", only_reachable_mode="path")>`，并给 ICFG 场景加 `max_depth/max_nodes` 护栏。
  - 预期收益：快速降低“分支不可达但数据流可连通”的误报。

- `common/syntaxflow/sfbuildin/buildin/golang/cwe-79-xss/golang-reflected-xss-beego.sf`
  - 主要问题：`until/exclude` 对 early-return guard 感知弱，容易把“被 guard 截断”的路径报出来。
  - 优先动作：在 sink 证据里补 `getCfg + cfgReachPath + cfgGuards`，先提升可解释性再做强过滤。
  - 预期收益：审计效率显著提升，减少“是否真实可达”的人工争议。

### P1（次优先级）

- `common/syntaxflow/sfbuildin/buildin/golang/cwe-863-incorrect-authorization/golang-filepath-missing-permission-check.sf`
  - 主要问题：`check` 与 `sink` 的控制流关系与数据相关性约束混在一起，容易“看见检查就降级”。
  - 优先动作：拆成两段判定：控制流用 **`$sinkCfg<cfgDominates(target="$checkCfg")>`**（从入口到 sink 是否必经 check）+ 既有 dataflow 相关性（数据流）。
  - 预期收益：减少“检查了别的变量也被当作已校验”的误判。

- `common/syntaxflow/sfbuildin/buildin/golang/cwe-942-cors/golang-beego-cors.sf`
  - 主要问题：source/sink 边界清晰，但对 allowlist/guard 的控制流证据薄弱。
  - 优先动作：为命中结果增加 `cfgBlock/cfgInst/cfgReachPath` 证据字段，形成固定审计模板。
  - 预期收益：降低“规则命中但难复核”的沟通成本。

### P2（治理与体验）

- `common/syntaxflow/sfbuildin/buildin/golang/cwe-1336-ssti/golang-sprig-ssti.sf`
  - 主要问题：模板执行链复杂，当前规则对条件分支与跨函数路径证据不足。
  - 优先动作：先补 evidence 模板（`cfgReachPath + cfgCondition/cfgConditionValues`），再评估是否进入 path 模式强过滤。
  - 预期收益：为后续“结构化 condition 谓词”落地提供样板规则。

### 统一改造规范（建议）

- 所有高风险规则统一使用 `config` 参数格式（`target=.../icfg=.../max_depth=.../max_nodes=...`）。
- 对“是否可达”敏感的规则优先开启 `only_reachable_mode=path`；默认保留 post 作为回退。
- 告警统一包含 `evidence_reach/evidence_path/evidence_block`，避免仅输出命中点字符串。
