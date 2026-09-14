# Deployment & network hardening

> Server install, mandatory knobs, data-dir backup, service units, reverse
> proxy, sizing, and an upgrade/security checklist. `pyproject.toml` carries
> engine + web dependencies (PDF parsing is an optional extra).

## 1. Install

```bash
# Python 3.10+ required (CI covers 3.10 / 3.12)
git clone https://github.com/ReconCheck/core.git reconcheck
cd reconcheck
python -m venv .venv
.venv/Scripts/python -m pip install -e ".[dev,web,pdf]"   # Windows; pdf is optional (text-layer PDF)
.venv/bin/python -m pip install -e ".[dev,web,pdf]"       # Linux/macOS

# smoke test
.venv/bin/reconcheck --version
.venv/bin/python -m uvicorn reconcheck.web.app:create_app --host 0.0.0.0 --port 8765
```

## 2. Mandatory knobs (deployment checklist)

| Item | Recommendation |
|---|---|
| `RECONCHECK_API_KEY` | **Set it.** Without it the API is unauthenticated (startup warning prints). Strong random: `openssl rand -hex 24` |
| `RECONCHECK_DATA` | a dedicated data directory (default `./data`); the whole thing is one backup unit |
| Listen address | keep it off the public internet; use a reverse proxy (see §5) if exposure is needed |
| `RECONCHECK_TTL_DAYS` | default 30; adjust to your retention policy (uploaded files + reports are pruned hourly; active jobs never touched) |
| `RECONCHECK_ALLOW_PRIVATE_FETCH` | unset by default → data-source fetch rejects loopback / private / link-local / cloud-metadata (169.254.x.x) addresses and re-checks after redirects; set `=1` only when the source genuinely lives on a **trusted intranet** (the http/https scheme whitelist always applies) |
| `RECONCHECK_DATA_KEY` | optional: Fernet key that encrypts data-source tokens at rest (`pip install "reconcheck[crypto]"`; generate with `python -c "from cryptography.fernet import Fernet;print(Fernet.generate_key().decode())"`). Without it tokens stay **plaintext on disk**. Lose the key and previously encrypted tokens read as absent |
| `RECONCHECK_OCR` | optional: `=1` enables Tesseract OCR for PDFs without a text layer (`pip install "reconcheck[pdf-ocr]"`); off by default — scans still fail with a clear error. Point `RECONCHECK_TESSERACT_CMD` at the binary when it is not on PATH |

## 3. Data directory layout (backup / restore)

```
<RECONCHECK_DATA>/
├── jobs.json          # job index (atomic writes; queued jobs replay after restart)
├── files/<job_id>/    # batch uploads (TTL-pruned)
├── reports/*.json     # report archive (TTL-pruned)
├── documents/<id>/    # document library (uploads + fetched copies + meta.json)
└── datasources/*.json # data source configs — tokens plaintext or Fernet-encrypted (RECONCHECK_DATA_KEY): treat backups as secrets
```

Backup = cold-copy the directory; restore = unpack at the same path and start.
By default **tokens are stored in plaintext** (an operator-configured credential
store, not a vault); set `RECONCHECK_DATA_KEY` to store them encrypted instead
(the key itself must be escrowed separately). Protect the directory either way.

## 4. Run as a service

**Linux (systemd)** `/etc/systemd/system/reconcheck.service`:

```ini
[Unit]
Description=ReconCheck verification engine
After=network.target

[Service]
User=reconcheck
WorkingDirectory=/opt/reconcheck
Environment=RECONCHECK_API_KEY=CHANGE_ME
Environment=RECONCHECK_DATA=/var/lib/reconcheck
ExecStart=/opt/reconcheck/.venv/bin/python -m uvicorn reconcheck.web.app:create_app --host 127.0.0.1 --port 8765
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

```bash
systemctl daemon-reload && systemctl enable --now reconcheck
```

**Windows (nssm)**:

```bat
nssm install ReconCheck ".venv\Scripts\python.exe" "-m uvicorn reconcheck.web.app:create_app --host 127.0.0.1 --port 8765"
nssm set ReconCheck AppEnvironmentExtra RECONCHECK_API_KEY=CHANGE_ME RECONCHECK_DATA=D:\reconcheck\data
nssm start ReconCheck
```

## 5. Reverse proxy (Nginx example)

```nginx
server {
    listen 8443 ssl;
    server_name reconcheck.internal;
    # ssl_certificate / ssl_certificate_key ...

    location / {
        proxy_pass http://127.0.0.1:8765;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        client_max_body_size 70m;   # upload cap is 64MB + multipart overhead
    }
}
```

`client_max_body_size` must be ≥ the 64 MB upload cap, or Nginx will 413
before the engine ever sees the file.

## 6. Sizing

| Dimension | Notes |
|---|---|
| Memory | XLSX loads streamed (flat memory); typical reconciliation tables (≤10k rows) need a few hundred MB. Uploads are capped at 64 MB |
| Disk | `files/` and `reports/` are TTL-pruned (default 30 days); `documents/` and `datasources/` are not — archive them separately if needed |
| CPU / concurrency | single process, single worker thread, serial queue; horizontal scale = more instances, each with its own data directory |
| Network | data-source fetch is a server-side request (20s timeout, 50 MB streaming cap, redirects re-checked); SSRF guard on by default: http/https only, loopback/private/link-local/metadata resolutions refused; trusted intranets opt out with `RECONCHECK_ALLOW_PRIVATE_FETCH=1` |

## 7. Upgrades & daily operations

- Upgrade = pull → reinstall deps → restart. `queued` jobs replay after
  restart; interrupted `running` jobs are re-submitted by the operator.
- Health: `GET /api/health`; watch disk usage under `RECONCHECK_DATA`; logs to
  Uvicorn stderr (systemd journal / nssm logs).
- Reports pruned by TTL return 404 — expected behaviour, not an error.

## 8. Pre-launch security checklist

- [ ] `RECONCHECK_API_KEY` set to a strong random value
- [ ] Not listening on the public internet (localhost + proxy, or a trusted VLAN)
- [ ] Data directory location and backup plan defined; backups treated as secrets (tokens plaintext by default; `RECONCHECK_DATA_KEY` enables encryption at rest)
- [ ] Scanned input expected? `pdf-ocr` extra installed and `RECONCHECK_OCR=1` set — otherwise the default stays a clear refusal
- [ ] Intranet data-source endpoints: `RECONCHECK_ALLOW_PRIVATE_FETCH=1` set deliberately and the endpoints themselves trusted (the env var turns off the private-address guard)
- [ ] Proxy `client_max_body_size` ≥ 70m (or matched to the cap)
- [ ] LLM participation disabled by default (`NullEnhancer`) — no external calls unless opted in