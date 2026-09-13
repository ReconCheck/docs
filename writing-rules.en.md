# Writing differential rules — guide

> The engine-side rule format is described in core `DESIGN.md`; this page is
> the practical guide: where rules live, every field, exceptions, the auto
> rule, and a debugging checklist. Rule packages and commercial agreements
> live in the private [rules](https://github.com/ReconCheck/rules) repo.

## 1. Where rules go and how they load

- **CLI**: `reconcheck compare A B --rules <file-or-dir>`; a directory loads
  every `*.yaml` it contains.
- **Web / API**: `examples/rules/` by default (`base.yaml` + `unit.yaml`); the
  built-in auto rule always backs it up (see §4).
- One YAML file may hold a single rule object or a list under `rules:`.

## 2. Field reference

```yaml
id: po-invoice-qty-mismatch      # required, unique; used in reports
applies_to: [purchase_order, invoice]  # informational for now
severity: high                   # high | medium | low
match_on: [料号]                 # both sides must carry these headers
compare: 数量                    # one header, or a list
tolerance:
  relative: 0      # |l - r| <= absolute + relative * max(|l|, |r|, 1)
  absolute: 0
exceptions:                      # any match -> not a finding
  - when:
      unit_conversion_between: [kg, g]
  - when:
      rounding: { decimals: 2 }
evidence:
  require: both_sides            # skip a finding that cannot cite both cells (recommended)
```

- `tolerance` formula: `|l − r| <= absolute + relative × max(|l|, |r|, 1)`.
  Relative suits amounts; absolute suits counts.
- Omitted `tolerance` → default `relative 0.001 / absolute 0.01`.
- `compare` accepts a list: `compare: [数量, 金额]` judges both columns with
  this rule.

## 3. Exceptions: the soul of a rule

Without exemptions every legitimate difference becomes an alert and users stop
looking. Implemented kinds:

| Kind | Example | Effect |
|---|---|---|
| Unit conversion | `when: unit_conversion_between: [kg, g]` | equal after unit conversion → exempt (any pair convertible via the unit table) |
| Rounding | `when: rounding: { decimals: 2 }` | equal at 2 decimals → exempt (FX tail diffs, unit prices) |
| Date tolerance | `when: dates_within: { days: 3 }` | both sides parse as dates and |Δ| ≤ N days → exempt; **a real date gap is still reported** (ISO, slash, dotted, compact, Chinese formats) |
| Case-insensitive text equality | `when: text_equal_ignore_case: true` | texts equal ignoring case → exempt; **a real difference (OK vs OPEN) is still reported** |

Any matched exception exempts; stack several under one rule. The date/text
exceptions let **non-numeric columns** produce findings (see §5, item 6).

## 4. Built-in auto rule (auto-numeric-diff)

- With no explicit rules, the engine compares **every shared numeric column**
  (tolerance relative 0.001 / absolute 0.01, severity medium).
- Alongside explicit rules it only fills columns no applying explicit rule
  covers — it never double-reports e.g. `数量` when a rule already judges it.
- It is a usability baseline, not a substitute for purpose-built rules.

## 5. Debugging: why nothing / why false positives

1. **Rule not applying** — `match_on` headers must exist on BOTH sides with
   identical names (full/half-width, spaces). Check `GET /api/documents/{id}`
   for the real headers.
2. **Column not judged** — `compare` values must coerce to numbers (`5 kg`
   works, prose doesn't). To judge non-numeric columns (dates, status), add a
   `dates_within` / `text_equal_ignore_case` exception.
3. **Tolerance too wide** — the diff sits under `absolute + relative×max(...)`.
4. **An exception fired** — unit conversion, rounding, date tolerance, or
   case-insensitive equality. Clear the exceptions temporarily to confirm.
5. **Evidence missing** — `evidence.require: both_sides` and one side lacks
   coordinates (e.g. merged cells) → skipped.
6. **Date/text differences silent** — non-numeric columns are not judged by
   default; a rule must declare `dates_within` / `text_equal_ignore_case` and
   both values must parse for differences to surface.

**Regression habit**: every rule package ships `fixtures/` (sanitised pairs +
expected findings/severity) and the engine result is diffed against it —
exactly how the core repo's `tests/` works.

## 6. A runnable example (ships in examples/rules/unit.yaml)

```yaml
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