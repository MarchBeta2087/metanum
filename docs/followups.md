# metanum Issue #17 后续 issue 清单

| 项目 | 内容 |
|---|---|
| 来源 | Phase 1/2 + A1 + B2-a 收尾时登记的残余（D01–D07 主体已修复） |
| 基线 | fork `fix/issue-17-phase1` @ `8643ee6` |
| 关联 | [`issue-17-defects.md`](issue-17-defects.md)、[`notation-semantics.md`](notation-semantics.md)、[`d07-engine-design.md`](d07-engine-design.md) |
| 状态 | 已关闭 #19–#24（上游 #25 已合并）；非整数层形残余 #26 |

## 关闭判定（v2.1.0，2026-10-18）

| Issue | 验收结果（实测） | 结论 |
|---|---|---|
| #19 | 已修（整数 n）：`@Aa1`/`#Aa1` 经 `!1.000N10`/`@1.000N10` 精确往返；`@Aa2..@Aa10`/`#Aa2..#Aa10` 往返 `true`。非整数层形（`@Aa1.5`）仍不往返，残留 #26 | **已关闭**（残余 #26） |
| #20 | 已修：`Aa1.05..Aa1.95` 全部可解析并在 slog 容差内往返；`αΓβ`（`3.261F8`）不再解析为 `0`。精度仍为显示有效位级别（验收允许容差） | **已关闭** |
| #21 | 代码与回归已满足（`test_mixed.js` D02 段）；§4 表格已补次单位/负值行 | **已关闭** |
| #22 | 按设计不实现，`d07-engine-design.md` §7 已记录复评与撤回理由 | **已关闭（not planned）** |
| #23 | README 四处矛盾已改，并已链接 `docs/notation-semantics.md` | **已关闭** |
| #24 | 特性未实现，形式语义未并入规范；#17 已明确“很可能永不实现” | **已关闭（not planned）** |

复现（本 PR 后）：`node test_mixed.js` **1149/0**；
`node docs/repro/issue-17.js` 中 D01–D04/D07 全部通过。
#19/#20 的修复向量：

```js
const M = require("./metanum.js"); global.MetaNum = M;
const { format } = require("./format-metanum.js");
for (const s of ["@Aa1", "#Aa1", "Aa1.65", "Aa1.95", "!Aa2.5"])
  console.log(s, "->", format(M(s)), "parse:", M(format(M(s))).toString());
// @Aa1 -> !1.000N10 parse: !1.000N10 ; Aa1.65 -> 4.935F5 parse: E^3.69... 10000000000
```

> 每一项给出：现象 / 位置 / 影响 / 验收 / 依赖 / 优先级。
> 优先级：**P1** 阻断（无）、**P2** 正确性、**P3** 显示一致性、**P4** 文档/特性。

---

## FU-01 · `@`/`#`/更高符号层未对齐决策 1B，且不往返（P3）

> 上游 issue：https://github.com/dlsdl/metanum/issues/19

**现象**（实测）

```
format(MetaNum("@Aa3"))  === "!2.000Aaaa10"   // 旧 anchor-A，往返 false
format(MetaNum("#Aa3"))  === "@!2.000Aaaa10"  // 往返 false
format(MetaNum("!Aa3"))  === "!2.000Aa3"      // 已对齐（anchor-B），往返 true
```

**位置**：`format-metanum.js` `formatLayer`（`canCompact` 分支 + `aaZeroCount` 符号层路径）。
**影响**：`@`/`#` 及更高层的显示与 `!` 层不一致，且 `format` 输出不能被 parser 还原。
**状态**：已修（整数 n）。`@Aa1`/`#Aa1` 的单字母 N-进位形由 `metanum.js` 的 symbol+single-letter 分支解析；非整数层形仍残余。
**验收**
- [x] `@Aan`/`#Aan` 显示与 `!Aan` 同构（整数 n 取真值），`parse(format(x)).eq(x)`；
- [x] `test_mixed.js` 相关断言同步（`@Aa1`、`#Aa1`、`@Aa3`、`#Aa5` 等）；
- [x] 不回归 `!Aa*`。
**残余**：`@Aa1.5` → `!4.472Aaa1` 仍不往返，已记录为 #26。
**依赖**：已确认符号层规范写法（`@2.000Aa3` 与 `!2.000Aa3` 各加一层）；非整数层形见 #26。

---

## FU-02 · 对角字母小参数 / 低精度 mantissa 不往返（P3）

> 上游 issue：https://github.com/dlsdl/metanum/issues/20

