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

## 3. Data directory layout (backup / restore)

```
<RECONCHECK_DATA>/
├── jobs.json          # job index (atomic writes; queued jobs replay after restart)
├── files/<job_id>/    # batch uploads (TTL-pruned)
├── reports/*.json     # report archive (TTL-pruned)
├── documents/<id>/    # document library (uploads + fetched copies + meta.json)
└── datasources/*.json # data source configs — include plaintext tokens: treat backups as secrets
```

Backup = cold-copy the directory; restore = unpack at the same path and start.
**Tokens are stored in plaintext** (an operator-configured credential store,
not a vault) — protect the directory accordingly.

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
| Network | data-source fetch is a server-side request (20s timeout, 50 MB streaming cap, follows redirects); the operator configures endpoints deliberately (v0 has no SSRF guard) |

## 7. Upgrades & daily operations

- Upgrade = pull → reinstall deps → restart. `queued` jobs replay after
  restart; interrupted `running` jobs are re-submitted by the operator.
- Health: `GET /api/health`; watch disk usage under `RECONCHECK_DATA`; logs to
  Uvicorn stderr (systemd journal / nssm logs).
- Reports pruned by TTL return 404 — expected behaviour, not an error.

## 8. Pre-launch security checklist

- [ ] `RECONCHECK_API_KEY` set to a strong random value
- [ ] Not listening on the public internet (localhost + proxy, or a trusted VLAN)
- [ ] Data directory location and backup plan defined; backups treated as secrets (plaintext tokens)
- [ ] No data-source config points at sensitive internal paths (SSRF surface)
- [ ] Proxy `client_max_body_size` ≥ 70m (or matched to the cap)
- [ ] LLM participation disabled by default (`NullEnhancer`) — no external calls unless opted in