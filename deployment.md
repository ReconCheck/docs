# 部署与内网防护

> 把 ReconCheck 装到服务器（而不是开发机）上的步骤、防护清单与资源评估。核心仓库的 `pyproject.toml` 同时提供引擎与 Web 依赖。

## 1. 安装

```bash
# 需要 Python 3.10+（CI 覆盖 3.10 / 3.12）
git clone https://github.com/ReconCheck/core.git reconcheck
cd reconcheck
python -m venv .venv
.venv/Scripts/python -m pip install -e ".[dev,web,pdf]"   # Windows（pdf 为 PDF 文本层解析，可选）
.venv/bin/python -m pip install -e ".[dev,web,pdf]"       # Linux/macOS

# 冒烟
.venv/bin/reconcheck --version
.venv/bin/python -m uvicorn reconcheck.web.app:create_app --host 0.0.0.0 --port 8765
```

## 2. 必配项（进部署 checklist）

| 项 | 建议 |
|---|---|
| `RECONCHECK_API_KEY` | **必须设置**。未设置时服务无鉴权，任何人可读写（启动会打印告警）。设为强随机串：`openssl rand -hex 24` |
| `RECONCHECK_DATA` | 指向独立数据目录（默认 `./data`）；磁盘备份只看这一个目录即可 |
| 监听地址 | 业务内网即使有鉴权也建议不暴露公网；需要对外时走反向代理（§5） |
| `RECONCHECK_TTL_DAYS` | 默认 30；按审计留存要求调整（上传文件与报告由后台每小时清理，在途任务不删） |
| `RECONCHECK_ALLOW_PRIVATE_FETCH` | 默认**不设** → 数据源拉取禁止回环/私网/链路本地/云元数据（169.254.x.x）地址，重定向后复检；仅当数据源确实在**可信内网**时设 `=1` 显式豁免（方案白名单 http/https 始终生效） |
| `RECONCHECK_DATA_KEY` | 选配：数据源令牌的落盘加密密钥（`pip install "reconcheck[crypto]"`，Fernet key：`python -c "from cryptography.fernet import Fernet;print(Fernet.generate_key().decode())"`）。设置后新保存的令牌以密文写入 `datasources/*.json`；**未设置 = 明文存储**。密钥丢失则旧密文令牌视为缺失 |
| `RECONCHECK_OCR` | 选配：`=1` 时对无文本层的 PDF 启用 Tesseract OCR（`pip install "reconcheck[pdf-ocr]"`）；默认关闭——扫描件仍明确报错。二进制不在 PATH 时用 `RECONCHECK_TESSERACT_CMD` 指定完整路径 |

## 3. 数据目录结构（备份/恢复）

```
<RECONCHECK_DATA>/
├── jobs.json          # 任务索引（原子写入；重启自动重放 queued 任务）
├── files/<job_id>/    # 批次上传的原始文件（TTL 清理）
├── reports/*.json     # 比对报告归档（TTL 清理）
├── documents/<id>/    # 文档库（上传/数据源拉取副本 + meta.json）
└── datasources/*.json # 数据源配置（令牌明文或 Fernet 密文，视 RECONCHECK_DATA_KEY——备份即凭证，注意保管）
```

备份 = 冷拷贝整个目录；恢复 = 解压回原路径后启动。默认**令牌明文存储**（操作员配置的凭证库，不是保险库）；配置 `RECONCHECK_DATA_KEY` 后改为加密存储（密钥需另行保管）。无论哪种，备份文件都按密级对待。

## 4. 服务化

**Linux（systemd）** `/etc/systemd/system/reconcheck.service`：

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

**Windows（nssm）**：

```bat
nssm install ReconCheck ".venv\Scripts\python.exe" "-m uvicorn reconcheck.web.app:create_app --host 127.0.0.1 --port 8765"
nssm set ReconCheck AppEnvironmentExtra RECONCHECK_API_KEY=CHANGE_ME RECONCHECK_DATA=D:\reconcheck\data
nssm start ReconCheck
```

## 5. 反向代理（Nginx 示例）

```nginx
server {
    listen 8443 ssl;
    server_name reconcheck.internal;
    # ssl_certificate / ssl_certificate_key ...

    location / {
        proxy_pass http://127.0.0.1:8765;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        client_max_body_size 70m;   # 上传上限 64MB + multipart 开销
    }
}
```

注意：`client_max_body_size` 必须 ≥ 上传上限（64MB），否则 Nginx 会在引擎之前把大文件 413 掉。

## 6. 资源评估

| 维度 | 说明 |
|---|---|
| 内存 | XLSX 只读流式加载，内存随行数平缓增长；典型对账表（万行内）数百 MB 内完成。上传本身 64MB 封顶 |
| 磁盘 | `files/` 与 `reports/` 受 TTL 清理（默认保留 30 天）；`documents/` 与 `datasources/` 不自动清理，按业务量另做归档 |
| CPU/并发 | 单进程 + 单 worker 线程串行队列：任务排队执行、互不并发。吞吐要求高时横向部署多实例，各自独立数据目录 |
| 网络 | 数据源拉取为服务器侧请求（20s 超时、50MB 流式上限、重定向后复检）；SSRF 防护默认开启：仅 http/https，拒绝回环/私网/链路本地/云元数据解析结果；可信内网用 `RECONCHECK_ALLOW_PRIVATE_FETCH=1` 豁免 |

## 7. 升级与日常

- 升级 = 拉新代码 → 重装依赖 → 重启。`jobs.json` 里 queued 的任务重启后自动重放；running 的中断任务由操作员重新提交。
- 日常看 `GET /api/health`、数据目录磁盘占用；日志走 Uvicorn stderr（systemd journal / nssm 日志）。
- 定期用 `GET /api/jobs/{id}` 抽查旧任务结果是否仍可拉取（报告若被 TTL 清理则为 404，属预期）。

## 8. 安全清单（上线前逐项勾）

- [ ] `RECONCHECK_API_KEY` 已设置且为强随机值
- [ ] 监听未直接暴露公网（本机 + 反代，或业务内网）
- [ ] 数据目录位置已定、备份策略已配、备份按密级保管（令牌默认明文落盘，可设 `RECONCHECK_DATA_KEY` 加密）
- [ ] 若业务需要扫描件：已安装 `pdf-ocr` 扩展并设 `RECONCHECK_OCR=1`（否则保持默认，明确报错）
- [ ] 数据源端点在内网时已设 `RECONCHECK_ALLOW_PRIVATE_FETCH=1`，且端点本身可信（该变量关闭私网地址拦截）
- [ ] `client_max_body_size` ≥ 70m（或与上传上限匹配）
- [ ] LLM 功能未启用时 `reconcheck/llm` 保持 NullEnhancer（默认，不产生任何外部调用）