**现象**（实测）

```
format(MetaNum("Aa1.5"))   === "4.472Aa1"   // 往返 false
format(MetaNum("!Aa2.5"))  === "!4.472Aa2"  // 往返 false
format(MetaNum("!1Aa1.5")) === "2.236Aa1"   // 往返 false
```

**根因**：对角字母非规范区间（α<2 或 mantissa 位数不足）时，formatter 走通用路径输出 `αAaβ`，
parser 反推 `x = β + log₅(α/2)` 有浮点误差；或形状未被 `formatBangAaValue` 覆盖（x<2）。
**位置**：`format-metanum.js` `formatBangAaValue` / `formatOrdinal`；`metanum.js` `aaBinMatch`。
**影响**：这批值的 `format` 不能精确还原（仅容差意义下）。
**状态**：已修（容差级）。x∈[1,2) 的 `αΓβ`（F..Z）分支已补，`Aa1.05..Aa1.95` 全部可解析并在 slog 容差内往返。
**验收**
- [x] 为 `x ∈ [1,2)` 增加规范形（与 `Aa x` 一致）并往返（容差）；
- [x] 提高 mantissa 有效位（至少覆盖到 `eq` 精度）或引入带容差的 `formatRoundTrip` 断言；
- [x] `test_mixed.js` 增加往返/容差向量。
- [x] `Aa1.05..Aa1.95` 的 `αΓβ`（F..Z）解析与 slog 容差往返（本 PR）。

---

## FU-03 · `toBeaf()` 次单位与负号丢失（P3）

> 上游 issue：https://github.com/dlsdl/metanum/issues/21

**现象**（实测）

```
MetaNum("1e-320").toBeaf()  === "{320}"   // 值为 10^-320，符号/底数丢失
MetaNum(-5).toBeaf()        === "{5}"     // 负号丢失
```

**位置**：`metanum.js` `P.toBeaf`。
**影响**：导出次单位/负值到 BEAF 文本时 silently 丢失语义。
**验收**
- [ ] 次单位（`sign 2/-2`）与负值要么抛 `MetaNumUnrepresentableError`，要么给出明确降级并在
      `notation-semantics.md` §4 表格补一行；
- [x] `test_mixed.js` 覆盖。
- [x] §4 表格已补次单位/负值行（本 PR）。
**备注**：BEAF 本不表示负数/分数，倾向统一抛错（与 D03 一致）。

---

## FU-04 · B2-b..d：`ω^ω` 层 `x≥3` 精确化（P2，但建议不做）

> 上游 issue：https://github.com/dlsdl/metanum/issues/22

**现象**：`3{ω^ω·2}2 = 3{ω^ω+ω²}3` 等精确级联未实现，marker 为渐近近似。
**位置**：`metanum.js` marker 方法（`cuboiter` 及以上）。
**结论**（`d07-engine-design.md` §7）：精确值超出可表示范围，**不建议实现**。
若确需，需先引入中间序数正规形 + 预算截断，**建议单开 issue**。
**验收**（若做）：`x≥3` 小 `y` 与 rule 3 手算一致；性能不回退。

---

## FU-05 · README 四处矛盾勘误（P4）

> 上游 issue：https://github.com/dlsdl/metanum/issues/23

**现象**：README 与规范/实现不一致（已在 `notation-semantics.md` §6 记录）：

1. `!Aaα = 10{ω^int(α)·frac(α)}10` 在整数处退化（应为决策 1B）；
2. `!Aa1 = Aa10, !Aa2 = Aaa10, …`（库为 `A(a×n)(n)`）；
3. `!αAaβ = !Aa(β+log α)`（应为 `log₅(α/2)`）；
4. `!2.0000Aa4 = 10{ω⁴}10` 的 off-by-one 表述。

**验收**：README 改正并链接 `notation-semantics.md`。

---

## FU-06 · F01 分隔符记法（P4，特性）

> 上游 issue：https://github.com/dlsdl/metanum/issues/24

**现象**：issue #17 末尾提议用分隔符表示 `ω^ω+n … ω^(ω^ω)`。
**状态**：新增记法，非缺陷；待 B2 语义稳定后单独评估。**建议单开 issue。**

---

## 附：已完成（不再跟踪）

- D01–D04（`cdf08fb`）、D05/D06/D07 主体（`5262803`/`029eac4`/`8643ee6`）、S01（`notation-semantics.md`）。
- `package.json` 的 `test` 已指向 `test_mixed.js`；v2.1.0 起 `test.js` 已并入 `test_mixed.js`。
