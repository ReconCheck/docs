# ReconCheck 用户使用指南

适用对象分三类：**对账/财务操作员**（网页操作）、**企业系统对接**（REST API）、**开发者**（CLI/嵌入）。本指南从零开始。

> 边界（先说清楚）：支持**表格类文件（CSV/TSV/XLSX）与 PDF 文本层**；**扫描件（无文本层的 PDF）与图片**可选走 OCR（装 `pdf-ocr` 扩展并设 `RECONCHECK_OCR=1`；未开启时明确报错）。文件名里需要包含**相同的业务编号**才能自动配对（见 FAQ）。

---

## 一、操作员：网页使用（5 分钟上手）

### 1. 启动服务

```bash
# 有 Python 环境的机器上（安装见 core README）
reconcheck-api
```

浏览器打开 `http://127.0.0.1:8765`。

### 2. 上传一批单据

- 把同一笔业务的几份文件拖进页面（如 `PO-240913-001.csv` 与 `INV-240913-001.csv`）。
- 页面会**自动拆掉单据种类词**（po/订单 与 inv/发票 等），按剩余业务编号 `240913001` 分组成对。
- 组下面会显示自动生成的对比对；只出现一次的文件会进「未配对」列表（常见原因：文件名没有共同编号）。

**三方核对（采购与销售链路通用）**：采购三单（PO+送货+发票）或销售三单（SO+出库+销项发票）一起核对时，可用 `reconcheck compare3` 或 `POST /api/compare3`（3 个文件或 3 个 `doc_ids`）——返回三对两两报告 + 「哪一方是偏离方」的判定（如 发票 vs 订单+送货单），适合月底对账一票到底。

### 3. 提交并查看结果

- 点「运行」。每对会独立执行，进度逐对显示。
- 结果页：顶部是**高/中/低差异计数 + 单据组数**；同一业务编号的文件聚合为一张**组卡片**（每份文件只出现一次，不再左右重复）——展开后先看**三方核对表**（组内 ≥3 份自动生成：字段 × 各单据值，离群值红色高亮），再看组内两两明细。
- 点差异的**证据链接**，页面会跳到原文表格并**高亮那个单元格**——这是核对的核心：每条判定都能指着原文复核。

### 4. 失败的对比对

某对解析失败（空文件、内容不是文本等）会标记为「失败」并显示原因，**不影响同一批其他对**。

### 5. 文档库

- 顶部可进入「文档库」：上传文件后能看到解析预览；可从文档库把单据加入比对队列（同一文档不会重复加入）。
- 支持下载原文件、删除。

### 6. 数据源（接入企业系统）

「数据源」页新增一个**自定义 Web API**：

| 表单字段 | 含义 |
|---|---|
| 类型 | `file`（接口直接返回文档字节流）或 `records`（返回 JSON 记录数组） |
| URL | 接口地址；`file` 型可含 `{id}` 占位符 |
| 鉴权 | 无 / Bearer Token / 自定义头（`header_name` 填头名，token 填密钥） |
| records_path | JSON 里的数组路径，如 `data.items` |
| id_field / name_field | 用于下拉选择的记录标识与显示名 |
| list_url | 可选：提供可选清单的接口 |

填完点「测试」验证连通（非 2xx 会标红）；「列出」看可选记录；「拉取」把选中的文档放进文档库，之后正常参与比对。

---

## 二、企业系统对接：REST API

服务（默认 `http://127.0.0.1:8765`）提供文档库、异步批处理、报告归档三组接口，完整端点表见 core README。

```bash
# 同步对比两份文件（拿完整报告 JSON）
curl -X POST http://127.0.0.1:8765/api/compare \
  -F "files=@po.csv" -F "files=@invoice.csv"

# 三方核对：采购订单 + 送货单 + 发票
curl -X POST http://127.0.0.1:8765/api/compare3 \
  -F "files=@PO-001.csv" -F "files=@DN-001.csv" -F "files=@INV-001.csv"

# 异步批处理：上传任意多份文件，自动配对，后台执行
curl -X POST http://127.0.0.1:8765/api/jobs \
  -F "files=@PO-240913-001.csv" -F "files=@INV-240913-001.csv"
# → 轮询 GET /api/jobs/{id} 直到 status=done，逐对拿 findings/report_id
# → GET /api/reports/{report_id} 取完整报告

# 企业数据源：配置 → 探测 → 拉取 → 入库
curl -X POST http://127.0.0.1:8765/api/datasources \
  -H "Content-Type: application/json" \
  -d '{"name":"ERP-PO","type":"records","url":"https://erp/api/orders","records_path":"data","auth":"bearer","token":"xxx"}'
curl -X POST http://127.0.0.1:8765/api/datasources/<id>/fetch
curl -X POST http://127.0.0.1:8765/api/jobs -d "doc_ids=<doc_a>,<doc_b>"
```

