# 接入企业系统：完整示例

> 目标读者：把 ReconCheck 接进 ERP / 财务中台的工程师。三种数据形态对应三套做法；本文给**可直接照抄的配置与调用**，含常见错误码语义。

## 1. 先明确要接什么

| 你的系统能提供 | 用 ReconcCheck 的类型 | 入口 |
|---|---|---|
| 单据文件下载（CSV/XLSX/PDF） | `type=file` | `/api/datasources/{id}/fetch` → 文档库 |
| JSON 记录数组（明细行） | `type=records` | 同上，自动转表格 |
| 两者都要、还想让用户挑选 | 任一类型 + `list_url` | `/api/datasources/{id}/list` |

## 2. records 型：明细 JSON 变表格

假设 ERP 有 `GET https://erp.internal/api/orders?order_no=PO-240913-001` 返回：

```json
{
  "code": 0,
  "data": {
    "header": { "order_no": "PO-240913-001" },
    "items": [
      { "line_no": 1, "part_no": "A0012", "qty": 100, "amount": 500.00 },
      { "line_no": 2, "part_no": "B0034", "qty": 200, "amount": 100.00 }
    ]
  }
}
```

配置（`POST /api/datasources`）：

```json
{
  "name": "ERP-PO-明细",
  "type": "records",
  "url": "https://erp.internal/api/orders?order_no=PO-240913-001",
  "auth": "header",
  "header_name": "X-ERP-TOKEN",
  "token": "放服务端令牌",
  "records_path": "data.items",
  "id_field": "line_no",
  "name_field": "part_no",
  "list_url": "https://erp.internal/api/orders/list"
}
```

- `records_path` 用点路径进入数组：`data.items`；缺失或不是数组 → fetch 返回 422 并提示检查。
- `id_field`/`name_field` 驱动前端下拉（`/list` 会按它们生成 `{id, name}` 列表）。
- `list_url` 返回的 JSON 也应含 `records_path` 可定位的数组；省略 `list_url` 时 `/list` 直接对 `url` 拉全量数组。

拉取入库并参与批处理：

```bash
curl -X POST .../api/datasources/<id>/fetch                                # → 文档库 doc_id
curl -X POST .../api/jobs -d "doc_ids=<po_doc>,<inv_doc>"                  # 引用文档库对比
curl -X POST .../api/jobs -F "files=@附件.csv"                              # 或混合上传
```

## 3. file 型：附件下载流

接口按 id 返回文件字节流（如 `GET /api/attachments/{id}`），且有一个列表接口提供可选单据：

```json
{
  "name": "ERP-附件",
  "type": "file",
  "url": "https://erp.internal/api/attachments/{id}",
  "auth": "bearer",
  "token": "...",
  "list_url": "https://erp.internal/api/attachments/list"
}
```

`/list` 返回 `[{"id": "1", "name": "PO-240913-001.csv"}, ...]` → 前端选择后 fetch 会把 `{id}` 替换为所选 id。没有 `{id}` 时 fetch 直接拉 `url` 本体。

## 4. 鉴权三种任选

| auth | 效果 | 示例头 |
|---|---|---|
| `none` | 无 | — |
| `bearer` | `Authorization: Bearer <token>` | 标准 OAuth2 即用 |
| `header` | `<header_name>: <token>` | 内部网关常见（如上 `X-ERP-TOKEN`） |

- `PUT /api/datasources/{id}` 更新时留空 `token` = 保留原密钥不清空；`GET` 永远只回 `has_token`，不泄露明文。
- 令牌落盘默认明文；设 `RECONCHECK_DATA_KEY`（安装 `crypto` 扩展）后以 Fernet 密文写入 `datasources/*.json`，读取时自动解密。
- 探测（`/probe`）会真实请求端点：连接失败/超时 → `ok=false`；**非 2xx 也判失败**（如 401 说明 token 不对，不要误以为通了）。

## 5. 错误码语义（对接时对照）

| 状态码 | 场景 |
|---|---|
| `400` | 参数冲突（如 files 与 doc_ids 混用）、配置缺 name/url |
| `404` | doc_id / datasource / report 不存在（id 非法或已被删） |
| `413` | 上传超过 64MB |
| `422` | 无配对/单文档、空上传、records_path 无记录、文档不可解析（内容非文本等） |
| `502` | 数据源拉取失败（连接、超时、非 2xx、超 50MB） |
| `401` | 设置了 `RECONCHECK_API_KEY` 但没带 / 带错 `X-API-Key` |

## 6. 最佳实践与边界

- **字段命名稳定**：`records_path`/`id_field` 依赖后端字段名；字段改名 = 数据源同步改配置。
- **分页**：v0 不做自动分页；`list_url` 应由业务侧返回完整列表（或适配成本端分页后再接入）。
- **大响应**：records 端点别一次吐几十 MB；拉取上限 50MB 流式中断，超出即 502。
- **SSRF 面**：fetch 是服务器侧请求；**防护默认开启**——仅 http/https，DNS 解析后拒绝回环/私网/链路本地/云元数据（169.254.x.x）地址，重定向后复检。数据源在可信内网时设 `RECONCHECK_ALLOW_PRIVATE_FETCH=1` 豁免（部署文档 §2）。
- **只读**：引擎不写业务系统；改动仅发生在本地数据目录（拉取副本、令牌、任务/报告）。
- **故障隔离**：一批里单对失败不影响其他对；失败带原因（解析/编码/空文档等），可记录进你的任务表。`GET /api/jobs/{id}` 的 `error` 字段与逐对 `error` 可直接落日志（中文原因，提示友好）。
- **三方核对**：月底对账常用 PO+送货单+发票三单一起核：`POST /api/compare3`（3 个文件或 3 个 `doc_ids`），返回逐对报告 + consensus/outlier 判定（数值对比含单位对齐与容差，键支持 `part_no`/`entity` 归一）。