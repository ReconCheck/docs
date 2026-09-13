# 差异判定规则编写指南

> 引擎侧规则格式已在 core `DESIGN.md` 描述；本文给**实际操作指引**：怎么写、怎么调、怎么排查。规则本体与私有规则库的商业化约定见 [rules](https://github.com/ReconCheck/rules) 仓库（私有）。

## 1. 规则放哪、怎么加载

- **CLI**：`reconcheck compare A B --rules <文件或目录>`。目录会加载其中所有 `*.yaml`。
- **Web / API**：默认加载 `examples/rules/`（`base.yaml` + `unit.yaml`）；内置自动规则始终兜底（见 §4）。
- 一份规则文件里可以放多条规则（一个 YAML 文档对象含 `rules:` 列表，或每条规则一个文件——两种都支持，示例用单文件单规则）。

## 2. 字段逐项说明

```yaml
id: po-invoice-qty-mismatch      # 必填，全局唯一，报告里用它
applies_to: [purchase_order, invoice]  # 目前仅作信息说明，不参与筛选
severity: high                   # high | medium | low（三档）
match_on: [料号]                 # 双侧都必须有这个表头才触发规则（见 §5 排查）
compare: 数量                    # 单个表头，或一个列表
tolerance:
  relative: 0      # |l - r| <= absolute + relative * max(|l|, |r|, 1)
  absolute: 0
exceptions:                      # 任一命中 → 不算差异
  - when:
      unit_conversion_between: [kg, g]
  - when:
      rounding: { decimals: 2 }
evidence:
  require: both_sides            # 引不到任一原文单元格就不报（默认建议开）
```

- `tolerance` 公式：`|l − r| <= absolute + relative × max(|l|, |r|, 1)`。相对容差按两侧较大值缩放，适合金额；绝对容差适合数量。
- 不写 `tolerance` 时引擎用默认 `relative 0.001 / absolute 0.01`。
- `compare` 支持列表：`compare: [数量, 金额]` 表示两条都按本规则判定（同一规则覆盖多列）。

## 3. exceptions：规则的灵魂

没有例外目录，引擎会把每一处合理差异都报成异常，用户三天就不看了。当前已实现的例外类型：

| 类型 | 示例 | 效果 |
|---|---|---|
| 单位换算 | `when: unit_conversion_between: [kg, g]` | 两侧数值按单位换算后相等 → 不报（也支持 `[件, 套]` 等任意可换算对，按 1:N 换算表） |
| 四舍五入 | `when: rounding: { decimals: 2 }` | 两侧保留 2 位小数相等 → 不报（适合汇率/单价尾差） |

例外是**并列命中任一即放行**；可以在一条规则下配多个例外。

## 4. 内置自动规则（auto-numeric-diff）

- 无任何显式规则时，引擎比较**所有双侧共有的数值列**（任一列有数值即参与），容差 相对 0.001 / 绝对 0.01，severity 默认 medium。
- 与显式规则共存时，自动规则**只补显式规则没覆盖的列**——例如你有 `数量`/`金额` 的显式规则，表里还有一列 `重量` 没被规则覆盖，自动规则会报它的差异，但不会对 `数量` 重复报。
- 这一兜底让"没配规则的表"也能出结果，属于可用性设计，不替代正式规则。

## 5. 排查：为什么没报 / 为什么误报

按顺序检查：

1. **规则没触发**：`match_on` 的表头双侧必须都存在，且列名完全一致（含全角/半角、空格）。先用 `GET /api/documents/{id}` 看预览确认表头长什么样。
2. **列没参与判定**：`compare` 列值必须能被转成数值（`5 kg` 可以，`文本描述` 不行）；非数值两侧会被自动规则跳过。
3. **容差太宽**：`|l−r|` 没超过 `absolute + relative×max(...)`。
4. **例外命中**：单位换算或四舍五入后相等。测试时把例外临时清掉看是否恢复报错。
5. **证据缺失**：`evidence.require: both_sides` 下若某侧单元格解析不到坐标（例如合并单元格），该条被跳过。

**回归习惯**：每个规则包配一份 `fixtures/`（脱敏对 + 期望 findings 数/字段/严重度），引擎跑完与期望比对——这正是核心仓库 `tests/` 的做法。

## 6. 一条能跑的完整示例

```yaml
# examples/rules/unit.yaml（仓库自带）
id: po-invoice-qty-unit-agnostic
applies_to: [purchase_order, invoice]
severity: high
match_on: [料号]
compare: 数量
tolerance:
  relative: 0
  absolute: 0
exceptions:
  - when:
      unit_conversion_between: [kg, g]
evidence:
  require: both_sides
```

```bash
reconcheck compare examples/po.csv examples/invoice.csv \
  --rules examples/rules --match-on 料号 --normalize 料号:part_no --output report.json
```