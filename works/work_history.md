# 工作历史与技术决策

## 2026-08-22 — 建立最小只读 MCP

### API 事实与 base URL

- 仓库资料确认三个业务 GET endpoint：
  - `/api/documents.json`
  - `/api/sources.json`
  - `/api/documents/{id}.json`
- `sources/openapi.json` 中 `servers[0].url=https://example.com` 显然是占位值；`sources/sources.md` 明确给出站点 `https://laoqianritan-create.github.io/`。本实现据此使用固定 base `https://laoqianritan-create.github.io/api`，并通过三个真实 endpoint 的 HTTP 200 响应确认。
- 没有实现 OpenAPI 未确认的写 endpoint、鉴权、搜索 endpoint 或通用 HTTP proxy。

### 技术选择

- 采用 Python、uv、官方 MCP Python SDK 当前稳定版 v2.0.0、`httpx`，并直接声明用于 tool schema 参数约束的 `pydantic`；三者已经是实现直接 import 的最小依赖集合。
- 使用 v2 高层 `MCPServer`，避免自行处理 JSON-RPC、schema、transport 和协议协商。
- `host`、`port`、`streamable_http_path` 按 SDK v2 要求传给 `mcp.run()`；host 固定 `127.0.0.1`，path 固定 `/mcp`，只有端口可从 CLI 修改。
- 未传入或放宽 `transport_security`，保留 SDK 为 localhost 自动配置的 DNS 重绑定防护。
- 直接运行命令选为 `uv run --frozen python server.py`。没有使用 `uv run mcp run server.py`，因为后者导入模块后自行运行，不会执行本项目 `main()` 中的端口解析和地址输出。

### Tool 设计

- 最终暴露四个只读 tools：`get_source_info`、`list_documents`、`search_documents`、`get_document`。
- 保留 `search_documents` 的理由：上游 API 页面明确说明全量索引用于“搜索、筛选”，而本工具只做很小、完全确定性的本地索引子串匹配。tool description 和 LLM Prompt 明确它不是上游、全文或语义搜索。
- `list_documents` 与 `search_documents` 每次重新读取全量索引；筛选、字段选择、offset/limit、匹配计数和 `has_more` 均为 MCP 计算。没有引入缓存系统。
- 列表/search 的 `limit` 限制为 1–50（默认 20），避免把 1728 项一次送入模型；结果保留上游顺序，不自行宣称或实现稳定日期排序。
- 四个 tools 均设置 `ToolAnnotations(read_only_hint=True, open_world_hint=False)`。annotations 只是客户端提示，真正只读边界仍由固定 URL、固定 endpoint 和 `httpx.AsyncClient.get()` 保证。

### 正文分块决策

- 初版曾计划让 `get_document` 直接返回完整详情。
- 实际索引检查发现最大条目 `word_count=238448`，真实详情 Markdown 为 243517 个字符；一次完整返回可能耗尽 MCP client/LLM 上下文。
- 因此采用最小字符分块：默认 20000、最大 50000 Unicode 字符，返回 `total_chars`、`has_more`、`next_offset`。这只做无损切片，不摘要或改写；短文仍一次完整返回。
- metadata 嵌套在 `document` 下，明确与 MCP 计算的分块字段区分。

### 上游延迟和错误处理

- 当前 `documents.json` 约 2.35 MB，冷请求观察到可能超过 30 秒，因此 read timeout 设为 60 秒、connect timeout 为 10 秒；不增加重试框架。
- 连接失败、非成功 HTTP 状态、无效 JSON、顶层或文档结构异常都通过普通异常形成 `is_error=true` tool result，不返回伪成功错误字符串。
- schema 负责类型、枚举、长度和范围；代码补充日期交叉校验、纯空白 query 和响应结构校验。

### 保守处理上游数据

- 当前 145 个 `mianji` 索引项的日期均为 `null`。日期筛选只匹配实际存在的 ISO 日期，不为播客推测日期。
- 发现当前索引首条的 title 与 ID/source_path 指向内容不一致。MCP 选择原样返回各字段，不根据某一字段重写另一字段。
- 未把观察到的数组顺序、索引完整性、更新频率或 GitHub Pages cache headers 提升为 API 保证。
- 上游的 `attribution_text` 被保留在来源、索引和详情返回中；Server Instructions 与 LLM Prompt 都要求引用时保留来源标注。
- 所有上游文字一律视为数据，不作为运行指令。

### 验证结果

- 依赖锁定、Python 编译、in-process 工具发现和参数错误检查通过。
- 三类上游 API 真实调用通过；列表、搜索、短文完整返回和超长正文分块均得到合理结果。
- HTTP 服务只监听 `127.0.0.1:8000`；伪造 Host 返回 421。
- SDK 自动发现协商 `2026-07-28`；legacy initialize 协商 `2025-11-25`；两者均成功列 tools 和调用 `get_source_info`。
- Codex CLI 0.149.0 通过临时、不持久化配置实际识别服务器并产生一次成功的 `mcp_tool_call`。最初模型 WebSocket 采样超时后自动回退 HTTPS，最终 tool 调用与回答成功。

### 文档同步

- `prompts/llm_prompt.md` 从通用示例改为四个最终 tool 的实际说明。
- 修改原因：tool 名称、参数、返回结构和上游限制已经确定，且 `get_document` 增加正文分块。
- 推荐工作流仍是“宏观阅读 → 提出假设 → 细节验证 → 循环深化”，但入口、搜索语义、日期缺失、分页/分块方式和引用要求均已具体化。
- README 已补齐安装、锁定依赖启动、自定义端口、MCP URL、Codex config/CLI 接入、tools、数据处理、限制和排查路径。

### 收尾审查

- 代码、文档和离线测试分别复核，均未发现阻断问题。
- 将搜索实现从拼接 `title`/`excerpt` 后匹配收紧为分别匹配任一字段，避免含换行 query 跨字段命中；普通关键词语义不变。
- 文档明确 `content_offset` 范围，并注明 `next_offset` 分块只有在两次请求间上游文档未变化时才能无损拼接。
- 错误说明收紧到实现实际检查的结构：顶层对象、索引对象数组和详情 Markdown 字符串，不对未校验的内层字段作保证。
