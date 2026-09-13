# ReconCheck docs

面向 [ReconCheck](https://github.com/ReconCheck/core) —— 跨单据核对引擎（cross-document verification engine）的文档。

> **Status: runnable closed loop.** 表格文件（CSV/TSV/XLSX）的解析—对齐—判定—引用闭环、Web API 批处理、企业数据源集成、前端上传界面均已可用；安全性（鉴权/上限/路径防护）与运维（持久化/TTL 清理）已加固。开发状态与 roadmap 见 core 仓库 README。

## 文档索引

| 文档 | 内容 |
|---|---|
| [用户使用指南 user-guide.md](./user-guide.md) | **用户怎么用**：操作员网页流程、企业系统 REST API 对接、开发者 CLI、FAQ |
| [User guide (English)](./user-guide.en.md) | English end-user guide: web UI, REST API, CLI, FAQ |
| [功能能力清单 capabilities.md](./capabilities.md) | **具体能实现哪些功能**（中文）：按引擎层 / Web API / 前端 / 健壮性安全分层的功能表，含使用入口与规划 |
| [Capabilities (English)](./capabilities.en.md) | English capability list — what this build can do, by layer |
| [规则编写指南 writing-rules.md](./writing-rules.md) | 差分规则怎么写、怎么调、怎么排查：字段逐项说明、exceptions、自动规则、调试清单 |
| [Writing rules (English)](./writing-rules.en.md) | English rule-writing guide: fields, exceptions, auto rule, debugging |
| [部署与内网防护 deployment.md](./deployment.md) | 服务器安装、必配项（API 密钥/TTL）、数据目录备份、systemd/nssm 服务化、反向代理、资源评估、安全清单 |
| [Deployment (English)](./deployment.en.md) | English deploy & hardening: install, knobs, backup, service units, proxy, sizing, checklist |
| [ERP 接入示例 integration.md](./integration.md) | records/file 两型的完整配置与调用、鉴权三种、错误码语义、最佳实践与边界 |
| [ERP integration (English)](./integration.en.md) | English integration examples: records/file types, auth modes, error codes, best practices |
| [作业层定位 job-layer.md](./job-layer.md) | 跨验证引擎的「作业层」设计：Document / Pair / Report 抽象、引擎可替换、只读约束 |
| [Job layer (English)](./job-layer.en.md) | English cross-engine job-layer positioning: engine-agnostic Document/Pair/Report abstractions |
| [核心设计 DESIGN.md](https://github.com/ReconCheck/core/blob/main/DESIGN.md) | 引擎四阶段流水线、数据模型、规则格式、Web 层、LLM 参与设计 |
| [快速开始 README.md](https://github.com/ReconCheck/core/blob/main/README.md) | 安装、CLI、Web UI、API 端点表、运维参数 |

> 文档状态：上述内容均已覆盖；随 core 仓库开发持续更新。

## 协作

核心接口仍在演进，文档以协作为主；如需补充请开 issue 而不是直接 PR。