- 拉取是**服务器侧请求**，无浏览器 CORS 问题。
- 报告 JSON 是契约：每条 finding 带 `evidence`（双侧坐标 + `cell://` href）。

### 权限

- 设置了环境变量 `RECONCHECK_API_KEY` 后，所有 `/api/*` 请求都需带 `X-API-Key` 头；**未设置时服务无鉴权**（启动会有告警，请勿暴露到不可信网络）。
- **网页上输入密钥**：页面里按 `Ctrl+K` 会弹出密钥输入框（浏览器本地保存，每个 `/api` 请求自动带上；清除密钥按 `Ctrl+K` 留空即可）。密钥错误时页面会提示并再次询问。

---

## 三、开发者：CLI / 嵌入

```bash
# 直接对比两份文件，输出报告 JSON
reconcheck compare examples/po.csv examples/invoice.csv \
  --rules examples/rules --match-on 料号 --normalize 料号:part_no --output report.json

# Python 嵌入
from reconcheck.comparison import compare_files
report, findings = compare_files("a.csv", "b.csv", match_on=["料号"])
```

规则文件（YAML）说明见 core 的 `DESIGN.md`；判定逻辑要点：

- 无规则时用内置自动规则（任意双侧数值列，容差 相对 0.001 / 绝对 0.01）。
- 规则可声明 `tolerance`、`exceptions`（单位换算 kg↔g、四舍五入）、`severity`、`evidence.require`。
- 自动规则**不会重复报**显式规则已覆盖的列。

---

## 四、常见问题 FAQ

**Q：为什么报「没有可配对的单据」？**
文件名剥离单据种类词后的业务编号不一致，或同组不足两份。命名示例：采购 `PO-240913-001` ↔ `INV-240913-001`、销售 `SO-240913-001` ↔ `SIV-240913-001`（`out`/`outbound`/`出库`、`siv`/`销项`、`dn`/`送货`、`inv`/`发票` 等种类词都会剥离）；`a.csv` ↔ `b.csv` 不能。

**Q：为什么结果是 0 差异？**
可能是真的无差异；也可能是两侧数值列未被显式规则覆盖但列名不同、或值本身非数值（文本）。自动规则只比对数值得出的共有列。

**Q：空文件 / 乱码文件会怎样？**
空文件 → 该对比对标记失败并提示；二进制垃圾 → 明确报「不是可读文本」（不会产出垃圾表格）。

**Q：文件大小限制？**
上传单文件 ≤ 64 MB；数据源拉取响应 ≤ 50 MB（流式截断）。超限会明确报错。

**Q：PDF 支持吗？**
支持有**文本层**的 PDF（导出/电子签章的单据）：线框表格优先，无框版面按坐标聚类兜底；**扫描件**（纯图片 PDF）默认报「需 OCR」——安装 `pip install "reconcheck[pdf-ocr]"` 并设 `RECONCHECK_OCR=1`（Tesseract 不在 PATH 时配 `RECONCHECK_TESSERACT_CMD`）即可启用按行 OCR。

**Q：中文文件名可以吗？**
可以解析；但要配对，文件名需含**相同的业务编号**（仅中文如「订单明细.csv」无法自动分组）。

**Q：上传的文件/报告会保留多久？**
默认 30 天（`RECONCHECK_TTL_DAYS` 可调），后台每小时清理，执行中的任务不受影响。

**Q：数据会不会泄露到外部？**
默认全部在本机处理；只有配置了数据源（拉取）或开启 LLM 参与（规划中）才会访问你明确指定的外部端点。

**Q：鉴权怎么开？**
`RECONCHECK_API_KEY=你的密钥` 再启动服务即可；接口需带 `X-API-Key` 头。