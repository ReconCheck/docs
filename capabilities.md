# ReconCheck 功能能力清单

> 与 [core](https://github.com/ReconCheck/core) 仓库同步更新。本文件回答「具体能实现哪些功能、在哪儿用、怎么用」。状态标注：✅ 已实现 · 🧭 设计已定、待实现 · 🕔 后续里程碑。

---

## 一、引擎层（CLI / Python API）

| 能力 | 状态 | 说明 |
|---|---|---|
| 表格解析 CSV / TSV / TXT | ✅ | 自动嗅探分隔符（`,` `;` `\t` `\|`），宽行自动补列名 |
| 表格解析 XLSX / XLSM | ✅ | 只读流式加载（内存恒定），多工作表可选，`data_only` 取值 |
| PDF 解析（文本层） | ✅ | 线框表格优先 + 无框版面兜底（按坐标聚类行列）；**扫描件（无文本层）默认报「需 OCR」明确错误**（不产出垃圾表）；装 `pdf-ocr` 扩展并设 `RECONCHECK_OCR=1` 后按行 OCR（Tesseract） |
| 编码识别 | ✅ | UTF-8（含 BOM）/ GB18030 / UTF-16 / Latin-1；**乱码置信度门槛**：二进制垃圾抛明确错误而非产出垃圾表 |
| 行对齐 | ✅ | 按 `match_on` 键列精确匹配，键归一化（料号 `A-012` → `a12`、实体后缀剥离、空白），单位保留在单元格 |
| 内置自动规则 | ✅ | 任意两侧共有数值列参与比对，容差 相对 0.001 / 绝对 0.01；**不对显式规则已覆盖的列重复报**（列级去重） |
| YAML 差分规则 | ✅ | `match_on` / `compare` / `severity` / `tolerance`(相对+绝对) / `exceptions` / `evidence.require: both_sides` |
| 例外（exceptions） | ✅ | 单位换算 `kg↔g`、四舍五入、**日期容差 `dates_within`**（超差仍报差异）、**忽略大小写文本相等**（真不一致仍报差异） |
| **三单核对 compare3** | ✅ | 采购链路（PO/送货/发票）与**销售链路（SO/出库/销项发票）**通用：逐对两两报告 + 按键/字段的 consensus/outlier 判定（数值按容差与单位对齐，键支持归一化） |
| 证据链 | ✅ | 每条 finding 双侧坐标 + `cell://` href（PDF 证据带页码）；报告 JSON 为稳定契约 |
| 报告告警 | ✅ | `warnings[]` 显式声明非致命问题（如**重复键 first-wins**：只有每键首行参与比对）；`documents[].kind` 标注单据种类（po/so/outbound/invoice/delivery） |
| CLI | ✅ | `reconcheck compare` / `compare3`，缺文件/非法输入干净报错（非 traceback） |

使用示例：

```bash
reconcheck compare examples/po.csv examples/invoice.csv \
  --rules examples/rules --match-on 料号 --normalize 料号:part_no --output report.json
```

## 二、Web API（FastAPI，默认 http://127.0.0.1:8765）

| 端点 | 状态 | 用途 |
|---|---|---|
| `GET /api/health` | ✅ | 存活 + 引擎版本 |
| `POST /api/compare` | ✅ | 同步对比（2 文件或 2 `doc_ids`），直接返回报告 |
| `POST /api/compare3` | ✅ | 同步三方核对：3 文件或 3 `doc_ids`，返回逐对报告 + consensus/outlier 判定 |
| `POST /api/jobs` | ✅ | **异步批处理**：批量上传文件 + 文档库引用，按文件名业务键自动配对，后台 worker 逐对执行 |
| `GET /api/jobs/{id}` | ✅ | 状态 / 进度 / 逐对结果；**败对可见**（不拖垮整批，带失败原因） |
| `GET /api/reports/{id}` | ✅ | 已归档报告 JSON |
| `GET/POST /api/documents` | ✅ | 文档库：列表 / 注册上传文件 |
| `GET /api/documents/{id}` | ✅ | 元信息 + 解析后表格预览（前 100 行） |
| `GET /api/documents/{id}/content` · `DELETE` | ✅ | 原始文件下载 / 删除 |
| `GET/POST/PUT/DELETE /api/datasources` | ✅ | 企业数据源（自定义 Web API）配置 |
| `POST /api/datasources/{id}/probe` | ✅ | 连通性测试（**非 2xx 视为失败**，含状态码/字节数） |
| `POST /api/datasources/{id}/list` | ✅ | 列出可选择的单据/记录 |
| `POST /api/datasources/{id}/fetch` | ✅ | 拉取文件流或 records JSON 至文档库 |

### 批处理语义

- 文件名中的单据种类词（采购：`po`/`order`/`采购`、`dn`/`送货`、`inv`/`发票`；销售：`so`/`sales`/`销售`、`out`/`outbound`/`出库`、`siv`/`销项`）被剥离，剩余业务编号相同的一组自动成对（如采购 `PO-240913-001` ↔ `INV-240913-001`，销售 `SO-240913-001` ↔ `SIV-240913-001`）；独立出现的文件进 unpaired 列表。
- 同 `doc_id` 重复引用、同内容重复上传会被去重（不产生无意义自比对）。
- 同一业务编号 **≥3 份**时，worker 在两两比对之外自动生成**三方核对报告**（consensus/outlier）；前端按组聚合为一张卡片（每份文件只出现一次），组内展开两两明细 + 三方表。
- 单组配对失败不影响同批其他对。

### 数据源能力

- **type=file**：端点返回文档字节流，URL 支持 `{id}` 占位；`list_url` 可选提供可选清单。
- **type=records**：端点返回 JSON，`records_path` 选择数组（如 `data.items`），自动转成表格（键并集为表头、一条记录一行）。
- 鉴权：`none` / `bearer` / 自定义 `header`（`header_name`）；令牌 API 只回显 `has_token`。落盘默认明文；配置 `RECONCHECK_DATA_KEY`（`crypto` 扩展）后 Fernet 加密存储。
- 拉取在上游是**服务器侧请求**（无浏览器 CORS 问题），响应**流式 50MB 上限**，超限即中断报错。

## 三、前端（无依赖静态页，`/` 挂载）

- ✅ 拖拽/选择批量上传（重复文件防抖），自动配对分组
- ✅ severity 汇总（高/中/低计数），点击 finding 的 `cell://` 证据**高亮原文单元格**
- ✅ 结果页渲染败对与原因；单个报告拉取失败不拖垮整页
- ✅ 文档库浏览/预览/加入比对队列（按 id 防重）
- ✅ 数据源配置表单：新建/编辑/探测/列出条目/拉取入库
- ✅ 中文错误提示（未配对、空文件、超限、文档被删等）

## 四、健壮性与安全（已实现）

| 能力 | 说明 |
|---|---|
| 上传上限 | 单文件 64MB，超限 413 |
| 拉取上限 | 数据源响应 50MB，流式中断（非下载后检查） |
| 路径穿越防护 | 文档/报告 id 强制 12 位十六进制白名单，`DELETE`/下载/报告读取均校验 |
| 可选鉴权 | `RECONCHECK_API_KEY` 开启全局 `X-API-Key` 校验；未配置时**启动打印告警** |
| 持久化 | 数据目录（`RECONCHECK_DATA`）持有 jobs/reports/documents/datasources；jobs.json 原子写入 |
| 重启恢复 | queued 任务重启后自动重放；TTL janitor 每小时清理超龄文件（`RECONCHECK_TTL_DAYS`，默认 30 天，在途任务不删） |
| 失败隔离 | 单对解析失败仅该对 failed（带原因），不中断同批 |
| 类型安全 | 配置校验、token 空值保留旧值、`X-API-Key` 头注入防护 |

## 五、只读审计点

ReconCheck 定位为**只读**：不写业务系统、不改源文件；对上游数据源的唯一写入是配置的令牌与拉取副本，均落在本地数据目录。

---

## 设计与规划

| 项目 | 状态 | 内容 |
|---|---|---|
| 跨引擎「作业层」定位 | 🧭 | 见 [job-layer.md](./job-layer.md)：把批处理/配对/报告抽象为引擎无关的作业层，未来可接其他验证引擎 |
| LLM 参与 | 🧭 | opt-in `LLMEnhancer`（接口已预留在 `reconcheck/llm`）：对齐消歧 + 差异解释；仅 OpenAI 兼容端点、超时/预算/失败回退到确定性结果（报告契约暂不含 LLM 标记字段，接入时以增量字段扩展） |
| 扫描件 OCR | ✅ | 无文本层 PDF 可选走 Tesseract（`pip install "reconcheck[pdf-ocr]"` + `RECONCHECK_OCR=1`）：按行输出文本；未安装/未开启时仍是明确报错。版面还原、多栏表格属后续规划 |
| 实体解析 | 🕔 | `华加` ↔ `深圳市华加生物科技有限公司` 的真对齐（当前为后缀/空白归一，三单键支持 part_no/entity 归一） |
| 规则 v1 稳定 | 🕔 | 例外目录扩充（日期/文本已入库）、evidence 输出格式冻结 |
| 分布式队列 | 🕔 | 当前单进程 worker + 文件持久化，无消息队列 |