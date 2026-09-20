# Data Formulator 源代码说明（SOURCE）


## 1. 源代码整体介绍

### 1.1 项目名称与性质

本仓库是 **Microsoft Data Formulator** 的完整源码：一个以 Python 为服务端、以 TypeScript/React
为客户端的 AI 数据可视化应用。发行形态包括 PyPI 包 `data_formulator`、Docker 镜像、以及
CI 构建的 Windows/macOS 桌面包。许可证为 MIT。当前开发版本号在 `pyproject.toml` 中为 **0.8.0b1**。

### 1.2 使用的程序语言

| 语言 | 用途 | 主要目录 |
|------|------|----------|
| Python 3.11+ | 后端、Agent、加载器、沙箱、测试 | `py-src/`, `tests/backend/` |
| TypeScript | 前端类型与逻辑 | `src/`, `tests/frontend/` |
| TSX | React 组件 | `src/views`, `src/components`, `src/app` |
| CSS / SCSS | 全局与组件样式 | `src/index.css`, `src/scss` |
| JSON | i18n、技能 tools schema、示例数据 | `src/i18n`, `analyst/skills/*/tools.json` |
| Markdown | 技能说明书、开发指南 | `SKILL.md`, `docs/` |
| YAML | 工作区元数据、CI | `workspace.yaml` 运行时, `.github/` |
| Dockerfile | 应用镜像与沙箱镜像 | `Dockerfile`, `sandbox/Dockerfile.sandbox` |
| Shell / Bat | 本地启动 | `local_server.sh`, `local_server.bat` |

后端核心依赖：Flask、pandas、DuckDB、LiteLLM、PyArrow、OpenAI SDK、各类 DB 驱动、
PyJWT、flask-session。前端核心依赖：React 18、Redux Toolkit、MUI、flint-chart、Vega/Vega-Lite、
i18next、TipTap、oidc-client-ts、Vite 7、Vitest。

### 1.3 开发工具与命令

- 包管理：Python 用 **uv**（lockfile `uv.lock`）或 pip；前端强制 **Yarn v1.22.22**（禁止 npm/pnpm 改锁文件）。
- 构建：`yarn build`（Vite）→ 静态资源进 Python 包；`uv build` / `python -m build` 出 wheel。
- 测试：`python -m pytest tests/backend -q`；`yarn test`（vitest run）。
- 质量：ESLint `eslint.config.js`；Python 无强制 black 配置于根，但测试即规格。
- IDE：`.vscode/`、`.cursor/rules` 与 skills 约束 AI 与人类同样的编码法。
- 容器：Docker Compose；数据库集成测试在 `tests/database-dockers`。
- 桌面：PyInstaller + pywebview（optional extra `desktop`）。

### 1.4 规模统计（基于当前工作区静态扫描）

下列数字来自对 `py-src`、`src`、`tests` 的文件扫描，不含 `node_modules` 与构建产物。


- 扫描到的源与测试文本文件数：**487**

- 上述文件总行数：**150029**

- 后端 Python 文件数：**125**，行数 **46681**

- 前端 TS/TSX/CSS 文件数：**116**，行数 **52153**

- 后端测试 Python 文件数：**141**，行数 **29796**

- 前端测试 TS 文件数：**46**，行数 **6391**

- Python 模块 AST 统计：类 **107**，模块级函数 **550**，方法 **859**

- 前端导出符号约 **551** 个

- pytest `test_*` 函数约 **1837** 个

- vitest `it(...)` 约 **378** 个


这些规模意味着：DF 已不是“研究原型小脚本”，而是带有完整认证、多后端存储、十几种数据源、
统一 Agent 与大量回归测试的中型全栈系统。阅读源码建议从 `app.py` 与 `analyst/agent.py`、
`src/app/dfSlice.tsx` 三条主线切入。

### 1.5 目录地图

```
/workspace
  py-src/data_formulator/   Python 包根
  src/                      前端源码
  tests/                    测试
  docs/                     用户/开发文档
  public/                   静态示例 json
  packaging/                打包相关
  .cursor/                  AI 规则与技能
  .github/                  CI
```

### 1.6 编码约定索引

- 错误：`docs/dev-guides/7-unified-error-handling.md`
- 流式：`1-streaming-protocol.md`
- 路径：`8-path-safety.md`
- 日志：`2-log-sanitization.md`
- 加载器：`3-data-loader-development.md`
- 认证：`4-authentication-oidc-tokenstore.md`
- i18n：`6-i18n-language-injection.md`
- 知识：`10-agent-knowledge-reasoning-log.md`
- 目录同步：`11-catalog-metadata-sync.md`
- 沙箱会话：`12-sandbox-session.md`
- 行数：`13-unified-row-limits.md`
- 模型降级：`14-model-capability-runtime-degradation.md`
- DataFrame 序列化：`15-dataframe-serialization.md`

阅读源码时把对应指南当作该子系统的“宪法”。

## 2. 源代码文件清单

下列清单按路径排序。对每个文件给出：所在目录、行数、主要功能、以及从 AST/export 提取的符号摘要。
这是维护者级地图，可用于 onboarding 与影响分析。


### 2.1 后端 Python 文件

#### `py-src/data_formulator/__init__.py`

- **目录**：`py-src/data_formulator`

- **行数**：12

- **所在子系统**：应用入口、Flask 装配、错误类型、连接器总控、模型注册、工作区工厂

- **主要功能简介**：该文件实现子系统中的一个具体单元，符号列表如下。

- **类**：无

- **函数**：run_app

- **常量**：无

- **阅读建议**：先看公开类的 `run`/`handle_*`/`to_dict`，再看私有辅助。改动后运行对应 `tests/backend` 文件。涉及用户字符串或错误码时同步 i18n。

#### `py-src/data_formulator/__main__.py`

- **目录**：`py-src/data_formulator`

- **行数**：4

- **所在子系统**：应用入口、Flask 装配、错误类型、连接器总控、模型注册、工作区工厂

- **主要功能简介**：该文件实现子系统中的一个具体单元，符号列表如下。

- **类**：无

- **函数**：无

- **常量**：无

- **阅读建议**：先看公开类的 `run`/`handle_*`/`to_dict`，再看私有辅助。改动后运行对应 `tests/backend` 文件。涉及用户字符串或错误码时同步 i18n。

#### `py-src/data_formulator/_startup_spinner.py`

- **目录**：`py-src/data_formulator`

- **行数**：79

- **所在子系统**：应用入口、Flask 装配、错误类型、连接器总控、模型注册、工作区工厂

- **主要功能简介**：Minimal startup spinner.  Animates a single line on a TTY while a slow import / setup step runs. Falls back to plain prints in non-TTY environments (gunicorn, Docker logs, CI, redirected stdout) so log files stay clean.  Usage:     with spinner("Loading AI agents"):         from data_formulator.routes.agents import agent_bp

- **类**：无

- **函数**：_enabled, _color, spinner

- **常量**：_FRAMES, _FRAME_INTERVAL, _INDENT

- **阅读建议**：先看公开类的 `run`/`handle_*`/`to_dict`，再看私有辅助。改动后运行对应 `tests/backend` 文件。涉及用户字符串或错误码时同步 i18n。

#### `py-src/data_formulator/agent_config.py`

- **目录**：`py-src/data_formulator`

- **行数**：152

- **所在子系统**：应用入口、Flask 装配、错误类型、连接器总控、模型注册、工作区工厂

- **主要功能简介**：Single source of truth for per-agent LLM call configuration.  Edit values here to tune latency vs. quality for each agent.  Per-agent overrides can also be set at runtime via environment variables:      DF_REASONING_EFFORT_DATA_TRANSFORM=medium     DF_REASONING_EFFORT_REPORT_GEN=high  Tiers ----- - ``"minimal"`` — fastest. Honoured natively only on the OpenAI GPT-5   base/mini/nano/5.x family (``g

- **类**：无

- **函数**：get_reasoning_effort, _supports_minimal, _supports_none, reasoning_effort_for

- **常量**：无

- **阅读建议**：先看公开类的 `run`/`handle_*`/`to_dict`，再看私有辅助。改动后运行对应 `tests/backend` 文件。涉及用户字符串或错误码时同步 i18n。

#### `py-src/data_formulator/agents/__init__.py`

- **目录**：`py-src/data_formulator/agents`

- **行数**：14

- **所在子系统**：专用 Agent、LLM Client、上下文拼装、语义类型、推理日志、网页抓取

- **主要功能简介**：该文件实现子系统中的一个具体单元，符号列表如下。

- **类**：无

- **函数**：无

- **常量**：无

- **阅读建议**：先看公开类的 `run`/`handle_*`/`to_dict`，再看私有辅助。改动后运行对应 `tests/backend` 文件。涉及用户字符串或错误码时同步 i18n。

#### `py-src/data_formulator/agents/agent_chart_restyle.py`

- **目录**：`py-src/data_formulator/agents`

- **行数**：362

- **所在子系统**：专用 Agent、LLM Client、上下文拼装、语义类型、推理日志、网页抓取

- **主要功能简介**：Chart restyle agent.  A single-turn agent that takes a Vega-Lite spec + a natural-language instruction and returns a modified Vega-Lite spec representing the same chart with the requested style changes applied.  The only HARD rule is that the agent must not touch the spec's `data` block — the caller strips data on input and re-attaches the live rows on output, so the data values, columns, and colu

- **类**：ChartRestyleAgent

- **函数**：无

- **常量**：_AGENT_ID, SYSTEM_PROMPT

  - `ChartRestyleAgent` 方法：__init__, run, _sanitize_config_ui, _enforce_guardrails, _collect_field_bindings

- **阅读建议**：先看公开类的 `run`/`handle_*`/`to_dict`，再看私有辅助。改动后运行对应 `tests/backend` 文件。涉及用户字符串或错误码时同步 i18n。

#### `py-src/data_formulator/agents/agent_code_explanation.py`

- **目录**：`py-src/data_formulator/agents`

- **行数**：222

- **所在子系统**：专用 Agent、LLM Client、上下文拼装、语义类型、推理日志、网页抓取

- **主要功能简介**：该文件实现子系统中的一个具体单元，符号列表如下。

- **类**：CodeExplanationAgent

- **函数**：无

- **常量**：_AGENT_ID, SYSTEM_PROMPT, EXAMPLE

  - `CodeExplanationAgent` 方法：__init__, run

- **阅读建议**：先看公开类的 `run`/`handle_*`/`to_dict`，再看私有辅助。改动后运行对应 `tests/backend` 文件。涉及用户字符串或错误码时同步 i18n。

#### `py-src/data_formulator/agents/agent_data_load.py`

- **目录**：`py-src/data_formulator/agents`

- **行数**：237

- **所在子系统**：专用 Agent、LLM Client、上下文拼装、语义类型、推理日志、网页抓取

- **主要功能简介**：该文件实现子系统中的一个具体单元，符号列表如下。

- **类**：DataLoadAgent

- **函数**：无

- **常量**：_AGENT_ID, SYSTEM_PROMPT, EXAMPLES

  - `DataLoadAgent` 方法：__init__, run

- **阅读建议**：先看公开类的 `run`/`handle_*`/`to_dict`，再看私有辅助。改动后运行对应 `tests/backend` 文件。涉及用户字符串或错误码时同步 i18n。

#### `py-src/data_formulator/agents/agent_data_loading_chat.py`

- **目录**：`py-src/data_formulator/agents`

- **行数**：2378

- **所在子系统**：专用 Agent、LLM Client、上下文拼装、语义类型、推理日志、网页抓取

- **主要功能简介**：Conversational data loading agent.  General-purpose conversational agent that can: - Extract tables from images / text / files - Execute Python code in a sandboxed environment - Show inline table previews - Prepare tables for user-confirmed loading

- **类**：DataLoadingAgent

- **函数**：_secure_filename, _unique_scratch_filename, _summarize_catalog_shape, _build_connector_summary_block

- **常量**：_AGENT_ID, PROBE_TURN_BUDGET, SYSTEM_PROMPT, TOOLS

  - `DataLoadingAgent` 方法：__init__, stream, _agentic_loop, _forced_summary_turn, _call_llm, _execute_tool, _tool_read_data_memory, _tool_append_data_memory, _tool_replace_data_memory, _tool_read_file, _tool_write_file, _tool_list_directory, _tool_execute_python, _tool_fetch_url, _tool_show_user_data_preview, _preview_saved_dfs, _preview_inline_tables, _preview_scratch_files, _tool_list_data, _tool_find_data, _tool_describe_data, _resolve_catalog_path, _tool_probe_data, _tool_propose_load_plan, _connectors_disabled, _tool_list_connectors, _tool_describe_connector, _tool_propose_connection, _normalize_load_plan_candidate, _known_source_ids

- **阅读建议**：先看公开类的 `run`/`handle_*`/`to_dict`，再看私有辅助。改动后运行对应 `tests/backend` 文件。涉及用户字符串或错误码时同步 i18n。

#### `py-src/data_formulator/agents/agent_diagnostics.py`

- **目录**：`py-src/data_formulator/agents`

- **行数**：147

- **所在子系统**：专用 Agent、LLM Client、上下文拼装、语义类型、推理日志、网页抓取

- **主要功能简介**：Unified diagnostics builder for all agent pipelines.  Centralises the JSON structure returned as ``result['diagnostics']``, ensuring a single schema definition for both back-end construction and front-end consumption (DiagnosticsViewer in MessageSnackbar.tsx).

- **类**：AgentDiagnostics

- **函数**：_now

- **常量**：无

  - `AgentDiagnostics` 方法：__init__, _base, for_error, for_response, for_json_only

- **阅读建议**：先看公开类的 `run`/`handle_*`/`to_dict`，再看私有辅助。改动后运行对应 `tests/backend` 文件。涉及用户字符串或错误码时同步 i18n。

#### `py-src/data_formulator/agents/agent_language.py`

- **目录**：`py-src/data_formulator/agents`

- **行数**：188

- **所在子系统**：专用 Agent、LLM Client、上下文拼装、语义类型、推理日志、网页抓取

- **主要功能简介**：Language instruction builder for Agent prompts.  Generates a prompt fragment that constrains LLM output language for user-visible fields while keeping all internal / programmatic fields stable in English.  Two modes are provided:  - **"full"** — detailed field-by-field rules for text-heavy agents   (ChartInsight, InteractiveExplore, ReportGen, CodeExplanation,   DataClean, DataAgent). - **"compact

- **类**：无

- **函数**：inject_language_instruction, build_language_instruction, _build_compact, _build_full

- **常量**：DEFAULT_LANGUAGE

- **阅读建议**：先看公开类的 `run`/`handle_*`/`to_dict`，再看私有辅助。改动后运行对应 `tests/backend` 文件。涉及用户字符串或错误码时同步 i18n。

#### `py-src/data_formulator/agents/agent_simple.py`

- **目录**：`py-src/data_formulator/agents`

- **行数**：238

- **所在子系统**：专用 Agent、LLM Client、上下文拼装、语义类型、推理日志、网页抓取

- **主要功能简介**：Lightweight single-turn agents that wrap a system prompt + one LLM call.  Each method takes a ``Client`` instance plus task-specific parameters and returns a plain dict result (no streaming, no workspace access).

- **类**：SimpleAgents

- **函数**：无

- **常量**：_AGENT_ID, _NL_FILTER_SYSTEM_PROMPT, _WORKSPACE_NAME_SYSTEM_PROMPT, _CHART_INTENT_SYSTEM_PROMPT

  - `SimpleAgents` 方法：__init__, nl_to_filter, workspace_name, classify_chart_intent

- **阅读建议**：先看公开类的 `run`/`handle_*`/`to_dict`，再看私有辅助。改动后运行对应 `tests/backend` 文件。涉及用户字符串或错误码时同步 i18n。

#### `py-src/data_formulator/agents/agent_sort_data.py`

- **目录**：`py-src/data_formulator/agents`

- **行数**：125

- **所在子系统**：专用 Agent、LLM Client、上下文拼装、语义类型、推理日志、网页抓取

- **主要功能简介**：该文件实现子系统中的一个具体单元，符号列表如下。

- **类**：SortDataAgent

- **函数**：无

- **常量**：_AGENT_ID, SYSTEM_PROMPT

  - `SortDataAgent` 方法：__init__, run

- **阅读建议**：先看公开类的 `run`/`handle_*`/`to_dict`，再看私有辅助。改动后运行对应 `tests/backend` 文件。涉及用户字符串或错误码时同步 i18n。

#### `py-src/data_formulator/agents/agent_starter_questions.py`

- **目录**：`py-src/data_formulator/agents`

- **行数**：120

- **所在子系统**：专用 Agent、LLM Client、上下文拼装、语义类型、推理日志、网页抓取

- **主要功能简介**：该文件实现子系统中的一个具体单元，符号列表如下。

- **类**：StarterQuestionsAgent

- **函数**：无

- **常量**：_AGENT_ID, SYSTEM_PROMPT

  - `StarterQuestionsAgent` 方法：__init__, run

- **阅读建议**：先看公开类的 `run`/`handle_*`/`to_dict`，再看私有辅助。改动后运行对应 `tests/backend` 文件。涉及用户字符串或错误码时同步 i18n。

#### `py-src/data_formulator/agents/agent_utils.py`

- **目录**：`py-src/data_formulator/agents`

- **行数**：794

- **所在子系统**：专用 Agent、LLM Client、上下文拼装、语义类型、推理日志、网页抓取

- **主要功能简介**：该文件实现子系统中的一个具体单元，符号列表如下。

- **类**：无

- **函数**：compose_system_prompt, attach_reasoning_content, accumulate_reasoning_content, _source_table_matches_catalog_entry, build_catalog_metadata_lookups, format_dataframe_sample_with_budget, string_to_py_varname, field_name_to_ts_variable_name, infer_ts_datatype, value_handling_func, table_hash, extract_code_from_gpt_response, find_matching_bracket, _strip_json_comments, _fix_json_trailing_commas, _lenient_json_loads, extract_json_objects, supplement_missing_block, get_field_summary, _format_import_options, generate_data_summary, ensure_output_variable_in_code

- **常量**：无

- **阅读建议**：先看公开类的 `run`/`handle_*`/`to_dict`，再看私有辅助。改动后运行对应 `tests/backend` 文件。涉及用户字符串或错误码时同步 i18n。

#### `py-src/data_formulator/agents/agent_utils_sql.py`

- **目录**：`py-src/data_formulator/agents`

- **行数**：39

- **所在子系统**：专用 Agent、LLM Client、上下文拼装、语义类型、推理日志、网页抓取

- **主要功能简介**：SQL-related utility functions for agents. These functions are used across multiple agents for DuckDB operations and SQL data summaries.

- **类**：无

- **函数**：sanitize_table_name, create_duckdb_conn_with_parquet_views

- **常量**：无

- **阅读建议**：先看公开类的 `run`/`handle_*`/`to_dict`，再看私有辅助。改动后运行对应 `tests/backend` 文件。涉及用户字符串或错误码时同步 i18n。

#### `py-src/data_formulator/agents/agent_workflow_distill.py`

- **目录**：`py-src/data_formulator/agents`

- **行数**：510

- **所在子系统**：专用 Agent、LLM Client、上下文拼装、语义类型、推理日志、网页抓取

- **主要功能简介**：Workflow distillation agent — extracts a replayable workflow from analysis context.  Given a user-visible analysis context (timeline of events) plus an optional user instruction, this agent calls an LLM to produce a structured Markdown workflow document with YAML front matter suitable for storage in the knowledge base.  Usage::      agent = WorkflowDistillAgent(client)     md_content = agent.run(w

- **类**：WorkflowDistillAgent

- **函数**：无

- **常量**：_AGENT_ID, SYSTEM_PROMPT

  - `WorkflowDistillAgent` 方法：__init__, run, _prompt_format_kwargs, _call_with_length_retry, _truncate_body_to_limit, _truncate, _truncate_code, _render_sample, _extract_context_summary, _render_events, _call_llm, _add_fallback_front_matter

- **阅读建议**：先看公开类的 `run`/`handle_*`/`to_dict`，再看私有辅助。改动后运行对应 `tests/backend` 文件。涉及用户字符串或错误码时同步 i18n。

#### `py-src/data_formulator/agents/client_utils.py`

- **目录**：`py-src/data_formulator/agents`

- **行数**：452

- **所在子系统**：专用 Agent、LLM Client、上下文拼装、语义类型、推理日志、网页抓取

- **主要功能简介**：该文件实现子系统中的一个具体单元，符号列表如下。

- **类**：Client

- **函数**：_synthesize_stream, _extract_json_objects, _match_tool_from_obj, _salvage_tool_calls_from_content

- **常量**：无

  - `Client` 方法：__init__, _strip_image_blocks, _strip_images_from_messages, _messages_contain_images, _is_image_deserialize_error, _is_reasoning_effort_error, from_config, ping, _dispatch, get_completion, get_completion_with_tools

- **阅读建议**：先看公开类的 `run`/`handle_*`/`to_dict`，再看私有辅助。改动后运行对应 `tests/backend` 文件。涉及用户字符串或错误码时同步 i18n。

#### `py-src/data_formulator/agents/context.py`

- **目录**：`py-src/data_formulator/agents`

- **行数**：459

- **所在子系统**：专用 Agent、LLM Client、上下文拼装、语义类型、推理日志、网页抓取

- **主要功能简介**：Shared context builders for agent prompts.  Extracted from DataAgent so that both DataAgent and InteractiveExploreAgent can construct tiered context (primary/other tables, focused thread, peripheral threads) from the same code.

- **类**：无

- **函数**：_get_workspace_metadata_lookups, build_focused_thread_context, build_peripheral_thread_context, _table_label, _client_schema_section, build_lightweight_table_context, handle_inspect_source_data, _fetch_live_metadata, handle_read_catalog_metadata

- **常量**：TABLE_SAMPLE_MAX_ROWS, TABLE_SAMPLE_CHAR_LIMIT

- **阅读建议**：先看公开类的 `run`/`handle_*`/`to_dict`，再看私有辅助。改动后运行对应 `tests/backend` 文件。涉及用户字符串或错误码时同步 i18n。

#### `py-src/data_formulator/agents/reasoning_log.py`

- **目录**：`py-src/data_formulator/agents`

- **行数**：264

- **所在子系统**：专用 Agent、LLM Client、上下文拼装、语义类型、推理日志、网页抓取

- **主要功能简介**：Structured reasoning logger for Agent sessions.  Each ``ReasoningLogger`` instance is bound to one Agent session and writes a JSONL file under ``DATA_FORMULATOR_HOME/agent-logs/<date>/<safe_identity_id>/<session_id>-<agent_type>.jsonl``.  The log level is controlled by the **DF_AGENT_LOG** environment variable:      off      – no-op; no file created, no I/O     on       – structured summaries (cou

- **类**：ReasoningLogger, _NullReasoningLogger

- **函数**：_today_str, _utc_now_iso, _parse_log_level, get_agent_logs_root, _safe_log_filename, _cleanup_expired_logs

- **常量**：_LOG_RETENTION_DAYS, _ON_FILTERED_KEYS

  - `ReasoningLogger` 方法：__init__, log, close, __enter__, __exit__, _ensure_fd_for_today, _sanitize_verbose

  - `_NullReasoningLogger` 方法：__init__, log, close

- **阅读建议**：先看公开类的 `run`/`handle_*`/`to_dict`，再看私有辅助。改动后运行对应 `tests/backend` 文件。涉及用户字符串或错误码时同步 i18n。

#### `py-src/data_formulator/agents/semantic_types.py`

- **目录**：`py-src/data_formulator/agents`

- **行数**：463

- **所在子系统**：专用 Agent、LLM Client、上下文拼装、语义类型、推理日志、网页抓取

- **主要功能简介**：============================================================================= SEMANTIC TYPE SYSTEM  (Python mirror of the TypeScript registry) =============================================================================  The **source of truth** for semantic types lives in the flint-chart library (npm package `flint-chart`, repo microsoft/flint-chart):     packages/flint-js/src/core/type-registry.

- **类**：无

- **函数**：is_measure_type, is_timeseries_type, is_categorical_type, is_ordinal_type, is_geo_type, is_non_measure_numeric, is_signed_measure, generate_semantic_types_prompt, get_vl_type, infer_vl_type_from_name

- **常量**：DATETIME, DATE, TIME, TIMESTAMP, YEAR, QUARTER, MONTH, WEEK, DAY, HOUR, YEAR_MONTH, YEAR_QUARTER, YEAR_WEEK, DECADE, DURATION, AMOUNT, PRICE, QUANTITY, TEMPERATURE, PERCENTAGE, PROFIT, PERCENTAGE_CHANGE, SENTIMENT, CORRELATION, COUNT …

- **阅读建议**：先看公开类的 `run`/`handle_*`/`to_dict`，再看私有辅助。改动后运行对应 `tests/backend` 文件。涉及用户字符串或错误码时同步 i18n。

#### `py-src/data_formulator/agents/web_utils.py`

- **目录**：`py-src/data_formulator/agents`

- **行数**：530

- **所在子系统**：专用 Agent、LLM Client、上下文拼装、语义类型、推理日志、网页抓取

- **主要功能简介**：该文件实现子系统中的一个具体单元，符号列表如下。

- **类**：无

- **函数**：_is_private_ip, _validate_url_for_ssrf, download_html_content, html_to_text, get_html_title, get_html_meta_description, _configured_max_fetch_bytes, _ssrf_safe_session, fetch_url_bytes, extract_tables_from_html, playwright_available, is_verification_challenge, render_url_with_playwright

- **常量**：DEFAULT_MAX_FETCH_BYTES, _BROWSER_HEADERS, _CHALLENGE_MARKERS

- **阅读建议**：先看公开类的 `run`/`handle_*`/`to_dict`，再看私有辅助。改动后运行对应 `tests/backend` 文件。涉及用户字符串或错误码时同步 i18n。

#### `py-src/data_formulator/analyst/__init__.py`

- **目录**：`py-src/data_formulator/analyst`

- **行数**：50

- **所在子系统**：统一 AnalystAgent 外壳与工具工厂

- **主要功能简介**：Analyst agent — a single user-facing data agent hosting multiple skills.  This package unifies the former ``DataAgent`` (structured-action visualization loop) and ``ReportGenAgent`` (streaming report writer) into one agent shell that loads *skills* on demand. See ``design-docs/35-unified-agent-skills- architecture.md`` for the full design.  Core ideas:   - **Inspection tools** gather information a

- **类**：无

- **函数**：无

- **常量**：无

- **阅读建议**：先看公开类的 `run`/`handle_*`/`to_dict`，再看私有辅助。改动后运行对应 `tests/backend` 文件。涉及用户字符串或错误码时同步 i18n。

#### `py-src/data_formulator/analyst/agent.py`

- **目录**：`py-src/data_formulator/analyst`

- **行数**：2157

- **所在子系统**：统一 AnalystAgent 外壳与工具工厂

- **主要功能简介**：AnalystAgent — the unified data analyst agent shell.  This is the single user-facing data agent that replaces the separate ``DataAgent`` (structured-action visualization loop) and ``ReportGenAgent`` (streaming report writer). It hosts a set of **core actions** plus a registry of **skills** that unlock additional **gated actions** on demand. See ``design-docs/35-unified-agent-skills-architecture.md

- **类**：_StreamingArgExtractor, AnalystAgent

- **函数**：_rescue_unpack_json_strings

- **常量**：_AGENT_ID, _CORE_SKILL, _SKILL_LOADED_BANNER, _SKILL_LOADED_RE, SYSTEM_PROMPT

  - `_StreamingArgExtractor` 方法：__init__, feed, _decode

  - `AnalystAgent` 方法：__init__, _explore_ns_dir, _legal_actions, run, _rehydrate_loaded_skills, _load_skill_into_context, _build_skill_body_message, _dispatch_skill_action, _route_skill_events, _set_action_observation, run_visualize_code, register_run_chart, run_explore_code, _run_explore_code, _run_visualize_code, _build_system_prompt, _build_initial_messages, _build_focused_thread_context, _build_peripheral_thread_context, _build_available_charts_context, _build_lightweight_table_context, _get_next_action, _current_tools, _loaded_skill_tool_map, _tool_loop, _commit_action, _is_transient_error, _open_stream, _stream_llm, _forward_stream_delta

- **阅读建议**：先看公开类的 `run`/`handle_*`/`to_dict`，再看私有辅助。改动后运行对应 `tests/backend` 文件。涉及用户字符串或错误码时同步 i18n。

#### `py-src/data_formulator/analyst/skills/__init__.py`

- **目录**：`py-src/data_formulator/analyst/skills`

- **行数**：394

- **所在子系统**：技能注册表与协议类型

- **主要功能简介**：Skill registry — discovery and eager instantiation of analyst skills.  Each skill lives in its own sub-package under this directory and ships a ``SKILL.md`` with YAML frontmatter (``name`` / ``description`` / ``when_to_use`` / ``always_on`` / ``actions``). At startup the registry scans those frontmatter blocks to build a cheap, always-resident index (tier-1 progressive disclosure) **and** imports 

- **类**：SkillRegistry

- **函数**：_parse_front_matter, _coerce_name_list, _meta_from_frontmatter, _instantiate_skill, _load_tool_specs, build_registry, _warn_on_name_collisions

- **常量**：SKILLS_DIR, SKILL_DOC_NAME, TOOLS_FILE_NAME, _FM_PATTERN

  - `SkillRegistry` 方法：canonical_name, _specs_split, names, list_metas, has, gated_skill_names, action_owner, render_registry_block, load_body, get_skill, tools_for, action_tools_for, action_required_fields, action_names, action_stream_spec

- **阅读建议**：先看公开类的 `run`/`handle_*`/`to_dict`，再看私有辅助。改动后运行对应 `tests/backend` 文件。涉及用户字符串或错误码时同步 i18n。

#### `py-src/data_formulator/analyst/skills/base.py`

- **目录**：`py-src/data_formulator/analyst/skills`

- **行数**：186

- **所在子系统**：技能注册表与协议类型

- **主要功能简介**：Skill protocol and shared types for the analyst agent.  A *skill* is a passive plugin the single analyst agent can switch on. It never runs its own agent loop; instead it contributes:   1. a ``SKILL.md`` doc (frontmatter + how-to body) — progressive disclosure,   2. zero or more **tools** the model may call once the skill is loaded,   3. zero or more **gated actions** it unlocks, and   4. **handle

- **类**：SkillMeta, SkillContext, ToolResult, Skill

- **函数**：无

- **常量**：无

  - `SkillMeta` 方法：（无）

  - `SkillContext` 方法：（无）

  - `ToolResult` 方法：（无）

  - `Skill` 方法：handle_tool, handle_action

- **阅读建议**：先看公开类的 `run`/`handle_*`/`to_dict`，再看私有辅助。改动后运行对应 `tests/backend` 文件。涉及用户字符串或错误码时同步 i18n。

#### `py-src/data_formulator/analyst/skills/core/__init__.py`

- **目录**：`py-src/data_formulator/analyst/skills/core`

- **行数**：9

- **所在子系统**：核心技能：探查工具与 visualize / ask_user

- **主要功能简介**：core skill — always-on baseline tools + actions for the analyst.  ``SKILL.md`` holds the base prompt body (the shell formats it into the system message); ``skill.py`` exposes ``get_skill()`` (the executable handler).

- **类**：无

- **函数**：无

- **常量**：无

- **阅读建议**：先看公开类的 `run`/`handle_*`/`to_dict`，再看私有辅助。改动后运行对应 `tests/backend` 文件。涉及用户字符串或错误码时同步 i18n。

#### `py-src/data_formulator/analyst/skills/core/skill.py`

- **目录**：`py-src/data_formulator/analyst/skills/core`

- **行数**：346

- **所在子系统**：核心技能：探查工具与 visualize / ask_user

- **主要功能简介**：core skill — the analyst's always-on baseline capabilities.  Every other skill is optional and gated; ``core`` is ``always_on`` and loaded automatically at the start of each run, so the agent is never truly empty. It contributes the built-in data-inspection **tools** (``explore`` / ``inspect_source_data`` — ``load_skill`` is assembled by the shell because its enum is dynamic) and the always-availa

- **类**：CoreSkill

- **函数**：get_skill

- **常量**：无

  - `CoreSkill` 方法：handle_tool, handle_action, _handle_visualize, _handle_interact, _format_observation, _sanitize_clarification_options, _sanitize_clarification_questions, _normalize_interact_action

- **阅读建议**：先看公开类的 `run`/`handle_*`/`to_dict`，再看私有辅助。改动后运行对应 `tests/backend` 文件。涉及用户字符串或错误码时同步 i18n。

#### `py-src/data_formulator/analyst/skills/data-loading/__init__.py`

- **目录**：`py-src/data_formulator/analyst/skills/data-loading`

- **行数**：1

- **所在子系统**：数据加载技能：发现工具与不可变加载计划

- **主要功能简介**：Analyst data-loading skill package.

- **类**：无

- **函数**：无

- **常量**：无

- **阅读建议**：先看公开类的 `run`/`handle_*`/`to_dict`，再看私有辅助。改动后运行对应 `tests/backend` 文件。涉及用户字符串或错误码时同步 i18n。

#### `py-src/data_formulator/analyst/skills/data-loading/skill.py`

- **目录**：`py-src/data_formulator/analyst/skills/data-loading`

- **行数**：362

- **所在子系统**：数据加载技能：发现工具与不可变加载计划

- **主要功能简介**：该文件实现子系统中的一个具体单元，符号列表如下。

- **类**：DataLoadingSkill

- **函数**：_source_is_available, get_skill

- **常量**：_PROBE_BUDGET_KEY, _CONNECTORS_LISTED_KEY, _CONNECTORS_DISABLED_NOTE

  - `DataLoadingSkill` 方法：handle_tool, handle_action, _connectors_disabled, _skill_state, _list_connectors, _describe_connector, _propose_connection, _already_loaded_tables, _propose_data_operation, _probe_budget

- **阅读建议**：先看公开类的 `run`/`handle_*`/`to_dict`，再看私有辅助。改动后运行对应 `tests/backend` 文件。涉及用户字符串或错误码时同步 i18n。

#### `py-src/data_formulator/analyst/skills/data_loading/skill.py`

- **目录**：`py-src/data_formulator/analyst/skills/data_loading`

- **行数**：362

- **所在子系统**：数据加载技能的兼容目录副本

- **主要功能简介**：该文件实现子系统中的一个具体单元，符号列表如下。

- **类**：DataLoadingSkill

- **函数**：_source_is_available, get_skill

- **常量**：_PROBE_BUDGET_KEY, _CONNECTORS_LISTED_KEY, _CONNECTORS_DISABLED_NOTE

  - `DataLoadingSkill` 方法：handle_tool, handle_action, _connectors_disabled, _skill_state, _list_connectors, _describe_connector, _propose_connection, _already_loaded_tables, _propose_data_operation, _probe_budget

- **阅读建议**：先看公开类的 `run`/`handle_*`/`to_dict`，再看私有辅助。改动后运行对应 `tests/backend` 文件。涉及用户字符串或错误码时同步 i18n。

#### `py-src/data_formulator/analyst/skills/report/__init__.py`

- **目录**：`py-src/data_formulator/analyst/skills/report`

- **行数**：9

- **所在子系统**：报告技能：inspect_chart 与 write_report 流式写作

- **主要功能简介**：report skill — streams a Markdown report from an exploration.  ``SKILL.md`` holds the instructions/action contract; ``skill.py`` exposes ``get_skill()`` (the executable handler, ported from ``agent_report_gen.py``).

- **类**：无

- **函数**：无

- **常量**：无

- **阅读建议**：先看公开类的 `run`/`handle_*`/`to_dict`，再看私有辅助。改动后运行对应 `tests/backend` 文件。涉及用户字符串或错误码时同步 i18n。

#### `py-src/data_formulator/analyst/skills/report/skill.py`

- **目录**：`py-src/data_formulator/analyst/skills/report`

- **行数**：212

- **所在子系统**：报告技能：inspect_chart 与 write_report 流式写作

- **主要功能简介**：report skill — turns an exploration into a Markdown report.  The analyst shell decides to write a report (the ``write_report`` **action**), then dispatches here. The model assembles the report in the **main agent loop**: it loads this skill, inspects whatever charts/data it needs via the skill-private ``inspect_chart`` tool (plus the always-on ``inspect_source_data``), and then emits ``write_repor

- **类**：ReportWritingSkill

- **函数**：_strip_leaked_tool_syntax, get_skill

- **常量**：_LEAK_SPECIAL_TOKEN, _LEAK_TOOLCALL

  - `ReportWritingSkill` 方法：handle_tool, handle_action, _handle_inspect_chart

- **阅读建议**：先看公开类的 `run`/`handle_*`/`to_dict`，再看私有辅助。改动后运行对应 `tests/backend` 文件。涉及用户字符串或错误码时同步 i18n。

#### `py-src/data_formulator/analyst/tools.py`

- **目录**：`py-src/data_formulator/analyst`

- **行数**：152

- **所在子系统**：统一 AnalystAgent 外壳与工具工厂

- **主要功能简介**：Inspection tools for the analyst agent.  Tools are parallel-safe, internal, side-effect-free capabilities the agent may call freely within a turn to gather information before committing to a single user-visible action. See ``design-docs/35`` §4.1.    - ``execute_python_script`` — run a general-purpose Python script in the     sandbox to inspect/compute (stdout returned).   - ``inspect_source_data`

- **类**：无

- **函数**：build_load_skill_tool, build_tools

- **常量**：无

- **阅读建议**：先看公开类的 `run`/`handle_*`/`to_dict`，再看私有辅助。改动后运行对应 `tests/backend` 文件。涉及用户字符串或错误码时同步 i18n。

#### `py-src/data_formulator/app.py`

- **目录**：`py-src/data_formulator`

- **行数**：560

- **所在子系统**：应用入口、Flask 装配、错误类型、连接器总控、模型注册、工作区工厂

- **主要功能简介**：该文件实现子系统中的一个具体单元，符号列表如下。

- **类**：CustomJSONEncoder

- **函数**：_resolve_data_home, configure_file_logging, configure_logging, _register_blueprints, _safety_checks, index_alt, get_auth_info, get_app_config, parse_args, run_app

- **常量**：APP_ROOT, _LOG_FORMAT, _FILE_HANDLER_MARKER

  - `CustomJSONEncoder` 方法：default

- **阅读建议**：先看公开类的 `run`/`handle_*`/`to_dict`，再看私有辅助。改动后运行对应 `tests/backend` 文件。涉及用户字符串或错误码时同步 i18n。

#### `py-src/data_formulator/auth/__init__.py`

- **目录**：`py-src/data_formulator/auth`

- **行数**：3

- **所在子系统**：身份解析、TokenStore、Azure CLI

- **主要功能简介**：该文件实现子系统中的一个具体单元，符号列表如下。

- **类**：无

- **函数**：无

- **常量**：无

- **阅读建议**：先看公开类的 `run`/`handle_*`/`to_dict`，再看私有辅助。改动后运行对应 `tests/backend` 文件。涉及用户字符串或错误码时同步 i18n。

#### `py-src/data_formulator/auth/azure_cli.py`

- **目录**：`py-src/data_formulator/auth`

- **行数**：40

- **所在子系统**：身份解析、TokenStore、Azure CLI

- **主要功能简介**：该文件实现子系统中的一个具体单元，符号列表如下。

- **类**：无

- **函数**：find_azure_cli, expose_azure_cli

- **常量**：无

- **阅读建议**：先看公开类的 `run`/`handle_*`/`to_dict`，再看私有辅助。改动后运行对应 `tests/backend` 文件。涉及用户字符串或错误码时同步 i18n。

#### `py-src/data_formulator/auth/gateways/__init__.py`

- **目录**：`py-src/data_formulator/auth/gateways`

- **行数**：3

- **所在子系统**：OAuth/OIDC/Kusto 登录回调蓝图

- **主要功能简介**：该文件实现子系统中的一个具体单元，符号列表如下。

- **类**：无

- **函数**：无

- **常量**：无

- **阅读建议**：先看公开类的 `run`/`handle_*`/`to_dict`，再看私有辅助。改动后运行对应 `tests/backend` 文件。涉及用户字符串或错误码时同步 i18n。

#### `py-src/data_formulator/auth/gateways/github_gateway.py`

- **目录**：`py-src/data_formulator/auth/gateways`

- **行数**：146

- **所在子系统**：OAuth/OIDC/Kusto 登录回调蓝图

- **主要功能简介**：GitHub OAuth authorization-code exchange gateway.  Provides ``/api/auth/github/login`` (redirect to GitHub) and ``/api/auth/github/callback`` (exchange code → token → user info → write Flask session).  The :class:`GitHubOAuthProvider` then reads from this session on subsequent requests.

- **类**：无

- **函数**：_error_redirect, _fetch_primary_email, github_login, github_callback, github_logout

- **常量**：无

- **阅读建议**：先看公开类的 `run`/`handle_*`/`to_dict`，再看私有辅助。改动后运行对应 `tests/backend` 文件。涉及用户字符串或错误码时同步 i18n。

#### `py-src/data_formulator/auth/gateways/kusto_oauth_gateway.py`

- **目录**：`py-src/data_formulator/auth/gateways`

- **行数**：252

- **所在子系统**：OAuth/OIDC/Kusto 登录回调蓝图

- **主要功能简介**：Microsoft delegated OAuth flow for Kusto connector access.

- **类**：无

- **函数**：_oauth_config, _callback_url, _normalize_cluster, _cluster_auth_metadata, _frontend_origin, _pkce_challenge, _popup_response, kusto_login, kusto_callback

- **常量**：_STATE_KEY, _KUSTO_HOST_SUFFIXES, _LOGIN_HOSTS

- **阅读建议**：先看公开类的 `run`/`handle_*`/`to_dict`，再看私有辅助。改动后运行对应 `tests/backend` 文件。涉及用户字符串或错误码时同步 i18n。

#### `py-src/data_formulator/auth/gateways/oidc_gateway.py`

- **目录**：`py-src/data_formulator/auth/gateways`

- **行数**：258

- **所在子系统**：OAuth/OIDC/Kusto 登录回调蓝图

- **主要功能简介**：Backend OIDC Confidential Client gateway.  When ``OIDC_CLIENT_SECRET`` is set (or ``AUTH_MODE=backend`` is forced), DF acts as a Confidential Client and handles the full Authorization Code flow server-side.  The browser never sees ``client_secret`` or raw tokens — only a session cookie.  Endpoint URLs are resolved from ``OIDCProvider.get_resolved_config()``, which supports auto-discovery via ``.we

- **类**：无

- **函数**：_get_oidc_config, _callback_url, _fetch_userinfo, oidc_login, _error_redirect, oidc_callback, oidc_status, oidc_logout, save_delegated_token, clear_service_token, auth_service_status

- **常量**：无

- **阅读建议**：先看公开类的 `run`/`handle_*`/`to_dict`，再看私有辅助。改动后运行对应 `tests/backend` 文件。涉及用户字符串或错误码时同步 i18n。

#### `py-src/data_formulator/auth/identity.py`

- **目录**：`py-src/data_formulator/auth`

- **行数**：248

- **所在子系统**：身份解析、TokenStore、Azure CLI

- **主要功能简介**：Authentication and identity management for Data Formulator.  Pluggable single-provider model with anonymous fallback::      AUTH_PROVIDER=oidc            → OIDCProvider   → user:<sub>     AUTH_PROVIDER=azure_easyauth  → AzureEasyAuth  → user:<principal>     (not set, localhost)          → single-user     → local:<os_username>     (not set, 0.0.0.0)           → anonymous only  → browser:<uuid>  Sec

- **类**：无

- **函数**：is_local_mode, _validate_identity_value, init_auth, get_identity_id, get_auth_result, get_sso_token, get_active_provider

- **常量**：_MAX_IDENTITY_LENGTH, _IDENTITY_RE

- **阅读建议**：先看公开类的 `run`/`handle_*`/`to_dict`，再看私有辅助。改动后运行对应 `tests/backend` 文件。涉及用户字符串或错误码时同步 i18n。

#### `py-src/data_formulator/auth/providers/__init__.py`

- **目录**：`py-src/data_formulator/auth/providers`

- **行数**：77

- **所在子系统**：OIDC / GitHub / Azure EasyAuth 提供者

- **主要功能简介**：Auto-discovery registry for AuthProvider subclasses.  On import, every ``.py`` module in this package (except ``base``) is scanned for concrete ``AuthProvider`` subclasses.  Each discovered class is instantiated once to read its ``name`` property, then stored in the registry keyed by that name.  Activation of a specific provider is controlled by the ``AUTH_PROVIDER`` environment variable in ``auth

- **类**：无

- **函数**：_discover_providers, get_provider_class, list_available_providers

- **常量**：无

- **阅读建议**：先看公开类的 `run`/`handle_*`/`to_dict`，再看私有辅助。改动后运行对应 `tests/backend` 文件。涉及用户字符串或错误码时同步 i18n。

#### `py-src/data_formulator/auth/providers/azure_easyauth.py`

- **目录**：`py-src/data_formulator/auth/providers`

- **行数**：53

- **所在子系统**：OIDC / GitHub / Azure EasyAuth 提供者

- **主要功能简介**：Azure App Service built-in authentication (EasyAuth) provider.  When Data Formulator is deployed on Azure App Service with authentication enabled, Azure verifies the user's identity *before* the request reaches Flask and injects trusted headers:  * ``X-MS-CLIENT-PRINCIPAL-ID`` — user's Object ID (always present) * ``X-MS-CLIENT-PRINCIPAL-NAME`` — display name (optional)  These headers are set by t

- **类**：AzureEasyAuthProvider

- **函数**：无

- **常量**：无

  - `AzureEasyAuthProvider` 方法：name, authenticate, get_auth_info

- **阅读建议**：先看公开类的 `run`/`handle_*`/`to_dict`，再看私有辅助。改动后运行对应 `tests/backend` 文件。涉及用户字符串或错误码时同步 i18n。

#### `py-src/data_formulator/auth/providers/base.py`

- **目录**：`py-src/data_formulator/auth/providers`

- **行数**：87

- **所在子系统**：OIDC / GitHub / Azure EasyAuth 提供者

- **主要功能简介**：Base classes for the pluggable authentication provider system.  AuthProvider subclasses are auto-discovered at startup. Each provider extracts and verifies user identity from an incoming Flask request.

- **类**：AuthResult, AuthProvider, AuthenticationError

- **函数**：无

- **常量**：无

  - `AuthResult` 方法：（无）

  - `AuthProvider` 方法：name, authenticate, enabled, on_configure, get_auth_info

  - `AuthenticationError` 方法：__init__

- **阅读建议**：先看公开类的 `run`/`handle_*`/`to_dict`，再看私有辅助。改动后运行对应 `tests/backend` 文件。涉及用户字符串或错误码时同步 i18n。

#### `py-src/data_formulator/auth/providers/github_oauth.py`

- **目录**：`py-src/data_formulator/auth/providers`

- **行数**：70

- **所在子系统**：OIDC / GitHub / Azure EasyAuth 提供者

- **主要功能简介**：GitHub OAuth 2.0 authentication provider.  GitHub is pure OAuth2 (not OIDC — there is no ``id_token``), so the authorization-code exchange must happen server-side.  This makes it a **stateful** (B-class) provider: the gateway blueprint handles the redirect dance and writes the result into the Flask session; this provider then reads the session on subsequent requests.  Configuration (environment va

- **类**：GitHubOAuthProvider

- **函数**：无

- **常量**：无

  - `GitHubOAuthProvider` 方法：__init__, name, enabled, get_auth_info, authenticate

- **阅读建议**：先看公开类的 `run`/`handle_*`/`to_dict`，再看私有辅助。改动后运行对应 `tests/backend` 文件。涉及用户字符串或错误码时同步 i18n。

#### `py-src/data_formulator/auth/providers/oidc.py`

- **目录**：`py-src/data_formulator/auth/providers`

- **行数**：395

- **所在子系统**：OIDC / GitHub / Azure EasyAuth 提供者

- **主要功能简介**：OIDC / OAuth2 authentication provider.  Supports both standards-compliant OIDC Identity Providers (with auto-discovery) and plain OAuth2 servers (with manually configured endpoint URLs).  Discovery strategy depends on AUTH_PROVIDER:     AUTH_PROVIDER=oidc   → tries /.well-known/openid-configuration     AUTH_PROVIDER=oauth2 → tries /.well-known/oauth-authorization-server  Minimal configuration::   

- **类**：OIDCProvider

- **函数**：is_backend_oidc_mode

- **常量**：无

  - `OIDCProvider` 方法：__init__, name, enabled, _build_ssl_context, _try_discovery, on_configure, get_resolved_config, _effective_scopes, get_auth_info, authenticate, _authenticate_session, _authenticate_jwt, _authenticate_userinfo

- **阅读建议**：先看公开类的 `run`/`handle_*`/`to_dict`，再看私有辅助。改动后运行对应 `tests/backend` 文件。涉及用户字符串或错误码时同步 i18n。

#### `py-src/data_formulator/auth/token_store.py`

- **目录**：`py-src/data_formulator/auth`

- **行数**：390

- **所在子系统**：身份解析、TokenStore、Azure CLI

- **主要功能简介**：Unified credential manager for all third-party systems.  Resolves credentials through a priority chain:   cached → refresh → sso_exchange → delegated → vault → none.  All callers (Agent, DataConnector, routes) use the same interface.

- **类**：TokenStore

- **函数**：无

- **常量**：_SSO_NS, _SVC_NS, _SSO_BLOCKED_NS

  - `TokenStore` 方法：get_access, get_sso_token, get_auth_status, store_service_token, clear_service_token, clear_session_tokens, block_sso_reconnect, allow_sso_reconnect, is_sso_reconnect_blocked, store_sso_tokens, _get_cached, _is_expired, _do_refresh, _do_sso_exchange, _try_vault, _vault_retrieve, _vault_store, _vault_delete, _refresh_sso, _get_auth_config, _all_auth_configs, _available_strategies, _resolve_env

- **阅读建议**：先看公开类的 `run`/`handle_*`/`to_dict`，再看私有辅助。改动后运行对应 `tests/backend` 文件。涉及用户字符串或错误码时同步 i18n。

#### `py-src/data_formulator/auth/vault/__init__.py`

- **目录**：`py-src/data_formulator/auth/vault`

- **行数**：111

- **所在子系统**：本地加密凭证库

- **主要功能简介**：Credential Vault factory — returns the global vault instance.  Key resolution (first match wins):  1. ``CREDENTIAL_VAULT_KEY`` env var  — explicit key (server deployments) 2. ``DATA_FORMULATOR_HOME/.vault_key`` file — auto-generated on first run 3. Neither → vault disabled, plugins fall back to session-only storage  For local single-user mode the vault is **zero-config**: a Fernet key is auto-gene

- **类**：无

- **函数**：get_data_formulator_home, _resolve_key, get_credential_vault

- **常量**：无

- **阅读建议**：先看公开类的 `run`/`handle_*`/`to_dict`，再看私有辅助。改动后运行对应 `tests/backend` 文件。涉及用户字符串或错误码时同步 i18n。

#### `py-src/data_formulator/auth/vault/base.py`

- **目录**：`py-src/data_formulator/auth/vault`

- **行数**：40

- **所在子系统**：本地加密凭证库

- **主要功能简介**：Abstract interface for credential storage backends.

- **类**：CredentialVault

- **函数**：无

- **常量**：无

  - `CredentialVault` 方法：store, retrieve, delete, list_sources

- **阅读建议**：先看公开类的 `run`/`handle_*`/`to_dict`，再看私有辅助。改动后运行对应 `tests/backend` 文件。涉及用户字符串或错误码时同步 i18n。

#### `py-src/data_formulator/auth/vault/local_vault.py`

- **目录**：`py-src/data_formulator/auth/vault`

- **行数**：95

- **所在子系统**：本地加密凭证库

- **主要功能简介**：SQLite + Fernet encrypted credential vault.  Storage location: ``DATA_FORMULATOR_HOME/credentials.db``  Generate a Fernet key::      python -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())"

- **类**：LocalCredentialVault

- **函数**：无

- **常量**：无

  - `LocalCredentialVault` 方法：__init__, _init_db, store, retrieve, delete, list_sources

- **阅读建议**：先看公开类的 `run`/`handle_*`/`to_dict`，再看私有辅助。改动后运行对应 `tests/backend` 文件。涉及用户字符串或错误码时同步 i18n。

#### `py-src/data_formulator/data_connector.py`

- **目录**：`py-src/data_formulator`

- **行数**：2720

- **所在子系统**：应用入口、Flask 装配、错误类型、连接器总控、模型注册、工作区工厂

- **主要功能简介**：DataConnector — generic lifecycle wrapper for ExternalDataLoader.  Takes any ``ExternalDataLoader`` class and auto-generates a Flask Blueprint with auth / catalog / data routes.  No per-connector code needed.  Usage::      from data_formulator.data_connector import DataConnector      connector = DataConnector.from_loader(         PostgreSQLDataLoader,         source_id="pg_prod",         display_n

- **类**：DataConnector, SourceSpec

- **函数**：_set_catalog_progress, _get_catalog_progress, _clear_catalog_progress, classify_and_raise_connector_error, _sanitize_error, _node_to_dict, _hierarchy_dicts, _catalog_pagination_args, _filter_catalog_tables, _lightweight_tree_for_response, _merged_catalog_tables, _catalog_tree_payload, _user_connector_key, _is_user_connector_key, _public_connector_id, _param_defs_by_name, _is_sensitive_or_auth_param, _connector_config_params, _loader_auth_mode, _visible_connector_items, _resolve_connector_with_key, _resolve_connector, resolve_live_loader, resolve_catalog_refresh_target, connector_is_available, _parse_source_table, _cached_source_metadata, list_data_loaders, discover_data_loader_options, pick_local_directory, _az_account_summary, azure_cli_status, azure_cli_login, list_connectors, create_connector, _connectors_dir, _connectors_jail, _validate_connector_id_for_fs, _safe_source_filename, _persist_user_connector, _remove_user_connector, _update_user_connector_display_name, update_connector, delete_connector, connector_connect, connector_disconnect, connector_get_status, connector_get_catalog, connector_get_catalog_tree, connector_get_catalog_progress, connector_get_cached_catalog_tree, connector_sync_catalog_metadata, connector_search_catalog, connector_import_data, connector_refresh_data, connector_preview_data, connector_column_values, connector_import_group, _resolve_env_refs, _get_df_home, _load_connectors_yaml, _load_admin_specs, _load_user_specs, load_connectors, register_data_connectors

- **常量**：_MAX_CATALOG_PAGE_SIZE, _USER_CONNECTOR_PREFIX, _RECONNECT_MAX_ATTEMPTS, _RECONNECT_BACKOFF_BASE, _CATALOG_PROGRESS_LOCK, _CONNECTOR_ID_RE

  - `DataConnector` 方法：__init__, from_loader, _manifest, get_frontend_config, _resolve_delegated_login, _get_identity, _get_vault, _vault_store, _vault_retrieve, _vault_delete, has_stored_credentials, _get_loader, _connect, _persist_credentials, _delete_credentials, _try_auto_reconnect, _try_ambient_reconnect, _inject_credentials, _try_sso_auto_connect, _require_loader

  - `SourceSpec` 方法：（无）

- **阅读建议**：先看公开类的 `run`/`handle_*`/`to_dict`，再看私有辅助。改动后运行对应 `tests/backend` 文件。涉及用户字符串或错误码时同步 i18n。

#### `py-src/data_formulator/data_loader/__init__.py`

- **目录**：`py-src/data_formulator/data_loader`

- **行数**：365

- **所在子系统**：外部数据加载器实现与插件扫描

- **主要功能简介**：Modular data-loader registry.  Two loader sources:  1. **Built-in** — declared in ``_LOADER_SPECS`` (this file).  Each loader    is independently imported via try/except so that a missing dependency    only disables that one loader.  2. **External plugins** — Python files matching ``*_data_loader.py``    found in the plugin directory.  Resolution order:     1. ``DF_PLUGIN_DIR`` env var — explicit 

- **类**：无

- **函数**：_scan_package_loaders, _resolve_plugin_dir, _plugin_scanning_enabled, _register_plugin_class, _load_plugin_file, _scan_plugin_dir, _enforce_deployment_restrictions, get_available_loaders

- **常量**：无

- **阅读建议**：先看公开类的 `run`/`handle_*`/`to_dict`，再看私有辅助。改动后运行对应 `tests/backend` 文件。涉及用户字符串或错误码时同步 i18n。

#### `py-src/data_formulator/data_loader/athena_data_loader.py`

- **目录**：`py-src/data_formulator/data_loader`

- **行数**：567

- **所在子系统**：外部数据加载器实现与插件扫描

- **主要功能简介**：该文件实现子系统中的一个具体单元，符号列表如下。

- **类**：AthenaDataLoader

- **函数**：_validate_athena_table_name, _validate_column_name, _validate_s3_url

- **常量**：ATHENA_TABLE_PATTERN, ATHENA_COLUMN_PATTERN, S3_URL_PATTERN

  - `AthenaDataLoader` 方法：list_params, auth_paths, infer_auth_path, __init__, _get_output_location, _execute_query, fetch_data_as_arrow, list_tables, catalog_hierarchy, ls, get_metadata, test_connection

- **阅读建议**：先看公开类的 `run`/`handle_*`/`to_dict`，再看私有辅助。改动后运行对应 `tests/backend` 文件。涉及用户字符串或错误码时同步 i18n。

#### `py-src/data_formulator/data_loader/azure_blob_data_loader.py`

- **目录**：`py-src/data_formulator/data_loader`

- **行数**：402

- **所在子系统**：外部数据加载器实现与插件扫描

- **主要功能简介**：该文件实现子系统中的一个具体单元，符号列表如下。

- **类**：AzureBlobDataLoader

- **函数**：无

- **常量**：无

  - `AzureBlobDataLoader` 方法：list_params, auth_paths, infer_auth_path, __init__, _azure_path, _read_sample, fetch_data_as_arrow, probe, list_tables, _is_supported_file, _estimate_row_count, _estimate_rows_by_sampling, _estimate_by_row_sampling, catalog_hierarchy, ls, get_metadata, test_connection

- **阅读建议**：先看公开类的 `run`/`handle_*`/`to_dict`，再看私有辅助。改动后运行对应 `tests/backend` 文件。涉及用户字符串或错误码时同步 i18n。

#### `py-src/data_formulator/data_loader/bigquery_data_loader.py`

- **目录**：`py-src/data_formulator/data_loader`

- **行数**：348

- **所在子系统**：外部数据加载器实现与插件扫描

- **主要功能简介**：该文件实现子系统中的一个具体单元，符号列表如下。

- **类**：BigQueryDataLoader

- **函数**：无

- **常量**：无

  - `BigQueryDataLoader` 方法：list_params, auth_paths, infer_auth_path, __init__, list_tables, fetch_data_as_arrow, probe, _build_select_parts, catalog_hierarchy, ls, get_metadata, test_connection

- **阅读建议**：先看公开类的 `run`/`handle_*`/`to_dict`，再看私有辅助。改动后运行对应 `tests/backend` 文件。涉及用户字符串或错误码时同步 i18n。

#### `py-src/data_formulator/data_loader/clickhouse_data_loader.py`

- **目录**：`py-src/data_formulator/data_loader`

- **行数**：752

- **所在子系统**：外部数据加载器实现与插件扫描

- **主要功能简介**：ClickHouse connector for Data Formulator.

- **类**：ClickHouseDataLoader

- **函数**：_as_bool

- **常量**：_QUOTED_RELATION_RE, _RAW_SQL_RE, _SOURCE_FILTER_OPERATORS, _LEGACY_OPERATORS

  - `ClickHouseDataLoader` 方法：list_params, auth_paths, __init__, close, __enter__, __exit__, __del__, _read_sql, _quote_identifier, _unwrap_type, _cast_function, _decimal_precision, _project_column, _column_casts, _casts_from_column_types, _build_select_list, _quote_relation, _resolve_source_table, _validated_size, _compile_filters, fetch_data_as_arrow, probe, _catalog_tables, _catalog_columns, list_tables, search_catalog, catalog_hierarchy, ls, get_metadata, get_column_types

- **阅读建议**：先看公开类的 `run`/`handle_*`/`to_dict`，再看私有辅助。改动后运行对应 `tests/backend` 文件。涉及用户字符串或错误码时同步 i18n。

#### `py-src/data_formulator/data_loader/connector_errors.py`

- **目录**：`py-src/data_formulator/data_loader`

- **行数**：216

- **所在子系统**：外部数据加载器实现与插件扫描

- **主要功能简介**：该文件实现子系统中的一个具体单元，符号列表如下。

- **类**：ConnectorErrorInfo

- **函数**：classify_connector_error, raise_connector_error, _exception_chain, _http_status, _has_any

- **常量**：无

  - `ConnectorErrorInfo` 方法：to_app_error, to_error_dict

- **阅读建议**：先看公开类的 `run`/`handle_*`/`to_dict`，再看私有辅助。改动后运行对应 `tests/backend` 文件。涉及用户字符串或错误码时同步 i18n。

#### `py-src/data_formulator/data_loader/cosmosdb_data_loader.py`

- **目录**：`py-src/data_formulator/data_loader`

- **行数**：349

- **所在子系统**：外部数据加载器实现与插件扫描

- **主要功能简介**：该文件实现子系统中的一个具体单元，符号列表如下。

- **类**：CosmosDBDataLoader

- **函数**：无

- **常量**：无

  - `CosmosDBDataLoader` 方法：list_params, __init__, close, __enter__, __exit__, __del__, _flatten_document, _convert_special_types, _process_documents, fetch_data_as_arrow, list_tables, catalog_hierarchy, ls, get_metadata, test_connection

- **阅读建议**：先看公开类的 `run`/`handle_*`/`to_dict`，再看私有辅助。改动后运行对应 `tests/backend` 文件。涉及用户字符串或错误码时同步 i18n。

#### `py-src/data_formulator/data_loader/databricks_data_loader.py`

- **目录**：`py-src/data_formulator/data_loader`

- **行数**：347

- **所在子系统**：外部数据加载器实现与插件扫描

- **主要功能简介**：该文件实现子系统中的一个具体单元，符号列表如下。

- **类**：DatabricksDataLoader

- **函数**：_bt

- **常量**：_HIDDEN_SCHEMAS, _HIDDEN_CATALOGS, _MAX_CATALOGS, _MAX_TABLES

  - `DatabricksDataLoader` 方法：list_params, auth_paths, infer_auth_path, delegated_login_config, __init__, _query_arrow, _query_rows, _resolve_source_table, catalog_hierarchy, _catalogs, list_tables, _list_tables_in_catalog, get_column_types, fetch_data_as_arrow

- **阅读建议**：先看公开类的 `run`/`handle_*`/`to_dict`，再看私有辅助。改动后运行对应 `tests/backend` 文件。涉及用户字符串或错误码时同步 i18n。

#### `py-src/data_formulator/data_loader/external_data_loader.py`

- **目录**：`py-src/data_formulator/data_loader`

- **行数**：1212

- **所在子系统**：外部数据加载器实现与插件扫描

- **主要功能简介**：该文件实现子系统中的一个具体单元，符号列表如下。

- **类**：CatalogCachePolicy, ConnectorParamError, CatalogNode, ExternalDataLoader

- **函数**：apply_import_projection, _merge_source_metadata, _esc_id, _esc_str, build_where_clause, build_where_clause_inline, build_source_filter_where_clause_inline, sanitize_table_name, infer_source_metadata_status

- **常量**：MAX_IMPORT_ROWS, SENSITIVE_PARAMS, _VALID_OPERATORS, _SOURCE_FILTER_OPERATOR_MAP, _DANGEROUS_IDENT_RE, SOURCE_METADATA_OK, SOURCE_METADATA_PARTIAL, SOURCE_METADATA_UNAVAILABLE, SOURCE_METADATA_SYNCED, SOURCE_METADATA_NOT_SYNCED

  - `CatalogCachePolicy` 方法：__post_init__

  - `ConnectorParamError` 方法：__init__

  - `CatalogNode` 方法：（无）

  - `ExternalDataLoader` 方法：catalog_cache_policy, _report_progress, get_safe_params, fetch_data_as_arrow, fetch_data_as_dataframe, ingest_to_workspace, list_params, validate_params, auth_paths, infer_auth_path, discover_param_options, auth_instructions, delegated_login_config, __init__, list_tables, catalog_hierarchy, effective_hierarchy, pinned_scope, ls, get_column_values, get_metadata, get_column_types, probe, _tables_to_catalog_tree, list_tables_tree, search_catalog, sync_catalog_metadata, ensure_table_keys, test_connection, auth_mode

- **阅读建议**：先看公开类的 `run`/`handle_*`/`to_dict`，再看私有辅助。改动后运行对应 `tests/backend` 文件。涉及用户字符串或错误码时同步 i18n。

#### `py-src/data_formulator/data_loader/guides/__init__.py`

- **目录**：`py-src/data_formulator/data_loader/guides`

- **行数**：1

- **所在子系统**：见文件名与导出。

- **主要功能简介**：Packaged Markdown connection guides for built-in data loaders.

- **类**：无

- **函数**：无

- **常量**：无

- **阅读建议**：先看公开类的 `run`/`handle_*`/`to_dict`，再看私有辅助。改动后运行对应 `tests/backend` 文件。涉及用户字符串或错误码时同步 i18n。

#### `py-src/data_formulator/data_loader/kusto_data_loader.py`

- **目录**：`py-src/data_formulator/data_loader`

- **行数**：867

- **所在子系统**：外部数据加载器实现与插件扫描

- **主要功能简介**：该文件实现子系统中的一个具体单元，符号列表如下。

- **类**：_KustoDelegatedCredential, KustoDataLoader

- **函数**：_coerce_int

- **常量**：_ISO_DATETIME_RE

  - `_KustoDelegatedCredential` 方法：__init__, get_token

  - `KustoDataLoader` 方法：list_params, auth_paths, infer_auth_path, delegated_login_config, __init__, _build_kcsb, _convert_kusto_datetime_columns, _stringify_dynamic_columns, query, fetch_data_as_arrow, probe, _kql_ident, _kql_lit, _kql_cmp_lit, _compile_probe_kql, _compile_kql_where, _resolve_source_table, discover_param_options, list_tables, _fetch_db_columns_bulk, _list_tables_in_db, catalog_hierarchy, ls, get_metadata, test_connection

- **阅读建议**：先看公开类的 `run`/`handle_*`/`to_dict`，再看私有辅助。改动后运行对应 `tests/backend` 文件。涉及用户字符串或错误码时同步 i18n。

#### `py-src/data_formulator/data_loader/local_folder_data_loader.py`

- **目录**：`py-src/data_formulator/data_loader`

- **行数**：342

- **所在子系统**：外部数据加载器实现与插件扫描

- **主要功能简介**：Local folder data loader — reads data files from a directory on the local filesystem.  Only available in local deployment mode (backend bound to localhost). Uses ConfinedDir to ensure all file access stays within the connected root directory.

- **类**：LocalFolderDataLoader

- **函数**：无

- **常量**：SUPPORTED_EXTENSIONS

  - `LocalFolderDataLoader` 方法：list_params, catalog_hierarchy, __init__, test_connection, ls, get_metadata, list_tables, fetch_data_as_arrow, probe, _file_metadata

- **阅读建议**：先看公开类的 `run`/`handle_*`/`to_dict`，再看私有辅助。改动后运行对应 `tests/backend` 文件。涉及用户字符串或错误码时同步 i18n。

#### `py-src/data_formulator/data_loader/mongodb_data_loader.py`

- **目录**：`py-src/data_formulator/data_loader`

- **行数**：519

- **所在子系统**：外部数据加载器实现与插件扫描

- **主要功能简介**：该文件实现子系统中的一个具体单元，符号列表如下。

- **类**：MongoDBDataLoader

- **函数**：无

- **常量**：无

  - `MongoDBDataLoader` 方法：list_params, auth_paths, infer_auth_path, __init__, close, __enter__, __exit__, __del__, _flatten_document, _convert_special_types, _process_documents, fetch_data_as_arrow, probe, _compile_probe_pipeline, _compile_match, list_tables, catalog_hierarchy, ls, get_metadata, test_connection

- **阅读建议**：先看公开类的 `run`/`handle_*`/`to_dict`，再看私有辅助。改动后运行对应 `tests/backend` 文件。涉及用户字符串或错误码时同步 i18n。

#### `py-src/data_formulator/data_loader/mssql_data_loader.py`

- **目录**：`py-src/data_formulator/data_loader`

- **行数**：784

- **所在子系统**：外部数据加载器实现与插件扫描

- **主要功能简介**：该文件实现子系统中的一个具体单元，符号列表如下。

- **类**：MSSQLDataLoader

- **函数**：无

- **常量**：无

  - `MSSQLDataLoader` 方法：list_params, auth_paths, infer_auth_path, __init__, _safe_select_list, _read_sql, _execute_query_raw, _execute_query, fetch_data_as_arrow, probe, list_tables, _list_tables_for_db, sync_catalog_metadata, catalog_hierarchy, ls, get_metadata, test_connection

- **阅读建议**：先看公开类的 `run`/`handle_*`/`to_dict`，再看私有辅助。改动后运行对应 `tests/backend` 文件。涉及用户字符串或错误码时同步 i18n。

#### `py-src/data_formulator/data_loader/mysql_data_loader.py`

- **目录**：`py-src/data_formulator/data_loader`

- **行数**：524

- **所在子系统**：外部数据加载器实现与插件扫描

- **主要功能简介**：该文件实现子系统中的一个具体单元，符号列表如下。

- **类**：MySQLDataLoader

- **函数**：无

- **常量**：无

  - `MySQLDataLoader` 方法：list_params, auth_paths, __init__, _get_conn, _read_sql, _safe_select_list, fetch_data_as_arrow, _fetch_data_as_arrow, probe, list_tables, _list_tables, search_catalog, catalog_hierarchy, ls, get_metadata, test_connection

- **阅读建议**：先看公开类的 `run`/`handle_*`/`to_dict`，再看私有辅助。改动后运行对应 `tests/backend` 文件。涉及用户字符串或错误码时同步 i18n。

#### `py-src/data_formulator/data_loader/postgresql_data_loader.py`

- **目录**：`py-src/data_formulator/data_loader`

- **行数**：860

- **所在子系统**：外部数据加载器实现与插件扫描

- **主要功能简介**：该文件实现子系统中的一个具体单元，符号列表如下。

- **类**：PostgreSQLDataLoader

- **函数**：无

- **常量**：_PG_CLIENT_ENCODING

  - `PostgreSQLDataLoader` 方法：list_params, __init__, _connection_kwargs, _resolve_source_table, _read_sql, _execute_on_conn, _safe_select_list, fetch_data_as_arrow, probe, list_tables, _list_tables, _cross_db_list_tables, _list_tables_for_db, sync_catalog_metadata, search_catalog, catalog_hierarchy, _tables_to_catalog_tree, _connect_to_db, _read_sql_on, ls, get_metadata, test_connection

- **阅读建议**：先看公开类的 `run`/`handle_*`/`to_dict`，再看私有辅助。改动后运行对应 `tests/backend` 文件。涉及用户字符串或错误码时同步 i18n。

#### `py-src/data_formulator/data_loader/probe_utils.py`

- **目录**：`py-src/data_formulator/data_loader`

- **行数**：429

- **所在子系统**：外部数据加载器实现与插件扫描

- **主要功能简介**：Shared building blocks for the connector ``probe`` capability (design 37).  A probe is a bounded, single-table SPJQ read (Select–Project–Aggregate, *no join*) the data-loading agent runs to size a slice and pick real filter values. Every loader implements its **own** ``probe`` using its backend's native query API — Postgres/MySQL/MSSQL/BigQuery compile SQL, Kusto compiles KQL, Mongo builds an aggr

- **类**：SqlDialect

- **函数**：clamp_probe_limit, quote_ident, probe_filters_to_source_filters, _lit, _contains_lit, _compile_where, compile_probe_sql, shape_probe_payload, probe_via_native_sql, run_probe_on_duckdb

- **常量**：PROBE_MAX_ROWS, PROBE_DEFAULT_ROWS, PROBE_SCAN_ROWS, _PROBE_AGG_OPS, _FILTER_OP_TO_SQL, _DANGEROUS_IDENT_RE, ANSI, DUCKDB, POSTGRES, MYSQL, CLICKHOUSE, MSSQL, BIGQUERY, ATHENA

  - `SqlDialect` 方法：（无）

- **阅读建议**：先看公开类的 `run`/`handle_*`/`to_dict`，再看私有辅助。改动后运行对应 `tests/backend` 文件。涉及用户字符串或错误码时同步 i18n。

#### `py-src/data_formulator/data_loader/s3_data_loader.py`

- **目录**：`py-src/data_formulator/data_loader`

- **行数**：308

- **所在子系统**：外部数据加载器实现与插件扫描

- **主要功能简介**：该文件实现子系统中的一个具体单元，符号列表如下。

- **类**：S3DataLoader

- **函数**：无

- **常量**：无

  - `S3DataLoader` 方法：list_params, auth_paths, infer_auth_path, __init__, fetch_data_as_arrow, probe, list_tables, _read_sample_arrow, _is_supported_file, _estimate_row_count, catalog_hierarchy, ls, get_metadata, test_connection

- **阅读建议**：先看公开类的 `run`/`handle_*`/`to_dict`，再看私有辅助。改动后运行对应 `tests/backend` 文件。涉及用户字符串或错误码时同步 i18n。

#### `py-src/data_formulator/data_loader/sample_datasets_loader.py`

- **目录**：`py-src/data_formulator/data_loader`

- **行数**：312

- **所在子系统**：外部数据加载器实现与插件扫描

- **主要功能简介**：Sample datasets data loader.  Exposes the built-in ``EXAMPLE_DATASETS`` catalog as a virtual data connector that behaves exactly like any other connector.  No auth, no external service of its own — table data is fetched on demand from the public URLs declared in :mod:`data_formulator.example_datasets_config`.  The connector is registered unconditionally at startup so that even in ``--disable_datab

- **类**：SampleDatasetsLoader

- **函数**：无

- **常量**：_SAMPLE_CACHE_LOCK, _SAMPLE_CACHE_MAX

  - `SampleDatasetsLoader` 方法：list_params, auth_mode, auth_config, catalog_hierarchy, __init__, test_connection, _datasets, _table_stem, _columns_from_sample, _resolve, list_tables, get_column_types, fetch_data_as_arrow, probe, _load_full_dataframe

- **阅读建议**：先看公开类的 `run`/`handle_*`/`to_dict`，再看私有辅助。改动后运行对应 `tests/backend` 文件。涉及用户字符串或错误码时同步 i18n。

#### `py-src/data_formulator/data_loader/superset_auth_bridge.py`

- **目录**：`py-src/data_formulator/data_loader`

- **行数**：89

- **所在子系统**：外部数据加载器实现与插件扫描

- **主要功能简介**：Authenticate users via the Superset REST API (JWT).

- **类**：SupersetAuthBridge

- **函数**：无

- **常量**：无

  - `SupersetAuthBridge` 方法：__init__, login, get_user_info, validate_token, refresh_token, exchange_sso_token

- **阅读建议**：先看公开类的 `run`/`handle_*`/`to_dict`，再看私有辅助。改动后运行对应 `tests/backend` 文件。涉及用户字符串或错误码时同步 i18n。

#### `py-src/data_formulator/data_loader/superset_client.py`

- **目录**：`py-src/data_formulator/data_loader`

- **行数**：182

- **所在子系统**：外部数据加载器实现与插件扫描

- **主要功能简介**：Thin wrapper around the Superset public REST API.

- **类**：SupersetClient

- **函数**：无

- **常量**：无

  - `SupersetClient` 方法：__init__, _headers, list_datasets, get_dataset_detail, get_dataset_columns, get_dataset_distinct_values, get_datasource_column_values, list_dashboards, get_dashboard_datasets, get_dashboard_detail, post_chart_data

- **阅读建议**：先看公开类的 `run`/`handle_*`/`to_dict`，再看私有辅助。改动后运行对应 `tests/backend` 文件。涉及用户字符串或错误码时同步 i18n。

#### `py-src/data_formulator/data_loader/superset_data_loader.py`

- **目录**：`py-src/data_formulator/data_loader`

- **行数**：1112

- **所在子系统**：外部数据加载器实现与插件扫描

- **主要功能简介**：SupersetLoader — ExternalDataLoader implementation for Apache Superset.  Treats Superset as a hierarchical data source:   dashboard (table_group) → dataset (table)  Authentication is JWT-based (``auth_mode() = "token"``).  Data is fetched via Superset's Chart Data API (``POST /api/v1/chart/data``), which only requires ``datasource access`` permission and automatically applies Row-Level Security (R

- **类**：SupersetLoader

- **函数**：无

- **常量**：无

  - `SupersetLoader` 方法：list_params, auth_paths, infer_auth_path, auth_mode, auth_config, delegated_login_config, catalog_hierarchy, __init__, _do_login, _try_sso_exchange, _is_token_expired, _ensure_token, test_connection, list_tables, _enrich_columns, search_catalog, ls, _build_dashboard_group_metadata, _build_chart_data_filters, _build_chart_data_orderby, get_metadata, get_column_types, _build_column_entry, _normalize_column_type, get_column_values, fetch_data_as_arrow, _convert_temporal_columns, _empty_arrow_table, list_tables_tree, _fetch_all_datasets

- **阅读建议**：先看公开类的 `run`/`handle_*`/`to_dict`，再看私有辅助。改动后运行对应 `tests/backend` 文件。涉及用户字符串或错误码时同步 i18n。

#### `py-src/data_formulator/data_operations/__init__.py`

- **目录**：`py-src/data_formulator/data_operations`

- **行数**：46

- **所在子系统**：结构化加载计划模型、仓库、执行器

- **主要功能简介**：该文件实现子系统中的一个具体单元，符号列表如下。

- **类**：无

- **函数**：无

- **常量**：无

- **阅读建议**：先看公开类的 `run`/`handle_*`/`to_dict`，再看私有辅助。改动后运行对应 `tests/backend` 文件。涉及用户字符串或错误码时同步 i18n。

#### `py-src/data_formulator/data_operations/actions.py`

- **目录**：`py-src/data_formulator/data_operations`

- **行数**：12

- **所在子系统**：结构化加载计划模型、仓库、执行器

- **主要功能简介**：该文件实现子系统中的一个具体单元，符号列表如下。

- **类**：无

- **函数**：build_data_operation_action

- **常量**：无

- **阅读建议**：先看公开类的 `run`/`handle_*`/`to_dict`，再看私有辅助。改动后运行对应 `tests/backend` 文件。涉及用户字符串或错误码时同步 i18n。

#### `py-src/data_formulator/data_operations/discovery.py`

- **目录**：`py-src/data_formulator/data_operations`

- **行数**：373

- **所在子系统**：结构化加载计划模型、仓库、执行器

- **主要功能简介**：该文件实现子系统中的一个具体单元，符号列表如下。

- **类**：ProbeBudget, ProbeGuidance, DataDiscoveryService

- **函数**：ensure_catalogs_current, ensure_no_auth_catalogs_cached, _freshness_payload

- **常量**：DEFAULT_PROBE_BUDGET, ANALYST_PROBE_GUIDANCE, STANDALONE_PROBE_GUIDANCE

  - `ProbeBudget` 方法：consume

  - `ProbeGuidance` 方法：（无）

  - `DataDiscoveryService` 方法：__init__, list_data, find_data, describe_data, resolve_catalog_path, resolve_load_table, probe_data

- **阅读建议**：先看公开类的 `run`/`handle_*`/`to_dict`，再看私有辅助。改动后运行对应 `tests/backend` 文件。涉及用户字符串或错误码时同步 i18n。

#### `py-src/data_formulator/data_operations/executor.py`

- **目录**：`py-src/data_formulator/data_operations`

- **行数**：196

- **所在子系统**：结构化加载计划模型、仓库、执行器

- **主要功能简介**：该文件实现子系统中的一个具体单元，符号列表如下。

- **类**：DataOperationExecutionResult, DataOperationExecutor

- **函数**：无

- **常量**：无

  - `DataOperationExecutionResult` 方法：（无）

  - `DataOperationExecutor` 方法：__init__, execute, _publish_connector_query, _find_published_results, _build_import_options, _allocate_table_name, _resolve_live_loader

- **阅读建议**：先看公开类的 `run`/`handle_*`/`to_dict`，再看私有辅助。改动后运行对应 `tests/backend` 文件。涉及用户字符串或错误码时同步 i18n。

#### `py-src/data_formulator/data_operations/models.py`

- **目录**：`py-src/data_formulator/data_operations`

- **行数**：411

- **所在子系统**：结构化加载计划模型、仓库、执行器

- **主要功能简介**：该文件实现子系统中的一个具体单元，符号列表如下。

- **类**：DataOperationStatus, OperationFilter, LoadQueryOrder, LoadQuery, ConnectorQueryStep, DataOperationPlan, OperationError, FailedOperationStep, DataOperation

- **函数**：_new_id, _canonical_hash, _freeze_json, _thaw_json, _step_from_dict

- **常量**：DATA_OPERATION_SCHEMA_VERSION

  - `DataOperationStatus` 方法：（无）

  - `OperationFilter` 方法：__post_init__, to_dict, from_dict

  - `LoadQueryOrder` 方法：__post_init__, to_dict, from_dict

  - `LoadQuery` 方法：__post_init__, to_dict, from_dict

  - `ConnectorQueryStep` 方法：to_dict, to_public_dict, from_dict

  - `DataOperationPlan` 方法：__post_init__, compute_hash, to_dict, to_public_dict, from_dict

  - `OperationError` 方法：to_dict, from_dict

  - `FailedOperationStep` 方法：to_dict, from_dict

  - `DataOperation` 方法：__post_init__, to_dict, to_public_dict, from_dict

- **阅读建议**：先看公开类的 `run`/`handle_*`/`to_dict`，再看私有辅助。改动后运行对应 `tests/backend` 文件。涉及用户字符串或错误码时同步 i18n。

#### `py-src/data_formulator/data_operations/repository.py`

- **目录**：`py-src/data_formulator/data_operations`

- **行数**：295

- **所在子系统**：结构化加载计划模型、仓库、执行器

- **主要功能简介**：该文件实现子系统中的一个具体单元，符号列表如下。

- **类**：DataOperationConflictError, StoredDataOperation, DataOperationRepository

- **函数**：resolve_interaction_response

- **常量**：STORE_FILENAME, STORE_VERSION

  - `DataOperationConflictError` 方法：（无）

  - `StoredDataOperation` 方法：to_dict, from_dict

  - `DataOperationRepository` 方法：__init__, for_workspace, create, get, get_awaiting_selection, select, complete, fail, finish, _record_execution, _select_operation, _read_unlocked, _write_unlocked

- **阅读建议**：先看公开类的 `run`/`handle_*`/`to_dict`，再看私有辅助。改动后运行对应 `tests/backend` 文件。涉及用户字符串或错误码时同步 i18n。

#### `py-src/data_formulator/datalake/__init__.py`

- **目录**：`py-src/data_formulator/datalake`

- **行数**：129

- **所在子系统**：工作区、Parquet、目录缓存、命名与元数据

- **主要功能简介**：Data Lake module for Data Formulator.  This module provides a unified data management layer that: - Manages user workspaces with identity-based directories - Stores user-uploaded files as-is (CSV, Excel, TXT, HTML, JSON, PDF) - Stores data from external loaders as parquet via pyarrow - Tracks all data sources in a workspace.yaml metadata file  Example usage:      from data_formulator.datalake impo

- **类**：无

- **函数**：无

- **常量**：无

- **阅读建议**：先看公开类的 `run`/`handle_*`/`to_dict`，再看私有辅助。改动后运行对应 `tests/backend` 文件。涉及用户字符串或错误码时同步 i18n。

#### `py-src/data_formulator/datalake/azure_blob_workspace.py`

- **目录**：`py-src/data_formulator/datalake`

- **行数**：777

- **所在子系统**：工作区、Parquet、目录缓存、命名与元数据

- **主要功能简介**：Azure Blob Storage–backed workspace for the Data Lake.  Drop-in replacement for :class:`Workspace` where every file (data files **and** ``workspace.yaml`` metadata) lives as a blob under::      <container>/<datalake_root>/<sanitized_identity_id>/  Requires ``azure-storage-blob`` (``pip install azure-storage-blob``).  Usage::      from azure.storage.blob import ContainerClient      container = Cont

- **类**：AzureBlobWorkspace

- **函数**：get_azure_workspace_scratch_path, _data_cache_ttl

- **常量**：无

  - `AzureBlobWorkspace` 方法：__init__, _blob_name, _data_blob_key, _cache_key, _get_blob, _blob_exists, _upload_bytes, _ensure_cached, _download_bytes, _delete_blob, _temp_local_copy, _cleanup_temp_files, _cleanup_scratch, __del__, _init_metadata, get_metadata, save_metadata, invalidate_metadata_cache, _atomic_update_metadata, get_file_path, file_exists, delete_table, delete_tables_by_source_file, cleanup, read_data_as_df, write_parquet_from_arrow, write_parquet, get_parquet_schema, get_parquet_path, run_parquet_sql

- **阅读建议**：先看公开类的 `run`/`handle_*`/`to_dict`，再看私有辅助。改动后运行对应 `tests/backend` 文件。涉及用户字符串或错误码时同步 i18n。

#### `py-src/data_formulator/datalake/azure_blob_workspace_manager.py`

- **目录**：`py-src/data_formulator/datalake`

- **行数**：348

- **所在子系统**：工作区、Parquet、目录缓存、命名与元数据

- **主要功能简介**：AzureBlobWorkspaceManager — manages multiple workspaces per user on Azure Blob Storage.  Extends WorkspaceManager, overriding storage operations to use Azure Blob instead of the local filesystem. Same interface, different backend.  Layout (blob prefixes):     <datalake_root>/users/<safe_id>/workspaces/<workspace_id>/       workspace_meta.json       workspace.yaml       session_state.json       dat

- **类**：AzureBlobWorkspaceManager

- **函数**：无

- **常量**：无

  - `AzureBlobWorkspaceManager` 方法：__init__, root, _ws_prefix, _blob_name, _blob_exists, _upload_blob, _download_blob, _delete_blobs_with_prefix, _list_workspace_prefixes, _upload_meta, _ensure_meta, list_workspaces, workspace_exists, get_workspace_path, create_workspace, open_workspace, create_and_open_workspace, delete_workspace, rename_workspace, update_display_name, save_session_state, load_session_state

- **阅读建议**：先看公开类的 `run`/`handle_*`/`to_dict`，再看私有辅助。改动后运行对应 `tests/backend` 文件。涉及用户字符串或错误码时同步 i18n。

#### `py-src/data_formulator/datalake/blob_disk_cache.py`

- **目录**：`py-src/data_formulator/datalake`

- **行数**：239

- **所在子系统**：工作区、Parquet、目录缓存、命名与元数据

- **主要功能简介**：Process-persistent, ETag-validated local disk cache for Azure Blob reads.  Azure blob workspaces build a *fresh* :class:`AzureBlobWorkspace` on every request, so their per-instance in-memory caches are always cold and every request re-downloads ``workspace.yaml`` and data blobs (parquet files can be many megabytes).  This module provides a single, process-global cache that survives across those sh

- **类**：CacheEntry, BlobDiskCache

- **函数**：get_blob_disk_cache

- **常量**：CACHE_DIR_NAME, _DEFAULT_MAX_BYTES

  - `CacheEntry` 方法：read_bytes

  - `BlobDiskCache` 方法：__init__, _stem, _bin_path, _meta_path, _load_index, get, is_fresh, mark_validated, put, invalidate, _atomic_write, _drop_locked, _evict_if_needed_locked

- **阅读建议**：先看公开类的 `run`/`handle_*`/`to_dict`，再看私有辅助。改动后运行对应 `tests/backend` 文件。涉及用户字符串或错误码时同步 i18n。

#### `py-src/data_formulator/datalake/catalog_cache.py`

- **目录**：`py-src/data_formulator/datalake`

- **行数**：716

- **所在子系统**：工作区、Parquet、目录缓存、命名与元数据

- **主要功能简介**：Catalog cache — persist lightweight list_tables() results to disk.  Stored as JSON files under ``<workspace_root>/catalog_cache/<source_id>.json``. Used by agents to search available data without live connections.  File format::      {         "source_id": "superset_prod",         "synced_at": "2026-04-28T10:00:00Z",         "tables": [             {                 "table_key": "a1b2c3d4-...",   

- **类**：CatalogSnapshot, CatalogSearchError

- **函数**：_utc_now, _parse_timestamp, _age_and_freshness, _cache_dir, _cache_jail, _cache_filename, _cache_file, _atomic_write_payload, _merge_catalog_listing, save_catalog, record_catalog_refresh_failure, _load_catalog_raw, load_catalog, load_catalog_snapshot, delete_catalog, list_cached_sources, _search_python, search_catalog_cache, list_sources_summary, list_path_children

- **常量**：CATALOG_CACHE_DIR, CATALOG_CACHE_SCHEMA_VERSION, LIST_DATA_LIMIT

  - `CatalogSnapshot` 方法：（无）

  - `CatalogSearchError` 方法：（无）

- **阅读建议**：先看公开类的 `run`/`handle_*`/`to_dict`，再看私有辅助。改动后运行对应 `tests/backend` 文件。涉及用户字符串或错误码时同步 i18n。

#### `py-src/data_formulator/datalake/catalog_refresh.py`

- **目录**：`py-src/data_formulator/datalake`

- **行数**：159

- **所在子系统**：工作区、Parquet、目录缓存、命名与元数据

- **主要功能简介**：该文件实现子系统中的一个具体单元，符号列表如下。

- **类**：无

- **函数**：_retry_allowed, _refresh_catalog, _complete_refresh, ensure_catalog_freshness

- **常量**：_REFRESH_EXECUTOR, _REFRESH_LOCK

- **阅读建议**：先看公开类的 `run`/`handle_*`/`to_dict`，再看私有辅助。改动后运行对应 `tests/backend` 文件。涉及用户字符串或错误码时同步 i18n。

#### `py-src/data_formulator/datalake/ephemeral_workspace.py`

- **目录**：`py-src/data_formulator/datalake`

- **行数**：184

- **所在子系统**：工作区、Parquet、目录缓存、命名与元数据

- **主要功能简介**：TTL-managed local workspaces for anonymous/demo deployments.  ``WORKSPACE_BACKEND=ephemeral`` uses the normal on-disk workspace format, but stores it under a separate root and removes inactive workspaces after a configurable TTL. A global LRU byte cap provides a second storage bound. Durable ``local`` workspaces never pass through this module.

- **类**：EphemeralWorkspaceManager

- **函数**：_positive_float_env, get_ephemeral_root, get_ephemeral_workspaces_root, _workspace_updated_at, _directory_size, _workspace_directories, _tombstone_path, _write_tombstone, cleanup_ephemeral_workspaces

- **常量**：_CLEANUP_LOCK, _LAST_CLEANUP_AT

  - `EphemeralWorkspaceManager` 方法：__init__, workspace_was_evicted, _touch, open_workspace, load_session_state

- **阅读建议**：先看公开类的 `run`/`handle_*`/`to_dict`，再看私有辅助。改动后运行对应 `tests/backend` 文件。涉及用户字符串或错误码时同步 i18n。

#### `py-src/data_formulator/datalake/file_manager.py`

- **目录**：`py-src/data_formulator/datalake`

- **行数**：367

- **所在子系统**：工作区、Parquet、目录缓存、命名与元数据

- **主要功能简介**：File manager for user-uploaded files in the Data Lake.  This module handles storing user-uploaded files (CSV, Excel, TXT, HTML, JSON, PDF) as-is in the workspace without conversion.

- **类**：无

- **函数**：normalize_text_encoding, is_supported_file, get_file_type, compute_file_hash, sanitize_table_name, generate_unique_filename, save_uploaded_file, save_uploaded_file_from_path, get_file_info

- **常量**：_TEXT_FILE_TYPES, SUPPORTED_EXTENSIONS, _TRUSTED_DETECTIONS

- **阅读建议**：先看公开类的 `run`/`handle_*`/`to_dict`，再看私有辅助。改动后运行对应 `tests/backend` 文件。涉及用户字符串或错误码时同步 i18n。

#### `py-src/data_formulator/datalake/naming.py`

- **目录**：`py-src/data_formulator/datalake`

- **行数**：54

- **所在子系统**：工作区、Parquet、目录缓存、命名与元数据

- **主要功能简介**：Lightweight ID / filename sanitisation helpers.  No heavy dependencies (no pandas, pyarrow, etc.) so that both :mod:`data_connector` and :mod:`datalake.catalog_cache` can import without pulling in the data stack.  For **table-name** sanitisation see :mod:`datalake.table_names`. For **data-file** sanitisation see :func:`datalake.parquet_utils.safe_data_filename`.

- **类**：无

- **函数**：safe_source_id

- **常量**：_SAFE_SOURCE_ID_RE

- **阅读建议**：先看公开类的 `run`/`handle_*`/`to_dict`，再看私有辅助。改动后运行对应 `tests/backend` 文件。涉及用户字符串或错误码时同步 i18n。

#### `py-src/data_formulator/datalake/parquet_utils.py`

- **目录**：`py-src/data_formulator/datalake`

- **行数**：257

- **所在子系统**：工作区、Parquet、目录缓存、命名与元数据

- **主要功能简介**：Parquet utility functions for the Data Lake.  Pure helper functions for parquet I/O, hashing, column introspection, and name sanitisation.  These utilities have **no dependency on Workspace** and are consumed by Workspace methods that handle metadata bookkeeping.

- **类**：无

- **函数**：safe_data_filename, sanitize_table_name, get_sample_rows_from_arrow, df_to_safe_records, _repair_invalid_unicode, normalize_dtype_to_app_type, get_arrow_column_info, get_column_info, compute_arrow_table_hash, sanitize_dataframe_for_arrow, compute_dataframe_hash

- **常量**：DEFAULT_COMPRESSION, DEFAULT_METADATA_SAMPLE_ROWS

- **阅读建议**：先看公开类的 `run`/`handle_*`/`to_dict`，再看私有辅助。改动后运行对应 `tests/backend` 文件。涉及用户字符串或错误码时同步 i18n。

#### `py-src/data_formulator/datalake/table_names.py`

- **目录**：`py-src/data_formulator/datalake`

- **行数**：188

- **所在子系统**：工作区、Parquet、目录缓存、命名与元数据

- **主要功能简介**：Single source of truth for table-name sanitisation across the datalake, API, data loaders, and DuckDB SQL helpers.  Different call sites historically used slightly different rules (empty-name fallback, digit prefixes, SQL keywords, allowed punctuation). Use the function that matches the **consumer**:  * :func:`sanitize_workspace_parquet_table_name` — logical parquet/workspace   table keys, HTTP ``

- **类**：无

- **函数**：sanitize_workspace_parquet_table_name, sanitize_upload_stem_table_name, sanitize_external_loader_table_name, sanitize_duckdb_sql_table_name

- **常量**：无

- **阅读建议**：先看公开类的 `run`/`handle_*`/`to_dict`，再看私有辅助。改动后运行对应 `tests/backend` 文件。涉及用户字符串或错误码时同步 i18n。

#### `py-src/data_formulator/datalake/workspace.py`

- **目录**：`py-src/data_formulator/datalake`

- **行数**：1038

- **所在子系统**：工作区、Parquet、目录缓存、命名与元数据

- **主要功能简介**：Workspace management for the Data Lake.  Each user has a workspace directory identified by their identity_id. The workspace contains all their data files (uploaded and ingested) plus a workspace.yaml metadata file.

- **类**：Workspace, WorkspaceWithTempData

- **函数**：get_data_formulator_home, get_default_workspace_root, get_user_home, sanitize_identity_dirname, _sanitize_identity_id, _configured_scratch_max_bytes, cleanup_stale_temp_files

- **常量**：SCRATCH_MAX_BYTES

  - `Workspace` 方法：__init__, _sanitize_identity_id, user_home, confined_root, confined_data, confined_scratch, prune_scratch, _init_metadata, get_file_path, file_exists, delete_table, _atomic_update_metadata, get_metadata, save_metadata, invalidate_metadata_cache, add_table_metadata, get_table_metadata, list_tables, get_fresh_name, delete_tables_by_source_file, cleanup, get_relative_data_file_path, read_data_as_df, write_parquet_from_arrow, write_parquet, get_parquet_schema, get_parquet_path, run_parquet_sql, refresh_parquet_from_arrow, refresh_parquet

  - `WorkspaceWithTempData` 方法：__init__, __getattr__, get_table_metadata, list_tables, read_data_as_df, __enter__, __exit__

- **阅读建议**：先看公开类的 `run`/`handle_*`/`to_dict`，再看私有辅助。改动后运行对应 `tests/backend` 文件。涉及用户字符串或错误码时同步 i18n。

#### `py-src/data_formulator/datalake/workspace_manager.py`

- **目录**：`py-src/data_formulator/datalake`

- **行数**：521

- **所在子系统**：工作区、Parquet、目录缓存、命名与元数据

- **主要功能简介**：WorkspaceManager — manages multiple workspaces per user.  Each workspace is a named folder containing:   - workspace_meta.json: lightweight metadata for fast listing   - workspace.yaml: all table metadata (single file)   - session_state.json: auto-persisted frontend state   - data/: data files (parquet, csv, etc.)  Users can create, list, open, delete, and switch workspaces.

- **类**：WorkspaceManager

- **函数**：_strip_sensitive

- **常量**：SESSION_STATE_FILENAME, WORKSPACE_META_FILENAME, _SENSITIVE_FIELDS

  - `WorkspaceManager` 方法：__init__, root, _safe_id, _write_meta, _ensure_meta, _has_content, list_workspaces, workspace_exists, get_workspace_path, move_workspaces_from, _merge_workspace, delete_all_workspaces, create_workspace, open_workspace, create_and_open_workspace, delete_workspace, rename_workspace, update_display_name, save_session_state, load_session_state

- **阅读建议**：先看公开类的 `run`/`handle_*`/`to_dict`，再看私有辅助。改动后运行对应 `tests/backend` 文件。涉及用户字符串或错误码时同步 i18n。

#### `py-src/data_formulator/datalake/workspace_metadata.py`

- **目录**：`py-src/data_formulator/datalake`

- **行数**：631

- **所在子系统**：工作区、Parquet、目录缓存、命名与元数据

- **主要功能简介**：Metadata management for the Data Lake workspace.  This module defines the schema and operations for workspace.yaml, which tracks all data sources (uploaded files and data loader ingests).

- **类**：WorkspaceLock, ColumnInfo, TableMetadata, WorkspaceMetadata, ImportedFrom, Derivation

- **函数**：make_json_safe, _read_metadata_file, _write_metadata_file, load_metadata, save_metadata, update_metadata, metadata_exists

- **常量**：METADATA_VERSION, METADATA_FILENAME, LOCK_FILENAME, MAX_LOCK_WAIT_SECONDS

  - `WorkspaceLock` 方法：__init__, __enter__, __exit__

  - `ColumnInfo` 方法：to_dict, from_dict

  - `TableMetadata` 方法：to_dict, from_dict

  - `WorkspaceMetadata` 方法：add_table, remove_table, get_table, list_tables, search_tables, to_dict, from_dict, create_new

  - `ImportedFrom` 方法：to_dict, from_dict

  - `Derivation` 方法：to_dict, from_dict

- **阅读建议**：先看公开类的 `run`/`handle_*`/`to_dict`，再看私有辅助。改动后运行对应 `tests/backend` 文件。涉及用户字符串或错误码时同步 i18n。

#### `py-src/data_formulator/desktop.py`

- **目录**：`py-src/data_formulator`

- **行数**：297

- **所在子系统**：应用入口、Flask 装配、错误类型、连接器总控、模型注册、工作区工厂

- **主要功能简介**：该文件实现子系统中的一个具体单元，符号列表如下。

- **类**：无

- **函数**：_configure_standard_streams, _signal_existing_instance, _claim_single_instance, _listen_for_activation, _activate_window, _available_port, _wait_until_ready, _enable_per_monitor_dpi, _run_self_test, _self_test_clr, run_desktop

- **常量**：_INSTANCE_HOST, _INSTANCE_PORT, _ACTIVATE_MESSAGE, _ACTIVATE_ACK, _LOADING_HTML

- **阅读建议**：先看公开类的 `run`/`handle_*`/`to_dict`，再看私有辅助。改动后运行对应 `tests/backend` 文件。涉及用户字符串或错误码时同步 i18n。

#### `py-src/data_formulator/error_handler.py`

- **目录**：`py-src/data_formulator`

- **行数**：350

- **所在子系统**：应用入口、Flask 装配、错误类型、连接器总控、模型注册、工作区工厂

- **主要功能简介**：Unified error handling and response helpers for the Data Formulator Flask application.  Public entry points:  * ``register_error_handlers(app)`` — call once during app setup to install   global error handlers and the request-id middleware. * ``classify_and_wrap_llm_error(exc)`` — convert a raw LLM / external-API   exception into a structured ``AppError``. * ``stream_error_event(error)`` — format a

- **类**：无

- **函数**：_safe_unexpected_detail, classify_and_wrap_llm_error, stream_error_event, stream_warning_event, collect_stream_warning, flush_stream_warnings, json_ok, stream_preflight_error, register_error_handlers

- **常量**：无

- **阅读建议**：先看公开类的 `run`/`handle_*`/`to_dict`，再看私有辅助。改动后运行对应 `tests/backend` 文件。涉及用户字符串或错误码时同步 i18n。

#### `py-src/data_formulator/errors.py`

- **目录**：`py-src/data_formulator`

- **行数**：149

- **所在子系统**：应用入口、Flask 装配、错误类型、连接器总控、模型注册、工作区工厂

- **主要功能简介**：Unified error types for the Data Formulator backend.  Every business error raised in routes / agents / data layer should be an ``AppError`` (or a subclass).  The global error handlers registered by ``error_handler.register_error_handlers`` convert ``AppError`` instances into a consistent JSON envelope before they reach the client.  ``ErrorCode`` provides machine-readable codes that the frontend ma

- **类**：ErrorCode, AppError

- **函数**：无

- **常量**：无

  - `ErrorCode` 方法：（无）

  - `AppError` 方法：__init__, get_http_status, to_dict

- **阅读建议**：先看公开类的 `run`/`handle_*`/`to_dict`，再看私有辅助。改动后运行对应 `tests/backend` 文件。涉及用户字符串或错误码时同步 i18n。

#### `py-src/data_formulator/example_datasets_config.py`

- **目录**：`py-src/data_formulator`

- **行数**：420

- **所在子系统**：应用入口、Flask 装配、错误类型、连接器总控、模型注册、工作区工厂

- **主要功能简介**：Sample datasets configuration for Data Formulator.

- **类**：无

- **函数**：无

- **常量**：EXAMPLE_DATASETS

- **阅读建议**：先看公开类的 `run`/`handle_*`/`to_dict`，再看私有辅助。改动后运行对应 `tests/backend` 文件。涉及用户字符串或错误码时同步 i18n。

#### `py-src/data_formulator/knowledge/__init__.py`

- **目录**：`py-src/data_formulator/knowledge`

- **行数**：3

- **所在子系统**：规则 / 工作流 / data-memory 存储

- **主要功能简介**：该文件实现子系统中的一个具体单元，符号列表如下。

- **类**：无

- **函数**：无

- **常量**：无

- **阅读建议**：先看公开类的 `run`/`handle_*`/`to_dict`，再看私有辅助。改动后运行对应 `tests/backend` 文件。涉及用户字符串或错误码时同步 i18n。

#### `py-src/data_formulator/knowledge/store.py`

- **目录**：`py-src/data_formulator/knowledge`

- **行数**：727

- **所在子系统**：规则 / 工作流 / data-memory 存储

- **主要功能简介**：Knowledge store — manages user knowledge files and data-source memory.  Each user has a ``knowledge/`` directory under their home with two sub-directories: ``rules`` and ``workflows``.  Every knowledge entry is a Markdown file with YAML front matter.  All file I/O is routed through :class:`ConfinedDir` for path safety.  Directory depth constraints:  - ``rules``: flat — only files directly under ``

- **类**：KnowledgeItemMeta, KnowledgeStore

- **函数**：_tokenize_query, parse_front_matter, _ensure_front_matter

- **常量**：VALID_CATEGORIES, DATA_MEMORY_FILE, DATA_MEMORY_HARD_MAX, DATA_MEMORY_TEMPLATE, _MAX_DEPTH, _ENGLISH_STOPWORDS, _MIN_TOKEN_LEN, _CJK_ASCII_RE, _FM_PATTERN

  - `KnowledgeItemMeta` 方法：__init__, from_raw

  - `KnowledgeStore` 方法：__init__, read_data_memory, rewrite_data_memory, append_data_memory, replace_data_memory, _migrate_experiences_to_workflows, _migrate_flat, validate_path, _jail, list_all, read, write, delete, find_workflow_by_workspace_id, load_always_apply_rules, format_rules_block, search, _match_score

- **阅读建议**：先看公开类的 `run`/`handle_*`/`to_dict`，再看私有辅助。改动后运行对应 `tests/backend` 文件。涉及用户字符串或错误码时同步 i18n。

#### `py-src/data_formulator/model_registry.py`

- **目录**：`py-src/data_formulator`

- **行数**：114

- **所在子系统**：应用入口、Flask 装配、错误类型、连接器总控、模型注册、工作区工厂

- **主要功能简介**：该文件实现子系统中的一个具体单元，符号列表如下。

- **类**：ModelRegistry

- **函数**：无

- **常量**：BUILTIN_PROVIDERS

  - `ModelRegistry` 方法：__init__, make_id, _discover_providers, _reload, get_config, list_public, is_global

- **阅读建议**：先看公开类的 `run`/`handle_*`/`to_dict`，再看私有辅助。改动后运行对应 `tests/backend` 文件。涉及用户字符串或错误码时同步 i18n。

#### `py-src/data_formulator/routes/__init__.py`

- **目录**：`py-src/data_formulator/routes`

- **行数**：3

- **所在子系统**：HTTP 蓝图：表、Agent、会话、知识、凭证、日志

- **主要功能简介**：该文件实现子系统中的一个具体单元，符号列表如下。

- **类**：无

- **函数**：无

- **常量**：无

- **阅读建议**：先看公开类的 `run`/`handle_*`/`to_dict`，再看私有辅助。改动后运行对应 `tests/backend` 文件。涉及用户字符串或错误码时同步 i18n。

#### `py-src/data_formulator/routes/agents.py`

- **目录**：`py-src/data_formulator/routes`

- **行数**：1096

- **所在子系统**：HTTP 蓝图：表、Agent、会话、知识、凭证、日志

- **主要功能简介**：该文件实现子系统中的一个具体单元，符号列表如下。

- **类**：无

- **函数**：_get_ui_lang, get_language_instruction, _get_knowledge_store, preview_data_operation, _with_warnings, _set_cors, get_client, list_global_models, check_available_models, test_model, process_data_on_load_request, sort_data_request, derive_starter_questions_request, analyst_streaming, request_code_expl, refresh_derived_data, workspace_name, nl_to_filter, classify_chart_intent, chart_restyle, scratch_upload, scratch_serve, data_loading_chat

- **常量**：PREVIEW_ROW_LIMIT

- **阅读建议**：先看公开类的 `run`/`handle_*`/`to_dict`，再看私有辅助。改动后运行对应 `tests/backend` 文件。涉及用户字符串或错误码时同步 i18n。

#### `py-src/data_formulator/routes/credentials.py`

- **目录**：`py-src/data_formulator/routes`

- **行数**：74

- **所在子系统**：HTTP 蓝图：表、Agent、会话、知识、凭证、日志

- **主要功能简介**：REST API for credential management (list / store / delete).  All endpoints are identity-scoped: the current user (from :func:`get_identity_id`) can only access their own stored credentials. Credential *values* are never returned to the frontend — ``/list`` only reveals which source_keys have stored credentials.

- **类**：无

- **函数**：list_credentials, store_credential, delete_credential

- **常量**：无

- **阅读建议**：先看公开类的 `run`/`handle_*`/`to_dict`，再看私有辅助。改动后运行对应 `tests/backend` 文件。涉及用户字符串或错误码时同步 i18n。

#### `py-src/data_formulator/routes/demo_stream.py`

- **目录**：`py-src/data_formulator/routes`

- **行数**：1169

- **所在子系统**：HTTP 蓝图：表、Agent、会话、知识、凭证、日志

- **主要功能简介**：Demo data REST APIs for streaming/refresh demos.  Design Philosophy: - Each endpoint returns a COMPLETE dataset (not just a single row) - Datasets are meaningful on their own for analysis/visualization - When refreshed, datasets change over time:   * New rows may be added (accumulating data)   * Existing values may update (latest readings) - This allows tracking trends, changes, and patterns over 

- **类**：无

- **函数**：_set_cors, make_csv_response, get_earthquakes, get_weather, get_weather_history, get_weather_forecast, get_weather_today, _yf_is_valid, _yf_format_timestamp, get_yfinance_history, get_yfinance_recent, get_yfinance_financials, _generate_sale_transaction, get_live_sales, get_info

- **常量**：EARTHQUAKE_RATE_LIMIT, WEATHER_RATE_LIMIT, YFINANCE_RATE_LIMIT, MOCK_RATE_LIMIT, WEATHER_CITIES, DEFAULT_SYMBOLS, SP100_SYMBOLS, _SALES_PRODUCTS, _SALES_REGIONS, _SALES_REGION_WEIGHTS, _SALES_CHANNELS, _SALES_CHANNEL_WEIGHTS

- **阅读建议**：先看公开类的 `run`/`handle_*`/`to_dict`，再看私有辅助。改动后运行对应 `tests/backend` 文件。涉及用户字符串或错误码时同步 i18n。

#### `py-src/data_formulator/routes/knowledge.py`

- **目录**：`py-src/data_formulator/routes`

- **行数**：393

- **所在子系统**：HTTP 蓝图：表、Agent、会话、知识、凭证、日志

- **主要功能简介**：Knowledge management API — CRUD + search + workflow distillation.  All endpoints use ``POST`` with JSON body.  Access is scoped to the current user via ``get_identity_id()`` and confined via ``ConfinedDir``.

- **类**：无

- **函数**：_get_store, _require_json_field, knowledge_limits, data_memory_read, data_memory_append, data_memory_rewrite, knowledge_list, knowledge_read, knowledge_write, knowledge_delete, knowledge_search, distill_workflow, _apply_session_front_matter, _strip_workflow_prefix, _serialize_front_matter, _workflow_filename

- **常量**：_EXP_PREFIX_RE, _UNSAFE_FILENAME_CHARS

- **阅读建议**：先看公开类的 `run`/`handle_*`/`to_dict`，再看私有辅助。改动后运行对应 `tests/backend` 文件。涉及用户字符串或错误码时同步 i18n。

#### `py-src/data_formulator/routes/logs.py`

- **目录**：`py-src/data_formulator/routes`

- **行数**：117

- **所在子系统**：HTTP 蓝图：表、Agent、会话、知识、凭证、日志

- **主要功能简介**：Server log inspection routes.  Data Formulator persists all server + Python-execution logs to a rotating file under ``<DATA_FORMULATOR_HOME>/logs/data_formulator.log`` (configured in ``app.configure_file_logging``). This is the artifact a user can send when reporting a problem.  Access policy — logs are **server-side only**:  * In **local single-user mode** (``is_local_mode()`` is true) the user *

- **类**：无

- **函数**：_require_local_mode, _log_path, logs_info, logs_tail, logs_download

- **常量**：_MAX_TAIL_LINES, _DEFAULT_TAIL_LINES

- **阅读建议**：先看公开类的 `run`/`handle_*`/`to_dict`，再看私有辅助。改动后运行对应 `tests/backend` 文件。涉及用户字符串或错误码时同步 i18n。

#### `py-src/data_formulator/routes/model_endpoints.py`

- **目录**：`py-src/data_formulator/routes`

- **行数**：94

- **所在子系统**：HTTP 蓝图：表、Agent、会话、知识、凭证、日志

- **主要功能简介**：Per-user history of non-secret model endpoint configurations.

- **类**：无

- **函数**：_history_path, _sanitize_entry, _read_history, _write_history, list_model_endpoints, remember_model_endpoint

- **常量**：_FILENAME, _MAX_ENTRIES, _MAX_FIELD_LENGTH, _FIELDS

- **阅读建议**：先看公开类的 `run`/`handle_*`/`to_dict`，再看私有辅助。改动后运行对应 `tests/backend` 文件。涉及用户字符串或错误码时同步 i18n。

#### `py-src/data_formulator/routes/sessions.py`

- **目录**：`py-src/data_formulator/routes`

- **行数**：374

- **所在子系统**：HTTP 蓝图：表、Agent、会话、知识、凭证、日志

- **主要功能简介**：Workspace management routes.  All backends expose the same workspace CRUD API. The ephemeral backend selects a TTL-managed local WorkspaceManager in ``workspace_factory``.  Routes:   POST /api/sessions/save        — auto-persist state to active workspace   GET  /api/sessions/list        — list all workspaces   POST /api/sessions/load        — switch to a workspace (open it)   POST /api/sessions/de

- **类**：无

- **函数**：_raise_if_storage_full, save_session, list_sessions, load_session, delete_session, create_workspace_route, rename_workspace_route, update_workspace_meta, export_session, import_session, migrate_workspaces, cleanup_anonymous

- **常量**：无

- **阅读建议**：先看公开类的 `run`/`handle_*`/`to_dict`，再看私有辅助。改动后运行对应 `tests/backend` 文件。涉及用户字符串或错误码时同步 i18n。

#### `py-src/data_formulator/routes/tables.py`

- **目录**：`py-src/data_formulator/routes`

- **行数**：1267

- **所在子系统**：HTTP 蓝图：表、Agent、会话、知识、凭证、日志

- **主要功能简介**：该文件实现子系统中的一个具体单元，符号列表如下。

- **类**：无

- **函数**：_get_workspace, _should_use_duckdb, _quote_duckdb, _quote_lit, _column_type_map, _extend_eod_if_timestamp, _build_filter_where_duckdb, _apply_filters_pandas, _dedup_dataframe_columns, _dedup_list, _build_parquet_sample_sql, _table_metadata_to_source_metadata, open_workspace, list_tables, _apply_aggregation_and_sample, _fetch_column_levels_duckdb, _safe_levels, sample_table, get_table_data, _read_upload_to_df, _resolve_excel_sheet, create_table, parse_file, sync_table_data, drop_table, upload_db_file, download_db_file, _stream_csv_from_duckdb, _stream_csv_from_dataframe, export_table_csv, reset_db_file, _is_numeric_duckdb_type, analyze_table, sanitize_table_name, classify_and_raise_db_error, sanitize_db_error_message

- **常量**：_LARGE_TABLE_THRESHOLD, _COLUMN_STATS_LEVELS_LIMIT, _CSV_STREAM_CHUNK_ROWS

- **阅读建议**：先看公开类的 `run`/`handle_*`/`to_dict`，再看私有辅助。改动后运行对应 `tests/backend` 文件。涉及用户字符串或错误码时同步 i18n。

#### `py-src/data_formulator/sandbox/__init__.py`

- **目录**：`py-src/data_formulator/sandbox`

- **行数**：22

- **所在子系统**：代码隔离执行：local / docker / not_a_sandbox

- **主要功能简介**：该文件实现子系统中的一个具体单元，符号列表如下。

- **类**：无

- **函数**：create_sandbox

- **常量**：SANDBOX_OPTIONS

- **阅读建议**：先看公开类的 `run`/`handle_*`/`to_dict`，再看私有辅助。改动后运行对应 `tests/backend` 文件。涉及用户字符串或错误码时同步 i18n。

#### `py-src/data_formulator/sandbox/base.py`

- **目录**：`py-src/data_formulator/sandbox`

- **行数**：55

- **所在子系统**：代码隔离执行：local / docker / not_a_sandbox

- **主要功能简介**：Abstract base class for code-execution sandboxes.  Every sandbox backend must subclass :class:`Sandbox` and implement :meth:`run_python_code`.  The return contract is a dict with:  * ``{'status': 'ok', 'content': <pandas.DataFrame>}``  on success * ``{'status': 'error', 'content': '<error message>'}`` on failure

- **类**：Sandbox

- **函数**：无

- **常量**：无

  - `Sandbox` 方法：run_python_code

- **阅读建议**：先看公开类的 `run`/`handle_*`/`to_dict`，再看私有辅助。改动后运行对应 `tests/backend` 文件。涉及用户字符串或错误码时同步 i18n。

#### `py-src/data_formulator/sandbox/docker_sandbox.py`

- **目录**：`py-src/data_formulator/sandbox`

- **行数**：263

- **所在子系统**：代码隔离执行：local / docker / not_a_sandbox

- **主要功能简介**：Docker-based sandbox for executing Python code in an isolated container.  The workspace directory is mounted **read-only** as the container's working directory so user scripts can read data files via e.g. ``pd.read_csv("file.csv")`` but cannot tamper with the host filesystem. The output DataFrame is serialised to Parquet and read back via a bind-mounted output directory.

- **类**：DockerSandbox

- **函数**：_safe_error_response

- **常量**：DEFAULT_DOCKER_IMAGE, DEFAULT_TIMEOUT

  - `DockerSandbox` 方法：__init__, run_python_code, _cleanup

- **阅读建议**：先看公开类的 `run`/`handle_*`/`to_dict`，再看私有辅助。改动后运行对应 `tests/backend` 文件。涉及用户字符串或错误码时同步 i18n。

#### `py-src/data_formulator/sandbox/local_sandbox.py`

- **目录**：`py-src/data_formulator/sandbox`

- **行数**：608

- **所在子系统**：代码隔离执行：local / docker / not_a_sandbox

- **主要功能简介**：Local sandbox -- executes Python code in a persistent warm subprocess.  The script runs with the workspace directory as its working directory so user scripts access files via e.g. ``pd.read_csv("sample.csv")``.

- **类**：_WarmWorkerPool, SandboxSession, LocalSandbox

- **函数**：_warm_worker_loop

- **常量**：无

  - `_WarmWorkerPool` 方法：__init__, _spawn, acquire, release, discard, shutdown

  - `SandboxSession` 方法：__init__, execute, close, save_namespace, restore_namespace, __enter__, __exit__

  - `LocalSandbox` 方法：run_python_code, _run_in_warm_subprocess

- **阅读建议**：先看公开类的 `run`/`handle_*`/`to_dict`，再看私有辅助。改动后运行对应 `tests/backend` 文件。涉及用户字符串或错误码时同步 i18n。

#### `py-src/data_formulator/sandbox/not_a_sandbox.py`

- **目录**：`py-src/data_formulator/sandbox`

- **行数**：79

- **所在子系统**：代码隔离执行：local / docker / not_a_sandbox

- **主要功能简介**：Unsandboxed main-process executor -- for benchmarking only.  This runs user code directly in the main process with no isolation. It is NOT exposed as a CLI option and should only be used to measure the raw execution overhead baseline in benchmarks.

- **类**：NotASandbox

- **函数**：无

- **常量**：无

  - `NotASandbox` 方法：run_python_code

- **阅读建议**：先看公开类的 `run`/`handle_*`/`to_dict`，再看私有辅助。改动后运行对应 `tests/backend` 文件。涉及用户字符串或错误码时同步 i18n。

#### `py-src/data_formulator/security/__init__.py`

- **目录**：`py-src/data_formulator/security`

- **行数**：3

- **所在子系统**：路径监禁、日志脱敏、代码签名、URL 白名单

- **主要功能简介**：该文件实现子系统中的一个具体单元，符号列表如下。

- **类**：无

- **函数**：无

- **常量**：无

- **阅读建议**：先看公开类的 `run`/`handle_*`/`to_dict`，再看私有辅助。改动后运行对应 `tests/backend` 文件。涉及用户字符串或错误码时同步 i18n。

#### `py-src/data_formulator/security/code_signing.py`

- **目录**：`py-src/data_formulator/security`

- **行数**：141

- **所在子系统**：路径监禁、日志脱敏、代码签名、URL 白名单

- **主要功能简介**：HMAC-based code signing for transformation code.  When the agent generates Python transformation code and the server executes it successfully, the server signs the code with a secret key. The signature is returned to the frontend alongside the code.  When the frontend later sends the code back for re-execution (e.g. during data refresh), the server verifies the signature before running the code.  

- **类**：无

- **函数**：_is_dev_mode, _get_secret, sign_code, verify_code, sign_result

- **常量**：_DEV_SECRET, MAX_CODE_SIZE

- **阅读建议**：先看公开类的 `run`/`handle_*`/`to_dict`，再看私有辅助。改动后运行对应 `tests/backend` 文件。涉及用户字符串或错误码时同步 i18n。

#### `py-src/data_formulator/security/log_sanitizer.py`

- **目录**：`py-src/data_formulator/security`

- **行数**：240

- **所在子系统**：路径监禁、日志脱敏、代码签名、URL 白名单

- **主要功能简介**：Log sanitization utilities for preventing sensitive data leakage.  Provides two layers of defense:  1. **Explicit utilities** — ``sanitize_url``, ``sanitize_params``,    ``redact_token`` — called at logging call-sites for precise control.  2. **SensitiveDataFilter** — a ``logging.Filter`` registered on handlers    as a safety net, automatically redacting patterns that slip through.  Usage::      f

- **类**：SensitiveDataFilter

- **函数**：sanitize_url, _is_sensitive_key, _sanitize_url_token, sanitize_params, redact_token, _apply_patterns

- **常量**：_REDACTED, _REDACTED_TOKEN, _RE_URL_CREDS, _RE_URL_LIKE, _SENSITIVE_KEY_NAMES, _RE_KEY_VALUE, _RE_BEARER, _RE_JWT_LIKE, _RE_DICT_SENSITIVE

  - `SensitiveDataFilter` 方法：filter

- **阅读建议**：先看公开类的 `run`/`handle_*`/`to_dict`，再看私有辅助。改动后运行对应 `tests/backend` 文件。涉及用户字符串或错误码时同步 i18n。

#### `py-src/data_formulator/security/path_safety.py`

- **目录**：`py-src/data_formulator/security`

- **行数**：136

- **所在子系统**：路径监禁、日志脱敏、代码签名、URL 白名单

- **主要功能简介**：Path confinement primitive — prevents path traversal at the API level.  Usage::      jail = ConfinedDir("/tmp/workspace")     safe = jail / "data/sales.parquet"        # OK     jail / "../etc/passwd"                     # raises ValueError     jail.write("data/out.parquet", raw_bytes)  # resolve + mkdir + write

- **类**：ConfinedDir

- **函数**：无

- **常量**：无

  - `ConfinedDir` 方法：__init__, root, resolve, write, read_text, write_text, exists, iterdir, rglob, unlink, __truediv__, __repr__

- **阅读建议**：先看公开类的 `run`/`handle_*`/`to_dict`，再看私有辅助。改动后运行对应 `tests/backend` 文件。涉及用户字符串或错误码时同步 i18n。

#### `py-src/data_formulator/security/sanitize.py`

- **目录**：`py-src/data_formulator/security`

- **行数**：210

- **所在子系统**：路径监禁、日志脱敏、代码签名、URL 白名单

- **主要功能简介**：Shared helpers for sanitizing error messages before they reach the client.

- **类**：无

- **函数**：_extract_traceback_summary, _structured_error_response, safe_error_response, classify_llm_error, sanitize_error_message

- **常量**：_GENERIC_5XX, _GENERIC_502, _GENERIC_4XX, _LLM_ERROR_GENERIC

- **阅读建议**：先看公开类的 `run`/`handle_*`/`to_dict`，再看私有辅助。改动后运行对应 `tests/backend` 文件。涉及用户字符串或错误码时同步 i18n。

#### `py-src/data_formulator/security/url_allowlist.py`

- **目录**：`py-src/data_formulator/security`

- **行数**：104

- **所在子系统**：路径监禁、日志脱敏、代码签名、URL 白名单

- **主要功能简介**：URL allowlist for user-provided LLM API base URLs.  When a user adds a custom model via the UI, they can supply an arbitrary ``api_base`` URL.  The server then makes outbound HTTP requests to that URL on behalf of the user.  Without validation this is a **Server-Side Request Forgery (SSRF)** vector — a malicious ``api_base`` could target internal services, cloud metadata endpoints, or private-netw

- **类**：无

- **函数**：_load_patterns, _is_allowlist_configured, validate_api_base

- **常量**：_ENV_KEY

- **阅读建议**：先看公开类的 `run`/`handle_*`/`to_dict`，再看私有辅助。改动后运行对应 `tests/backend` 文件。涉及用户字符串或错误码时同步 i18n。

#### `py-src/data_formulator/workflows/__init__.py`

- **目录**：`py-src/data_formulator/workflows`

- **行数**：1

- **所在子系统**：语义类型到 Vega-Lite 图表的组装算法

- **主要功能简介**：该文件实现子系统中的一个具体单元，符号列表如下。

- **类**：无

- **函数**：无

- **常量**：无

- **阅读建议**：先看公开类的 `run`/`handle_*`/`to_dict`，再看私有辅助。改动后运行对应 `tests/backend` 文件。涉及用户字符串或错误码时同步 i18n。

#### `py-src/data_formulator/workflows/chart_semantics.py`

- **目录**：`py-src/data_formulator/workflows`

- **行数**：600

- **所在子系统**：语义类型到 Vega-Lite 图表的组装算法

- **主要功能简介**：============================================================================= CHART SEMANTICS — Lightweight type resolution for VL spec assembly =============================================================================  Provides semantic-aware type resolution for create_vl_plots.py:   - Type registry (maps semantic types → VL encoding types)   - VL type resolution (nominal / ordinal / temporal

- **类**：TypeRegistryEntry, ChannelSemantics

- **函数**：get_registry_entry, is_registered, resolve_vl_type, _looks_like_year_integers, _infer_vl_type_from_data, _is_likely_timestamp, _timestamp_to_ms, _looks_like_date, _try_parse_date, infer_ordinal_sort_order, _match_sequence, _expand_to_full_year, convert_temporal_data, _extract_sem_type, resolve_channel_semantics

- **常量**：_UNKNOWN, _MAX_TIMESTAMP_SEC, _MAX_TIMESTAMP_MS, _DATE_PATTERNS, _MONTH_FULL, _MONTH_ABBR, _MONTH_NUM, _DOW_FULL, _DOW_ABBR, _DOW_FULL_SUN, _DOW_ABBR_SUN, _QUARTER, _COMPASS_8, _COMPASS_4

  - `TypeRegistryEntry` 方法：（无）

  - `ChannelSemantics` 方法：（无）

- **阅读建议**：先看公开类的 `run`/`handle_*`/`to_dict`，再看私有辅助。改动后运行对应 `tests/backend` 文件。涉及用户字符串或错误码时同步 i18n。

#### `py-src/data_formulator/workflows/create_vl_plots.py`

- **目录**：`py-src/data_formulator/workflows`

- **行数**：2018

- **所在子系统**：语义类型到 Vega-Lite 图表的组装算法

- **主要功能简介**：该文件实现子系统中的一个具体单元，符号列表如下。

- **类**：无

- **函数**：field_metadata_to_semantic_types, resolve_field_type, detect_field_type, coerce_field_type, get_chart_template, create_chart_spec, fields_to_encodings, assemble_vegailte_chart, _build_initial_spec, _post_process_chart, _post_process_lollipop, _post_process_regression, _post_process_ranged_dot, _post_process_candlestick, _post_process_waterfall, _post_process_density, _post_process_radar, _post_process_pyramid, _post_process_streamgraph, _post_process_bump, _post_process_strip, _post_process_rose, _apply_semantic_encoding, _apply_spec_quality, _apply_chart_config, _get_top_values, vl_spec_to_png, spec_to_base64

- **常量**：CHART_TEMPLATES, _BAR_LIKE_CHARTS

- **阅读建议**：先看公开类的 `run`/`handle_*`/`to_dict`，再看私有辅助。改动后运行对应 `tests/backend` 文件。涉及用户字符串或错误码时同步 i18n。

#### `py-src/data_formulator/workspace_factory.py`

- **目录**：`py-src/data_formulator`

- **行数**：149

- **所在子系统**：应用入口、Flask 装配、错误类型、连接器总控、模型注册、工作区工厂

- **主要功能简介**：Flask-aware workspace factory.  Reads the workspace backend configuration from Flask's ``current_app.config`` (populated by CLI args / env vars in ``app.py``) and returns the appropriate :class:`Workspace` subclass.  This keeps the data-layer modules (``datalake.workspace``, ``datalake.azure_blob_workspace``) free of any Flask dependency.  Multi-workspace support:   - Each user has a WorkspaceMana

- **类**：无

- **函数**：_build_azure_container_client, _get_user_workspaces_root, _get_backend, get_workspace_manager, get_active_workspace_id, get_workspace

- **常量**：无

- **阅读建议**：先看公开类的 `run`/`handle_*`/`to_dict`，再看私有辅助。改动后运行对应 `tests/backend` 文件。涉及用户字符串或错误码时同步 i18n。


### 2.2 前端 TypeScript/TSX 文件

#### `src/api/knowledgeApi.ts`

- **目录**：`src/api`

- **行数**：192

- **字节**：5964

- **导出符号**：KnowledgeCategory, KnowledgeItem, KnowledgeLimits, KnowledgeSearchResult, readDataMemory, appendDataMemory, rewriteDataMemory, fetchKnowledgeLimits, listKnowledge, readKnowledge, writeKnowledge, deleteKnowledge, searchKnowledge, DistillWorkflowResult, SessionWorkflowContext, distillSessionWorkflow

- **主要功能简介**：文件路径表明其所属分层（app 状态、views 页面、components 可复用、data 类型、i18n）。该文件约 192 行，承担界面或客户端协议的一部分。与后端的耦合点通常是 `getUrls()` 与 `apiRequest`/`streamRequest`。

- **修改注意**：用户可见字符串走 i18n；API 错误走 handleApiError；不要在 reducer 里做网络请求；异步放 thunk。

#### `src/app/App.tsx`

- **目录**：`src/app`

- **行数**：1779

- **字节**：84658

- **导出符号**：toolName, AppFCProps, AppFC

- **主要功能简介**：文件路径表明其所属分层（app 状态、views 页面、components 可复用、data 类型、i18n）。该文件约 1779 行，承担界面或客户端协议的一部分。与后端的耦合点通常是 `getUrls()` 与 `apiRequest`/`streamRequest`。

- **修改注意**：用户可见字符串走 i18n；API 错误走 handleApiError；不要在 reducer 里做网络请求；异步放 thunk。

#### `src/app/AuthButton.tsx`

- **目录**：`src/app`

- **行数**：173

- **字节**：6472

- **导出符号**：AuthButton

- **主要功能简介**：文件路径表明其所属分层（app 状态、views 页面、components 可复用、data 类型、i18n）。该文件约 173 行，承担界面或客户端协议的一部分。与后端的耦合点通常是 `getUrls()` 与 `apiRequest`/`streamRequest`。

- **修改注意**：用户可见字符串走 i18n；API 错误走 handleApiError；不要在 reducer 里做网络请求；异步放 thunk。

#### `src/app/IdentityMigrationDialog.tsx`

- **目录**：`src/app`

- **行数**：156

- **字节**：5595

- **导出符号**：MigrationDialogProps, IdentityMigrationDialog

- **主要功能简介**：文件路径表明其所属分层（app 状态、views 页面、components 可复用、data 类型、i18n）。该文件约 156 行，承担界面或客户端协议的一部分。与后端的耦合点通常是 `getUrls()` 与 `apiRequest`/`streamRequest`。

- **修改注意**：用户可见字符串走 i18n；API 错误走 handleApiError；不要在 reducer 里做网络请求；异步放 thunk。

#### `src/app/LayoutProvider.tsx`

- **目录**：`src/app`

- **行数**：234

- **字节**：9673

- **导出符号**：DensityPreference, LayoutContextValue, LayoutProvider, useLayout, useContainerSize, useSettledValue

- **主要功能简介**：文件路径表明其所属分层（app 状态、views 页面、components 可复用、data 类型、i18n）。该文件约 234 行，承担界面或客户端协议的一部分。与后端的耦合点通常是 `getUrls()` 与 `apiRequest`/`streamRequest`。

- **修改注意**：用户可见字符串走 i18n；API 错误走 handleApiError；不要在 reducer 里做网络请求；异步放 thunk。

#### `src/app/OidcCallback.tsx`

- **目录**：`src/app`

- **行数**：119

- **字节**：4051

- **导出符号**：OidcCallback

- **主要功能简介**：文件路径表明其所属分层（app 状态、views 页面、components 可复用、data 类型、i18n）。该文件约 119 行，承担界面或客户端协议的一部分。与后端的耦合点通常是 `getUrls()` 与 `apiRequest`/`streamRequest`。

- **修改注意**：用户可见字符串走 i18n；API 错误走 handleApiError；不要在 reducer 里做网络请求；异步放 thunk。

#### `src/app/agentInteractionPolicy.ts`

- **目录**：`src/app`

- **行数**：7

- **字节**：200

- **导出符号**：shouldAutoFocusGeneratedChart

- **主要功能简介**：文件路径表明其所属分层（app 状态、views 页面、components 可复用、data 类型、i18n）。该文件约 7 行，承担界面或客户端协议的一部分。与后端的耦合点通常是 `getUrls()` 与 `apiRequest`/`streamRequest`。

- **修改注意**：用户可见字符串走 i18n；API 错误走 handleApiError；不要在 reducer 里做网络请求；异步放 thunk。

#### `src/app/apiClient.ts`

- **目录**：`src/app`

- **行数**：317

- **字节**：10071

- **导出符号**：ApiError, StreamEvent, ApiRequestError, parseApiResponse, parseStreamLine, apiRequest, assertDownloadResponseOk

- **主要功能简介**：文件路径表明其所属分层（app 状态、views 页面、components 可复用、data 类型、i18n）。该文件约 317 行，承担界面或客户端协议的一部分。与后端的耦合点通常是 `getUrls()` 与 `apiRequest`/`streamRequest`。

- **修改注意**：用户可见字符串走 i18n；API 错误走 handleApiError；不要在 reducer 里做网络请求；异步放 thunk。

#### `src/app/chartCache.ts`

- **目录**：`src/app`

- **行数**：136

- **字节**：4755

- **导出符号**：ChartCacheEntry, getCachedChart, setCachedChart, invalidateChart, clearCache, getChartPngDataUrl, downscaleImageForAgent, computeCacheKey

- **主要功能简介**：文件路径表明其所属分层（app 状态、views 页面、components 可复用、data 类型、i18n）。该文件约 136 行，承担界面或客户端协议的一部分。与后端的耦合点通常是 `getUrls()` 与 `apiRequest`/`streamRequest`。

- **修改注意**：用户可见字符串走 i18n；API 错误走 handleApiError；不要在 reducer 里做网络请求；异步放 thunk。

#### `src/app/chartRecommendation.ts`

- **目录**：`src/app`

- **行数**：113

- **字节**：4211

- **导出符号**：resolveRecommendedChart, resolveChartFields

- **主要功能简介**：文件路径表明其所属分层（app 状态、views 页面、components 可复用、data 类型、i18n）。该文件约 113 行，承担界面或客户端协议的一部分。与后端的耦合点通常是 `getUrls()` 与 `apiRequest`/`streamRequest`。

- **修改注意**：用户可见字符串走 i18n；API 错误走 handleApiError；不要在 reducer 里做网络请求；异步放 thunk。

#### `src/app/clarification.ts`

- **目录**：`src/app`

- **行数**：119

- **字节**：4631

- **导出符号**：NormalizedClarification, normalizeClarifyEvent, formatClarificationResponses

- **主要功能简介**：文件路径表明其所属分层（app 状态、views 页面、components 可复用、data 类型、i18n）。该文件约 119 行，承担界面或客户端协议的一部分。与后端的耦合点通常是 `getUrls()` 与 `apiRequest`/`streamRequest`。

- **修改注意**：用户可见字符串走 i18n；API 错误走 handleApiError；不要在 reducer 里做网络请求；异步放 thunk。

#### `src/app/connectorFormPersistence.ts`

- **目录**：`src/app`

- **行数**：16

- **字节**：762

- **导出符号**：stripConnectorPrefillFromEntries

- **主要功能简介**：文件路径表明其所属分层（app 状态、views 页面、components 可复用、data 类型、i18n）。该文件约 16 行，承担界面或客户端协议的一部分。与后端的耦合点通常是 `getUrls()` 与 `apiRequest`/`streamRequest`。

- **修改注意**：用户可见字符串走 i18n；API 错误走 handleApiError；不要在 reducer 里做网络请求；异步放 thunk。

#### `src/app/connectorNames.ts`

- **目录**：`src/app`

- **行数**：39

- **字节**：948

- **导出符号**：deriveConnectorDisplayName

- **主要功能简介**：文件路径表明其所属分层（app 状态、views 页面、components 可复用、data 类型、i18n）。该文件约 39 行，承担界面或客户端协议的一部分。与后端的耦合点通常是 `getUrls()` 与 `apiRequest`/`streamRequest`。

- **修改注意**：用户可见字符串走 i18n；API 错误走 handleApiError；不要在 reducer 里做网络请求；异步放 thunk。

#### `src/app/dfSlice.tsx`

- **目录**：`src/app`

- **行数**：2761

- **字节**：135191

- **导出符号**：generateFreshChart, SSEMessage, ServerConfig, ModelConfig, FocusedId, DEFAULT_ROW_LIMIT, ClientConfig, GeneratedReport, DataFormulatorState, fetchFieldSemanticType, generateStarterQuestions, fetchColumnStats, fetchCodeExpl, fetchGlobalModelList, fetchAvailableModels, dataFormulatorSlice, selectTableIds, selectRefreshConfigs, dfSelectors, getDataFieldItems, dfActions, dataFormulatorReducer

- **主要功能简介**：文件路径表明其所属分层（app 状态、views 页面、components 可复用、data 类型、i18n）。该文件约 2761 行，承担界面或客户端协议的一部分。与后端的耦合点通常是 `getUrls()` 与 `apiRequest`/`streamRequest`。

- **修改注意**：用户可见字符串走 i18n；API 错误走 handleApiError；不要在 reducer 里做网络请求；异步放 thunk。

#### `src/app/displayRowsCache.ts`

- **目录**：`src/app`

- **行数**：44

- **字节**：1521

- **导出符号**：DisplayRowsEntry, displayRowsCache, computeDisplayRowsCacheKey

- **主要功能简介**：文件路径表明其所属分层（app 状态、views 页面、components 可复用、data 类型、i18n）。该文件约 44 行，承担界面或客户端协议的一部分。与后端的耦合点通常是 `getUrls()` 与 `apiRequest`/`streamRequest`。

- **修改注意**：用户可见字符串走 i18n；API 错误走 handleApiError；不要在 reducer 里做网络请求；异步放 thunk。

#### `src/app/errorCodes.ts`

- **目录**：`src/app`

- **行数**：72

- **字节**：2328

- **导出符号**：ERROR_CODE_I18N_MAP, getErrorMessage

- **主要功能简介**：文件路径表明其所属分层（app 状态、views 页面、components 可复用、data 类型、i18n）。该文件约 72 行，承担界面或客户端协议的一部分。与后端的耦合点通常是 `getUrls()` 与 `apiRequest`/`streamRequest`。

- **修改注意**：用户可见字符串走 i18n；API 错误走 handleApiError；不要在 reducer 里做网络请求；异步放 thunk。

#### `src/app/errorHandler.ts`

- **目录**：`src/app`

- **行数**：123

- **字节**：3854

- **导出符号**：extractErrorMessage, HandleApiErrorOptions, handleApiError

- **主要功能简介**：文件路径表明其所属分层（app 状态、views 页面、components 可复用、data 类型、i18n）。该文件约 123 行，承担界面或客户端协议的一部分。与后端的耦合点通常是 `getUrls()` 与 `apiRequest`/`streamRequest`。

- **修改注意**：用户可见字符串走 i18n；API 错误走 handleApiError；不要在 reducer 里做网络请求；异步放 thunk。

#### `src/app/identity.ts`

- **目录**：`src/app`

- **行数**：121

- **字节**：3568

- **导出符号**：IdentityType, Identity, UserInfo, generateUUID, getBrowserId, clearBrowserId, resolveIdentity, getIdentityKey

- **主要功能简介**：文件路径表明其所属分层（app 状态、views 页面、components 可复用、data 类型、i18n）。该文件约 121 行，承担界面或客户端协议的一部分。与后端的耦合点通常是 `getUrls()` 与 `apiRequest`/`streamRequest`。

- **修改注意**：用户可见字符串走 i18n；API 错误走 handleApiError；不要在 reducer 里做网络请求；异步放 thunk。

#### `src/app/inputTablePreviewCache.ts`

- **目录**：`src/app`

- **行数**：49

- **字节**：1670

- **导出符号**：INPUT_TABLE_PREVIEW_ROW_LIMIT, getInputTablePreview, setInputTablePreview, invalidateInputTablePreview, clearInputTablePreviewCache, replaceInputTablePreviews

- **主要功能简介**：文件路径表明其所属分层（app 状态、views 页面、components 可复用、data 类型、i18n）。该文件约 49 行，承担界面或客户端协议的一部分。与后端的耦合点通常是 `getUrls()` 与 `apiRequest`/`streamRequest`。

- **修改注意**：用户可见字符串走 i18n；API 错误走 handleApiError；不要在 reducer 里做网络请求；异步放 thunk。

#### `src/app/intentClassifier.ts`

- **目录**：`src/app`

- **行数**：60

- **字节**：2375

- **导出符号**：ChartPromptIntent, classifyChartIntent

- **主要功能简介**：文件路径表明其所属分层（app 状态、views 页面、components 可复用、data 类型、i18n）。该文件约 60 行，承担界面或客户端协议的一部分。与后端的耦合点通常是 `getUrls()` 与 `apiRequest`/`streamRequest`。

- **修改注意**：用户可见字符串走 i18n；API 错误走 handleApiError；不要在 reducer 里做网络请求；异步放 thunk。

#### `src/app/layout.ts`

- **目录**：`src/app`

- **行数**：542

- **字节**：23453

- **导出符号**：Density, WidthClass, HeightClass, MIN_SUPPORTED, WIDTH_BREAKPOINTS, HEIGHT_BREAKPOINTS, DENSITY_SCALE, LayoutTokens, REFERENCE, resolveWidthClass, resolveHeightClass, densityForWidthClass, threadColumnsForWidthClass, maxThreadColumnsForWidthClass, defaultThreadColumns, layoutFor, stripPaddingLeft, SCROLLBAR_ALLOWANCE, COLUMN_FIT_TOLERANCE, threadPaneWidthFor, threadStripWidthFor, fittableThreadColumnsFor, textVar, iconVar, buttonVar, gridSizeCaps, CHART_STRETCH_STEPS, chartStretchCeiling, CHART_SIZE_STOPS, DEFAULT_CHART_SIZE_STOP_INDEX, chartSizeStopIndex, defaultChartSizeStop, DIALOG_VIEWPORT_MARGIN, dialogHeight, dialogWidth, ShellBudget, minimumShellBudget, sidebarFitsExpanded, maxThreadColumnsForWidth, COMFORTABLE_CANVAS, comfortableThreadColumns, clampDensityForViewport

- **主要功能简介**：文件路径表明其所属分层（app 状态、views 页面、components 可复用、data 类型、i18n）。该文件约 542 行，承担界面或客户端协议的一部分。与后端的耦合点通常是 `getUrls()` 与 `apiRequest`/`streamRequest`。

- **修改注意**：用户可见字符串走 i18n；API 错误走 handleApiError；不要在 reducer 里做网络请求；异步放 thunk。

#### `src/app/loadableState.ts`

- **目录**：`src/app`

- **行数**：48

- **字节**：1306

- **导出符号**：LoadableStatus, LoadableState, idleLoadable, loadingLoadable, successLoadable, errorLoadable, getLoadableErrorMessage

- **主要功能简介**：文件路径表明其所属分层（app 状态、views 页面、components 可复用、data 类型、i18n）。该文件约 48 行，承担界面或客户端协议的一部分。与后端的耦合点通常是 `getUrls()` 与 `apiRequest`/`streamRequest`。

- **修改注意**：用户可见字符串走 i18n；API 错误走 handleApiError；不要在 reducer 里做网络请求；异步放 thunk。

#### `src/app/oidcConfig.ts`

- **目录**：`src/app`

- **行数**：227

- **字节**：7921

- **导出符号**：OidcEndpointMetadata, OidcConfig, AuthInfo, isBackendAuth, getAuthInfo, getOidcConfig, getUserManager, getAccessToken, getOidcUser, _resetForTesting, _setUserManagerForTesting

- **主要功能简介**：文件路径表明其所属分层（app 状态、views 页面、components 可复用、data 类型、i18n）。该文件约 227 行，承担界面或客户端协议的一部分。与后端的耦合点通常是 `getUrls()` 与 `apiRequest`/`streamRequest`。

- **修改注意**：用户可见字符串走 i18n；API 错误走 handleApiError；不要在 reducer 里做网络请求；异步放 thunk。

#### `src/app/restyle.ts`

- **目录**：`src/app`

- **行数**：327

- **字节**：13249

- **导出符号**：buildEmbeddedDataForChart, buildSpecForRestyle, buildDataContext, RestyleResult, callRestyleAgent, makeVariant, sanitizeConfigUI, applyVariantConfigUI

- **主要功能简介**：文件路径表明其所属分层（app 状态、views 页面、components 可复用、data 类型、i18n）。该文件约 327 行，承担界面或客户端协议的一部分。与后端的耦合点通常是 `getUrls()` 与 `apiRequest`/`streamRequest`。

- **修改注意**：用户可见字符串走 i18n；API 错误走 handleApiError；不要在 reducer 里做网络请求；异步放 thunk。

#### `src/app/stateMigrations.ts`

- **目录**：`src/app`

- **行数**：357

- **字节**：16379

- **导出符号**：DF_STATE_VERSION, migrateState

- **主要功能简介**：文件路径表明其所属分层（app 状态、views 页面、components 可复用、data 类型、i18n）。该文件约 357 行，承担界面或客户端协议的一部分。与后端的耦合点通常是 `getUrls()` 与 `apiRequest`/`streamRequest`。

- **修改注意**：用户可见字符串走 i18n；API 错误走 handleApiError；不要在 reducer 里做网络请求；异步放 thunk。

#### `src/app/store.ts`

- **目录**：`src/app`

- **行数**：54

- **字节**：2069

- **导出符号**：AppDispatch, persistor, store

- **主要功能简介**：文件路径表明其所属分层（app 状态、views 页面、components 可复用、data 类型、i18n）。该文件约 54 行，承担界面或客户端协议的一部分。与后端的耦合点通常是 `getUrls()` 与 `apiRequest`/`streamRequest`。

- **修改注意**：用户可见字符串走 i18n；API 错误走 handleApiError；不要在 reducer 里做网络请求；异步放 thunk。

#### `src/app/tableResolution.ts`

- **目录**：`src/app`

- **行数**：61

- **字节**：1926

- **导出符号**：workspaceTableIdOf, AnalystTableRef, toAnalystTableRef, materializeInputTablePreview, materializeTables

- **主要功能简介**：文件路径表明其所属分层（app 状态、views 页面、components 可复用、data 类型、i18n）。该文件约 61 行，承担界面或客户端协议的一部分。与后端的耦合点通常是 `getUrls()` 与 `apiRequest`/`streamRequest`。

- **修改注意**：用户可见字符串走 i18n；API 错误走 handleApiError；不要在 reducer 里做网络请求；异步放 thunk。

#### `src/app/tableThunks.ts`

- **目录**：`src/app`

- **行数**：390

- **字节**：16966

- **导出符号**：LoadTablePayload, LoadTableResult, resolveDatabaseImportLimit, loadTable, buildDictTableFromWorkspace, hasLocalOnlyAncestor

- **主要功能简介**：文件路径表明其所属分层（app 状态、views 页面、components 可复用、data 类型、i18n）。该文件约 390 行，承担界面或客户端协议的一部分。与后端的耦合点通常是 `getUrls()` 与 `apiRequest`/`streamRequest`。

- **修改注意**：用户可见字符串走 i18n；API 错误走 handleApiError；不要在 reducer 里做网络请求；异步放 thunk。

#### `src/app/tokens.ts`

- **目录**：`src/app`

- **行数**：269

- **字节**：12348

- **导出符号**：borderColor, sidebarEdge, DividerBorderStyle, ComponentBorderStyle, ViewBorderStyle, shadow, transition, floatingPillSx, conversationWidth, radius, AppPaletteEntry, AppPalette, palettes, defaultPaletteKey, paletteKeys, bgAlpha

- **主要功能简介**：文件路径表明其所属分层（app 状态、views 页面、components 可复用、data 类型、i18n）。该文件约 269 行，承担界面或客户端协议的一部分。与后端的耦合点通常是 `getUrls()` 与 `apiRequest`/`streamRequest`。

- **修改注意**：用户可见字符串走 i18n；API 错误走 handleApiError；不要在 reducer 里做网络请求；异步放 thunk。

#### `src/app/useAutoSave.tsx`

- **目录**：`src/app`

- **行数**：121

- **字节**：4745

- **导出符号**：getSerializableState, useAutoSave

- **主要功能简介**：文件路径表明其所属分层（app 状态、views 页面、components 可复用、data 类型、i18n）。该文件约 121 行，承担界面或客户端协议的一部分。与后端的耦合点通常是 `getUrls()` 与 `apiRequest`/`streamRequest`。

- **修改注意**：用户可见字符串走 i18n；API 错误走 handleApiError；不要在 reducer 里做网络请求；异步放 thunk。

#### `src/app/useDataRefresh.tsx`

- **目录**：`src/app`

- **行数**：653

- **字节**：29147

- **导出符号**：useDataRefresh, useDerivedTableRefresh

- **主要功能简介**：文件路径表明其所属分层（app 状态、views 页面、components 可复用、data 类型、i18n）。该文件约 653 行，承担界面或客户端协议的一部分。与后端的耦合点通常是 `getUrls()` 与 `apiRequest`/`streamRequest`。

- **修改注意**：用户可见字符串走 i18n；API 错误走 handleApiError；不要在 reducer 里做网络请求；异步放 thunk。

#### `src/app/useKnowledgeStore.ts`

- **目录**：`src/app`

- **行数**：201

- **字节**：6323

- **导出符号**：KnowledgeCategoryState, useKnowledgeStore

- **主要功能简介**：文件路径表明其所属分层（app 状态、views 页面、components 可复用、data 类型、i18n）。该文件约 201 行，承担界面或客户端协议的一部分。与后端的耦合点通常是 `getUrls()` 与 `apiRequest`/`streamRequest`。

- **修改注意**：用户可见字符串走 i18n；API 错误走 handleApiError；不要在 reducer 里做网络请求；异步放 thunk。

#### `src/app/useWorkspaceAutoName.tsx`

- **目录**：`src/app`

- **行数**：98

- **字节**：4241

- **导出符号**：isUntitledWorkspaceName, useWorkspaceAutoName

- **主要功能简介**：文件路径表明其所属分层（app 状态、views 页面、components 可复用、data 类型、i18n）。该文件约 98 行，承担界面或客户端协议的一部分。与后端的耦合点通常是 `getUrls()` 与 `apiRequest`/`streamRequest`。

- **修改注意**：用户可见字符串走 i18n；API 错误走 handleApiError；不要在 reducer 里做网络请求；异步放 thunk。

#### `src/app/utils.tsx`

- **目录**：`src/app`

- **行数**：678

- **字节**：24458

- **导出符号**：getUrls, SourceTableRef, CONNECTOR_ACTION_URLS, CONNECTOR_URLS, fetchWithIdentity, getAgentLanguage, translateBackend, translateBackendOptions, usePrevious, computeContentHash, runCodeOnInputListsInVM, extractFieldsFromEncodingMap, prepVisTable, assembleVegaChart, hashCode, resolveRecommendedChart, resolveChartFields

- **主要功能简介**：文件路径表明其所属分层（app 状态、views 页面、components 可复用、data 类型、i18n）。该文件约 678 行，承担界面或客户端协议的一部分。与后端的耦合点通常是 `getUrls()` 与 `apiRequest`/`streamRequest`。

- **修改注意**：用户可见字符串走 i18n；API 错误走 handleApiError；不要在 reducer 里做网络请求；异步放 thunk。

#### `src/app/workspaceDB.ts`

- **目录**：`src/app`

- **行数**：129

- **字节**：4134

- **导出符号**：TableIndexEntry, WorkspaceEntry, workspaceDB

- **主要功能简介**：文件路径表明其所属分层（app 状态、views 页面、components 可复用、data 类型、i18n）。该文件约 129 行，承担界面或客户端协议的一部分。与后端的耦合点通常是 `getUrls()` 与 `apiRequest`/`streamRequest`。

- **修改注意**：用户可见字符串走 i18n；API 错误走 handleApiError；不要在 reducer 里做网络请求；异步放 thunk。

#### `src/app/workspaceService.ts`

- **目录**：`src/app`

- **行数**：298

- **字节**：12124

- **导出符号**：WorkspaceSummary, onWorkspaceListChanged, WorkspaceLoadSupersededError, listWorkspaces, loadWorkspace, deleteWorkspace, updateWorkspaceMeta, saveWorkspaceState, exportWorkspace, importWorkspace, deleteTableFromWorkspace, deleteTablesFromWorkspace, isWorkspaceReadOnly

- **主要功能简介**：文件路径表明其所属分层（app 状态、views 页面、components 可复用、data 类型、i18n）。该文件约 298 行，承担界面或客户端协议的一部分。与后端的耦合点通常是 `getUrls()` 与 `apiRequest`/`streamRequest`。

- **修改注意**：用户可见字符串走 i18n；API 错误走 handleApiError；不要在 reducer 里做网络请求；异步放 thunk。

#### `src/components/AnvilLoader.tsx`

- **目录**：`src/components`

- **行数**：116

- **字节**：4107

- **导出符号**：AnvilLoaderProps, AnvilLoader

- **主要功能简介**：文件路径表明其所属分层（app 状态、views 页面、components 可复用、data 类型、i18n）。该文件约 116 行，承担界面或客户端协议的一部分。与后端的耦合点通常是 `getUrls()` 与 `apiRequest`/`streamRequest`。

- **修改注意**：用户可见字符串走 i18n；API 错误走 handleApiError；不要在 reducer 里做网络请求；异步放 thunk。

#### `src/components/CatalogTree.tsx`

- **目录**：`src/components`

- **行数**：245

- **字节**：9850

- **导出符号**：CatalogTreeNode, collectNamespaceIds, mergeChildrenAtPath, appendChildrenAtPath, findNodeByPath, StyledTreeItem, CountBadge, RenderCatalogTreeOptions, renderCatalogTreeItems

- **主要功能简介**：文件路径表明其所属分层（app 状态、views 页面、components 可复用、data 类型、i18n）。该文件约 245 行，承担界面或客户端协议的一部分。与后端的耦合点通常是 `getUrls()` 与 `apiRequest`/`streamRequest`。

- **修改注意**：用户可见字符串走 i18n；API 错误走 handleApiError；不要在 reducer 里做网络请求；异步放 thunk。

#### `src/components/ChartTemplates.tsx`

- **目录**：`src/components`

- **行数**：157

- **字节**：7955

- **导出符号**：CHART_ICONS, CHART_TEMPLATES, getChartTemplate, getChartChannels, channels, channelGroups

- **主要功能简介**：文件路径表明其所属分层（app 状态、views 页面、components 可复用、data 类型、i18n）。该文件约 157 行，承担界面或客户端协议的一部分。与后端的耦合点通常是 `getUrls()` 与 `apiRequest`/`streamRequest`。

- **修改注意**：用户可见字符串走 i18n；API 错误走 handleApiError；不要在 reducer 里做网络请求；异步放 thunk。

#### `src/components/ComponentType.tsx`

- **目录**：`src/components`

- **行数**：687

- **字节**：25926

- **导出符号**：FieldSource, FieldItem, duplicateField, ROOTLESS_THREAD_ID, Trigger, Actor, ClarificationOption, ClarificationQuestion, ClarificationResponse, DelegateTarget, InteractionEntry, DeriveStatus, LoadedTableNode, PendingClarification, DraftNode, ThreadNode, TextTurn, DataCleanTableOutput, DataCleanBlock, ChatAttachment, InlineTablePreview, CodeExecution, PendingTableLoad, LoadPlanCandidate, LoadPlan, ConnectorFormPrompt, ConnectorFormArtifact, FormArtifact, ChatMessage, DataSourceType, DataSourceConfig, InputTableSource, InputTableColumn, FieldSemanticsInfo, TableSemanticsInfo, InputTableSnapshot, InputTablePreview, InputTable, DictTable, TableNode, createDictTable, ChartStyleVariant, VariantConfigControl, Chart, computeInsightKey, computeEncodingFingerprint, isVariantStale, EncodingMap, EncodingItem, ChartTemplate, AGGR_OP_LIST, AggrOp, Channel, EncodingDropResult, ConnectorAuthPath, ConnectorInstance

- **主要功能简介**：文件路径表明其所属分层（app 状态、views 页面、components 可复用、data 类型、i18n）。该文件约 687 行，承担界面或客户端协议的一部分。与后端的耦合点通常是 `getUrls()` 与 `apiRequest`/`streamRequest`。

- **修改注意**：用户可见字符串走 i18n；API 错误走 handleApiError；不要在 reducer 里做网络请求；异步放 thunk。

#### `src/components/ConnectorFormCard.tsx`

- **目录**：`src/components`

- **行数**：393

- **字节**：18853

- **导出符号**：ConnectorFormCard

- **主要功能简介**：文件路径表明其所属分层（app 状态、views 页面、components 可复用、data 类型、i18n）。该文件约 393 行，承担界面或客户端协议的一部分。与后端的耦合点通常是 `getUrls()` 与 `apiRequest`/`streamRequest`。

- **修改注意**：用户可见字符串走 i18n；API 错误走 handleApiError；不要在 reducer 里做网络请求；异步放 thunk。

#### `src/components/ConnectorTablePreview.tsx`

- **目录**：`src/components`

- **行数**：738

- **字节**：37277

- **导出符号**：ColumnMeta, PreviewFilter, SourceFilter, ConnectorTablePreviewProps, inferInputType, defaultOperatorForType, coerceFilters, ConnectorTablePreview

- **主要功能简介**：文件路径表明其所属分层（app 状态、views 页面、components 可复用、data 类型、i18n）。该文件约 738 行，承担界面或客户端协议的一部分。与后端的耦合点通常是 `getUrls()` 与 `apiRequest`/`streamRequest`。

- **修改注意**：用户可见字符串走 i18n；API 错误走 handleApiError；不要在 reducer 里做网络请求；异步放 thunk。

#### `src/components/DataOperationCard.tsx`

- **目录**：`src/components`

- **行数**：99

- **字节**：4247

- **导出符号**：DataOperationCard

- **主要功能简介**：文件路径表明其所属分层（app 状态、views 页面、components 可复用、data 类型、i18n）。该文件约 99 行，承担界面或客户端协议的一部分。与后端的耦合点通常是 `getUrls()` 与 `apiRequest`/`streamRequest`。

- **修改注意**：用户可见字符串走 i18n；API 错误走 handleApiError；不要在 reducer 里做网络请求；异步放 thunk。

#### `src/components/DndTypes.ts`

- **目录**：`src/components`

- **行数**：17

- **字节**：471

- **导出符号**：CATALOG_TABLE_ITEM, CatalogTableDragItem

- **主要功能简介**：文件路径表明其所属分层（app 状态、views 页面、components 可复用、data 类型、i18n）。该文件约 17 行，承担界面或客户端协议的一部分。与后端的耦合点通常是 `getUrls()` 与 `apiRequest`/`streamRequest`。

- **修改注意**：用户可见字符串走 i18n；API 错误走 handleApiError；不要在 reducer 里做网络请求；异步放 thunk。

#### `src/components/FunComponents.tsx`

- **目录**：`src/components`

- **行数**：78

- **字节**：3063

- **导出符号**：WritingPencil, ShimmerText, WritingIndicator, ThinkingBufferEffect

- **主要功能简介**：文件路径表明其所属分层（app 状态、views 页面、components 可复用、data 类型、i18n）。该文件约 78 行，承担界面或客户端协议的一部分。与后端的耦合点通常是 `getUrls()` 与 `apiRequest`/`streamRequest`。

- **修改注意**：用户可见字符串走 i18n；API 错误走 handleApiError；不要在 reducer 里做网络请求；异步放 thunk。

#### `src/components/LoadPlanCard.tsx`

- **目录**：`src/components`

- **行数**：495

- **字节**：23909

- **导出符号**：PresentedLoadCandidate, buildLoadQueryImportOptions, LoadPlanCard

- **主要功能简介**：文件路径表明其所属分层（app 状态、views 页面、components 可复用、data 类型、i18n）。该文件约 495 行，承担界面或客户端协议的一部分。与后端的耦合点通常是 `getUrls()` 与 `apiRequest`/`streamRequest`。

- **修改注意**：用户可见字符串走 i18n；API 错误走 handleApiError；不要在 reducer 里做网络请求；异步放 thunk。

#### `src/components/MarkdownEditor.tsx`

- **目录**：`src/components`

- **行数**：100

- **字节**：3829

- **导出符号**：MarkdownEditor

- **主要功能简介**：文件路径表明其所属分层（app 状态、views 页面、components 可复用、data 类型、i18n）。该文件约 100 行，承担界面或客户端协议的一部分。与后端的耦合点通常是 `getUrls()` 与 `apiRequest`/`streamRequest`。

- **修改注意**：用户可见字符串走 i18n；API 错误走 handleApiError；不要在 reducer 里做网络请求；异步放 thunk。

#### `src/components/ResizeHandle.tsx`

- **目录**：`src/components`

- **行数**：115

- **字节**：4123

- **导出符号**：ResizeHandleProps, ResizeHandle

- **主要功能简介**：文件路径表明其所属分层（app 状态、views 页面、components 可复用、data 类型、i18n）。该文件约 115 行，承担界面或客户端协议的一部分。与后端的耦合点通常是 `getUrls()` 与 `apiRequest`/`streamRequest`。

- **修改注意**：用户可见字符串走 i18n；API 错误走 handleApiError；不要在 reducer 里做网络请求；异步放 thunk。

#### `src/components/RotatingTextBlock.tsx`

- **目录**：`src/components`

- **行数**：49

- **字节**：1438

- **导出符号**：RotatingTextBlock

- **主要功能简介**：文件路径表明其所属分层（app 状态、views 页面、components 可复用、data 类型、i18n）。该文件约 49 行，承担界面或客户端协议的一部分。与后端的耦合点通常是 `getUrls()` 与 `apiRequest`/`streamRequest`。

- **修改注意**：用户可见字符串走 i18n；API 错误走 handleApiError；不要在 reducer 里做网络请求；异步放 thunk。

#### `src/components/ScrollFade.tsx`

- **目录**：`src/components`

- **行数**：114

- **字节**：4137

- **导出符号**：SCROLL_FADE, useScrollFade, ScrollFadeEdge, ScrollFadeContainer

- **主要功能简介**：文件路径表明其所属分层（app 状态、views 页面、components 可复用、data 类型、i18n）。该文件约 114 行，承担界面或客户端协议的一部分。与后端的耦合点通常是 `getUrls()` 与 `apiRequest`/`streamRequest`。

- **修改注意**：用户可见字符串走 i18n；API 错误走 handleApiError；不要在 reducer 里做网络请求；异步放 thunk。

#### `src/components/TablePreviewRow.tsx`

- **目录**：`src/components`

- **行数**：152

- **字节**：7655

- **导出符号**：TablePreviewData, TablePreviewRowProps, TablePreviewRow

- **主要功能简介**：文件路径表明其所属分层（app 状态、views 页面、components 可复用、data 类型、i18n）。该文件约 152 行，承担界面或客户端协议的一部分。与后端的耦合点通常是 `getUrls()` 与 `apiRequest`/`streamRequest`。

- **修改注意**：用户可见字符串走 i18n；API 错误走 handleApiError；不要在 reducer 里做网络请求；异步放 thunk。

#### `src/components/VirtualizedCatalogTree.tsx`

- **目录**：`src/components`

- **行数**：531

- **字节**：24910

- **导出符号**：VirtualizedCatalogTreeProps, VirtualizedCatalogTree

- **主要功能简介**：文件路径表明其所属分层（app 状态、views 页面、components 可复用、data 类型、i18n）。该文件约 531 行，承担界面或客户端协议的一部分。与后端的耦合点通常是 `getUrls()` 与 `apiRequest`/`streamRequest`。

- **修改注意**：用户可见字符串走 i18n；API 错误走 handleApiError；不要在 reducer 里做网络请求；异步放 thunk。

#### `src/components/filterFormat.ts`

- **目录**：`src/components`

- **行数**：69

- **字节**：2634

- **导出符号**：FILTER_OPERATOR_SYMBOLS, formatFilterOperator, formatFilterChipLabel

- **主要功能简介**：文件路径表明其所属分层（app 状态、views 页面、components 可复用、data 类型、i18n）。该文件约 69 行，承担界面或客户端协议的一部分。与后端的耦合点通常是 `getUrls()` 与 `apiRequest`/`streamRequest`。

- **修改注意**：用户可见字符串走 i18n；API 错误走 handleApiError；不要在 reducer 里做网络请求；异步放 thunk。

#### `src/data/column.ts`

- **目录**：`src/data`

- **行数**：37

- **字节**：755

- **导出符号**：Column

- **主要功能简介**：文件路径表明其所属分层（app 状态、views 页面、components 可复用、data 类型、i18n）。该文件约 37 行，承担界面或客户端协议的一部分。与后端的耦合点通常是 `getUrls()` 与 `apiRequest`/`streamRequest`。

- **修改注意**：用户可见字符串走 i18n；API 错误走 handleApiError；不要在 reducer 里做网络请求；异步放 thunk。

#### `src/data/table.ts`

- **目录**：`src/data`

- **行数**：73

- **字节**：1862

- **导出符号**：ColumnTable

- **主要功能简介**：文件路径表明其所属分层（app 状态、views 页面、components 可复用、data 类型、i18n）。该文件约 73 行，承担界面或客户端协议的一部分。与后端的耦合点通常是 `getUrls()` 与 `apiRequest`/`streamRequest`。

- **修改注意**：用户可见字符串走 i18n；API 错误走 handleApiError；不要在 reducer 里做网络请求；异步放 thunk。

#### `src/data/types.ts`

- **目录**：`src/data`

- **行数**：156

- **字节**：6005

- **导出符号**：Type, TypeList, CoerceType, TestType, isBoolean, isNumber, isDate, getDType, testType, mapApiTypeToAppType, isTemporalType

- **主要功能简介**：文件路径表明其所属分层（app 状态、views 页面、components 可复用、data 类型、i18n）。该文件约 156 行，承担界面或客户端协议的一部分。与后端的耦合点通常是 `getUrls()` 与 `apiRequest`/`streamRequest`。

- **修改注意**：用户可见字符串走 i18n；API 错误走 handleApiError；不要在 reducer 里做网络请求；异步放 thunk。

#### `src/data/utils.ts`

- **目录**：`src/data`

- **行数**：328

- **字节**：10897

- **导出符号**：readFileText, loadTextDataWrapper, createTableFromText, createTableFromFromObjectArray, inferTypeFromValueArray, refineTemporalType, convertTypeToDtype, coerceValueArrayFromTypes, coerceValueFromTypes, computeUniqueValues, tupleEqual, resolveExcelCellValue, loadBinaryDataWrapper, exportTableToDsv

- **主要功能简介**：文件路径表明其所属分层（app 状态、views 页面、components 可复用、data 类型、i18n）。该文件约 328 行，承担界面或客户端协议的一部分。与后端的耦合点通常是 `getUrls()` 与 `apiRequest`/`streamRequest`。

- **修改注意**：用户可见字符串走 i18n；API 错误走 handleApiError；不要在 reducer 里做网络请求；异步放 thunk。

#### `src/dataOperations/models.ts`

- **目录**：`src/dataOperations`

- **行数**：230

- **字节**：8309

- **导出符号**：DATA_OPERATION_SCHEMA_VERSION, JsonValue, DataOperationStatus, OperationFilter, LoadQuery, ConnectorQueryStepSummary, DataOperationStepSummary, DataOperationPlan, OperationError, FailedOperationStep, DataOperation, parseDataOperation

- **主要功能简介**：文件路径表明其所属分层（app 状态、views 页面、components 可复用、data 类型、i18n）。该文件约 230 行，承担界面或客户端协议的一部分。与后端的耦合点通常是 `getUrls()` 与 `apiRequest`/`streamRequest`。

- **修改注意**：用户可见字符串走 i18n；API 错误走 handleApiError；不要在 reducer 里做网络请求；异步放 thunk。

#### `src/i18n/index.ts`

- **目录**：`src/i18n`

- **行数**：33

- **字节**：849

- **导出符号**：无（入口或样式副作用）

- **主要功能简介**：文件路径表明其所属分层（app 状态、views 页面、components 可复用、data 类型、i18n）。该文件约 33 行，承担界面或客户端协议的一部分。与后端的耦合点通常是 `getUrls()` 与 `apiRequest`/`streamRequest`。

- **修改注意**：用户可见字符串走 i18n；API 错误走 handleApiError；不要在 reducer 里做网络请求；异步放 thunk。

#### `src/i18n/locales/en/index.ts`

- **目录**：`src/i18n/locales/en`

- **行数**：27

- **字节**：620

- **导出符号**：无（入口或样式副作用）

- **主要功能简介**：文件路径表明其所属分层（app 状态、views 页面、components 可复用、data 类型、i18n）。该文件约 27 行，承担界面或客户端协议的一部分。与后端的耦合点通常是 `getUrls()` 与 `apiRequest`/`streamRequest`。

- **修改注意**：用户可见字符串走 i18n；API 错误走 handleApiError；不要在 reducer 里做网络请求；异步放 thunk。

#### `src/i18n/locales/index.ts`

- **目录**：`src/i18n/locales`

- **行数**：8

- **字节**：142

- **导出符号**：en, zh

- **主要功能简介**：文件路径表明其所属分层（app 状态、views 页面、components 可复用、data 类型、i18n）。该文件约 8 行，承担界面或客户端协议的一部分。与后端的耦合点通常是 `getUrls()` 与 `apiRequest`/`streamRequest`。

- **修改注意**：用户可见字符串走 i18n；API 错误走 handleApiError；不要在 reducer 里做网络请求；异步放 thunk。

#### `src/i18n/locales/zh/index.ts`

- **目录**：`src/i18n/locales/zh`

- **行数**：27

- **字节**：620

- **导出符号**：无（入口或样式副作用）

- **主要功能简介**：文件路径表明其所属分层（app 状态、views 页面、components 可复用、data 类型、i18n）。该文件约 27 行，承担界面或客户端协议的一部分。与后端的耦合点通常是 `getUrls()` 与 `apiRequest`/`streamRequest`。

- **修改注意**：用户可见字符串走 i18n；API 错误走 handleApiError；不要在 reducer 里做网络请求；异步放 thunk。

#### `src/i18n/vega-locale.ts`

- **目录**：`src/i18n`

- **行数**：50

- **字节**：1865

- **导出符号**：syncVegaLocale

- **主要功能简介**：文件路径表明其所属分层（app 状态、views 页面、components 可复用、data 类型、i18n）。该文件约 50 行，承担界面或客户端协议的一部分。与后端的耦合点通常是 `getUrls()` 与 `apiRequest`/`streamRequest`。

- **修改注意**：用户可见字符串走 i18n；API 错误走 handleApiError；不要在 reducer 里做网络请求；异步放 thunk。

#### `src/icons.tsx`

- **目录**：`src`

- **行数**：286

- **字节**：16083

- **导出符号**：connectorSortOrder, getConnectorIcon, DatabaseViewIcon, default, default, default, default, default, default, GenericDBIcon, RelationalDBIcon, BooleanIcon, NumericalIcon, StringIcon, DateIcon, DateTimeIcon, TimeIcon, DurationIcon, UnknownIcon

- **主要功能简介**：文件路径表明其所属分层（app 状态、views 页面、components 可复用、data 类型、i18n）。该文件约 286 行，承担界面或客户端协议的一部分。与后端的耦合点通常是 `getUrls()` 与 `apiRequest`/`streamRequest`。

- **修改注意**：用户可见字符串走 i18n；API 错误走 handleApiError；不要在 reducer 里做网络请求；异步放 thunk。

#### `src/index.css`

- **目录**：`src`

- **行数**：91

- **字节**：2006

- **导出符号**：无（入口或样式副作用）

- **主要功能简介**：文件路径表明其所属分层（app 状态、views 页面、components 可复用、data 类型、i18n）。该文件约 91 行，承担界面或客户端协议的一部分。与后端的耦合点通常是 `getUrls()` 与 `apiRequest`/`streamRequest`。

- **修改注意**：用户可见字符串走 i18n；API 错误走 handleApiError；不要在 reducer 里做网络请求；异步放 thunk。

#### `src/index.tsx`

- **目录**：`src`

- **行数**：29

- **字节**：700

- **导出符号**：无（入口或样式副作用）

- **主要功能简介**：文件路径表明其所属分层（app 状态、views 页面、components 可复用、data 类型、i18n）。该文件约 29 行，承担界面或客户端协议的一部分。与后端的耦合点通常是 `getUrls()` 与 `apiRequest`/`streamRequest`。

- **修改注意**：用户可见字符串走 i18n；API 错误走 handleApiError；不要在 reducer 里做网络请求；异步放 thunk。

#### `src/mui.d.ts`

- **目录**：`src`

- **行数**：8

- **字节**：166

- **导出符号**：无（入口或样式副作用）

- **主要功能简介**：文件路径表明其所属分层（app 状态、views 页面、components 可复用、data 类型、i18n）。该文件约 8 行，承担界面或客户端协议的一部分。与后端的耦合点通常是 `getUrls()` 与 `apiRequest`/`streamRequest`。

- **修改注意**：用户可见字符串走 i18n；API 错误走 handleApiError；不要在 reducer 里做网络请求；异步放 thunk。

#### `src/types.d.ts`

- **目录**：`src`

- **行数**：30

- **字节**：565

- **导出符号**：无（入口或样式副作用）

- **主要功能简介**：文件路径表明其所属分层（app 状态、views 页面、components 可复用、data 类型、i18n）。该文件约 30 行，承担界面或客户端协议的一部分。与后端的耦合点通常是 `getUrls()` 与 `apiRequest`/`streamRequest`。

- **修改注意**：用户可见字符串走 i18n；API 错误走 handleApiError；不要在 reducer 里做网络请求；异步放 thunk。

#### `src/views/About.tsx`

- **目录**：`src/views`

- **行数**：254

- **字节**：11764

- **导出符号**：About

- **主要功能简介**：文件路径表明其所属分层（app 状态、views 页面、components 可复用、data 类型、i18n）。该文件约 254 行，承担界面或客户端协议的一部分。与后端的耦合点通常是 `getUrls()` 与 `apiRequest`/`streamRequest`。

- **修改注意**：用户可见字符串走 i18n；API 错误走 handleApiError；不要在 reducer 里做网络请求；异步放 thunk。

#### `src/views/AgentChatInput.tsx`

- **目录**：`src/views`

- **行数**：601

- **字节**：25874

- **导出符号**：AgentChatInputProps, AgentChatInput

- **主要功能简介**：文件路径表明其所属分层（app 状态、views 页面、components 可复用、data 类型、i18n）。该文件约 601 行，承担界面或客户端协议的一部分。与后端的耦合点通常是 `getUrls()` 与 `apiRequest`/`streamRequest`。

- **修改注意**：用户可见字符串走 i18n；API 错误走 handleApiError；不要在 reducer 里做网络请求；异步放 thunk。

#### `src/views/AgentPausePanel.tsx`

- **目录**：`src/views`

- **行数**：673

- **字节**：33762

- **导出符号**：ClarificationPanel, ExplanationPanel

- **主要功能简介**：文件路径表明其所属分层（app 状态、views 页面、components 可复用、data 类型、i18n）。该文件约 673 行，承担界面或客户端协议的一部分。与后端的耦合点通常是 `getUrls()` 与 `apiRequest`/`streamRequest`。

- **修改注意**：用户可见字符串走 i18n；API 错误走 handleApiError；不要在 reducer 里做网络请求；异步放 thunk。

#### `src/views/AgentRulesDialog.tsx`

- **目录**：`src/views`

- **行数**：393

- **字节**：16341

- **导出符号**：AgentRulesDialog

- **主要功能简介**：文件路径表明其所属分层（app 状态、views 页面、components 可复用、data 类型、i18n）。该文件约 393 行，承担界面或客户端协议的一部分。与后端的耦合点通常是 `getUrls()` 与 `apiRequest`/`streamRequest`。

- **修改注意**：用户可见字符串走 i18n；API 错误走 handleApiError；不要在 reducer 里做网络请求；异步放 thunk。

#### `src/views/AgentToyIcon.tsx`

- **目录**：`src/views`

- **行数**：138

- **字节**：5986

- **导出符号**：AgentToyVariant, AgentToyIcon, AnimatedAgentToyIcon

- **主要功能简介**：文件路径表明其所属分层（app 状态、views 页面、components 可复用、data 类型、i18n）。该文件约 138 行，承担界面或客户端协议的一部分。与后端的耦合点通常是 `getUrls()` 与 `apiRequest`/`streamRequest`。

- **修改注意**：用户可见字符串走 i18n；API 错误走 handleApiError；不要在 reducer 里做网络请求；异步放 thunk。

#### `src/views/ChartQuickConfig.tsx`

- **目录**：`src/views`

- **行数**：417

- **字节**：21104

- **导出符号**：ChartQuickConfigProps, ChartQuickConfig

- **主要功能简介**：文件路径表明其所属分层（app 状态、views 页面、components 可复用、data 类型、i18n）。该文件约 417 行，承担界面或客户端协议的一部分。与后端的耦合点通常是 `getUrls()` 与 `apiRequest`/`streamRequest`。

- **修改注意**：用户可见字符串走 i18n；API 错误走 handleApiError；不要在 reducer 里做网络请求；异步放 thunk。

#### `src/views/ChartRenderService.tsx`

- **目录**：`src/views`

- **行数**：360

- **字节**：15260

- **导出符号**：ChartRenderService

- **主要功能简介**：文件路径表明其所属分层（app 状态、views 页面、components 可复用、data 类型、i18n）。该文件约 360 行，承担界面或客户端协议的一部分。与后端的耦合点通常是 `getUrls()` 与 `apiRequest`/`streamRequest`。

- **修改注意**：用户可见字符串走 i18n；API 错误走 handleApiError；不要在 reducer 里做网络请求；异步放 thunk。

#### `src/views/ChartUtils.tsx`

- **目录**：`src/views`

- **行数**：65

- **字节**：3271

- **导出符号**：无（入口或样式副作用）

- **主要功能简介**：文件路径表明其所属分层（app 状态、views 页面、components 可复用、data 类型、i18n）。该文件约 65 行，承担界面或客户端协议的一部分。与后端的耦合点通常是 `getUrls()` 与 `apiRequest`/`streamRequest`。

- **修改注意**：用户可见字符串走 i18n；API 错误走 handleApiError；不要在 reducer 里做网络请求；异步放 thunk。

#### `src/views/ChartVariantStrip.tsx`

- **目录**：`src/views`

- **行数**：636

- **字节**：28828

- **导出符号**：ChartVariantStripProps, ChartVariantStrip

- **主要功能简介**：文件路径表明其所属分层（app 状态、views 页面、components 可复用、data 类型、i18n）。该文件约 636 行，承担界面或客户端协议的一部分。与后端的耦合点通常是 `getUrls()` 与 `apiRequest`/`streamRequest`。

- **修改注意**：用户可见字符串走 i18n；API 错误走 handleApiError；不要在 reducer 里做网络请求；异步放 thunk。

#### `src/views/ChartifactDialog.tsx`

- **目录**：`src/views`

- **行数**：308

- **字节**：9782

- **导出符号**：convertToChartifact, openChartifactViewer

- **主要功能简介**：文件路径表明其所属分层（app 状态、views 页面、components 可复用、data 类型、i18n）。该文件约 308 行，承担界面或客户端协议的一部分。与后端的耦合点通常是 `getUrls()` 与 `apiRequest`/`streamRequest`。

- **修改注意**：用户可见字符串走 i18n；API 错误走 handleApiError；不要在 reducer 里做网络请求；异步放 thunk。

#### `src/views/ChatDialog.tsx`

- **目录**：`src/views`

- **行数**：450

- **字节**：17694

- **导出符号**：GroupHeader, GroupItems, ChatDialogProps, ChatDialog

- **主要功能简介**：文件路径表明其所属分层（app 状态、views 页面、components 可复用、data 类型、i18n）。该文件约 450 行，承担界面或客户端协议的一部分。与后端的耦合点通常是 `getUrls()` 与 `apiRequest`/`streamRequest`。

- **修改注意**：用户可见字符串走 i18n；API 错误走 handleApiError；不要在 reducer 里做网络请求；异步放 thunk。

#### `src/views/ColumnFilterPopover.tsx`

- **目录**：`src/views`

- **行数**：632

- **字节**：24359

- **导出符号**：RangeFilter, InFilter, ContainsFilter, ColumnFilter, ColumnFilterPopover

- **主要功能简介**：文件路径表明其所属分层（app 状态、views 页面、components 可复用、data 类型、i18n）。该文件约 632 行，承担界面或客户端协议的一部分。与后端的耦合点通常是 `getUrls()` 与 `apiRequest`/`streamRequest`。

- **修改注意**：用户可见字符串走 i18n；API 错误走 handleApiError；不要在 reducer 里做网络请求；异步放 thunk。

#### `src/views/DBTableManager.tsx`

- **目录**：`src/views`

- **行数**：1248

- **字节**：69513

- **导出符号**：DataLoaderForm

- **主要功能简介**：文件路径表明其所属分层（app 状态、views 页面、components 可复用、data 类型、i18n）。该文件约 1248 行，承担界面或客户端协议的一部分。与后端的耦合点通常是 `getUrls()` 与 `apiRequest`/`streamRequest`。

- **修改注意**：用户可见字符串走 i18n；API 错误走 handleApiError；不要在 reducer 里做网络请求；异步放 thunk。

#### `src/views/DataFormulator.tsx`

- **目录**：`src/views`

- **行数**：1200

- **字节**：60233

- **导出符号**：DataFormulatorFC

- **主要功能简介**：文件路径表明其所属分层（app 状态、views 页面、components 可复用、data 类型、i18n）。该文件约 1200 行，承担界面或客户端协议的一部分。与后端的耦合点通常是 `getUrls()` 与 `apiRequest`/`streamRequest`。

- **修改注意**：用户可见字符串走 i18n；API 错误走 handleApiError；不要在 reducer 里做网络请求；异步放 thunk。

#### `src/views/DataFrameTable.tsx`

- **目录**：`src/views`

- **行数**：273

- **字节**：12886

- **导出符号**：DataFrameTableProps, DataFrameTable

- **主要功能简介**：文件路径表明其所属分层（app 状态、views 页面、components 可复用、data 类型、i18n）。该文件约 273 行，承担界面或客户端协议的一部分。与后端的耦合点通常是 `getUrls()` 与 `apiRequest`/`streamRequest`。

- **修改注意**：用户可见字符串走 i18n；API 错误走 handleApiError；不要在 reducer 里做网络请求；异步放 thunk。

#### `src/views/DataLoadingChat.tsx`

- **目录**：`src/views`

- **行数**：2013

- **字节**：103591

- **导出符号**：DataLoadingChat

- **主要功能简介**：文件路径表明其所属分层（app 状态、views 页面、components 可复用、data 类型、i18n）。该文件约 2013 行，承担界面或客户端协议的一部分。与后端的耦合点通常是 `getUrls()` 与 `apiRequest`/`streamRequest`。

- **修改注意**：用户可见字符串走 i18n；API 错误走 handleApiError；不要在 reducer 里做网络请求；异步放 thunk。

#### `src/views/DataSourceSidebar.tsx`

- **目录**：`src/views`

- **行数**：2544

- **字节**：134422

- **导出符号**：DataSourceSidebar

- **主要功能简介**：文件路径表明其所属分层（app 状态、views 页面、components 可复用、data 类型、i18n）。该文件约 2544 行，承担界面或客户端协议的一部分。与后端的耦合点通常是 `getUrls()` 与 `apiRequest`/`streamRequest`。

- **修改注意**：用户可见字符串走 i18n；API 错误走 handleApiError；不要在 reducer 里做网络请求；异步放 thunk。

#### `src/views/DataThread.tsx`

- **目录**：`src/views`

- **行数**：3405

- **字节**：169184

- **导出符号**：ThinkingStepsBanner, ThinkingBanner, DataThread

- **主要功能简介**：文件路径表明其所属分层（app 状态、views 页面、components 可复用、data 类型、i18n）。该文件约 3405 行，承担界面或客户端协议的一部分。与后端的耦合点通常是 `getUrls()` 与 `apiRequest`/`streamRequest`。

- **修改注意**：用户可见字符串走 i18n；API 错误走 handleApiError；不要在 reducer 里做网络请求；异步放 thunk。

#### `src/views/DataThreadCards.tsx`

- **目录**：`src/views`

- **行数**：384

- **字节**：16651

- **导出符号**：BuildTableCardProps

- **主要功能简介**：文件路径表明其所属分层（app 状态、views 页面、components 可复用、data 类型、i18n）。该文件约 384 行，承担界面或客户端协议的一部分。与后端的耦合点通常是 `getUrls()` 与 `apiRequest`/`streamRequest`。

- **修改注意**：用户可见字符串走 i18n；API 错误走 handleApiError；不要在 reducer 里做网络请求；异步放 thunk。

#### `src/views/DataView.tsx`

- **目录**：`src/views`

- **行数**：373

- **字节**：18344

- **导出符号**：FreeDataViewProps, FreeDataViewFC

- **主要功能简介**：文件路径表明其所属分层（app 状态、views 页面、components 可复用、data 类型、i18n）。该文件约 373 行，承担界面或客户端协议的一部分。与后端的耦合点通常是 `getUrls()` 与 `apiRequest`/`streamRequest`。

- **修改注意**：用户可见字符串走 i18n；API 错误走 handleApiError；不要在 reducer 里做网络请求；异步放 thunk。

#### `src/views/EncodingBox.tsx`

- **目录**：`src/views`

- **行数**：766

- **字节**：32002

- **导出符号**：LittleConceptCardProps, LittleConceptCard, EncodingBoxProps, EncodingBox

- **主要功能简介**：文件路径表明其所属分层（app 状态、views 页面、components 可复用、data 类型、i18n）。该文件约 766 行，承担界面或客户端协议的一部分。与后端的耦合点通常是 `getUrls()` 与 `apiRequest`/`streamRequest`。

- **修改注意**：用户可见字符串走 i18n；API 错误走 handleApiError；不要在 reducer 里做网络请求；异步放 thunk。

#### `src/views/EncodingShelfCard.tsx`

- **目录**：`src/views`

- **行数**：1179

- **字节**：54336

- **导出符号**：ConfigSlider, EncodingShelfCardProps, renderTextWithEmphasis, TriggerCard, StylePreset, STYLE_PRESETS, EncodingShelfCard

- **主要功能简介**：文件路径表明其所属分层（app 状态、views 页面、components 可复用、data 类型、i18n）。该文件约 1179 行，承担界面或客户端协议的一部分。与后端的耦合点通常是 `getUrls()` 与 `apiRequest`/`streamRequest`。

- **修改注意**：用户可见字符串走 i18n；API 错误走 handleApiError；不要在 reducer 里做网络请求；异步放 thunk。

#### `src/views/EncodingShelfThread.tsx`

- **目录**：`src/views`

- **行数**：103

- **字节**：3247

- **导出符号**：EncodingShelfThreadProps, EncodingShelfThread

- **主要功能简介**：文件路径表明其所属分层（app 状态、views 页面、components 可复用、data 类型、i18n）。该文件约 103 行，承担界面或客户端协议的一部分。与后端的耦合点通常是 `getUrls()` 与 `apiRequest`/`streamRequest`。

- **修改注意**：用户可见字符串走 i18n；API 错误走 handleApiError；不要在 reducer 里做网络请求；异步放 thunk。

#### `src/views/ExampleSessions.tsx`

- **目录**：`src/views`

- **行数**：195

- **字节**：6711

- **导出符号**：ExampleSession, fetchExampleSessions, exampleSessions, ExampleSessionCard

- **主要功能简介**：文件路径表明其所属分层（app 状态、views 页面、components 可复用、data 类型、i18n）。该文件约 195 行，承担界面或客户端协议的一部分。与后端的耦合点通常是 `getUrls()` 与 `apiRequest`/`streamRequest`。

- **修改注意**：用户可见字符串走 i18n；API 错误走 handleApiError；不要在 reducer 里做网络请求；异步放 thunk。

#### `src/views/ExplComponents.tsx`

- **目录**：`src/views`

- **行数**：394

- **字节**：13976

- **导出符号**：ConceptExplanationItem, ConceptExplCardsProps, ConceptExplCards, extractConceptExplanations, CodeExplanationCard

- **主要功能简介**：文件路径表明其所属分层（app 状态、views 页面、components 可复用、data 类型、i18n）。该文件约 394 行，承担界面或客户端协议的一部分。与后端的耦合点通常是 `getUrls()` 与 `apiRequest`/`streamRequest`。

- **修改注意**：用户可见字符串走 i18n；API 错误走 handleApiError；不要在 reducer 里做网络请求；异步放 thunk。

#### `src/views/InteractionEntryCard.tsx`

- **目录**：`src/views`

- **行数**：756

- **字节**：36920

- **导出符号**：getStepIconComponent, PlanStepsView, CompactMarkdown, renderFieldHighlights, stripFieldMarkers, InteractionEntryCardProps, InteractionEntryCard, ResolvedConversationCardProps, ResolvedConversationCard, getEntryGutterIcon, getDefaultGutterIcon

- **主要功能简介**：文件路径表明其所属分层（app 状态、views 页面、components 可复用、data 类型、i18n）。该文件约 756 行，承担界面或客户端协议的一部分。与后端的耦合点通常是 `getUrls()` 与 `apiRequest`/`streamRequest`。

- **修改注意**：用户可见字符串走 i18n；API 错误走 handleApiError；不要在 reducer 里做网络请求；异步放 thunk。

#### `src/views/KnowledgePanel.tsx`

- **目录**：`src/views`

- **行数**：644

- **字节**：30015

- **导出符号**：KnowledgePanel

- **主要功能简介**：文件路径表明其所属分层（app 状态、views 页面、components 可复用、data 类型、i18n）。该文件约 644 行，承担界面或客户端协议的一部分。与后端的耦合点通常是 `getUrls()` 与 `apiRequest`/`streamRequest`。

- **修改注意**：用户可见字符串走 i18n；API 错误走 handleApiError；不要在 reducer 里做网络请求；异步放 thunk。

#### `src/views/LocalInstallUpgradePanel.tsx`

- **目录**：`src/views`

- **行数**：299

- **字节**：11420

- **导出符号**：LocalInstallUpgradePanel

- **主要功能简介**：文件路径表明其所属分层（app 状态、views 页面、components 可复用、data 类型、i18n）。该文件约 299 行，承担界面或客户端协议的一部分。与后端的耦合点通常是 `getUrls()` 与 `apiRequest`/`streamRequest`。

- **修改注意**：用户可见字符串走 i18n；API 错误走 handleApiError；不要在 reducer 里做网络请求；异步放 thunk。

#### `src/views/LogViewerDialog.tsx`

- **目录**：`src/views`

- **行数**：325

- **字节**：13480

- **导出符号**：LogViewerDialog

- **主要功能简介**：文件路径表明其所属分层（app 状态、views 页面、components 可复用、data 类型、i18n）。该文件约 325 行，承担界面或客户端协议的一部分。与后端的耦合点通常是 `getUrls()` 与 `apiRequest`/`streamRequest`。

- **修改注意**：用户可见字符串走 i18n；API 错误走 handleApiError；不要在 reducer 里做网络请求；异步放 thunk。

#### `src/views/MessageSnackbar.tsx`

- **目录**：`src/views`

- **行数**：353

- **字节**：17230

- **导出符号**：Message, MessageSnackbar

- **主要功能简介**：文件路径表明其所属分层（app 状态、views 页面、components 可复用、data 类型、i18n）。该文件约 353 行，承担界面或客户端协议的一部分。与后端的耦合点通常是 `getUrls()` 与 `apiRequest`/`streamRequest`。

- **修改注意**：用户可见字符串走 i18n；API 错误走 handleApiError；不要在 reducer 里做网络请求；异步放 thunk。

#### `src/views/ModelSelectionDialog.tsx`

- **目录**：`src/views`

- **行数**：821

- **字节**：36026

- **导出符号**：ModelSelectionButton

- **主要功能简介**：文件路径表明其所属分层（app 状态、views 页面、components 可复用、data 类型、i18n）。该文件约 821 行，承担界面或客户端协议的一部分。与后端的耦合点通常是 `getUrls()` 与 `apiRequest`/`streamRequest`。

- **修改注意**：用户可见字符串走 i18n；API 错误走 handleApiError；不要在 reducer 里做网络请求；异步放 thunk。

#### `src/views/MultiTablePreview.tsx`

- **目录**：`src/views`

- **行数**：234

- **字节**：9837

- **导出符号**：MultiTablePreviewProps, MultiTablePreview

- **主要功能简介**：文件路径表明其所属分层（app 状态、views 页面、components 可复用、data 类型、i18n）。该文件约 234 行，承担界面或客户端协议的一部分。与后端的耦合点通常是 `getUrls()` 与 `apiRequest`/`streamRequest`。

- **修改注意**：用户可见字符串走 i18n；API 错误走 handleApiError；不要在 reducer 里做网络请求；异步放 thunk。

#### `src/views/OperatorCard.tsx`

- **目录**：`src/views`

- **行数**：65

- **字节**：1982

- **导出符号**：OperatorCardProp, OperatorCard

- **主要功能简介**：文件路径表明其所属分层（app 状态、views 页面、components 可复用、data 类型、i18n）。该文件约 65 行，承担界面或客户端协议的一部分。与后端的耦合点通常是 `getUrls()` 与 `apiRequest`/`streamRequest`。

- **修改注意**：用户可见字符串走 i18n；API 错误走 handleApiError；不要在 reducer 里做网络请求；异步放 thunk。

#### `src/views/ReactTable.tsx`

- **目录**：`src/views`

- **行数**：212

- **字节**：9301

- **导出符号**：ColumnDef, CustomReactTable

- **主要功能简介**：文件路径表明其所属分层（app 状态、views 页面、components 可复用、data 类型、i18n）。该文件约 212 行，承担界面或客户端协议的一部分。与后端的耦合点通常是 `getUrls()` 与 `apiRequest`/`streamRequest`。

- **修改注意**：用户可见字符串走 i18n；API 错误走 handleApiError；不要在 reducer 里做网络请求；异步放 thunk。

#### `src/views/RefreshDataDialog.tsx`

- **目录**：`src/views`

- **行数**：556

- **字节**：22566

- **导出符号**：RefreshDataDialogProps, RefreshDataDialog

- **主要功能简介**：文件路径表明其所属分层（app 状态、views 页面、components 可复用、data 类型、i18n）。该文件约 556 行，承担界面或客户端协议的一部分。与后端的耦合点通常是 `getUrls()` 与 `apiRequest`/`streamRequest`。

- **修改注意**：用户可见字符串走 i18n；API 错误走 handleApiError；不要在 reducer 里做网络请求；异步放 thunk。

#### `src/views/ReportView.tsx`

- **目录**：`src/views`

- **行数**：780

- **字节**：33511

- **导出符号**：ReportView

- **主要功能简介**：文件路径表明其所属分层（app 状态、views 页面、components 可复用、data 类型、i18n）。该文件约 780 行，承担界面或客户端协议的一部分。与后端的耦合点通常是 `getUrls()` 与 `apiRequest`/`streamRequest`。

- **修改注意**：用户可见字符串走 i18n；API 错误走 handleApiError；不要在 reducer 里做网络请求；异步放 thunk。

#### `src/views/SelectableDataGrid.tsx`

- **目录**：`src/views`

- **行数**：849

- **字节**：38577

- **导出符号**：ColumnDef, SelectableDataGrid

- **主要功能简介**：文件路径表明其所属分层（app 状态、views 页面、components 可复用、data 类型、i18n）。该文件约 849 行，承担界面或客户端协议的一部分。与后端的耦合点通常是 `getUrls()` 与 `apiRequest`/`streamRequest`。

- **修改注意**：用户可见字符串走 i18n；API 错误走 handleApiError；不要在 reducer 里做网络请求；异步放 thunk。

#### `src/views/SessionDistill.tsx`

- **目录**：`src/views`

- **行数**：734

- **字节**：31216

- **导出符号**：SessionThread, BuildSessionResult, findSessionWorkflow, collectSessionThreads, buildSessionWorkflowContext, SessionDistillDialogProps, SessionDistillDialog

- **主要功能简介**：文件路径表明其所属分层（app 状态、views 页面、components 可复用、data 类型、i18n）。该文件约 734 行，承担界面或客户端协议的一部分。与后端的耦合点通常是 `getUrls()` 与 `apiRequest`/`streamRequest`。

- **修改注意**：用户可见字符串走 i18n；API 错误走 handleApiError；不要在 reducer 里做网络请求；异步放 thunk。

#### `src/views/SimpleChartRecBox.tsx`

- **目录**：`src/views`

- **行数**：2629

- **字节**：142105

- **导出符号**：SimpleChartRecBox

- **主要功能简介**：文件路径表明其所属分层（app 状态、views 页面、components 可复用、data 类型、i18n）。该文件约 2629 行，承担界面或客户端协议的一部分。与后端的耦合点通常是 `getUrls()` 与 `apiRequest`/`streamRequest`。

- **修改注意**：用户可见字符串走 i18n；API 错误走 handleApiError；不要在 reducer 里做网络请求；异步放 thunk。

#### `src/views/SourceTableShelf.tsx`

- **目录**：`src/views`

- **行数**：982

- **字节**：44715

- **导出符号**：SHELF_VISIBLE_LIMIT, SourceTableShelf

- **主要功能简介**：文件路径表明其所属分层（app 状态、views 页面、components 可复用、data 类型、i18n）。该文件约 982 行，承担界面或客户端协议的一部分。与后端的耦合点通常是 `getUrls()` 与 `apiRequest`/`streamRequest`。

- **修改注意**：用户可见字符串走 i18n；API 错误走 handleApiError；不要在 reducer 里做网络请求；异步放 thunk。

#### `src/views/TestPanel.tsx`

- **目录**：`src/views`

- **行数**：89

- **字节**：2700

- **导出符号**：TestPanelProps, TestPanelState, TestPanel

- **主要功能简介**：文件路径表明其所属分层（app 状态、views 页面、components 可复用、data 类型、i18n）。该文件约 89 行，承担界面或客户端协议的一部分。与后端的耦合点通常是 `getUrls()` 与 `apiRequest`/`streamRequest`。

- **修改注意**：用户可见字符串走 i18n；API 错误走 handleApiError；不要在 reducer 里做网络请求；异步放 thunk。

#### `src/views/TiptapReportEditor.tsx`

- **目录**：`src/views`

- **行数**：791

- **字节**：34038

- **导出符号**：TiptapReportEditorProps, InspectStep, TiptapReportEditor

- **主要功能简介**：文件路径表明其所属分层（app 状态、views 页面、components 可复用、data 类型、i18n）。该文件约 791 行，承担界面或客户端协议的一部分。与后端的耦合点通常是 `getUrls()` 与 `apiRequest`/`streamRequest`。

- **修改注意**：用户可见字符串走 i18n；API 错误走 handleApiError；不要在 reducer 里做网络请求；异步放 thunk。

#### `src/views/UnifiedDataUploadDialog.tsx`

- **目录**：`src/views`

- **行数**：2639

- **字节**：128998

- **导出符号**：UploadTabType, LocalFolderPanel, DataLoadMenuProps, DataLoadMenu, UnifiedDataUploadDialogProps, UnifiedDataUploadDialog, type ConnectorInstance

- **主要功能简介**：文件路径表明其所属分层（app 状态、views 页面、components 可复用、data 类型、i18n）。该文件约 2639 行，承担界面或客户端协议的一部分。与后端的耦合点通常是 `getUrls()` 与 `apiRequest`/`streamRequest`。

- **修改注意**：用户可见字符串走 i18n；API 错误走 handleApiError；不要在 reducer 里做网络请求；异步放 thunk。

#### `src/views/ViewUtils.tsx`

- **目录**：`src/views`

- **行数**：187

- **字节**：6780

- **导出符号**：DENSE_MENU_SLOT_PROPS, groupConceptItems, getIconFromType, formatCellValue, getColumnAlign, getIconFromDtype

- **主要功能简介**：文件路径表明其所属分层（app 状态、views 页面、components 可复用、data 类型、i18n）。该文件约 187 行，承担界面或客户端协议的一部分。与后端的耦合点通常是 `getUrls()` 与 `apiRequest`/`streamRequest`。

- **修改注意**：用户可见字符串走 i18n；API 错误走 handleApiError；不要在 reducer 里做网络请求；异步放 thunk。

#### `src/views/VisualizationView.tsx`

- **目录**：`src/views`

- **行数**：1968

- **字节**：103928

- **导出符号**：VisPanelProps, VisPanelState, ChartEditorFC, VisualizationViewFC, generateChartSkeleton, getDataTable, checkChartAvailability

- **主要功能简介**：文件路径表明其所属分层（app 状态、views 页面、components 可复用、data 类型、i18n）。该文件约 1968 行，承担界面或客户端协议的一部分。与后端的耦合点通常是 `getUrls()` 与 `apiRequest`/`streamRequest`。

- **修改注意**：用户可见字符串走 i18n；API 错误走 handleApiError；不要在 reducer 里做网络请求；异步放 thunk。

#### `src/views/dataLoadingSuggestions.ts`

- **目录**：`src/views`

- **行数**：210

- **字节**：8725

- **导出符号**：DataLoadingSuggestion, SuggestionPayload, BuildSuggestionsArgs, buildDataLoadingSuggestions, DataLoadingQuickAction, buildDataLoadingQuickActions

- **主要功能简介**：文件路径表明其所属分层（app 状态、views 页面、components 可复用、data 类型、i18n）。该文件约 210 行，承担界面或客户端协议的一部分。与后端的耦合点通常是 `getUrls()` 与 `apiRequest`/`streamRequest`。

- **修改注意**：用户可见字符串走 i18n；API 错误走 handleApiError；不要在 reducer 里做网络请求；异步放 thunk。

#### `src/views/threadLayout.ts`

- **目录**：`src/views`

- **行数**：56

- **字节**：2274

- **导出符号**：CARD_WIDTH, CARD_GAP, PANEL_PADDING, MAX_THREAD_COLUMNS, COLUMN_FIT_TOLERANCE, threadPaneWidth, fittableThreadColumns

- **主要功能简介**：文件路径表明其所属分层（app 状态、views 页面、components 可复用、data 类型、i18n）。该文件约 56 行，承担界面或客户端协议的一部分。与后端的耦合点通常是 `getUrls()` 与 `apiRequest`/`streamRequest`。

- **修改注意**：用户可见字符串走 i18n；API 错误走 handleApiError；不要在 reducer 里做网络请求；异步放 thunk。

#### `src/views/workflowContext.ts`

- **目录**：`src/views`

- **行数**：319

- **字节**：12129

- **导出符号**：MESSAGE_CONTENT_LIMIT, TOOL_ARGS_LIMIT, SAMPLE_ROW_COUNT, TOOL_USES_CODE_FONT, isLeafDerivedTable, buildLeafEvents, buildDistillModelConfig

- **主要功能简介**：文件路径表明其所属分层（app 状态、views 页面、components 可复用、data 类型、i18n）。该文件约 319 行，承担界面或客户端协议的一部分。与后端的耦合点通常是 `getUrls()` 与 `apiRequest`/`streamRequest`。

- **修改注意**：用户可见字符串走 i18n；API 错误走 handleApiError；不要在 reducer 里做网络请求；异步放 thunk。


### 2.3 测试文件清单

#### `tests/backend/agents/test_agent_diagnostics.py`

- **行数**：295；**用例数**：24

- **测试类**：TestSharedFields, TestForError, TestForResponse, TestForJsonOnly, TestSchemaCompatibility, TestMultipleInstances

- **用例**：test_for_error_has_shared_keys, test_for_response_has_shared_keys, test_for_json_only_has_shared_keys, test_agent_name_propagated, test_timestamp_is_iso8601, test_model_info_preserved, test_prompt_components_complete, test_llm_request_message_count, test_contains_error_field, test_no_parsing_or_execution_sections, test_top_level_sections, test_llm_response_fields, test_parsing_fields, test_execution_fields, test_execution_error_fields_present_when_failed, test_performance_rounding, test_has_llm_response_and_performance, test_no_parsing_or_execution_sections, test_defaults_when_no_kwargs, test_for_response_key_set, test_for_error_key_set, test_for_json_only_key_set, test_different_agent_names, test_prompt_isolation

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/agents/test_agent_language.py`

- **行数**：235；**用例数**：38

- **测试类**：TestLanguageRegistry, TestBuildLanguageInstructionEnglish, TestBuildLanguageInstructionFull, TestBuildLanguageInstructionCompact, TestBuildLanguageInstructionUnknown, TestInjectLanguageInstruction

- **用例**：test_english_in_registry, test_common_languages_present, test_display_names_are_non_empty_strings, test_default_language_is_en, test_extra_rules_values_are_strings, test_extra_rules_codes_are_subset_of_display_names, test_english_returns_empty_string, test_english_compact_also_returns_empty, test_empty_string_defaults_to_en_returns_empty, test_none_coerced_to_default_returns_empty, test_whitespace_only_returns_empty, test_case_insensitive_en, test_non_english_returns_non_empty, test_result_contains_language_marker, test_result_contains_display_name, test_full_mode_is_default, test_full_mode_mentions_user_visible_fields, test_full_mode_mentions_internal_fields, test_zh_extra_rules_injected, test_ja_extra_rules_injected, test_lang_without_extra_rules_has_no_extra_block, test_compact_returns_non_empty_for_non_english, test_compact_contains_language_marker, test_compact_shorter_than_full, test_compact_mentions_display_instruction, test_compact_instructs_english_for_code, test_compact_zh_extra_rules_present, test_unknown_code_returns_non_empty, test_unknown_code_uses_raw_code_as_display_name, test_empty_instruction_is_noop, test_non_empty_instruction_appended_by_default, test_instruction_appended_after_base, test_marker_found_inserts_before_marker, test_marker_not_found_falls_back_to_append, test_marker_at_start_of_string_not_inserted, test_original_prompt_is_preserved_in_output, test_marker_insertion_preserves_rest_of_prompt, test_round_trip_with_build_and_inject

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/agents/test_agent_utils_sql_table_names.py`

- **行数**：49；**用例数**：4

- **测试类**：无

- **用例**：test_sql_sanitize_preserves_safe_ascii_symbols, test_sql_sanitize_replaces_spaces_and_hyphens_for_ascii_names, test_sql_sanitize_preserves_unicode_identifier, test_create_duckdb_views_supports_unicode_view_names

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/agents/test_analyst_connector_skill.py`

- **行数**：126；**用例数**：4

- **测试类**：无

- **用例**：test_list_and_describe_connectors, test_propose_connection_requires_listing_first, test_propose_connection_emits_prefilled_canvas_form_without_echo, test_local_folder_is_available_when_registered

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/agents/test_analyst_scratch_files.py`

- **行数**：59；**用例数**：3

- **测试类**：TestScratchFileInjection

- **用例**：test_scratch_note_injected, test_no_note_without_files, test_file_bytes_not_inlined

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/agents/test_client_image_strip.py`

- **行数**：238；**用例数**：17

- **测试类**：TestStripImageBlocks, TestStripImagesFromMessages, TestIsImageDeserializeError, TestGetCompletionRetryLitellm, TestGetCompletionRetryOpenAI

- **用例**：test_removes_image_url_items, test_keeps_all_when_no_images, test_returns_empty_list_when_all_images, test_passthrough_non_list, test_preserves_non_dict_items, test_strips_images_from_multimodal_message, test_does_not_mutate_original, test_handles_plain_text_messages, test_preserves_non_dict_messages, test_detects_known_patterns, test_ignores_unrelated_errors, test_case_insensitive, test_retries_on_image_error, test_raises_unrelated_error, test_no_retry_on_success, test_retries_on_image_error, test_raises_unrelated_error

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/agents/test_client_utils.py`

- **行数**：499；**用例数**：57

- **测试类**：TestModelNamePrefixing, TestOllamaApiBaseNormalisation, TestAzureCredentialSelection, TestStripImageBlocks, TestStripImagesFromMessages, TestIsImageDeserializeError, TestMessagesContainImages, TestFromConfig, TestExtractJsonObjects, TestMatchToolFromObj, TestSalvageToolCallsFromContent, TestMatchToolWireFormats

- **用例**：test_gemini_prefix_added_when_missing, test_gemini_prefix_not_doubled, test_anthropic_prefix_added_when_missing, test_anthropic_prefix_not_doubled, test_ollama_prefix_added_when_missing, test_ollama_prefix_not_doubled, test_openai_model_prefixed, test_trailing_slash_stripped, test_trailing_api_stripped, test_trailing_api_slash_stripped, test_non_api_suffix_preserved, test_default_base_when_none, test_desktop_keyless_model_uses_azure_cli_credential, test_blank_api_version_is_not_defaulted, test_explicit_api_version_is_preserved, test_api_base_is_required, test_string_content_unchanged, test_image_url_blocks_removed, test_non_image_blocks_preserved, test_mixed_list_with_non_dict_preserved, test_all_images_removed_returns_empty_list, test_system_message_unchanged, test_image_blocks_removed_from_user_message, test_text_blocks_preserved_in_user_message, test_original_messages_not_mutated, test_non_dict_messages_preserved, test_image_url_expected_text_detected, test_unknown_variant_image_url_detected, test_unrelated_error_not_detected, test_empty_string_not_detected, test_partial_match_image_url_without_expected, test_upstream_failure_with_images_detected, test_upstream_failure_without_images_not_detected, test_unsupported_image_message_with_images_detected, test_multimodal_message_detected, test_text_only_messages_not_detected, test_creates_client_from_dict, test_strips_whitespace_from_values, test_optional_fields_absent_when_empty, test_gemini_prefix_applied_via_from_config …

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/agents/test_context.py`

- **行数**：22；**用例数**：1

- **测试类**：无

- **用例**：test_focused_context_includes_text_turn_and_loading_decision

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/agents/test_core_chart_contract.py`

- **行数**：49；**用例数**：2

- **测试类**：无

- **用例**：test_visualize_schema_requires_title_and_exposes_subtitle, test_visualize_handler_forwards_title_and_subtitle

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/agents/test_data_loading_chat_images.py`

- **行数**：59；**用例数**：3

- **测试类**：无

- **用例**：test_convert_message_keeps_instruction_before_image_attachment, test_convert_message_ignores_empty_image_attachment, test_build_system_prompt_accepts_workspace_table_name_strings

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/agents/test_data_loading_discovery_tools.py`

- **行数**：774；**用例数**：54

- **测试类**：TestListData, TestFindData, TestDescribeData, TestProposeLoadPlan, TestNormalizeLoadQueryFilters, TestBuildSystemPromptConnectorSummary, TestDataMemoryTools, TestProbeData, TestConnectorTools

- **用例**：test_connection, test_bootstraps_uncached_zero_auth_connector, test_no_args_returns_sources_summary, test_no_user_home_returns_empty_sources, test_source_id_at_root, test_source_id_with_path_drills_into_folder, test_filter_narrows_tables, test_invalid_path_type_returns_error, test_bootstraps_uncached_zero_auth_connector, test_empty_query_returns_error, test_searches_catalog_with_regex, test_scope_with_source_id, test_scope_with_path_prefix, test_scope_workspace_skips_catalog, test_bad_regex_returns_error, test_no_match_returns_note_and_valid_sources, test_delegates_to_handle_read_catalog_metadata, test_missing_params_still_calls_with_empty_strings, test_preserves_all_option_groups, test_empty_options_returns_empty_action, test_resolves_superset_dataset_id_from_catalog, test_strips_wildcards_and_upgrades_eq_to_ilike, test_strips_wildcards_from_like, test_like_without_wildcards_upgraded_to_ilike, test_eq_without_wildcards_stays_eq, test_symbol_operators_mapped, test_contains_mapped_to_ilike, test_is_null_no_value, test_empty_wildcard_only_value_skipped, test_invalid_operator_falls_back_to_eq, test_non_list_returns_empty, test_missing_column_skipped, test_includes_connector_summary_when_sources_exist, test_shows_none_when_no_sources, test_graceful_when_user_home_missing, test_includes_current_date_and_time, test_tools_are_exposed, test_read_append_and_rewrite, test_read_defaults_to_one_hundred_lines_and_pages, test_read_rejects_invalid_regex …

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/agents/test_data_loading_skill.py`

- **行数**：405；**用例数**：11

- **测试类**：无

- **用例**：test_registry_exposes_discovery_tools_only_after_skill_load, test_proposal_persists_executable_plan_and_emits_display_only_pause, test_narration_is_the_response_shown_to_the_user, test_invalid_proposal_returns_recoverable_observation, test_proposal_does_not_require_plan_descriptions, test_minimal_proposal_resolves_table_fields_from_catalog, test_canonical_proposal_does_not_add_canvas_prose, test_proposal_rejects_exact_query_already_loaded_in_workspace, test_discovery_parameter_contract_matches_standalone_agent, test_skill_uses_shared_catalog_discovery, test_probe_budget_is_shared_within_run_and_isolated_between_runs

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/agents/test_duckdb_notes_prompt.py`

- **行数**：36；**用例数**：2

- **测试类**：无

- **用例**：test_duckdb_notes_mentions_non_ascii_double_quoting, test_duckdb_notes_mentions_identifier_quoting_rule

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/agents/test_generate_data_summary.py`

- **行数**：107；**用例数**：5

- **测试类**：TestInlineRowsFallback

- **用例**：test_workspace_table_uses_parquet, test_derived_table_falls_back_to_inline_rows, test_no_workspace_no_rows_shows_unavailable, test_inline_rows_shows_in_memory_path, test_inline_rows_sample_size_respected

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/agents/test_model_registry.py`

- **行数**：157；**用例数**：11

- **测试类**：TestModelDiscovery, TestPublicListingSecurity, TestCustomProvider

- **用例**：test_discovers_all_enabled_providers, test_total_model_count, test_empty_env_yields_no_models, test_skips_provider_without_models, test_skips_disabled_provider, test_no_api_key_in_public_info, test_public_fields_are_complete, test_full_config_contains_api_key, test_custom_provider_uses_explicit_endpoint, test_builtin_provider_uses_own_name_as_endpoint, test_custom_provider_defaults_to_openai_endpoint

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/agents/test_provenance_models.py`

- **行数**：105；**用例数**：7

- **测试类**：TestImportedFrom, TestDerivation

- **用例**：test_data_loader_roundtrip, test_upload_roundtrip, test_url_roundtrip, test_stream_roundtrip, test_paste_roundtrip, test_example_roundtrip, test_roundtrip

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/agents/test_reasoning_content_helpers.py`

- **行数**：103；**用例数**：11

- **测试类**：TestAttachReasoningContent, TestAccumulateReasoningContent

- **用例**：test_present, test_absent, test_none_value_not_attached, test_empty_string_attached, test_first_chunk, test_subsequent_chunks, test_no_attr, test_no_attr_preserves_accumulator, test_none_value_no_change, test_empty_string_delta_no_change, test_multiple_chunks_sequence

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/agents/test_reasoning_logger.py`

- **行数**：423；**用例数**：31

- **测试类**：TestLogFileCreation, TestOffMode, TestOnMode, TestVerboseMode, TestSecurityConstraints, TestExpiredLogCleanup, TestContextManager, TestNullReasoningLogger

- **用例**：test_creates_file_in_correct_directory, test_jsonl_format_each_line_parseable, test_each_line_has_step_type_and_ts, test_close_makes_file_complete, test_close_is_idempotent, test_rotates_to_new_date_directory, test_off_creates_no_file, test_off_case_insensitive, test_on_writes_structured_summary, test_on_strips_messages_defensively, test_on_auto_fields_cannot_be_overridden, test_default_is_off, test_verbose_writes_full_content, test_verbose_sanitizes_api_key, test_verbose_sanitizes_password_in_list, test_verbose_sanitizes_top_level_kwargs, test_verbose_case_insensitive, test_no_api_key_in_log, test_no_connection_string_in_log, test_confined_dir_prevents_traversal, test_old_directories_removed, test_cleanup_ignores_non_date_dirs, test_cleanup_failure_does_not_raise, test_cleanup_nonexistent_dir_does_not_raise, test_cleanup_runs_in_background, test_exception_still_closes_file, test_off_mode_context_manager_safe, test_log_is_noop, test_close_is_noop, test_context_manager, test_level_is_off

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/agents/test_semantic_types.py`

- **行数**：349；**用例数**：34

- **测试类**：TestAllSemanticTypes, TestIsMeasureType, TestIsTimeseriesType, TestIsCategoricalType, TestIsOrdinalType, TestIsGeoType, TestIsNonMeasureNumeric, TestIsSignedMeasure, TestGetVlType, TestInferVlTypeFromName, TestGenerateSemanticTypesPrompt

- **用例**：test_no_duplicates, test_includes_known_types, test_every_vl_map_key_is_in_all_types, test_vl_map_values_are_valid, test_all_types_covered_in_semantic_categories, test_measure_types_return_true, test_non_measure_types_return_false, test_unknown_string_returns_false, test_timeseries_types_return_true, test_non_timeseries_return_false, test_categorical_types_return_true, test_non_categorical_return_false, test_ordinal_types_return_true, test_non_ordinal_return_false, test_geo_types_return_true, test_non_geo_return_false, test_non_measure_numerics_return_true, test_others_return_false, test_signed_measures_return_true, test_non_signed_return_false, test_vl_type_mapping, test_unknown_type_returns_none, test_temporal_names, test_ordinal_names, test_quantitative_names, test_nominal_names, test_no_signal_names_return_none, test_temporal_takes_priority_over_ordinal, test_case_insensitive, test_returns_non_empty_string, test_contains_category_headers, test_contains_type_names, test_contains_guidelines, test_all_registered_types_appear_in_prompt

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/agents/test_sort_data_agent.py`

- **行数**：200；**用例数**：10

- **测试类**：TestInputConstruction, TestResponseParsing

- **用例**：test_input_key_is_values_not_value, test_input_name_is_preserved, test_input_values_are_preserved, test_unicode_values_serialised_correctly, test_valid_json_block_in_content_returns_ok, test_json_wrapped_in_text_is_extracted, test_unparseable_content_returns_error_status, test_multiple_choices_produce_multiple_candidates, test_agent_field_is_set, test_dialog_includes_system_and_user_and_assistant

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/agents/test_tool_path_safety.py`

- **行数**：196；**用例数**：13

- **测试类**：TestToolReadFile, TestToolListDirectory, TestToolWriteFile, TestPreviewScratchFiles

- **用例**：test_read_valid_file, test_traversal_blocked, test_absolute_path_blocked, test_nonexistent_file, test_empty_path_blocked, test_list_valid_directory, test_list_root_directory, test_traversal_blocked, test_nonexistent_directory, test_write_valid_file, test_traversal_sanitized_and_confined, test_traversal_blocked, test_valid_scratch_file

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/agents/test_workflow_distill.py`

- **行数**：571；**用例数**：26

- **测试类**：TestExtractContextSummary, TestRunWithMockedLLM, TestWorkflowFilename, TestDistillEndpoint

- **用例**：test_renders_each_event_type, test_empty_events_returns_marker, test_user_content_is_not_displaycontent, test_skips_non_dict_events, test_create_table_basic, test_create_chart_without_encoding, test_renders_multi_thread_with_headers, test_produces_valid_markdown, test_fallback_front_matter_added, test_retries_once_when_body_too_long, test_retry_asks_for_slack_under_limit, test_hard_trims_when_retry_still_over_limit, test_no_retry_when_body_within_limit, test_language_instruction_injected_into_system_prompt, test_language_code_zh_injects_chinese_instruction, test_language_code_en_no_extra_instruction, test_derives_from_title, test_fallback_when_title_blank, test_rejects_path_traversal, test_strips_reserved_and_control_chars, test_missing_context_returns_error, test_missing_model_returns_error, test_missing_events_returns_error, test_missing_events_field_returns_error, test_successful_distill, test_category_hint_creates_subdir

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/auth/sso_provider_contracts/__init__.py`

- **行数**：2；**用例数**：0

- **测试类**：无

- **用例**：

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/auth/sso_provider_contracts/provider_fixtures.py`

- **行数**：255；**用例数**：0

- **测试类**：无

- **用例**：

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/auth/sso_provider_contracts/test_oauth_provider_contracts.py`

- **行数**：179；**用例数**：5

- **测试类**：TestGitHubOAuthContract

- **用例**：test_login_redirect_matches_github_oauth_contract, test_callback_exchanges_code_and_stores_github_session, test_callback_fetches_primary_email_when_github_user_email_is_private, test_callback_rejects_invalid_github_oauth_state, test_callback_rejects_missing_github_access_token

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/auth/sso_provider_contracts/test_oidc_provider_contracts.py`

- **行数**：342；**用例数**：6

- **测试类**：TestMainstreamOIDCProviderContracts

- **用例**：test_discovery_metadata_resolves_provider_endpoints, test_jwks_jwt_validation_accepts_provider_claim_shape, test_jwks_jwt_validation_accepts_missing_optional_email_claim, test_userinfo_fallback_accepts_provider_userinfo_response, test_backend_gateway_uses_provider_authorization_endpoint, test_backend_gateway_exchanges_code_and_stores_userinfo

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/auth/test_auth.py`

- **行数**：139；**用例数**：16

- **测试类**：TestValidateIdentityValue, TestGetIdentityId

- **用例**：test_valid_uuid, test_valid_email, test_strips_whitespace, test_empty_raises, test_whitespace_only_raises, test_too_long_raises, test_path_separator_rejected, test_shell_metachar_rejected, test_control_chars_rejected, test_azure_principal_returns_user_prefix, test_browser_identity_returns_browser_prefix, test_client_cannot_spoof_user_prefix, test_azure_header_takes_priority_over_browser, test_missing_all_headers_raises, test_malformed_azure_header_rejected, test_browser_identity_strips_prefix

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/auth/test_auth_info_endpoint.py`

- **行数**：91；**用例数**：4

- **测试类**：TestAuthInfoEndpoint

- **用例**：test_anonymous_mode_returns_none_action, test_oidc_provider_returns_frontend_action, test_github_provider_returns_redirect_action, test_azure_provider_returns_transparent_action

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/auth/test_auth_provider_chain.py`

- **行数**：302；**用例数**：27

- **测试类**：TestProviderDiscovery, TestInitAuth, TestProviderDispatch, TestAnonymousMode, TestSSOToken, TestAuthenticationErrorPropagation, TestAllPhase1ProvidersDiscovered, TestOIDCViaInitAuth, TestIdentityRegexExpansion

- **用例**：test_azure_easyauth_is_discovered, test_get_provider_class_returns_class, test_unknown_provider_returns_none, test_no_env_var_stays_anonymous, test_explicit_anonymous_stays_anonymous, test_azure_easyauth_activates, test_unknown_provider_logs_error_stays_none, test_allow_anonymous_false, test_provider_authenticated_returns_user_prefix, test_provider_miss_falls_back_to_anonymous, test_provider_miss_no_anonymous_raises, test_browser_identity_works, test_prefixed_identity_stripped, test_missing_header_raises, test_spoofed_user_prefix_forced_to_browser, test_anonymous_mode_returns_none, test_provider_without_token_returns_none, test_authentication_error_becomes_value_error, test_oidc_discovered, test_github_discovered, test_azure_easyauth_discovered, test_oidc_activates_when_configured, test_oidc_disabled_without_env_vars, test_oidc_no_bearer_falls_back_to_anonymous, test_oidc_style_sub_claims_accepted, test_path_separator_still_rejected, test_space_still_rejected

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/auth/test_azure_cli.py`

- **行数**：16；**用例数**：1

- **测试类**：无

- **用例**：test_expose_azure_cli_adds_executable_directory_to_path

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/auth/test_azure_easyauth_provider.py`

- **行数**：94；**用例数**：9

- **测试类**：TestAzureEasyAuthProviderMetadata, TestAzureEasyAuthAuthenticate

- **用例**：test_name, test_enabled_always_true, test_get_auth_info_action, test_principal_header_present, test_principal_header_with_display_name, test_empty_display_name_becomes_none, test_no_header_returns_none, test_strips_whitespace_from_user_id, test_raw_token_is_none

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/auth/test_credential_vault.py`

- **行数**：157；**用例数**：18

- **测试类**：TestStoreAndRetrieve, TestUserIsolation, TestDelete, TestListSources, TestEncryptionKeyMismatch, TestEdgeCases

- **用例**：test_round_trip, test_retrieve_missing_returns_none, test_overwrite, test_different_users_isolated, test_cross_user_retrieve_returns_none, test_browser_identity_works, test_delete_removes_credential, test_delete_nonexistent_is_noop, test_delete_one_source_keeps_others, test_empty_initially, test_lists_stored_sources, test_list_after_delete, test_list_isolated_per_user, test_wrong_key_returns_none, test_invalid_key_raises, test_complex_credentials, test_unicode_credentials, test_empty_credentials_dict

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/auth/test_credential_vault_factory.py`

- **行数**：171；**用例数**：8

- **测试类**：TestExplicitKey, TestAutoGeneratedKey, TestFactoryReturnsNone, TestSingleton

- **用例**：test_env_key_takes_priority, test_default_type_is_local, test_auto_generates_key_when_no_env, test_reuses_existing_key_file, test_auto_generated_vault_is_functional, test_unknown_type_returns_none, test_multiple_calls_return_same_instance, test_singleton_is_cached

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/auth/test_flask_session_config.py`

- **行数**：50；**用例数**：5

- **测试类**：TestFlaskSecretKey, TestSessionConfig

- **用例**：test_uses_env_secret_key, test_falls_back_to_random_when_env_missing, test_permanent_session_lifetime_is_one_year, test_session_cookie_httponly, test_session_cookie_samesite

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/auth/test_github_oauth_provider.py`

- **行数**：131；**用例数**：11

- **测试类**：TestGitHubProviderMetadata, TestGitHubAuthenticate, TestGitHubGateway

- **用例**：test_name, test_enabled_with_both_vars, test_disabled_without_client_id, test_disabled_without_client_secret, test_get_auth_info_is_redirect, test_session_with_github_user, test_empty_session_returns_none, test_wrong_provider_in_session_returns_none, test_missing_provider_key_returns_none, test_callback_missing_code_redirects, test_logout_returns_json_ok

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/auth/test_kusto_oauth_gateway.py`

- **行数**：148；**用例数**：4

- **测试类**：无

- **用例**：test_login_uses_cluster_scope_and_pkce, test_login_rejects_untrusted_cluster_urls, test_callback_exchanges_code_and_posts_token_to_opener, test_callback_state_cannot_be_replayed

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/auth/test_oidc_gateway.py`

- **行数**：441；**用例数**：29

- **测试类**：TestOIDCLogin, TestOIDCCallback, TestOIDCStatus, TestOIDCLogout, TestAutoDetection, TestClearServiceToken, TestSaveDelegatedToken, TestAuthServiceStatus

- **用例**：test_login_redirects_to_authorize_url, test_login_sets_state_in_session, test_login_disabled_without_secret, test_login_fails_without_authorize_url, test_login_redirect_uri_uses_auth_callback, test_callback_rejects_missing_code, test_callback_rejects_invalid_state, test_callback_success_stores_tokens_and_redirects, test_callback_token_exchange_failure, test_callback_disabled_without_secret, test_callback_access_denied_returns_access_denied_error, test_callback_access_denied_clears_oauth_state, test_callback_idp_server_error_returns_token_exchange_failed, test_callback_access_denied_preserves_existing_sso_session, test_status_authenticated, test_status_not_authenticated, test_status_frontend_mode, test_logout_clears_session, test_logout_does_not_delete_vault_credentials, test_secret_present_implies_backend, test_no_secret_implies_frontend, test_auth_mode_overrides_auto_detection, test_auth_mode_backend_without_secret, test_clear_token_removes_from_session, test_clear_nonexistent_token_is_ok, test_save_token_success, test_save_token_missing_fields, test_save_token_missing_system_id, test_service_status_returns_dict

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/auth/test_oidc_provider.py`

- **行数**：276；**用例数**：18

- **测试类**：TestOIDCProviderMetadata, TestOIDCAuthenticate, TestOIDCAuthErrors, TestOIDCSkip

- **用例**：test_name, test_enabled_with_config, test_disabled_without_issuer, test_disabled_without_client_id, test_get_auth_info_action_is_frontend, test_default_scopes_include_offline_access, test_custom_scopes_override_defaults, test_valid_jwt_returns_auth_result, test_numeric_sub_is_rejected, test_expired_token_raises, test_wrong_issuer_raises, test_wrong_audience_raises, test_wrong_key_raises, test_missing_sub_claim_raises, test_no_authorization_header, test_non_bearer_authorization, test_empty_bearer_token, test_no_jwks_client_returns_none

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/auth/test_token_store.py`

- **行数**：393；**用例数**：34

- **测试类**：TestStoreServiceToken, TestStoreSSO, TestExpiry, TestGetAccess, TestRefresh, TestSSOExchange, TestGetAuthStatus, TestAvailableStrategies, TestResolveEnv

- **用例**：test_store_and_retrieve_via_session, test_store_overwrites_previous, test_clear_service_token, test_clear_sso_exchange_token_blocks_auto_reconnect, test_store_service_token_allows_auto_reconnect_again, test_clear_session_tokens_preserves_vault, test_store_sso_tokens, test_get_sso_token_backend_mode, test_get_sso_token_backend_expired_no_refresh, test_get_sso_token_frontend_mode, test_is_expired_true, test_is_expired_false, test_is_expired_missing_key, test_returns_none_when_no_auth_config, test_returns_cached_token, test_skips_expired_cache_tries_refresh, test_falls_through_to_vault, test_returns_none_when_all_fail, test_refresh_success, test_refresh_no_token_url, test_refresh_http_failure, test_sso_exchange_success, test_sso_exchange_no_sso_token, test_sso_exchange_no_exchange_url, test_sso_exchange_skips_when_blocked, test_returns_status_for_configured_systems, test_returns_unauthorized_for_missing_token, test_sso_exchange_available, test_delegated_popup, test_manual_credentials, test_oauth2_redirect, test_resolve_env, test_resolve_env_missing, test_resolve_env_empty_key

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/benchmarks/benchmark_sandbox.py`

- **行数**：232；**用例数**：0

- **测试类**：无

- **用例**：

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/benchmarks/benchmark_workspace.py`

- **行数**：530；**用例数**：0

- **测试类**：无

- **用例**：

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/data/test_all_loader_verification.py`

- **行数**：231；**用例数**：13

- **测试类**：TestAllLoaderCatalogHierarchies, TestScopePinningAllLoaders, TestAuthModes, TestStaticMethods, TestDataConnectorWrapping

- **用例**：test_all_loaders_have_hierarchy, test_hierarchies_match_expected, test_last_level_is_importable, test_pinning_removes_level, test_no_pinning_returns_full_hierarchy, test_default_auth_mode_is_connection, test_superset_uses_token_mode, test_all_loaders_have_list_params, test_all_loaders_have_auth_instructions, test_all_loaders_have_required_host_or_identifier, test_rate_limit_returns_dict_or_none, test_all_loaders_can_be_wrapped, test_all_loaders_blueprints_have_all_routes

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/data/test_atomic_metadata_update.py`

- **行数**：53；**用例数**：1

- **测试类**：无

- **用例**：test_concurrent_add_table_no_lost_updates

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/data/test_catalog_cache.py`

- **行数**：692；**用例数**：49

- **测试类**：TestSaveLoadCatalog, TestDeleteCatalog, TestListCachedSources, TestSearchCatalogCache, TestStructuredFieldSearch, TestListSourcesSummary, TestListPathChildren, TestSearchCatalogCacheExtended, TestConnectorConnectCatalogSave, TestCatalogCacheSyncedAt, TestSearchReturnsTableKey

- **用例**：test_save_creates_directory_and_file, test_load_returns_saved_tables, test_load_returns_none_for_missing, test_save_overwrites_existing, test_source_id_with_special_chars, test_legacy_synced_at_populates_both_freshness_clocks, test_listing_and_metadata_have_independent_freshness, test_listing_refresh_preserves_enriched_metadata, test_refresh_failure_preserves_last_good_tables, test_atomic_replace_failure_preserves_existing_catalog, test_delete_removes_file, test_delete_nonexistent_is_silent, test_delete_rejects_symlink_escape, test_returns_source_ids, test_returns_empty_for_missing_dir, test_returns_canonical_id_with_colon, test_falls_back_to_stem_when_source_id_missing, test_search_by_table_name, test_search_by_description, test_search_by_column_name, test_search_excludes_imported_tables, test_search_returns_empty_for_no_match, test_search_respects_limit_per_source, test_table_name_match_reports_table_name_reason, test_table_description_match, test_column_name_match, test_column_description_match, test_no_match_returns_empty, test_exclude_tables_drops_matches, test_search_catalog_cache_end_to_end, test_regex_query_alternation, test_flat_and_hierarchical, test_empty_when_no_cache, test_root_lists_folders_and_top_level_tables, test_drill_into_folder, test_filter_narrows_results, test_missing_source_returns_empty, test_truncation_includes_hint, test_regex_alternation_matches_two_tables, test_exclude_pattern_filters_out_matches …

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/data/test_catalog_refresh.py`

- **行数**：109；**用例数**：4

- **测试类**：无

- **用例**：test_disconnected_source_serves_stale_without_refresh, test_missing_connected_source_refreshes_synchronously, test_stale_refresh_is_deduplicated_and_serves_stale, test_recent_failure_suppresses_retry_and_preserves_tables

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/data/test_catalog_search_loaders.py`

- **行数**：95；**用例数**：3

- **测试类**：无

- **用例**：test_postgresql_search_catalog_returns_lightweight_tree, test_mysql_search_catalog_returns_lightweight_tree, test_superset_search_catalog_returns_dataset_and_dashboard_matches

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/data/test_catalog_sync_base.py`

- **行数**：137；**用例数**：15

- **测试类**：TestSyncCatalogMetadataDefault, TestEnsureTableKeys, TestSourceMetadataStatusConstants, TestCatalogErrorCodes

- **用例**：test_returns_list_tables_results, test_passes_table_filter, test_empty_tables, test_existing_table_key_preserved, test_fallback_to_source_name, test_fallback_to_name, test_fallback_to_name_no_metadata, test_empty_list, test_does_not_overwrite_existing_key, test_synced_value, test_not_synced_value, test_partial_value, test_unavailable_value, test_catalog_sync_timeout, test_catalog_not_found

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/data/test_connector_directory_storage.py`

- **行数**：285；**用例数**：11

- **测试类**：TestSafeSourceId, TestPersistAndRemove, TestLoadUserSpecs, TestLoadConnectorsIntegration

- **用例**：test_connection, test_sanitises_special_chars, test_persist_creates_directory_and_json, test_persist_overwrites_existing, test_remove_deletes_file, test_remove_nonexistent_is_silent, test_remove_rejects_symlink_escape, test_loads_from_json_directory, test_loads_multiple_connectors, test_returns_empty_when_no_connectors, test_loads_user_connectors_from_directory

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/data/test_connector_errors.py`

- **行数**：74；**用例数**：4

- **测试类**：无

- **用例**：test_classify_connector_error, test_http_401_maps_to_connector_auth_failed, test_http_500_with_lost_connection_maps_to_connection_failed, test_azure_sql_firewall_denial_exposes_safe_client_ip

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/data/test_data_connector_config.py`

- **行数**：618；**用例数**：26

- **测试类**：TestResolveEnvRefs, TestLoadConnectorsYaml, TestEnvVarParsing, TestLoadAdminSpecs, TestUserConnectorPersistence, TestLoadConnectors, TestRegisterConnectedSources

- **用例**：test_connection, test_resolves_env_var, test_missing_env_var_becomes_empty, test_non_env_ref_passed_through, test_load_valid_file, test_returns_empty_for_missing_file, test_returns_empty_for_bad_yaml, test_parse_env_sources_basic, test_parse_env_sources_multiple, test_parse_env_sources_missing_type_skipped, test_parse_env_sources_default_name, test_load_from_connectors_yaml, test_env_overrides_yaml, test_multiple_instances_same_type, test_env_ref_resolution_in_yaml_params, test_save_and_load_user_connectors, test_load_user_specs_returns_empty_if_no_file, test_loads_user_connectors_on_first_call, test_does_not_overwrite_admin_connectors, test_second_call_is_noop, test_user_connectors_are_scoped_by_identity, test_create_connector_persists_only_non_auth_params, test_registers_blueprints, test_skips_unknown_loader_type, test_logs_disabled_loaders, test_frontend_config_in_sources

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/data/test_data_connector_framework.py`

- **行数**：908；**用例数**：55

- **测试类**：TestSharedRouteRegistration, TestFrontendConfig, TestConnectorList, TestAuthRoutes, TestCatalogRoutes, TestDataRoutes, TestErrorHandling, TestIdentityIsolation, TestScopePinning, TestHelpers

- **用例**：test_connection, test_connection, test_connection, test_shared_routes_registered, test_frontend_config_structure, test_pinned_params_excluded_from_form, test_hierarchy_included, test_pinned_source_effective_hierarchy, test_pinned_params_do_not_expose_auth_or_sensitive_values, test_sso_auto_connect_respects_session_block, test_connect_success, test_connect_merges_default_params, test_connect_bad_host_returns_error, test_connect_fails_when_test_connection_fails, test_get_status_returns_structured_connection_error, test_delete_connector_clears_status, test_rename_connector_preserves_stable_id, test_disconnect_connector_clears_loader_and_credentials, test_status_connected, test_status_not_connected, test_status_not_connected_after_no_connect, test_safe_params_exclude_password, test_ls_root, test_ls_returns_hierarchy, test_ls_drill_down_to_tables, test_ls_with_filter, test_ls_not_connected_returns_error, test_catalog_metadata, test_catalog_tree, test_catalog_tree_with_filter, test_search_catalog_route, test_search_catalog_empty_query_returns_empty_tree, test_search_catalog_not_connected_returns_error, test_preview, test_preview_missing_source_table, test_import_requires_source_table, test_import_success, test_refresh_requires_table_name, test_column_values_success, test_column_values_missing_source_table …

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/data/test_data_connector_vault.py`

- **行数**：462；**用例数**：23

- **测试类**：TestVaultHelpers, TestConnectStoresCredentials, TestDeleteCredentials, TestAutoReconnect, TestNoVaultFallback, TestIdentityIsolation

- **用例**：test_connection, test_vault_store_and_retrieve, test_vault_retrieve_when_empty, test_vault_delete, test_vault_unavailable_returns_false, test_has_stored_credentials, test_vault_exception_is_caught, test_connect_does_not_auto_persist, test_persist_credentials_stores_in_vault, test_connect_via_route_stores_in_vault, test_connect_via_route_persist_false, test_connect_persist_false_clears_old_vault_entry, test_delete_credentials_clears_vault, test_require_loader_auto_reconnects, test_auto_reconnect_cleans_stale_creds, test_auto_reconnect_exception_cleans_stale_creds, test_status_reports_stored_credentials, test_auth_status_not_connected_no_vault, test_connect_without_vault, test_connect_route_without_vault_not_persisted, test_require_loader_no_vault_raises, test_different_users_separate_vault_entries, test_delete_only_affects_own_user

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/data/test_df_to_safe_records.py`

- **行数**：93；**用例数**：9

- **测试类**：TestDatetimeSerialization, TestMixedTypes, TestEdgeCases

- **用例**：test_datetime_column_returns_iso_string, test_datetime_with_time_component, test_nat_becomes_null, test_int_string_datetime_mixed, test_float_with_nan, test_empty_dataframe, test_empty_dataframe_no_columns, test_default_handler_catches_exotic_types, test_malformed_unicode_is_replaced_without_mutating_input

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/data/test_ephemeral_workspace.py`

- **行数**：136；**用例数**：4

- **测试类**：无

- **用例**：test_ephemeral_matches_local_workspace_lifecycle_without_retention, test_factory_uses_same_lazy_workspace_contract, test_cleanup_expires_only_ephemeral_workspaces, test_cleanup_lru_evicts_oldest_workspace

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/data/test_excel_fixture_parsing.py`

- **行数**：24；**用例数**：1

- **测试类**：无

- **用例**：test_manual_xls_fixture_can_be_parsed

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/data/test_external_data_loader_table_names.py`

- **行数**：37；**用例数**：6

- **测试类**：无

- **用例**：test_external_loader_sanitize_rejects_empty_name, test_external_loader_sanitize_prefixes_sql_keywords, test_external_loader_sanitize_truncates_overlong_names, test_external_loader_sanitize_preserves_pure_chinese_name, test_external_loader_sanitize_normalizes_mixed_unicode_name, test_external_loader_sanitize_applies_safe_prefix_without_losing_unicode

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/data/test_file_manager_encoding.py`

- **行数**：228；**用例数**：21

- **测试类**：无

- **用例**：test_utf8_passthrough, test_ascii_passthrough, test_empty_content, test_utf8_bom_stripped, test_utf8_bom_stripped_for_txt, test_gbk_converted_to_utf8, test_gb18030_bmp_converted_to_utf8, test_gbk_txt_converted_to_utf8, test_excel_type_not_converted, test_json_type_not_converted, test_parquet_type_not_converted, test_gbk_multicolumn_csv, test_gbk_medium_content, test_gbk_large_content, test_shift_jis_with_halfwidth_katakana, test_pure_kanji_shift_jis_decoded_as_gbk_is_known_tradeoff, test_euckr_decoded_as_gbk_is_known_tradeoff, test_euckr_with_large_content, test_latin1_french_converted_to_utf8, test_latin1_german_decoded_as_gbk_is_known_tradeoff, test_windows1251_russian_converted_to_utf8

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/data/test_file_manager_table_names.py`

- **行数**：83；**用例数**：15

- **测试类**：无

- **用例**：test_preserves_pure_chinese_name, test_preserves_japanese_name, test_preserves_korean_name, test_preserves_cyrillic_name, test_preserves_mixed_unicode_and_ascii, test_collapses_consecutive_underscores, test_normalizes_spaces_and_hyphens, test_normalizes_special_chars_to_single_underscore, test_strips_file_extension, test_empty_name_returns_unnamed, test_dotfile_name_treated_as_stem, test_only_special_chars_returns_unnamed, test_digit_prefix_gets_underscore, test_result_is_lowercase, test_no_leading_trailing_underscores

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/data/test_json_chinese_serialization.py`

- **行数**：216；**用例数**：17

- **测试类**：TestJsonEnsureAsciiBasic, TestAgentMessageSerialization, TestStreamEventSerialization, TestWorkspaceStateSerialization, TestEdgeCases

- **用例**：test_non_ascii_string_preserved, test_ascii_only_string_unchanged, test_mixed_ascii_and_chinese, test_goal_with_chinese_description_should_be_readable, test_chart_spec_with_chinese_field_names, test_nested_chinese_values_all_preserved, test_ok_event_with_chinese_content, test_error_event_with_chinese_message, test_roundtrip_preserves_chinese, test_state_with_chinese_table_names, test_roundtrip_state_preserves_all_fields, test_default_str_handles_non_serializable_types, test_empty_string, test_none_value, test_emoji_preserved, test_mixed_languages, test_special_json_chars_still_escaped

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/data/test_loader_auth_paths.py`

- **行数**：78；**用例数**：5

- **测试类**：无

- **用例**：test_multi_auth_loaders_expose_expected_paths, test_auth_path_fields_do_not_overlap, test_s3_default_credentials_do_not_require_access_keys, test_s3_access_key_path_requires_both_keys, test_superset_only_exposes_sso_when_configured

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/data/test_local_folder_loader.py`

- **行数**：313；**用例数**：31

- **测试类**：TestConfinedDir, TestLocalFolderDataLoader

- **用例**：test_resolve_valid_relative_path, test_reject_absolute_path, test_reject_dotdot_traversal, test_reject_dotdot_in_middle, test_reject_empty_path, test_symlink_escape_rejected, test_write_creates_parents, test_resolve_with_mkdir_parents, test_repr, test_list_params, test_test_connection_valid_dir, test_test_connection_nonexistent_dir, test_list_tables_recursive, test_list_tables_non_recursive, test_list_tables_with_filter, test_list_tables_with_file_pattern, test_list_tables_path_hierarchy, test_fetch_csv, test_fetch_tsv, test_fetch_parquet, test_fetch_jsonl, test_fetch_subdirectory_file, test_fetch_with_size_limit, test_fetch_path_traversal_rejected, test_fetch_unsupported_type_rejected, test_metadata_parquet, test_metadata_csv, test_ls_root, test_ls_subdirectory, test_ls_with_filter, test_catalog_hierarchy

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/data/test_max_import_rows.py`

- **行数**：68；**用例数**：7

- **测试类**：TestMaxImportRowsCap, TestLoaderImportsMaxImportRows

- **用例**：test_max_import_rows_constant_value, test_size_over_limit_is_capped, test_size_under_limit_is_preserved, test_no_size_defaults_to_max, test_size_exactly_at_limit, test_size_negative_treated_as_given, test_loader_has_max_import_rows

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/data/test_normalize_dtype.py`

- **行数**：97；**用例数**：8

- **测试类**：TestNormalizeDtypeToAppType

- **用例**：test_datetime_types, test_date_types, test_time_types, test_duration_types, test_integer_types, test_float_types, test_boolean_types, test_string_fallback

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/data/test_parquet_utils_table_names.py`

- **行数**：32；**用例数**：5

- **测试类**：无

- **用例**：test_parquet_sanitize_keeps_output_non_empty_for_dangerous_input, test_parquet_sanitize_keeps_ascii_names_lowercase, test_parquet_sanitize_preserves_pure_chinese_table_name, test_parquet_sanitize_preserves_unicode_when_name_starts_with_digit, test_parquet_sanitize_normalizes_separators_without_losing_unicode

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/data/test_phase5_agent_metadata.py`

- **行数**：395；**用例数**：26

- **测试类**：TestWorkspaceSearch, TestCatalogCache, TestFieldSummaryWithDescription, TestSummarySystemDescription, TestCatalogMetadataLookups, TestMergeSourceMetadataEmptyClear, TestProgressiveContext

- **用例**：test_match_table_name, test_match_table_description, test_match_column_name, test_match_column_description, test_empty_query_returns_empty, test_no_match, test_respects_limit, test_save_and_load, test_load_nonexistent_returns_none, test_delete_catalog, test_search_catalog_cache, test_search_excludes_imported_tables, test_no_description, test_with_description, test_with_verbose_name, test_with_expression, test_with_all_metadata, test_only_source_description, test_uses_system_description_from_workspace_metadata, test_attached_metadata_is_ignored_if_present, test_lookups_use_workspace_user_home, test_lookups_graceful_without_user_home, test_empty_description_clears_existing, test_missing_key_preserves_existing, test_few_tables_includes_samples, test_many_tables_include_bounded_samples

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/data/test_plugin_scanner.py`

- **行数**：282；**用例数**：13

- **测试类**：无

- **用例**：test_scanner_disabled_in_hosted_mode, test_scanner_enabled_in_local_mode, test_scanner_opt_in_overrides_hosted_gate, test_missing_dependency_recorded_with_pip_hint, test_no_subclass_recorded_in_disabled, test_broken_plugin_does_not_leak_sys_modules, test_plugin_overriding_builtin_is_rejected, test_duplicate_plugin_keys_are_rejected, test_multiple_subclasses_registers_first_alphabetically, test_missing_plugin_dir_is_silent, test_empty_plugin_dir_is_silent, test_plugin_dir_defaults_to_data_formulator_home, test_df_plugin_dir_overrides_data_formulator_home

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/data/test_safe_data_filename.py`

- **行数**：28；**用例数**：3

- **测试类**：无

- **用例**：test_chinese_filename_preserves_unicode, test_path_traversal_strips_directory_components, test_empty_filename_raises_valueerror

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/data/test_source_metadata.py`

- **行数**：532；**用例数**：35

- **测试类**：TestColumnInfoDescription, TestTableMetadataColumnDescriptions, TestMergeSourceMetadata, TestIngestMetadataEnrichment, TestGetColumnTypesDefault, TestInferSourceMetadataStatus, TestCatalogTreeMetadataStatus, TestRefreshPreservesImportOptions, TestFormatImportOptions, TestSourceMetadataImportOptions

- **用例**：test_from_dict_without_description, test_from_dict_with_description, test_to_dict_omits_none_description, test_to_dict_includes_description_when_present, test_roundtrip_with_description, test_roundtrip_without_description, test_serialize_columns_with_descriptions, test_deserialize_old_metadata_no_column_description, test_merges_table_description, test_merges_column_descriptions, test_no_crash_on_empty_source_meta, test_no_crash_when_columns_none, test_empty_description_clears_existing, test_missing_key_preserves_existing, test_ingest_enriches_metadata_on_success, test_ingest_succeeds_when_metadata_fails, test_preserves_table_description, test_returns_empty_when_no_metadata, test_none_metadata_returns_unavailable, test_empty_metadata_returns_unavailable, test_columns_without_descriptions_returns_synced, test_columns_with_description_returns_synced, test_both_table_and_column_metadata_returns_synced, test_explicit_status_overrides_inference, test_no_columns_key_returns_partial_with_desc, test_empty_columns_returns_partial, test_tree_leaf_has_status_injected, test_explicit_status_preserved_in_tree, test_import_options_retained_after_refresh, test_empty_returns_empty, test_sort_only, test_filters_and_limit, test_full_options, test_import_options_in_response, test_import_options_none_when_absent

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/data/test_superset_catalog_sync.py`

- **行数**：362；**用例数**：18

- **测试类**：TestGetDatasetColumns, TestListTablesUuidPassthrough, TestSupersetSyncCatalogMetadata, TestListTablesIncludesColumns, TestBuildColumnEntryExtra

- **用例**：test_calls_correct_endpoint, test_returns_empty_on_empty_result, test_fallback_to_detail_on_http_error, test_propagates_error_when_both_endpoints_fail, test_uuid_and_description_in_metadata, test_no_uuid_when_missing, test_enriches_columns_and_sets_table_key, test_column_fetch_failure_marks_unavailable, test_empty_columns_marks_partial, test_table_key_fallback_without_uuid, test_list_tables_includes_columns, test_list_tables_column_failure_marks_unavailable, test_certification_extra, test_warning_markdown_extra, test_empty_extra_no_effect, test_null_extra_no_effect, test_invalid_json_extra_no_crash, test_extra_combined_with_other_fields

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/data/test_superset_smart_filter.py`

- **行数**：499；**用例数**：52

- **测试类**：TestNormalizeColumnType, TestBuildChartDataFilters, TestBuildChartDataOrderby, TestGetColumnTypes, TestGetColumnValues, TestFetchDataAsArrow, TestSupersetURLResolution, TestValidateParams, TestClassifyConnectorError

- **用例**：test_classification, test_is_dttm_takes_precedence, test_empty_filters, test_eq_filter, test_neq_filter, test_comparison_operators, test_in_filter, test_not_in_filter, test_between_splits_into_two, test_is_null_filter, test_is_not_null_filter, test_like_filter, test_ilike_filter, test_invalid_operator_skipped, test_missing_column_skipped, test_in_with_empty_list_skipped, test_between_with_wrong_length_skipped, test_multiple_filters, test_non_dict_filter_skipped, test_empty_sort_columns, test_ascending, test_descending, test_defaults_to_ascending, test_multiple_columns, test_skips_invalid_columns, test_returns_normalized_types, test_invalid_source_table, test_api_failure_returns_empty, test_tier1_datasource_api, test_tier2_dataset_distinct, test_tier3_chart_data_fallback, test_invalid_source_table, test_keyword_filtering, test_has_more_pagination, test_limit_clamped, test_deduplication, test_all_tiers_fail_returns_empty, test_passes_sort_to_chart_data_api, test_passes_filters_and_sort, test_empty_result_returns_column_schema …

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/data/test_sync_catalog_api.py`

- **行数**：316；**用例数**：7

- **测试类**：TestSyncCatalogMetadataEndpoint

- **用例**：test_connection, test_returns_tree_and_summary, test_partial_sync_returns_message_code, test_full_sync_returns_complete_message_code, test_timeout_returns_catalog_sync_timeout, test_missing_connector_returns_error, test_sync_then_catalog_tree_preserves_column_metadata_in_cache

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/data/test_sync_catalog_cross_db.py`

- **行数**：438；**用例数**：9

- **测试类**：TestPostgreSQLSyncCatalogMetadata, TestMSSQLSyncCatalogMetadata

- **用例**：test_single_db_delegates_to_list_tables, test_multi_db_iterates_all_databases, test_multi_db_skips_failing_database, test_multi_db_table_filter_applied, test_single_db_delegates_to_list_tables, test_list_tables_groups_tables_and_views, test_ls_browses_tables_and_views_directories, test_multi_db_iterates_all_databases, test_multi_db_skips_failing_database

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/data/test_table_name_contracts.py`

- **行数**：24；**用例数**：3

- **测试类**：无

- **用例**：test_route_sanitize_should_not_turn_pure_chinese_name_into_placeholder, test_route_sanitize_should_keep_unicode_and_apply_safe_prefix_if_needed, test_route_sanitize_should_delegate_to_parquet_sanitizer_for_ascii_name

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/data/test_unicode_table_name_sanitization.py`

- **行数**：51；**用例数**：3

- **测试类**：无

- **用例**：test_parquet_sanitize_table_name_should_preserve_unicode, test_external_loader_sanitize_should_preserve_unicode, test_sql_sanitize_should_preserve_unicode_identifiers

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/data/test_workspace_fresh_names.py`

- **行数**：30；**用例数**：3

- **测试类**：无

- **用例**：test_workspace_get_fresh_name_appends_numeric_suffix_for_ascii_name, test_workspace_get_fresh_name_preserves_unicode_and_suffixes, test_workspace_get_fresh_name_applies_safe_prefix_without_losing_unicode

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/data/test_workspace_manager.py`

- **行数**：537；**用例数**：37

- **测试类**：TestWorkspaceLifecycle, TestSessionState, TestOpenWorkspace, TestWorkspaceMigrationOps, TestLegacyWorkspaceAutoRepair, TestEmptyWorkspaceVisibility

- **用例**：test_list_empty, test_create_workspace, test_create_duplicate_raises, test_list_workspaces, test_workspace_exists, test_workspace_exists_is_directory_based, test_delete_workspace, test_delete_nonexistent, test_rename_workspace, test_rename_nonexistent_raises, test_rename_to_existing_raises, test_save_and_load, test_sensitive_fields_stripped, test_load_nonexistent, test_update_display_name_patches_session_state, test_update_display_name_skips_missing_session_state, test_save_to_nonexistent_workspace_raises, test_overwrite_session_state, test_open_workspace_returns_workspace, test_open_nonexistent_raises, test_create_and_open_workspace, test_write_data_in_workspace, test_session_state_persists_with_data, test_move_workspaces_from_merges_existing_and_cleans_source, test_delete_all_workspaces_removes_dirs_and_files, test_move_workspaces_from_succeeds_when_source_locked, test_delete_all_workspaces_skips_locked_entries, test_legacy_workspace_with_only_yaml_appears_in_list, test_legacy_workspace_with_only_session_state_appears_in_list, test_legacy_workspace_with_empty_dir_appears_in_list, test_workspace_exists_consistent_with_create, test_move_legacy_workspace_auto_repairs_meta, test_provisional_workspace_is_hidden, test_naming_promotes_a_provisional_workspace, test_content_written_outside_save_is_still_listed, test_workspace_with_tables_is_visible, test_zero_count_workspace_is_visible

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/data/test_workspace_path_safety.py`

- **行数**：49；**用例数**：5

- **测试类**：TestWorkspacePathTraversal

- **用例**：test_normal_identity_succeeds, test_dotdot_identity_sanitized_safely, test_slash_identity_sanitized_safely, test_empty_identity_rejected, test_path_must_be_under_root

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/data/test_workspace_scratch.py`

- **行数**：61；**用例数**：2

- **测试类**：无

- **用例**：test_prune_scratch_uses_backend_neutral_confined_directory, test_deleting_azure_workspace_removes_local_scratch

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/data/test_workspace_source_file_ops.py`

- **行数**：59；**用例数**：3

- **测试类**：无

- **用例**：test_deletes_all_tables_from_matching_source, test_does_not_affect_other_source_files, test_returns_empty_when_no_match

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/data_loader/test_auth_paths.py`

- **行数**：42；**用例数**：3

- **测试类**：无

- **用例**：test_mysql_declares_password_auth_path, test_mysql_validation_materializes_defaults, test_mssql_sorted_fetch_places_order_by_after_top

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/data_loader/test_clickhouse_loader.py`

- **行数**：452；**用例数**：21

- **测试类**：无

- **用例**：test_static_contract_and_sensitive_password, test_constructor_maps_secure_read_only_options, test_constructor_normalizes_none_optional_parameters, test_constructor_preserves_falsy_non_none_parameters, test_connection_failure_redacts_password, test_omitted_port_follows_transport_default, test_fetch_enforces_row_cap_and_quotes_identifiers, test_fetch_rejects_invalid_sizes, test_fetch_parameterizes_filters_and_quotes_sort_columns, test_fetch_rejects_raw_sql_and_dangerous_identifiers, test_list_tables_returns_arrow_metadata_and_excludes_catalog_details, test_pinned_database_catalog_and_lazy_browsing, test_probe_cannot_escape_pinned_database, test_get_metadata_returns_empty_when_a_query_fails, test_close_and_connection_check, test_cast_function_covers_lossy_types, test_fetch_projects_lossy_columns, test_fetch_projects_within_explicit_column_list, test_fetch_keeps_plain_star_without_lossy_columns, test_fetch_survives_column_type_lookup_failure, test_metadata_sample_projects_lossy_columns

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/data_loader/test_databricks_connection.py`

- **行数**：134；**用例数**：7

- **测试类**：无

- **用例**：test_token_is_default_auth_path_without_oauth, test_sign_in_path_appears_when_oauth_configured, test_token_path_requires_access_token, test_hierarchy_is_catalog_schema_table, test_effective_hierarchy_pins_provided_catalog, test_resolve_source_table_variants, test_live_unity_catalog_roundtrip

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/data_loader/test_kusto_connection.py`

- **行数**：178；**用例数**：11

- **测试类**：无

- **用例**：test_connection_uses_direct_sdk_probe, test_connection_returns_false_when_live_probe_fails, test_database_is_required, test_service_principal_path_requires_complete_credentials, test_ambient_path_does_not_require_service_principal_fields, test_microsoft_sign_in_is_default_when_oauth_is_configured, test_ambient_is_default_when_oauth_is_not_configured, test_connector_manifest_preserves_root_oauth_url, test_delegated_credential_refreshes_expired_token, test_legacy_complete_service_principal_infers_path, test_database_options_are_loaded_only_on_demand

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/data_loader/test_probe.py`

- **行数**：435；**用例数**：29

- **测试类**：TestCompileProbeSql, TestProbeViaDuckDB, TestProbeUnavailable, TestProbeViaSql, TestSqlDialects, TestKustoKql, TestMongoPipeline

- **用例**：test_sample_projection, test_sample_all_columns, test_count, test_group_by_count_order, test_count_distinct, test_filter_applied, test_invalid_agg_op_raises, test_count_distinct_without_column_raises, test_count, test_distinct_values_with_frequency, test_filter_applied_locally_even_when_loader_ignores_it, test_date_range, test_sample_projection, test_output_capped_at_probe_max_rows, test_scan_cap_marks_approximate, test_empty_path_errors, test_base_probe_reports_unavailable, test_compiles_native_sql_and_returns_exact, test_filter_compiles_into_where, test_invalid_query_returns_error_without_executing, test_mssql_top_and_brackets, test_mysql_backtick_and_emulated_ilike, test_bigquery_backtick_path_relation, test_summarize_by_pipeline, test_projection_and_take, test_invalid_agg_raises, test_group_pipeline_with_distinct, test_between_match, test_invalid_agg_raises

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/data_operations/test_executor.py`

- **行数**：169；**用例数**：5

- **测试类**：无

- **用例**：test_executor_materializes_bounded_table_with_provenance, test_executor_publishes_source_descriptions, test_executor_keeps_successful_tables_when_later_step_fails, test_executor_allocates_distinct_fresh_names, test_executor_recovers_published_tables_without_refetching

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/data_operations/test_models.py`

- **行数**：184；**用例数**：10

- **测试类**：无

- **用例**：test_operation_round_trip_preserves_identity_and_status, test_operation_round_trip_preserves_discovery_presentation, test_plan_hash_depends_on_executable_steps_not_display_text, test_plan_rejects_tampered_serialized_hash, test_operation_rejects_unknown_selected_plan, test_models_are_immutable, test_load_query_rejects_multiple_order_clauses, test_nested_filter_values_cannot_change_after_hashing, test_action_factory_owns_the_versioned_wire_envelope, test_load_query_requires_positive_limit_and_valid_order

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/data_operations/test_repository.py`

- **行数**：186；**用例数**：10

- **测试类**：无

- **用例**：test_repository_round_trips_full_executable_operation, test_create_supersedes_only_same_conversation, test_select_is_idempotent_and_rejects_conflicts, test_interaction_response_returns_trusted_plan_label, test_elaborate_requires_an_awaiting_operation, test_execution_transitions_are_atomic_and_idempotent, test_failed_execution_records_typed_error, test_repository_lives_under_ephemeral_workspace_scratch, test_finish_records_partial_result, test_select_rejects_superseded_operation

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/errors/__init__.py`

- **行数**：1；**用例数**：0

- **测试类**：无

- **用例**：

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/errors/test_api_error_protocol_contract.py`

- **行数**：127；**用例数**：6

- **测试类**：TestTablesErrorProtocol, TestStreamingErrorProtocol

- **用例**：test_download_db_file_returns_structured_business_error, test_export_csv_missing_table_returns_structured_error, test_export_csv_invalid_delimiter_returns_structured_error, test_db_error_classification_defaults_to_http_200, test_stream_preflight_error_uses_json_error_envelope, test_analyst_streaming_emits_top_level_type_events

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/errors/test_error_handler.py`

- **行数**：509；**用例数**：51

- **测试类**：TestClassifyAndWrapLlmError, TestStreamErrorEvent, TestRegisterErrorHandlers, TestRequestIdMiddleware, TestStreamWarningEvent, TestCollectAndFlushStreamWarnings, TestJsonOk, TestStreamPreflightError, TestErrorCodeHttpStatusMapping

- **用例**：test_auth_error, test_rate_limit, test_context_too_long, test_model_not_found, test_timeout, test_service_error_502, test_content_filter, test_access_denied, test_bad_request, test_unknown_error, test_message_never_contains_original_exception, test_detail_contains_original_for_logging, test_output_is_valid_ndjson_line, test_error_structure, test_token_absent_from_error_event, test_raw_exception_wrapped, test_unicode_safe, test_app_error_returns_200_with_error_body, test_app_error_retryable, test_app_error_debug_mode_includes_detail, test_unexpected_error_returns_500, test_unexpected_error_debug_includes_safe_detail, test_413_returns_unified_format, test_api_404_returns_json, test_response_has_json_content_type, test_response_has_request_id_header, test_client_provided_request_id_is_echoed, test_request_id_is_uuid_when_not_provided, test_output_is_valid_ndjson_line, test_warning_structure, test_detail_included, test_message_code_included, test_unicode_safe, test_no_warnings_returns_empty, test_collect_then_flush, test_flush_clears_accumulator, test_outside_request_context_is_noop, test_basic_success, test_none_data, test_custom_status_code …

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/errors/test_errors.py`

- **行数**：163；**用例数**：14

- **测试类**：TestErrorCode, TestAppErrorConstruction, TestAppErrorToDict, TestAppErrorSubclass

- **用例**：test_error_code_exists, test_code_values_are_strings, test_basic_construction, test_custom_status_code, test_detail_and_retry, test_inherits_from_exception, test_can_be_raised_and_caught, test_to_dict_without_detail, test_to_dict_excludes_detail_by_default, test_to_dict_includes_detail_when_requested, test_to_dict_include_detail_no_op_when_detail_is_none, test_to_dict_with_retry_true, test_subclass_preserves_interface, test_subclass_caught_as_app_error

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/knowledge/__init__.py`

- **行数**：1；**用例数**：0

- **测试类**：无

- **用例**：

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/knowledge/test_knowledge_store.py`

- **行数**：575；**用例数**：71

- **测试类**：TestDataMemory, TestListAll, TestRead, TestWrite, TestDelete, TestValidatePath, TestFrontMatter, TestSearch, TestLoadAlwaysApplyRules, TestFormatRulesBlock, TestTokenizeQuery, TestMatchScore

- **用例**：test_read_creates_reserved_markdown_file, test_persists_across_store_instances_for_same_user, test_append_and_rewrite, test_replace_exact_text_and_delete, test_replace_rejects_missing_match, test_rejects_empty_append_and_oversized_rewrite, test_lists_rules, test_lists_workflows_in_subdirs, test_empty_category_returns_empty, test_front_matter_title_fallback_to_stem, test_reads_content, test_read_nonexistent_raises, test_creates_new_file, test_updates_existing_file, test_auto_adds_front_matter, test_preserves_existing_front_matter, test_writes_workflows_in_subdir, test_deletes_file, test_delete_nonexistent_raises, test_rules_flat_file_ok, test_rules_subdir_rejected, test_workflows_one_subdir_ok, test_workflows_two_subdirs_rejected, test_skills_rejected_as_invalid, test_non_md_extension_rejected, test_invalid_category_rejected, test_empty_path_rejected, test_traversal_blocked_by_confined_dir, test_valid_front_matter, test_no_front_matter_degrades, test_invalid_yaml_degrades, test_search_by_title, test_search_by_filename, test_search_by_body, test_empty_query_returns_empty, test_no_match_returns_empty, test_max_results_limit, test_search_filters_by_category, test_search_skips_always_apply_rules, test_search_returns_non_always_apply_rules …

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/routes/__init__.py`

- **行数**：1；**用例数**：0

- **测试类**：无

- **用例**：

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/routes/test_agent_diagnostics_wiring.py`

- **行数**：80；**用例数**：3

- **测试类**：TestDataLoadAgentWiring

- **用例**：test_run_attaches_diagnostics, test_run_parse_failure_still_has_diagnostics, test_init_backward_compatible_without_model_info

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/routes/test_analyst_data_operation_flow.py`

- **行数**：227；**用例数**：3

- **测试类**：无

- **用例**：test_operation_preview_is_bounded_and_display_only, test_selected_operation_executes_without_model_turn, test_expired_operation_resumes_analyst_for_rediscovery

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/routes/test_create_table_replace_source.py`

- **行数**：122；**用例数**：3

- **测试类**：无

- **用例**：test_overwrite_same_table_name, test_replace_source_removes_old_tables, test_fewer_sheets_after_replace_no_orphans

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/routes/test_create_table_xls_upload.py`

- **行数**：176；**用例数**：5

- **测试类**：无

- **用例**：test_upload_xls_creates_table_and_returns_columns, test_upload_xls_preserves_chinese_column_names, test_upload_xls_table_name_sanitized_for_unicode, test_upload_xls_rejects_missing_table_name, test_list_tables_returns_sample_rows_for_xls

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/routes/test_credential_routes.py`

- **行数**：192；**用例数**：11

- **测试类**：TestStoreEndpoint, TestListEndpoint, TestDeleteEndpoint, TestUserIsolation

- **用例**：test_store_success, test_store_missing_fields, test_store_no_vault_returns_error, test_list_empty, test_list_after_store, test_list_no_vault_returns_empty, test_delete_success, test_delete_missing_key, test_delete_no_vault_returns_error, test_different_users_see_own_credentials, test_delete_only_affects_own

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/routes/test_credentials_contract.py`

- **行数**：125；**用例数**：8

- **测试类**：TestListCredentials, TestStoreCredential, TestDeleteCredential

- **用例**：test_no_vault_returns_empty_sources, test_vault_returns_sources, test_no_vault_returns_service_unavailable, test_missing_fields_returns_invalid_request, test_success_returns_source_key, test_no_vault_returns_service_unavailable, test_missing_source_key_returns_invalid_request, test_success_returns_source_key

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/routes/test_csv_encoding_roundtrip.py`

- **行数**：155；**用例数**：7

- **测试类**：TestWorkspaceRoundTrip, TestParseFileEndpoint, TestCreateAndGetTable

- **用例**：test_gbk_csv_columns_are_chinese, test_gbk_csv_cell_values_are_correct, test_utf8_csv_still_works, test_utf8_bom_csv_columns_correct, test_gbk_csv_parse_returns_chinese_columns, test_gbk_csv_parse_returns_correct_rows, test_gbk_upload_then_get_returns_chinese

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/routes/test_data_loaders_discovery.py`

- **行数**：215；**用例数**：5

- **测试类**：无

- **用例**：test_plugin_appears_in_discovery_endpoint, test_builtin_loader_marked_as_builtin, test_mysql_auth_path_surfaces_in_discovery, test_display_name_default_titlecases_registry_key, test_plugins_block_surfaces_loaded_and_rejected

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/routes/test_data_loading_chat_route.py`

- **行数**：159；**用例数**：4

- **测试类**：TestDataLoadingChatValidation, TestDataLoadingChatSuccess, TestDataLoadingChatErrors

- **用例**：test_non_json_request_returns_error, test_messages_forwarded_to_agent, test_image_messages_are_forwarded, test_agent_exception_streams_error_event

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/routes/test_knowledge_contract.py`

- **行数**：126；**用例数**：9

- **测试类**：TestKnowledgeLimits, TestKnowledgeList, TestKnowledgeRead, TestKnowledgeSearch

- **用例**：test_returns_limits_in_success_envelope, test_missing_category_returns_error, test_invalid_category_returns_error, test_success_returns_items, test_missing_fields_returns_error, test_file_not_found_returns_error, test_success_returns_content, test_success_returns_results, test_invalid_categories_type_returns_error

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/routes/test_knowledge_routes.py`

- **行数**：456；**用例数**：28

- **测试类**：TestDataMemory, TestKnowledgeList, TestKnowledgeRead, TestKnowledgeWrite, TestKnowledgeDelete, TestKnowledgeSearch, TestDistillWorkflow

- **用例**：test_read_creates_user_memory, test_append_then_rewrite, test_rejects_non_string_content, test_list_empty, test_list_with_entries, test_list_invalid_category, test_list_missing_category, test_read_existing, test_read_nonexistent, test_read_traversal_rejected, test_write_creates_file, test_write_updates_file, test_write_traversal_rejected, test_write_non_md_rejected, test_delete_existing, test_delete_nonexistent, test_search_returns_results, test_search_empty_query, test_search_invalid_category, test_search_filters_by_category, test_distill_workflow_from_context, test_distill_workflow_llm_timeout_returns_structured_error, test_distill_workflow_missing_context, test_distill_workflow_missing_threads, test_distill_workflow_missing_workspace, test_distill_session_uses_descriptive_title, test_distill_session_upserts_existing_workspace_file, test_distill_session_strips_legacy_title_prefix

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/routes/test_list_global_models_api.py`

- **行数**：91；**用例数**：5

- **测试类**：TestListGlobalModelsEndpoint

- **用例**：test_returns_all_configured_models, test_response_has_required_fields, test_no_api_key_in_response, test_all_models_marked_global, test_empty_env_returns_empty_list

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/routes/test_parse_file_endpoint.py`

- **行数**：89；**用例数**：4

- **测试类**：无

- **用例**：test_parse_xls_returns_sheet_data, test_parse_file_rejects_missing_file, test_parse_file_rejects_unsupported_format, test_parse_csv_via_endpoint

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/routes/test_same_basename_upload.py`

- **行数**：273；**用例数**：7

- **测试类**：TestPreviewIdVsWorkspaceIdMismatch, TestSameBasenameDifferentExtension, TestFileArrayMismatch

- **用例**：test_preview_id_differs_from_workspace_name, test_orphan_cleanup_would_wrongly_remove_on_reupload, test_both_tables_exist_after_upload, test_both_readable_after_upload, test_reupload_csv_does_not_destroy_other_table, test_replace_source_only_affects_same_source_file, test_index_fallback_sends_wrong_file

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/routes/test_sample_table_pagination.py`

- **行数**：135；**用例数**：8

- **测试类**：TestSampleTablePagination

- **用例**：test_no_offset_returns_first_page, test_offset_skips_rows, test_offset_with_desc_sort, test_offset_with_desc_sort_page2, test_offset_beyond_total_returns_empty, test_total_row_count_consistent_across_pages, test_backward_compat_no_offset, test_pages_cover_all_rows_without_overlap

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/routes/test_session_export_import.py`

- **行数**：204；**用例数**：8

- **测试类**：TestExportSession, TestImportSession

- **用例**：test_export_uses_workspace_id_from_body, test_export_rejects_missing_workspace_id, test_export_rejects_missing_state, test_export_returns_error_for_unknown_workspace, test_import_creates_workspace_when_not_existing, test_import_opens_existing_workspace, test_import_falls_back_to_active_workspace, test_import_rejects_missing_file

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/routes/test_session_routes_migration.py`

- **行数**：124；**用例数**：5

- **测试类**：TestSaveSessionRoute, TestMigrateRoute, TestCleanupAnonymousRoute

- **用例**：test_save_session_reports_storage_full, test_migrate_moves_and_cleans_source, test_migrate_rejects_non_user, test_cleanup_success, test_cleanup_rejects_non_user

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/routes/test_upload_parquet_conversion.py`

- **行数**：324；**用例数**：17

- **测试类**：TestParquetConversion, TestMultiSheetEnglish, TestMultiSheetChinese, TestMultiSheetMixed, TestReplaceSourceWithParquet, TestMetadataIntegrity

- **用例**：test_csv_upload_stored_as_parquet, test_xlsx_upload_stored_as_parquet, test_gbk_csv_converted_correctly, test_upload_orders_sheet_via_suffix, test_upload_returns_sheet_via_suffix, test_upload_both_sheets_coexist, test_sheet_hint_overrides_inference, test_invalid_sheet_hint_falls_back, test_upload_sales_sheet, test_upload_profit_sheet, test_chinese_sheet_hint, test_upload_summary_sheet, test_upload_detail_sheet_chinese, test_upload_q1_sheet, test_load_all_three_sheets, test_replace_source_removes_parquet_tables, test_source_file_and_original_name_recorded

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/routes/test_workspace_name_api.py`

- **行数**：86；**用例数**：2

- **测试类**：TestWorkspaceNameEndpoint

- **用例**：test_workspace_name_uses_selected_model_payload, test_workspace_name_requires_model

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/security/test_code_signing.py`

- **行数**：94；**用例数**：13

- **测试类**：TestSignVerifyRoundTrip, TestSignResult

- **用例**：test_valid_signature_accepted, test_tampered_code_rejected, test_tampered_signature_rejected, test_empty_code_returns_empty_sig, test_empty_code_verify_returns_false, test_empty_signature_verify_returns_false, test_whitespace_matters, test_unicode_code, test_signature_is_hex_string, test_adds_signature_when_code_present, test_no_signature_when_code_empty, test_no_signature_when_code_missing, test_returns_result_for_chaining

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/security/test_confined_dir_extended.py`

- **行数**：176；**用例数**：26

- **测试类**：TestReadText, TestWriteText, TestExists, TestIterdir, TestRglob, TestUnlink, TestExistingApiRegression

- **用例**：test_reads_existing_file, test_reads_utf8_by_default, test_traversal_raises, test_nonexistent_file_raises, test_creates_new_file, test_creates_parent_dirs, test_returns_resolved_path, test_traversal_raises, test_existing_file_returns_true, test_missing_file_returns_false, test_traversal_returns_false, test_directory_returns_true, test_lists_root_contents, test_lists_subdirectory, test_empty_directory_returns_empty, test_traversal_raises, test_finds_matching_files, test_no_match_returns_empty, test_rglob_in_subdirectory, test_deletes_existing_file, test_traversal_raises, test_nonexistent_raises, test_resolve_normal, test_resolve_traversal_raises, test_write_bytes, test_truediv_operator

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/security/test_confined_dir_migration.py`

- **行数**：247；**用例数**：23

- **测试类**：TestWorkspaceConfinedProperties, TestAgentToolsUseConfinedProperties, TestScratchRoutesConfinedMigration

- **用例**：test_confined_root_is_confineddir, test_confined_data_is_confineddir, test_confined_scratch_is_confineddir, test_confined_root_points_to_workspace_path, test_confined_data_points_to_data_subdir, test_confined_scratch_points_to_scratch_subdir, test_confined_root_rejects_traversal, test_confined_data_rejects_traversal, test_confined_scratch_rejects_traversal, test_get_file_path_uses_confined_data, test_get_file_path_traversal_sanitized, test_data_dir_created, test_scratch_dir_created, test_read_file_traversal_blocked, test_write_file_traversal_blocked, test_list_directory_traversal_blocked, test_preview_scratch_traversal_blocked, test_scratch_serve_normal, test_scratch_serve_traversal_rejected, test_scratch_serve_nonexistent, test_scratch_upload_normal, test_scratch_upload_traversal_sanitized, test_scratch_upload_no_file_returns_error

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/security/test_docker_sandbox_path.py`

- **行数**：147；**用例数**：7

- **测试类**：TestOutputPathValidation

- **用例**：test_traversal_via_slashes_neutralized, test_traversal_via_dotdot_neutralized, test_empty_output_variable_rejected, test_normal_output_variable_not_rejected, test_output_variable_with_separator, test_docker_stderr_returns_sanitized_diagnostics, test_start_exception_returns_generic_content

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/security/test_global_model_security.py`

- **行数**：265；**用例数**：17

- **测试类**：TestGetClientGlobalResolution, TestSharedErrorSanitization, TestClassifyLlmError

- **用例**：test_global_model_gets_real_api_key, test_user_model_keeps_own_credentials, test_global_claim_for_unregistered_id_is_rejected, test_global_claim_cannot_bypass_the_api_base_allowlist, test_user_model_api_base_is_still_validated, test_resolving_a_global_model_does_not_mutate_the_registry, test_sanitize_redacts_api_key_patterns, test_sanitize_truncates_long_messages, test_sanitize_escapes_html, test_auth_error_401, test_auth_error_invalid_key, test_rate_limit_429, test_context_length, test_model_not_found, test_timeout, test_unknown_error_generic_fallback, test_never_includes_raw_exception_text

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/security/test_local_folder_deployment.py`

- **行数**：74；**用例数**：4

- **测试类**：TestLocalFolderDeploymentRestriction

- **用例**：test_local_mode_keeps_local_folder, test_multi_user_mode_disables_local_folder, test_ephemeral_mode_disables_local_folder, test_create_connector_rejects_disabled_type

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/security/test_log_sanitizer.py`

- **行数**：324；**用例数**：41

- **测试类**：TestSanitizeUrl, TestSanitizeParams, TestRedactToken, TestApplyPatterns, TestSensitiveDataFilter

- **用例**：test_url_with_password, test_url_without_credentials, test_url_with_port, test_empty_string, test_non_url_string, test_s3_url_no_creds, test_multiple_urls_in_string, test_url_with_sensitive_query_params, test_url_without_sensitive_query_params_preserves_query, test_masks_password, test_masks_multiple_keys, test_case_insensitive, test_nested_dict, test_does_not_mutate_original, test_extra_keys, test_empty_dict, test_connection_string, test_long_token, test_short_token_fully_masked, test_empty_token, test_custom_visible, test_boundary_length, test_url_credentials, test_url_query_credentials, test_key_value_password, test_key_value_api_key, test_key_value_with_colon, test_bearer_token, test_bare_jwt, test_dict_repr_with_password, test_dict_repr_double_quotes, test_no_false_positive, test_no_false_positive_on_normal_url, test_mixed_patterns, test_filter_redacts_password_in_message, test_filter_redacts_url_creds, test_filter_percent_style_args, test_filter_disabled_by_env, test_filter_always_returns_true, test_filter_handles_bad_format_args …

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/security/test_sandbox.py`

- **行数**：516；**用例数**：34

- **测试类**：TestCreateSandbox, TestLocalSandbox, TestDockerSandbox, TestDockerSandboxNoDocker, TestSandboxSession, TestSandboxSessionSaveRestore

- **用例**：test_local, test_docker, test_default_is_local, test_simple_transform, test_derived_columns, test_read_csv_from_workspace, test_read_parquet_from_workspace, test_parquet_modules_preimported, test_write_to_workdir_blocked, test_syntax_error, test_runtime_error, test_non_dataframe_output, test_duckdb_sql, test_simple_transform, test_derived_columns, test_read_csv_from_workspace, test_syntax_error, test_runtime_error, test_non_dataframe_output, test_json_usage, test_duckdb_sql, test_missing_docker_returns_error, test_variable_persists_across_calls, test_dataframe_persists, test_close_clears_namespace, test_context_manager, test_error_does_not_break_session, test_backward_compat_3tuple, test_save_and_restore_dataframe, test_save_scalars_only, test_save_empty_namespace_returns_false, test_restore_missing_dir_returns_false, test_restore_does_not_clobber_existing_vars, test_multiple_dataframes

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/security/test_sandbox_security.py`

- **行数**：158；**用例数**：10

- **测试类**：TestLocalSandboxFileWriteBlocked, TestLocalSandboxProcessExecBlocked, TestDockerSandboxSecurity

- **用例**：test_open_write_blocked, test_csv_write_blocked, test_os_system, test_os_popen, test_os_execvp, test_os_spawnlp, test_os_kill, test_os_via_sys_modules, test_os_putenv, test_workspace_readonly

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/security/test_sanitize.py`

- **行数**：126；**用例数**：16

- **测试类**：TestApiKeyRedaction, TestPathRedaction, TestStackTraceStripping, TestHtmlEscaping, TestTruncation, TestEdgeCases

- **用例**：test_api_key_equals, test_api_token_equals, test_generic_token_equals, test_password_equals, test_no_false_positive_on_normal_text, test_unix_home_path, test_unix_opt_path, test_windows_path, test_tmp_path, test_traceback_removed, test_file_line_references_stripped, test_script_tag_escaped, test_long_message_truncated, test_short_message_not_truncated, test_empty_string, test_unicode_error

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/security/test_scratch_serve.py`

- **行数**：81；**用例数**：5

- **测试类**：TestScratchServePathSafety

- **用例**：test_normal_file_served, test_path_traversal_returns_error, test_dotdot_single_returns_error, test_nonexistent_file_returns_error, test_response_uses_send_file_not_send_from_directory

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/security/test_startup_safety.py`

- **行数**：67；**用例数**：3

- **测试类**：TestStartupSafetyChecks

- **用例**：test_multi_user_no_sandbox_logs_critical, test_local_mode_no_warning, test_multi_user_with_docker_sandbox_no_warning

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/security/test_superset_bridge_security.py`

- **行数**：108；**用例数**：4

- **测试类**：TestSupersetBridgeOriginValidation, TestSupersetBridgeTemplate

- **用例**：test_allows_known_data_formulator_origins, test_rejects_untrusted_or_non_origin_values, test_allows_env_configured_origin, test_script_payload_escapes_script_breakout

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/security/test_url_allowlist.py`

- **行数**：178；**用例数**：26

- **测试类**：TestOpenMode, TestEnforceMode, TestEmptyBaseAlwaysAllowed, TestCaseInsensitive, TestPatternLoading, TestGlobEdgeCases

- **用例**：test_any_url_allowed, test_private_ip_allowed_in_open_mode, test_localhost_allowed_in_open_mode, test_empty_base_allowed, test_none_base_allowed, test_openai_allowed, test_azure_wildcard_allowed, test_ollama_localhost_allowed, test_unlisted_url_rejected, test_private_ip_rejected, test_internal_network_rejected, test_localhost_wrong_port_rejected, test_none_allowed_in_enforce_mode, test_empty_string_allowed_in_enforce_mode, test_uppercase_url_matches, test_uppercase_pattern_matches, test_unset_returns_none, test_empty_string_returns_none, test_whitespace_only_returns_none, test_comma_separated_parsed, test_whitespace_trimmed, test_subdomain_wildcard, test_deep_subdomain_wildcard, test_no_path_still_matches_with_slash, test_exact_domain_no_trailing_slash_no_match, test_pattern_without_slash_star_matches_bare_domain

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/test_desktop_single_instance.py`

- **行数**：70；**用例数**：3

- **测试类**：无

- **用例**：test_second_instance_signals_primary, test_unrelated_port_occupant_is_not_treated_as_existing_instance, test_activate_window_restores_and_shows_window

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/test_model_endpoints.py`

- **行数**：96；**用例数**：5

- **测试类**：无

- **用例**：test_sanitize_entry_keeps_only_non_secret_fields, test_history_round_trip_and_deduplication, test_invalid_history_is_treated_as_empty, test_history_file_contains_no_unrecognized_fields, test_api_isolates_history_by_identity_and_drops_keys

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/backend/test_startup_spinner.py`

- **行数**：56；**用例数**：2

- **测试类**：无

- **用例**：test_spinner_output_is_cp1252_safe, test_desktop_standard_streams_replace_unencodable_output

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/conftest.py`

- **行数**：40；**用例数**：0

- **测试类**：无

- **用例**：

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/database-dockers/bigquery/test_bigquery_loader.py`

- **行数**：365；**用例数**：15

- **测试类**：TestBigQueryEmulatorProbe, TestBigQueryDataLoader, TestBigQueryDataLoaderStatic

- **用例**：test_bq_emulator_available_false_quickly_on_closed_port, test_list_tables, test_list_tables_with_filter, test_list_tables_specific_dataset, test_fetch_data_as_arrow_from_table, test_fetch_data_respects_size, test_fetch_data_invalid_table_raises, test_ingest_table_to_workspace, test_ingest_table_auto_name, test_ingest_products_table, test_ingest_orders_table, test_ingest_sanitizes_table_name, test_get_table_info_from_datalake, test_list_params, test_auth_instructions

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/database-dockers/cosmosdb/seed_data.py`

- **行数**：146；**用例数**：0

- **测试类**：无

- **用例**：

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/database-dockers/cosmosdb/test_cosmosdb_loader.py`

- **行数**：336；**用例数**：22

- **测试类**：TestCosmosDBDataLoader, TestCosmosDBDataLoaderStatic

- **用例**：test_list_tables, test_list_tables_with_filter, test_list_tables_specific_container, test_list_tables_row_count, test_fetch_data_as_arrow, test_fetch_data_respects_size, test_ingest_table_to_workspace, test_ingest_nested_documents_flattened, test_ingest_with_arrays_flattened, test_ingest_sanitizes_table_name, test_get_table_info_from_datalake, test_connection_close, test_context_manager, test_flatten_document, test_convert_special_types, test_catalog_hierarchy, test_ls_containers, test_test_connection, test_get_metadata, test_list_params, test_auth_instructions, test_flatten_strips_cosmos_metadata

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/database-dockers/mongodb/test_mongodb_loader.py`

- **行数**：282；**用例数**：17

- **测试类**：TestMongoDBDataLoader, TestMongoDBDataLoaderStatic

- **用例**：test_list_tables, test_list_tables_with_filter, test_list_tables_specific_collection, test_list_tables_row_count, test_fetch_data_as_arrow, test_fetch_data_respects_size, test_ingest_table_to_workspace, test_ingest_nested_documents_flattened, test_ingest_with_arrays_flattened, test_ingest_sanitizes_table_name, test_get_table_info_from_datalake, test_connection_close, test_context_manager, test_flatten_document, test_convert_special_types, test_list_params, test_auth_instructions

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/database-dockers/mysql/test_mysql_datalake.py`

- **行数**：149；**用例数**：3

- **测试类**：TestMySQLDataLake

- **用例**：test_connect_and_list_tables, test_ingest_table_into_datalake, test_get_table_info_from_datalake

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/database-dockers/mysql/test_mysql_loader.py`

- **行数**：225；**用例数**：10

- **测试类**：TestMySQLDataLoader, TestMySQLDataLoaderStatic

- **用例**：test_list_tables, test_list_tables_with_filter, test_fetch_data_as_arrow_from_table, test_fetch_data_respects_size, test_ingest_table_to_workspace, test_ingest_products_table, test_get_table_info_from_datalake, test_list_params, test_auth_instructions, test_fetch_data_as_arrow_uses_source_filters

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/database-dockers/postgres/test_postgresql_loader.py`

- **行数**：373；**用例数**：20

- **测试类**：TestPostgreSQLDataLoader, TestPostgreSQLDataLoaderStatic

- **用例**：test_list_tables, test_list_tables_with_filter, test_fetch_data_as_arrow_from_table, test_fetch_data_respects_size, test_ingest_table_to_workspace, test_ingest_products_table, test_get_table_info_from_datalake, test_list_params, test_auth_instructions, test_connect_forces_utf8_client_encoding, test_resolve_source_table_three_parts, test_resolve_source_table_three_parts_same_db, test_resolve_source_table_two_parts, test_resolve_source_table_one_part, test_ls_table_nodes_include_source_name, test_ls_schema_filters_postgresql_temp_schemas, test_ls_table_level_supports_limit_offset, test_source_filter_helper_compiles_postgres_operators, test_fetch_data_as_arrow_uses_source_filters, test_secondary_connection_forces_utf8_client_encoding

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/database-dockers/superset/sample_data.py`

- **行数**：232；**用例数**：0

- **测试类**：无

- **用例**：

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/database-dockers/superset/superset_config.py`

- **行数**：144；**用例数**：0

- **测试类**：无

- **用例**：

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/database-dockers/superset/test_superset_data_connector.py`

- **行数**：467；**用例数**：16

- **测试类**：TestSupersetAuth, TestSupersetCatalog, TestSupersetData, TestSupersetTokenRefresh, TestSupersetFrontendConfig

- **用例**：test_connect_success, test_connect_bad_credentials, test_auth_mode_is_token, test_disconnect_and_status, test_ls_root_lists_dashboards_and_all_datasets, test_ls_dashboard_lists_its_datasets, test_ls_all_datasets, test_ls_with_filter, test_catalog_metadata, test_list_tables_flat, test_preview, test_import, test_connect_with_expired_token_triggers_refresh, test_config_structure, test_pinned_url, test_hierarchy_is_dashboard_dataset

- **功能简介**：对该路径所暗示的生产模块做回归。夹具在文件内或根 conftest。

#### `tests/frontend/setup.ts`

- **行数**：2；**it 数**：0

- **describe**：

- **用例标题**：

#### `tests/frontend/unit/app/AuthButton.test.tsx`

- **行数**：114；**it 数**：1

- **describe**：AuthButton backend logout

- **用例标题**：clears backend session and switches persisted identity to browser

#### `tests/frontend/unit/app/IdentityMigrationDialog.test.tsx`

- **行数**：235；**it 数**：7

- **describe**：Anonymous user logs in and sees migration dialog, User clicks , User clicks 

- **用例标题**：shows the dialog when anonymous workspaces exist; auto-closes when no anonymous workspaces exist; does NOT call cleanup-anonymous (anonymous data preserved); does NOT call migrate endpoint; never shows ; navigates to home page; calls migrate endpoint and shows importing state

#### `tests/frontend/unit/app/LayoutProvider.test.tsx`

- **行数**：86；**it 数**：6

- **describe**：LayoutProvider, shell allocation at the floor

- **用例标题**：classifies the minimum supported viewport as compact and short; leaves a standard desktop at the reference layout; publishes the scale to CSS so stylesheets can follow; scales button geometry with spacious layouts; seats one thread column and still clears the canvas minimum; gives a wide screen more columns without starving the canvas

#### `tests/frontend/unit/app/OidcCallback.test.tsx`

- **行数**：99；**it 数**：3

- **describe**：OidcCallback error handling

- **用例标题**：redirects to /?auth_error=access_denied when IdP returns error; redirects with encoded error for other IdP error values; proceeds with signinRedirectCallback when no error param

#### `tests/frontend/unit/app/agentInteractionPolicy.test.ts`

- **行数**：12；**it 数**：1

- **describe**：agent interaction policy

- **用例标题**：keeps generated chart auto-focus disabled while the user is viewing a chart

#### `tests/frontend/unit/app/agentMetadataTimeout.test.ts`

- **行数**：204；**it 数**：8

- **describe**：agent metadata thunks

- **用例标题**：does not add a frontend timeout to semantic type requests; retries semantic type inference once for transient model errors; does not retry non-retryable semantic type errors; does not add a frontend timeout to code explanation requests; shows a warning when global model list loading fails; ignores aborted global model list requests; clears testing model status when connectivity check fails; ignores aborted connectivity checks

#### `tests/frontend/unit/app/apiClient.test.ts`

- **行数**：506；**it 数**：37

- **describe**：ApiRequestError, parseApiResponse, new unified format, Phase 2 unified format, strict protocol validation, parseStreamLine, apiRequest, assertDownloadResponseOk, streamRequest

- **用例标题**：should carry apiError and httpStatus; isRetryable returns true when retry flag is set; isRetryable returns false by default; isAuthError returns true for auth codes; isAuthError returns false for non-auth codes; should parse success response; should throw ApiRequestError on error response; should include detail when present; should parse status:; should throw on error with HTTP 4xx status; should throw on error with HTTP 5xx status; should reject legacy error_message field; should reject legacy message field; should reject status:; should reject legacy result field; should parse a normal event; should parse an error event; should parse a done event; should return null for empty lines; should return null for malformed JSON; should handle unicode content; should reject legacy status-wrapped events; should reject legacy status error wrapper; should return data on 200 + status:; should reject 200 + status:

#### `tests/frontend/unit/app/chartInsightContract.test.ts`

- **行数**：163；**it 数**：5

- **describe**：chart insight contract

- **用例标题**：invalidates insight text when channel or aggregation changes; passes title and subtitle through Flint assembly; applies a Flint theme preset to the assembled base chart; does not let Flint mutate frozen chart properties while applying theme defaults; preserves the dark canvas supplied by the Power BI theme

#### `tests/frontend/unit/app/clarification.test.ts`

- **行数**：116；**it 数**：9

- **describe**：clarification helpers

- **用例标题**：normalizes structured clarify events and translates backend codes; defaults responseType to free_text when no options are provided; defaults responseType to single_choice when options are provided; accepts bare-string options; preserves opaque option values separately from translated labels; rejects clarify events without questions; formats single response as just the answer; formats multiple selections with 1-based indices; appends freeform text on its own line after selections

#### `tests/frontend/unit/app/connectorFormPersistence.test.ts`

- **行数**：53；**it 数**：2

- **describe**：connector form persistence

- **用例标题**：removes transient prefills from standalone chat messages; removes transient prefills from generalized form artifacts

#### `tests/frontend/unit/app/dfSelectors.test.ts`

- **行数**：118；**it 数**：10

- **describe**：dfSelectors.getActiveModel

- **用例标题**：should return the selected model when it exists; should fall back to the first model when selectedModelId does not match; should return undefined when the models array is empty; should return undefined when models is empty even with a selectedModelId; should return the first model when selectedModelId is undefined; should find a model in globalModels by selectedModelId; should prefer exact match in globalModels over first user model; should fall back to first globalModel when no id matches and models is empty; should fall back to globalModel (first in combined array) over user model; should handle undefined globalModels gracefully

#### `tests/frontend/unit/app/dfSliceTableCollections.test.ts`

- **行数**：471；**it 数**：11

- **describe**：split table collections, text artifact canvas ownership

- **用例标题**：stores a Flint theme and returns an active custom variant to the base chart; stores inferred field semantics separately from physical metadata; automatically migrates legacy tables when state is loaded; drops the retired mini agent setting from loaded state; stores input metadata without rows and tracks derived tables separately; stores an authored parent edge when a report is finalized; preserves a draft parent edge when promoting its result; repairs authored child edges when a text turn is removed; removes a terminal response; removes loaded-table references with their shelf table; reparents authored children when a derived table is removed

#### `tests/frontend/unit/app/errorCodes.test.ts`

- **行数**：89；**it 数**：9

- **describe**：ERROR_CODE_I18N_MAP, getErrorMessage

- **用例标题**：should have mappings for all major error codes; should map to errors.* i18n keys; should return translated message for known code; should return translated message for LLM rate limit; should fall back to backend message for unknown code; should fall back to backend message when i18n key has no translation; should use TABLE_NOT_FOUND translation; should use STORAGE_FULL translation; should use CONNECTOR_AUTH_FAILED translation

#### `tests/frontend/unit/app/errorHandler.test.ts`

- **行数**：233；**it 数**：18

- **describe**：handleApiError, ApiRequestError handling, AbortError handling, plain Error handling, silent option, callback options, RTK serialized error handling, extractErrorMessage

- **用例标题**：should dispatch addMessages for an ApiRequestError; should include detail in the dispatched message; should include request_id in the dispatched message detail; should include diagnostics with the full apiError; should silently ignore AbortError; should dispatch addMessages for a plain Error; should handle non-Error values; should not dispatch when silent is true; should call onAuth for auth errors and skip dispatch; should call onRetryable for retryable errors and skip dispatch; should dispatch normally when callback is not provided for matching error; should extract message from RTK serialized error object; should not produce [object Object] for serialized errors; should extract message from ApiRequestError; should extract message from plain Error; should extract message from RTK serialized error (plain object with .message); should fallback to String() for unknown values; should not return [object Object] for objects with message

#### `tests/frontend/unit/app/fetchWithIdentity.test.ts`

- **行数**：180；**it 数**：9

- **describe**：fetchWithIdentity, Bearer token attachment, 401 retry with silent renew

- **用例标题**：should attach Authorization header when OIDC token is available; should not attach Authorization header in anonymous mode; should always attach X-Identity-Id header; preserves an explicit workspace instead of the active workspace; should not modify headers for non-API URLs; should retry once after silent renew on 401; should return 401 when silent renew fails; should not retry when no UserManager is available; should not retry on non-401 errors

#### `tests/frontend/unit/app/getAccessToken.test.ts`

- **行数**：73；**it 数**：5

- **describe**：getAccessToken

- **用例标题**：returns token when user exists and not expired; returns null when no user stored; calls signinSilent when token is expired and returns refreshed token; returns null when token is expired and signinSilent fails; returns null when token is expired and signinSilent returns null

#### `tests/frontend/unit/app/i18nLocales.test.ts`

- **行数**：31；**it 数**：1

- **describe**：i18n locale bundles

- **用例标题**：keeps Simplified Chinese translation keys aligned with English

#### `tests/frontend/unit/app/inputTablePreviewCache.test.ts`

- **行数**：43；**it 数**：1

- **describe**：input table preview cache

- **用例标题**：bounds rows and invalidates stale content versions

#### `tests/frontend/unit/app/layout.test.ts`

- **行数**：451；**it 数**：44

- **describe**：reference layout, density scaling, size classes, type scale, canvas content sizing, default thread columns, column capacity as the window resizes, chart size stops, minimum screen budget

- **用例标题**：is the app as hardcoded today; leaves geometry untouched at reference density; reproduces the thread pane widths the Allotment snaps to; round-trips pane width back to column count; keeps real headroom below each snap point; never claims more columns than the strip can draw; still counts columns when zoom reports a sub-pixel pane width; uses designed type stops, not a multiplied reference; keeps the ramp rhythm at every density; moves every stop up as density increases; scales lengths but never counts; yields whole pixels; asks for more thread columns as width grows; scales density up with the screen, and only down on small ones; exposes a CSS variable for every token; seeds index.css with the reference values; never gives less room than the old fixed caps; gives a big canvas materially more table; bounds the table so it cannot run away on a 4K screen; raises the chart ceiling with the room, within bounds; holds steady through a drag instead of changing every pixel; never moves backwards as the canvas widens; starts at two once the screen affords it, even with one thread; stays at one where the shell cannot seat two; follows the content once it needs more than two

#### `tests/frontend/unit/app/loadableState.test.ts`

- **行数**：43；**it 数**：4

- **describe**：loadableState

- **用例标题**：preserves previous data while entering loading; marks successful empty data with the empty status; extracts API error messages for error state; starts as idle without data

#### `tests/frontend/unit/app/oidcConfig.test.ts`

- **行数**：65；**it 数**：3

- **describe**：getAuthInfo

- **用例标题**：unwraps the unified API success envelope; keeps legacy flat auth info responses compatible; returns null when auth info is unavailable

#### `tests/frontend/unit/app/rehydrateBackfill.test.ts`

- **行数**：51；**it 数**：3

- **describe**：rehydrating a payload that predates a collection

- **用例标题**：backfills a missing array so consumers can read .length; leaves existing collections untouched; replaces a non-array value with an empty array

#### `tests/frontend/unit/app/stateMigrations.test.ts`

- **行数**：134；**it 数**：5

- **describe**：state migrations

- **用例标题**：applies the single v3 split, semantic extraction, and legacy cleanup; normalizes partial pre-release states into v3 without duplicates; upgrades an already split pre-release state to v4; moves loaded-table thread edges into reference nodes; unifies authored table, draft, and report edges on parentNodeId

#### `tests/frontend/unit/app/tableLoadsInFlight.test.ts`

- **行数**：62；**it 数**：5

- **describe**：tableLoadsInFlight

- **用例标题**：starts at zero; counts a load as in flight until it settles; clears the counter when a load fails; tracks concurrent loads independently; never drops below zero when a settle arrives after a state reset

#### `tests/frontend/unit/app/tableResolution.test.ts`

- **行数**：84；**it 数**：6

- **describe**：workspaceTableIdOf, toAnalystTableRef

- **用例标题**：uses the workspace table id for workspace-backed inputs; uses the materialized copy for connector-backed inputs; falls back to the entry id when a connector input is not materialized; sends the workspace id, the user-facing name, and the snapshot schema; omits row_count when the snapshot has no known count; never carries preview rows

#### `tests/frontend/unit/app/tableThunks.test.ts`

- **行数**：70；**it 数**：6

- **describe**：resolveDatabaseImportLimit, buildDictTableFromWorkspace

- **用例标题**：does not treat an intentional query limit as safety truncation; applies the safety cap to unbounded and oversized imports; preserves column descriptions in metadata; omits description when not provided by backend; uses table-level loader description as DictTable.description; works with no descriptions at all

#### `tests/frontend/unit/app/useAutoSave.test.tsx`

- **行数**：93；**it 数**：2

- **describe**：useAutoSave

- **用例标题**：notifies the frontend when auto-save fails; strips connector form prefills from workspace snapshots

#### `tests/frontend/unit/app/useWorkspaceAutoName.test.tsx`

- **行数**：138；**it 数**：3

- **describe**：useWorkspaceAutoName

- **用例标题**：recognizes the workspace placeholder name; calls the workspace name API with the selected server-managed model payload; does not auto-name a custom workspace name

#### `tests/frontend/unit/app/workspaceService.test.ts`

- **行数**：162；**it 数**：7

- **describe**：ephemeral workspace recovery, local workspace parity

- **用例标题**：stores a row-free browser snapshot after a successful server save; loads an expired server workspace from its browser snapshot as read-only; returns only the server workspace list without consulting recovery storage; propagates a missing-workspace error without consulting recovery storage; saves only to the server without creating a recovery snapshot; hydrates previews against the workspace being loaded; rejects a load superseded by a newer workspace switch

#### `tests/frontend/unit/components/ConnectorTablePreview.test.tsx`

- **行数**：79；**it 数**：2

- **describe**：ConnectorTablePreview source metadata

- **用例标题**：shows the source table description directly; uses descriptions on table headers without restoring the old metadata panel

#### `tests/frontend/unit/components/DataOperationCard.test.tsx`

- **行数**：92；**it 数**：2

- **describe**：DataOperationCard

- **用例标题**：renders immutable plan alternatives without execution controls; shows partial status and failed step names

#### `tests/frontend/unit/components/LoadPlanCard.test.tsx`

- **行数**：159；**it 数**：4

- **describe**：LoadPlanCard, buildLoadQueryImportOptions

- **用例标题**：presents connector and scratch candidates through one selection flow; does not fetch a preview for a scratch-only plan; converts the canonical load query to connector import options; bounds previews without changing the requested load limit

#### `tests/frontend/unit/components/filterFormat.test.ts`

- **行数**：26；**it 数**：4

- **describe**：formatFilterChipLabel

- **用例标题**：uses label-value language for equality and membership; uses a compact range instead of query syntax; spells out operators whose meaning matters; keeps familiar comparison symbols

#### `tests/frontend/unit/data/coerceDate.test.ts`

- **行数**：97；**it 数**：13

- **describe**：coerceDate, coerceDateTime, coerceTime, coerceDuration

- **用例标题**：should return null for null input; should return null for undefined input; should return null for empty string; should convert Date object to date-only ISO string; should pass through string date values unchanged; should pass through numeric timestamps unchanged; should return null for null/undefined/empty; should convert Date object to full ISO string; should pass through ISO datetime strings unchanged; should return null for null/undefined/empty; should pass through time strings unchanged; should return null for null/undefined/empty; should pass through duration values unchanged

#### `tests/frontend/unit/data/resolveExcelCellValue.test.ts`

- **行数**：107；**it 数**：16

- **describe**：resolveExcelCellValue

- **用例标题**：should return null for null; should return null for undefined; should return string as-is; should return number as-is; should return boolean as-is; should return empty string as-is; should convert Date to ISO string; should join richText segments; should handle richText with missing text fields; should extract text from hyperlink object; should fall back to hyperlink URL when text is empty; should resolve formula result (primitive); should resolve formula result (Date); should return null for formula with undefined result; should return null for error cell value; should stringify unknown objects

#### `tests/frontend/unit/data/typeInference.test.ts`

- **行数**：213；**it 数**：34

- **describe**：testDate (strict YYYY-MM-DD), testDateTime, testTime, testDuration, inferTypeFromValueArray, mapApiTypeToAppType

- **用例标题**：should accept ISO date strings; should reject datetime strings (should be DateTime); should reject pure year numbers; should reject time-only strings; should reject non-string values; should handle whitespace trimming; should accept ISO datetime strings; should accept Date objects; should reject date-only strings; should reject time-only strings; should reject non-date strings; should accept time strings; should accept time with timezone; should reject date strings; should reject datetime strings; should reject non-strings; should accept ISO 8601 duration strings; should reject bare ; should reject non-duration strings; should infer Boolean for boolean values; should infer Integer for whole numbers; should infer Number for decimal numbers; should infer Date for YYYY-MM-DD values; should infer DateTime for YYYY-MM-DDTHH:mm:ss values; should infer Time for HH:mm:ss values

#### `tests/frontend/unit/dataOperations/models.test.ts`

- **行数**：92；**it 数**：6

- **describe**：parseDataOperation

- **用例标题**：maps the versioned wire contract to a detached view model; parses optional discovery canvas presentation; rejects an unsupported schema version; rejects a selected plan outside the operation; rejects malformed plan hashes; parses partial results and structured step failures

#### `tests/frontend/unit/views/ClarificationPanel.test.tsx`

- **行数**：300；**it 数**：8

- **describe**：ClarificationPanel

- **用例标题**：selects an operation plan without submitting until Continue is clicked; offers only the loading options, leaving other requests to the chat input; submits a single-choice question immediately when an option is clicked; records partial selections via onSelectAnswer without submitting; renders an inline input under a free-text question and submits it tagged to that question; lets a single-choice question take a typed answer instead of a chip; supersedes a selected option when the user types a custom answer; records a typed answer live and submits it on Enter

#### `tests/frontend/unit/views/DataFrameTable.test.tsx`

- **行数**：44；**it 数**：3

- **describe**：DataFrameTable

- **用例标题**：renders column headers; adds dotted underline to headers with descriptions; does not set native title when columnDescriptions is provided for that col

#### `tests/frontend/unit/views/DataLoadingChat.test.tsx`

- **行数**：130；**it 数**：1

- **describe**：DataLoadingChat canvas

- **用例标题**：opens Python-produced tables in the unified right-side load plan

#### `tests/frontend/unit/views/DataSourceSidebar.test.tsx`

- **行数**：198；**it 数**：3

- **describe**：DataSourceSidebar

- **用例标题**：leaves loading state when catalog fetch fails; returns to the landing state without creating an empty workspace; shows recently modified sessions first and can switch to creation order

#### `tests/frontend/unit/views/SessionDistill.test.tsx`

- **行数**：154；**it 数**：8

- **describe**：buildSessionWorkflowContext, findSessionWorkflow

- **用例标题**：returns null when no threads are supplied; packs every supplied thread into the payload, in order; counts steps as create_table events across all threads; drops tool-call events when over the byte budget; drops oldest threads when payload still exceeds budget after lighter trimming; returns the entry whose sourceWorkspaceId matches; returns undefined when nothing matches; returns undefined when workspaceId is empty

#### `tests/frontend/unit/views/experienceContext.test.ts`

- **行数**：318；**it 数**：14

- **describe**：isLeafDerivedTable, buildLeafEvents, buildDistillModelConfig

- **用例标题**：returns true for a derived table with no children; returns false for a derived table that has children; returns false for non-derived tables; returns null for non-derived table; returns null when no user-originated message exists in chain; builds a flat event timeline from a single-step chain; emits one create_table per derived step in a multi-step chain; skips the deleted middle table (chain re-parented); emits create_chart paired with create_table when the step has a chart; drops error-role and empty interaction entries; sends raw code (not a code shape summary); includes columns, row_count, and sample_rows on create_table; preserves global model identity so the backend can resolve server credentials; preserves user model api_base for custom endpoints

#### `tests/frontend/unit/views/formatCellValue.test.ts`

- **行数**：105；**it 数**：16

- **describe**：formatCellValue

- **用例标题**：should return empty string for null/undefined; should format numbers with locale separators; should not add separators for non-measure numeric semantics; should keep separators for measure numeric semantics; should format booleans as strings; should pass through plain strings; should work without dataType parameter; should format Date type with locale date; should handle invalid date gracefully; should format DateTime type with locale datetime; should handle invalid datetime gracefully; should format Time type with locale time; should handle invalid time gracefully; should format Duration from milliseconds; should not over-format sub-second Duration values; should pass through non-numeric Duration as string

#### `tests/frontend/unit/views/safeCellRender.test.tsx`

- **行数**：96；**it 数**：13

- **describe**：safeCellRender – inline pattern (ReactTable / SelectableDataGrid), formatFn – DataLoadingThread format callback

- **用例标题**：should render string values directly; should render number values directly; should render null without crashing; should render undefined without crashing; should render boolean as string; should safely render a Date object as string; should safely render a plain object as string; should safely render an array as string; should pass through string values; should pass through number values; should pass through null; should convert Date object to string; should convert arbitrary object to string


## 3. 关键依赖与运行时关系

- **LLM**：`agents/client_utils.py` 的 Client 封装 LiteLLM，所有 Agent 共用。
- **表计算**：pandas + DuckDB（parquet 视图）。
- **可视化**：flint-chart 生成 Vega-Lite，vega 渲染。
- **存储**：本地 FS / Azure Blob / ephemeral。
- **认证**：oidc-client-ts（浏览器）与 PyJWT（服务器）。

## 4. 构建产物

- `py-src/data_formulator/dist`：前端生产资源（构建后）。
- `dist/*.whl`：Python 发行包。
- 桌面：packaging 流程产出 exe/app。

## 5. 如何做影响分析

1. 用本清单定位符号。
2. 搜索同名测试。
3. 检查 dev-guide 是否覆盖该跨切。
4. 若改协议，同时改前端 apiClient 测试与错误契约测试。

## 6. 文件清单之后的维护

新增源文件时：更新本 SOURCE 的对应小节（至少路径、行数、功能一句话）、补测试、遵守包管理约定。
删除文件时同步删除测试与文档引用，避免死链。

以下重复强调规模数字，便于引用：后端 Python 约四万六千行，前端 TS 约五万行，测试合计三万行以上。
这是一份需要持续投资测试与文档的代码库，而不是一次性 demo。


## 7. 分目录源码导读（详细）

### `py-src/data_formulator` 导读

该目录包含 12 个 Python 文件，合计 5006 行。应用入口、Flask 装配、错误类型、连接器总控、模型注册、工作区工厂阅读顺序建议从 `__init__.py` 的导出开始，再进入最核心的类文件。目录内的循环依赖应当避免：例如 routes 可以依赖 datalake，但 datalake 不应反向依赖 Flask 路由。如果必须用到请求上下文，应通过 workspace_factory 等门面，而不是 import routes。

`__init__.py` 以 12 行实现其职责。其中类 （无类） 构成主要抽象。维护者应保持函数短小，把协议细节（JSON 形状）集中在 to_dict/from_dict。性能敏感路径（目录列举、大表 sample）应避免把整表 to_dict 进内存后在 Python 层过滤，优先 Arrow/DuckDB 下推。

`__main__.py` 以 4 行实现其职责。其中类 （无类） 构成主要抽象。维护者应保持函数短小，把协议细节（JSON 形状）集中在 to_dict/from_dict。性能敏感路径（目录列举、大表 sample）应避免把整表 to_dict 进内存后在 Python 层过滤，优先 Arrow/DuckDB 下推。

`_startup_spinner.py` 以 79 行实现其职责。其中类 （无类） 构成主要抽象。维护者应保持函数短小，把协议细节（JSON 形状）集中在 to_dict/from_dict。性能敏感路径（目录列举、大表 sample）应避免把整表 to_dict 进内存后在 Python 层过滤，优先 Arrow/DuckDB 下推。

`agent_config.py` 以 152 行实现其职责。其中类 （无类） 构成主要抽象。维护者应保持函数短小，把协议细节（JSON 形状）集中在 to_dict/from_dict。性能敏感路径（目录列举、大表 sample）应避免把整表 to_dict 进内存后在 Python 层过滤，优先 Arrow/DuckDB 下推。

`app.py` 以 560 行实现其职责。其中类 ['CustomJSONEncoder'] 构成主要抽象。维护者应保持函数短小，把协议细节（JSON 形状）集中在 to_dict/from_dict。性能敏感路径（目录列举、大表 sample）应避免把整表 to_dict 进内存后在 Python 层过滤，优先 Arrow/DuckDB 下推。

`data_connector.py` 以 2720 行实现其职责。其中类 ['DataConnector', 'SourceSpec'] 构成主要抽象。维护者应保持函数短小，把协议细节（JSON 形状）集中在 to_dict/from_dict。性能敏感路径（目录列举、大表 sample）应避免把整表 to_dict 进内存后在 Python 层过滤，优先 Arrow/DuckDB 下推。

`desktop.py` 以 297 行实现其职责。其中类 （无类） 构成主要抽象。维护者应保持函数短小，把协议细节（JSON 形状）集中在 to_dict/from_dict。性能敏感路径（目录列举、大表 sample）应避免把整表 to_dict 进内存后在 Python 层过滤，优先 Arrow/DuckDB 下推。

`error_handler.py` 以 350 行实现其职责。其中类 （无类） 构成主要抽象。维护者应保持函数短小，把协议细节（JSON 形状）集中在 to_dict/from_dict。性能敏感路径（目录列举、大表 sample）应避免把整表 to_dict 进内存后在 Python 层过滤，优先 Arrow/DuckDB 下推。

`errors.py` 以 149 行实现其职责。其中类 ['ErrorCode', 'AppError'] 构成主要抽象。维护者应保持函数短小，把协议细节（JSON 形状）集中在 to_dict/from_dict。性能敏感路径（目录列举、大表 sample）应避免把整表 to_dict 进内存后在 Python 层过滤，优先 Arrow/DuckDB 下推。

`example_datasets_config.py` 以 420 行实现其职责。其中类 （无类） 构成主要抽象。维护者应保持函数短小，把协议细节（JSON 形状）集中在 to_dict/from_dict。性能敏感路径（目录列举、大表 sample）应避免把整表 to_dict 进内存后在 Python 层过滤，优先 Arrow/DuckDB 下推。

`model_registry.py` 以 114 行实现其职责。其中类 ['ModelRegistry'] 构成主要抽象。维护者应保持函数短小，把协议细节（JSON 形状）集中在 to_dict/from_dict。性能敏感路径（目录列举、大表 sample）应避免把整表 to_dict 进内存后在 Python 层过滤，优先 Arrow/DuckDB 下推。

`workspace_factory.py` 以 149 行实现其职责。其中类 （无类） 构成主要抽象。维护者应保持函数短小，把协议细节（JSON 形状）集中在 to_dict/from_dict。性能敏感路径（目录列举、大表 sample）应避免把整表 to_dict 进内存后在 Python 层过滤，优先 Arrow/DuckDB 下推。

### `py-src/data_formulator/agents` 导读

该目录包含 18 个 Python 文件，合计 7542 行。专用 Agent、LLM Client、上下文拼装、语义类型、推理日志、网页抓取阅读顺序建议从 `__init__.py` 的导出开始，再进入最核心的类文件。目录内的循环依赖应当避免：例如 routes 可以依赖 datalake，但 datalake 不应反向依赖 Flask 路由。如果必须用到请求上下文，应通过 workspace_factory 等门面，而不是 import routes。

`__init__.py` 以 14 行实现其职责。其中类 （无类） 构成主要抽象。维护者应保持函数短小，把协议细节（JSON 形状）集中在 to_dict/from_dict。性能敏感路径（目录列举、大表 sample）应避免把整表 to_dict 进内存后在 Python 层过滤，优先 Arrow/DuckDB 下推。

`agent_chart_restyle.py` 以 362 行实现其职责。其中类 ['ChartRestyleAgent'] 构成主要抽象。维护者应保持函数短小，把协议细节（JSON 形状）集中在 to_dict/from_dict。性能敏感路径（目录列举、大表 sample）应避免把整表 to_dict 进内存后在 Python 层过滤，优先 Arrow/DuckDB 下推。

`agent_code_explanation.py` 以 222 行实现其职责。其中类 ['CodeExplanationAgent'] 构成主要抽象。维护者应保持函数短小，把协议细节（JSON 形状）集中在 to_dict/from_dict。性能敏感路径（目录列举、大表 sample）应避免把整表 to_dict 进内存后在 Python 层过滤，优先 Arrow/DuckDB 下推。

`agent_data_load.py` 以 237 行实现其职责。其中类 ['DataLoadAgent'] 构成主要抽象。维护者应保持函数短小，把协议细节（JSON 形状）集中在 to_dict/from_dict。性能敏感路径（目录列举、大表 sample）应避免把整表 to_dict 进内存后在 Python 层过滤，优先 Arrow/DuckDB 下推。

`agent_data_loading_chat.py` 以 2378 行实现其职责。其中类 ['DataLoadingAgent'] 构成主要抽象。维护者应保持函数短小，把协议细节（JSON 形状）集中在 to_dict/from_dict。性能敏感路径（目录列举、大表 sample）应避免把整表 to_dict 进内存后在 Python 层过滤，优先 Arrow/DuckDB 下推。

`agent_diagnostics.py` 以 147 行实现其职责。其中类 ['AgentDiagnostics'] 构成主要抽象。维护者应保持函数短小，把协议细节（JSON 形状）集中在 to_dict/from_dict。性能敏感路径（目录列举、大表 sample）应避免把整表 to_dict 进内存后在 Python 层过滤，优先 Arrow/DuckDB 下推。

`agent_language.py` 以 188 行实现其职责。其中类 （无类） 构成主要抽象。维护者应保持函数短小，把协议细节（JSON 形状）集中在 to_dict/from_dict。性能敏感路径（目录列举、大表 sample）应避免把整表 to_dict 进内存后在 Python 层过滤，优先 Arrow/DuckDB 下推。

`agent_simple.py` 以 238 行实现其职责。其中类 ['SimpleAgents'] 构成主要抽象。维护者应保持函数短小，把协议细节（JSON 形状）集中在 to_dict/from_dict。性能敏感路径（目录列举、大表 sample）应避免把整表 to_dict 进内存后在 Python 层过滤，优先 Arrow/DuckDB 下推。

`agent_sort_data.py` 以 125 行实现其职责。其中类 ['SortDataAgent'] 构成主要抽象。维护者应保持函数短小，把协议细节（JSON 形状）集中在 to_dict/from_dict。性能敏感路径（目录列举、大表 sample）应避免把整表 to_dict 进内存后在 Python 层过滤，优先 Arrow/DuckDB 下推。

`agent_starter_questions.py` 以 120 行实现其职责。其中类 ['StarterQuestionsAgent'] 构成主要抽象。维护者应保持函数短小，把协议细节（JSON 形状）集中在 to_dict/from_dict。性能敏感路径（目录列举、大表 sample）应避免把整表 to_dict 进内存后在 Python 层过滤，优先 Arrow/DuckDB 下推。

`agent_utils.py` 以 794 行实现其职责。其中类 （无类） 构成主要抽象。维护者应保持函数短小，把协议细节（JSON 形状）集中在 to_dict/from_dict。性能敏感路径（目录列举、大表 sample）应避免把整表 to_dict 进内存后在 Python 层过滤，优先 Arrow/DuckDB 下推。

`agent_utils_sql.py` 以 39 行实现其职责。其中类 （无类） 构成主要抽象。维护者应保持函数短小，把协议细节（JSON 形状）集中在 to_dict/from_dict。性能敏感路径（目录列举、大表 sample）应避免把整表 to_dict 进内存后在 Python 层过滤，优先 Arrow/DuckDB 下推。

`agent_workflow_distill.py` 以 510 行实现其职责。其中类 ['WorkflowDistillAgent'] 构成主要抽象。维护者应保持函数短小，把协议细节（JSON 形状）集中在 to_dict/from_dict。性能敏感路径（目录列举、大表 sample）应避免把整表 to_dict 进内存后在 Python 层过滤，优先 Arrow/DuckDB 下推。

`client_utils.py` 以 452 行实现其职责。其中类 ['Client'] 构成主要抽象。维护者应保持函数短小，把协议细节（JSON 形状）集中在 to_dict/from_dict。性能敏感路径（目录列举、大表 sample）应避免把整表 to_dict 进内存后在 Python 层过滤，优先 Arrow/DuckDB 下推。

`context.py` 以 459 行实现其职责。其中类 （无类） 构成主要抽象。维护者应保持函数短小，把协议细节（JSON 形状）集中在 to_dict/from_dict。性能敏感路径（目录列举、大表 sample）应避免把整表 to_dict 进内存后在 Python 层过滤，优先 Arrow/DuckDB 下推。

`reasoning_log.py` 以 264 行实现其职责。其中类 ['ReasoningLogger', '_NullReasoningLogger'] 构成主要抽象。维护者应保持函数短小，把协议细节（JSON 形状）集中在 to_dict/from_dict。性能敏感路径（目录列举、大表 sample）应避免把整表 to_dict 进内存后在 Python 层过滤，优先 Arrow/DuckDB 下推。

`semantic_types.py` 以 463 行实现其职责。其中类 （无类） 构成主要抽象。维护者应保持函数短小，把协议细节（JSON 形状）集中在 to_dict/from_dict。性能敏感路径（目录列举、大表 sample）应避免把整表 to_dict 进内存后在 Python 层过滤，优先 Arrow/DuckDB 下推。

`web_utils.py` 以 530 行实现其职责。其中类 （无类） 构成主要抽象。维护者应保持函数短小，把协议细节（JSON 形状）集中在 to_dict/from_dict。性能敏感路径（目录列举、大表 sample）应避免把整表 to_dict 进内存后在 Python 层过滤，优先 Arrow/DuckDB 下推。

### `py-src/data_formulator/analyst` 导读

该目录包含 3 个 Python 文件，合计 2359 行。统一 AnalystAgent 外壳与工具工厂阅读顺序建议从 `__init__.py` 的导出开始，再进入最核心的类文件。目录内的循环依赖应当避免：例如 routes 可以依赖 datalake，但 datalake 不应反向依赖 Flask 路由。如果必须用到请求上下文，应通过 workspace_factory 等门面，而不是 import routes。

`__init__.py` 以 50 行实现其职责。其中类 （无类） 构成主要抽象。维护者应保持函数短小，把协议细节（JSON 形状）集中在 to_dict/from_dict。性能敏感路径（目录列举、大表 sample）应避免把整表 to_dict 进内存后在 Python 层过滤，优先 Arrow/DuckDB 下推。

`agent.py` 以 2157 行实现其职责。其中类 ['_StreamingArgExtractor', 'AnalystAgent'] 构成主要抽象。维护者应保持函数短小，把协议细节（JSON 形状）集中在 to_dict/from_dict。性能敏感路径（目录列举、大表 sample）应避免把整表 to_dict 进内存后在 Python 层过滤，优先 Arrow/DuckDB 下推。

`tools.py` 以 152 行实现其职责。其中类 （无类） 构成主要抽象。维护者应保持函数短小，把协议细节（JSON 形状）集中在 to_dict/from_dict。性能敏感路径（目录列举、大表 sample）应避免把整表 to_dict 进内存后在 Python 层过滤，优先 Arrow/DuckDB 下推。

### `py-src/data_formulator/analyst/skills` 导读

该目录包含 2 个 Python 文件，合计 580 行。技能注册表与协议类型阅读顺序建议从 `__init__.py` 的导出开始，再进入最核心的类文件。目录内的循环依赖应当避免：例如 routes 可以依赖 datalake，但 datalake 不应反向依赖 Flask 路由。如果必须用到请求上下文，应通过 workspace_factory 等门面，而不是 import routes。

`__init__.py` 以 394 行实现其职责。其中类 ['SkillRegistry'] 构成主要抽象。维护者应保持函数短小，把协议细节（JSON 形状）集中在 to_dict/from_dict。性能敏感路径（目录列举、大表 sample）应避免把整表 to_dict 进内存后在 Python 层过滤，优先 Arrow/DuckDB 下推。

`base.py` 以 186 行实现其职责。其中类 ['SkillMeta', 'SkillContext', 'ToolResult', 'Skill'] 构成主要抽象。维护者应保持函数短小，把协议细节（JSON 形状）集中在 to_dict/from_dict。性能敏感路径（目录列举、大表 sample）应避免把整表 to_dict 进内存后在 Python 层过滤，优先 Arrow/DuckDB 下推。

### `py-src/data_formulator/analyst/skills/core` 导读

该目录包含 2 个 Python 文件，合计 355 行。核心技能：探查工具与 visualize / ask_user阅读顺序建议从 `__init__.py` 的导出开始，再进入最核心的类文件。目录内的循环依赖应当避免：例如 routes 可以依赖 datalake，但 datalake 不应反向依赖 Flask 路由。如果必须用到请求上下文，应通过 workspace_factory 等门面，而不是 import routes。

`__init__.py` 以 9 行实现其职责。其中类 （无类） 构成主要抽象。维护者应保持函数短小，把协议细节（JSON 形状）集中在 to_dict/from_dict。性能敏感路径（目录列举、大表 sample）应避免把整表 to_dict 进内存后在 Python 层过滤，优先 Arrow/DuckDB 下推。

`skill.py` 以 346 行实现其职责。其中类 ['CoreSkill'] 构成主要抽象。维护者应保持函数短小，把协议细节（JSON 形状）集中在 to_dict/from_dict。性能敏感路径（目录列举、大表 sample）应避免把整表 to_dict 进内存后在 Python 层过滤，优先 Arrow/DuckDB 下推。

### `py-src/data_formulator/analyst/skills/data-loading` 导读

该目录包含 2 个 Python 文件，合计 363 行。数据加载技能：发现工具与不可变加载计划阅读顺序建议从 `__init__.py` 的导出开始，再进入最核心的类文件。目录内的循环依赖应当避免：例如 routes 可以依赖 datalake，但 datalake 不应反向依赖 Flask 路由。如果必须用到请求上下文，应通过 workspace_factory 等门面，而不是 import routes。

`__init__.py` 以 1 行实现其职责。其中类 （无类） 构成主要抽象。维护者应保持函数短小，把协议细节（JSON 形状）集中在 to_dict/from_dict。性能敏感路径（目录列举、大表 sample）应避免把整表 to_dict 进内存后在 Python 层过滤，优先 Arrow/DuckDB 下推。

`skill.py` 以 362 行实现其职责。其中类 ['DataLoadingSkill'] 构成主要抽象。维护者应保持函数短小，把协议细节（JSON 形状）集中在 to_dict/from_dict。性能敏感路径（目录列举、大表 sample）应避免把整表 to_dict 进内存后在 Python 层过滤，优先 Arrow/DuckDB 下推。

### `py-src/data_formulator/analyst/skills/data_loading` 导读

该目录包含 1 个 Python 文件，合计 362 行。数据加载技能的兼容目录副本阅读顺序建议从 `__init__.py` 的导出开始，再进入最核心的类文件。目录内的循环依赖应当避免：例如 routes 可以依赖 datalake，但 datalake 不应反向依赖 Flask 路由。如果必须用到请求上下文，应通过 workspace_factory 等门面，而不是 import routes。

`skill.py` 以 362 行实现其职责。其中类 ['DataLoadingSkill'] 构成主要抽象。维护者应保持函数短小，把协议细节（JSON 形状）集中在 to_dict/from_dict。性能敏感路径（目录列举、大表 sample）应避免把整表 to_dict 进内存后在 Python 层过滤，优先 Arrow/DuckDB 下推。

### `py-src/data_formulator/analyst/skills/report` 导读

该目录包含 2 个 Python 文件，合计 221 行。报告技能：inspect_chart 与 write_report 流式写作阅读顺序建议从 `__init__.py` 的导出开始，再进入最核心的类文件。目录内的循环依赖应当避免：例如 routes 可以依赖 datalake，但 datalake 不应反向依赖 Flask 路由。如果必须用到请求上下文，应通过 workspace_factory 等门面，而不是 import routes。

`__init__.py` 以 9 行实现其职责。其中类 （无类） 构成主要抽象。维护者应保持函数短小，把协议细节（JSON 形状）集中在 to_dict/from_dict。性能敏感路径（目录列举、大表 sample）应避免把整表 to_dict 进内存后在 Python 层过滤，优先 Arrow/DuckDB 下推。

`skill.py` 以 212 行实现其职责。其中类 ['ReportWritingSkill'] 构成主要抽象。维护者应保持函数短小，把协议细节（JSON 形状）集中在 to_dict/from_dict。性能敏感路径（目录列举、大表 sample）应避免把整表 to_dict 进内存后在 Python 层过滤，优先 Arrow/DuckDB 下推。

### `py-src/data_formulator/auth` 导读

该目录包含 4 个 Python 文件，合计 681 行。身份解析、TokenStore、Azure CLI阅读顺序建议从 `__init__.py` 的导出开始，再进入最核心的类文件。目录内的循环依赖应当避免：例如 routes 可以依赖 datalake，但 datalake 不应反向依赖 Flask 路由。如果必须用到请求上下文，应通过 workspace_factory 等门面，而不是 import routes。

`__init__.py` 以 3 行实现其职责。其中类 （无类） 构成主要抽象。维护者应保持函数短小，把协议细节（JSON 形状）集中在 to_dict/from_dict。性能敏感路径（目录列举、大表 sample）应避免把整表 to_dict 进内存后在 Python 层过滤，优先 Arrow/DuckDB 下推。

`azure_cli.py` 以 40 行实现其职责。其中类 （无类） 构成主要抽象。维护者应保持函数短小，把协议细节（JSON 形状）集中在 to_dict/from_dict。性能敏感路径（目录列举、大表 sample）应避免把整表 to_dict 进内存后在 Python 层过滤，优先 Arrow/DuckDB 下推。

`identity.py` 以 248 行实现其职责。其中类 （无类） 构成主要抽象。维护者应保持函数短小，把协议细节（JSON 形状）集中在 to_dict/from_dict。性能敏感路径（目录列举、大表 sample）应避免把整表 to_dict 进内存后在 Python 层过滤，优先 Arrow/DuckDB 下推。

`token_store.py` 以 390 行实现其职责。其中类 ['TokenStore'] 构成主要抽象。维护者应保持函数短小，把协议细节（JSON 形状）集中在 to_dict/from_dict。性能敏感路径（目录列举、大表 sample）应避免把整表 to_dict 进内存后在 Python 层过滤，优先 Arrow/DuckDB 下推。

### `py-src/data_formulator/auth/gateways` 导读

该目录包含 4 个 Python 文件，合计 659 行。OAuth/OIDC/Kusto 登录回调蓝图阅读顺序建议从 `__init__.py` 的导出开始，再进入最核心的类文件。目录内的循环依赖应当避免：例如 routes 可以依赖 datalake，但 datalake 不应反向依赖 Flask 路由。如果必须用到请求上下文，应通过 workspace_factory 等门面，而不是 import routes。

`__init__.py` 以 3 行实现其职责。其中类 （无类） 构成主要抽象。维护者应保持函数短小，把协议细节（JSON 形状）集中在 to_dict/from_dict。性能敏感路径（目录列举、大表 sample）应避免把整表 to_dict 进内存后在 Python 层过滤，优先 Arrow/DuckDB 下推。

`github_gateway.py` 以 146 行实现其职责。其中类 （无类） 构成主要抽象。维护者应保持函数短小，把协议细节（JSON 形状）集中在 to_dict/from_dict。性能敏感路径（目录列举、大表 sample）应避免把整表 to_dict 进内存后在 Python 层过滤，优先 Arrow/DuckDB 下推。

`kusto_oauth_gateway.py` 以 252 行实现其职责。其中类 （无类） 构成主要抽象。维护者应保持函数短小，把协议细节（JSON 形状）集中在 to_dict/from_dict。性能敏感路径（目录列举、大表 sample）应避免把整表 to_dict 进内存后在 Python 层过滤，优先 Arrow/DuckDB 下推。

`oidc_gateway.py` 以 258 行实现其职责。其中类 （无类） 构成主要抽象。维护者应保持函数短小，把协议细节（JSON 形状）集中在 to_dict/from_dict。性能敏感路径（目录列举、大表 sample）应避免把整表 to_dict 进内存后在 Python 层过滤，优先 Arrow/DuckDB 下推。

### `py-src/data_formulator/auth/providers` 导读

该目录包含 5 个 Python 文件，合计 682 行。OIDC / GitHub / Azure EasyAuth 提供者阅读顺序建议从 `__init__.py` 的导出开始，再进入最核心的类文件。目录内的循环依赖应当避免：例如 routes 可以依赖 datalake，但 datalake 不应反向依赖 Flask 路由。如果必须用到请求上下文，应通过 workspace_factory 等门面，而不是 import routes。

`__init__.py` 以 77 行实现其职责。其中类 （无类） 构成主要抽象。维护者应保持函数短小，把协议细节（JSON 形状）集中在 to_dict/from_dict。性能敏感路径（目录列举、大表 sample）应避免把整表 to_dict 进内存后在 Python 层过滤，优先 Arrow/DuckDB 下推。

`azure_easyauth.py` 以 53 行实现其职责。其中类 ['AzureEasyAuthProvider'] 构成主要抽象。维护者应保持函数短小，把协议细节（JSON 形状）集中在 to_dict/from_dict。性能敏感路径（目录列举、大表 sample）应避免把整表 to_dict 进内存后在 Python 层过滤，优先 Arrow/DuckDB 下推。

`base.py` 以 87 行实现其职责。其中类 ['AuthResult', 'AuthProvider', 'AuthenticationError'] 构成主要抽象。维护者应保持函数短小，把协议细节（JSON 形状）集中在 to_dict/from_dict。性能敏感路径（目录列举、大表 sample）应避免把整表 to_dict 进内存后在 Python 层过滤，优先 Arrow/DuckDB 下推。

`github_oauth.py` 以 70 行实现其职责。其中类 ['GitHubOAuthProvider'] 构成主要抽象。维护者应保持函数短小，把协议细节（JSON 形状）集中在 to_dict/from_dict。性能敏感路径（目录列举、大表 sample）应避免把整表 to_dict 进内存后在 Python 层过滤，优先 Arrow/DuckDB 下推。

`oidc.py` 以 395 行实现其职责。其中类 ['OIDCProvider'] 构成主要抽象。维护者应保持函数短小，把协议细节（JSON 形状）集中在 to_dict/from_dict。性能敏感路径（目录列举、大表 sample）应避免把整表 to_dict 进内存后在 Python 层过滤，优先 Arrow/DuckDB 下推。

### `py-src/data_formulator/auth/vault` 导读

该目录包含 3 个 Python 文件，合计 246 行。本地加密凭证库阅读顺序建议从 `__init__.py` 的导出开始，再进入最核心的类文件。目录内的循环依赖应当避免：例如 routes 可以依赖 datalake，但 datalake 不应反向依赖 Flask 路由。如果必须用到请求上下文，应通过 workspace_factory 等门面，而不是 import routes。

`__init__.py` 以 111 行实现其职责。其中类 （无类） 构成主要抽象。维护者应保持函数短小，把协议细节（JSON 形状）集中在 to_dict/from_dict。性能敏感路径（目录列举、大表 sample）应避免把整表 to_dict 进内存后在 Python 层过滤，优先 Arrow/DuckDB 下推。

`base.py` 以 40 行实现其职责。其中类 ['CredentialVault'] 构成主要抽象。维护者应保持函数短小，把协议细节（JSON 形状）集中在 to_dict/from_dict。性能敏感路径（目录列举、大表 sample）应避免把整表 to_dict 进内存后在 Python 层过滤，优先 Arrow/DuckDB 下推。

`local_vault.py` 以 95 行实现其职责。其中类 ['LocalCredentialVault'] 构成主要抽象。维护者应保持函数短小，把协议细节（JSON 形状）集中在 to_dict/from_dict。性能敏感路径（目录列举、大表 sample）应避免把整表 to_dict 进内存后在 Python 层过滤，优先 Arrow/DuckDB 下推。

### `py-src/data_formulator/data_loader` 导读

该目录包含 21 个 Python 文件，合计 10886 行。外部数据加载器实现与插件扫描阅读顺序建议从 `__init__.py` 的导出开始，再进入最核心的类文件。目录内的循环依赖应当避免：例如 routes 可以依赖 datalake，但 datalake 不应反向依赖 Flask 路由。如果必须用到请求上下文，应通过 workspace_factory 等门面，而不是 import routes。

`__init__.py` 以 365 行实现其职责。其中类 （无类） 构成主要抽象。维护者应保持函数短小，把协议细节（JSON 形状）集中在 to_dict/from_dict。性能敏感路径（目录列举、大表 sample）应避免把整表 to_dict 进内存后在 Python 层过滤，优先 Arrow/DuckDB 下推。

`athena_data_loader.py` 以 567 行实现其职责。其中类 ['AthenaDataLoader'] 构成主要抽象。维护者应保持函数短小，把协议细节（JSON 形状）集中在 to_dict/from_dict。性能敏感路径（目录列举、大表 sample）应避免把整表 to_dict 进内存后在 Python 层过滤，优先 Arrow/DuckDB 下推。

`azure_blob_data_loader.py` 以 402 行实现其职责。其中类 ['AzureBlobDataLoader'] 构成主要抽象。维护者应保持函数短小，把协议细节（JSON 形状）集中在 to_dict/from_dict。性能敏感路径（目录列举、大表 sample）应避免把整表 to_dict 进内存后在 Python 层过滤，优先 Arrow/DuckDB 下推。

`bigquery_data_loader.py` 以 348 行实现其职责。其中类 ['BigQueryDataLoader'] 构成主要抽象。维护者应保持函数短小，把协议细节（JSON 形状）集中在 to_dict/from_dict。性能敏感路径（目录列举、大表 sample）应避免把整表 to_dict 进内存后在 Python 层过滤，优先 Arrow/DuckDB 下推。

`clickhouse_data_loader.py` 以 752 行实现其职责。其中类 ['ClickHouseDataLoader'] 构成主要抽象。维护者应保持函数短小，把协议细节（JSON 形状）集中在 to_dict/from_dict。性能敏感路径（目录列举、大表 sample）应避免把整表 to_dict 进内存后在 Python 层过滤，优先 Arrow/DuckDB 下推。

`connector_errors.py` 以 216 行实现其职责。其中类 ['ConnectorErrorInfo'] 构成主要抽象。维护者应保持函数短小，把协议细节（JSON 形状）集中在 to_dict/from_dict。性能敏感路径（目录列举、大表 sample）应避免把整表 to_dict 进内存后在 Python 层过滤，优先 Arrow/DuckDB 下推。

`cosmosdb_data_loader.py` 以 349 行实现其职责。其中类 ['CosmosDBDataLoader'] 构成主要抽象。维护者应保持函数短小，把协议细节（JSON 形状）集中在 to_dict/from_dict。性能敏感路径（目录列举、大表 sample）应避免把整表 to_dict 进内存后在 Python 层过滤，优先 Arrow/DuckDB 下推。

`databricks_data_loader.py` 以 347 行实现其职责。其中类 ['DatabricksDataLoader'] 构成主要抽象。维护者应保持函数短小，把协议细节（JSON 形状）集中在 to_dict/from_dict。性能敏感路径（目录列举、大表 sample）应避免把整表 to_dict 进内存后在 Python 层过滤，优先 Arrow/DuckDB 下推。

`external_data_loader.py` 以 1212 行实现其职责。其中类 ['CatalogCachePolicy', 'ConnectorParamError', 'CatalogNode', 'ExternalDataLoader'] 构成主要抽象。维护者应保持函数短小，把协议细节（JSON 形状）集中在 to_dict/from_dict。性能敏感路径（目录列举、大表 sample）应避免把整表 to_dict 进内存后在 Python 层过滤，优先 Arrow/DuckDB 下推。

`kusto_data_loader.py` 以 867 行实现其职责。其中类 ['_KustoDelegatedCredential', 'KustoDataLoader'] 构成主要抽象。维护者应保持函数短小，把协议细节（JSON 形状）集中在 to_dict/from_dict。性能敏感路径（目录列举、大表 sample）应避免把整表 to_dict 进内存后在 Python 层过滤，优先 Arrow/DuckDB 下推。

`local_folder_data_loader.py` 以 342 行实现其职责。其中类 ['LocalFolderDataLoader'] 构成主要抽象。维护者应保持函数短小，把协议细节（JSON 形状）集中在 to_dict/from_dict。性能敏感路径（目录列举、大表 sample）应避免把整表 to_dict 进内存后在 Python 层过滤，优先 Arrow/DuckDB 下推。

`mongodb_data_loader.py` 以 519 行实现其职责。其中类 ['MongoDBDataLoader'] 构成主要抽象。维护者应保持函数短小，把协议细节（JSON 形状）集中在 to_dict/from_dict。性能敏感路径（目录列举、大表 sample）应避免把整表 to_dict 进内存后在 Python 层过滤，优先 Arrow/DuckDB 下推。

`mssql_data_loader.py` 以 784 行实现其职责。其中类 ['MSSQLDataLoader'] 构成主要抽象。维护者应保持函数短小，把协议细节（JSON 形状）集中在 to_dict/from_dict。性能敏感路径（目录列举、大表 sample）应避免把整表 to_dict 进内存后在 Python 层过滤，优先 Arrow/DuckDB 下推。

`mysql_data_loader.py` 以 524 行实现其职责。其中类 ['MySQLDataLoader'] 构成主要抽象。维护者应保持函数短小，把协议细节（JSON 形状）集中在 to_dict/from_dict。性能敏感路径（目录列举、大表 sample）应避免把整表 to_dict 进内存后在 Python 层过滤，优先 Arrow/DuckDB 下推。

`postgresql_data_loader.py` 以 860 行实现其职责。其中类 ['PostgreSQLDataLoader'] 构成主要抽象。维护者应保持函数短小，把协议细节（JSON 形状）集中在 to_dict/from_dict。性能敏感路径（目录列举、大表 sample）应避免把整表 to_dict 进内存后在 Python 层过滤，优先 Arrow/DuckDB 下推。

`probe_utils.py` 以 429 行实现其职责。其中类 ['SqlDialect'] 构成主要抽象。维护者应保持函数短小，把协议细节（JSON 形状）集中在 to_dict/from_dict。性能敏感路径（目录列举、大表 sample）应避免把整表 to_dict 进内存后在 Python 层过滤，优先 Arrow/DuckDB 下推。

`s3_data_loader.py` 以 308 行实现其职责。其中类 ['S3DataLoader'] 构成主要抽象。维护者应保持函数短小，把协议细节（JSON 形状）集中在 to_dict/from_dict。性能敏感路径（目录列举、大表 sample）应避免把整表 to_dict 进内存后在 Python 层过滤，优先 Arrow/DuckDB 下推。

`sample_datasets_loader.py` 以 312 行实现其职责。其中类 ['SampleDatasetsLoader'] 构成主要抽象。维护者应保持函数短小，把协议细节（JSON 形状）集中在 to_dict/from_dict。性能敏感路径（目录列举、大表 sample）应避免把整表 to_dict 进内存后在 Python 层过滤，优先 Arrow/DuckDB 下推。

`superset_auth_bridge.py` 以 89 行实现其职责。其中类 ['SupersetAuthBridge'] 构成主要抽象。维护者应保持函数短小，把协议细节（JSON 形状）集中在 to_dict/from_dict。性能敏感路径（目录列举、大表 sample）应避免把整表 to_dict 进内存后在 Python 层过滤，优先 Arrow/DuckDB 下推。

`superset_client.py` 以 182 行实现其职责。其中类 ['SupersetClient'] 构成主要抽象。维护者应保持函数短小，把协议细节（JSON 形状）集中在 to_dict/from_dict。性能敏感路径（目录列举、大表 sample）应避免把整表 to_dict 进内存后在 Python 层过滤，优先 Arrow/DuckDB 下推。

`superset_data_loader.py` 以 1112 行实现其职责。其中类 ['SupersetLoader'] 构成主要抽象。维护者应保持函数短小，把协议细节（JSON 形状）集中在 to_dict/from_dict。性能敏感路径（目录列举、大表 sample）应避免把整表 to_dict 进内存后在 Python 层过滤，优先 Arrow/DuckDB 下推。

### `py-src/data_formulator/data_loader/guides` 导读

该目录包含 1 个 Python 文件，合计 1 行。阅读顺序建议从 `__init__.py` 的导出开始，再进入最核心的类文件。目录内的循环依赖应当避免：例如 routes 可以依赖 datalake，但 datalake 不应反向依赖 Flask 路由。如果必须用到请求上下文，应通过 workspace_factory 等门面，而不是 import routes。

`__init__.py` 以 1 行实现其职责。其中类 （无类） 构成主要抽象。维护者应保持函数短小，把协议细节（JSON 形状）集中在 to_dict/from_dict。性能敏感路径（目录列举、大表 sample）应避免把整表 to_dict 进内存后在 Python 层过滤，优先 Arrow/DuckDB 下推。

### `py-src/data_formulator/data_operations` 导读

该目录包含 6 个 Python 文件，合计 1333 行。结构化加载计划模型、仓库、执行器阅读顺序建议从 `__init__.py` 的导出开始，再进入最核心的类文件。目录内的循环依赖应当避免：例如 routes 可以依赖 datalake，但 datalake 不应反向依赖 Flask 路由。如果必须用到请求上下文，应通过 workspace_factory 等门面，而不是 import routes。

`__init__.py` 以 46 行实现其职责。其中类 （无类） 构成主要抽象。维护者应保持函数短小，把协议细节（JSON 形状）集中在 to_dict/from_dict。性能敏感路径（目录列举、大表 sample）应避免把整表 to_dict 进内存后在 Python 层过滤，优先 Arrow/DuckDB 下推。

`actions.py` 以 12 行实现其职责。其中类 （无类） 构成主要抽象。维护者应保持函数短小，把协议细节（JSON 形状）集中在 to_dict/from_dict。性能敏感路径（目录列举、大表 sample）应避免把整表 to_dict 进内存后在 Python 层过滤，优先 Arrow/DuckDB 下推。

`discovery.py` 以 373 行实现其职责。其中类 ['ProbeBudget', 'ProbeGuidance', 'DataDiscoveryService'] 构成主要抽象。维护者应保持函数短小，把协议细节（JSON 形状）集中在 to_dict/from_dict。性能敏感路径（目录列举、大表 sample）应避免把整表 to_dict 进内存后在 Python 层过滤，优先 Arrow/DuckDB 下推。

`executor.py` 以 196 行实现其职责。其中类 ['DataOperationExecutionResult', 'DataOperationExecutor'] 构成主要抽象。维护者应保持函数短小，把协议细节（JSON 形状）集中在 to_dict/from_dict。性能敏感路径（目录列举、大表 sample）应避免把整表 to_dict 进内存后在 Python 层过滤，优先 Arrow/DuckDB 下推。

`models.py` 以 411 行实现其职责。其中类 ['DataOperationStatus', 'OperationFilter', 'LoadQueryOrder', 'LoadQuery', 'ConnectorQueryStep', 'DataOperationPlan', 'OperationError', 'FailedOperationStep', 'DataOperation'] 构成主要抽象。维护者应保持函数短小，把协议细节（JSON 形状）集中在 to_dict/from_dict。性能敏感路径（目录列举、大表 sample）应避免把整表 to_dict 进内存后在 Python 层过滤，优先 Arrow/DuckDB 下推。

`repository.py` 以 295 行实现其职责。其中类 ['DataOperationConflictError', 'StoredDataOperation', 'DataOperationRepository'] 构成主要抽象。维护者应保持函数短小，把协议细节（JSON 形状）集中在 to_dict/from_dict。性能敏感路径（目录列举、大表 sample）应避免把整表 to_dict 进内存后在 Python 层过滤，优先 Arrow/DuckDB 下推。

### `py-src/data_formulator/datalake` 导读

该目录包含 14 个 Python 文件，合计 5608 行。工作区、Parquet、目录缓存、命名与元数据阅读顺序建议从 `__init__.py` 的导出开始，再进入最核心的类文件。目录内的循环依赖应当避免：例如 routes 可以依赖 datalake，但 datalake 不应反向依赖 Flask 路由。如果必须用到请求上下文，应通过 workspace_factory 等门面，而不是 import routes。

`__init__.py` 以 129 行实现其职责。其中类 （无类） 构成主要抽象。维护者应保持函数短小，把协议细节（JSON 形状）集中在 to_dict/from_dict。性能敏感路径（目录列举、大表 sample）应避免把整表 to_dict 进内存后在 Python 层过滤，优先 Arrow/DuckDB 下推。

`azure_blob_workspace.py` 以 777 行实现其职责。其中类 ['AzureBlobWorkspace'] 构成主要抽象。维护者应保持函数短小，把协议细节（JSON 形状）集中在 to_dict/from_dict。性能敏感路径（目录列举、大表 sample）应避免把整表 to_dict 进内存后在 Python 层过滤，优先 Arrow/DuckDB 下推。

`azure_blob_workspace_manager.py` 以 348 行实现其职责。其中类 ['AzureBlobWorkspaceManager'] 构成主要抽象。维护者应保持函数短小，把协议细节（JSON 形状）集中在 to_dict/from_dict。性能敏感路径（目录列举、大表 sample）应避免把整表 to_dict 进内存后在 Python 层过滤，优先 Arrow/DuckDB 下推。

`blob_disk_cache.py` 以 239 行实现其职责。其中类 ['CacheEntry', 'BlobDiskCache'] 构成主要抽象。维护者应保持函数短小，把协议细节（JSON 形状）集中在 to_dict/from_dict。性能敏感路径（目录列举、大表 sample）应避免把整表 to_dict 进内存后在 Python 层过滤，优先 Arrow/DuckDB 下推。

`catalog_cache.py` 以 716 行实现其职责。其中类 ['CatalogSnapshot', 'CatalogSearchError'] 构成主要抽象。维护者应保持函数短小，把协议细节（JSON 形状）集中在 to_dict/from_dict。性能敏感路径（目录列举、大表 sample）应避免把整表 to_dict 进内存后在 Python 层过滤，优先 Arrow/DuckDB 下推。

`catalog_refresh.py` 以 159 行实现其职责。其中类 （无类） 构成主要抽象。维护者应保持函数短小，把协议细节（JSON 形状）集中在 to_dict/from_dict。性能敏感路径（目录列举、大表 sample）应避免把整表 to_dict 进内存后在 Python 层过滤，优先 Arrow/DuckDB 下推。

`ephemeral_workspace.py` 以 184 行实现其职责。其中类 ['EphemeralWorkspaceManager'] 构成主要抽象。维护者应保持函数短小，把协议细节（JSON 形状）集中在 to_dict/from_dict。性能敏感路径（目录列举、大表 sample）应避免把整表 to_dict 进内存后在 Python 层过滤，优先 Arrow/DuckDB 下推。

`file_manager.py` 以 367 行实现其职责。其中类 （无类） 构成主要抽象。维护者应保持函数短小，把协议细节（JSON 形状）集中在 to_dict/from_dict。性能敏感路径（目录列举、大表 sample）应避免把整表 to_dict 进内存后在 Python 层过滤，优先 Arrow/DuckDB 下推。

`naming.py` 以 54 行实现其职责。其中类 （无类） 构成主要抽象。维护者应保持函数短小，把协议细节（JSON 形状）集中在 to_dict/from_dict。性能敏感路径（目录列举、大表 sample）应避免把整表 to_dict 进内存后在 Python 层过滤，优先 Arrow/DuckDB 下推。

`parquet_utils.py` 以 257 行实现其职责。其中类 （无类） 构成主要抽象。维护者应保持函数短小，把协议细节（JSON 形状）集中在 to_dict/from_dict。性能敏感路径（目录列举、大表 sample）应避免把整表 to_dict 进内存后在 Python 层过滤，优先 Arrow/DuckDB 下推。

`table_names.py` 以 188 行实现其职责。其中类 （无类） 构成主要抽象。维护者应保持函数短小，把协议细节（JSON 形状）集中在 to_dict/from_dict。性能敏感路径（目录列举、大表 sample）应避免把整表 to_dict 进内存后在 Python 层过滤，优先 Arrow/DuckDB 下推。

`workspace.py` 以 1038 行实现其职责。其中类 ['Workspace', 'WorkspaceWithTempData'] 构成主要抽象。维护者应保持函数短小，把协议细节（JSON 形状）集中在 to_dict/from_dict。性能敏感路径（目录列举、大表 sample）应避免把整表 to_dict 进内存后在 Python 层过滤，优先 Arrow/DuckDB 下推。

`workspace_manager.py` 以 521 行实现其职责。其中类 ['WorkspaceManager'] 构成主要抽象。维护者应保持函数短小，把协议细节（JSON 形状）集中在 to_dict/from_dict。性能敏感路径（目录列举、大表 sample）应避免把整表 to_dict 进内存后在 Python 层过滤，优先 Arrow/DuckDB 下推。

`workspace_metadata.py` 以 631 行实现其职责。其中类 ['WorkspaceLock', 'ColumnInfo', 'TableMetadata', 'WorkspaceMetadata', 'ImportedFrom', 'Derivation'] 构成主要抽象。维护者应保持函数短小，把协议细节（JSON 形状）集中在 to_dict/from_dict。性能敏感路径（目录列举、大表 sample）应避免把整表 to_dict 进内存后在 Python 层过滤，优先 Arrow/DuckDB 下推。

### `py-src/data_formulator/knowledge` 导读

该目录包含 2 个 Python 文件，合计 730 行。规则 / 工作流 / data-memory 存储阅读顺序建议从 `__init__.py` 的导出开始，再进入最核心的类文件。目录内的循环依赖应当避免：例如 routes 可以依赖 datalake，但 datalake 不应反向依赖 Flask 路由。如果必须用到请求上下文，应通过 workspace_factory 等门面，而不是 import routes。

`__init__.py` 以 3 行实现其职责。其中类 （无类） 构成主要抽象。维护者应保持函数短小，把协议细节（JSON 形状）集中在 to_dict/from_dict。性能敏感路径（目录列举、大表 sample）应避免把整表 to_dict 进内存后在 Python 层过滤，优先 Arrow/DuckDB 下推。

`store.py` 以 727 行实现其职责。其中类 ['KnowledgeItemMeta', 'KnowledgeStore'] 构成主要抽象。维护者应保持函数短小，把协议细节（JSON 形状）集中在 to_dict/from_dict。性能敏感路径（目录列举、大表 sample）应避免把整表 to_dict 进内存后在 Python 层过滤，优先 Arrow/DuckDB 下推。

### `py-src/data_formulator/routes` 导读

该目录包含 9 个 Python 文件，合计 4587 行。HTTP 蓝图：表、Agent、会话、知识、凭证、日志阅读顺序建议从 `__init__.py` 的导出开始，再进入最核心的类文件。目录内的循环依赖应当避免：例如 routes 可以依赖 datalake，但 datalake 不应反向依赖 Flask 路由。如果必须用到请求上下文，应通过 workspace_factory 等门面，而不是 import routes。

`__init__.py` 以 3 行实现其职责。其中类 （无类） 构成主要抽象。维护者应保持函数短小，把协议细节（JSON 形状）集中在 to_dict/from_dict。性能敏感路径（目录列举、大表 sample）应避免把整表 to_dict 进内存后在 Python 层过滤，优先 Arrow/DuckDB 下推。

`agents.py` 以 1096 行实现其职责。其中类 （无类） 构成主要抽象。维护者应保持函数短小，把协议细节（JSON 形状）集中在 to_dict/from_dict。性能敏感路径（目录列举、大表 sample）应避免把整表 to_dict 进内存后在 Python 层过滤，优先 Arrow/DuckDB 下推。

`credentials.py` 以 74 行实现其职责。其中类 （无类） 构成主要抽象。维护者应保持函数短小，把协议细节（JSON 形状）集中在 to_dict/from_dict。性能敏感路径（目录列举、大表 sample）应避免把整表 to_dict 进内存后在 Python 层过滤，优先 Arrow/DuckDB 下推。

`demo_stream.py` 以 1169 行实现其职责。其中类 （无类） 构成主要抽象。维护者应保持函数短小，把协议细节（JSON 形状）集中在 to_dict/from_dict。性能敏感路径（目录列举、大表 sample）应避免把整表 to_dict 进内存后在 Python 层过滤，优先 Arrow/DuckDB 下推。

`knowledge.py` 以 393 行实现其职责。其中类 （无类） 构成主要抽象。维护者应保持函数短小，把协议细节（JSON 形状）集中在 to_dict/from_dict。性能敏感路径（目录列举、大表 sample）应避免把整表 to_dict 进内存后在 Python 层过滤，优先 Arrow/DuckDB 下推。

`logs.py` 以 117 行实现其职责。其中类 （无类） 构成主要抽象。维护者应保持函数短小，把协议细节（JSON 形状）集中在 to_dict/from_dict。性能敏感路径（目录列举、大表 sample）应避免把整表 to_dict 进内存后在 Python 层过滤，优先 Arrow/DuckDB 下推。

`model_endpoints.py` 以 94 行实现其职责。其中类 （无类） 构成主要抽象。维护者应保持函数短小，把协议细节（JSON 形状）集中在 to_dict/from_dict。性能敏感路径（目录列举、大表 sample）应避免把整表 to_dict 进内存后在 Python 层过滤，优先 Arrow/DuckDB 下推。

`sessions.py` 以 374 行实现其职责。其中类 （无类） 构成主要抽象。维护者应保持函数短小，把协议细节（JSON 形状）集中在 to_dict/from_dict。性能敏感路径（目录列举、大表 sample）应避免把整表 to_dict 进内存后在 Python 层过滤，优先 Arrow/DuckDB 下推。

`tables.py` 以 1267 行实现其职责。其中类 （无类） 构成主要抽象。维护者应保持函数短小，把协议细节（JSON 形状）集中在 to_dict/from_dict。性能敏感路径（目录列举、大表 sample）应避免把整表 to_dict 进内存后在 Python 层过滤，优先 Arrow/DuckDB 下推。

### `py-src/data_formulator/sandbox` 导读

该目录包含 5 个 Python 文件，合计 1027 行。代码隔离执行：local / docker / not_a_sandbox阅读顺序建议从 `__init__.py` 的导出开始，再进入最核心的类文件。目录内的循环依赖应当避免：例如 routes 可以依赖 datalake，但 datalake 不应反向依赖 Flask 路由。如果必须用到请求上下文，应通过 workspace_factory 等门面，而不是 import routes。

`__init__.py` 以 22 行实现其职责。其中类 （无类） 构成主要抽象。维护者应保持函数短小，把协议细节（JSON 形状）集中在 to_dict/from_dict。性能敏感路径（目录列举、大表 sample）应避免把整表 to_dict 进内存后在 Python 层过滤，优先 Arrow/DuckDB 下推。

`base.py` 以 55 行实现其职责。其中类 ['Sandbox'] 构成主要抽象。维护者应保持函数短小，把协议细节（JSON 形状）集中在 to_dict/from_dict。性能敏感路径（目录列举、大表 sample）应避免把整表 to_dict 进内存后在 Python 层过滤，优先 Arrow/DuckDB 下推。

`docker_sandbox.py` 以 263 行实现其职责。其中类 ['DockerSandbox'] 构成主要抽象。维护者应保持函数短小，把协议细节（JSON 形状）集中在 to_dict/from_dict。性能敏感路径（目录列举、大表 sample）应避免把整表 to_dict 进内存后在 Python 层过滤，优先 Arrow/DuckDB 下推。

`local_sandbox.py` 以 608 行实现其职责。其中类 ['_WarmWorkerPool', 'SandboxSession', 'LocalSandbox'] 构成主要抽象。维护者应保持函数短小，把协议细节（JSON 形状）集中在 to_dict/from_dict。性能敏感路径（目录列举、大表 sample）应避免把整表 to_dict 进内存后在 Python 层过滤，优先 Arrow/DuckDB 下推。

`not_a_sandbox.py` 以 79 行实现其职责。其中类 ['NotASandbox'] 构成主要抽象。维护者应保持函数短小，把协议细节（JSON 形状）集中在 to_dict/from_dict。性能敏感路径（目录列举、大表 sample）应避免把整表 to_dict 进内存后在 Python 层过滤，优先 Arrow/DuckDB 下推。

### `py-src/data_formulator/security` 导读

该目录包含 6 个 Python 文件，合计 834 行。路径监禁、日志脱敏、代码签名、URL 白名单阅读顺序建议从 `__init__.py` 的导出开始，再进入最核心的类文件。目录内的循环依赖应当避免：例如 routes 可以依赖 datalake，但 datalake 不应反向依赖 Flask 路由。如果必须用到请求上下文，应通过 workspace_factory 等门面，而不是 import routes。

`__init__.py` 以 3 行实现其职责。其中类 （无类） 构成主要抽象。维护者应保持函数短小，把协议细节（JSON 形状）集中在 to_dict/from_dict。性能敏感路径（目录列举、大表 sample）应避免把整表 to_dict 进内存后在 Python 层过滤，优先 Arrow/DuckDB 下推。

`code_signing.py` 以 141 行实现其职责。其中类 （无类） 构成主要抽象。维护者应保持函数短小，把协议细节（JSON 形状）集中在 to_dict/from_dict。性能敏感路径（目录列举、大表 sample）应避免把整表 to_dict 进内存后在 Python 层过滤，优先 Arrow/DuckDB 下推。

`log_sanitizer.py` 以 240 行实现其职责。其中类 ['SensitiveDataFilter'] 构成主要抽象。维护者应保持函数短小，把协议细节（JSON 形状）集中在 to_dict/from_dict。性能敏感路径（目录列举、大表 sample）应避免把整表 to_dict 进内存后在 Python 层过滤，优先 Arrow/DuckDB 下推。

`path_safety.py` 以 136 行实现其职责。其中类 ['ConfinedDir'] 构成主要抽象。维护者应保持函数短小，把协议细节（JSON 形状）集中在 to_dict/from_dict。性能敏感路径（目录列举、大表 sample）应避免把整表 to_dict 进内存后在 Python 层过滤，优先 Arrow/DuckDB 下推。

`sanitize.py` 以 210 行实现其职责。其中类 （无类） 构成主要抽象。维护者应保持函数短小，把协议细节（JSON 形状）集中在 to_dict/from_dict。性能敏感路径（目录列举、大表 sample）应避免把整表 to_dict 进内存后在 Python 层过滤，优先 Arrow/DuckDB 下推。

`url_allowlist.py` 以 104 行实现其职责。其中类 （无类） 构成主要抽象。维护者应保持函数短小，把协议细节（JSON 形状）集中在 to_dict/from_dict。性能敏感路径（目录列举、大表 sample）应避免把整表 to_dict 进内存后在 Python 层过滤，优先 Arrow/DuckDB 下推。

### `py-src/data_formulator/workflows` 导读

该目录包含 3 个 Python 文件，合计 2619 行。语义类型到 Vega-Lite 图表的组装算法阅读顺序建议从 `__init__.py` 的导出开始，再进入最核心的类文件。目录内的循环依赖应当避免：例如 routes 可以依赖 datalake，但 datalake 不应反向依赖 Flask 路由。如果必须用到请求上下文，应通过 workspace_factory 等门面，而不是 import routes。

`__init__.py` 以 1 行实现其职责。其中类 （无类） 构成主要抽象。维护者应保持函数短小，把协议细节（JSON 形状）集中在 to_dict/from_dict。性能敏感路径（目录列举、大表 sample）应避免把整表 to_dict 进内存后在 Python 层过滤，优先 Arrow/DuckDB 下推。

`chart_semantics.py` 以 600 行实现其职责。其中类 ['TypeRegistryEntry', 'ChannelSemantics'] 构成主要抽象。维护者应保持函数短小，把协议细节（JSON 形状）集中在 to_dict/from_dict。性能敏感路径（目录列举、大表 sample）应避免把整表 to_dict 进内存后在 Python 层过滤，优先 Arrow/DuckDB 下推。

`create_vl_plots.py` 以 2018 行实现其职责。其中类 （无类） 构成主要抽象。维护者应保持函数短小，把协议细节（JSON 形状）集中在 to_dict/from_dict。性能敏感路径（目录列举、大表 sample）应避免把整表 to_dict 进内存后在 Python 层过滤，优先 Arrow/DuckDB 下推。


## 8. 前端分层导读

src/app 是神经系统：store、slice、thunk、api 客户端、身份。src/views 是器官：每个视图文件对应一块用户可指着说出名字的界面。src/components 是组织：无路由意识的可复用块。src/data 是细胞：列式表与类型推断，尽量纯函数。src/i18n 是语言中枢：所有文案的唯一来源。改 bug 时先判断缺陷在哪一层，避免在 view 里复制 slice 已有的派生数据。

### `src`

共 5 个文件，444 行。该层的接口应保持稳定，向内重构自由。若导出被测试直接 import，更改签名必须改测试。组件文件若超过约 800 行，考虑按子组件拆分，但不要为拆而拆导致 props 钻透过深。

- `icons.tsx`（286 行）：导出 19 项。将其视为独立编译单元，避免从 views 反向 import 深层私有函数。

- `index.css`（91 行）：导出 0 项。将其视为独立编译单元，避免从 views 反向 import 深层私有函数。

- `index.tsx`（29 行）：导出 0 项。将其视为独立编译单元，避免从 views 反向 import 深层私有函数。

- `mui.d.ts`（8 行）：导出 0 项。将其视为独立编译单元，避免从 views 反向 import 深层私有函数。

- `types.d.ts`（30 行）：导出 0 项。将其视为独立编译单元，避免从 views 反向 import 深层私有函数。

### `src/api`

共 1 个文件，192 行。该层的接口应保持稳定，向内重构自由。若导出被测试直接 import，更改签名必须改测试。组件文件若超过约 800 行，考虑按子组件拆分，但不要为拆而拆导致 props 钻透过深。

- `knowledgeApi.ts`（192 行）：导出 16 项。将其视为独立编译单元，避免从 views 反向 import 深层私有函数。

### `src/app`

共 35 个文件，10891 行。该层的接口应保持稳定，向内重构自由。若导出被测试直接 import，更改签名必须改测试。组件文件若超过约 800 行，考虑按子组件拆分，但不要为拆而拆导致 props 钻透过深。

- `App.tsx`（1779 行）：导出 3 项。将其视为独立编译单元，避免从 views 反向 import 深层私有函数。

- `AuthButton.tsx`（173 行）：导出 1 项。将其视为独立编译单元，避免从 views 反向 import 深层私有函数。

- `IdentityMigrationDialog.tsx`（156 行）：导出 2 项。将其视为独立编译单元，避免从 views 反向 import 深层私有函数。

- `LayoutProvider.tsx`（234 行）：导出 6 项。将其视为独立编译单元，避免从 views 反向 import 深层私有函数。

- `OidcCallback.tsx`（119 行）：导出 1 项。将其视为独立编译单元，避免从 views 反向 import 深层私有函数。

- `agentInteractionPolicy.ts`（7 行）：导出 1 项。将其视为独立编译单元，避免从 views 反向 import 深层私有函数。

- `apiClient.ts`（317 行）：导出 7 项。将其视为独立编译单元，避免从 views 反向 import 深层私有函数。

- `chartCache.ts`（136 行）：导出 8 项。将其视为独立编译单元，避免从 views 反向 import 深层私有函数。

- `chartRecommendation.ts`（113 行）：导出 2 项。将其视为独立编译单元，避免从 views 反向 import 深层私有函数。

- `clarification.ts`（119 行）：导出 3 项。将其视为独立编译单元，避免从 views 反向 import 深层私有函数。

- `connectorFormPersistence.ts`（16 行）：导出 1 项。将其视为独立编译单元，避免从 views 反向 import 深层私有函数。

- `connectorNames.ts`（39 行）：导出 1 项。将其视为独立编译单元，避免从 views 反向 import 深层私有函数。

- `dfSlice.tsx`（2761 行）：导出 22 项。将其视为独立编译单元，避免从 views 反向 import 深层私有函数。

- `displayRowsCache.ts`（44 行）：导出 3 项。将其视为独立编译单元，避免从 views 反向 import 深层私有函数。

- `errorCodes.ts`（72 行）：导出 2 项。将其视为独立编译单元，避免从 views 反向 import 深层私有函数。

- `errorHandler.ts`（123 行）：导出 3 项。将其视为独立编译单元，避免从 views 反向 import 深层私有函数。

- `identity.ts`（121 行）：导出 8 项。将其视为独立编译单元，避免从 views 反向 import 深层私有函数。

- `inputTablePreviewCache.ts`（49 行）：导出 6 项。将其视为独立编译单元，避免从 views 反向 import 深层私有函数。

- `intentClassifier.ts`（60 行）：导出 2 项。将其视为独立编译单元，避免从 views 反向 import 深层私有函数。

- `layout.ts`（542 行）：导出 42 项。将其视为独立编译单元，避免从 views 反向 import 深层私有函数。

- `loadableState.ts`（48 行）：导出 7 项。将其视为独立编译单元，避免从 views 反向 import 深层私有函数。

- `oidcConfig.ts`（227 行）：导出 11 项。将其视为独立编译单元，避免从 views 反向 import 深层私有函数。

- `restyle.ts`（327 行）：导出 8 项。将其视为独立编译单元，避免从 views 反向 import 深层私有函数。

- `stateMigrations.ts`（357 行）：导出 2 项。将其视为独立编译单元，避免从 views 反向 import 深层私有函数。

- `store.ts`（54 行）：导出 3 项。将其视为独立编译单元，避免从 views 反向 import 深层私有函数。

- `tableResolution.ts`（61 行）：导出 5 项。将其视为独立编译单元，避免从 views 反向 import 深层私有函数。

- `tableThunks.ts`（390 行）：导出 6 项。将其视为独立编译单元，避免从 views 反向 import 深层私有函数。

- `tokens.ts`（269 行）：导出 16 项。将其视为独立编译单元，避免从 views 反向 import 深层私有函数。

- `useAutoSave.tsx`（121 行）：导出 2 项。将其视为独立编译单元，避免从 views 反向 import 深层私有函数。

- `useDataRefresh.tsx`（653 行）：导出 2 项。将其视为独立编译单元，避免从 views 反向 import 深层私有函数。

- `useKnowledgeStore.ts`（201 行）：导出 2 项。将其视为独立编译单元，避免从 views 反向 import 深层私有函数。

- `useWorkspaceAutoName.tsx`（98 行）：导出 2 项。将其视为独立编译单元，避免从 views 反向 import 深层私有函数。

- `utils.tsx`（678 行）：导出 17 项。将其视为独立编译单元，避免从 views 反向 import 深层私有函数。

- `workspaceDB.ts`（129 行）：导出 3 项。将其视为独立编译单元，避免从 views 反向 import 深层私有函数。

- `workspaceService.ts`（298 行）：导出 13 项。将其视为独立编译单元，避免从 views 反向 import 深层私有函数。

### `src/components`

共 17 个文件，4155 行。该层的接口应保持稳定，向内重构自由。若导出被测试直接 import，更改签名必须改测试。组件文件若超过约 800 行，考虑按子组件拆分，但不要为拆而拆导致 props 钻透过深。

- `AnvilLoader.tsx`（116 行）：导出 2 项。将其视为独立编译单元，避免从 views 反向 import 深层私有函数。

- `CatalogTree.tsx`（245 行）：导出 9 项。将其视为独立编译单元，避免从 views 反向 import 深层私有函数。

- `ChartTemplates.tsx`（157 行）：导出 6 项。将其视为独立编译单元，避免从 views 反向 import 深层私有函数。

- `ComponentType.tsx`（687 行）：导出 56 项。将其视为独立编译单元，避免从 views 反向 import 深层私有函数。

- `ConnectorFormCard.tsx`（393 行）：导出 1 项。将其视为独立编译单元，避免从 views 反向 import 深层私有函数。

- `ConnectorTablePreview.tsx`（738 行）：导出 8 项。将其视为独立编译单元，避免从 views 反向 import 深层私有函数。

- `DataOperationCard.tsx`（99 行）：导出 1 项。将其视为独立编译单元，避免从 views 反向 import 深层私有函数。

- `DndTypes.ts`（17 行）：导出 2 项。将其视为独立编译单元，避免从 views 反向 import 深层私有函数。

- `FunComponents.tsx`（78 行）：导出 4 项。将其视为独立编译单元，避免从 views 反向 import 深层私有函数。

- `LoadPlanCard.tsx`（495 行）：导出 3 项。将其视为独立编译单元，避免从 views 反向 import 深层私有函数。

- `MarkdownEditor.tsx`（100 行）：导出 1 项。将其视为独立编译单元，避免从 views 反向 import 深层私有函数。

- `ResizeHandle.tsx`（115 行）：导出 2 项。将其视为独立编译单元，避免从 views 反向 import 深层私有函数。

- `RotatingTextBlock.tsx`（49 行）：导出 1 项。将其视为独立编译单元，避免从 views 反向 import 深层私有函数。

- `ScrollFade.tsx`（114 行）：导出 4 项。将其视为独立编译单元，避免从 views 反向 import 深层私有函数。

- `TablePreviewRow.tsx`（152 行）：导出 3 项。将其视为独立编译单元，避免从 views 反向 import 深层私有函数。

- `VirtualizedCatalogTree.tsx`（531 行）：导出 2 项。将其视为独立编译单元，避免从 views 反向 import 深层私有函数。

- `filterFormat.ts`（69 行）：导出 3 项。将其视为独立编译单元，避免从 views 反向 import 深层私有函数。

### `src/data`

共 4 个文件，594 行。该层的接口应保持稳定，向内重构自由。若导出被测试直接 import，更改签名必须改测试。组件文件若超过约 800 行，考虑按子组件拆分，但不要为拆而拆导致 props 钻透过深。

- `column.ts`（37 行）：导出 1 项。将其视为独立编译单元，避免从 views 反向 import 深层私有函数。

- `table.ts`（73 行）：导出 1 项。将其视为独立编译单元，避免从 views 反向 import 深层私有函数。

- `types.ts`（156 行）：导出 11 项。将其视为独立编译单元，避免从 views 反向 import 深层私有函数。

- `utils.ts`（328 行）：导出 14 项。将其视为独立编译单元，避免从 views 反向 import 深层私有函数。

### `src/dataOperations`

共 1 个文件，230 行。该层的接口应保持稳定，向内重构自由。若导出被测试直接 import，更改签名必须改测试。组件文件若超过约 800 行，考虑按子组件拆分，但不要为拆而拆导致 props 钻透过深。

- `models.ts`（230 行）：导出 12 项。将其视为独立编译单元，避免从 views 反向 import 深层私有函数。

### `src/i18n`

共 2 个文件，83 行。该层的接口应保持稳定，向内重构自由。若导出被测试直接 import，更改签名必须改测试。组件文件若超过约 800 行，考虑按子组件拆分，但不要为拆而拆导致 props 钻透过深。

- `index.ts`（33 行）：导出 0 项。将其视为独立编译单元，避免从 views 反向 import 深层私有函数。

- `vega-locale.ts`（50 行）：导出 1 项。将其视为独立编译单元，避免从 views 反向 import 深层私有函数。

### `src/i18n/locales`

共 1 个文件，8 行。该层的接口应保持稳定，向内重构自由。若导出被测试直接 import，更改签名必须改测试。组件文件若超过约 800 行，考虑按子组件拆分，但不要为拆而拆导致 props 钻透过深。

- `index.ts`（8 行）：导出 2 项。将其视为独立编译单元，避免从 views 反向 import 深层私有函数。

### `src/i18n/locales/en`

共 1 个文件，27 行。该层的接口应保持稳定，向内重构自由。若导出被测试直接 import，更改签名必须改测试。组件文件若超过约 800 行，考虑按子组件拆分，但不要为拆而拆导致 props 钻透过深。

- `index.ts`（27 行）：导出 0 项。将其视为独立编译单元，避免从 views 反向 import 深层私有函数。

### `src/i18n/locales/zh`

共 1 个文件，27 行。该层的接口应保持稳定，向内重构自由。若导出被测试直接 import，更改签名必须改测试。组件文件若超过约 800 行，考虑按子组件拆分，但不要为拆而拆导致 props 钻透过深。

- `index.ts`（27 行）：导出 0 项。将其视为独立编译单元，避免从 views 反向 import 深层私有函数。

### `src/views`

共 48 个文件，35502 行。该层的接口应保持稳定，向内重构自由。若导出被测试直接 import，更改签名必须改测试。组件文件若超过约 800 行，考虑按子组件拆分，但不要为拆而拆导致 props 钻透过深。

- `About.tsx`（254 行）：导出 1 项。将其视为独立编译单元，避免从 views 反向 import 深层私有函数。

- `AgentChatInput.tsx`（601 行）：导出 2 项。将其视为独立编译单元，避免从 views 反向 import 深层私有函数。

- `AgentPausePanel.tsx`（673 行）：导出 2 项。将其视为独立编译单元，避免从 views 反向 import 深层私有函数。

- `AgentRulesDialog.tsx`（393 行）：导出 1 项。将其视为独立编译单元，避免从 views 反向 import 深层私有函数。

- `AgentToyIcon.tsx`（138 行）：导出 3 项。将其视为独立编译单元，避免从 views 反向 import 深层私有函数。

- `ChartQuickConfig.tsx`（417 行）：导出 2 项。将其视为独立编译单元，避免从 views 反向 import 深层私有函数。

- `ChartRenderService.tsx`（360 行）：导出 1 项。将其视为独立编译单元，避免从 views 反向 import 深层私有函数。

- `ChartUtils.tsx`（65 行）：导出 0 项。将其视为独立编译单元，避免从 views 反向 import 深层私有函数。

- `ChartVariantStrip.tsx`（636 行）：导出 2 项。将其视为独立编译单元，避免从 views 反向 import 深层私有函数。

- `ChartifactDialog.tsx`（308 行）：导出 2 项。将其视为独立编译单元，避免从 views 反向 import 深层私有函数。

- `ChatDialog.tsx`（450 行）：导出 4 项。将其视为独立编译单元，避免从 views 反向 import 深层私有函数。

- `ColumnFilterPopover.tsx`（632 行）：导出 5 项。将其视为独立编译单元，避免从 views 反向 import 深层私有函数。

- `DBTableManager.tsx`（1248 行）：导出 1 项。将其视为独立编译单元，避免从 views 反向 import 深层私有函数。

- `DataFormulator.tsx`（1200 行）：导出 1 项。将其视为独立编译单元，避免从 views 反向 import 深层私有函数。

- `DataFrameTable.tsx`（273 行）：导出 2 项。将其视为独立编译单元，避免从 views 反向 import 深层私有函数。

- `DataLoadingChat.tsx`（2013 行）：导出 1 项。将其视为独立编译单元，避免从 views 反向 import 深层私有函数。

- `DataSourceSidebar.tsx`（2544 行）：导出 1 项。将其视为独立编译单元，避免从 views 反向 import 深层私有函数。

- `DataThread.tsx`（3405 行）：导出 3 项。将其视为独立编译单元，避免从 views 反向 import 深层私有函数。

- `DataThreadCards.tsx`（384 行）：导出 1 项。将其视为独立编译单元，避免从 views 反向 import 深层私有函数。

- `DataView.tsx`（373 行）：导出 2 项。将其视为独立编译单元，避免从 views 反向 import 深层私有函数。

- `EncodingBox.tsx`（766 行）：导出 4 项。将其视为独立编译单元，避免从 views 反向 import 深层私有函数。

- `EncodingShelfCard.tsx`（1179 行）：导出 7 项。将其视为独立编译单元，避免从 views 反向 import 深层私有函数。

- `EncodingShelfThread.tsx`（103 行）：导出 2 项。将其视为独立编译单元，避免从 views 反向 import 深层私有函数。

- `ExampleSessions.tsx`（195 行）：导出 4 项。将其视为独立编译单元，避免从 views 反向 import 深层私有函数。

- `ExplComponents.tsx`（394 行）：导出 5 项。将其视为独立编译单元，避免从 views 反向 import 深层私有函数。

- `InteractionEntryCard.tsx`（756 行）：导出 11 项。将其视为独立编译单元，避免从 views 反向 import 深层私有函数。

- `KnowledgePanel.tsx`（644 行）：导出 1 项。将其视为独立编译单元，避免从 views 反向 import 深层私有函数。

- `LocalInstallUpgradePanel.tsx`（299 行）：导出 1 项。将其视为独立编译单元，避免从 views 反向 import 深层私有函数。

- `LogViewerDialog.tsx`（325 行）：导出 1 项。将其视为独立编译单元，避免从 views 反向 import 深层私有函数。

- `MessageSnackbar.tsx`（353 行）：导出 2 项。将其视为独立编译单元，避免从 views 反向 import 深层私有函数。

- `ModelSelectionDialog.tsx`（821 行）：导出 1 项。将其视为独立编译单元，避免从 views 反向 import 深层私有函数。

- `MultiTablePreview.tsx`（234 行）：导出 2 项。将其视为独立编译单元，避免从 views 反向 import 深层私有函数。

- `OperatorCard.tsx`（65 行）：导出 2 项。将其视为独立编译单元，避免从 views 反向 import 深层私有函数。

- `ReactTable.tsx`（212 行）：导出 2 项。将其视为独立编译单元，避免从 views 反向 import 深层私有函数。

- `RefreshDataDialog.tsx`（556 行）：导出 2 项。将其视为独立编译单元，避免从 views 反向 import 深层私有函数。

- `ReportView.tsx`（780 行）：导出 1 项。将其视为独立编译单元，避免从 views 反向 import 深层私有函数。

- `SelectableDataGrid.tsx`（849 行）：导出 2 项。将其视为独立编译单元，避免从 views 反向 import 深层私有函数。

- `SessionDistill.tsx`（734 行）：导出 7 项。将其视为独立编译单元，避免从 views 反向 import 深层私有函数。

- `SimpleChartRecBox.tsx`（2629 行）：导出 1 项。将其视为独立编译单元，避免从 views 反向 import 深层私有函数。

- `SourceTableShelf.tsx`（982 行）：导出 2 项。将其视为独立编译单元，避免从 views 反向 import 深层私有函数。

- `TestPanel.tsx`（89 行）：导出 3 项。将其视为独立编译单元，避免从 views 反向 import 深层私有函数。

- `TiptapReportEditor.tsx`（791 行）：导出 3 项。将其视为独立编译单元，避免从 views 反向 import 深层私有函数。

- `UnifiedDataUploadDialog.tsx`（2639 行）：导出 7 项。将其视为独立编译单元，避免从 views 反向 import 深层私有函数。

- `ViewUtils.tsx`（187 行）：导出 6 项。将其视为独立编译单元，避免从 views 反向 import 深层私有函数。

- `VisualizationView.tsx`（1968 行）：导出 7 项。将其视为独立编译单元，避免从 views 反向 import 深层私有函数。

- `dataLoadingSuggestions.ts`（210 行）：导出 6 项。将其视为独立编译单元，避免从 views 反向 import 深层私有函数。

- `threadLayout.ts`（56 行）：导出 7 项。将其视为独立编译单元，避免从 views 反向 import 深层私有函数。

- `workflowContext.ts`（319 行）：导出 7 项。将其视为独立编译单元，避免从 views 反向 import 深层私有函数。
