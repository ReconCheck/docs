# ReconCheck — Capabilities

> Kept in sync with [core](https://github.com/ReconCheck/core). This file answers "what can it actually do, where, and how". Status: ✅ implemented · 🧭 designed, not yet built · 🕔 later milestone.

---

## 1. Engine layer (CLI / Python API)

| Capability | Status | Notes |
|---|---|---|
| Tabular parsing CSV / TSV / TXT | ✅ | delimiter sniffing (`,` `;` `\t` `\|`); ragged rows get generated headers |
| Tabular parsing XLSX / XLSM | ✅ | read-only streaming load (flat memory), multi-sheet, `data_only` values, optional sheet filter |
| Encoding detection | ✅ | UTF-8 (with/without BOM) / GB18030 / UTF-16 / Latin-1; **plausibility gate**: binary junk raises a clear `TextDecodeError` instead of parsing as a garbage table |
| Row alignment | ✅ | exact key match on `match_on` with key normalisation (part numbers `A-012` → `a12`, entity suffix stripping, whitespace); cell-level units preserved |
| Built-in auto rule | ✅ | compares every shared numeric column, tolerance relative 0.001 / absolute 0.01; **never double-reports a column an explicit rule already covers** (column-level dedupe) |
| YAML differential rules | ✅ | `match_on` / `compare` (single or list) / `severity` (high·medium·low) / `tolerance` (relative + absolute) / `exceptions` (unit conversion `kg↔g`, rounding) / `evidence.require: both_sides` |
| Evidence chain | ✅ | every finding carries both-side coordinates + a `cell://` href; the report JSON is the contract |
| CLI | ✅ | `reconcheck compare LEFT RIGHT [--rules --match-on --normalize --sheet --output]`; missing files and bad input fail cleanly (no traceback) |

```bash
reconcheck compare examples/po.csv examples/invoice.csv \
  --rules examples/rules --match-on 料号 --normalize 料号:part_no --output report.json
```

## 2. Web API (FastAPI, default http://127.0.0.1:8765)

| Endpoint | Status | Purpose |
|---|---|---|
| `GET /api/health` | ✅ | liveness + engine version |
| `POST /api/compare` | ✅ | synchronous compare (2 files or 2 `doc_ids`), returns the report |
| `POST /api/jobs` | ✅ | **async batch**: upload files + reference library docs; auto-pairs by business key; background worker runs every pair |
| `GET /api/jobs/{id}` | ✅ | status / progress / per-pair summary; **failed pairs stay visible** with the reason, without breaking the batch |
| `GET /api/reports/{id}` | ✅ | archived report JSON |
| `GET/POST /api/documents` | ✅ | document library: list / register uploads |
| `GET /api/documents/{id}` | ✅ | meta + parsed-table preview (first 100 rows) |
| `GET /api/documents/{id}/content` · `DELETE` | ✅ | raw download / delete |
| `GET/POST/PUT/DELETE /api/datasources` | ✅ | enterprise data source (custom web API) configuration |
| `POST /api/datasources/{id}/probe` | ✅ | connectivity test (non-2xx counts as failure, includes status/bytes) |
| `POST /api/datasources/{id}/list` | ✅ | list pickable records/documents |
| `POST /api/datasources/{id}/fetch` | ✅ | fetch a file stream or records JSON into the library |

### Batch semantics

- Document-kind tokens (`po`/`order`/`采购`, `inv`/`发票`, …) are stripped from filenames; files sharing the remaining business key pair up (e.g. `PO-240913-001` ↔ `INV-240913-001`); singletons land in `unpaired`.
- Duplicate references (same `doc_id`, or byte-identical uploads) are deduped — no pointless self-comparisons.
- One bad pair fails on its own; the rest of the batch still finishes.

### Data sources

- **type=file** — endpoint returns the document byte stream; URL may contain a `{id}` placeholder; optional `list_url` serves the picker entries.
- **type=records** — endpoint returns JSON; `records_path` selects the array (e.g. `data.items`); records become a table (union of keys as headers, one row per record).
- Auth: `none` / `bearer` / custom `header` (`header_name`); tokens stored in the local data dir, the API only echoes `has_token`.
- Fetching is a **server-side request** (no CORS) with a **50 MB streaming cap** — oversized replies abort mid-download.

## 3. Frontend (dependency-free static page served at `/`)

- ✅ drag & drop / file picker batch upload with duplicate guards; automatic pairing groups
- ✅ severity summary (high / medium / low); clicking a finding's `cell://` evidence **highlights the exact source cell**
- ✅ results page renders failed pairs with their reasons; one broken report fetch never takes the whole page down
- ✅ document library: browse / preview / add to the compare queue (deduped by id)
- ✅ data source form: create / edit / test / list entries / fetch into the library
- ✅ friendly messages in Chinese for unpaired, empty, oversized, deleted-document and similar cases

## 4. Robustness & security (implemented)

| Capability | Notes |
|---|---|
| Upload limit | 64 MB per file, 413 above |
| Fetch limit | 50 MB per data-source response, enforced while streaming |
| Path-traversal guard | document/report ids validated against a 12-hex allowlist for delete / download / report reads |
| Optional auth | `RECONCHECK_API_KEY` turns on global `X-API-Key` checks; a loud startup warning when unset |
| Persistence | `RECONCHECK_DATA` holds jobs/reports/documents/datasources; jobs.json written atomically |
| Restart recovery | queued jobs are replayed after a restart; a TTL janitor prunes old files hourly (`RECONCHECK_TTL_DAYS`, default 30, active jobs never touched) |
| Failure isolation | a failing pair is marked `failed` with its reason; the batch continues |
| Validation | config validation, empty token keeps the stored secret, response-header injection sanitised |

## 5. Read-only audit stance

ReconCheck is **read-only**: it never writes to business systems and never modifies source files. Its only writes are the operator's data-source tokens and fetched copies — all inside the local data directory.

---

## Design & roadmap

| Item | Status | Content |
|---|---|---|
| Cross-engine "job layer" | 🧭 | see [job-layer.md](./job-layer.md): Document / Pair / Report as engine-agnostic abstractions; future engine adapters; LLM stays an engine-layer pipe |
| LLM participation | 🧭 | opt-in `LLMEnhancer` (interface reserved in `reconcheck/llm`): alignment disambiguation + finding explanations; OpenAI-compatible endpoints only, timeouts/budget/rollback to the deterministic result, `llm_augmented` flags |
| PDF / scanned documents | 🕔 | layout reconstruction, borderless tables, multi-column PDFs (largest milestone) |
| Entity resolution | 🕔 | real linking (`华加` ↔ `深圳市华加生物科技有限公司`; today: suffix/whitespace normalisation) |
| Three-way match | 🕔 | purchase order / delivery note / invoice |
| Rule format v1 | 🕔 | richer exception catalogue, frozen evidence output |
| Distributed queue | 🕔 | today: single-process worker + file persistence, no message queue |