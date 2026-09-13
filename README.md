# ReconCheck docs

面向 [ReconCheck](https://github.com/ReconCheck/core) —— 跨单据核对引擎（cross-document verification engine）的文档。

> **Status: runnable closed loop.** 表格文件（CSV/TSV/XLSX）的解析—对齐—判定—引用闭环、Web API 批处理、企业数据源集成、前端上传界面均已可用；安全性（鉴权/上限/路径防护）与运维（持久化/TTL 清理）已加固。开发状态与 roadmap 见 core 仓库 README。

## 文档索引

| 文档 | 内容 |
|---|---|
| [功能能力清单 capabilities.md](./capabilities.md) | **具体能实现哪些功能**：按引擎层 / Web API / 前端 / 健壮性安全分层的功能表，含使用入口与规划 |
| [作业层定位 job-layer.md](./job-layer.md) | 跨验证引擎的「作业层」设计：Document / Pair / Report 抽象、引擎可替换、只读约束 |
| [核心设计 DESIGN.md](https://github.com/ReconCheck/core/blob/main/DESIGN.md) | 引擎四阶段流水线、数据模型、规则格式、Web 层、LLM 参与设计 |
| [快速开始 README.md](https://github.com/ReconCheck/core/blob/main/README.md) | 安装、CLI、Web UI、API 端点表、运维参数 |

## 内容规划（待补充）

- **Writing rules** — 差分规则编写指南与实例（引擎侧格式见 core DESIGN.md，规则规范见私有 [rules](https://github.com/ReconCheck/rules) 仓库）
- **Deployment** — 内网部署、`RECONCHECK_API_KEY` 防护、资源评估
- **Integration** — 接入 ERP / 数据源的完整示例（含 records JSON 的 `records_path`/`id_field` 最佳实践）

## 协作

核心接口仍在演进，文档以协作为主；如需补充请开 issue 而不是直接 PR。