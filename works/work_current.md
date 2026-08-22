# 当前项目状态

更新日期：2026-08-22

## 当前阶段

本次任务书要求的最小阶段已完成：公开只读 API → 本地 Streamable HTTP MCP Server → 本地 Codex 的链路已经实现并验证。

未执行 commit、push、branch、PR，也未修改 `sources/`。

## 已实现

- `server.py`
  - 官方 MCP Python SDK v2 高层 `MCPServer`；
  - Streamable HTTP endpoint：`http://127.0.0.1:8000/mcp`；
  - host 固定为 `127.0.0.1`，仅 `--port` 可选；
  - 保留 SDK 默认 DNS 重绑定 Host/Origin 防护；
  - 精简 Server Instructions；
  - 固定公开上游地址，只有 GET，不提供任意 URL/method/endpoint。
- 四个只读 tools：
  - `get_source_info`
  - `list_documents`
  - `search_documents`
  - `get_document`
- `pyproject.toml` 与 `uv.lock`
  - Python 3.10+；
  - 直接依赖只有 `mcp`、`httpx` 与用于 tool schema 约束的 `pydantic`；
  - 当前 lock 固定 `mcp==2.0.0`、`httpx==0.28.1`、`pydantic==2.13.4`。
- `.gitignore`
  - 忽略 `.venv`、Python cache 与 macOS `.DS_Store`。
- 已按最终工具同步：
  - `README.md`
  - `prompts/llm_prompt.md`
  - `works/work_history.md`

## 工具行为摘要

- `get_source_info` 基本原样返回来源统计和引用要求。
- `list_documents` 每次读取上游全量索引，在 MCP 内存中做来源/日期筛选、轻量字段选择和 offset/limit。
- `search_documents` 分别在索引 `title` 或 `excerpt` 上做 Unicode casefold 字面子串匹配，不跨字段拼接；不是上游、全文或语义搜索。
- `get_document` 按精确 ID 获取详情，把 metadata 放在 `document`，把 `markdown` 按 Unicode 字符分块；默认 20,000、最多 50,000 字符，并返回 `next_offset`。
- 不缓存、不重试、不排序、不摘要、不改写事实字段。

## 已完成验证

### 依赖与静态检查

- `uv sync --no-cache` 成功生成并安装 lock 环境；随后 `uv lock --check --offline` 与 `uv sync --frozen --offline` 通过。
- `uv pip check` 确认当前 31 个已安装包兼容。
- `python -m py_compile server.py` 成功。
- MCP in-process client 协商成功，四个 tools 的名称、schema、Server Instructions 与 `read_only_hint=true` 均已检查。
- 参数错误验证成功：超限 `limit`/`content_limit`、反向日期范围、纯空白 query、含路径分隔符的 document ID 均返回 `is_error=true`。
- fixture 验证确认搜索分别匹配 `title` 或 `excerpt`，不会让含换行的 query 跨两个字段拼接命中。

### 真实上游调用

- `get_source_info`：HTTP 200，当前返回 `laoqian=1583`、`mianji=145`。
- `list_documents(source_key="laoqian", limit=2)`：成功返回两项。
- `search_documents(query="gamma")`：成功命中 `Life is long gamma`。
- 短文详情：1196/1196 个 Markdown 字符一次完整返回，`has_more=false`。
- 当前最大逐字稿详情：请求 100 字符后返回 `total_chars=243517`、`has_more=true`、`next_offset=100`，分块行为正确。
- 沙箱禁止外网时，上游调用返回清晰的 `ConnectError` tool error；联网后同一调用成功，说明错误路径与正常路径均可解释。

### Streamable HTTP 与 Codex

- 启动输出正确显示 `MCP server: <http://127.0.0.1:8000/mcp>`。
- `lsof` 确认仅 `TCP 127.0.0.1:8000 (LISTEN)`，没有 `0.0.0.0`/`*` 监听。
- Python MCP client 自动发现：协议 `2026-07-28`，成功列出四个 tools 并调用 `get_source_info`。
- Python MCP client legacy initialize：协议 `2025-11-25`，成功列 tools 并调用代表工具。
- 伪造 `Host: untrusted.example` 的请求返回 HTTP 421，默认本地 HTTP 安全检查有效。
- Codex CLI 0.149.0 使用 `--ephemeral --ignore-user-config` 和临时 `-c` MCP 配置，实际产生 `mcp_tool_call` 并成功调用 `laoqianritan/get_source_info`；没有写入用户 MCP 配置。
- Codex 模型会话最初 WebSocket 采样超时 5 次，自动回退 HTTPS 后成功。这是 Codex 外部采样链路现象，不是 MCP 初始化或 tool 调用失败。
- 验证结束后已停止测试服务，端口 8000 无残留监听。

### 收尾审查

- 代码、文档和离线测试三路独立复核均无阻断项。
- 根据复核意见补充了 `content_offset` 上限、分块拼接依赖上游内容未变化的前提和已校验响应结构的精确范围，并消除了 query 跨标题/摘要字段命中的边角。
- 最终 `git diff --check`、离线 lock/sync、编译、依赖兼容检查和定向搜索 fixture 均通过；`sources/` 仍无改动。

## 已知上游限制与歧义

- `sources/openapi.json` 的 server 是 `https://example.com` 占位值；实际 base URL 根据 `sources/sources.md` 的站点地址确定为 `https://laoqianritan-create.github.io/`，并经真实请求验证。
- 只有三个业务 GET endpoint；上游无已确认的分页、筛选、排序或搜索参数。
- 当前全量索引 1728 项、约 2.35 MB；冷请求可能较慢。MCP 读取超时为 60 秒，不实现缓存或重试。
- 当前 145 个 `mianji` 条目的 `date` 都是 `null`；日期筛选会排除它们，不补造日期。
- 上游首条索引记录可见 title 与 ID/source_path 指向内容不一致；MCP 原样保留，不静默修复。
- 上游没有承诺稳定排序、请求频率、更新时效、绝对完整性或错误响应 schema。
- 上游内容必须保留各自 `attribution_text`。

## Todo

- 等待人工审查当前工作树并自行 commit。
- 如上游 API 文档或字段未来变化，先更新 `sources/` 参考资料，再同步检查 server tool schema、`prompts/llm_prompt.md`、README 与本文件。
- 当前没有阻断本阶段交付的未解决实现问题。
