# Integrating an enterprise system — worked examples

> For engineers wiring ReconCheck into an ERP / finance platform. Three data
> shapes → three approaches; configs and calls below are copy-paste ready,
> along with error-code semantics.

## 1. Decide what you are integrating

| Your system offers | ReconCheck type | Entry point |
|---|---|---|
| Document downloads (CSV/XLSX/PDF) | `type=file` | `POST /api/datasources/{id}/fetch` → document library |
| JSON record arrays (line items) | `type=records` | same; records become a table |
| Both, with user pickers | either + `list_url` | `POST /api/datasources/{id}/list` |

## 2. records type: JSON detail → table

Say the ERP has `GET https://erp.internal/api/orders?order_no=PO-240913-001`
returning:

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

Config (`POST /api/datasources`):

```json
{
  "name": "ERP-PO-明细",
  "type": "records",
  "url": "https://erp.internal/api/orders?order_no=PO-240913-001",
  "auth": "header",
  "header_name": "X-ERP-TOKEN",
  "token": "server-side token",
  "records_path": "data.items",
  "id_field": "line_no",
  "name_field": "part_no",
  "list_url": "https://erp.internal/api/orders/list"
}
```

- `records_path` is a dot path into the array: `data.items`; missing or
  non-array → 422 with a hint.
- `id_field` / `name_field` drive the UI picker (`/list` returns `{id, name}`).
- `list_url` must also expose an array at `records_path`; if omitted, `/list`
  pulls the full `url` response.

Fetch into the library and reuse:

```bash
curl -X POST .../api/datasources/<id>/fetch                                # → doc_id
curl -X POST .../api/jobs -d "doc_ids=<po_doc>,<inv_doc>"                  # compare library docs
curl -X POST .../api/compare3 -d "doc_ids=<po>,<dn>,<inv>"                 # three-way (month-end)
```

## 3. file type: attachment streams

Endpoint returns the file bytes by id (`GET /api/attachments/{id}`) and a
list endpoint provides the picker:

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

`/list` returns `[{"id": "1", "name": "PO-240913-001.csv"}, …]`; the UI
fetch substitutes `{id}`. Without `{id}`, fetch pulls `url` as-is.

## 4. Auth — three modes

| auth | effect | example |
|---|---|---|
| `none` | nothing | — |
| `bearer` | `Authorization: Bearer <token>` | standard OAuth2 |
| `header` | `<header_name>: <token>` | internal gateways |

- `PUT /api/datasources/{id}` with an empty `token` keeps the stored secret;
  `GET` only ever echoes `has_token`.
- Tokens stay plaintext on disk by default; set `RECONCHECK_DATA_KEY` (install
  the `crypto` extra) to store them Fernet-encrypted in `datasources/*.json` —
  loading decrypts transparently.
- `/probe` really hits the endpoint: connection/timeout failure → `ok=false`;
  **non-2xx also counts as failed** (a 401 means the token is wrong, not "it
  works").

## 5. Error-code semantics

| Code | Scenario |
|---|---|
| `400` | ambiguous params (files + doc_ids mixed), missing name/url |
| `404` | doc_id / datasource / report missing (bad or deleted id) |
| `413` | upload above 64 MB |
| `422` | no pairable docs / empty upload / records_path empty / unreadable document |
| `502` | data-source fetch failure (connect, timeout, non-2xx, >50 MB) |
| `401` | `RECONCHECK_API_KEY` set but `X-API-Key` missing/wrong |

## 6. Best practices and boundaries

- **Stable field names**: `records_path`/`id_field` bind to backend fields;
  renaming a field means editing the source config.
- **Pagination**: v0 does not auto-paginate; `list_url` should serve the full
  list (or you paginate yourself before exposing it).
- **Payload size**: don't return tens of MB per records call; fetch cuts off
  at 50 MB while streaming (→ 502).
- **SSRF surface**: fetch is a server-side request; the guard is **on by
  default** — http/https only, and after DNS resolution loopback/private/
  link-local/cloud-metadata (169.254.x.x) addresses are refused, with a
  re-check after redirects. Set `RECONCHECK_ALLOW_PRIVATE_FETCH=1` only when
  the source lives on a trusted intranet (deployment §2).
- **Read-only**: the engine never writes to business systems; its only writes
  are local (fetched copies, tokens, jobs/reports).
- **Failure isolation**: one failing pair never breaks the batch; reasons
  come back in `error` fields per pair and on the job.
- **Three-way**: `POST /api/compare3` (3 files or 3 `doc_ids`) returns every
  pairwise report plus consensus/outlier verdicts — numeric values compared
  with the active tolerance and unit-aligned, keys normalisable
  (`part_no` / `entity`).