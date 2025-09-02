## Discourse MCP — Agent Guide

### What this is
- **Purpose**: An MCP (Model Context Protocol) stdio server that exposes Discourse forum capabilities as tools for AI agents.
- **Entry point**: `src/index.ts` → compiled to `dist/index.js` (binary name: `discourse-mcp`).
- **SDK**: `@modelcontextprotocol/sdk`. Node ≥ 18.

### How it works
- On start, the server validates CLI flags via Zod, constructs a dynamic site state, and registers tools on an MCP server named `@discourse/mcp`.
- Choose a target Discourse site by either:
  - Calling the `discourse_select_site` tool (validates via `/about.json`), or
  - Starting with `--site <url>` to tether to a single site (validates via `/about.json` and hides `discourse_select_site`).
- Outputs are text-oriented; some tools embed compact JSON in fenced code blocks for structured extraction.

### Authentication & permissions
- Supported auth:
  - **None** (read-only public data)
  - Per-site overrides via `--auth_pairs`, e.g. `[{"site":"https://example.com","api_key":"...","api_username":"system"}]`.
- **Writes are disabled by default**. Write tools (`discourse_create_post`, `discourse_create_topic`, `discourse_create_category`, `discourse_create_user`) are only registered when all are true:
  - `--allow_writes` AND not `--read_only` AND a matching `auth_pairs` entry exists for the selected site.
- Secrets are never logged; config is redacted before logging.

### Tools exposed (built-in)
- **discourse_search**
  - **Input**: `{ query: string; with_private?: boolean; max_results?: number (1–50, default 10) }`
  - **Output**: Top topics with titles and URLs; appends a JSON footer of `{ results: [{ id, url, title }] }` inside a fenced block.
- **discourse_read_topic**
  - **Input**: `{ topic_id: number; post_limit?: number (1–20, default 5); start_post_number?: number }`
  - **Output**: Title, category, tags, and the first N posts (raw preferred when available) up to the configured max read length per post; includes canonical topic link.
- **discourse_read_post**
  - **Input**: `{ post_id: number }`
  - **Output**: Author, timestamp, content (raw preferred) truncated to the configured max read length, and a direct link.
- **discourse_list_categories**
  - **Input**: `{}`
  - **Output**: Category names with topic counts.
- **discourse_list_tags**
  - **Input**: `{}`
  - **Output**: Tags with usage counts (or notice if tags are disabled).
- **discourse_get_user**
  - **Input**: `{ username: string }`
  - **Output**: Display name, trust level, joined date, short bio, and profile link.
- **discourse_filter_topics**
  - **Input**: `{ filter: string; page?: number; per_page?: number (1–50) }`
  - **Output**: Paginated topic list with titles and URLs; appends a JSON footer `{ page, per_page, results: [{ id, url, title }], next_url? }`.
  - Query language: key:value tokens separated by spaces; category/categories (comma = OR, `=category` = without subcats, `-` exclude); tag/tags (comma = OR, `+` = AND) and tag_group; status:(open|closed|archived|listed|unlisted|public); personal `in:` (bookmarked|watching|tracking|muted|pinned); dates created/activity/latest-post-(before|after) as `YYYY-MM-DD` or `N` days; numeric likes[-op]-(min|max), posts-(min|max), posters-(min|max), views-(min|max); `order:` with optional `-asc`; free text terms allowed.
- **discourse_create_post** (conditionally available; see permissions)
  - **Input**: `{ topic_id: number; raw: string (≤ 30k chars) }`
  - **Output**: Link to created post/topic. Includes a simple 1 req/sec rate limit.
- **discourse_create_topic** (conditionally available; see permissions)
  - **Input**: `{ title: string; raw: string (≤ 30k chars); category_id?: number; tags?: string[] }`
  - **Output**: Link to created topic. Includes a simple 1 req/sec rate limit.
- **discourse_create_category** (conditionally available; see permissions)
  - **Input**: `{ name: string; color?: hex; text_color?: hex; parent_category_id?: number; description?: string }`
  - **Output**: Link to created category. Includes a simple 1 req/sec rate limit.
- **discourse_select_site** (hidden when `--site` is provided)
  - **Input**: `{ site: string }`
  - **Output**: Confirms selection; validates via `/about.json`. Triggers remote tool discovery when enabled.

### Remote Tool Execution API (optional)
- If the target Discourse site exposes an MCP-compatible Tool Execution API:
  - GET `/ai/tools` is discovered after selecting a site when `tools_mode` is `auto` (default) or `tool_exec_api` (or immediately at startup if `--site` is provided).
  - Each remote tool is registered dynamically using its JSON Schema input.
  - Calls POST `/ai/tools/{name}/call` with `{ arguments, context: {} }`.
  - Results may include `details.artifacts[]`; links are surfaced at the end of the tool output.
- Set `--tools_mode=discourse_api_only` to disable remote tool discovery.

### CLI configuration
- **Optional flags**:
  - `--auth_pairs` (JSON)
  - `--read_only` (default true), `--allow_writes` (default false)
  - `--timeout_ms <number>` (default 15000)
  - `--concurrency <number>` (default 4)
  - `--cache_dir <path>` (currently unused; in-memory caching is built-in)
  - `--log_level <silent|error|info|debug>` (default info)
  - `--tools_mode <auto|discourse_api_only|tool_exec_api>` (default auto)
  - `--profile <path.json>`: load partial config from JSON (flags override)
  - `--site <url>`: tether to a single site (hides `discourse_select_site`)
  - `--default-search <prefix>`: unconditionally prefix every search query (e.g., `category:support tag:ai`)
  - `--max-read-length <number>` (default 50000): maximum number of characters returned for post content in `discourse_read_post` and per-post content in `discourse_read_topic`. Tools prefer `raw` content via Discourse API (`include_raw=true`) when available.

### Networking & resilience
- User-Agent: `Discourse-MCP/0.x (+https://github.com/discourse-mcp)`.
- Retries on 429/5xx with backoff (3 attempts).
- Lightweight in-memory GET cache for selected endpoints (e.g., topics, site metadata).

### Errors & rate limits
- Tool failures return `isError: true` with human-readable messages.
- `discourse.create_post` and `discourse.create_category` enforce ~1 request/second to avoid flooding.

### Source map
- MCP server and CLI: `src/index.ts`
- HTTP client: `src/http/client.ts`
- Tool registry: `src/tools/registry.ts`
- Built-in tools: `src/tools/builtin/*`
- Remote tools: `src/tools/remote/tool_exec_api.ts`
- Logging/redaction: `src/util/logger.ts`, `src/util/redact.ts`

### Quick start (for human operators)
- Build: `pnpm build`
- Run: `node dist/index.js`
- Select site with `discourse_select_site` in your client
