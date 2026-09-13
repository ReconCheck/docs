# ReconCheck — End-user guide

Three audiences: **reconciliation / finance operators** (web UI), **enterprise
integrations** (REST API), **developers** (CLI / embedding).

> Scope first: tabular files (CSV / TSV / XLSX) and **PDF text layers** are
> supported; **scanned PDFs and images need OCR**, which is not wired yet.
> Files auto-pair only when their names share a business number (see FAQ).

---

## 1. Operators: the web UI (5-minute loop)

### Start the service

```bash
reconcheck-api        # then open http://127.0.0.1:8765
```

### Upload a batch

- Drag the documents of one business case into the page (e.g.
  `PO-240913-001.csv` and `INV-240913-001.csv`).
- Kind words are stripped (`po`/`order` ↔ `inv`/`invoice` …) and the remaining
  business number `240913001` groups them into pairs.
- Grouped pairs are listed; files that appear once land in **unpaired**
  (usually: no shared number in the filename).

**Three-way verification** — for the purchase chain (PO + delivery note +
invoice) or the **sales chain (sales order + outbound + sales invoice)**,
use `reconcheck compare3` or `POST /api/compare3` (3 files or 3 `doc_ids`):
you get every pairwise report plus a consensus pass that names the **outlier
side** (e.g. "invoice disagrees with PO + delivery note").

### Run and read results

- Top summary: **high / medium / low** finding counts.
- Each finding: field, both values, row/column, severity.
- Click a finding's **evidence link** → the page jumps to the original table
  and **highlights the exact cell**. Every claim is checkable against the
  source — that is the whole point.

### Failed pairs

A pair that fails (empty file, unreadable content, …) is marked **failed with
its reason**; the rest of the batch still runs.

### Document library

Upload-view-delete original files, see parsed previews, add library documents
to a compare queue (a document is never queued twice).

### Data sources (enterprise systems)

The "data sources" page configures a **custom web API**:

| Field | Meaning |
|---|---|
| Type | `file` (endpoint returns the document byte stream) or `records` (returns JSON records) |
| URL | endpoint; `file` URLs may contain a `{id}` placeholder |
| Auth | none / Bearer token / custom header (`header_name` + token) |
| records_path | JSON array path, e.g. `data.items` |
| id_field / name_field | picker id / display name in the record list |
| list_url | optional endpoint serving the pickable list |

Then **Test** (non-2xx is marked failed — a 401 means the token is wrong),
**List** the records, **Fetch** a document into the library.

## 2. Enterprise integrations: REST API

```bash
# synchronous compare
curl -X POST http://127.0.0.1:8765/api/compare \
  -F "files=@po.csv" -F "files=@invoice.csv"

# three-way (PO + delivery note + invoice)
curl -X POST http://127.0.0.1:8765/api/compare3 \
  -F "files=@PO-001.csv" -F "files=@DN-001.csv" -F "files=@INV-001.csv"

# async batch: upload many files, auto-pair, poll, fetch reports
curl -X POST http://127.0.0.1:8765/api/jobs -F "files=@..."
# then GET /api/jobs/{id} until done → GET /api/reports/{report_id}

# data source: configure → probe → list → fetch → use doc_ids in jobs
curl -X POST http://127.0.0.1:8765/api/datasources -H "Content-Type: application/json" \
  -d '{"name":"ERP-PO","type":"records","url":"https://erp/api/orders","records_path":"data","auth":"bearer","token":"xxx"}'
```

- Fetching is a **server-side request** (no CORS).
- The report JSON is the contract; every finding carries `evidence` with
  both-side coordinates and a `cell://` href.

### Auth

With `RECONCHECK_API_KEY` set, every `/api/*` call needs an `X-API-Key`
header; **without it the API is unauthenticated** (startup warning — do not
expose to untrusted networks).

**Entering the key in the web UI:** press `Ctrl+K` for a key prompt (stored
in browser localStorage, attached to every `/api` call automatically; cancel
or leave blank to clear). On a 401 the page prompts again.

## 3. Developers: CLI / embedding

```bash
reconcheck compare examples/po.csv examples/invoice.csv \
  --rules examples/rules --match-on 料号 --normalize 料号:part_no --output report.json
reconcheck compare3 examples/po.csv examples/dn.csv examples/invoice.csv --match-on 料号
```

```python
from reconcheck.comparison import compare_files
report, findings = compare_files("a.csv", "b.csv", match_on=["料号"])
```

Rule format and exception semantics: core `DESIGN.md` and the docs
[writing-rules](./writing-rules.md) guide.

## 4. FAQ

**Q: "No comparable pairs"?** The business numbers after stripping kind words
differ, or a group needs ≥2 files. Naming that works: purchase
`PO-240913-001` ↔ `INV-240913-001`, sales `SO-240913-001` ↔ `SIV-240913-001`
(kind tokens stripped: `po`/`order`/`采购`, `so`/`sales`/`销售`,
`out`/`outbound`/`出库`, `dn`/`送货`, `inv`/`发票`, `siv`/`销项`, …);
`a.csv` ↔ `b.csv` does not pair.

**Q: Zero findings — suspicious?** Possibly truly no differences; or the
numeric columns differ in name / are not numeric. The auto rule only compares
shared numeric columns. Non-numeric columns (dates, status text) only produce
findings when a rule declares `dates_within` / `text_equal_ignore_case`.

**Q: Empty / binary junk files?** Empty → that pair is marked failed;
binary junk → a clear "not readable text" error (no garbage table).

**Q: PDF support?** Text-layer PDFs yes (ruling-line tables first, layout
fallback for borderless print-outs). Scanned (image-only) PDFs report "OCR
required" — not wired yet.

**Q: Size limits?** Upload ≤ 64 MB per file; data-source fetch ≤ 50 MB
(streaming cap). Oversized input errors out clearly.

**Q: Chinese filenames?** Parsing is fine; pairing needs a **shared business
number** in the name (a pure-Chinese name like `订单明细.csv` won't group).

**Q: Retention?** Default 30 days (`RECONCHECK_TTL_DAYS`), hourly janitor,
active jobs never touched.

**Q: Does data leave the box?** No by default; only operator-configured data
sources (fetch) — and, in the future, opt-in LLM endpoints — are contacted.