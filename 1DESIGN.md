# Data Formulator 设计说明书（1DESIGN）

> 文档性质：详细设计说明书（Detailed Design Specification）  

> 对应产品：Microsoft Data Formulator 0.8.0b1  

> 范围：整体架构、全部数据模型、功能、函数、算法与测试用例  

> 读者：架构师、后端/前端开发、测试、安全评审、Agent 平台工程师  

> 生成依据：仓库当前源码、开发指南与自动化测试清单的静态分析  


## 0. 文档导读与阅读地图

本说明书不是营销材料，而是一份面向实现者的设计基线。阅读顺序建议：

1. **第 1–2 章**：先建立系统边界与分层图，避免在细节里迷路。
2. **第 3 章**：掌握四条数据模型线（持久化 / 运行时 / Redux / 传输契约），后续所有 API 都围绕它们旋转。
3. **第 4 章**：按子系统阅读功能规格——Analyst、连接器、工作区、认证、沙箱、知识库、前端线程。
4. **第 5 章**：算法与协议。这些是“为什么这样实现”的核心，评审 PR 时优先对照。
5. **第 6 章起**：逐模块、逐类、逐函数的详细字典。可当 API 手册使用。
6. **测试专章**：每个测试文件的意图与覆盖面，用于判断改动是否需要补测。

设计上有几条不可妥协的不变量，后续章节反复出现：

- **业务错误 HTTP 200**：只有鉴权失败用 401/403；流式与非流式错误形状统一。
- **ConfinedDir 是唯一路径原语**：禁止 `Path(root) / user_input`。
- **日志必须脱敏**：`sanitize_params` / `SensitiveDataFilter` 双层防护。
- **前端用户可见字符串必须走 i18n**。
- **Agent 对用户可见的提交动作一次只能一个**（inspection 可并行）。
- **大表权威存储在服务端 Parquet**，浏览器只持有预览与派生样本。

违反上述不变量的改动，即使功能正确，也不应合并。

## 1. 产品定位、问题陈述与设计目标

### 1.1 问题陈述

数据分析工具长期分裂为两类：

- **BI 工具**（Power BI、Tableau、Superset）：治理好、图表强，但迭代一个新问题要建数据集、建度量、拖字段，探索成本高。
- **Notebook / SQL IDE**：灵活，但上下文在单元格和聊天里流失，难以把“探索过程”变成可分享的叙事。
- **纯 ChatGPT 式数据助手**：自然语言门槛低，但（a）连不上企业内部源；（b）没有可视化工作记忆；（c）一次答错很难分支对比。

Data Formulator 选择第三条路：**以可视化工作区为中心的 Agent 系统**。用户问的是分析问题，系统产出的是表、图、报告和可分支的 Data Thread，而不是一堵文本墙。

### 1.2 设计目标

| 编号 | 目标 | 实现落点 |
|------|------|----------|
| G1 | 任意数据可进入同一工作区 | ExternalDataLoader + FileManager + 截图/文本抽取 |
| G2 | Agent 在行动前必须看见结构 | inspect_source_data / catalog / probe / 加载计划确认 |
| G3 | 探索过程可分支、可回溯 | Data Thread + DraftNode + Trigger.interaction |
| G4 | 图表可编辑、可风格化 | Flint + EncodingShelf + ChartRestyleAgent |
| G5 | 结果可沉淀为报告与知识 | report skill + KnowledgeStore + WorkflowDistillAgent |
| G6 | 可在单机、演示、企业三种剖面部署 | local / ephemeral / azure_blob + AUTH_PROVIDER |
| G7 | 默认安全 | 沙箱、路径监禁、错误消毒、SSRF、代码签名 |
| G8 | 模型可替换 | LiteLLM + ModelRegistry + 自定义 endpoint 白名单 |

### 1.3 非目标

- 不是通用多智能体操作系统，不提供任意 MCP 服务器热插拔（当前技能是进程内 Python 包）。
- 不是企业级 BI 语义层替代品：度量治理仍在源系统（Superset/Databricks/Kusto）。
- 不是无限制代码执行平台：沙箱默认禁止写工作区以外的文件、禁止 subprocess。
- 不把密钥下发到浏览器：全局模型凭证留在服务端；连接器密码进 Vault。

### 1.4 版本演进对设计的影响

从 README 与代码注释可见一条清晰演进：

- v0.1：多表 join、数据集锚定。
- v0.2：DuckDB + 大数据。
- v0.2.1：外部 loader。
- v0.5：Agent 模式、抽取、报告。
- v0.6：URL/数据库实时刷新。
- v0.7：统一 DataAgent + Data Thread + Flint + 中英 UI + 持久工作区。
- **v0.8**：统一 AnalystAgent + Skill 架构 + 更多源 + 加载计划（DataOperation）。

因此 0.8 的架构中心不再是“多个专用 Agent 路由”，而是 **一个外壳 + 可加载技能**。遗留的 DataLoadingAgent（独立 NDJSON 对话）仍然存在，用于数据接入侧栏；分析主路径已合并。

## 2. 总体架构设计

### 2.1 逻辑视图

系统分为十层（表现、传输、应用、智能体、数据接入、数据操作、存储、执行、安全、认证/知识）。浏览器只负责意图表达与可视化，权威状态在 Workspace。Agent 不直接持有数据库连接字符串，而是通过 DataConnector 解析出的 live loader 访问源，通过 Workspace 访问已导入表。

### 2.2 进程与部署视图

| 进程 | 角色 |
|------|------|
| Flask 主进程 | HTTP、会话、连接器、Agent 编排 |
| LocalSandbox worker | 预热的 Python 子进程，执行 explore/visualize 代码 |
| Docker 容器（可选） | 一次性隔离执行 |
| Vite 开发服务器 | 仅开发态；生产前端编译进 `py-src/data_formulator/dist` |
| 桌面 WebView | `data_formulator_desktop` 单实例包装 |

Docker Compose 部署时 **不能** 再嵌套 `SANDBOX=docker`，因为子容器 bind mount 的是容器路径而非宿主机路径。这是架构级约束，不是文档脚注。

### 2.3 关键数据流

**加载数据**：UI 表单/聊天 → `/api/connectors/import-data` 或 `/api/tables/create-table` → loader.fetch_data_as_arrow → Workspace.write_parquet → TableMetadata → 前端 loadTable thunk → inputTables + preview cache。

**分析提问**：Data Thread 输入 → `/api/agent/analyst-streaming` → AnalystAgent.run → 沙箱 + visualize → NDJSON action/result/completion → Redux DraftNode/Chart。

**加载计划**：data-loading skill propose_data_operation → 前端 DataOperationCard 确认 → interaction_response → DataOperationExecutor → 多表导入。

**报告**：load_skill(report) → write_report 流式 text_delta(channel=report) → generatedReports + Tiptap 编辑器。

### 2.4 控制流：统一错误

`register_error_handlers` 捕获 AppError：鉴权码映射 401/403，其余 200 + `{status:error,error:{code,message,retry}}`。流式预检用 `stream_preflight_error`（仍是 200 JSON）；流内用 `stream_error_event`。前端 `apiRequest`/`streamRequest` + `handleApiError` 是唯一合法消费路径。

### 2.5 可扩展点

1. **新数据源**：实现 ExternalDataLoader 子类，放入 `data_loader/` 或插件目录，提供 list_params / fetch_data_as_arrow / catalog。
2. **新技能**：`analyst/skills/<name>/` 提供 SKILL.md、tools.json、skill.py:get_skill()。
3. **新认证**：实现 AuthProvider，在 providers 包发现机制中注册。
4. **新工作区后端**：实现 WorkspaceManager 协议并在 factory 分支。
5. **新图种**：flint-chart 模板 + ChartTemplates 图标 + 必要时 create_vl_plots 后处理。

扩展时必须同步：错误码 i18n、loader 测试、技能工具 schema、前端连接器表单。



## 3. 数据模型详细设计

数据模型是本系统的合同。后端 dataclass 冻结（frozen）以保证计划不可变；前端 TypeScript interface 描述 Redux 形状；传输层用 JSON，注意 `to_public_dict` 会剥掉 source_id 等内部字段，避免把连接器内部键泄漏到聊天气泡。

### 3.1 身份模型 Identity

- 格式：`local:<os_user>` | `browser:<uuid>` | `user:<idp_sub>`
- 校验：`identity.py` 中正则，禁止 `/`、`..`、过长字符串，防止用作目录名时逃逸。
- 前端 `Identity` 含 type/id；持久化时 browser→user 迁移通过 IdentityMigrationDialog 合并工作区。
- 与 Workspace 的关系：所有磁盘路径都挂在 `users/<safe_id>/` 下，身份即租户。

### 3.2 工作区与表元数据

WorkspaceMetadata / TableMetadata / ColumnInfo / ImportedFrom / Derivation：

- **ImportedFrom**：记录 source_id、source_table、import_options、连接器类型，支持刷新。
- **Derivation**：记录生成该派生表的 Python 代码、签名、父表列表。
- **ColumnInfo**：物理 dtype、描述、语义类型注释。
- 文件布局：`data/*.parquet`、`data/` 上传原件、`scratch/` Agent 产物、`workspace.yaml`。
- 锁：`WorkspaceLock` 避免并发元数据写互相覆盖（有 atomic metadata 测试覆盖）。

### 3.3 DataOperation 模型（加载计划）

这是 0.8 最重要的结构化模型之一，前后端镜像实现。

- `DataOperationStatus`：draft / awaiting_selection / running / loaded / partially_loaded / failed / cancelled / superseded。
- `OperationFilter`：column + operator + 冻结 JSON 值。operator 必须落在 loader 允许集合，防 SQL 注入。
- `LoadQueryOrder`：至多一个 order_by（模型约束，降低各方言实现复杂度）。
- `LoadQuery`：filters / columns / order_by / limit，limit≥1。
- `ConnectorQueryStep`：kind=connector_query，含 source_id、table_key、display_name、source_table、query。
- `DataOperationPlan`：步骤元组 + canonical hash。
- `DataOperation`：id、status、plan、错误列表、产物表名。
- 序列化：`to_dict` 全量（执行用），`to_public_dict` 给 UI（去内部 id）。

不可变性：计划一旦提出，用户只能确认/取消/选择候选项，不能在卡片里随意改 SQL。这是为了让 Agent 的提议可审计、可去重。

### 3.4 Analyst 协议模型

- `SkillMeta`：name、description、always_on、tools、actions、when_to_use。
- `SkillContext`：workspace、runtime（执行代码）、identity、已注册图表等。
- `ToolResult`：ok / message / 可选 data。
- `Event`：dict，外壳打上 iteration 后进入 NDJSON。
- 流事件 type：action、result、tool_start、tool_result、skill_loaded、text_delta、context_info、completion、interact、data_operation_result、error、warning。

### 3.5 错误模型

- `ErrorCode`：AUTH_*、INVALID_REQUEST、TABLE_NOT_FOUND、FILE_*、VALIDATION_ERROR、LLM_*、CONNECTOR_*、DB_*、CODE_EXECUTION_ERROR、AGENT_ERROR、CATALOG_*、INTERNAL_ERROR、SERVICE_UNAVAILABLE、STORAGE_FULL。
- `AppError`：code、message（安全英文回退）、retry、detail（仅 debug）。
- 前端 `ERROR_CODE_I18N_MAP` 映射到 `errors.json`。

### 3.6 前端 Redux 核心类型

`DataFormulatorState` 是客户端会话的超结构，字段包括 identity、models、inputTables、derivedTables、loadedTableNodes、tableSemantics、draftNodes、charts、conceptShelfItems、messages、focusedId、viewMode、generatedReports、textTurns、dataLoadingChat*、activeWorkspace、starterQuestions 等。

`DictTable`：id、displayId、names、rows（派生表样本）、virtual、derive（Trigger+code）、source、contentHash、parentNodeId。

`Chart`：id、chartType、encodingMap、tableRef、config、themeId、styleVariants、scaleFactor。

`DraftNode`：分析进行中的临时节点，status 为 running/clarifying/completed/error/interrupted。

`InteractionEntry`：from/to Actor、role（prompt/clarify/instruction/error/explain/delegate）、content、attachments、timestamp。

`ChatMessage`：数据加载对话气泡，可挂 LoadPlan、ConnectorForm、DataOperation、代码块、表预览。

### 3.7 连接器配置模型

`SourceSpec`：YAML/env 声明的管理型连接器。`DataConnector` 运行时包装 loader 类、vault、自动重连。前端 `CONNECTORS` 数组来自 `/api/app-config`，含 params_form、hierarchy、auth_mode、delegated_login。

### 3.8 知识模型

KnowledgeItem：markdown + YAML front matter（name、always_apply、tags）。data-memory.md 是按用户隔离的连接器记忆。限制常量 `KNOWLEDGE_LIMITS` 防止提示词被超大规则挤爆。

### 3.9 模型配置 ModelConfig

id、endpoint、model、api_key（用户模型）、api_base、api_version、auth_mode、is_global。全局模型凭证永不进入 redux-persist。

### 3.10 目录节点 CatalogNode

层次键由 loader.catalog_hierarchy 定义（如 catalog/schema/table）。缓存策略 CatalogCachePolicy：listing_ttl、metadata_ttl、refresh_cost、automatic_refresh。

以下各节按源码模块展开每一个类与函数，作为数据模型与行为的最终清单。


## 4. 后端模块、类与函数详细设计

本章按目录遍历 `py-src/data_formulator` 下全部 Python 模块。对每个类给出字段、方法、算法与设计约束；对每个函数给出签名与调用约定。这是实现级清单，评审时应以源码为准核对签名，以本章为准理解意图。


### 目录 `py-src/data_formulator`

**目录职责**：应用入口、Flask 装配、错误类型、连接器总控、模型注册、工作区工厂

本目录内模块共享同一边界：只通过公开函数与相邻层通信。禁止跨层 import 私有 `_` 符号（测试除外）。新增文件必须补测试，并检查是否触发路径安全、日志脱敏、错误协议三条规则。


#### 模块 `py-src/data_formulator/__init__.py`（约 12 行）

**模块文档**：该文件未写模块级 docstring。其职责由文件名与导出符号定义，修改时应补充文档字符串，避免意图漂移。

符号统计：类 0 个，函数 1 个，模块级常量 0 个。下列说明覆盖全部符号。

##### 函数 `run_app()`
- **说明**：Launch the Data Formulator Flask application.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

模块级测试建议：至少覆盖（1）正常路径；（2）非法输入；（3）权限/路径拒绝；（4）上游失败时的 AppError 映射。相关用例见后文测试专章。


#### 模块 `py-src/data_formulator/__main__.py`（约 4 行）

**模块文档**：该文件未写模块级 docstring。其职责由文件名与导出符号定义，修改时应补充文档字符串，避免意图漂移。

本文件主要为导出聚合或入口转发，无额外类/函数定义。


#### 模块 `py-src/data_formulator/_startup_spinner.py`（约 79 行）

**模块文档**：Minimal startup spinner.  Animates a single line on a TTY while a slow import / setup step runs. Falls back to plain prints in non-TTY environments (gunicorn, Docker logs, CI, redirected stdout) so log files stay clean.  Usage:     with spinner("Loading AI agents"):         from data_formulator.routes.agents import agent_bp

符号统计：类 0 个，函数 3 个，模块级常量 3 个。下列说明覆盖全部符号。

**常量**：

- **常量 `_FRAMES`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `_FRAME_INTERVAL`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `_INDENT`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

##### 函数 `_enabled()`
- **说明**：模块级函数 `_enabled` 封装可复用算法或路由处理。参数经过校验后进入核心逻辑，返回结构化结果或通过异常表达业务失败。
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_color(code, text)`
- **说明**：模块级函数 `_color` 封装可复用算法或路由处理。参数经过校验后进入核心逻辑，返回结构化结果或通过异常表达业务失败。
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `spinner(label)`
- **说明**：Context manager that animates `label` on stdout while the body runs.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

模块级测试建议：至少覆盖（1）正常路径；（2）非法输入；（3）权限/路径拒绝；（4）上游失败时的 AppError 映射。相关用例见后文测试专章。


#### 模块 `py-src/data_formulator/agent_config.py`（约 152 行）

**模块文档**：Single source of truth for per-agent LLM call configuration.  Edit values here to tune latency vs. quality for each agent.  Per-agent overrides can also be set at runtime via environment variables:      DF_REASONING_EFFORT_DATA_TRANSFORM=medium     DF_REASONING_EFFORT_REPORT_GEN=high  Tiers ----- - ``"minimal"`` — fastest. Honoured natively only on the OpenAI GPT-5   base/mini/nano/5.x family (``gpt-5``, ``gpt-5-mini``, ``gpt-5-nano``,   ``gpt-5.1``, ...). On the GPT-5 ``codex`` / ``pro`` variants   :func:`reasoning_effort_for` maps it to ``"none"`` (their lightest   supported tier). On every other reasoning model (o-series, Claude   extended-thinking, Gemini, ...) it is downgraded to ``"low"``. - ``"none"`` — only accepted by GPT-5 ``codex`` / ``pro``. Downgraded to   ``"low"`` elsewhere.

符号统计：类 0 个，函数 4 个，模块级常量 0 个。下列说明覆盖全部符号。

##### 函数 `get_reasoning_effort(agent_id)`
- **说明**：Return the *configured* tier for ``agent_id``.  Resolution order:     1. ``DF_REASONING_EFFORT_<AGENT_ID>`` env var     2. ``AGENT_REASONING_EFFORT[agent_id]``     3. ``DEFAULT_REASONING_EFFORT``  Note: this does **not** consider the target model. Use :func:`reasoning_effort_for` at call time to also apply the GPT-5-only ``"minimal"`` gating.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_supports_minimal(model)`
- **说明**：``"minimal"`` is only accepted by a subset of OpenAI GPT-5 chat models.  Supported (per OpenAI API):     ``gpt-5``, ``gpt-5-mini``, ``gpt-5-nano``, ``gpt-5.1``, ``gpt-5.4``,     and future GPT-5.x sub-versions of those base variants.  NOT supported (these reject ``"minimal"`` but accept ``"none"`` / ``xhigh`` instead): ``gpt-5-codex``, ``gpt-5-pro``.  Provider prefixes such as ``openai/gpt-5-mini``, ``azure/gpt-5``, ``openai/responses/gpt-5.4`` are all covered by the substring check.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_supports_none(model)`
- **说明**：``"none"`` is the lightest tier on the GPT-5 ``codex`` / ``pro`` chat models (which reject ``"minimal"``). Other providers (Claude, Gemini, o-series) don't accept ``"none"`` as a reasoning_effort value, so we only use it for these specific GPT-5 variants.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `reasoning_effort_for(agent_id, model)`
- **说明**：Resolve the reasoning_effort to actually send to LiteLLM.  - Reads the configured tier via :func:`get_reasoning_effort`. - For configured ``"minimal"``:     * keep ``"minimal"`` on GPT-5 base / mini / nano / 5.x;     * map to ``"none"`` on GPT-5 codex / pro (which support ``"none"``       but not ``"minimal"``);     * fall back to ``"low"`` on every other reasoning model. - For configured ``"none"`` on a non-supporting model, fall back to   ``"low"``.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

模块级测试建议：至少覆盖（1）正常路径；（2）非法输入；（3）权限/路径拒绝；（4）上游失败时的 AppError 映射。相关用例见后文测试专章。


#### 模块 `py-src/data_formulator/app.py`（约 560 行）

**模块文档**：该文件未写模块级 docstring。其职责由文件名与导出符号定义，修改时应补充文档字符串，避免意图漂移。

符号统计：类 1 个，函数 10 个，模块级常量 3 个。下列说明覆盖全部符号。

**常量**：

- **常量 `APP_ROOT`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `_LOG_FORMAT`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `_FILE_HANDLER_MARKER`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

##### 类 `CustomJSONEncoder`
- **基类**：json.JSONEncoder。
- **字段标注**：无模块级注解字段（可能使用 dataclass / 动态属性）。
- **方法数**：1。
- **职责推断**：该类位于对应模块中，承担 `CustomJSONEncoder` 所表示的领域对象或服务。其方法覆盖构造、序列化、领域操作与错误处理；调用方应通过公开方法交互，避免依赖私有实现细节。
  - **`default(self, obj)`**：内部实现细节见源码。
  设计约束：公开方法必须返回安全数据（无密钥、无绝对路径）；异常应转换为 AppError 或 ToolResult 可恢复错误，而不是把 Python 堆栈直接交给前端。

##### 函数 `_resolve_data_home()`
- **说明**：Resolve the Data Formulator home directory for logging.  Mirrors ``get_data_formulator_home()`` but is safe to call outside an application context (logging is configured before the app runs). Order: CLI_ARGS['data_dir'] > DATA_FORMULATOR_HOME env > ~/.data_formulator.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `configure_file_logging()`
- **说明**：Attach a rotating file handler that persists all logs under ``<DATA_FORMULATOR_HOME>/logs/data_formulator.log``.  This is the artifact users can send when reporting problems. Output is passed through :class:`SensitiveDataFilter` so API keys / tokens are redacted. The handler is idempotent — if the resolved path changes (e.g. ``--data-dir`` supplied after early configuration) the old handler is replaced.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `configure_logging()`
- **说明**：Configure logging for the Flask application.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_register_blueprints()`
- **说明**：Import and register blueprints. This is where heavy imports happen. Called at module level (for gunicorn) and from run_app() (for CLI). Guarded to prevent double registration.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_safety_checks()`
- **说明**：Warn about dangerous configuration combinations at startup.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `index_alt(path)`
- **说明**：模块级函数 `index_alt` 封装可复用算法或路由处理。参数经过校验后进入核心逻辑，返回结构化结果或通过异常表达业务失败。
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `get_auth_info()`
- **说明**：Return authentication configuration for the frontend.  The response tells the frontend how to initiate login based on the active provider (OIDC PKCE, GitHub redirect, transparent, or none).
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `get_app_config()`
- **说明**：Provide frontend configuration settings from CLI arguments
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `parse_args()`
- **说明**：模块级函数 `parse_args` 封装可复用算法或路由处理。参数经过校验后进入核心逻辑，返回结构化结果或通过异常表达业务失败。
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `run_app()`
- **说明**：模块级函数 `run_app` 封装可复用算法或路由处理。参数经过校验后进入核心逻辑，返回结构化结果或通过异常表达业务失败。
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

模块级测试建议：至少覆盖（1）正常路径；（2）非法输入；（3）权限/路径拒绝；（4）上游失败时的 AppError 映射。相关用例见后文测试专章。


#### 模块 `py-src/data_formulator/data_connector.py`（约 2720 行）

**模块文档**：DataConnector — generic lifecycle wrapper for ExternalDataLoader.  Takes any ``ExternalDataLoader`` class and auto-generates a Flask Blueprint with auth / catalog / data routes.  No per-connector code needed.  Usage::      from data_formulator.data_connector import DataConnector      connector = DataConnector.from_loader(         PostgreSQLDataLoader,         source_id="pg_prod",         display_name="Production DB",         default_params={"host": "db.corp", "database": "prod"},     )     app.register_blueprint(connector.create_blueprint())

符号统计：类 2 个，函数 65 个，模块级常量 6 个。下列说明覆盖全部符号。

**常量**：

- **常量 `_MAX_CATALOG_PAGE_SIZE`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `_USER_CONNECTOR_PREFIX`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `_RECONNECT_MAX_ATTEMPTS`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `_RECONNECT_BACKOFF_BASE`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `_CATALOG_PROGRESS_LOCK`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `_CONNECTOR_ID_RE`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

##### 函数 `_set_catalog_progress(connector_id, message)`
- **说明**：模块级函数 `_set_catalog_progress` 封装可复用算法或路由处理。参数经过校验后进入核心逻辑，返回结构化结果或通过异常表达业务失败。
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_get_catalog_progress(connector_id)`
- **说明**：模块级函数 `_get_catalog_progress` 封装可复用算法或路由处理。参数经过校验后进入核心逻辑，返回结构化结果或通过异常表达业务失败。
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_clear_catalog_progress(connector_id)`
- **说明**：模块级函数 `_clear_catalog_progress` 封装可复用算法或路由处理。参数经过校验后进入核心逻辑，返回结构化结果或通过异常表达业务失败。
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `classify_and_raise_connector_error(error)`
- **说明**：Classify a connector error and raise ``AppError``.  Preserves the historical entry point while delegating classification to the shared DataLoader/connector classifier.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_sanitize_error(error)`
- **说明**：Legacy wrapper — prefer ``classify_and_raise_connector_error``.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_node_to_dict(node)`
- **说明**：模块级函数 `_node_to_dict` 封装可复用算法或路由处理。参数经过校验后进入核心逻辑，返回结构化结果或通过异常表达业务失败。
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_hierarchy_dicts(levels)`
- **说明**：模块级函数 `_hierarchy_dicts` 封装可复用算法或路由处理。参数经过校验后进入核心逻辑，返回结构化结果或通过异常表达业务失败。
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_catalog_pagination_args(data)`
- **说明**：Parse optional catalog pagination args from a request body.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_filter_catalog_tables(tables, name_filter)`
- **说明**：Filter flat catalog tables without querying the source again.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_lightweight_tree_for_response(tree)`
- **说明**：Drop heavy metadata fields from a catalog tree response.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_merged_catalog_tables(user_home, source_id, flat_tables)`
- **说明**：Return loader-produced catalog tables as-is.  User annotations were removed in favor of source-system metadata only. This helper kept its name to avoid call-site churn; it's now a no-op pass-through around the loader output. Future automated metadata enrichers (LLM-generated descriptions, etc.) would hook in here.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_catalog_tree_payload(loader, flat_tables)`
- **说明**：Build the lightweight tree sent to the frontend.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_user_connector_key(identity, source_id)`
- **说明**：Return the internal registry key for a user-owned connector.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_is_user_connector_key(key)`
- **说明**：模块级函数 `_is_user_connector_key` 封装可复用算法或路由处理。参数经过校验后进入核心逻辑，返回结构化结果或通过异常表达业务失败。
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_public_connector_id(registry_key, connector)`
- **说明**：Return the public connector ID exposed to clients.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_param_defs_by_name(loader_class)`
- **说明**：模块级函数 `_param_defs_by_name` 封装可复用算法或路由处理。参数经过校验后进入核心逻辑，返回结构化结果或通过异常表达业务失败。
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_is_sensitive_or_auth_param(loader_class, name)`
- **说明**：Return whether a loader parameter should not be exposed as pinned config.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_connector_config_params(loader_class, params)`
- **说明**：Keep only non-sensitive params in persisted connector config.  Auth-tier identifiers (e.g. ``user``) are kept; only truly sensitive fields (passwords, tokens, anything marked ``sensitive: True``) are stripped. Sensitive credentials live in the per-identity vault, not on disk.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_loader_auth_mode(loader_class)`
- **说明**：Return the effective auth mode, preferring modern auth_config().
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_visible_connector_items(identity)`
- **说明**：Return registry entries visible to the current identity.  Admin connectors are global. User connectors are keyed by identity in the process registry. Raw non-admin entries are treated as legacy/test globals; newly created user connectors should use ``_user_connector_key``.  When external connectors are disabled (browser-only / hosted mode), only built-in admin connectors (e.g. ``sample_datasets``) are exposed — previously-persisted user connectors on disk are hidden so the sidebar stays clean an
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_resolve_connector_with_key(data)`
- **说明**：Look up a connector visible to the current request identity.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 类 `DataConnector`
- **基类**：无显式基类。
- **字段标注**：无模块级注解字段（可能使用 dataclass / 动态属性）。
- **方法数**：20。
- **文档字符串**：Generic lifecycle wrapper for an ExternalDataLoader.  Provides connect / disconnect / status, catalog browsing, and data import / preview / refresh — all driven by the underlying loader's existing methods.  Routes live on the shared ``connectors_bp`` blueprint; this class is a plain Python object (no per-instance blueprint).
  - **`__init__(self, loader_class, source_id, display_name, default_params, icon)`**：内部实现细节见源码。
  - **`from_loader(cls, loader_class, source_id, display_name, default_params, icon)`**：内部实现细节见源码。
  - **`_manifest(self)`**：内部实现细节见源码。
  - **`get_frontend_config(self, include_pinned_in_form)`**：Build the frontend payload describing this connector's form.  Args:     include_pinned_in_form: When True, params that have saved         values in ``_default_params`` are still emitted in         ``params_form`` so the 
  - **`_resolve_delegated_login(self)`**：Resolve delegated login config, converting relative URLs to absolute.
  - **`_get_identity()`**：内部实现细节见源码。
  - **`_get_vault()`**：Return the credential vault (or None if unavailable).
  - **`_vault_store(self, identity, user_params)`**：Encrypt and persist user_params for this source. Returns True on success.
  - **`_vault_retrieve(self, identity)`**：Retrieve stored user_params from the vault. Returns None if absent.
  - **`_vault_delete(self, identity)`**：Delete stored credentials from the vault.
  - **`has_stored_credentials(self, identity)`**：Check if the vault has credentials for this identity+source.
  - **`_get_loader(self, identity)`**：内部实现细节见源码。
  - **`_connect(self, user_params, persist)`**：Instantiate a loader with merged params (default + user).  Note: This only creates the loader and caches it in-memory. Vault persistence is handled separately by the caller after connection verification succeeds.
  - **`_persist_credentials(self, user_params)`**：Store credentials in the vault for the current identity.
  - **`_delete_credentials(self)`**：Delete: clear in-memory loader AND vault credentials.
  - **`_try_auto_reconnect(self, identity)`**：Attempt to restore a connection from vault credentials.  Retries the connection test up to ``_RECONNECT_MAX_ATTEMPTS`` times with exponential backoff to ride out transient failures. Stale vault credentials are only clear
  - **`_try_ambient_reconnect(self, identity)`**：Reconnect using only the connector's pinned connection params.  Some connectors authenticate with *ambient*, host-provided credentials rather than anything stored per-user — e.g. Kusto via ``DefaultAzureCredential`` (``a
  - **`_inject_credentials(self, params)`**：Inject the best available credentials via TokenStore.  Falls back to the legacy SSO token injection when TokenStore is unavailable (e.g. outside a request context).
  - **`_try_sso_auto_connect(self, identity)`**：Try to auto-connect using TokenStore or the current SSO token.  Only applies to token/sso_exchange-mode loaders when no vault credentials exist.
  - **`_require_loader(self)`**：内部实现细节见源码。
  设计约束：公开方法必须返回安全数据（无密钥、无绝对路径）；异常应转换为 AppError 或 ToolResult 可恢复错误，而不是把 Python 堆栈直接交给前端。

##### 函数 `_resolve_connector(data)`
- **说明**：Look up a DataConnector from the request body's ``connector_id``.  Returns the connector or raises ``AppError``.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `resolve_live_loader(source_id)`
- **说明**：Resolve a live, connected loader for ``source_id`` in the current identity.  Mirrors the request-route resolution path (``_resolve_connector_with_key`` + ``_require_loader``) so callers running inside a Flask request context — notably the data-loading agent's live tools — can turn a ``source_id`` into a connected :class:`ExternalDataLoader` mid-turn.  Raises ``AppError`` if the source is unknown to the identity, or ``ValueError`` if it is known but not connected.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `resolve_catalog_refresh_target(source_id)`
- **说明**：Resolve policy and an existing loader without reconnecting credentials.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `connector_is_available(source_id)`
- **说明**：Whether ``source_id`` could be loaded from right now, without touching it.  Deliberately avoids ``test_connection`` so discovery can screen every source cheaply: a live loader, stored credentials, or usable SSO all count, since each lets the load path reconnect on demand. Returns ``None`` when this can't be determined (no request context, connectors disabled, unknown id) — callers must not treat that as unavailable.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_parse_source_table(raw)`
- **说明**：Normalise the ``source_table`` value from a request body.  Accepts two shapes: - **structured** ``{"id": "42", "name": "orders_fact"}`` — id is the   opaque identifier the loader needs, name is human-readable. - **plain string** ``"public.users"`` — used as both id and name   (backward-compatible for simple DB loaders).  Returns ``(source_id, source_name)``.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_cached_source_metadata(source, source_table_id)`
- **说明**：Look up a table's metadata (columns + description) from the synced catalog cache.  Lets load/refresh enrich persisted ``TableMetadata`` without a live source round-trip — the columns and descriptions were already collected in bulk at sync time. Returns ``None`` when the catalog has nothing usable for this table (caller then falls back to a live fetch).
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `list_data_loaders()`
- **说明**：Return available loader types + their param definitions.  This is the discovery endpoint — tells the frontend what kinds of connectors can be created.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `discover_data_loader_options()`
- **说明**：Discover values for one loader parameter after an explicit UI action.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `pick_local_directory()`
- **说明**：Open a native OS directory picker and return the selected path.  Only available in local deployment mode (backend bound to localhost).  Strategy per platform (each uses tools that ship with the OS):  - **macOS**: ``osascript`` (AppleScript) — ships with every Mac. - **Windows**: PowerShell ``System.Windows.Forms.FolderBrowserDialog``   — built into Windows 10+. - **Linux**: tries in order: ``zenity`` (GNOME), ``kdialog`` (KDE),   ``tkinter`` (Python stdlib, if compiled with Tk).  If no dialog to
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_az_account_summary()`
- **说明**：Return the currently signed-in Azure CLI account, or None.  Runs ``az account show`` (no shell) and parses the JSON output. Returns ``None`` when the CLI is missing, the user is not signed in, or the call fails/times out.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `azure_cli_status()`
- **说明**：Report whether the local Azure CLI is installed and signed in.  Only available in local deployment mode. Used by connectors that support Microsoft Entra ID auth (e.g. SQL Server) to show the current sign-in state next to an in-app "Sign in with Azure CLI" button.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `azure_cli_login()`
- **说明**：Run ``az login`` on the local machine, opening the system browser.  Only available in local deployment mode (the backend runs on the user's own machine, so the browser opens for them). Blocks until the interactive sign-in completes or times out. On success returns the signed-in account.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `list_connectors()`
- **说明**：List all registered connector instances (admin + user) with connection status.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `create_connector()`
- **说明**：Create a new user connector instance from a loader type.  Request body::      {         "loader_type": "mysql",         "display_name": "MySQL · prod",         "params": {"host": "...", "port": "3306", ...},         "icon": "mysql",         "persist": true     }  Persists to ``DATA_FORMULATOR_HOME/users/<identity>/connectors/<source_id>.json``.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_connectors_dir(identity)`
- **说明**：Return ``DATA_FORMULATOR_HOME/users/<identity>/connectors/``.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_connectors_jail(identity)`
- **说明**：Return a confined view of the per-user connectors directory.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_validate_connector_id_for_fs(connector_id)`
- **说明**：Validate connector id before any filesystem-derived usage.  Allows stable connector IDs used by this app (alnum plus ``_.:-``), rejects separators/whitespace and other special characters.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_safe_source_filename(source_id)`
- **说明**：Sanitise a source_id into a safe, collision-resistant filename component.  Delegates to :func:`datalake.naming.safe_source_id` — the single source of truth shared with :mod:`datalake.catalog_cache`.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_persist_user_connector(identity, spec)`
- **说明**：Write a single connector spec to ``connectors/<source_id>.json``.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_remove_user_connector(identity, connector_id)`
- **说明**：Remove a connector spec from ``connectors/<source_id>.json``.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_update_user_connector_display_name(identity, connector_id, display_name)`
- **说明**：模块级函数 `_update_user_connector_display_name` 封装可复用算法或路由处理。参数经过校验后进入核心逻辑，返回结构化结果或通过异常表达业务失败。
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `update_connector(connector_id)`
- **说明**：Rename a user connector without changing its stable source ID.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `delete_connector(connector_id)`
- **说明**：Delete a **user** connector instance, clear vault credentials, and remove from config.  Admin connectors cannot be deleted (returns 403).
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `connector_connect()`
- **说明**：(Re)connect / authenticate a connector instance.  Accepts ``connector_id`` plus two modes:  **Credential mode** (default)::      {"connector_id": "mysql:prod", "params": {...}, "persist": true}  **Token mode** (delegated/SSO)::      {"connector_id": "...", "mode": "token", "access_token": "eyJ...",      "refresh_token": "...", "user": {...}, "params": {...}, "persist": true}
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `connector_disconnect()`
- **说明**：Disconnect a connector for the current identity.  This clears the in-memory loader and stored credentials for this connector, but keeps the connector definition itself so it can be reconnected later.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `connector_get_status()`
- **说明**：Check connection status (no side effects — no auto-reconnect).
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `connector_get_catalog()`
- **说明**：Browse a catalog node (merged ls + metadata).
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `connector_get_catalog_tree()`
- **说明**：Build nested tree from cache, falling back to live lightweight listing.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `connector_get_catalog_progress()`
- **说明**：Return the latest catalog-listing progress message for a connector.  Polled by the frontend while a live ``get-catalog-tree`` request is in flight so it can show which database/source is being queried alongside the spinner. Returns an empty message when nothing is in progress.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `connector_get_cached_catalog_tree()`
- **说明**：Return the catalog tree from the **disk cache** without querying the source.  Used by the frontend when expanding a connector after page reload. Falls back to ``status: "miss"`` when no cache exists so the frontend can decide whether to trigger a live sync.  Response (hit)::      {"status": "ok", "tree": [...], "hierarchy": [...],      "effective_hierarchy": [...], "synced_at": "..."}  Response (miss)::      {"status": "miss"}
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `connector_sync_catalog_metadata()`
- **说明**：Full metadata sync — enriched catalog for agent search and tree display.  Calls ``loader.sync_catalog_metadata()`` which returns all tables with as-complete-as-possible column info, writes the result to ``catalog_cache``, and returns the full tree for the frontend to render.  Response::      {         "status": "ok",         "tree": [...],         "sync_summary": {"synced": N, "partial": N, "failed": N, "total": N}     }
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `connector_search_catalog()`
- **说明**：Search a connected connector's catalog without reconnecting.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `connector_import_data()`
- **说明**：模块级函数 `connector_import_data` 封装可复用算法或路由处理。参数经过校验后进入核心逻辑，返回结构化结果或通过异常表达业务失败。
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `connector_refresh_data()`
- **说明**：模块级函数 `connector_refresh_data` 封装可复用算法或路由处理。参数经过校验后进入核心逻辑，返回结构化结果或通过异常表达业务失败。
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `connector_preview_data()`
- **说明**：模块级函数 `connector_preview_data` 封装可复用算法或路由处理。参数经过校验后进入核心逻辑，返回结构化结果或通过异常表达业务失败。
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `connector_column_values()`
- **说明**：Return distinct values for a dataset column (smart filter support).
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `connector_import_group()`
- **说明**：Import all tables from a table_group with shared filters.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 类 `SourceSpec`
- **基类**：无显式基类。
- **字段标注**：source_id, loader_type, display_name, default_params, icon, auto_connect, source。
- **方法数**：0。
- **文档字符串**：A single connector entry from config (YAML or env vars).
  设计约束：公开方法必须返回安全数据（无密钥、无绝对路径）；异常应转换为 AppError 或 ToolResult 可恢复错误，而不是把 Python 堆栈直接交给前端。

##### 函数 `_resolve_env_refs(params)`
- **说明**：Resolve ``${ENV_VAR}`` references in param values.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_get_df_home()`
- **说明**：Return DATA_FORMULATOR_HOME as a Path.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_load_connectors_yaml(path)`
- **说明**：Load a connectors.yaml file and return the list of connector entries.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_load_admin_specs()`
- **说明**：Load admin connectors from DATA_FORMULATOR_HOME/connectors.yaml + env vars.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_load_user_specs(identity)`
- **说明**：Load user connectors from ``connectors/`` directory.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `load_connectors(identity)`
- **说明**：Ensure DATA_CONNECTORS contains admin + user connectors for *identity*.  Admin connectors are loaded at startup by :func:`register_data_connectors`. Calling this with an identity lazily adds the user's connectors on first request.  Subsequent calls for the same identity are no-ops.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `register_data_connectors(app)`
- **说明**：Register the global connectors blueprint + admin-provisioned connectors.  Called from ``app.py`` during startup.  - Registers ``connectors_bp`` with all shared routes. - Loads admin connectors from ``DATA_FORMULATOR_HOME/connectors.yaml``   and ``DF_SOURCES__*`` env vars. - User connectors are loaded lazily on first request (need identity).
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

模块级测试建议：至少覆盖（1）正常路径；（2）非法输入；（3）权限/路径拒绝；（4）上游失败时的 AppError 映射。相关用例见后文测试专章。


#### 模块 `py-src/data_formulator/desktop.py`（约 297 行）

**模块文档**：该文件未写模块级 docstring。其职责由文件名与导出符号定义，修改时应补充文档字符串，避免意图漂移。

符号统计：类 0 个，函数 11 个，模块级常量 5 个。下列说明覆盖全部符号。

**常量**：

- **常量 `_INSTANCE_HOST`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `_INSTANCE_PORT`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `_ACTIVATE_MESSAGE`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `_ACTIVATE_ACK`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `_LOADING_HTML`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

##### 函数 `_configure_standard_streams()`
- **说明**：模块级函数 `_configure_standard_streams` 封装可复用算法或路由处理。参数经过校验后进入核心逻辑，返回结构化结果或通过异常表达业务失败。
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_signal_existing_instance(timeout)`
- **说明**：模块级函数 `_signal_existing_instance` 封装可复用算法或路由处理。参数经过校验后进入核心逻辑，返回结构化结果或通过异常表达业务失败。
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_claim_single_instance()`
- **说明**：模块级函数 `_claim_single_instance` 封装可复用算法或路由处理。参数经过校验后进入核心逻辑，返回结构化结果或通过异常表达业务失败。
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_listen_for_activation(coordinator, activate)`
- **说明**：模块级函数 `_listen_for_activation` 封装可复用算法或路由处理。参数经过校验后进入核心逻辑，返回结构化结果或通过异常表达业务失败。
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_activate_window(window, activate)`
- **说明**：模块级函数 `_activate_window` 封装可复用算法或路由处理。参数经过校验后进入核心逻辑，返回结构化结果或通过异常表达业务失败。
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_available_port()`
- **说明**：模块级函数 `_available_port` 封装可复用算法或路由处理。参数经过校验后进入核心逻辑，返回结构化结果或通过异常表达业务失败。
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_wait_until_ready(url, timeout)`
- **说明**：模块级函数 `_wait_until_ready` 封装可复用算法或路由处理。参数经过校验后进入核心逻辑，返回结构化结果或通过异常表达业务失败。
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_enable_per_monitor_dpi()`
- **说明**：Upgrade DPI awareness to Per-Monitor V2 before the GUI starts.  pywebview's WinForms backend only calls SetProcessDPIAware() (system DPI aware), which locks the scale factor at startup; on high-DPI displays the WebView2 content is then stretched after resizing or maximizing. Per-Monitor V2 lets Windows re-render the window for the monitor it is on. It requires Windows 10 1703+; failures degrade silently to the backend's default.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_run_self_test()`
- **说明**：Exercise sandbox execution inside a packaged build.  Parquet reads pull in pyarrow modules that live in the PyInstaller archive, a path that only exists in frozen builds and cannot be covered by pytest.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_self_test_clr()`
- **说明**：Load the managed pythonnet assembly the WinForms backend depends on.  Python.Runtime.dll is a .NET assembly; if packaging rewrites it the CLR cannot resolve Loader.Initialize and the GUI dies at startup. Importing `clr` reproduces that load without needing a desktop session.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `run_desktop()`
- **说明**：模块级函数 `run_desktop` 封装可复用算法或路由处理。参数经过校验后进入核心逻辑，返回结构化结果或通过异常表达业务失败。
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

模块级测试建议：至少覆盖（1）正常路径；（2）非法输入；（3）权限/路径拒绝；（4）上游失败时的 AppError 映射。相关用例见后文测试专章。


#### 模块 `py-src/data_formulator/error_handler.py`（约 350 行）

**模块文档**：Unified error handling and response helpers for the Data Formulator Flask application.  Public entry points:  * ``register_error_handlers(app)`` — call once during app setup to install   global error handlers and the request-id middleware. * ``classify_and_wrap_llm_error(exc)`` — convert a raw LLM / external-API   exception into a structured ``AppError``. * ``stream_error_event(error)`` — format an error as a single   NDJSON line for streaming endpoints. * ``json_ok(data)`` — build a success JSON response with the unified envelope. * ``stream_preflight_error(error)`` — build an error response for streaming   endpoint pre-flight validation failures (always HTTP 200).

符号统计：类 0 个，函数 9 个，模块级常量 0 个。下列说明覆盖全部符号。

##### 函数 `_safe_unexpected_detail(exc, request_id)`
- **说明**：Return a safe debug hint for an unhandled exception.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `classify_and_wrap_llm_error(exc)`
- **说明**：Convert a raw LLM / external-API exception into a structured AppError.  Reuses ``classify_llm_error`` from ``sanitize.py`` for the safe user-facing message, then maps to an ``ErrorCode`` and retry flag. The original exception text is preserved in ``detail`` for server-side logging but is **never** included in the client-facing ``message``.
- **算法要点**：用正则把上游 LLM 异常映射到 ErrorCode（限流、鉴权、上下文过长、内容过滤等）。
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `stream_error_event(error)`
- **说明**：Format an error as a single NDJSON line for streaming endpoints.  Returns a string ending with ``\n`` that can be directly yielded from a Flask streaming generator.
- **算法要点**：把 AppError 编码为一行 NDJSON {type:error,error:{code,message,retry}}，流式端点始终 HTTP 200。
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `stream_warning_event(message)`
- **说明**：Format a non-fatal warning as a single NDJSON line.  Unlike :func:`stream_error_event`, a warning does **not** abort the stream — it is an advisory notice (e.g. "table X unavailable, using degraded context") that the frontend can display as a toast / snackbar.  Returns a string ending with ``\n``.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `collect_stream_warning(message)`
- **说明**：Accumulate a warning in the current request context.  Stored on ``flask.g`` so that any code running within a request (including inside agent helper functions that cannot ``yield``) can emit warnings.  The streaming generator drains them via :func:`flush_stream_warnings`.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `flush_stream_warnings()`
- **说明**：Return and clear all accumulated warning NDJSON lines.  Call this inside a streaming generator (wrapped with ``stream_with_context``) to inject pending warnings into the stream.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `json_ok(data)`
- **说明**：Build a unified success JSON response.  Returns ``(Response, status_code)`` for Flask.  The envelope uses ``"status": "success"`` (not legacy ``"ok"``) and wraps the payload in the ``"data"`` key::      {"status": "success", "data": {...}}  The response body must not include endpoint-specific top-level fields.
- **算法要点**：统一成功 envelope：{status:success,data:...}，禁止路由手写 jsonify 成功体。
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `stream_preflight_error(error)`
- **说明**：Build an error response for streaming pre-flight validation failures.  Streaming endpoints call this (instead of ``raise AppError()``) when validation fails *before* the NDJSON stream starts.  The frontend's ``streamRequest()`` detects the ``application/json`` content-type (vs expected ``application/x-ndjson``) and throws ``ApiRequestError``.  **Always returns HTTP 200** — consistent with non-streaming error policy.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `register_error_handlers(app)`
- **说明**：Register global error handlers and request-id middleware on *app*.  Call this once during application setup (in ``app.py``).  It installs:  * ``AppError`` handler — structured JSON, HTTP 200 for business errors,   401/403 only for auth errors. * ``413`` handler — file too large. * ``404`` handler — JSON for ``/api/`` routes, SPA fallback otherwise. * Catch-all ``Exception`` handler — generic 500 JSON. * ``before_request`` / ``after_request`` hooks for ``X-Request-Id``.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

模块级测试建议：至少覆盖（1）正常路径；（2）非法输入；（3）权限/路径拒绝；（4）上游失败时的 AppError 映射。相关用例见后文测试专章。


#### 模块 `py-src/data_formulator/errors.py`（约 149 行）

**模块文档**：Unified error types for the Data Formulator backend.  Every business error raised in routes / agents / data layer should be an ``AppError`` (or a subclass).  The global error handlers registered by ``error_handler.register_error_handlers`` convert ``AppError`` instances into a consistent JSON envelope before they reach the client.  ``ErrorCode`` provides machine-readable codes that the frontend maps to localised user-facing messages via the i18n ``errors`` namespace.

符号统计：类 2 个，函数 0 个，模块级常量 0 个。下列说明覆盖全部符号。

##### 类 `ErrorCode`
- **基类**：无显式基类。
- **字段标注**：无模块级注解字段（可能使用 dataclass / 动态属性）。
- **方法数**：0。
- **文档字符串**：Machine-readable error codes.  The frontend uses these codes for: * Differential error handling (auth errors → redirect, rate-limit → retry) * i18n lookup (code → translated user message)  Convention: values are ``UPPER_SNAKE_CASE`` strings identical to the attribute name so that ``ErrorCode.TABLE_NOT_FOUND == "TABLE_NOT_FOUND"``.
  设计约束：公开方法必须返回安全数据（无密钥、无绝对路径）；异常应转换为 AppError 或 ToolResult 可恢复错误，而不是把 Python 堆栈直接交给前端。

##### 类 `AppError`
- **基类**：Exception。
- **字段标注**：无模块级注解字段（可能使用 dataclass / 动态属性）。
- **方法数**：3。
- **文档字符串**：Unified business exception for the Data Formulator backend.  Parameters ---------- code:     A machine-readable ``ErrorCode`` constant. message:     A safe, user-readable description (English).  This is the *fallback*     text shown to the user when the frontend has no i18n translation for     *code*.  **Must not** contain secrets, paths, or stack traces. status_code:     The HTTP status code to r
  - **`__init__(self, code, message)`**：内部实现细节见源码。
  - **`get_http_status(self)`**：Return the HTTP status code for this error.  Only auth-related codes (AUTH_REQUIRED, AUTH_EXPIRED, ACCESS_DENIED) return non-200.  All other application errors return HTTP 200 so that the protocol is consistent with stre
  - **`to_dict(self, include_detail)`**：Serialise to the ``error`` object in the JSON response envelope.
  设计约束：公开方法必须返回安全数据（无密钥、无绝对路径）；异常应转换为 AppError 或 ToolResult 可恢复错误，而不是把 Python 堆栈直接交给前端。

模块级测试建议：至少覆盖（1）正常路径；（2）非法输入；（3）权限/路径拒绝；（4）上游失败时的 AppError 映射。相关用例见后文测试专章。


#### 模块 `py-src/data_formulator/example_datasets_config.py`（约 420 行）

**模块文档**：Sample datasets configuration for Data Formulator.

符号统计：类 0 个，函数 0 个，模块级常量 1 个。下列说明覆盖全部符号。

**常量**：

- **常量 `EXAMPLE_DATASETS`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

模块级测试建议：至少覆盖（1）正常路径；（2）非法输入；（3）权限/路径拒绝；（4）上游失败时的 AppError 映射。相关用例见后文测试专章。


#### 模块 `py-src/data_formulator/model_registry.py`（约 114 行）

**模块文档**：该文件未写模块级 docstring。其职责由文件名与导出符号定义，修改时应补充文档字符串，避免意图漂移。

符号统计：类 1 个，函数 0 个，模块级常量 1 个。下列说明覆盖全部符号。

**常量**：

- **常量 `BUILTIN_PROVIDERS`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

##### 类 `ModelRegistry`
- **基类**：无显式基类。
- **字段标注**：无模块级注解字段（可能使用 dataclass / 动态属性）。
- **方法数**：7。
- **文档字符串**：Load global model configurations from environment variables.  Supports both built-in providers (openai / azure / anthropic / gemini / ollama) and arbitrary custom providers (e.g. DEEPSEEK, QWEN).  For a custom provider, set:     {PROVIDER}_ENABLED=true     {PROVIDER}_ENDPOINT=openai        # actual call type; defaults to openai     {PROVIDER}_API_KEY=<key>     {PROVIDER}_API_BASE=<url>     {PROVID
  - **`__init__(self)`**：内部实现细节见源码。
  - **`make_id(provider, model)`**：内部实现细节见源码。
  - **`_discover_providers(self)`**：Return the lowercase names of all enabled providers by scanning every environment variable that ends with _ENABLED=true.
  - **`_reload(self)`**：内部实现细节见源码。
  - **`get_config(self, model_id)`**：Return the full config (including credentials) for a global model.
  - **`list_public(self)`**：Return public info for all globally configured models. Sensitive fields (api_key) are intentionally excluded.
  - **`is_global(self, model_id)`**：内部实现细节见源码。
  设计约束：公开方法必须返回安全数据（无密钥、无绝对路径）；异常应转换为 AppError 或 ToolResult 可恢复错误，而不是把 Python 堆栈直接交给前端。

模块级测试建议：至少覆盖（1）正常路径；（2）非法输入；（3）权限/路径拒绝；（4）上游失败时的 AppError 映射。相关用例见后文测试专章。


#### 模块 `py-src/data_formulator/workspace_factory.py`（约 149 行）

**模块文档**：Flask-aware workspace factory.  Reads the workspace backend configuration from Flask's ``current_app.config`` (populated by CLI args / env vars in ``app.py``) and returns the appropriate :class:`Workspace` subclass.  This keeps the data-layer modules (``datalake.workspace``, ``datalake.azure_blob_workspace``) free of any Flask dependency.  Multi-workspace support:   - Each user has a WorkspaceManager with multiple named workspaces.   - ``get_workspace()`` returns the active workspace (read from X-Workspace-Id header).   - ``get_workspace_manager()`` returns the WorkspaceManager for workspace CRUD.   - Backend is stateless: workspace ID comes from the frontend on every request.

符号统计：类 0 个，函数 6 个，模块级常量 0 个。下列说明覆盖全部符号。

##### 函数 `_build_azure_container_client(cfg)`
- **说明**：Create an Azure ``ContainerClient`` using the best available credential.  Resolution order:  1. **Connection string** (``AZURE_BLOB_CONNECTION_STRING``) — shared-key    access, convenient for local development and testing. 2. **Account URL** (``AZURE_BLOB_ACCOUNT_URL``) + ``DefaultAzureCredential``    — uses Entra ID (Managed Identity on Azure, ``az login`` locally,    workload identity on Kubernetes).  No secrets required in production.  At least one of the two must be set.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_get_user_workspaces_root(identity_id)`
- **说明**：Return the workspaces root for a user: <home>/users/<safe_id>/workspaces/.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_get_backend()`
- **说明**：Read workspace backend from Flask config.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `get_workspace_manager(identity_id)`
- **说明**：Return a :class:`WorkspaceManager` (or Azure subclass) for the given user.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `get_active_workspace_id()`
- **说明**：Read the active workspace ID from the current Flask request's X-Workspace-Id header.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `get_workspace(identity_id)`
- **说明**：Return the active :class:`Workspace` for *identity_id*.  All backends use their configured WorkspaceManager with lazy creation. Ephemeral workspaces use the same local format with bounded retention.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

模块级测试建议：至少覆盖（1）正常路径；（2）非法输入；（3）权限/路径拒绝；（4）上游失败时的 AppError 映射。相关用例见后文测试专章。


### 目录 `py-src/data_formulator/agents`

**目录职责**：专用 Agent、LLM Client、上下文拼装、语义类型、推理日志、网页抓取

本目录内模块共享同一边界：只通过公开函数与相邻层通信。禁止跨层 import 私有 `_` 符号（测试除外）。新增文件必须补测试，并检查是否触发路径安全、日志脱敏、错误协议三条规则。


#### 模块 `py-src/data_formulator/agents/__init__.py`（约 14 行）

**模块文档**：该文件未写模块级 docstring。其职责由文件名与导出符号定义，修改时应补充文档字符串，避免意图漂移。

本文件主要为导出聚合或入口转发，无额外类/函数定义。


#### 模块 `py-src/data_formulator/agents/agent_chart_restyle.py`（约 362 行）

**模块文档**：Chart restyle agent.  A single-turn agent that takes a Vega-Lite spec + a natural-language instruction and returns a modified Vega-Lite spec representing the same chart with the requested style changes applied.  The only HARD rule is that the agent must not touch the spec's `data` block — the caller strips data on input and re-attaches the live rows on output, so the data values, columns, and column names are fixed and reused as-is.  Encoding/mark/aggregation changes are allowed when the user's instruction calls for them (e.g. "swap x and y", "make this a stacked bar"); the prompt's soft guidance is to preserve them by default.  See: design-docs/28-chart-style-refinement-agent.md

符号统计：类 1 个，函数 0 个，模块级常量 2 个。下列说明覆盖全部符号。

**常量**：

- **常量 `_AGENT_ID`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `SYSTEM_PROMPT`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

##### 类 `ChartRestyleAgent`
- **基类**：object。
- **字段标注**：无模块级注解字段（可能使用 dataclass / 动态属性）。
- **方法数**：5。
- **文档字符串**：Single LLM call to produce a restyled Vega-Lite spec.
  - **`__init__(self, client, language_instruction)`**：内部实现细节见源码。
  - **`run(self, vl_spec, instruction, chart_type, data_sample, style_reference_spec)`**：Generate a restyled spec.  Args:     vl_spec: The current Vega-Lite spec (with `data` already stripped).     instruction: The user's natural-language style instruction.     chart_type: The chart template label (e.g. "Bar 主循环入口：组装上下文、调用 LLM、划分 inspection/action、回灌 observation、在无 action 时以纯文本 completion 结束。
  - **`_sanitize_config_ui(self, raw)`**：Validate the LLM-authored configUI array into a clean list.  Each control is a declarative "write value at path" knob — there is no code. We validate the path (non-empty, no prototype-polluting segments) and the per-type
  - **`_enforce_guardrails(self, original, candidate)`**：Apply post-hoc guardrails to a candidate spec.  Hard guardrail: strip any `data` block the model emitted. The caller re-attaches live data; we don't want the model to invent or filter rows.  Field-binding changes are NOT
  - **`_collect_field_bindings(self, spec)`**：Collect a flat map of (path -> field name) for every encoding.<channel>.field anywhere in the spec (top-level + nested under `layer`, `concat`, `vconcat`, `hconcat`, `spec`, `repeat`).
  设计约束：公开方法必须返回安全数据（无密钥、无绝对路径）；异常应转换为 AppError 或 ToolResult 可恢复错误，而不是把 Python 堆栈直接交给前端。

模块级测试建议：至少覆盖（1）正常路径；（2）非法输入；（3）权限/路径拒绝；（4）上游失败时的 AppError 映射。相关用例见后文测试专章。


#### 模块 `py-src/data_formulator/agents/agent_code_explanation.py`（约 222 行）

**模块文档**：该文件未写模块级 docstring。其职责由文件名与导出符号定义，修改时应补充文档字符串，避免意图漂移。

符号统计：类 1 个，函数 0 个，模块级常量 3 个。下列说明覆盖全部符号。

**常量**：

- **常量 `_AGENT_ID`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `SYSTEM_PROMPT`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `EXAMPLE`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

##### 类 `CodeExplanationAgent`
- **基类**：object。
- **字段标注**：无模块级注解字段（可能使用 dataclass / 动态属性）。
- **方法数**：2。
- **职责推断**：该类位于对应模块中，承担 `CodeExplanationAgent` 所表示的领域对象或服务。其方法覆盖构造、序列化、领域操作与错误处理；调用方应通过公开方法交互，避免依赖私有实现细节。
  - **`__init__(self, client, workspace, language_instruction)`**：内部实现细节见源码。
  - **`run(self, input_tables, code, n)`**：内部实现细节见源码。 主循环入口：组装上下文、调用 LLM、划分 inspection/action、回灌 observation、在无 action 时以纯文本 completion 结束。
  设计约束：公开方法必须返回安全数据（无密钥、无绝对路径）；异常应转换为 AppError 或 ToolResult 可恢复错误，而不是把 Python 堆栈直接交给前端。

模块级测试建议：至少覆盖（1）正常路径；（2）非法输入；（3）权限/路径拒绝；（4）上游失败时的 AppError 映射。相关用例见后文测试专章。


#### 模块 `py-src/data_formulator/agents/agent_data_load.py`（约 237 行）

**模块文档**：该文件未写模块级 docstring。其职责由文件名与导出符号定义，修改时应补充文档字符串，避免意图漂移。

符号统计：类 1 个，函数 0 个，模块级常量 3 个。下列说明覆盖全部符号。

**常量**：

- **常量 `_AGENT_ID`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `SYSTEM_PROMPT`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `EXAMPLES`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

##### 类 `DataLoadAgent`
- **基类**：object。
- **字段标注**：无模块级注解字段（可能使用 dataclass / 动态属性）。
- **方法数**：2。
- **职责推断**：该类位于对应模块中，承担 `DataLoadAgent` 所表示的领域对象或服务。其方法覆盖构造、序列化、领域操作与错误处理；调用方应通过公开方法交互，避免依赖私有实现细节。
  - **`__init__(self, client, workspace, language_instruction, model_info)`**：内部实现细节见源码。
  - **`run(self, input_data, n)`**：内部实现细节见源码。 主循环入口：组装上下文、调用 LLM、划分 inspection/action、回灌 observation、在无 action 时以纯文本 completion 结束。
  设计约束：公开方法必须返回安全数据（无密钥、无绝对路径）；异常应转换为 AppError 或 ToolResult 可恢复错误，而不是把 Python 堆栈直接交给前端。

模块级测试建议：至少覆盖（1）正常路径；（2）非法输入；（3）权限/路径拒绝；（4）上游失败时的 AppError 映射。相关用例见后文测试专章。


#### 模块 `py-src/data_formulator/agents/agent_data_loading_chat.py`（约 2378 行）

**模块文档**：Conversational data loading agent.  General-purpose conversational agent that can: - Extract tables from images / text / files - Execute Python code in a sandboxed environment - Show inline table previews - Prepare tables for user-confirmed loading

符号统计：类 1 个，函数 4 个，模块级常量 4 个。下列说明覆盖全部符号。

**常量**：

- **常量 `_AGENT_ID`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `PROBE_TURN_BUDGET`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `SYSTEM_PROMPT`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `TOOLS`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

##### 函数 `_secure_filename(name)`
- **说明**：Sanitise a user-supplied filename to prevent path traversal.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_unique_scratch_filename(scratch_jail, filename)`
- **说明**：Return a scratch filename that does not collide with an existing file.  If ``filename`` already exists in scratch, append ``-1``, ``-2``, … before the extension until a free name is found. Prevents multiple fetches/writes that share a URL basename (e.g. several 'press-release-webcast.html') from overwriting each other. Returns the sanitized filename unchanged when there is no conflict.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_summarize_catalog_shape(tables)`
- **说明**：Return ``(table_count, distinct_folder_count)`` for a catalog.  Folder count is 0 when no table has a hierarchical ``path`` (depth >= 2); flat catalogs report 0 folders so the summary stays terse.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_build_connector_summary_block(user_home)`
- **说明**：Render a compact directory of cached connector catalogs.  Only shows source IDs with table counts (and folder counts when the catalog is hierarchical). The agent is expected to call ``list_data`` for full inventory. Strictly hard-capped at ``max_total_chars``.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 类 `DataLoadingAgent`
- **基类**：无显式基类。
- **字段标注**：无模块级注解字段（可能使用 dataclass / 动态属性）。
- **方法数**：36。
- **文档字符串**：Conversational agent for data loading and extraction.
  - **`__init__(self, client, workspace, available_datasets, language_instruction, knowledge_store, row_limit)`**：内部实现细节见源码。
  - **`stream(self, messages)`**：Stream a conversation turn. Yields SSE event dicts.  Parameters ---------- messages : list[dict]     Chat history in the format:     [{"role": "user", "content": "...", "attachments": [...]}, ...]
  - **`_agentic_loop(self, llm_messages, collected_text, actions, max_iterations)`**：Inner loop extracted so stream_chat can wrap it in a SandboxSession.
  - **`_forced_summary_turn(self, llm_messages, collected_text)`**：Elicit a final, tool-free response after the tool-call limit is reached.  Without this, a long multi-step turn ends the moment the loop hits max_iterations — right after a tool call — and the agent never gets the turn wh
  - **`_call_llm(self, messages, stream)`**：Call the LLM with tool definitions.
  - **`_execute_tool(self, name, args)`**：Execute a tool and return result dict.
  - **`_tool_read_data_memory(self, args)`**：内部实现细节见源码。
  - **`_tool_append_data_memory(self, args)`**：内部实现细节见源码。
  - **`_tool_replace_data_memory(self, args)`**：内部实现细节见源码。
  - **`_tool_read_file(self, args, workspace_jail)`**：Read a file from the workspace with unix-like paging (offset/max_lines) and optional regex search (pattern), confined to the workspace directory.
  - **`_tool_write_file(self, args, scratch_jail)`**：Write a file to scratch directory.
  - **`_tool_list_directory(self, args, workspace_jail)`**：List files in a workspace directory.
  - **`_tool_execute_python(self, args)`**：Execute Python code in sandbox. Auto-saves all DataFrames to scratch/.
  - **`_tool_fetch_url(self, args, scratch_jail)`**：Fetch a public http(s) URL server-side and save the raw payload to scratch/.  fetch_url does NOT parse content — it only gets the URL into scratch so the agent can then read it (read_file, paged) or process it (execute_p
  - **`_tool_show_user_data_preview(self, args, scratch_jail)`**：Unified data preview. To load from a connected source (including the built-in 'sample_datasets'), use propose_load_plan instead.
  - **`_preview_saved_dfs(self, df_names, scratch_jail)`**：Preview DataFrames auto-saved by execute_python.
  - **`_preview_inline_tables(self, tables, scratch_jail)`**：Preview inline CSV tables (from text/image extraction).
  - **`_preview_scratch_files(self, scratch_files, scratch_dir)`**：Read scratch CSV files and build preview actions.
  - **`_tool_list_data(self, args)`**：Browse the catalog hierarchy.  Three modes:   * no args                       → per-source summary   * source_id only                → top-level entries of that source   * source_id + path              → direct children 
  - **`_tool_find_data(self, args)`**：Regex search across cached catalogs.  ``scope`` accepts: 'all' (default), 'workspace', 'connected', '<source_id>', or '<source_id>:<path/segments>'. The path-scoped form restricts catalog search to a subtree.  Workspace 
  - **`_tool_describe_data(self, args)`**：Read detailed metadata for one table.  Delegates to context handler.
  - **`_resolve_catalog_path(self, source_id, table_key)`**：Return the catalog ``path`` for a table_key, or ``None`` if unknown.  Used by ``probe_data`` to turn the model-facing ``table_key`` into the loader-facing catalog path that ``probe``/``get_metadata`` expect.
  - **`_tool_probe_data(self, args)`**：Run a bounded SPJQ probe on one connected-source table (design 37 §4.2).  Resolves the live loader mid-turn, maps ``table_key`` → catalog path, and delegates to ``loader.probe``. Guarded by a per-turn budget so a chatty 
  - **`_tool_propose_load_plan(self, args)`**：Produce a structured load plan action for frontend rendering.  Candidates are validated against the cached catalog before they leave this turn.  If *every* candidate fails to resolve, we return a recoverable error so the
  - **`_connectors_disabled(self)`**：True when external data connectors are turned off for this deployment (e.g. ephemeral / --disable-database). In that mode there are NO database/cloud connectors to offer — only file upload and the built-in sample dataset
  - **`_tool_list_connectors(self, args)`**：List creatable connector TYPES with high-level metadata only.  The available set is deployment-dependent (missing dependencies and external plugins both change it), so the model cannot know it a priori — it must call thi
  - **`_tool_describe_connector(self, args)`**：Return full setup detail (params + auth) for ONE connector type.
  - **`_tool_propose_connection(self, args)`**：Emit a connect_form action so the UI renders an inline setup form.  The action carries source_type + prefilled (values the user provided this conversation, which may include credentials they chose to share). The frontend
  - **`_normalize_load_plan_candidate(self, candidate)`**：Resolve a model-proposed candidate into frontend import shape.  The model sees catalog names and stable table keys, but each loader may require a different opaque import id.  Superset, for example, must be loaded by nume
  - **`_known_source_ids(self)`**：Return the set of cached source_ids the agent can legitimately use.
  - **`_format_valid_sources_hint(self)`**：Compact directory of valid source_ids for the model retry path.
  - **`_lookup_catalog_entry(self, source_id, table_key)`**：内部实现细节见源码。
  - **`_normalize_load_query_filters(filters)`**：内部实现细节见源码。
  - **`_build_system_prompt(self, last_user_text)`**：Build the system prompt with current workspace context.  *last_user_text* is used to search the knowledge store for workflows relevant to the user's current request.  Falls back to a generic query when empty.
  - **`_table_display_name(table)`**：Return a table name from workspace strings or metadata-like objects.
  - **`_convert_message(self, msg)`**：Convert a chat message to LLM message format.
  设计约束：公开方法必须返回安全数据（无密钥、无绝对路径）；异常应转换为 AppError 或 ToolResult 可恢复错误，而不是把 Python 堆栈直接交给前端。

模块级测试建议：至少覆盖（1）正常路径；（2）非法输入；（3）权限/路径拒绝；（4）上游失败时的 AppError 映射。相关用例见后文测试专章。


#### 模块 `py-src/data_formulator/agents/agent_diagnostics.py`（约 147 行）

**模块文档**：Unified diagnostics builder for all agent pipelines.  Centralises the JSON structure returned as ``result['diagnostics']``, ensuring a single schema definition for both back-end construction and front-end consumption (DiagnosticsViewer in MessageSnackbar.tsx).

符号统计：类 1 个，函数 1 个，模块级常量 0 个。下列说明覆盖全部符号。

##### 类 `AgentDiagnostics`
- **基类**：无显式基类。
- **字段标注**：无模块级注解字段（可能使用 dataclass / 动态属性）。
- **方法数**：5。
- **文档字符串**：Captures prompt context once at agent init, then builds diagnostics per request.
  - **`__init__(self, agent_name, model_info, base_system_prompt, agent_coding_rules, language_instruction, assembled_system_prompt)`**：内部实现细节见源码。
  - **`_base(self, messages)`**：内部实现细节见源码。
  - **`for_error(self, messages, error)`**：内部实现细节见源码。
  - **`for_response(self, messages)`**：内部实现细节见源码。
  - **`for_json_only(self, messages)`**：内部实现细节见源码。
  设计约束：公开方法必须返回安全数据（无密钥、无绝对路径）；异常应转换为 AppError 或 ToolResult 可恢复错误，而不是把 Python 堆栈直接交给前端。

##### 函数 `_now()`
- **说明**：模块级函数 `_now` 封装可复用算法或路由处理。参数经过校验后进入核心逻辑，返回结构化结果或通过异常表达业务失败。
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

模块级测试建议：至少覆盖（1）正常路径；（2）非法输入；（3）权限/路径拒绝；（4）上游失败时的 AppError 映射。相关用例见后文测试专章。


#### 模块 `py-src/data_formulator/agents/agent_language.py`（约 188 行）

**模块文档**：Language instruction builder for Agent prompts.  Generates a prompt fragment that constrains LLM output language for user-visible fields while keeping all internal / programmatic fields stable in English.  Two modes are provided:  - **"full"** — detailed field-by-field rules for text-heavy agents   (ChartInsight, InteractiveExplore, ReportGen, CodeExplanation,   DataClean, DataAgent). - **"compact"** — a short 3-sentence instruction for code-generation   agents (DataRec, DataTransformation, DataLoad) so that the extra   text does not distract the model from writing correct code.  Usage:     from data_formulator.agents.agent_language import build_language_instruction      instruction = build_language_instruction("zh")              # full (default)     instruction = build_language_instructio

符号统计：类 0 个，函数 4 个，模块级常量 1 个。下列说明覆盖全部符号。

**常量**：

- **常量 `DEFAULT_LANGUAGE`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

##### 函数 `inject_language_instruction(system_prompt, language_instruction)`
- **说明**：Inject a language instruction block into a system prompt.  Parameters ---------- system_prompt : str     The base system prompt. language_instruction : str     The ``[LANGUAGE INSTRUCTION]`` block (empty string = no-op). marker : str | None     If provided, insert *before* this marker string.     Otherwise the instruction is appended at the end.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `build_language_instruction(language)`
- **说明**：Return a prompt instruction block for the given language code.  Parameters ---------- language : str     BCP-47 primary subtag, e.g. ``"zh"``, ``"en"``, ``"ja"``. mode : ``"full"`` | ``"compact"``     ``"full"``    – detailed field-level rules (for text-heavy agents).     ``"compact"`` – minimal instruction (for code-generation agents).  Returns ``""`` when *language* is ``"en"``, or when it is empty / whitespace-only (which normalises to the default language, English). For unrecognised codes (e
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_build_compact(display_name, extra)`
- **说明**：模块级函数 `_build_compact` 封装可复用算法或路由处理。参数经过校验后进入核心逻辑，返回结构化结果或通过异常表达业务失败。
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_build_full(display_name, extra)`
- **说明**：模块级函数 `_build_full` 封装可复用算法或路由处理。参数经过校验后进入核心逻辑，返回结构化结果或通过异常表达业务失败。
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

模块级测试建议：至少覆盖（1）正常路径；（2）非法输入；（3）权限/路径拒绝；（4）上游失败时的 AppError 映射。相关用例见后文测试专章。


#### 模块 `py-src/data_formulator/agents/agent_simple.py`（约 238 行）

**模块文档**：Lightweight single-turn agents that wrap a system prompt + one LLM call.  Each method takes a ``Client`` instance plus task-specific parameters and returns a plain dict result (no streaming, no workspace access).

符号统计：类 1 个，函数 0 个，模块级常量 4 个。下列说明覆盖全部符号。

**常量**：

- **常量 `_AGENT_ID`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `_NL_FILTER_SYSTEM_PROMPT`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `_WORKSPACE_NAME_SYSTEM_PROMPT`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `_CHART_INTENT_SYSTEM_PROMPT`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

##### 类 `SimpleAgents`
- **基类**：无显式基类。
- **字段标注**：无模块级注解字段（可能使用 dataclass / 动态属性）。
- **方法数**：4。
- **文档字符串**：Collection of lightweight single-turn LLM agents.
  - **`__init__(self, client, language_instruction)`**：内部实现细节见源码。
  - **`nl_to_filter(self, columns, instruction)`**：Translate *instruction* into structured filter conditions.  Parameters ---------- columns : list[dict]     Column schema, each entry ``{"name": ..., "type": ...}``. instruction : str     Natural-language filter descripti
  - **`workspace_name(self, table_names, user_query)`**：Generate a short display name for a workspace.  Returns the display name string (already truncated to 60 chars).
  - **`classify_chart_intent(self, instruction)`**：Classify a chart-prompt as STYLE or DATA.  Used by the encoding-shelf input on Enter to decide whether to send the prompt to the chart-restyle agent (cheap, single LLM call, modifies vlSpec only) or to the full data agen
  设计约束：公开方法必须返回安全数据（无密钥、无绝对路径）；异常应转换为 AppError 或 ToolResult 可恢复错误，而不是把 Python 堆栈直接交给前端。

模块级测试建议：至少覆盖（1）正常路径；（2）非法输入；（3）权限/路径拒绝；（4）上游失败时的 AppError 映射。相关用例见后文测试专章。


#### 模块 `py-src/data_formulator/agents/agent_sort_data.py`（约 125 行）

**模块文档**：该文件未写模块级 docstring。其职责由文件名与导出符号定义，修改时应补充文档字符串，避免意图漂移。

符号统计：类 1 个，函数 0 个，模块级常量 2 个。下列说明覆盖全部符号。

**常量**：

- **常量 `_AGENT_ID`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `SYSTEM_PROMPT`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

##### 类 `SortDataAgent`
- **基类**：object。
- **字段标注**：无模块级注解字段（可能使用 dataclass / 动态属性）。
- **方法数**：2。
- **职责推断**：该类位于对应模块中，承担 `SortDataAgent` 所表示的领域对象或服务。其方法覆盖构造、序列化、领域操作与错误处理；调用方应通过公开方法交互，避免依赖私有实现细节。
  - **`__init__(self, client, language_instruction)`**：内部实现细节见源码。
  - **`run(self, name, values, n)`**：内部实现细节见源码。 主循环入口：组装上下文、调用 LLM、划分 inspection/action、回灌 observation、在无 action 时以纯文本 completion 结束。
  设计约束：公开方法必须返回安全数据（无密钥、无绝对路径）；异常应转换为 AppError 或 ToolResult 可恢复错误，而不是把 Python 堆栈直接交给前端。

模块级测试建议：至少覆盖（1）正常路径；（2）非法输入；（3）权限/路径拒绝；（4）上游失败时的 AppError 映射。相关用例见后文测试专章。


#### 模块 `py-src/data_formulator/agents/agent_starter_questions.py`（约 120 行）

**模块文档**：该文件未写模块级 docstring。其职责由文件名与导出符号定义，修改时应补充文档字符串，避免意图漂移。

符号统计：类 1 个，函数 0 个，模块级常量 2 个。下列说明覆盖全部符号。

**常量**：

- **常量 `_AGENT_ID`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `SYSTEM_PROMPT`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

##### 类 `StarterQuestionsAgent`
- **基类**：object。
- **字段标注**：无模块级注解字段（可能使用 dataclass / 动态属性）。
- **方法数**：2。
- **职责推断**：该类位于对应模块中，承担 `StarterQuestionsAgent` 所表示的领域对象或服务。其方法覆盖构造、序列化、领域操作与错误处理；调用方应通过公开方法交互，避免依赖私有实现细节。
  - **`__init__(self, client, language_instruction)`**：内部实现细节见源码。
  - **`run(self, tables, primary_table, n)`**：Generate a short list of starter exploration questions.  ``tables`` is a list of dicts with ``name``, optional ``description`` and either ``columns`` and/or ``sample_rows``. ``primary_table`` is the name of the table the 主循环入口：组装上下文、调用 LLM、划分 inspection/action、回灌 observation、在无 action 时以纯文本 completion 结束。
  设计约束：公开方法必须返回安全数据（无密钥、无绝对路径）；异常应转换为 AppError 或 ToolResult 可恢复错误，而不是把 Python 堆栈直接交给前端。

模块级测试建议：至少覆盖（1）正常路径；（2）非法输入；（3）权限/路径拒绝；（4）上游失败时的 AppError 映射。相关用例见后文测试专章。


#### 模块 `py-src/data_formulator/agents/agent_utils.py`（约 794 行）

**模块文档**：该文件未写模块级 docstring。其职责由文件名与导出符号定义，修改时应补充文档字符串，避免意图漂移。

符号统计：类 0 个，函数 22 个，模块级常量 0 个。下列说明覆盖全部符号。

##### 函数 `compose_system_prompt(base_prompt)`
- **说明**：Assemble a system prompt by appending coding rules and injecting a language block.  - ``agent_coding_rules`` (already-combined rules text) is appended under an   ``[AGENT CODING RULES]`` preamble when non-empty. - ``language_instruction`` is inserted before ``language_marker`` if the marker   is found in the resulting prompt, otherwise appended at the end.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `attach_reasoning_content(msg, choice_message)`
- **说明**：Attach ``reasoning_content`` from an LLM response to an assistant message dict.  Some reasoning models (currently DeepSeek V4) return a ``reasoning_content`` field alongside the regular ``content``. In multi-turn conversations this field **must** be echoed back in the assistant message, otherwise the API may reject the request or the chain-of-thought context is lost.  For models that do not produce this field the function is a safe no-op.  Args:     msg: The assistant message dict (mutated in-pl
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `accumulate_reasoning_content(accumulated, delta)`
- **说明**：Accumulate ``reasoning_content`` from streaming delta chunks.  In streaming mode, reasoning models (currently DeepSeek V4) deliver ``reasoning_content`` as incremental ``delta.reasoning_content`` chunks, similar to ``delta.content``.  This helper concatenates them.  For non-reasoning models the delta has no such attribute; the accumulator is returned unchanged.  Args:     accumulated: The string accumulated so far, or ``None``.     delta: A streaming ``choice.delta`` object.  Returns:     Update
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_source_table_matches_catalog_entry(source_table, catalog_entry)`
- **说明**：模块级函数 `_source_table_matches_catalog_entry` 封装可复用算法或路由处理。参数经过校验后进入核心逻辑，返回结构化结果或通过异常表达业务失败。
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `build_catalog_metadata_lookups(workspace)`
- **说明**：Build table/column metadata overlays from catalog cache (loader-only).  Returns ------- 4-tuple of (table_desc_cache, col_desc_cache, table_extra_cache, col_meta_cache) where col_meta_cache maps table_name -> {col_name: {"verbose_name": ..., "expression": ...}}.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `format_dataframe_sample_with_budget(df, max_rows, max_chars)`
- **说明**：Return the largest head() sample that fits within a character budget.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `string_to_py_varname(var_str)`
- **说明**：模块级函数 `string_to_py_varname` 封装可复用算法或路由处理。参数经过校验后进入核心逻辑，返回结构化结果或通过异常表达业务失败。
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `field_name_to_ts_variable_name(field_name)`
- **说明**：模块级函数 `field_name_to_ts_variable_name` 封装可复用算法或路由处理。参数经过校验后进入核心逻辑，返回结构化结果或通过异常表达业务失败。
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `infer_ts_datatype(df, name)`
- **说明**：模块级函数 `infer_ts_datatype` 封装可复用算法或路由处理。参数经过校验后进入核心逻辑，返回结构化结果或通过异常表达业务失败。
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `value_handling_func(val)`
- **说明**：process values to make it comparable
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `table_hash(table)`
- **说明**：hash a table, mostly for the purpose of comparison
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `extract_code_from_gpt_response(code_raw, language)`
- **说明**：search for matches and then look for pairs of ```...``` to extract code
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `find_matching_bracket(text, start_index, bracket_type)`
- **说明**：Find the index of the matching closing bracket for JSON objects or arrays.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_strip_json_comments(s)`
- **说明**：Remove single-line ``//`` comments from a JSON-like string.  Correctly skips ``//`` that appears inside quoted strings.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_fix_json_trailing_commas(s)`
- **说明**：Remove trailing commas before ``}`` or ``]``.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_lenient_json_loads(json_str)`
- **说明**：Try ``json.loads`` first; on failure, strip comments / trailing commas and retry.  Returns the parsed object or raises ``ValueError``.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `extract_json_objects(text)`
- **说明**：Extracts JSON objects and arrays from a text string.   Returns a list of parsed JSON objects and arrays.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `supplement_missing_block(client, messages, assistant_content, parsed_json, code_blocks, prefix)`
- **说明**：When model produces only JSON or only code, request the missing block.  Smaller models often fail to produce both JSON + code in a single response.  Rather than retrying the full prompt (which tends to reproduce the same partial output), we ask for *just* the missing piece in a focused single-task follow-up — much higher success rate.  Returns (parsed_json, code_blocks, supplement_content, elapsed_seconds). supplement_content is None if no supplement was needed or it failed.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `get_field_summary(field_name, df, field_sample_size, max_val_chars, column_description, verbose_name, expression)`
- **说明**：模块级函数 `get_field_summary` 封装可复用算法或路由处理。参数经过校验后进入核心逻辑，返回结构化结果或通过异常表达业务失败。
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_format_import_options(opts)`
- **说明**：Format import_options into a concise human-readable provenance line.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `generate_data_summary(input_tables, workspace, include_data_samples, field_sample_size, row_sample_size, sample_char_limit, max_val_chars, table_name_prefix, primary_tables)`
- **说明**：Generate a natural, well-organized summary of input tables by reading workspace parquet files.  All tables (including temp tables) should be in the workspace before calling this function. Use WorkspaceWithTempData context manager to mount temp tables to workspace.  When ``primary_tables`` is provided, the output is structured into tiered sections: - **[PRIMARY TABLE]** / **[PRIMARY TABLES]**: Full detail for the tables the user is focused on. - **[OTHER AVAILABLE TABLES]**: Full detail for the r
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `ensure_output_variable_in_code(code, output_variable)`
- **说明**：Zero-cost regex patch: align code's actual output with the JSON-declared variable.  This is a deterministic local fix (<1ms, 0 tokens) that runs *before* sandbox execution, avoiding an expensive LLM repair round-trip. It scans all top-level assignments (not just the last line, which may be ``print(...)``), picks the last non-library one, and appends an alias ``output_variable = <detected>``.  Returns ------- (patched_code, was_patched, detected_variable_name)
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

模块级测试建议：至少覆盖（1）正常路径；（2）非法输入；（3）权限/路径拒绝；（4）上游失败时的 AppError 映射。相关用例见后文测试专章。


#### 模块 `py-src/data_formulator/agents/agent_utils_sql.py`（约 39 行）

**模块文档**：SQL-related utility functions for agents. These functions are used across multiple agents for DuckDB operations and SQL data summaries.

符号统计：类 0 个，函数 2 个，模块级常量 0 个。下列说明覆盖全部符号。

##### 函数 `sanitize_table_name(table_name)`
- **说明**：Sanitize table name for DuckDB views; see :func:`sanitize_duckdb_sql_table_name`.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `create_duckdb_conn_with_parquet_views(workspace, input_tables)`
- **说明**：Create an in-memory DuckDB connection with a view for each parquet table in the workspace. Input tables are expected to be parquet-backed tables in the datalake (parquet-to-parquet).  Args:     workspace: Workspace instance     input_tables: list of dicts with 'name' key for the table name  Returns:     DuckDB connection with views created for all input tables
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

模块级测试建议：至少覆盖（1）正常路径；（2）非法输入；（3）权限/路径拒绝；（4）上游失败时的 AppError 映射。相关用例见后文测试专章。


#### 模块 `py-src/data_formulator/agents/agent_workflow_distill.py`（约 510 行）

**模块文档**：Workflow distillation agent — extracts a replayable workflow from analysis context.  Given a user-visible analysis context (timeline of events) plus an optional user instruction, this agent calls an LLM to produce a structured Markdown workflow document with YAML front matter suitable for storage in the knowledge base.  Usage::      agent = WorkflowDistillAgent(client)     md_content = agent.run(workflow_context, user_instruction="...")

符号统计：类 1 个，函数 0 个，模块级常量 2 个。下列说明覆盖全部符号。

**常量**：

- **常量 `_AGENT_ID`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `SYSTEM_PROMPT`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

##### 类 `WorkflowDistillAgent`
- **基类**：无显式基类。
- **字段标注**：_LANG_NAMES, RETRY_MARGIN, TRUNCATION_MARKER。
- **方法数**：12。
- **文档字符串**：Distills analysis context into a reusable workflow document.
  - **`__init__(self, client, language_instruction, language_code, timeout_seconds)`**：内部实现细节见源码。
  - **`run(self, context, user_instruction)`**：Distill a workflow document from user-visible session context. 主循环入口：组装上下文、调用 LLM、划分 inspection/action、回灌 observation、在无 action 时以纯文本 completion 结束。
  - **`_prompt_format_kwargs(self)`**：Build template kwargs for SYSTEM_PROMPT.
  - **`_call_with_length_retry(self, messages, soft_limit, hard_limit)`**：Call the LLM, nudging it to stay near *soft_limit* characters.  ``soft_limit`` is advisory guidance: if the first response overshoots it we retry once asking the model to condense. We only ever hard-truncate at ``hard_li
  - **`_truncate_body_to_limit(cls, content, body_limit)`**：If the body of *content* exceeds *body_limit*, truncate it.  Front matter is preserved verbatim; only the body is trimmed. Returns *content* unchanged when within the limit.
  - **`_truncate(value, limit)`**：内部实现细节见源码。
  - **`_truncate_code(code, limit)`**：Return the first *limit* characters of meaningful code lines.
  - **`_render_sample(rows, max_rows)`**：Render a small data sample as a compact one-line-per-row block.  Mirrors the frontend preview — the user sees the same sample.
  - **`_extract_context_summary(cls, context)`**：Render the multi-thread session payload as a compact text block.  ``context['threads']`` is a list of ``{thread_id, events[]}`` dicts (session-scoped distillation, see design-docs/24). Threads are rendered under ``### Th
  - **`_render_events(cls, events)`**：Render a flat event list as a compact text block.
  - **`_call_llm(self, messages)`**：Single LLM call to generate the workflow document.
  - **`_add_fallback_front_matter(content, source_id, today, source_field)`**：Prepend front matter if the LLM didn't include it.
  设计约束：公开方法必须返回安全数据（无密钥、无绝对路径）；异常应转换为 AppError 或 ToolResult 可恢复错误，而不是把 Python 堆栈直接交给前端。

模块级测试建议：至少覆盖（1）正常路径；（2）非法输入；（3）权限/路径拒绝；（4）上游失败时的 AppError 映射。相关用例见后文测试专章。


#### 模块 `py-src/data_formulator/agents/client_utils.py`（约 452 行）

**模块文档**：该文件未写模块级 docstring。其职责由文件名与导出符号定义，修改时应补充文档字符串，避免意图漂移。

符号统计：类 1 个，函数 4 个，模块级常量 0 个。下列说明覆盖全部符号。

##### 函数 `_synthesize_stream(response)`
- **说明**：Yield LiteLLM-style streaming chunks reconstructed from a *buffered* response, so a caller that consumes a stream sees the same data.  Used for Ollama: LiteLLM's Ollama streaming path does not parse native tool calls (it leaks the call as raw JSON ``content`` with ``finish_reason='stop'``), whereas the buffered path parses them correctly. We therefore call Ollama non-streaming and replay the result as a stream.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_extract_json_objects(text)`
- **说明**：Return top-level brace-balanced JSON object substrings found in ``text``.  String-aware (ignores braces inside quoted strings) so it survives code payloads that contain ``{`` / ``}``. Used to recover an action that a weak model emitted as plain content instead of a native tool call.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_match_tool_from_obj(obj, tools, _depth)`
- **说明**：Map a parsed JSON object to ``(tool_name, arguments_dict)`` if it matches one of ``tools``' schemas, else ``None``.  Handles three shapes weak models emit instead of a native tool call:   * nested wrapper — ``{"thought": ..., "action": {"name": "visualize",     "arguments": {...}}}`` (a key points to an object describing the call);   * flat explicit wrapper — ``{"name"/"tool"/"action": "visualize",     "arguments": {...}}`` (the object names the tool directly);   * bare arguments — ``{"code": ..
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_salvage_tool_calls_from_content(response, tools)`
- **说明**：If ``response`` carries an action as JSON *content* but no native ``tool_calls``, rewrite it into a proper tool call in place.  Weak / open models under a long system prompt frequently emit the action (e.g. ``visualize``/``ask_user``) as a JSON object in the assistant content channel rather than as a native function call. This recovers that action so the agent — which only consumes native ``tool_calls`` — can proceed.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 类 `Client`
- **基类**：object。
- **字段标注**：无模块级注解字段（可能使用 dataclass / 动态属性）。
- **方法数**：11。
- **文档字符串**：Returns a LiteLLM client configured for the specified endpoint and model. Supports OpenAI, Azure, Ollama, and other providers via LiteLLM.
  - **`__init__(self, endpoint, model, api_key, api_base, api_version)`**：内部实现细节见源码。
  - **`_strip_image_blocks(self, content)`**：Remove image_url blocks from multimodal content arrays.
  - **`_strip_images_from_messages(self, messages)`**：Create a copy of messages with image_url blocks removed.
  - **`_messages_contain_images(self, messages)`**：Return whether messages contain an image_url content block.
  - **`_is_image_deserialize_error(self, error_text, has_images)`**：Detect provider errors caused by image blocks on text-only models.
  - **`_is_reasoning_effort_error(self, error_text)`**：Detect provider errors caused by an unsupported ``reasoning_effort`` value (e.g. ``"minimal"`` on a model that only accepts ``none/low/medium/high/xhigh``). The provider message reliably mentions the parameter name.  Als
  - **`from_config(cls, model_config)`**：Create a client instance from model configuration.  Args:     model_config: Dictionary containing endpoint, model, api_key, api_base, api_version      Returns:     Client instance for making API calls
  - **`ping(self, timeout)`**：Lightweight connectivity check: send a minimal completion with max_tokens=3 and a short timeout.  Raises on any failure.
  - **`_dispatch(self)`**：Issue the LiteLLM call, transparently handling Ollama streaming.  Ollama's streaming path in LiteLLM fails to parse native tool calls, so for Ollama we always call non-streaming and, when the caller asked for a stream, r
  - **`get_completion(self, messages, stream, reasoning_effort)`**：Send a chat completion request via LiteLLM.  All providers (OpenAI, Azure, Anthropic, etc.) are handled uniformly by LiteLLM.  ``drop_params=True`` ensures unsupported parameters (like ``reasoning_effort`` on non-reasoni
  - **`get_completion_with_tools(self, messages, tools, stream, reasoning_effort)`**：Send a chat completion request with tool definitions via LiteLLM.  Same as ``get_completion`` but accepts ``tools`` (and optional ``tool_choice``, ``parallel_tool_calls``, etc. via ``**kwargs``).
  设计约束：公开方法必须返回安全数据（无密钥、无绝对路径）；异常应转换为 AppError 或 ToolResult 可恢复错误，而不是把 Python 堆栈直接交给前端。

模块级测试建议：至少覆盖（1）正常路径；（2）非法输入；（3）权限/路径拒绝；（4）上游失败时的 AppError 映射。相关用例见后文测试专章。


#### 模块 `py-src/data_formulator/agents/context.py`（约 459 行）

**模块文档**：Shared context builders for agent prompts.  Extracted from DataAgent so that both DataAgent and InteractiveExploreAgent can construct tiered context (primary/other tables, focused thread, peripheral threads) from the same code.

符号统计：类 0 个，函数 9 个，模块级常量 2 个。下列说明覆盖全部符号。

**常量**：

- **常量 `TABLE_SAMPLE_MAX_ROWS`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `TABLE_SAMPLE_CHAR_LIMIT`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

##### 函数 `_get_workspace_metadata_lookups(workspace)`
- **说明**：Return table descriptions, column descriptions, and import options from workspace metadata.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `build_focused_thread_context(focused_thread)`
- **说明**：Build Tier 2: detailed focused thread context.  Each step includes user question, agent reasoning, chart type + encodings, created table metadata, and agent summary.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `build_peripheral_thread_context(other_threads)`
- **说明**：Build Tier 3: minimal peripheral thread context.  One line per step, just display_instruction + chart type.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_table_label(table)`
- **说明**：``Table: <workspace id>``, plus the name the user sees when it differs.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_client_schema_section(table, label)`
- **说明**：Fall back to the client-sent schema when the workspace file can't be read.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `build_lightweight_table_context(input_tables, workspace, primary_tables)`
- **说明**：Build compact table context with schema, metadata, value samples, and rows.  When ``primary_tables`` is provided, tables are grouped into [PRIMARY TABLE(S)] and [OTHER AVAILABLE TABLES] sections.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `handle_inspect_source_data(table_names, input_tables, workspace)`
- **说明**：Handle an inspect_source_data tool call.  Returns a data summary string for the requested tables. Every table includes at most 5 sample rows, bounded by 1000 characters per table so wide tables do not cut off later schema or metadata content.  If a table cannot be read (e.g. not found in workspace), the error is included in the summary instead of crashing the entire request.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_fetch_live_metadata(source_id, path)`
- **说明**：Best-effort live ``get_metadata`` for a catalog table node.  Resolves the connected loader for ``source_id`` within the current request identity and returns its live metadata dict (columns, types, row_count, sample_rows). Returns ``None`` on any failure — callers must degrade to the cache-only view. Never raises.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `handle_read_catalog_metadata(source_id, table_key, workspace)`
- **说明**：Handle a read_catalog_metadata tool call.  Reads the cached catalog entry and overlays user annotations to produce a merged metadata view for the LLM.  Returns a text summary safe for LLM consumption (no credentials or internal paths).  The user home directory is resolved from ``workspace.user_home``.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

模块级测试建议：至少覆盖（1）正常路径；（2）非法输入；（3）权限/路径拒绝；（4）上游失败时的 AppError 映射。相关用例见后文测试专章。


#### 模块 `py-src/data_formulator/agents/reasoning_log.py`（约 264 行）

**模块文档**：Structured reasoning logger for Agent sessions.  Each ``ReasoningLogger`` instance is bound to one Agent session and writes a JSONL file under ``DATA_FORMULATOR_HOME/agent-logs/<date>/<safe_identity_id>/<session_id>-<agent_type>.jsonl``.  The log level is controlled by the **DF_AGENT_LOG** environment variable:      off      – no-op; no file created, no I/O     on       – structured summaries (counts, latencies, tool names); no full messages     verbose  – full messages content, sanitised via ``log_sanitizer``  Default is ``off``.  Expired logs (> 30 days old) are cleaned up best-effort in a background thread at logger creation time.

符号统计：类 2 个，函数 6 个，模块级常量 2 个。下列说明覆盖全部符号。

**常量**：

- **常量 `_LOG_RETENTION_DAYS`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `_ON_FILTERED_KEYS`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

##### 函数 `_today_str()`
- **说明**：UTC date string for directory partitioning.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_utc_now_iso()`
- **说明**：UTC timestamp in ISO 8601 format.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_parse_log_level()`
- **说明**：Read ``DF_AGENT_LOG`` env var (case-insensitive, default ``off``).
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `get_agent_logs_root()`
- **说明**：Return the system-level root for administrator Agent logs.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_safe_log_filename(session_id, agent_type)`
- **说明**：Return a safe single-component JSONL log filename.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_cleanup_expired_logs(agent_logs_root)`
- **说明**：Delete date sub-directories older than ``_LOG_RETENTION_DAYS``.  Only inspects immediate children whose names look like ``YYYY-MM-DD``. Runs best-effort — failures are logged as warnings and swallowed.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 类 `ReasoningLogger`
- **基类**：无显式基类。
- **字段标注**：无模块级注解字段（可能使用 dataclass / 动态属性）。
- **方法数**：7。
- **文档字符串**：Structured JSONL logger for a single Agent session.  Usage::      with ReasoningLogger(identity_id, "DataAgent", session_id) as rlog:         rlog.log("session_start", user_question="...", model="gpt-4o")         ...         rlog.log("session_end", status="success")
  - **`__init__(self, identity_id, agent_type, session_id)`**：内部实现细节见源码。
  - **`log(self, step_type)`**：Append one JSON line to the log.  In ``on`` mode, keys listed in ``_ON_FILTERED_KEYS`` (e.g. ``messages``) are stripped defensively so that callers cannot accidentally write full conversation content.  In ``verbose`` mod
  - **`close(self)`**：Close the underlying file descriptor (idempotent).
  - **`__enter__(self)`**：内部实现细节见源码。
  - **`__exit__(self, exc_type, exc_val, exc_tb)`**：内部实现细节见源码。
  - **`_ensure_fd_for_today(self)`**：Open today's log file, rotating date partitions when needed.
  - **`_sanitize_verbose(kwargs)`**：Sanitise *kwargs* for ``verbose`` mode.  First runs ``sanitize_params`` on the top-level dict (catches ``api_key``, ``token``, etc. passed as direct kwargs), then recurses into nested dicts and lists.
  设计约束：公开方法必须返回安全数据（无密钥、无绝对路径）；异常应转换为 AppError 或 ToolResult 可恢复错误，而不是把 Python 堆栈直接交给前端。

##### 类 `_NullReasoningLogger`
- **基类**：ReasoningLogger。
- **字段标注**：无模块级注解字段（可能使用 dataclass / 动态属性）。
- **方法数**：3。
- **文档字符串**：No-op logger used when identity or log storage is unavailable.
  - **`__init__(self)`**：内部实现细节见源码。
  - **`log(self, step_type)`**：内部实现细节见源码。
  - **`close(self)`**：内部实现细节见源码。
  设计约束：公开方法必须返回安全数据（无密钥、无绝对路径）；异常应转换为 AppError 或 ToolResult 可恢复错误，而不是把 Python 堆栈直接交给前端。

模块级测试建议：至少覆盖（1）正常路径；（2）非法输入；（3）权限/路径拒绝；（4）上游失败时的 AppError 映射。相关用例见后文测试专章。


#### 模块 `py-src/data_formulator/agents/semantic_types.py`（约 463 行）

**模块文档**：============================================================================= SEMANTIC TYPE SYSTEM  (Python mirror of the TypeScript registry) =============================================================================  The **source of truth** for semantic types lives in the flint-chart library (npm package `flint-chart`, repo microsoft/flint-chart):     packages/flint-js/src/core/type-registry.ts  This file mirrors the registered types and provides:   1. String constants for every type in the TS TYPE_REGISTRY   2. Classification sets (measures, temporal, categorical, etc.)   3. Prompt generation for the DataLoadAgent LLM call   4. VL-type mapping + name-heuristic inference for create_vl_plots.py   5. Legacy compatibility list  When a type is added/removed in the TS registry, update this

符号统计：类 0 个，函数 10 个，模块级常量 45 个。下列说明覆盖全部符号。

**常量**：

- **常量 `DATETIME`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `DATE`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `TIME`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `TIMESTAMP`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `YEAR`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `QUARTER`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `MONTH`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `WEEK`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `DAY`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `HOUR`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `YEAR_MONTH`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `YEAR_QUARTER`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `YEAR_WEEK`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `DECADE`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `DURATION`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `AMOUNT`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `PRICE`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `QUANTITY`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `TEMPERATURE`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `PERCENTAGE`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `PROFIT`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `PERCENTAGE_CHANGE`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `SENTIMENT`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `CORRELATION`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `COUNT`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `NUMBER`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `RANK`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `SCORE`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `ID`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `LATITUDE`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `LONGITUDE`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `COUNTRY`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `STATE`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `CITY`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `REGION`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `ADDRESS`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `ZIP_CODE`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `CATEGORY`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `NAME`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `STATUS`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `BOOLEAN`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `DIRECTION`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `RANGE`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `UNKNOWN`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `LEGACY_SEMANTIC_TYPES`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

##### 函数 `is_measure_type(semantic_type)`
- **说明**：Check if a semantic type is a true measure (suitable for quantitative encoding).
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `is_timeseries_type(semantic_type)`
- **说明**：Check if a semantic type is suitable for time-series X axis.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `is_categorical_type(semantic_type)`
- **说明**：Check if a semantic type is categorical (suitable for color/grouping).
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `is_ordinal_type(semantic_type)`
- **说明**：Check if a semantic type is ordinal (has inherent order).
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `is_geo_type(semantic_type)`
- **说明**：Check if a semantic type is geographic.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `is_non_measure_numeric(semantic_type)`
- **说明**：Check if a semantic type is numeric but should not be aggregated.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `is_signed_measure(semantic_type)`
- **说明**：Check if a semantic type is a signed measure (can go negative).
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `generate_semantic_types_prompt()`
- **说明**：Generate the semantic types section for the LLM prompt.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `get_vl_type(semantic_type)`
- **说明**：Get the Vega-Lite encoding type for a semantic type. Returns 'quantitative', 'ordinal', 'nominal', or 'temporal', or None if unknown.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `infer_vl_type_from_name(column_name)`
- **说明**：Infer a likely Vega-Lite type from a column name using pattern matching. Returns 'quantitative', 'ordinal', 'nominal', 'temporal', or None if no strong signal is found.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

模块级测试建议：至少覆盖（1）正常路径；（2）非法输入；（3）权限/路径拒绝；（4）上游失败时的 AppError 映射。相关用例见后文测试专章。


#### 模块 `py-src/data_formulator/agents/web_utils.py`（约 530 行）

**模块文档**：该文件未写模块级 docstring。其职责由文件名与导出符号定义，修改时应补充文档字符串，避免意图漂移。

符号统计：类 0 个，函数 13 个，模块级常量 3 个。下列说明覆盖全部符号。

**常量**：

- **常量 `DEFAULT_MAX_FETCH_BYTES`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `_BROWSER_HEADERS`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `_CHALLENGE_MARKERS`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

##### 函数 `_is_private_ip(ip_str)`
- **说明**：Check if an IP address is private, internal, or otherwise restricted.  Args:     ip_str: IP address as a string      Returns:     bool: True if IP is private/restricted, False if public
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_validate_url_for_ssrf(url)`
- **说明**：Validate a URL to prevent SSRF attacks.  Performs the following checks: 1. Protocol validation (HTTP/HTTPS only) 2. Private IP blocking  Args:     url: The URL to validate      Returns:     str: The validated URL      Raises:     ValueError: If the URL fails any security checks
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `download_html_content(url, timeout, headers)`
- **说明**：Download HTML content from a given URL with SSRF protection.  This function implements comprehensive SSRF protection: 1. Protocol validation (HTTP/HTTPS only) 2. Private IP blocking (before request) 3. Redirect validation (validates all redirect destinations) 4. Timeout limits (prevents slowloris attacks) 5. Logging of all accessed URLs (for security auditing)  Args:     url (str): The URL to download HTML from     timeout (int): Request timeout in seconds (default: 30, max: 60)     headers (dic
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `html_to_text(html_content, remove_scripts, remove_styles)`
- **说明**：Convert HTML content to readable text by extracting and cleaning the text content.  Args:     html_content (str): HTML content as a string     remove_scripts (bool): Whether to remove script tags (default: True)     remove_styles (bool): Whether to remove style tags (default: True)      Returns:     str: Clean, readable text content
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `get_html_title(html_content)`
- **说明**：Extract the title from HTML content.  Args:     html_content (str): HTML content as a string      Returns:     str or None: The title if found, None otherwise
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `get_html_meta_description(html_content)`
- **说明**：Extract the meta description from HTML content.  Args:     html_content (str): HTML content as a string      Returns:     str or None: The meta description if found, None otherwise
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_configured_max_fetch_bytes()`
- **说明**：Per-file scratch cap from the server's CLI_ARGS, falling back to the default.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_ssrf_safe_session()`
- **说明**：Create a requests.Session that re-validates every request/redirect for SSRF.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `fetch_url_bytes(url, timeout, max_bytes, headers)`
- **说明**：Fetch a remote resource (HTML page or data file) with SSRF protection and a size cap.  Unlike ``download_html_content`` this does not assume the response is HTML; it returns the raw bytes plus metadata so the caller can decide how to interpret the content (e.g. CSV / JSON / Excel data file vs. an HTML page to scrape).  Args:     url: The URL to fetch.     timeout: Request timeout in seconds (capped at 60).     max_bytes: Maximum number of bytes to read from the response body.     headers: Option
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `extract_tables_from_html(html_content, max_tables)`
- **说明**：Extract HTML ``<table>`` elements into pandas DataFrames.  Args:     html_content: Raw HTML string.     max_tables: Maximum number of tables to return.  Returns:     list[pandas.DataFrame]: Parsed tables (may be empty). Never raises; on failure     returns an empty list.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `playwright_available()`
- **说明**：Return True if the optional ``playwright`` package is importable.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `is_verification_challenge(html_content)`
- **说明**：Heuristically detect a browser/human-verification interstitial (e.g. Cloudflare Turnstile) rather than the real page. Such challenges cannot be cleared by a plain fetch or a headless render; the caller should surface this and stop retrying.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `render_url_with_playwright(url, timeout_ms, wait_ms)`
- **说明**：Render a JavaScript-heavy page with a headless browser and return the final HTML.  This is an OPTIONAL fallback used only when static fetching yields no usable content. The ``playwright`` package (and its browser binaries) must be installed separately::      uv pip install playwright && python -m playwright install chromium  Security note: the target URL is SSRF-validated before navigation, but a headless browser can issue arbitrary sub-resource requests that are NOT individually filtered by thi
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

模块级测试建议：至少覆盖（1）正常路径；（2）非法输入；（3）权限/路径拒绝；（4）上游失败时的 AppError 映射。相关用例见后文测试专章。


### 目录 `py-src/data_formulator/analyst`

**目录职责**：统一 AnalystAgent 外壳与工具工厂

本目录内模块共享同一边界：只通过公开函数与相邻层通信。禁止跨层 import 私有 `_` 符号（测试除外）。新增文件必须补测试，并检查是否触发路径安全、日志脱敏、错误协议三条规则。


#### 模块 `py-src/data_formulator/analyst/__init__.py`（约 50 行）

**模块文档**：Analyst agent — a single user-facing data agent hosting multiple skills.  This package unifies the former ``DataAgent`` (structured-action visualization loop) and ``ReportGenAgent`` (streaming report writer) into one agent shell that loads *skills* on demand. See ``design-docs/35-unified-agent-skills- architecture.md`` for the full design.  Core ideas:   - **Inspection tools** gather information and are parallel-safe; their results     come back to the agent and are never shown to the user. The shell ships a     small core set (``inspect_source_data``, ``execute_python_script``, ``load_skill``); a     loaded skill may contribute additional tools (e.g. ``inspect_chart``).   - **Actions** are committing surfaces — at most one per turn. Each returns an     observation the shell feeds back as 

本文件主要为导出聚合或入口转发，无额外类/函数定义。


#### 模块 `py-src/data_formulator/analyst/agent.py`（约 2157 行）

**模块文档**：AnalystAgent — the unified data analyst agent shell.  This is the single user-facing data agent that replaces the separate ``DataAgent`` (structured-action visualization loop) and ``ReportGenAgent`` (streaming report writer). It hosts a set of **core actions** plus a registry of **skills** that unlock additional **gated actions** on demand. See ``design-docs/35-unified-agent-skills-architecture.md`` and the action turn model in ``design-docs/36-artifact-turn-model.md``.  Architecture (a vanilla tool-calling loop, plus the skills layer):   - **Inspection tools** (``execute_python_script``, ``inspect_source_data``, ``load_skill``,     plus skill-private tools) are called via the tool-calling API to gather     information. Parallel-safe, internal, no side effects.   - **Committing actions** (

符号统计：类 2 个，函数 1 个，模块级常量 5 个。下列说明覆盖全部符号。

**常量**：

- **常量 `_AGENT_ID`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `_CORE_SKILL`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `_SKILL_LOADED_BANNER`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `_SKILL_LOADED_RE`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `SYSTEM_PROMPT`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

##### 函数 `_rescue_unpack_json_strings(data)`
- **说明**：In-place: parse values that are JSON-encoded strings back to objects.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 类 `_StreamingArgExtractor`
- **基类**：无显式基类。
- **字段标注**：无模块级注解字段（可能使用 dataclass / 动态属性）。
- **方法数**：3。
- **文档字符串**：Incrementally extract the decoded string value of a top-level JSON key from a growing tool-call ``arguments`` fragment.  ``feed`` is given the full accumulated arguments so far and returns only the newly-decoded suffix of the target field's value (``""`` while nothing new can be safely decoded yet).
  - **`__init__(self, field)`**：内部实现细节见源码。
  - **`feed(self, args_so_far)`**：内部实现细节见源码。
  - **`_decode(self, args)`**：Return the decoded value-so-far of the field, or ``None`` if the value has not started or a trailing escape is incomplete.
  设计约束：公开方法必须返回安全数据（无密钥、无绝对路径）；异常应转换为 AppError 或 ToolResult 可恢复错误，而不是把 Python 堆栈直接交给前端。

##### 类 `AnalystAgent`
- **基类**：无显式基类。
- **字段标注**：无模块级注解字段（可能使用 dataclass / 动态属性）。
- **方法数**：34。
- **文档字符串**：Unified data analyst agent — core actions + on-demand skills.
  - **`__init__(self, client, workspace, skill_registry, agent_exploration_rules, agent_coding_rules, language_instruction, max_iterations, max_repair_attempts, identity_id)`**：内部实现细节见源码。
  - **`_explore_ns_dir(self)`**：Directory for cross-turn namespace serialisation.
  - **`_legal_actions(self)`**：The set of committing actions currently legal to emit.  Every legal action is owned by a *loaded* skill. ``core`` is always loaded, so its baseline actions are always legal; a gated skill's actions become legal once that
  - **`run(self, input_tables, user_question, focused_thread, other_threads, trajectory, completed_step_count, primary_tables, attached_images, charts, scratch_files, conversation_id)`**：Run the unified analyst loop.  Yields event dicts with ``type`` in:     ``"action"``        – the agent's committed action (for UI)     ``"result"``        – a visualization result (data + chart)     ``"tool_start"`` / ` 主循环入口：组装上下文、调用 LLM、划分 inspection/action、回灌 observation、在无 action 时以纯文本 completion 结束。
  - **`_rehydrate_loaded_skills(self, trajectory)`**：Re-open skill gates for bodies still present in a resumed trajectory.  A skill is "loaded" iff its ``[SKILL LOADED: <name>]`` body is in context. On resume ``_loaded_skills`` has just been reset to ``{core}``, so scan th
  - **`_load_skill_into_context(self, name, trajectory)`**：Load a skill's ``SKILL.md`` body into the trajectory.  Returns ``(ok, message)``. On success the body is appended as a user message and ``name`` is recorded in ``_loaded_skills``; the gated actions it declares become leg
  - **`_build_skill_body_message(self, name)`**：Resolve a skill's body into a ``user`` message *without* appending it.  Returns ``(ok, message, body_msg)``. On success ``name`` is recorded in ``_loaded_skills`` (so the gated actions become legal immediately) and ``bod
  - **`_dispatch_skill_action(self, skill_name, action_type, action, trajectory, iteration, completed_steps, narration)`**：Render a skill's action via ``handle_action`` and return its observation string (or ``None``).  The skill does the *processing* (validate, run, emit events) and yields events back; this method *routes* those events to th
  - **`_route_skill_events(self, gen, iteration, trajectory, completed_steps)`**：The shell's router: a skill yields events to *here* (never straight to the frontend), and this is the single place that decides what to forward upstream — re-yielding each event after enriching it with shell-owned bookke
  - **`_set_action_observation(self, messages, tool_call_id, observation)`**：Feed an action's observation back as its tool-call result.  The committing action was recorded as an assistant tool call answered by an empty placeholder ``tool`` message (see ``_commit_action``); fill that placeholder w
  - **`run_visualize_code(self)`**：Public alias so skills can run visualize code via ``ctx.runtime``.
  - **`register_run_chart(self, transform_result, chart_spec)`**：Register a chart created mid-run so gated skills (e.g. report) can reference and inspect it within the same run.  The entry mirrors the shape the frontend forwards for pre-existing charts (``chart_id`` / ``chart_type`` /
  - **`run_explore_code(self, code, input_tables)`**：Public alias so skills can run explore code via ``ctx.runtime``.
  - **`_run_explore_code(self, code, input_tables)`**：Run explore code in sandbox, capturing stdout.
  - **`_run_visualize_code(self, code, output_variable, chart_spec, field_metadata, field_display_names, display_instruction, title, subtitle, messages)`**：Run visualize code in sandbox and assemble chart.
  - **`_build_system_prompt(self, has_primary_tables, has_focused_thread, has_other_threads, has_attached_images, has_charts)`**：内部实现细节见源码。
  - **`_build_initial_messages(self, input_tables, user_question, focused_thread, other_threads, primary_tables, attached_images, charts, scratch_files)`**：Build the initial messages with 3-tier context.
  - **`_build_focused_thread_context(self, focused_thread)`**：内部实现细节见源码。
  - **`_build_peripheral_thread_context(self, other_threads)`**：内部实现细节见源码。
  - **`_build_available_charts_context(charts)`**：Render the ``[AVAILABLE CHARTS]`` block from the chart descriptors.  Mirrors the legacy report agent's listing (id, type, encodings, table ref) so chart_ids stay stable across the run — the report skill's ``inspect_chart
  - **`_build_lightweight_table_context(self, input_tables, primary_tables)`**：内部实现细节见源码。
  - **`_get_next_action(self, trajectory, input_tables, outer_iteration)`**：Call the LLM with tools, run the inspection tool rounds internally, and surface the single committing action the turn ends with (as an ``agent_action`` event).
  - **`_current_tools(self)`**：The tool set offered this turn: inspection tools (core tools + load_skill + loaded skills' tools) plus the committing **action** tools of loaded skills (core's visualize/delegate always; write_report once the report skil
  - **`_loaded_skill_tool_map(self)`**：Map ``tool_name -> skill instance`` for inspection tools unlocked by loaded skills. Tool names come from the registry's ``tools.json`` specs; the value is the skill processor that handles them.
  - **`_tool_loop(self, messages, max_tool_rounds, max_json_retries, json_retries, llm_calls_in_cycle, rlog, input_tables, outer_iteration)`**：Inner tool-calling loop, wrapped by _get_next_action in a SandboxSession context manager.
  - **`_commit_action(self, action_calls, readonly_calls, messages, content, choice, rlog, outer_iteration, llm_calls_in_cycle)`**：Apply the one-action-per-turn cardinality guard and commit.  A turn ends with exactly one committing action. When the model emits more than one action (or mixes an action with inspection calls in the same response), we t
  - **`_is_transient_error(exc)`**：内部实现细节见源码。
  - **`_open_stream(self, messages, tools)`**：Open a *streaming* LLM call with tool definitions, retrying on transient errors *before* any tokens are consumed.  ``stream=True`` is what makes live report streaming possible: the loop's LLM call always streams, and the
  - **`_stream_llm(self, messages, tools)`**：Stream the LLM call, forwarding any *streaming* action's argument live, and return a reconstructed non-streaming-shaped response for the loop.  The agent owns this generic forwarding envelope (design-docs/36 §5): it accu
  - **`_forward_stream_delta(self, slot, streamers)`**：Forward a streaming action's growing argument as channel ``text_delta``s.  Decides once per tool-call slot whether it is a streaming action (by name, via the registry); if so, emits the ``action`` commitment event the fi
  - **`_strip_images(trajectory)`**：Return a copy of the trajectory with image_url blocks removed.
  - **`_log_session_end(rlog, status, total_iterations, total_llm_calls, session_start_time)`**：Write ``session_end`` to the reasoning log (does not close it).
  - **`_error_event(iteration, message)`**：Build an ``"error"`` event dict for the streaming response.
  - **`_snapshot_dialog(messages)`**：Snapshot the conversation for the Agent Log dialog.
  设计约束：公开方法必须返回安全数据（无密钥、无绝对路径）；异常应转换为 AppError 或 ToolResult 可恢复错误，而不是把 Python 堆栈直接交给前端。

模块级测试建议：至少覆盖（1）正常路径；（2）非法输入；（3）权限/路径拒绝；（4）上游失败时的 AppError 映射。相关用例见后文测试专章。


#### 模块 `py-src/data_formulator/analyst/tools.py`（约 152 行）

**模块文档**：Inspection tools for the analyst agent.  Tools are parallel-safe, internal, side-effect-free capabilities the agent may call freely within a turn to gather information before committing to a single user-visible action. See ``design-docs/35`` §4.1.    - ``execute_python_script`` — run a general-purpose Python script in the     sandbox to inspect/compute (stdout returned).   - ``inspect_source_data`` — schema + stats + sample rows for source tables.   - ``load_skill`` — pull a skill's ``SKILL.md`` body into context, unlocking     its gated actions (progressive disclosure; reading a doc is read-only).  ``inspect_chart`` is a skill-private tool used by report-style skills and is contributed by those skills rather than living in the always-on tool set.

符号统计：类 0 个，函数 2 个，模块级常量 0 个。下列说明覆盖全部符号。

##### 函数 `build_load_skill_tool(skill_names)`
- **说明**：Build the ``load_skill`` tool, constraining ``name`` to known skills.  Loading a skill pulls its ``SKILL.md`` body into context and unlocks the gated actions it declares. Reading a doc is read-only and idempotent, so this is a tool (parallel-safe) rather than a serialized action.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `build_tools(skill_names, extra_tools, action_tools)`
- **说明**：Assemble the tool set exposed to the LLM each turn.  Three groups share the one function-calling surface (see ``design-docs/36``):    * **inspection tools** (``explore`` / ``inspect_source_data`` / a loaded     skill's own tools) — contributed by the always-on ``core`` skill and any     loaded skills, arriving via ``extra_tools``. Parallel-safe, non-committing.   * **``load_skill``** — the progressive-disclosure switch, added here with     its ``name`` enum built from ``skill_names`` (the loadab
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

模块级测试建议：至少覆盖（1）正常路径；（2）非法输入；（3）权限/路径拒绝；（4）上游失败时的 AppError 映射。相关用例见后文测试专章。


### 目录 `py-src/data_formulator/analyst/skills`

**目录职责**：技能注册表与协议类型

本目录内模块共享同一边界：只通过公开函数与相邻层通信。禁止跨层 import 私有 `_` 符号（测试除外）。新增文件必须补测试，并检查是否触发路径安全、日志脱敏、错误协议三条规则。


#### 模块 `py-src/data_formulator/analyst/skills/__init__.py`（约 394 行）

**模块文档**：Skill registry — discovery and eager instantiation of analyst skills.  Each skill lives in its own sub-package under this directory and ships a ``SKILL.md`` with YAML frontmatter (``name`` / ``description`` / ``when_to_use`` / ``always_on`` / ``actions``). At startup the registry scans those frontmatter blocks to build a cheap, always-resident index (tier-1 progressive disclosure) **and** imports each skill's Python code module so the skill instance is always available to the agent.  The distinction is deliberate: a skill's code is always imported and callable; what ``load_skill(name)`` does is flip a *switch* that exposes the skill's tools, opens its action gate, and injects its ``SKILL.md`` body into context — i.e. it controls exposure to the model, not availability of the code.  Convent

符号统计：类 1 个，函数 7 个，模块级常量 4 个。下列说明覆盖全部符号。

**常量**：

- **常量 `SKILLS_DIR`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `SKILL_DOC_NAME`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `TOOLS_FILE_NAME`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `_FM_PATTERN`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

##### 函数 `_parse_front_matter(content)`
- **说明**：Return ``(frontmatter_dict, body)``. Degrades gracefully to ``({}, content)``.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_coerce_name_list(raw)`
- **说明**：Normalize a frontmatter name list (``tools``/``actions``) to a tuple.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_meta_from_frontmatter(raw, fallback_name)`
- **说明**：模块级函数 `_meta_from_frontmatter` 封装可复用算法或路由处理。参数经过校验后进入核心逻辑，返回结构化结果或通过异常表达业务失败。
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 类 `SkillRegistry`
- **基类**：无显式基类。
- **字段标注**：metas, skills, tool_specs, _doc_paths。
- **方法数**：15。
- **文档字符串**：Index of discovered skills, keyed by skill name.  Holds three declarative things per skill, all resolved at build time: the cheap frontmatter (``SkillMeta``), the eagerly-instantiated code module (the *processor*: ``handle_tool`` / ``handle_action``), and the skill's ``tools.json`` schemas (``tool_specs``). The doc *body* is read lazily.
  - **`canonical_name(self, name)`**：Resolve a public skill name, accepting legacy underscore aliases.
  - **`_specs_split(self, name)`**：Partition a skill's ``tool_specs`` into ``(inspection_tools, actions)`` using its frontmatter ``tools:`` / ``actions:`` lists as the authority.  A spec whose function name is declared in ``actions:`` is a committing acti
  - **`names(self)`**：内部实现细节见源码。
  - **`list_metas(self)`**：内部实现细节见源码。
  - **`has(self, name)`**：内部实现细节见源码。
  - **`gated_skill_names(self)`**：Skills that load on demand (not ``always_on``).
  - **`action_owner(self, action)`**：Return the skill name that unlocks ``action``, or ``None`` if no gated skill declares it (i.e. it is a core action).
  - **`render_registry_block(self)`**：Tier-1 progressive-disclosure listing for the base prompt.  One line per gated skill: name, the actions it unlocks, and a short ``when_to_use``/``description``. Bodies are pulled on demand via ``load_skill``; only this c
  - **`load_body(self, name)`**：Return the ``SKILL.md`` body (frontmatter stripped) for ``name``.
  - **`get_skill(self, name)`**：Return the (eagerly-instantiated) skill code module, or ``None`` for an unknown or guidance-only skill.
  - **`tools_for(self, names)`**：Merge the inspection tool specs contributed by the named (loaded) skills.
  - **`action_tools_for(self, names)`**：Render the committing-action tool specs unlocked by the named (loaded) skills.  These are offered alongside the inspection tools each round; the agent partitions the model's response by which tool names are committing ac
  - **`action_required_fields(self, name)`**：Return the required argument names for the action ``name`` (empty if unknown), read from the action schema's ``parameters.required``. Used for a cheap pre-dispatch completeness check.
  - **`action_names(self)`**：All committing-action names declared by any skill's frontmatter ``actions:`` — the universe of committing tool names, used to partition a response's tool calls into inspection tools vs committing actions.
  - **`action_stream_spec(self, action)`**：Return ``(stream_field, stream_channel)`` for a *streaming* action, or ``None`` for a buffered one.  Streaming is a property of the **loop**, not the schema (design-docs/36 §5): a skill declares which of its actions stre
  设计约束：公开方法必须返回安全数据（无密钥、无绝对路径）；异常应转换为 AppError 或 ToolResult 可恢复错误，而不是把 Python 堆栈直接交给前端。

##### 函数 `_instantiate_skill(name)`
- **说明**：Import ``skills/<name>/skill.py`` and call ``get_skill()``.  Returns ``None`` (not an error) for a guidance-only skill with no code module, and logs a warning for a malformed one.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_load_tool_specs(skill_dir)`
- **说明**：Load a skill's declarative tool/action schemas from ``tools.json``.  ``tools.json`` sits next to ``SKILL.md`` and is a flat JSON list of standard OpenAI function-tool specs covering BOTH the skill's inspection tools and its committing actions; which is which is decided by the frontmatter ``tools:`` / ``actions:`` lists. A skill with no ``tools.json`` (e.g. guidance-only) gets an empty list.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `build_registry(skills_dir)`
- **说明**：Scan ``skills_dir`` for ``<name>/SKILL.md``, build the index, eagerly instantiate each skill's code module, and load its ``tools.json`` schemas.
- **算法要点**：扫描 skills/*/SKILL.md + tools.json + skill.py:get_skill()，组装 SkillRegistry。
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_warn_on_name_collisions(registry)`
- **说明**：Warn (don't raise) when skills declare clashing action or tool names.  Two flat namespaces share one function-calling surface: a committing action resolves to a single owner (first declarer wins) and inspection tools are merged into one name-unique list — and since a committing action is *also* a tool call, its name must not clash with an inspection tool name either. A clash means one skill silently shadows another. Today the built-in skills don't collide, so this is a guard for when users drop 
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

模块级测试建议：至少覆盖（1）正常路径；（2）非法输入；（3）权限/路径拒绝；（4）上游失败时的 AppError 映射。相关用例见后文测试专章。


#### 模块 `py-src/data_formulator/analyst/skills/base.py`（约 186 行）

**模块文档**：Skill protocol and shared types for the analyst agent.  A *skill* is a passive plugin the single analyst agent can switch on. It never runs its own agent loop; instead it contributes:   1. a ``SKILL.md`` doc (frontmatter + how-to body) — progressive disclosure,   2. zero or more **tools** the model may call once the skill is loaded,   3. zero or more **gated actions** it unlocks, and   4. **handlers** (``handle_tool`` / ``handle_action``) that perform any      compute / rendering and yield channel-tagged events.  The shell stays skill-agnostic: it merges a loaded skill's tools into the model's tool list, opens the gate for its actions, routes tool calls to ``handle_tool`` and emitted actions to ``handle_action``, and forwards whatever events come back. "Loading" a skill controls only *expo

符号统计：类 4 个，函数 0 个，模块级常量 0 个。下列说明覆盖全部符号。

##### 类 `SkillMeta`
- **基类**：无显式基类。
- **字段标注**：name, description, when_to_use, always_on, tool_names, action_names。
- **方法数**：0。
- **文档字符串**：A skill's frontmatter — the cheap, always-resident registry entry.  Mirrors Anthropic Agent Skills tier-1 disclosure: only ``name`` and ``description`` (plus an optional ``when_to_use``) are kept resident in the base prompt so the model knows *when* to reach for the skill; the body is loaded on demand via the ``load_skill`` tool.
  设计约束：公开方法必须返回安全数据（无密钥、无绝对路径）；异常应转换为 AppError 或 ToolResult 可恢复错误，而不是把 Python 堆栈直接交给前端。

##### 类 `SkillContext`
- **基类**：无显式基类。
- **字段标注**：client, workspace, language_instruction, trajectory, payload, runtime。
- **方法数**：0。
- **文档字符串**：Shared handles + per-turn state passed to a skill handler.  Carries the substrate a handler needs (LLM client, workspace, language instruction) plus the live trajectory and any data the action operates on. Skills read from here rather than reaching into the agent shell.
  设计约束：公开方法必须返回安全数据（无密钥、无绝对路径）；异常应转换为 AppError 或 ToolResult 可恢复错误，而不是把 Python 堆栈直接交给前端。

##### 类 `ToolResult`
- **基类**：无显式基类。
- **字段标注**：text, images。
- **方法数**：0。
- **文档字符串**：Return value of a skill's ``handle_tool``.  ``text`` is fed back to the model as the tool-result message. ``images`` are base64 data-URLs (e.g. a rendered chart) that the shell attaches as a follow-up vision message, since tool-result messages cannot carry image content on most providers.
  设计约束：公开方法必须返回安全数据（无密钥、无绝对路径）；异常应转换为 AppError 或 ToolResult 可恢复错误，而不是把 Python 堆栈直接交给前端。

##### 类 `Skill`
- **基类**：Protocol。
- **字段标注**：无模块级注解字段（可能使用 dataclass / 动态属性）。
- **方法数**：2。
- **文档字符串**：A passive plugin the agent shell exposes once its skill is *loaded*.  A skill never runs its own agent loop. It is a pure **processor**: two handlers that perform any compute / rendering. Its *declarative* surface — metadata (``SKILL.md`` frontmatter → ``SkillMeta``) and the inspection tool / committing action *schemas* (``tools.json``) — lives in data files the registry loads, not on the class. T
  - **`handle_tool(self, name, args, ctx)`**：Execute an inspection tool the model called. ``name`` is one of this skill's ``tools``; ``args`` is the parsed tool arguments. Parallel-safe; returns text (and optional images) for the model to read.
  - **`handle_action(self, action, spec, ctx)`**：Dispatch a committing **action** the model emitted as a tool call: validate the arguments, run any compute / rendering, and yield channel-tagged events as it goes (result / delegate / text_delta / …). It then **returns**
  设计约束：公开方法必须返回安全数据（无密钥、无绝对路径）；异常应转换为 AppError 或 ToolResult 可恢复错误，而不是把 Python 堆栈直接交给前端。

模块级测试建议：至少覆盖（1）正常路径；（2）非法输入；（3）权限/路径拒绝；（4）上游失败时的 AppError 映射。相关用例见后文测试专章。


### 目录 `py-src/data_formulator/analyst/skills/core`

**目录职责**：核心技能：探查工具与 visualize / ask_user

本目录内模块共享同一边界：只通过公开函数与相邻层通信。禁止跨层 import 私有 `_` 符号（测试除外）。新增文件必须补测试，并检查是否触发路径安全、日志脱敏、错误协议三条规则。


#### 模块 `py-src/data_formulator/analyst/skills/core/__init__.py`（约 9 行）

**模块文档**：core skill — always-on baseline tools + actions for the analyst.  ``SKILL.md`` holds the base prompt body (the shell formats it into the system message); ``skill.py`` exposes ``get_skill()`` (the executable handler).

本文件主要为导出聚合或入口转发，无额外类/函数定义。


#### 模块 `py-src/data_formulator/analyst/skills/core/skill.py`（约 346 行）

**模块文档**：core skill — the analyst's always-on baseline capabilities.  Every other skill is optional and gated; ``core`` is ``always_on`` and loaded automatically at the start of each run, so the agent is never truly empty. It contributes the built-in data-inspection **tools** (``explore`` / ``inspect_source_data`` — ``load_skill`` is assembled by the shell because its enum is dynamic) and the always-available **actions** — the committing tool calls the agent acts with (``visualize`` / ``interact``; see ``design-docs/36``).  Each handler does *processing* (validate the action arguments, run/normalize, emit events) and **returns an observation string** that the shell appends to the trajectory as the action's tool-call result — exactly like an inspection tool. There is no control verdict: the agent re

符号统计：类 1 个，函数 1 个，模块级常量 0 个。下列说明覆盖全部符号。

##### 类 `CoreSkill`
- **基类**：无显式基类。
- **字段标注**：无模块级注解字段（可能使用 dataclass / 动态属性）。
- **方法数**：8。
- **文档字符串**：The core skill processor: the ``explore`` / ``inspect_source_data`` tool handlers and the ``visualize`` / ``interact`` action handlers.  Tool/action *schemas* live in ``core/tools.json`` and the skill's metadata in ``SKILL.md`` frontmatter (``load_skill`` is assembled by the shell because its enum is dynamic); this class is purely behaviour — it validates an action's arguments and returns an obser
  - **`handle_tool(self, name, args, ctx)`**：Execute a core inspection tool by delegating to the shell runtime.  (In practice the shell's tool loop intercepts these inline — they need loop-level sandbox state — but implementing them here keeps the skill self-consis
  - **`handle_action(self, action, spec, ctx)`**：内部实现细节见源码。
  - **`_handle_visualize(self, action, ctx)`**：内部实现细节见源码。
  - **`_handle_interact(self, action, ctx)`**：Render a structured question/explanation widget and end the run.  ``interact`` is the one *terminal* action: the agent cannot observe its own question, so there is nothing to feed back. On a valid payload it yields the w
  - **`_format_observation(step_index, display_instruction, code, data, workspace, chart_id)`**：Build the trajectory observation for a successful visualize step.
  - **`_sanitize_clarification_options(cls, raw_options)`**：内部实现细节见源码。
  - **`_sanitize_clarification_questions(cls, raw_questions)`**：内部实现细节见源码。
  - **`_normalize_interact_action(cls, action)`**：Normalize the ``interact`` action to ``{questions: [...]}``.  Subsumes the clarify + explain shapes:   * the native shape carries ``questions: [{text, options?, required?,     responseType?}, ...]`` — clarifications (req
  设计约束：公开方法必须返回安全数据（无密钥、无绝对路径）；异常应转换为 AppError 或 ToolResult 可恢复错误，而不是把 Python 堆栈直接交给前端。

##### 函数 `get_skill()`
- **说明**：Factory used by the registry's eager instantiation.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

模块级测试建议：至少覆盖（1）正常路径；（2）非法输入；（3）权限/路径拒绝；（4）上游失败时的 AppError 映射。相关用例见后文测试专章。


### 目录 `py-src/data_formulator/analyst/skills/data-loading`

**目录职责**：数据加载技能：发现工具与不可变加载计划

本目录内模块共享同一边界：只通过公开函数与相邻层通信。禁止跨层 import 私有 `_` 符号（测试除外）。新增文件必须补测试，并检查是否触发路径安全、日志脱敏、错误协议三条规则。


#### 模块 `py-src/data_formulator/analyst/skills/data-loading/__init__.py`（约 1 行）

**模块文档**：Analyst data-loading skill package.

本文件主要为导出聚合或入口转发，无额外类/函数定义。


#### 模块 `py-src/data_formulator/analyst/skills/data-loading/skill.py`（约 362 行）

**模块文档**：该文件未写模块级 docstring。其职责由文件名与导出符号定义，修改时应补充文档字符串，避免意图漂移。

符号统计：类 1 个，函数 2 个，模块级常量 3 个。下列说明覆盖全部符号。

**常量**：

- **常量 `_PROBE_BUDGET_KEY`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `_CONNECTORS_LISTED_KEY`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `_CONNECTORS_DISABLED_NOTE`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

##### 类 `DataLoadingSkill`
- **基类**：无显式基类。
- **字段标注**：无模块级注解字段（可能使用 dataclass / 动态属性）。
- **方法数**：10。
- **文档字符串**：Read-only connected-source discovery for the unified analyst.
  - **`handle_tool(self, name, args, ctx)`**：内部实现细节见源码。
  - **`handle_action(self, action, spec, ctx)`**：内部实现细节见源码。
  - **`_connectors_disabled()`**：内部实现细节见源码。
  - **`_skill_state(ctx)`**：内部实现细节见源码。
  - **`_list_connectors(self, ctx)`**：内部实现细节见源码。
  - **`_describe_connector(self, args)`**：内部实现细节见源码。
  - **`_propose_connection(self, spec, ctx)`**：内部实现细节见源码。
  - **`_already_loaded_tables(steps, workspace)`**：内部实现细节见源码。
  - **`_propose_data_operation(spec, ctx)`**：内部实现细节见源码。
  - **`_probe_budget(ctx)`**：内部实现细节见源码。
  设计约束：公开方法必须返回安全数据（无密钥、无绝对路径）；异常应转换为 AppError 或 ToolResult 可恢复错误，而不是把 Python 堆栈直接交给前端。

##### 函数 `_source_is_available(source_id)`
- **说明**：Only False when we can positively tell the source is unreachable.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `get_skill()`
- **说明**：模块级函数 `get_skill` 封装可复用算法或路由处理。参数经过校验后进入核心逻辑，返回结构化结果或通过异常表达业务失败。
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

模块级测试建议：至少覆盖（1）正常路径；（2）非法输入；（3）权限/路径拒绝；（4）上游失败时的 AppError 映射。相关用例见后文测试专章。


### 目录 `py-src/data_formulator/analyst/skills/data_loading`

**目录职责**：数据加载技能的兼容目录副本

本目录内模块共享同一边界：只通过公开函数与相邻层通信。禁止跨层 import 私有 `_` 符号（测试除外）。新增文件必须补测试，并检查是否触发路径安全、日志脱敏、错误协议三条规则。


#### 模块 `py-src/data_formulator/analyst/skills/data_loading/skill.py`（约 362 行）

**模块文档**：该文件未写模块级 docstring。其职责由文件名与导出符号定义，修改时应补充文档字符串，避免意图漂移。

符号统计：类 1 个，函数 2 个，模块级常量 3 个。下列说明覆盖全部符号。

**常量**：

- **常量 `_PROBE_BUDGET_KEY`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `_CONNECTORS_LISTED_KEY`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `_CONNECTORS_DISABLED_NOTE`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

##### 类 `DataLoadingSkill`
- **基类**：无显式基类。
- **字段标注**：无模块级注解字段（可能使用 dataclass / 动态属性）。
- **方法数**：10。
- **文档字符串**：Read-only connected-source discovery for the unified analyst.
  - **`handle_tool(self, name, args, ctx)`**：内部实现细节见源码。
  - **`handle_action(self, action, spec, ctx)`**：内部实现细节见源码。
  - **`_connectors_disabled()`**：内部实现细节见源码。
  - **`_skill_state(ctx)`**：内部实现细节见源码。
  - **`_list_connectors(self, ctx)`**：内部实现细节见源码。
  - **`_describe_connector(self, args)`**：内部实现细节见源码。
  - **`_propose_connection(self, spec, ctx)`**：内部实现细节见源码。
  - **`_already_loaded_tables(steps, workspace)`**：内部实现细节见源码。
  - **`_propose_data_operation(spec, ctx)`**：内部实现细节见源码。
  - **`_probe_budget(ctx)`**：内部实现细节见源码。
  设计约束：公开方法必须返回安全数据（无密钥、无绝对路径）；异常应转换为 AppError 或 ToolResult 可恢复错误，而不是把 Python 堆栈直接交给前端。

##### 函数 `_source_is_available(source_id)`
- **说明**：Only False when we can positively tell the source is unreachable.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `get_skill()`
- **说明**：模块级函数 `get_skill` 封装可复用算法或路由处理。参数经过校验后进入核心逻辑，返回结构化结果或通过异常表达业务失败。
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

模块级测试建议：至少覆盖（1）正常路径；（2）非法输入；（3）权限/路径拒绝；（4）上游失败时的 AppError 映射。相关用例见后文测试专章。


### 目录 `py-src/data_formulator/analyst/skills/report`

**目录职责**：报告技能：inspect_chart 与 write_report 流式写作

本目录内模块共享同一边界：只通过公开函数与相邻层通信。禁止跨层 import 私有 `_` 符号（测试除外）。新增文件必须补测试，并检查是否触发路径安全、日志脱敏、错误协议三条规则。


#### 模块 `py-src/data_formulator/analyst/skills/report/__init__.py`（约 9 行）

**模块文档**：report skill — streams a Markdown report from an exploration.  ``SKILL.md`` holds the instructions/action contract; ``skill.py`` exposes ``get_skill()`` (the executable handler, ported from ``agent_report_gen.py``).

本文件主要为导出聚合或入口转发，无额外类/函数定义。


#### 模块 `py-src/data_formulator/analyst/skills/report/skill.py`（约 212 行）

**模块文档**：report skill — turns an exploration into a Markdown report.  The analyst shell decides to write a report (the ``write_report`` **action**), then dispatches here. The model assembles the report in the **main agent loop**: it loads this skill, inspects whatever charts/data it needs via the skill-private ``inspect_chart`` tool (plus the always-on ``inspect_source_data``), and then emits ``write_report`` — a committing tool call carrying the **full Markdown** in its ``report`` argument.  ``write_report`` is the one *streaming* action (``stream_field="report"`` on the ``report`` channel — declared via ``streaming_actions`` below). When the model writes the report as that argument, the **agent loop** forwards it live as incremental ``report``-channel ``text_delta``s as the tokens arrive (design-

符号统计：类 1 个，函数 2 个，模块级常量 2 个。下列说明覆盖全部符号。

**常量**：

- **常量 `_LEAK_SPECIAL_TOKEN`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `_LEAK_TOOLCALL`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

##### 函数 `_strip_leaked_tool_syntax(text)`
- **说明**：Remove leaked harmony special tokens and tool-call headers (with their trailing JSON args) from the report. Clean prose is untouched.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 类 `ReportWritingSkill`
- **基类**：无显式基类。
- **字段标注**：无模块级注解字段（可能使用 dataclass / 动态属性）。
- **方法数**：3。
- **文档字符串**：The report skill processor: the ``inspect_chart`` tool handler and the ``write_report`` action handler.  Tool/action *schemas* live in ``report/tools.json`` and the skill's metadata in ``SKILL.md`` frontmatter; this class is purely behaviour. The ``write_report`` action streams its ``report`` argument on the ``report`` channel; the agent loop owns that forwarding envelope and this handler is the b
  - **`handle_tool(self, name, args, ctx)`**：内部实现细节见源码。
  - **`handle_action(self, action, spec, ctx)`**：内部实现细节见源码。
  - **`_handle_inspect_chart(self, chart_ids, charts)`**：Inspect charts by *reading their data*, not by rendering them.  The agent "reads" a chart from its encodings + sample rows (+ the code that produced it), which it can further interrogate with ``execute_python_script``. T
  设计约束：公开方法必须返回安全数据（无密钥、无绝对路径）；异常应转换为 AppError 或 ToolResult 可恢复错误，而不是把 Python 堆栈直接交给前端。

##### 函数 `get_skill()`
- **说明**：Factory used by the registry's eager instantiation.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

模块级测试建议：至少覆盖（1）正常路径；（2）非法输入；（3）权限/路径拒绝；（4）上游失败时的 AppError 映射。相关用例见后文测试专章。


### 目录 `py-src/data_formulator/auth`

**目录职责**：身份解析、TokenStore、Azure CLI

本目录内模块共享同一边界：只通过公开函数与相邻层通信。禁止跨层 import 私有 `_` 符号（测试除外）。新增文件必须补测试，并检查是否触发路径安全、日志脱敏、错误协议三条规则。


#### 模块 `py-src/data_formulator/auth/__init__.py`（约 3 行）

**模块文档**：该文件未写模块级 docstring。其职责由文件名与导出符号定义，修改时应补充文档字符串，避免意图漂移。

本文件主要为导出聚合或入口转发，无额外类/函数定义。


#### 模块 `py-src/data_formulator/auth/azure_cli.py`（约 40 行）

**模块文档**：该文件未写模块级 docstring。其职责由文件名与导出符号定义，修改时应补充文档字符串，避免意图漂移。

符号统计：类 0 个，函数 2 个，模块级常量 0 个。下列说明覆盖全部符号。

##### 函数 `find_azure_cli()`
- **说明**：模块级函数 `find_azure_cli` 封装可复用算法或路由处理。参数经过校验后进入核心逻辑，返回结构化结果或通过异常表达业务失败。
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `expose_azure_cli()`
- **说明**：模块级函数 `expose_azure_cli` 封装可复用算法或路由处理。参数经过校验后进入核心逻辑，返回结构化结果或通过异常表达业务失败。
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

模块级测试建议：至少覆盖（1）正常路径；（2）非法输入；（3）权限/路径拒绝；（4）上游失败时的 AppError 映射。相关用例见后文测试专章。


#### 模块 `py-src/data_formulator/auth/identity.py`（约 248 行）

**模块文档**：Authentication and identity management for Data Formulator.  Pluggable single-provider model with anonymous fallback::      AUTH_PROVIDER=oidc            → OIDCProvider   → user:<sub>     AUTH_PROVIDER=azure_easyauth  → AzureEasyAuth  → user:<principal>     (not set, localhost)          → single-user     → local:<os_username>     (not set, 0.0.0.0)           → anonymous only  → browser:<uuid>  Security Model: - Local users: Fixed OS-derived identity (single-user localhost only) - Anonymous users: Browser UUID from X-Identity-Id header (prefixed with "browser:") - Authenticated users: Verified identity from a configured AuthProvider (prefixed with "user:") - Namespacing ensures authenticated user data cannot be accessed by spoofing headers

符号统计：类 0 个，函数 7 个，模块级常量 2 个。下列说明覆盖全部符号。

**常量**：

- **常量 `_MAX_IDENTITY_LENGTH`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `_IDENTITY_RE`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

##### 函数 `is_local_mode()`
- **说明**：True when running in single-user localhost mode.  This is the canonical check for features that should only be available when the backend runs on the user's local machine (e.g. local folder data source, native OS dialogs).
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_validate_identity_value(value, source)`
- **说明**：Validate and return a trimmed identity value.  Raises ``ValueError`` if the value is empty, too long, or contains characters that should never appear in an identity string (e.g. path separators, control characters, shell metacharacters).
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `init_auth(app)`
- **说明**：Initialise the authentication subsystem.  Call once after app creation.  Reads ``AUTH_PROVIDER`` to select a provider and ``ALLOW_ANONYMOUS`` to control whether unauthenticated requests are permitted.  When no provider is configured and the server is bound to a loopback address (``127.0.0.1`` / ``localhost``), enables single-user localhost mode with a fixed ``local:<os_username>`` identity.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `get_identity_id()`
- **说明**：Return the namespaced identity for the current request.  Resolution order:  1. Active AuthProvider → ``user:<verified_id>`` 2. Single-user localhost → ``local:<os_username>`` 3. Anonymous fallback (``ALLOW_ANONYMOUS=true``) → ``browser:<uuid>`` 4. Neither → ``ValueError``  Returns:     ``"user:<id>"``, ``"local:<username>"``, or ``"browser:<id>"``  Raises:     ValueError: when no identity can be determined.
- **算法要点**：按本机回环、SSO、浏览器 UUID 三级解析身份，并用正则校验，防止路径穿越式身份伪造。
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `get_auth_result()`
- **说明**：Return the full :class:`AuthResult` for the current request.  Only available after :func:`get_identity_id` authenticated via a provider (i.e. the identity starts with ``user:``).  Returns ``None`` for anonymous / browser identities.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `get_sso_token()`
- **说明**：Return the raw SSO access-token for the current request.  Useful for pass-through to external systems that share the same IdP. Returns ``None`` when the user is anonymous or the provider does not supply a token.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `get_active_provider()`
- **说明**：Return the currently active provider, or ``None`` in anonymous mode.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

模块级测试建议：至少覆盖（1）正常路径；（2）非法输入；（3）权限/路径拒绝；（4）上游失败时的 AppError 映射。相关用例见后文测试专章。


#### 模块 `py-src/data_formulator/auth/token_store.py`（约 390 行）

**模块文档**：Unified credential manager for all third-party systems.  Resolves credentials through a priority chain:   cached → refresh → sso_exchange → delegated → vault → none.  All callers (Agent, DataConnector, routes) use the same interface.

符号统计：类 1 个，函数 0 个，模块级常量 3 个。下列说明覆盖全部符号。

**常量**：

- **常量 `_SSO_NS`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `_SVC_NS`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `_SSO_BLOCKED_NS`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

##### 类 `TokenStore`
- **基类**：无显式基类。
- **字段标注**：无模块级注解字段（可能使用 dataclass / 动态属性）。
- **方法数**：23。
- **文档字符串**：Session-backed credential store with a six-level resolution chain.
  - **`get_access(self, system_id)`**：Return the best available credential for *system_id*.  Returns an access_token string, a credentials dict, or ``None``.
  - **`get_sso_token(self)`**：Return the DF-level SSO access token.
  - **`get_auth_status(self)`**：Batch status check for all configured systems.
  - **`store_service_token(self, system_id, access_token, refresh_token, expires_in, user)`**：Store a token acquired via popup or manual login.
  - **`clear_service_token(self, system_id)`**：Clear cached token AND vault credentials for a system.  Session + vault are always cleared together for explicit disconnect. SSO-backed systems are also blocked from auto-reconnecting in the current browser session until
  - **`clear_session_tokens(self)`**：Clear current-session SSO and service tokens without touching vault.
  - **`block_sso_reconnect(self, system_id)`**：Prevent SSO auto-exchange for a system in this browser session.
  - **`allow_sso_reconnect(self, system_id)`**：Allow SSO auto-exchange again after an explicit login.
  - **`is_sso_reconnect_blocked(self, system_id)`**：Return whether SSO auto-exchange is blocked for this system.
  - **`store_sso_tokens(self, access_token, refresh_token, expires_in, user_info)`**：Store SSO tokens after backend OIDC callback.
  - **`_get_cached(self, system_id)`**：内部实现细节见源码。
  - **`_is_expired(cached)`**：内部实现细节见源码。
  - **`_do_refresh(self, system_id, cached, config)`**：Refresh an expired token. Returns new access_token or None.
  - **`_do_sso_exchange(self, system_id, config)`**：Exchange SSO token for a system-specific token.
  - **`_try_vault(self, system_id, config)`**：Try vault credentials. Returns credentials dict or None.
  - **`_vault_retrieve(self, system_id)`**：内部实现细节见源码。
  - **`_vault_store(self, system_id, credentials)`**：内部实现细节见源码。
  - **`_vault_delete(self, system_id)`**：内部实现细节见源码。
  - **`_refresh_sso(self)`**：Refresh the SSO token using refresh_token.
  - **`_get_auth_config(self, system_id)`**：内部实现细节见源码。
  - **`_all_auth_configs(self)`**：Collect auth_config from all registered Loaders.
  - **`_available_strategies(self, system_id, config)`**：What can the user do to authenticate this system?
  - **`_resolve_env(env_key)`**：内部实现细节见源码。
  设计约束：公开方法必须返回安全数据（无密钥、无绝对路径）；异常应转换为 AppError 或 ToolResult 可恢复错误，而不是把 Python 堆栈直接交给前端。

模块级测试建议：至少覆盖（1）正常路径；（2）非法输入；（3）权限/路径拒绝；（4）上游失败时的 AppError 映射。相关用例见后文测试专章。


### 目录 `py-src/data_formulator/auth/gateways`

**目录职责**：OAuth/OIDC/Kusto 登录回调蓝图

本目录内模块共享同一边界：只通过公开函数与相邻层通信。禁止跨层 import 私有 `_` 符号（测试除外）。新增文件必须补测试，并检查是否触发路径安全、日志脱敏、错误协议三条规则。


#### 模块 `py-src/data_formulator/auth/gateways/__init__.py`（约 3 行）

**模块文档**：该文件未写模块级 docstring。其职责由文件名与导出符号定义，修改时应补充文档字符串，避免意图漂移。

本文件主要为导出聚合或入口转发，无额外类/函数定义。


#### 模块 `py-src/data_formulator/auth/gateways/github_gateway.py`（约 146 行）

**模块文档**：GitHub OAuth authorization-code exchange gateway.  Provides ``/api/auth/github/login`` (redirect to GitHub) and ``/api/auth/github/callback`` (exchange code → token → user info → write Flask session).  The :class:`GitHubOAuthProvider` then reads from this session on subsequent requests.

符号统计：类 0 个，函数 5 个，模块级常量 0 个。下列说明覆盖全部符号。

##### 函数 `_error_redirect(code)`
- **说明**：模块级函数 `_error_redirect` 封装可复用算法或路由处理。参数经过校验后进入核心逻辑，返回结构化结果或通过异常表达业务失败。
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_fetch_primary_email(access_token)`
- **说明**：Fetch the user's primary verified email when /user omits private email.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `github_login()`
- **说明**：Redirect the browser to GitHub's authorization page.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `github_callback()`
- **说明**：Handle the OAuth callback — exchange code for token, fetch user.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `github_logout()`
- **说明**：Clear the GitHub session data.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

模块级测试建议：至少覆盖（1）正常路径；（2）非法输入；（3）权限/路径拒绝；（4）上游失败时的 AppError 映射。相关用例见后文测试专章。


#### 模块 `py-src/data_formulator/auth/gateways/kusto_oauth_gateway.py`（约 252 行）

**模块文档**：Microsoft delegated OAuth flow for Kusto connector access.

符号统计：类 0 个，函数 9 个，模块级常量 3 个。下列说明覆盖全部符号。

**常量**：

- **常量 `_STATE_KEY`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `_KUSTO_HOST_SUFFIXES`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `_LOGIN_HOSTS`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

##### 函数 `_oauth_config(authority_host)`
- **说明**：模块级函数 `_oauth_config` 封装可复用算法或路由处理。参数经过校验后进入核心逻辑，返回结构化结果或通过异常表达业务失败。
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_callback_url()`
- **说明**：模块级函数 `_callback_url` 封装可复用算法或路由处理。参数经过校验后进入核心逻辑，返回结构化结果或通过异常表达业务失败。
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_normalize_cluster(cluster)`
- **说明**：模块级函数 `_normalize_cluster` 封装可复用算法或路由处理。参数经过校验后进入核心逻辑，返回结构化结果或通过异常表达业务失败。
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_cluster_auth_metadata(cluster)`
- **说明**：Return the SDK-compatible resource scope and login endpoint.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_frontend_origin(value)`
- **说明**：模块级函数 `_frontend_origin` 封装可复用算法或路由处理。参数经过校验后进入核心逻辑，返回结构化结果或通过异常表达业务失败。
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_pkce_challenge(verifier)`
- **说明**：模块级函数 `_pkce_challenge` 封装可复用算法或路由处理。参数经过校验后进入核心逻辑，返回结构化结果或通过异常表达业务失败。
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_popup_response(origin, payload)`
- **说明**：模块级函数 `_popup_response` 封装可复用算法或路由处理。参数经过校验后进入核心逻辑，返回结构化结果或通过异常表达业务失败。
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `kusto_login()`
- **说明**：Start an Authorization Code + PKCE flow for the selected cluster.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `kusto_callback()`
- **说明**：Exchange the Microsoft authorization code and notify the opener.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

模块级测试建议：至少覆盖（1）正常路径；（2）非法输入；（3）权限/路径拒绝；（4）上游失败时的 AppError 映射。相关用例见后文测试专章。


#### 模块 `py-src/data_formulator/auth/gateways/oidc_gateway.py`（约 258 行）

**模块文档**：Backend OIDC Confidential Client gateway.  When ``OIDC_CLIENT_SECRET`` is set (or ``AUTH_MODE=backend`` is forced), DF acts as a Confidential Client and handles the full Authorization Code flow server-side.  The browser never sees ``client_secret`` or raw tokens — only a session cookie.  Endpoint URLs are resolved from ``OIDCProvider.get_resolved_config()``, which supports auto-discovery via ``.well-known/openid-configuration``. Manual ``OIDC_*_URL`` env vars are NOT required when the SSO server exposes a standard discovery endpoint.  Callback URL (``/auth/callback``) is shared with the frontend PKCE flow so that only one redirect URI needs to be registered in the IdP.

符号统计：类 0 个，函数 11 个，模块级常量 0 个。下列说明覆盖全部符号。

##### 函数 `_get_oidc_config()`
- **说明**：Return resolved OIDC config from the active OIDCProvider.  The provider runs auto-discovery during ``on_configure()``, so endpoint URLs are available even when the ``OIDC_*_URL`` env vars are not set.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_callback_url()`
- **说明**：模块级函数 `_callback_url` 封装可复用算法或路由处理。参数经过校验后进入核心逻辑，返回结构化结果或通过异常表达业务失败。
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_fetch_userinfo(access_token, userinfo_url)`
- **说明**：模块级函数 `_fetch_userinfo` 封装可复用算法或路由处理。参数经过校验后进入核心逻辑，返回结构化结果或通过异常表达业务失败。
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `oidc_login()`
- **说明**：Redirect user to SSO authorization page.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_error_redirect(code)`
- **说明**：Redirect to the SPA root with an ``auth_error`` query param.  This lets the frontend display a translated, user-friendly message instead of showing raw JSON to end-users.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `oidc_callback()`
- **说明**：Exchange authorization code for tokens (backend confidential flow).
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `oidc_status()`
- **说明**：Check SSO login status.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `oidc_logout()`
- **说明**：Clear current-session tokens (SSO + all services), preserving vault.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `save_delegated_token()`
- **说明**：Receive token from frontend popup and store in TokenStore.  Called after the popup postMessage flow completes.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `clear_service_token(system_id)`
- **说明**：Disconnect from a specific service (clear its cached token).
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `auth_service_status()`
- **说明**：Return authorization status for all configured systems.  Agent calls this before starting analysis. Frontend calls this to show connection indicators.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

模块级测试建议：至少覆盖（1）正常路径；（2）非法输入；（3）权限/路径拒绝；（4）上游失败时的 AppError 映射。相关用例见后文测试专章。


### 目录 `py-src/data_formulator/auth/providers`

**目录职责**：OIDC / GitHub / Azure EasyAuth 提供者

本目录内模块共享同一边界：只通过公开函数与相邻层通信。禁止跨层 import 私有 `_` 符号（测试除外）。新增文件必须补测试，并检查是否触发路径安全、日志脱敏、错误协议三条规则。


#### 模块 `py-src/data_formulator/auth/providers/__init__.py`（约 77 行）

**模块文档**：Auto-discovery registry for AuthProvider subclasses.  On import, every ``.py`` module in this package (except ``base``) is scanned for concrete ``AuthProvider`` subclasses.  Each discovered class is instantiated once to read its ``name`` property, then stored in the registry keyed by that name.  Activation of a specific provider is controlled by the ``AUTH_PROVIDER`` environment variable in ``auth.py`` — discovery only populates the *available* set.

符号统计：类 0 个，函数 3 个，模块级常量 0 个。下列说明覆盖全部符号。

##### 函数 `_discover_providers()`
- **说明**：Scan this package for AuthProvider subclasses and register them.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `get_provider_class(name)`
- **说明**：Return the provider class registered under *name*, or ``None``.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `list_available_providers()`
- **说明**：Return sorted names of all discovered providers.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

模块级测试建议：至少覆盖（1）正常路径；（2）非法输入；（3）权限/路径拒绝；（4）上游失败时的 AppError 映射。相关用例见后文测试专章。


#### 模块 `py-src/data_formulator/auth/providers/azure_easyauth.py`（约 53 行）

**模块文档**：Azure App Service built-in authentication (EasyAuth) provider.  When Data Formulator is deployed on Azure App Service with authentication enabled, Azure verifies the user's identity *before* the request reaches Flask and injects trusted headers:  * ``X-MS-CLIENT-PRINCIPAL-ID`` — user's Object ID (always present) * ``X-MS-CLIENT-PRINCIPAL-NAME`` — display name (optional)  These headers are set by the Azure infrastructure and cannot be forged by end-user clients.

符号统计：类 1 个，函数 0 个，模块级常量 0 个。下列说明覆盖全部符号。

##### 类 `AzureEasyAuthProvider`
- **基类**：AuthProvider。
- **字段标注**：无模块级注解字段（可能使用 dataclass / 动态属性）。
- **方法数**：3。
- **职责推断**：该类位于对应模块中，承担 `AzureEasyAuthProvider` 所表示的领域对象或服务。其方法覆盖构造、序列化、领域操作与错误处理；调用方应通过公开方法交互，避免依赖私有实现细节。
  - **`name(self)`**：内部实现细节见源码。
  - **`authenticate(self, request)`**：内部实现细节见源码。
  - **`get_auth_info(self)`**：内部实现细节见源码。
  设计约束：公开方法必须返回安全数据（无密钥、无绝对路径）；异常应转换为 AppError 或 ToolResult 可恢复错误，而不是把 Python 堆栈直接交给前端。

模块级测试建议：至少覆盖（1）正常路径；（2）非法输入；（3）权限/路径拒绝；（4）上游失败时的 AppError 映射。相关用例见后文测试专章。


#### 模块 `py-src/data_formulator/auth/providers/base.py`（约 87 行）

**模块文档**：Base classes for the pluggable authentication provider system.  AuthProvider subclasses are auto-discovered at startup. Each provider extracts and verifies user identity from an incoming Flask request.

符号统计：类 3 个，函数 0 个，模块级常量 0 个。下列说明覆盖全部符号。

##### 类 `AuthResult`
- **基类**：无显式基类。
- **字段标注**：user_id, display_name, email, raw_token。
- **方法数**：0。
- **文档字符串**：Successful authentication outcome.  ``raw_token`` carries the original access_token so that downstream code (e.g. SSO pass-through to external BI systems) can reuse it without a second authentication round-trip.
  设计约束：公开方法必须返回安全数据（无密钥、无绝对路径）；异常应转换为 AppError 或 ToolResult 可恢复错误，而不是把 Python 堆栈直接交给前端。

##### 类 `AuthProvider`
- **基类**：ABC。
- **字段标注**：无模块级注解字段（可能使用 dataclass / 动态属性）。
- **方法数**：5。
- **文档字符串**：Abstract base for authentication providers.  Lifecycle:     1. ``__init__``  -- read env vars / lightweight setup     2. ``enabled``   -- checked by ``init_auth()``; False → skip     3. ``on_configure(app)`` -- called once after Flask app is ready     4. ``authenticate(request)`` -- called on every incoming request  Return conventions for ``authenticate``:     * ``AuthResult`` → authentication suc
  - **`name(self)`**：Short identifier used in AUTH_PROVIDER env var and logs.
  - **`authenticate(self, request)`**：Try to extract a verified identity from *request*.
  - **`enabled(self)`**：Whether required configuration (env vars, etc.) is present.
  - **`on_configure(self, app)`**：Called once after the Flask app is created (e.g. fetch JWKS).
  - **`get_auth_info(self)`**：Describe this provider to the frontend via ``/api/auth/info``.  The ``action`` field tells the frontend how to initiate login: ``"frontend"`` (OIDC PKCE), ``"redirect"`` (server-side OAuth), ``"form"`` (username/password
  设计约束：公开方法必须返回安全数据（无密钥、无绝对路径）；异常应转换为 AppError 或 ToolResult 可恢复错误，而不是把 Python 堆栈直接交给前端。

##### 类 `AuthenticationError`
- **基类**：Exception。
- **字段标注**：无模块级注解字段（可能使用 dataclass / 动态属性）。
- **方法数**：1。
- **文档字符串**：Raised when credentials are present but verification fails.
  - **`__init__(self, message, provider)`**：内部实现细节见源码。
  设计约束：公开方法必须返回安全数据（无密钥、无绝对路径）；异常应转换为 AppError 或 ToolResult 可恢复错误，而不是把 Python 堆栈直接交给前端。

模块级测试建议：至少覆盖（1）正常路径；（2）非法输入；（3）权限/路径拒绝；（4）上游失败时的 AppError 映射。相关用例见后文测试专章。


#### 模块 `py-src/data_formulator/auth/providers/github_oauth.py`（约 70 行）

**模块文档**：GitHub OAuth 2.0 authentication provider.  GitHub is pure OAuth2 (not OIDC — there is no ``id_token``), so the authorization-code exchange must happen server-side.  This makes it a **stateful** (B-class) provider: the gateway blueprint handles the redirect dance and writes the result into the Flask session; this provider then reads the session on subsequent requests.  Configuration (environment variables)::      GITHUB_CLIENT_ID       — OAuth App client ID  (required)     GITHUB_CLIENT_SECRET   — OAuth App secret      (required)

符号统计：类 1 个，函数 0 个，模块级常量 0 个。下列说明覆盖全部符号。

##### 类 `GitHubOAuthProvider`
- **基类**：AuthProvider。
- **字段标注**：无模块级注解字段（可能使用 dataclass / 动态属性）。
- **方法数**：5。
- **职责推断**：该类位于对应模块中，承担 `GitHubOAuthProvider` 所表示的领域对象或服务。其方法覆盖构造、序列化、领域操作与错误处理；调用方应通过公开方法交互，避免依赖私有实现细节。
  - **`__init__(self)`**：内部实现细节见源码。
  - **`name(self)`**：内部实现细节见源码。
  - **`enabled(self)`**：内部实现细节见源码。
  - **`get_auth_info(self)`**：内部实现细节见源码。
  - **`authenticate(self, request)`**：Read identity from the Flask session (set by the gateway).
  设计约束：公开方法必须返回安全数据（无密钥、无绝对路径）；异常应转换为 AppError 或 ToolResult 可恢复错误，而不是把 Python 堆栈直接交给前端。

模块级测试建议：至少覆盖（1）正常路径；（2）非法输入；（3）权限/路径拒绝；（4）上游失败时的 AppError 映射。相关用例见后文测试专章。


#### 模块 `py-src/data_formulator/auth/providers/oidc.py`（约 395 行）

**模块文档**：OIDC / OAuth2 authentication provider.  Supports both standards-compliant OIDC Identity Providers (with auto-discovery) and plain OAuth2 servers (with manually configured endpoint URLs).  Discovery strategy depends on AUTH_PROVIDER:     AUTH_PROVIDER=oidc   → tries /.well-known/openid-configuration     AUTH_PROVIDER=oauth2 → tries /.well-known/oauth-authorization-server  Minimal configuration::      OIDC_ISSUER_URL   — IdP issuer URL  (e.g. https://keycloak.example.com/realms/main)     OIDC_CLIENT_ID    — Registered client / application ID  When discovery succeeds, all endpoints are auto-discovered. Otherwise, set the endpoints manually::      OIDC_AUTHORIZE_URL  — Authorization endpoint     OIDC_TOKEN_URL      — Token endpoint     OIDC_USERINFO_URL   — UserInfo endpoint  (used for token v

符号统计：类 1 个，函数 1 个，模块级常量 0 个。下列说明覆盖全部符号。

##### 函数 `is_backend_oidc_mode()`
- **说明**：Determine whether backend (Confidential Client) OIDC mode is active.  Auto-detected from ``OIDC_CLIENT_SECRET`` presence. ``AUTH_MODE`` env var overrides auto-detection when explicitly set.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 类 `OIDCProvider`
- **基类**：AuthProvider。
- **字段标注**：无模块级注解字段（可能使用 dataclass / 动态属性）。
- **方法数**：13。
- **文档字符串**：Verify access-tokens via JWKS signature or UserInfo introspection.
  - **`__init__(self)`**：内部实现细节见源码。
  - **`name(self)`**：内部实现细节见源码。
  - **`enabled(self)`**：内部实现细节见源码。
  - **`_build_ssl_context(self)`**：Return an unverified SSL context when OIDC_VERIFY_SSL=false.
  - **`_try_discovery(self, discovery_url)`**：Fetch and validate a discovery document. Returns None on failure.
  - **`on_configure(self, app)`**：内部实现细节见源码。
  - **`get_resolved_config(self)`**：Return resolved OIDC endpoints and credentials.  Values come from auto-discovery (populated during ``on_configure``) with manual env-var overrides taking precedence.  Used by the backend OIDC gateway so it does not need 
  - **`_effective_scopes(self)`**：内部实现细节见源码。
  - **`get_auth_info(self)`**：内部实现细节见源码。
  - **`authenticate(self, request)`**：内部实现细节见源码。
  - **`_authenticate_session(self)`**：Authenticate from server-side session (backend OIDC mode).
  - **`_authenticate_jwt(self, token)`**：Verify JWT signature locally using JWKS.
  - **`_authenticate_userinfo(self, token)`**：Validate token by calling the IdP's UserInfo endpoint.
  设计约束：公开方法必须返回安全数据（无密钥、无绝对路径）；异常应转换为 AppError 或 ToolResult 可恢复错误，而不是把 Python 堆栈直接交给前端。

模块级测试建议：至少覆盖（1）正常路径；（2）非法输入；（3）权限/路径拒绝；（4）上游失败时的 AppError 映射。相关用例见后文测试专章。


### 目录 `py-src/data_formulator/auth/vault`

**目录职责**：本地加密凭证库

本目录内模块共享同一边界：只通过公开函数与相邻层通信。禁止跨层 import 私有 `_` 符号（测试除外）。新增文件必须补测试，并检查是否触发路径安全、日志脱敏、错误协议三条规则。


#### 模块 `py-src/data_formulator/auth/vault/__init__.py`（约 111 行）

**模块文档**：Credential Vault factory — returns the global vault instance.  Key resolution (first match wins):  1. ``CREDENTIAL_VAULT_KEY`` env var  — explicit key (server deployments) 2. ``DATA_FORMULATOR_HOME/.vault_key`` file — auto-generated on first run 3. Neither → vault disabled, plugins fall back to session-only storage  For local single-user mode the vault is **zero-config**: a Fernet key is auto-generated on first access and persisted to the data directory.  Server admins who want deterministic keys (e.g. for Docker volume mounts) can set ``CREDENTIAL_VAULT_KEY`` explicitly.

符号统计：类 0 个，函数 3 个，模块级常量 0 个。下列说明覆盖全部符号。

##### 函数 `get_data_formulator_home()`
- **说明**：Lazy import to avoid circular deps at module load time.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_resolve_key(home)`
- **说明**：Resolve the Fernet encryption key.  Priority: 1. CREDENTIAL_VAULT_KEY env var (explicit, for server deployments) 2. Auto-generated key file at ``home/.vault_key``
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `get_credential_vault()`
- **说明**：Return the global :class:`CredentialVault` singleton.  Returns ``None`` when: - Data connectors are disabled (nothing needs credentials) - Key resolution fails
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

模块级测试建议：至少覆盖（1）正常路径；（2）非法输入；（3）权限/路径拒绝；（4）上游失败时的 AppError 映射。相关用例见后文测试专章。


#### 模块 `py-src/data_formulator/auth/vault/base.py`（约 40 行）

**模块文档**：Abstract interface for credential storage backends.

符号统计：类 1 个，函数 0 个，模块级常量 0 个。下列说明覆盖全部符号。

##### 类 `CredentialVault`
- **基类**：ABC。
- **字段标注**：无模块级注解字段（可能使用 dataclass / 动态属性）。
- **方法数**：4。
- **文档字符串**：Encrypted per-user credential storage.  Credentials are keyed by ``(user_identity, source_key)``:  - *user_identity* comes from :func:`auth.get_identity_id`   (e.g. ``"user:alice@corp.com"`` or ``"browser:uuid-123"``) - *source_key* is the plugin ID (e.g. ``"superset"``, ``"metabase"``)
  - **`store(self, user_id, source_key, credentials)`**：Store (or overwrite) credentials for *(user_id, source_key)*.
  - **`retrieve(self, user_id, source_key)`**：Retrieve credentials, or ``None`` if absent / undecryptable.
  - **`delete(self, user_id, source_key)`**：Delete credentials.  No-op if nothing stored.
  - **`list_sources(self, user_id)`**：Return source_keys that have stored credentials for *user_id*.
  设计约束：公开方法必须返回安全数据（无密钥、无绝对路径）；异常应转换为 AppError 或 ToolResult 可恢复错误，而不是把 Python 堆栈直接交给前端。

模块级测试建议：至少覆盖（1）正常路径；（2）非法输入；（3）权限/路径拒绝；（4）上游失败时的 AppError 映射。相关用例见后文测试专章。


#### 模块 `py-src/data_formulator/auth/vault/local_vault.py`（约 95 行）

**模块文档**：SQLite + Fernet encrypted credential vault.  Storage location: ``DATA_FORMULATOR_HOME/credentials.db``  Generate a Fernet key::      python -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())"

符号统计：类 1 个，函数 0 个，模块级常量 0 个。下列说明覆盖全部符号。

##### 类 `LocalCredentialVault`
- **基类**：CredentialVault。
- **字段标注**：无模块级注解字段（可能使用 dataclass / 动态属性）。
- **方法数**：6。
- **文档字符串**：Fernet-encrypted credentials backed by a local SQLite database.
  - **`__init__(self, db_path, encryption_key)`**：内部实现细节见源码。
  - **`_init_db(self)`**：内部实现细节见源码。
  - **`store(self, user_id, source_key, credentials)`**：内部实现细节见源码。
  - **`retrieve(self, user_id, source_key)`**：内部实现细节见源码。
  - **`delete(self, user_id, source_key)`**：内部实现细节见源码。
  - **`list_sources(self, user_id)`**：内部实现细节见源码。
  设计约束：公开方法必须返回安全数据（无密钥、无绝对路径）；异常应转换为 AppError 或 ToolResult 可恢复错误，而不是把 Python 堆栈直接交给前端。

模块级测试建议：至少覆盖（1）正常路径；（2）非法输入；（3）权限/路径拒绝；（4）上游失败时的 AppError 映射。相关用例见后文测试专章。


### 目录 `py-src/data_formulator/data_loader`

**目录职责**：外部数据加载器实现与插件扫描

本目录内模块共享同一边界：只通过公开函数与相邻层通信。禁止跨层 import 私有 `_` 符号（测试除外）。新增文件必须补测试，并检查是否触发路径安全、日志脱敏、错误协议三条规则。


#### 模块 `py-src/data_formulator/data_loader/__init__.py`（约 365 行）

**模块文档**：Modular data-loader registry.  Two loader sources:  1. **Built-in** — declared in ``_LOADER_SPECS`` (this file).  Each loader    is independently imported via try/except so that a missing dependency    only disables that one loader.  2. **External plugins** — Python files matching ``*_data_loader.py``    found in the plugin directory.  Resolution order:     1. ``DF_PLUGIN_DIR`` env var — explicit override (useful for       team-shared dirs, read-only mounts, dev iteration).    2. ``DATA_FORMULATOR_HOME/plugins`` — the default location,       consistent with every other DF artifact.    3. ``~/.data_formulator/plugins/`` — final fallback when       ``DATA_FORMULATOR_HOME`` is unset.     Any ``ExternalDataLoader`` subclass found in such a file is    auto-registered.  If a plugin key collides 

符号统计：类 0 个，函数 8 个，模块级常量 0 个。下列说明覆盖全部符号。

##### 函数 `_scan_package_loaders()`
- **说明**：Import built-in loaders from ``_LOADER_SPECS``.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_resolve_plugin_dir()`
- **说明**：Resolve the plugin directory.  Order: ``DF_PLUGIN_DIR`` (explicit override) > ``DATA_FORMULATOR_HOME/plugins`` (default) > ``~/.data_formulator/plugins`` (fallback).
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_plugin_scanning_enabled()`
- **说明**：Return ``(enabled, reason)``.  Plugin loading executes arbitrary Python in the server process, so it is only enabled by default in single-user local mode.  Hosted deployments must opt in via ``DF_ALLOW_PLUGINS=1``.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_register_plugin_class(key, cls, py_file)`
- **说明**：Register a plugin loader class.  Plugins are **not** allowed to override built-in loaders or earlier plugin loaders. Silent overrides are a credential-exfiltration risk (a malicious ``mysql_data_loader.py`` could replace the built-in MySQL connector and capture every existing MySQL connection's password).  Collisions are recorded in ``PLUGIN_ERRORS`` so the UI can surface them at the top of the connector picker.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_load_plugin_file(py_file)`
- **说明**：Load a single ``*_data_loader.py`` plugin file.  On failure the key is recorded in ``DISABLED_LOADERS`` so the UI can surface why the plugin is missing.  ``sys.modules`` is cleaned up on failure to avoid leaking a half-initialized module.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_scan_plugin_dir()`
- **说明**：Scan ``PLUGIN_DIR`` for ``*_data_loader.py`` files.  The registry key is derived from the filename: ``my_custom_data_loader.py`` → ``my_custom``.  Plugins override built-ins with the same key.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_enforce_deployment_restrictions()`
- **说明**：Disable local-only loaders in multi-user mode.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `get_available_loaders()`
- **说明**：Return all registered loaders (built-in + plugins).
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

模块级测试建议：至少覆盖（1）正常路径；（2）非法输入；（3）权限/路径拒绝；（4）上游失败时的 AppError 映射。相关用例见后文测试专章。


#### 模块 `py-src/data_formulator/data_loader/athena_data_loader.py`（约 567 行）

**模块文档**：该文件未写模块级 docstring。其职责由文件名与导出符号定义，修改时应补充文档字符串，避免意图漂移。

符号统计：类 1 个，函数 3 个，模块级常量 3 个。下列说明覆盖全部符号。

**常量**：

- **常量 `ATHENA_TABLE_PATTERN`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `ATHENA_COLUMN_PATTERN`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `S3_URL_PATTERN`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

##### 函数 `_validate_athena_table_name(table_name)`
- **说明**：Validate that table_name is a safe Athena identifier (database.table format).
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_validate_column_name(column_name)`
- **说明**：Validate that column_name is a safe identifier.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_validate_s3_url(url)`
- **说明**：Validate that URL is a proper S3 URL.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 类 `AthenaDataLoader`
- **基类**：ExternalDataLoader。
- **字段标注**：无模块级注解字段（可能使用 dataclass / 动态属性）。
- **方法数**：12。
- **文档字符串**：AWS Athena data loader implementation.  Executes SQL queries on Athena and reads results from S3 via PyArrow. Output location is taken from the workgroup configuration or the output_location param. Use ingest_to_workspace() to store results as parquet in the workspace.
  - **`list_params()`**：内部实现细节见源码。 声明连接器表单字段：类型、是否必填、是否敏感、tier（connection/auth/filter），供前端动态渲染。
  - **`auth_paths(cls)`**：内部实现细节见源码。
  - **`infer_auth_path(cls, params)`**：内部实现细节见源码。
  - **`__init__(self, params)`**：内部实现细节见源码。
  - **`_get_output_location(self)`**：Get the output location for query results.  Priority: user-provided output_location > workgroup configuration.
  - **`_execute_query(self, query)`**：Execute an Athena query and wait for completion.  Returns the S3 path to the query results (CSV file).
  - **`fetch_data_as_arrow(self, source_table, import_options)`**：Fetch data from Athena as a PyArrow Table.  Executes the query on Athena and reads the CSV results from S3 using PyArrow's S3 filesystem. 按 import_options（列投影、过滤、排序、行数上限）从源系统拉数并转为 Arrow Table，再写入工作区 Parquet。
  - **`list_tables(self, table_filter)`**：List tables from Athena catalog (Glue Data Catalog).
  - **`catalog_hierarchy()`**：内部实现细节见源码。
  - **`ls(self, path, filter)`**：内部实现细节见源码。
  - **`get_metadata(self, path)`**：内部实现细节见源码。
  - **`test_connection(self)`**：内部实现细节见源码。
  设计约束：公开方法必须返回安全数据（无密钥、无绝对路径）；异常应转换为 AppError 或 ToolResult 可恢复错误，而不是把 Python 堆栈直接交给前端。

模块级测试建议：至少覆盖（1）正常路径；（2）非法输入；（3）权限/路径拒绝；（4）上游失败时的 AppError 映射。相关用例见后文测试专章。


#### 模块 `py-src/data_formulator/data_loader/azure_blob_data_loader.py`（约 402 行）

**模块文档**：该文件未写模块级 docstring。其职责由文件名与导出符号定义，修改时应补充文档字符串，避免意图漂移。

符号统计：类 1 个，函数 0 个，模块级常量 0 个。下列说明覆盖全部符号。

##### 类 `AzureBlobDataLoader`
- **基类**：ExternalDataLoader。
- **字段标注**：无模块级注解字段（可能使用 dataclass / 动态属性）。
- **方法数**：17。
- **职责推断**：该类位于对应模块中，承担 `AzureBlobDataLoader` 所表示的领域对象或服务。其方法覆盖构造、序列化、领域操作与错误处理；调用方应通过公开方法交互，避免依赖私有实现细节。
  - **`list_params()`**：内部实现细节见源码。 声明连接器表单字段：类型、是否必填、是否敏感、tier（connection/auth/filter），供前端动态渲染。
  - **`auth_paths(cls)`**：内部实现细节见源码。
  - **`infer_auth_path(cls, params)`**：内部实现细节见源码。
  - **`__init__(self, params)`**：内部实现细节见源码。
  - **`_azure_path(self, azure_url)`**：Convert Azure URL to path for PyArrow (container/blob).
  - **`_read_sample(self, azure_url, limit)`**：Read sample rows from an Azure blob using PyArrow. Returns a pandas DataFrame.
  - **`fetch_data_as_arrow(self, source_table, import_options)`**：Fetch data from Azure Blob as a PyArrow Table.  For files (parquet, csv), reads directly using PyArrow's Azure filesystem. 按 import_options（列投影、过滤、排序、行数上限）从源系统拉数并转为 Arrow Table，再写入工作区 Parquet。
  - **`probe(self, path, query)`**：Read the blob into DuckDB and compute the SPJQ there.
  - **`list_tables(self, table_filter)`**：内部实现细节见源码。
  - **`_is_supported_file(self, blob_name)`**：Check if the file type is supported (PyArrow can read it).
  - **`_estimate_row_count(self, azure_url, blob_properties)`**：Estimate the number of rows in a file.
  - **`_estimate_rows_by_sampling(self, azure_url, blob_properties, file_extension)`**：Estimate row count for text-based files using PyArrow sampling.
  - **`_estimate_by_row_sampling(self, azure_url, file_extension)`**：Estimate row count by reading a capped sample with PyArrow.
  - **`catalog_hierarchy()`**：内部实现细节见源码。
  - **`ls(self, path, filter)`**：内部实现细节见源码。
  - **`get_metadata(self, path)`**：内部实现细节见源码。
  - **`test_connection(self)`**：内部实现细节见源码。
  设计约束：公开方法必须返回安全数据（无密钥、无绝对路径）；异常应转换为 AppError 或 ToolResult 可恢复错误，而不是把 Python 堆栈直接交给前端。

模块级测试建议：至少覆盖（1）正常路径；（2）非法输入；（3）权限/路径拒绝；（4）上游失败时的 AppError 映射。相关用例见后文测试专章。


#### 模块 `py-src/data_formulator/data_loader/bigquery_data_loader.py`（约 348 行）

**模块文档**：该文件未写模块级 docstring。其职责由文件名与导出符号定义，修改时应补充文档字符串，避免意图漂移。

符号统计：类 1 个，函数 0 个，模块级常量 0 个。下列说明覆盖全部符号。

##### 类 `BigQueryDataLoader`
- **基类**：ExternalDataLoader。
- **字段标注**：无模块级注解字段（可能使用 dataclass / 动态属性）。
- **方法数**：12。
- **文档字符串**：BigQuery data loader implementation
  - **`list_params()`**：内部实现细节见源码。 声明连接器表单字段：类型、是否必填、是否敏感、tier（connection/auth/filter），供前端动态渲染。
  - **`auth_paths(cls)`**：内部实现细节见源码。
  - **`infer_auth_path(cls, params)`**：内部实现细节见源码。
  - **`__init__(self, params)`**：内部实现细节见源码。
  - **`list_tables(self, table_filter)`**：List tables from BigQuery datasets
  - **`fetch_data_as_arrow(self, source_table, import_options)`**：Fetch data from BigQuery as a PyArrow Table using native Arrow support.  BigQuery's Python client provides .to_arrow() for efficient Arrow-native data transfer, avoiding pandas conversion overhead. 按 import_options（列投影、过滤、排序、行数上限）从源系统拉数并转为 Arrow Table，再写入工作区 Parquet。
  - **`probe(self, path, query)`**：Compile the SPJQ to BigQuery Standard SQL and run it server-side.
  - **`_build_select_parts(self, table_ref, table_name)`**：Build SELECT parts handling nested BigQuery fields.
  - **`catalog_hierarchy()`**：内部实现细节见源码。
  - **`ls(self, path, filter)`**：内部实现细节见源码。
  - **`get_metadata(self, path)`**：内部实现细节见源码。
  - **`test_connection(self)`**：内部实现细节见源码。
  设计约束：公开方法必须返回安全数据（无密钥、无绝对路径）；异常应转换为 AppError 或 ToolResult 可恢复错误，而不是把 Python 堆栈直接交给前端。

模块级测试建议：至少覆盖（1）正常路径；（2）非法输入；（3）权限/路径拒绝；（4）上游失败时的 AppError 映射。相关用例见后文测试专章。


#### 模块 `py-src/data_formulator/data_loader/clickhouse_data_loader.py`（约 752 行）

**模块文档**：ClickHouse connector for Data Formulator.

符号统计：类 1 个，函数 1 个，模块级常量 4 个。下列说明覆盖全部符号。

**常量**：

- **常量 `_QUOTED_RELATION_RE`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `_RAW_SQL_RE`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `_SOURCE_FILTER_OPERATORS`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `_LEGACY_OPERATORS`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

##### 函数 `_as_bool(value)`
- **说明**：模块级函数 `_as_bool` 封装可复用算法或路由处理。参数经过校验后进入核心逻辑，返回结构化结果或通过异常表达业务失败。
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 类 `ClickHouseDataLoader`
- **基类**：ExternalDataLoader。
- **字段标注**：无模块级注解字段（可能使用 dataclass / 动态属性）。
- **方法数**：31。
- **文档字符串**：Read-only ClickHouse loader using the official HTTP client.
  - **`list_params()`**：内部实现细节见源码。 声明连接器表单字段：类型、是否必填、是否敏感、tier（connection/auth/filter），供前端动态渲染。
  - **`auth_paths(cls)`**：内部实现细节见源码。
  - **`__init__(self, params)`**：内部实现细节见源码。
  - **`close(self)`**：内部实现细节见源码。
  - **`__enter__(self)`**：内部实现细节见源码。
  - **`__exit__(self, exc_type, exc_value, traceback)`**：内部实现细节见源码。
  - **`__del__(self)`**：内部实现细节见源码。
  - **`_read_sql(self, query, parameters)`**：内部实现细节见源码。
  - **`_quote_identifier(name)`**：内部实现细节见源码。
  - **`_unwrap_type(column_type)`**：Strip ``Nullable``/``LowCardinality`` wrappers off a ClickHouse type.
  - **`_cast_function(cls, column_type)`**：The function that renders a ClickHouse type faithfully through Arrow.  ClickHouse's Arrow output format is a lossy projection of its type system: several types are serialised as their physical representation, so they arr
  - **`_decimal_precision(base_type)`**：Precision of a ``Decimal(P, S)`` type; 0 when it cannot be read.
  - **`_project_column(cls, column, cast)`**：Apply ``cast`` to ``column``, keeping the column's original name.
  - **`_column_casts(self, database, table)`**：Map column name -> cast template, for the columns that need one.
  - **`_casts_from_column_types(cls, rows)`**：内部实现细节见源码。
  - **`_build_select_list(cls, casts, columns)`**：The SELECT list that projects every degraded column back into shape.
  - **`_quote_relation(cls, database, table)`**：内部实现细节见源码。
  - **`_resolve_source_table(self, source_table)`**：内部实现细节见源码。
  - **`_validated_size(value)`**：内部实现细节见源码。
  - **`_compile_filters(cls, filters)`**：内部实现细节见源码。
  - **`fetch_data_as_arrow(self, source_table, import_options)`**：内部实现细节见源码。 按 import_options（列投影、过滤、排序、行数上限）从源系统拉数并转为 Arrow Table，再写入工作区 Parquet。
  - **`probe(self, path, query)`**：内部实现细节见源码。
  - **`_catalog_tables(self)`**：内部实现细节见源码。
  - **`_catalog_columns(self)`**：内部实现细节见源码。
  - **`list_tables(self, table_filter)`**：内部实现细节见源码。
  - **`search_catalog(self, query, limit)`**：内部实现细节见源码。
  - **`catalog_hierarchy()`**：内部实现细节见源码。
  - **`ls(self, path, filter)`**：内部实现细节见源码。
  - **`get_metadata(self, path)`**：内部实现细节见源码。
  - **`get_column_types(self, source_table)`**：内部实现细节见源码。
  - **`test_connection(self)`**：内部实现细节见源码。
  设计约束：公开方法必须返回安全数据（无密钥、无绝对路径）；异常应转换为 AppError 或 ToolResult 可恢复错误，而不是把 Python 堆栈直接交给前端。

模块级测试建议：至少覆盖（1）正常路径；（2）非法输入；（3）权限/路径拒绝；（4）上游失败时的 AppError 映射。相关用例见后文测试专章。


#### 模块 `py-src/data_formulator/data_loader/connector_errors.py`（约 216 行）

**模块文档**：该文件未写模块级 docstring。其职责由文件名与导出符号定义，修改时应补充文档字符串，避免意图漂移。

符号统计：类 1 个，函数 5 个，模块级常量 0 个。下列说明覆盖全部符号。

##### 类 `ConnectorErrorInfo`
- **基类**：无显式基类。
- **字段标注**：code, message, retry, detail。
- **方法数**：2。
- **职责推断**：该类位于对应模块中，承担 `ConnectorErrorInfo` 所表示的领域对象或服务。其方法覆盖构造、序列化、领域操作与错误处理；调用方应通过公开方法交互，避免依赖私有实现细节。
  - **`to_app_error(self)`**：内部实现细节见源码。
  - **`to_error_dict(self)`**：内部实现细节见源码。
  设计约束：公开方法必须返回安全数据（无密钥、无绝对路径）；异常应转换为 AppError 或 ToolResult 可恢复错误，而不是把 Python 堆栈直接交给前端。

##### 函数 `classify_connector_error(error)`
- **说明**：Map loader/connector failures to a small set of stable error codes.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `raise_connector_error(error)`
- **说明**：模块级函数 `raise_connector_error` 封装可复用算法或路由处理。参数经过校验后进入核心逻辑，返回结构化结果或通过异常表达业务失败。
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_exception_chain(error)`
- **说明**：模块级函数 `_exception_chain` 封装可复用算法或路由处理。参数经过校验后进入核心逻辑，返回结构化结果或通过异常表达业务失败。
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_http_status(chain)`
- **说明**：模块级函数 `_http_status` 封装可复用算法或路由处理。参数经过校验后进入核心逻辑，返回结构化结果或通过异常表达业务失败。
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_has_any(text)`
- **说明**：模块级函数 `_has_any` 封装可复用算法或路由处理。参数经过校验后进入核心逻辑，返回结构化结果或通过异常表达业务失败。
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

模块级测试建议：至少覆盖（1）正常路径；（2）非法输入；（3）权限/路径拒绝；（4）上游失败时的 AppError 映射。相关用例见后文测试专章。


#### 模块 `py-src/data_formulator/data_loader/cosmosdb_data_loader.py`（约 349 行）

**模块文档**：该文件未写模块级 docstring。其职责由文件名与导出符号定义，修改时应补充文档字符串，避免意图漂移。

符号统计：类 1 个，函数 0 个，模块级常量 0 个。下列说明覆盖全部符号。

##### 类 `CosmosDBDataLoader`
- **基类**：ExternalDataLoader。
- **字段标注**：无模块级注解字段（可能使用 dataclass / 动态属性）。
- **方法数**：15。
- **职责推断**：该类位于对应模块中，承担 `CosmosDBDataLoader` 所表示的领域对象或服务。其方法覆盖构造、序列化、领域操作与错误处理；调用方应通过公开方法交互，避免依赖私有实现细节。
  - **`list_params()`**：内部实现细节见源码。 声明连接器表单字段：类型、是否必填、是否敏感、tier（connection/auth/filter），供前端动态渲染。
  - **`__init__(self, params)`**：内部实现细节见源码。
  - **`close(self)`**：Close the Cosmos DB connection.
  - **`__enter__(self)`**：Context manager entry
  - **`__exit__(self, exc_type, exc_val, exc_tb)`**：Context manager exit - ensures connection is closed
  - **`__del__(self)`**：Destructor to ensure connection is closed
  - **`_flatten_document(doc, parent_key, sep)`**：Use recursion to flatten nested Cosmos DB documents. Skips internal Cosmos metadata fields (_rid, _self, _etag, _attachments, _ts).
  - **`_convert_special_types(doc)`**：Convert special types to serializable types.
  - **`_process_documents(self, documents)`**：Process Cosmos DB documents, flatten and convert to DataFrame.
  - **`fetch_data_as_arrow(self, source_table, import_options)`**：内部实现细节见源码。 按 import_options（列投影、过滤、排序、行数上限）从源系统拉数并转为 Arrow Table，再写入工作区 Parquet。
  - **`list_tables(self, table_filter)`**：List all containers in the database.
  - **`catalog_hierarchy()`**：内部实现细节见源码。
  - **`ls(self, path, filter)`**：内部实现细节见源码。
  - **`get_metadata(self, path)`**：内部实现细节见源码。
  - **`test_connection(self)`**：内部实现细节见源码。
  设计约束：公开方法必须返回安全数据（无密钥、无绝对路径）；异常应转换为 AppError 或 ToolResult 可恢复错误，而不是把 Python 堆栈直接交给前端。

模块级测试建议：至少覆盖（1）正常路径；（2）非法输入；（3）权限/路径拒绝；（4）上游失败时的 AppError 映射。相关用例见后文测试专章。


#### 模块 `py-src/data_formulator/data_loader/databricks_data_loader.py`（约 347 行）

**模块文档**：该文件未写模块级 docstring。其职责由文件名与导出符号定义，修改时应补充文档字符串，避免意图漂移。

符号统计：类 1 个，函数 1 个，模块级常量 4 个。下列说明覆盖全部符号。

**常量**：

- **常量 `_HIDDEN_SCHEMAS`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `_HIDDEN_CATALOGS`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `_MAX_CATALOGS`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `_MAX_TABLES`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

##### 函数 `_bt(name)`
- **说明**：Backtick-quote a Databricks/Spark SQL identifier.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 类 `DatabricksDataLoader`
- **基类**：ExternalDataLoader。
- **字段标注**：无模块级注解字段（可能使用 dataclass / 动态属性）。
- **方法数**：14。
- **文档字符串**：Databricks SQL loader for browsing and importing Unity Catalog data.  Connects to a Databricks SQL warehouse via ``databricks-sql-connector`` and browses the Unity Catalog three-level namespace (``catalog.schema.table``). Data is fetched Arrow-native via the connector's ``fetchall_arrow()``.
  - **`list_params()`**：内部实现细节见源码。 声明连接器表单字段：类型、是否必填、是否敏感、tier（connection/auth/filter），供前端动态渲染。
  - **`auth_paths(cls)`**：内部实现细节见源码。
  - **`infer_auth_path(cls, params)`**：内部实现细节见源码。
  - **`delegated_login_config()`**：内部实现细节见源码。
  - **`__init__(self, params)`**：内部实现细节见源码。
  - **`_query_arrow(self, query)`**：Run *query* and return results as a PyArrow Table (Arrow-native).
  - **`_query_rows(self, query)`**：Run *query* and return a list of row dicts.
  - **`_resolve_source_table(self, source_table)`**：Parse ``catalog.schema.table`` (filling pinned params when partial).
  - **`catalog_hierarchy()`**：内部实现细节见源码。
  - **`_catalogs(self)`**：内部实现细节见源码。
  - **`list_tables(self, table_filter)`**：List Unity Catalog tables within the pinned/browsable scope.  Uses each catalog's ``information_schema`` to batch-fetch tables, columns and comments (two queries per catalog) so browsing stays fast.
  - **`_list_tables_in_catalog(self, catalog, table_filter)`**：内部实现细节见源码。
  - **`get_column_types(self, source_table)`**：Return source-level column types/comments for a single table.  Uses ``DESCRIBE TABLE`` rather than ``information_schema.columns``: the latter is unreliable for special catalogs (e.g. the built-in ``samples`` catalog expo
  - **`fetch_data_as_arrow(self, source_table, import_options)`**：内部实现细节见源码。 按 import_options（列投影、过滤、排序、行数上限）从源系统拉数并转为 Arrow Table，再写入工作区 Parquet。
  设计约束：公开方法必须返回安全数据（无密钥、无绝对路径）；异常应转换为 AppError 或 ToolResult 可恢复错误，而不是把 Python 堆栈直接交给前端。

模块级测试建议：至少覆盖（1）正常路径；（2）非法输入；（3）权限/路径拒绝；（4）上游失败时的 AppError 映射。相关用例见后文测试专章。


#### 模块 `py-src/data_formulator/data_loader/external_data_loader.py`（约 1212 行）

**模块文档**：该文件未写模块级 docstring。其职责由文件名与导出符号定义，修改时应补充文档字符串，避免意图漂移。

符号统计：类 4 个，函数 9 个，模块级常量 10 个。下列说明覆盖全部符号。

**常量**：

- **常量 `MAX_IMPORT_ROWS`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `SENSITIVE_PARAMS`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `_VALID_OPERATORS`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `_SOURCE_FILTER_OPERATOR_MAP`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `_DANGEROUS_IDENT_RE`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `SOURCE_METADATA_OK`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `SOURCE_METADATA_PARTIAL`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `SOURCE_METADATA_UNAVAILABLE`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `SOURCE_METADATA_SYNCED`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `SOURCE_METADATA_NOT_SYNCED`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

##### 函数 `apply_import_projection(table, import_options)`
- **说明**：Apply and validate the shared load-query projection after source fetch.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 类 `CatalogCachePolicy`
- **基类**：无显式基类。
- **字段标注**：listing_ttl_seconds, metadata_ttl_seconds, refresh_cost, automatic_refresh, automatic_refresh_kind, minimum_retry_seconds。
- **方法数**：1。
- **职责推断**：该类位于对应模块中，承担 `CatalogCachePolicy` 所表示的领域对象或服务。其方法覆盖构造、序列化、领域操作与错误处理；调用方应通过公开方法交互，避免依赖私有实现细节。
  - **`__post_init__(self)`**：内部实现细节见源码。
  设计约束：公开方法必须返回安全数据（无密钥、无绝对路径）；异常应转换为 AppError 或 ToolResult 可恢复错误，而不是把 Python 堆栈直接交给前端。

##### 类 `ConnectorParamError`
- **基类**：ValueError。
- **字段标注**：无模块级注解字段（可能使用 dataclass / 动态属性）。
- **方法数**：1。
- **文档字符串**：Raised when required connector parameters are missing or empty.
  - **`__init__(self, missing, loader_name)`**：内部实现细节见源码。
  设计约束：公开方法必须返回安全数据（无密钥、无绝对路径）；异常应转换为 AppError 或 ToolResult 可恢复错误，而不是把 Python 堆栈直接交给前端。

##### 函数 `_merge_source_metadata(table_metadata, source_meta)`
- **说明**：Merge source-system metadata into a persisted ``TableMetadata``.  Updates the object **in place**:  * ``table_metadata.description`` ← ``source_meta["description"]`` if present. * Each column's ``description`` ← matching column entry in   ``source_meta["columns"]`` if present.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_esc_id(name, quote_char)`
- **说明**：Quote a SQL identifier, escaping embedded quote characters.  E.g. ``_esc_id('col`name', '`')`` → `` `col``name` `` Rejects names with semicolons, null bytes, or SQL comment sequences.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_esc_str(value)`
- **说明**：Escape a string literal for SQL single-quote interpolation.  Doubles single-quotes and strips null bytes.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `build_where_clause(conditions, quote_char)`
- **说明**：Build a WHERE clause from structured filter conditions.  Each condition is a dict with:     - column (str): column name     - operator (str): one of _VALID_OPERATORS     - value: single value, list (IN/NOT IN), or [lo, hi] (BETWEEN)  Returns (clause_str, params) where clause_str is like "WHERE `col1` > ? AND `col2` IN (?, ?)" and params is the flat list of bind values.  Returns ("", []) if conditions is empty.  The caller is responsible for using parameterized execution with the returned params 
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `build_where_clause_inline(conditions, quote_char)`
- **说明**：Build a WHERE clause with values inlined (for ADBC drivers that don't support parameterized queries).  Values are escaped: strings are single-quoted with internal quotes doubled; numbers are passed as-is; None becomes NULL.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `build_source_filter_where_clause_inline(source_filters, quote_char, dialect)`
- **说明**：Build a SQL WHERE clause from frontend ``source_filters``.  ``source_filters`` use a source-agnostic operator vocabulary (``EQ``, ``NEQ``, ``GTE``, ``ILIKE``, ...). SQL loaders should compile that contract to their own dialect here instead of making the frontend emit dialect SQL.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `sanitize_table_name(name_as)`
- **说明**：Backward-compatible alias; see :func:`sanitize_external_loader_table_name`.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `infer_source_metadata_status(metadata)`
- **说明**：Infer ``source_metadata_status`` from a catalog node's metadata dict.  Returns ``"synced"`` when column metadata is present, ``"partial"`` when only table-level metadata is available or the column list is known to be empty, and ``"unavailable"`` otherwise. Loaders may override the status by setting ``source_metadata_status`` explicitly.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 类 `CatalogNode`
- **基类**：无显式基类。
- **字段标注**：name, node_type, path, metadata。
- **方法数**：0。
- **文档字符串**：A node in the data source's catalog tree.  Three kinds of node:  * ``"namespace"`` — expandable container (database, schema, bucket, …).   The hierarchy's ``label`` tells the UI what to call it. * ``"table"`` — importable leaf (table, file, dataset, …). * ``"table_group"`` — a loadable bundle of related tables with optional   shared filters (e.g. a BI dashboard).  Rendered as a non-expandable   le
  设计约束：公开方法必须返回安全数据（无密钥、无绝对路径）；异常应转换为 AppError 或 ToolResult 可恢复错误，而不是把 Python 堆栈直接交给前端。

##### 类 `ExternalDataLoader`
- **基类**：ABC。
- **字段标注**：progress_callback, AUTH_GUIDE, DISPLAY_NAME, DESCRIPTION。
- **方法数**：32。
- **文档字符串**：Abstract base class for external data loaders.  Data loaders fetch data from external sources (databases, cloud storage, etc.) and store data as parquet files in the workspace. DuckDB is not used for storage; it is only the computation engine elsewhere in the application.  Ingest flow: External Source → PyArrow Table → Parquet (workspace).  - `fetch_data_as_arrow()`: each loader must implement; fe
  - **`catalog_cache_policy(cls)`**：Describe catalog freshness and safe automatic refresh behavior.
  - **`_report_progress(self, message)`**：Emit a high-level progress message if a sink is attached.  Safe to call unconditionally: no-op when no callback is set, and callback failures are swallowed so progress reporting can never break the underlying operation.
  - **`get_safe_params(self)`**：Get connection parameters with sensitive values removed.  Uses the ``sensitive`` flag from :meth:`list_params` as the primary source of truth, falling back to the ``SENSITIVE_PARAMS`` name set for params not declared in 
  - **`fetch_data_as_arrow(self, source_table, import_options)`**：Fetch data from the external source as a PyArrow Table.  This is the primary method for data fetching. Each loader must implement this method to fetch data directly as Arrow format for optimal performance. Only source_ta 按 import_options（列投影、过滤、排序、行数上限）从源系统拉数并转为 Arrow Table，再写入工作区 Parquet。
  - **`fetch_data_as_dataframe(self, source_table, import_options)`**：Fetch data from the external source as a pandas DataFrame.  This method converts the Arrow table to pandas. For better performance, prefer using `fetch_data_as_arrow()` directly when possible.
  - **`ingest_to_workspace(self, workspace, table_name, source_table, import_options, source_metadata)`**：Fetch data from external source and store as parquet in workspace.  Uses PyArrow for efficient data transfer: External Source → Arrow → Parquet. This avoids pandas conversion overhead entirely.  After writing the parquet
  - **`list_params()`**：Return list of parameters needed to configure this data loader. 声明连接器表单字段：类型、是否必填、是否敏感、tier（connection/auth/filter），供前端动态渲染。
  - **`validate_params(cls, params)`**：Validate params against ``list_params()`` declarations.  Raises ``ConnectorParamError`` listing all missing required parameters. When *skip_auth_tier* is True, parameters with ``tier="auth"`` are not checked (useful for 
  - **`auth_paths(cls)`**：Declare mutually exclusive authentication paths for the form.  The compatibility adapter exposes existing auth-tier fields as one credentials path. Loaders with alternatives should override this.
  - **`infer_auth_path(cls, params)`**：Choose a path for legacy callers that do not send ``_auth_path``.
  - **`discover_param_options(cls, param_name, params)`**：Discover selectable parameter values after an explicit request.  Loaders may override this for fields whose options require a live, lightweight service call. Discovery is never run automatically.
  - **`auth_instructions(cls)`**：Return the loader's packaged Markdown connection guide.
  - **`delegated_login_config()`**：Return config for delegated (popup-based) token login, or None.  When a loader supports logging in via the external system's own login page (e.g. Superset's token bridge), return a dict with:  * ``"login_url"`` — URL to 
  - **`__init__(self, params)`**：Initialize the data loader.  Args:     params: Configuration parameters for the loader (e.g. host, credentials).
  - **`list_tables(self, table_filter)`**：List all accessible tables within the current pinned scope.  This is the **flat / eager** complement to :meth:`ls`:  * ``list_tables()`` returns *every* importable table the user can   reach given the connection params (
  - **`catalog_hierarchy()`**：Declare the *full* hierarchy of this data source.  Returns an ordered list from root to leaf.  Each entry:  * ``"key"``  — internal identifier, matches a param name in   ``list_params()`` when the level is pinnable (e.g.
  - **`effective_hierarchy(self)`**：Return the *browsable* hierarchy — full hierarchy minus pinned levels.  A level is **pinned** when:  1. Its ``key`` appears in the loader's ``list_params()`` with    ``scope_level=True`` (or when ``key`` matches a param 
  - **`pinned_scope(self)`**：Return ``{level_key: value}`` for every pinned hierarchy level.  These are the levels that were fixed at connection time and are hidden from tree browsing.
  - **`ls(self, path, filter)`**：List children at a catalog path (like ``ls`` in a filesystem).  This is the **lazy / hierarchical** complement to :meth:`list_tables`. It returns one level of the catalog at a time, which is better for large catalogs but
  - **`get_column_values(self, source_table, column_name, keyword, limit, offset)`**：Return distinct values for a column (used for smart filter inputs).  Subclasses may override to provide richer results (e.g. via native Superset APIs).  The default returns an empty list, signalling that the frontend sho
  - **`get_metadata(self, path)`**：Get detailed metadata for a single catalog node.  For a table: columns, types, row count, sample rows. Default: finds the node via ``ls`` and returns its metadata dict.
  - **`get_column_types(self, source_table)`**：Return source-level column type info for a table.  Returns ``{"columns": [{"name": str, "type": str, "is_dttm": bool}, ...], "description": str | None}``. The ``type`` is the *original* source type (e.g. ``TIMESTAMP``, `
  - **`probe(self, path, query)`**：Run a bounded single-table SPJQ read (design 37 §4.2).  The ``query`` object supports ``filters`` (source-agnostic ``EQ/NEQ/…`` vocabulary), ``columns`` (projection), ``group_by``, ``aggregates`` (``count/count_distinct/
  - **`_tables_to_catalog_tree(self, tables)`**：Build a nested catalog tree from ``list_tables``-style entries.
  - **`list_tables_tree(self, table_filter)`**：Build a nested tree from :meth:`list_tables` results.  Returns ``{"hierarchy": [...], "effective_hierarchy": [...], "tree": [...]}``.  Each table entry keeps the full metadata (columns, sample_rows, row_count) from ``lis
  - **`search_catalog(self, query, limit)`**：Return lightweight catalog search results as a tree.  The default implementation reuses ``list_tables(table_filter=...)`` for compatibility. Large or special loaders should override this to avoid fetching columns, sample
  - **`sync_catalog_metadata(self, table_filter)`**：Full metadata sync for catalog cache.  Default implementation: returns ``list_tables()`` results as-is. SQL-based loaders (PostgreSQL, MySQL, etc.) already include full column info from ``information_schema`` in ``list_t
  - **`ensure_table_keys(tables)`**：Ensure every table record has a ``table_key`` field.  If a record lacks ``table_key``, falls back to ``metadata["_source_name"]`` → ``name``.  Warns on records where an explicit key is missing so loader authors notice an
  - **`test_connection(self)`**：Validate the connection is alive.  Default: tries a lightweight ``list_tables`` call. Subclasses should override with something cheaper (e.g. ``SELECT 1``).
  - **`auth_mode()`**：Return ``'connection'`` (default) or ``'token'``.  Legacy interface kept for backward compatibility. New loaders should implement :meth:`auth_config` instead.
  - **`auth_config()`**：Declare how this loader authenticates with its target system.  The :class:`~data_formulator.auth.token_store.TokenStore` reads this to determine which credential strategies to attempt.  Supported modes and required keys:
  - **`rate_limit()`**：Optional rate-limit hints.  ``None`` = no limit.
  设计约束：公开方法必须返回安全数据（无密钥、无绝对路径）；异常应转换为 AppError 或 ToolResult 可恢复错误，而不是把 Python 堆栈直接交给前端。

模块级测试建议：至少覆盖（1）正常路径；（2）非法输入；（3）权限/路径拒绝；（4）上游失败时的 AppError 映射。相关用例见后文测试专章。


#### 模块 `py-src/data_formulator/data_loader/kusto_data_loader.py`（约 867 行）

**模块文档**：该文件未写模块级 docstring。其职责由文件名与导出符号定义，修改时应补充文档字符串，避免意图漂移。

符号统计：类 2 个，函数 1 个，模块级常量 1 个。下列说明覆盖全部符号。

**常量**：

- **常量 `_ISO_DATETIME_RE`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

##### 类 `_KustoDelegatedCredential`
- **基类**：无显式基类。
- **字段标注**：无模块级注解字段（可能使用 dataclass / 动态属性）。
- **方法数**：2。
- **文档字符串**：Azure TokenCredential backed by an OAuth refresh token.
  - **`__init__(self, cluster, access_token, refresh_token, expires_at)`**：内部实现细节见源码。
  - **`get_token(self)`**：内部实现细节见源码。
  设计约束：公开方法必须返回安全数据（无密钥、无绝对路径）；异常应转换为 AppError 或 ToolResult 可恢复错误，而不是把 Python 堆栈直接交给前端。

##### 函数 `_coerce_int(value)`
- **说明**：Best-effort conversion of a Kusto stat field to ``int``.  ``.show tables details`` returns numeric stats that may arrive as ints, floats, strings, or ``None`` depending on the SDK/cluster. Returns ``None`` when the value is missing or not a number.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 类 `KustoDataLoader`
- **基类**：ExternalDataLoader。
- **字段标注**：无模块级注解字段（可能使用 dataclass / 动态属性）。
- **方法数**：25。
- **职责推断**：该类位于对应模块中，承担 `KustoDataLoader` 所表示的领域对象或服务。其方法覆盖构造、序列化、领域操作与错误处理；调用方应通过公开方法交互，避免依赖私有实现细节。
  - **`list_params()`**：内部实现细节见源码。 声明连接器表单字段：类型、是否必填、是否敏感、tier（connection/auth/filter），供前端动态渲染。
  - **`auth_paths(cls)`**：内部实现细节见源码。
  - **`infer_auth_path(cls, params)`**：内部实现细节见源码。
  - **`delegated_login_config()`**：内部实现细节见源码。
  - **`__init__(self, params)`**：内部实现细节见源码。
  - **`_build_kcsb(self)`**：Build the Kusto connection string builder using the best available credential, in priority order.  1. Explicit Kusto-audience ``access_token`` (delegated user token) 2. Service principal (``client_id`` / ``client_secret`
  - **`_convert_kusto_datetime_columns(self, df)`**：Convert Kusto datetime columns to proper pandas datetime format
  - **`_stringify_dynamic_columns(df)`**：Serialize Kusto ``dynamic`` (nested JSON) column values to strings.  Dynamic columns arrive as Python ``dict``/``list`` objects. Left as-is they render as ``[object Object]`` in the UI, break value hashing/ summary code 
  - **`query(self, kql, no_truncation)`**：内部实现细节见源码。
  - **`fetch_data_as_arrow(self, source_table, import_options)`**：Fetch data from Kusto/Azure Data Explorer as a PyArrow Table.  Kusto SDK returns pandas, so we convert to Arrow format.  Args:     source_table: Kusto table name     size: Maximum number of rows to fetch     sort_columns 按 import_options（列投影、过滤、排序、行数上限）从源系统拉数并转为 Arrow Table，再写入工作区 Parquet。
  - **`probe(self, path, query)`**：Compile the SPJQ to KQL and run ``summarize`` on the cluster.  Native pushdown: the filter/group/aggregate runs over the *whole* table on the Kusto engine (not a local sample), so the result is exact — this is how Kusto 
  - **`_kql_ident(name)`**：Quote a column as a KQL bracketed identifier ``['name']``.
  - **`_kql_lit(value)`**：Render a scalar as a KQL literal (string double-quoted, escaped).
  - **`_kql_cmp_lit(value)`**：Render a literal for a comparison/range/set predicate.  ISO-8601 date/datetime strings are emitted as KQL ``datetime(...)`` literals — KQL rejects comparing a ``datetime`` column with a string (``SEM0064: Cannot compare 
  - **`_compile_probe_kql(self, table, query, out_limit)`**：Compile a probe SPJQ object into a KQL query pipeline.  ``T | where … | summarize <aggs> by <keys> | order by … | take N``. Only bare columns and the fixed aggregate vocabulary are emitted.
  - **`_compile_kql_where(self, filters)`**：Compile probe ``filters`` into a list of KQL ``where`` predicates.
  - **`_resolve_source_table(self, source_table)`**：Parse a source_table identifier into ``(database, table)``.  Cross-database catalog entries are ``"database.table"`` and must be split even when a database is pinned — otherwise the whole identifier gets bracket-quoted (
  - **`discover_param_options(cls, param_name, params)`**：List accessible databases only when explicitly requested.
  - **`list_tables(self, table_filter)`**：List tables from the configured Kusto database.  Uses `.show tables details` for lightweight metadata (name, DocString). A bulk `.show database schema as json` also fetches every table's columns in one control command.
  - **`_fetch_db_columns_bulk(self, db_name)`**：Fetch column schemas for *every* table in a database with a single control command (``.show database schema as json``).  This replaces a per-table ``.show table schema`` (one round-trip per table) with one query for the 
  - **`_list_tables_in_db(self, db_name, table_filter, path_prefix, fetch_columns)`**：List tables in a single database (control command runs in-context).
  - **`catalog_hierarchy()`**：内部实现细节见源码。
  - **`ls(self, path, filter)`**：内部实现细节见源码。
  - **`get_metadata(self, path)`**：Live *structural* metadata for one table (columns only).  Intentionally lean. Row counts and byte sizes are **not** fetched here: ``.show table ['T'] details`` runs an extent-stat aggregation that is slow on large tables
  - **`test_connection(self)`**：Verify live cluster access without invoking result conversion.  Connection testing must not use :meth:`query`: that method converts the response to pandas and normalizes its columns, so a local result conversion failure 
  设计约束：公开方法必须返回安全数据（无密钥、无绝对路径）；异常应转换为 AppError 或 ToolResult 可恢复错误，而不是把 Python 堆栈直接交给前端。

模块级测试建议：至少覆盖（1）正常路径；（2）非法输入；（3）权限/路径拒绝；（4）上游失败时的 AppError 映射。相关用例见后文测试专章。


#### 模块 `py-src/data_formulator/data_loader/local_folder_data_loader.py`（约 342 行）

**模块文档**：Local folder data loader — reads data files from a directory on the local filesystem.  Only available in local deployment mode (backend bound to localhost). Uses ConfinedDir to ensure all file access stays within the connected root directory.

符号统计：类 1 个，函数 0 个，模块级常量 1 个。下列说明覆盖全部符号。

**常量**：

- **常量 `SUPPORTED_EXTENSIONS`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

##### 类 `LocalFolderDataLoader`
- **基类**：ExternalDataLoader。
- **字段标注**：无模块级注解字段（可能使用 dataclass / 动态属性）。
- **方法数**：10。
- **文档字符串**：Browse and import data files from a local directory.
  - **`list_params()`**：内部实现细节见源码。 声明连接器表单字段：类型、是否必填、是否敏感、tier（connection/auth/filter），供前端动态渲染。
  - **`catalog_hierarchy()`**：内部实现细节见源码。
  - **`__init__(self, params)`**：内部实现细节见源码。
  - **`test_connection(self)`**：Validate the root directory exists and is readable.
  - **`ls(self, path, filter)`**：List children at a catalog path.  path=[] → list top-level folders and files. path=["subfolder"] → list contents of subfolder.
  - **`get_metadata(self, path)`**：Get detailed metadata for a single file, including sample rows.
  - **`list_tables(self, table_filter)`**：Return data files as 'tables', with subdirectories as namespaces.
  - **`fetch_data_as_arrow(self, source_table, import_options)`**：Read a file from the connected folder into an Arrow table. 按 import_options（列投影、过滤、排序、行数上限）从源系统拉数并转为 Arrow Table，再写入工作区 Parquet。
  - **`probe(self, path, query)`**：Read the file into DuckDB and compute the SPJQ there.
  - **`_file_metadata(self, filepath)`**：Extract lightweight metadata without reading the full file.
  设计约束：公开方法必须返回安全数据（无密钥、无绝对路径）；异常应转换为 AppError 或 ToolResult 可恢复错误，而不是把 Python 堆栈直接交给前端。

模块级测试建议：至少覆盖（1）正常路径；（2）非法输入；（3）权限/路径拒绝；（4）上游失败时的 AppError 映射。相关用例见后文测试专章。


#### 模块 `py-src/data_formulator/data_loader/mongodb_data_loader.py`（约 519 行）

**模块文档**：该文件未写模块级 docstring。其职责由文件名与导出符号定义，修改时应补充文档字符串，避免意图漂移。

符号统计：类 1 个，函数 0 个，模块级常量 0 个。下列说明覆盖全部符号。

##### 类 `MongoDBDataLoader`
- **基类**：ExternalDataLoader。
- **字段标注**：无模块级注解字段（可能使用 dataclass / 动态属性）。
- **方法数**：20。
- **职责推断**：该类位于对应模块中，承担 `MongoDBDataLoader` 所表示的领域对象或服务。其方法覆盖构造、序列化、领域操作与错误处理；调用方应通过公开方法交互，避免依赖私有实现细节。
  - **`list_params()`**：内部实现细节见源码。 声明连接器表单字段：类型、是否必填、是否敏感、tier（connection/auth/filter），供前端动态渲染。
  - **`auth_paths(cls)`**：内部实现细节见源码。
  - **`infer_auth_path(cls, params)`**：内部实现细节见源码。
  - **`__init__(self, params)`**：内部实现细节见源码。
  - **`close(self)`**：Close the MongoDB connection.
  - **`__enter__(self)`**：Context manager entry
  - **`__exit__(self, exc_type, exc_val, exc_tb)`**：Context manager exit - ensures connection is closed
  - **`__del__(self)`**：Destructor to ensure connection is closed
  - **`_flatten_document(doc, parent_key, sep)`**：Use recursion to flatten nested MongoDB documents
  - **`_convert_special_types(doc)`**：Convert MongoDB special types (ObjectId, datetime, etc.) to serializable types
  - **`_process_documents(self, documents)`**：Process MongoDB documents list, flatten and convert to DataFrame
  - **`fetch_data_as_arrow(self, source_table, import_options)`**：内部实现细节见源码。 按 import_options（列投影、过滤、排序、行数上限）从源系统拉数并转为 Arrow Table，再写入工作区 Parquet。
  - **`probe(self, path, query)`**：Compile the SPJQ to a MongoDB aggregation pipeline and run it.  Native pushdown: ``$match / $group / $sort / $limit`` runs server-side over the whole collection, so the result is exact.
  - **`_compile_probe_pipeline(self, query, out_limit)`**：Compile a probe SPJQ object into a MongoDB aggregation pipeline.
  - **`_compile_match(filters)`**：Compile probe ``filters`` into a MongoDB ``$match`` document.
  - **`list_tables(self, table_filter)`**：List all collections
  - **`catalog_hierarchy()`**：内部实现细节见源码。
  - **`ls(self, path, filter)`**：内部实现细节见源码。
  - **`get_metadata(self, path)`**：内部实现细节见源码。
  - **`test_connection(self)`**：内部实现细节见源码。
  设计约束：公开方法必须返回安全数据（无密钥、无绝对路径）；异常应转换为 AppError 或 ToolResult 可恢复错误，而不是把 Python 堆栈直接交给前端。

模块级测试建议：至少覆盖（1）正常路径；（2）非法输入；（3）权限/路径拒绝；（4）上游失败时的 AppError 映射。相关用例见后文测试专章。


#### 模块 `py-src/data_formulator/data_loader/mssql_data_loader.py`（约 784 行）

**模块文档**：该文件未写模块级 docstring。其职责由文件名与导出符号定义，修改时应补充文档字符串，避免意图漂移。

符号统计：类 1 个，函数 0 个，模块级常量 0 个。下列说明覆盖全部符号。

##### 类 `MSSQLDataLoader`
- **基类**：ExternalDataLoader。
- **字段标注**：无模块级注解字段（可能使用 dataclass / 动态属性）。
- **方法数**：17。
- **职责推断**：该类位于对应模块中，承担 `MSSQLDataLoader` 所表示的领域对象或服务。其方法覆盖构造、序列化、领域操作与错误处理；调用方应通过公开方法交互，避免依赖私有实现细节。
  - **`list_params()`**：内部实现细节见源码。 声明连接器表单字段：类型、是否必填、是否敏感、tier（connection/auth/filter），供前端动态渲染。
  - **`auth_paths(cls)`**：内部实现细节见源码。
  - **`infer_auth_path(cls, params)`**：内部实现细节见源码。
  - **`__init__(self, params)`**：内部实现细节见源码。
  - **`_safe_select_list(self, schema, table_name)`**：Build a SELECT column list that converts unsupported types to text. Uses .STAsText() for spatial types, CAST(... AS NVARCHAR(MAX)) for others. Returns '*' if no unsupported columns are found.
  - **`_read_sql(self, query)`**：Execute a query and return results as a PyArrow Table (no pandas).
  - **`_execute_query_raw(self, query)`**：Execute a query (no error wrapping).
  - **`_execute_query(self, query)`**：Execute a query and return results as a PyArrow Table.
  - **`fetch_data_as_arrow(self, source_table, import_options)`**：Fetch data from SQL Server as a PyArrow Table. 按 import_options（列投影、过滤、排序、行数上限）从源系统拉数并转为 Arrow Table，再写入工作区 Parquet。
  - **`probe(self, path, query)`**：Compile the SPJQ to T-SQL (TOP / bracket quoting) and run it.
  - **`list_tables(self, table_filter)`**：List all tables and views from a SQL Server database.  Only queries INFORMATION_SCHEMA in batch; does NOT run per-table SELECT TOP or COUNT(*) to keep catalog browsing fast.
  - **`_list_tables_for_db(self, db, table_filter)`**：List tables and views in a database using three-part naming.  Like ``list_tables`` but queries *[db].INFORMATION_SCHEMA* and returns three-part ``database.schema.table`` identifiers.
  - **`sync_catalog_metadata(self, table_filter)`**：Full metadata sync across all accessible databases.  When ``database`` is specified in connection params, behaves like the base class (delegates to ``list_tables``).  When ``database`` is empty, iterates every online use
  - **`catalog_hierarchy()`**：内部实现细节见源码。
  - **`ls(self, path, filter)`**：内部实现细节见源码。
  - **`get_metadata(self, path)`**：内部实现细节见源码。
  - **`test_connection(self)`**：内部实现细节见源码。
  设计约束：公开方法必须返回安全数据（无密钥、无绝对路径）；异常应转换为 AppError 或 ToolResult 可恢复错误，而不是把 Python 堆栈直接交给前端。

模块级测试建议：至少覆盖（1）正常路径；（2）非法输入；（3）权限/路径拒绝；（4）上游失败时的 AppError 映射。相关用例见后文测试专章。


#### 模块 `py-src/data_formulator/data_loader/mysql_data_loader.py`（约 524 行）

**模块文档**：该文件未写模块级 docstring。其职责由文件名与导出符号定义，修改时应补充文档字符串，避免意图漂移。

符号统计：类 1 个，函数 0 个，模块级常量 0 个。下列说明覆盖全部符号。

##### 类 `MySQLDataLoader`
- **基类**：ExternalDataLoader。
- **字段标注**：无模块级注解字段（可能使用 dataclass / 动态属性）。
- **方法数**：16。
- **职责推断**：该类位于对应模块中，承担 `MySQLDataLoader` 所表示的领域对象或服务。其方法覆盖构造、序列化、领域操作与错误处理；调用方应通过公开方法交互，避免依赖私有实现细节。
  - **`list_params()`**：内部实现细节见源码。 声明连接器表单字段：类型、是否必填、是否敏感、tier（connection/auth/filter），供前端动态渲染。
  - **`auth_paths(cls)`**：内部实现细节见源码。
  - **`__init__(self, params)`**：内部实现细节见源码。
  - **`_get_conn(self)`**：Return a live connection, reconnecting if the previous one was lost.
  - **`_read_sql(self, query)`**：Execute a query and return results as a PyArrow Table (no pandas).  Caller must hold self._lock if thread safety is needed.
  - **`_safe_select_list(self, schema, table_name)`**：Build a SELECT column list that converts unsupported types to text. Uses ST_AsText() for geometry types, CAST(... AS CHAR) for others. Returns '*' if no unsupported columns are found.
  - **`fetch_data_as_arrow(self, source_table, import_options)`**：Fetch data from MySQL as a PyArrow Table. 按 import_options（列投影、过滤、排序、行数上限）从源系统拉数并转为 Arrow Table，再写入工作区 Parquet。
  - **`_fetch_data_as_arrow(self, source_table, import_options)`**：内部实现细节见源码。
  - **`probe(self, path, query)`**：Compile the SPJQ to MySQL and run it server-side.
  - **`list_tables(self, table_filter)`**：List available tables from MySQL database.
  - **`_list_tables(self, table_filter)`**：List tables from MySQL database(s) within pinned scope.  Only queries information_schema in batch; does NOT run per-table SELECT * LIMIT or COUNT(*) to keep catalog browsing fast.
  - **`search_catalog(self, query, limit)`**：Search database/table names without fetching columns, samples, or counts.
  - **`catalog_hierarchy()`**：内部实现细节见源码。
  - **`ls(self, path, filter)`**：内部实现细节见源码。
  - **`get_metadata(self, path)`**：内部实现细节见源码。
  - **`test_connection(self)`**：内部实现细节见源码。
  设计约束：公开方法必须返回安全数据（无密钥、无绝对路径）；异常应转换为 AppError 或 ToolResult 可恢复错误，而不是把 Python 堆栈直接交给前端。

模块级测试建议：至少覆盖（1）正常路径；（2）非法输入；（3）权限/路径拒绝；（4）上游失败时的 AppError 映射。相关用例见后文测试专章。


#### 模块 `py-src/data_formulator/data_loader/postgresql_data_loader.py`（约 860 行）

**模块文档**：该文件未写模块级 docstring。其职责由文件名与导出符号定义，修改时应补充文档字符串，避免意图漂移。

符号统计：类 1 个，函数 0 个，模块级常量 1 个。下列说明覆盖全部符号。

**常量**：

- **常量 `_PG_CLIENT_ENCODING`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

##### 类 `PostgreSQLDataLoader`
- **基类**：ExternalDataLoader。
- **字段标注**：无模块级注解字段（可能使用 dataclass / 动态属性）。
- **方法数**：22。
- **职责推断**：该类位于对应模块中，承担 `PostgreSQLDataLoader` 所表示的领域对象或服务。其方法覆盖构造、序列化、领域操作与错误处理；调用方应通过公开方法交互，避免依赖私有实现细节。
  - **`list_params()`**：内部实现细节见源码。 声明连接器表单字段：类型、是否必填、是否敏感、tier（connection/auth/filter），供前端动态渲染。
  - **`__init__(self, params)`**：内部实现细节见源码。
  - **`_connection_kwargs(self, dbname)`**：内部实现细节见源码。
  - **`_resolve_source_table(self, source_table)`**：Parse a source_table string into (database, schema, table).  Accepts:   - ``"database.schema.table"`` — cross-database reference from lazy catalog   - ``"schema.table"``          — same-database reference   - ``"table"``
  - **`_read_sql(self, query)`**：Execute a query and return results as a PyArrow Table (no pandas).
  - **`_execute_on_conn(conn, query)`**：Run *query* on *conn* and return a PyArrow Table.
  - **`_safe_select_list(self, schema, table_name, dbname)`**：Build a SELECT column list that converts unsupported types to text. Uses ST_AsText() for PostGIS types, ::text for others. Returns '*' if no unsupported columns are found.
  - **`fetch_data_as_arrow(self, source_table, import_options)`**：Fetch data from PostgreSQL as a PyArrow Table. 按 import_options（列投影、过滤、排序、行数上限）从源系统拉数并转为 Arrow Table，再写入工作区 Parquet。
  - **`probe(self, path, query)`**：Compile the SPJQ to PostgreSQL and run it server-side.
  - **`list_tables(self, table_filter)`**：List available tables from PostgreSQL.  When ``database`` is specified, queries only that database. When ``database`` is empty, iterates every accessible database so the result includes tables from all databases — consis
  - **`_list_tables(self, table_filter)`**：List tables from PostgreSQL.  Only queries information_schema and pg_description in batch; does NOT run per-table SELECT * LIMIT or COUNT(*) to keep catalog browsing fast.
  - **`_cross_db_list_tables(self, table_filter)`**：Iterate all accessible databases and collect tables.  Shared by :meth:`list_tables` and :meth:`sync_catalog_metadata` when no database is pinned.  Logs per-database counts so the user can see which databases were scanned
  - **`_list_tables_for_db(self, db, table_filter)`**：List tables in a specific database with full column and comment info.  Like ``_list_tables`` but queries *db* via a fresh connection and returns three-part ``database.schema.table`` identifiers.
  - **`sync_catalog_metadata(self, table_filter)`**：Full metadata sync across all accessible databases.  When ``database`` is specified in connection params, behaves like the base class (delegates to ``list_tables``).  When ``database`` is empty, iterates every accessible
  - **`search_catalog(self, query, limit)`**：Search database/schema/table names without fetching table metadata.
  - **`catalog_hierarchy()`**：内部实现细节见源码。
  - **`_tables_to_catalog_tree(self, tables)`**：Build tree, then append empty databases as namespace nodes.  In cross-database mode (no pinned database), ensures all scanned databases appear in the tree — even those with zero user tables.
  - **`_connect_to_db(self, dbname)`**：Open a new connection to a specific database on the same server.
  - **`_read_sql_on(self, query, dbname)`**：Run a query, optionally on a different database.
  - **`ls(self, path, filter, limit, offset)`**：内部实现细节见源码。
  - **`get_metadata(self, path)`**：内部实现细节见源码。
  - **`test_connection(self)`**：内部实现细节见源码。
  设计约束：公开方法必须返回安全数据（无密钥、无绝对路径）；异常应转换为 AppError 或 ToolResult 可恢复错误，而不是把 Python 堆栈直接交给前端。

模块级测试建议：至少覆盖（1）正常路径；（2）非法输入；（3）权限/路径拒绝；（4）上游失败时的 AppError 映射。相关用例见后文测试专章。


#### 模块 `py-src/data_formulator/data_loader/probe_utils.py`（约 429 行）

**模块文档**：Shared building blocks for the connector ``probe`` capability (design 37).  A probe is a bounded, single-table SPJQ read (Select–Project–Aggregate, *no join*) the data-loading agent runs to size a slice and pick real filter values. Every loader implements its **own** ``probe`` using its backend's native query API — Postgres/MySQL/MSSQL/BigQuery compile SQL, Kusto compiles KQL, Mongo builds an aggregation pipeline, file/object sources read the files into DuckDB.  This module holds only the *reusable* pieces so a loader can one-line the common path instead of reimplementing it:  * :func:`compile_probe_sql` — SPJQ query object → a single SQL SELECT, dialect   aware (quoting + ``LIMIT``/``TOP``). Shared by every SQL backend **and** the   DuckDB read-and-compute path. * :func:`probe_via_native_

符号统计：类 1 个，函数 10 个，模块级常量 14 个。下列说明覆盖全部符号。

**常量**：

- **常量 `PROBE_MAX_ROWS`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `PROBE_DEFAULT_ROWS`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `PROBE_SCAN_ROWS`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `_PROBE_AGG_OPS`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `_FILTER_OP_TO_SQL`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `_DANGEROUS_IDENT_RE`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `ANSI`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `DUCKDB`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `POSTGRES`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `MYSQL`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `CLICKHOUSE`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `MSSQL`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `BIGQUERY`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `ATHENA`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

##### 类 `SqlDialect`
- **基类**：无显式基类。
- **字段标注**：name, open_quote, close_quote, limit_style, ilike。
- **方法数**：0。
- **文档字符串**：How a SQL backend differs when compiling a probe.  ``open_quote``/``close_quote`` bracket an identifier (``"``/``"`` for ANSI, ``` `` ```/``` `` ``` for MySQL, ``[``/``]`` for SQL Server). ``limit_style`` is ``"suffix"`` for ``… LIMIT N`` or ``"top"`` for ``SELECT TOP N …``. ``ilike`` is ``"native"`` when the backend has an ``ILIKE`` operator, else ``"lower_like"`` to emulate it with ``LOWER(col) 
  设计约束：公开方法必须返回安全数据（无密钥、无绝对路径）；异常应转换为 AppError 或 ToolResult 可恢复错误，而不是把 Python 堆栈直接交给前端。

##### 函数 `clamp_probe_limit(limit)`
- **说明**：Clamp a probe ``limit`` into ``[1, PROBE_MAX_ROWS]`` with a default.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `quote_ident(name, dialect)`
- **说明**：Quote a SQL identifier for ``dialect``, escaping the close-quote char.  Works for symmetric quotes (``"col"`` → ``"col"``, embedded ``"`` doubled) and bracket quoting (``[col]``, embedded ``]`` doubled). Rejects names with semicolons, null bytes, or SQL comment sequences.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `probe_filters_to_source_filters(filters)`
- **说明**：Normalise probe ``filters`` (``{column, op, value}``) into the ``source_filters`` shape (``{column, operator, value}``) understood by loader ``fetch_data_as_arrow`` implementations. Unknown ops are dropped.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_lit(v)`
- **说明**：Escape a value as a SQL literal (single-quoted, injection-safe).
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_contains_lit(v)`
- **说明**：Escape a value as a ``%value%`` LIKE literal.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_compile_where(filters, dialect)`
- **说明**：Compile probe ``filters`` into a dialect-aware ``WHERE`` clause.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `compile_probe_sql(query, out_limit)`
- **说明**：Compile a probe SPJQ ``query`` (design 37 §4.2) into a single SELECT.  ``relation`` is the already-qualified/quoted table expression (or a DuckDB ``read_parquet(...)`` scan). Only bare columns and a fixed set of aggregate ops are emitted — never raw expressions. Filters are always applied here so the result is correct regardless of what a loader pushed down. Raises ``ValueError`` on an invalid aggregate op.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `shape_probe_payload(result, out_limit)`
- **说明**：Shape a probe result table into the wire payload (row-cap aware).
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `probe_via_native_sql(query)`
- **说明**：Run a probe by compiling to SQL and executing on the source engine.  ``relation`` is the already-qualified/quoted table expression, ``execute`` runs a SQL string against the loader's native connection and returns Arrow. The source does the filtering/grouping/aggregation over the whole table, so the result is exact. Returns the wire payload or ``{error}``.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `run_probe_on_duckdb(loader, path, query)`
- **说明**：Read the source data into DuckDB and compute the probe there.  The native operation for a file/object source *is* reading the file, so this fetches up to ``scan_size`` rows via ``loader.fetch_data_as_arrow`` (pushing filters down when the loader supports them), registers the Arrow table in DuckDB and runs the compiled SPJQ. For file sources ``scan_size`` is large enough to read the whole file (exact); as a thin sample fallback it bounds the sample (``exact=false`` when the cap is hit). Correctne
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

模块级测试建议：至少覆盖（1）正常路径；（2）非法输入；（3）权限/路径拒绝；（4）上游失败时的 AppError 映射。相关用例见后文测试专章。


#### 模块 `py-src/data_formulator/data_loader/s3_data_loader.py`（约 308 行）

**模块文档**：该文件未写模块级 docstring。其职责由文件名与导出符号定义，修改时应补充文档字符串，避免意图漂移。

符号统计：类 1 个，函数 0 个，模块级常量 0 个。下列说明覆盖全部符号。

##### 类 `S3DataLoader`
- **基类**：ExternalDataLoader。
- **字段标注**：无模块级注解字段（可能使用 dataclass / 动态属性）。
- **方法数**：14。
- **职责推断**：该类位于对应模块中，承担 `S3DataLoader` 所表示的领域对象或服务。其方法覆盖构造、序列化、领域操作与错误处理；调用方应通过公开方法交互，避免依赖私有实现细节。
  - **`list_params()`**：内部实现细节见源码。 声明连接器表单字段：类型、是否必填、是否敏感、tier（connection/auth/filter），供前端动态渲染。
  - **`auth_paths(cls)`**：内部实现细节见源码。
  - **`infer_auth_path(cls, params)`**：内部实现细节见源码。
  - **`__init__(self, params)`**：内部实现细节见源码。
  - **`fetch_data_as_arrow(self, source_table, import_options)`**：Fetch data from S3 as a PyArrow Table using PyArrow's native S3 filesystem.  For files (parquet, csv), reads directly using PyArrow. 按 import_options（列投影、过滤、排序、行数上限）从源系统拉数并转为 Arrow Table，再写入工作区 Parquet。
  - **`probe(self, path, query)`**：Read the file into DuckDB and compute the SPJQ there.
  - **`list_tables(self, table_filter)`**：List available files from S3 bucket.
  - **`_read_sample_arrow(self, s3_url, limit)`**：Read sample data using PyArrow S3 filesystem.
  - **`_is_supported_file(self, key)`**：Check if the file type is supported (CSV, Parquet, JSON).
  - **`_estimate_row_count(self, s3_url)`**：Estimate the number of rows in a file.
  - **`catalog_hierarchy()`**：内部实现细节见源码。
  - **`ls(self, path, filter)`**：内部实现细节见源码。
  - **`get_metadata(self, path)`**：内部实现细节见源码。
  - **`test_connection(self)`**：内部实现细节见源码。
  设计约束：公开方法必须返回安全数据（无密钥、无绝对路径）；异常应转换为 AppError 或 ToolResult 可恢复错误，而不是把 Python 堆栈直接交给前端。

模块级测试建议：至少覆盖（1）正常路径；（2）非法输入；（3）权限/路径拒绝；（4）上游失败时的 AppError 映射。相关用例见后文测试专章。


#### 模块 `py-src/data_formulator/data_loader/sample_datasets_loader.py`（约 312 行）

**模块文档**：Sample datasets data loader.  Exposes the built-in ``EXAMPLE_DATASETS`` catalog as a virtual data connector that behaves exactly like any other connector.  No auth, no external service of its own — table data is fetched on demand from the public URLs declared in :mod:`data_formulator.example_datasets_config`.  The connector is registered unconditionally at startup so that even in ``--disable_database`` mode users still have a zero-config way to load data and explore Data Formulator.

符号统计：类 1 个，函数 0 个，模块级常量 2 个。下列说明覆盖全部符号。

**常量**：

- **常量 `_SAMPLE_CACHE_LOCK`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `_SAMPLE_CACHE_MAX`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

##### 类 `SampleDatasetsLoader`
- **基类**：ExternalDataLoader。
- **字段标注**：无模块级注解字段（可能使用 dataclass / 动态属性）。
- **方法数**：15。
- **文档字符串**：Browse and import the built-in sample datasets.
  - **`list_params()`**：内部实现细节见源码。 声明连接器表单字段：类型、是否必填、是否敏感、tier（connection/auth/filter），供前端动态渲染。
  - **`auth_mode()`**：内部实现细节见源码。
  - **`auth_config()`**：内部实现细节见源码。
  - **`catalog_hierarchy()`**：内部实现细节见源码。
  - **`__init__(self, params)`**：内部实现细节见源码。
  - **`test_connection(self)`**：内部实现细节见源码。
  - **`_datasets(self)`**：内部实现细节见源码。
  - **`_table_stem(table_entry, idx)`**：内部实现细节见源码。
  - **`_columns_from_sample(self, sample, fmt)`**：Infer ``(columns, sample_rows)`` from an embedded preview payload.
  - **`_resolve(self, source_table)`**：Look up ``(dataset, table_entry, table_idx)`` by ``"Dataset/stem"``.  Also accepts the bare dataset name when the dataset has a single table, for convenience.
  - **`list_tables(self, table_filter)`**：内部实现细节见源码。
  - **`get_column_types(self, source_table)`**：内部实现细节见源码。
  - **`fetch_data_as_arrow(self, source_table, import_options)`**：内部实现细节见源码。 按 import_options（列投影、过滤、排序、行数上限）从源系统拉数并转为 Arrow Table，再写入工作区 Parquet。
  - **`probe(self, path, query)`**：Read the sample file into DuckDB and compute the SPJQ there.
  - **`_load_full_dataframe(self, url, fmt, source_table)`**：Return the full parsed DataFrame for a sample dataset URL.  Results are cached in-process: sample dataset URLs are static and small, and previews/loads otherwise re-download + re-parse the entire file on every click, whi
  设计约束：公开方法必须返回安全数据（无密钥、无绝对路径）；异常应转换为 AppError 或 ToolResult 可恢复错误，而不是把 Python 堆栈直接交给前端。

模块级测试建议：至少覆盖（1）正常路径；（2）非法输入；（3）权限/路径拒绝；（4）上游失败时的 AppError 映射。相关用例见后文测试专章。


#### 模块 `py-src/data_formulator/data_loader/superset_auth_bridge.py`（约 89 行）

**模块文档**：Authenticate users via the Superset REST API (JWT).

符号统计：类 1 个，函数 0 个，模块级常量 0 个。下列说明覆盖全部符号。

##### 类 `SupersetAuthBridge`
- **基类**：无显式基类。
- **字段标注**：无模块级注解字段（可能使用 dataclass / 动态属性）。
- **方法数**：6。
- **文档字符串**：Proxy authentication through Superset's public JWT API.
  - **`__init__(self, superset_url, timeout)`**：内部实现细节见源码。
  - **`login(self, username, password)`**：Forward credentials to Superset and return the JWT payload.
  - **`get_user_info(self, access_token)`**：Fetch the authenticated user's profile.
  - **`validate_token(self, access_token)`**：Return user info when the token is still valid, else *None*.
  - **`refresh_token(self, refresh_tok)`**：Exchange a refresh token for a new access token.  Superset (flask-jwt-extended) expects the refresh token as a Bearer header, not in the JSON body.
  - **`exchange_sso_token(self, sso_access_token)`**：Exchange an SSO access token for a Superset JWT.  Requires ``TokenExchangeView`` deployed on the Superset side (see design doc 8, Phase 2).  Returns the same dict shape as :meth:`login` — ``{"access_token": ..., "refresh
  设计约束：公开方法必须返回安全数据（无密钥、无绝对路径）；异常应转换为 AppError 或 ToolResult 可恢复错误，而不是把 Python 堆栈直接交给前端。

模块级测试建议：至少覆盖（1）正常路径；（2）非法输入；（3）权限/路径拒绝；（4）上游失败时的 AppError 映射。相关用例见后文测试专章。


#### 模块 `py-src/data_formulator/data_loader/superset_client.py`（约 182 行）

**模块文档**：Thin wrapper around the Superset public REST API.

符号统计：类 1 个，函数 0 个，模块级常量 0 个。下列说明覆盖全部符号。

##### 类 `SupersetClient`
- **基类**：无显式基类。
- **字段标注**：无模块级注解字段（可能使用 dataclass / 动态属性）。
- **方法数**：11。
- **文档字符串**：Every Superset API call goes through this class so that upstream changes only require edits in one place.
  - **`__init__(self, base_url, timeout)`**：内部实现细节见源码。
  - **`_headers(self, access_token)`**：内部实现细节见源码。
  - **`list_datasets(self, access_token, page, page_size)`**：Return datasets the current user can see (DatasourceFilter).
  - **`get_dataset_detail(self, access_token, dataset_id)`**：内部实现细节见源码。
  - **`get_dataset_columns(self, access_token, dataset_id)`**：Fetch column metadata for a dataset.  Tries the lightweight ``/api/v1/dataset/{pk}/column`` first (Superset 4.1+).  Falls back to the full ``/api/v1/dataset/{pk}`` detail endpoint and extracts ``columns`` from the respon
  - **`get_dataset_distinct_values(self, access_token, column_name)`**：内部实现细节见源码。
  - **`get_datasource_column_values(self, access_token, dataset_id, column_name)`**：内部实现细节见源码。
  - **`list_dashboards(self, access_token, page, page_size)`**：内部实现细节见源码。
  - **`get_dashboard_datasets(self, access_token, dashboard_id)`**：内部实现细节见源码。
  - **`get_dashboard_detail(self, access_token, dashboard_id)`**：内部实现细节见源码。
  - **`post_chart_data(self, access_token, dataset_id, queries, result_format)`**：Execute a query via Chart Data API (dataset-level permission).  Unlike SQL Lab, this endpoint only requires ``datasource access`` permission on the target dataset and automatically applies Row-Level Security (RLS) rules.
  设计约束：公开方法必须返回安全数据（无密钥、无绝对路径）；异常应转换为 AppError 或 ToolResult 可恢复错误，而不是把 Python 堆栈直接交给前端。

模块级测试建议：至少覆盖（1）正常路径；（2）非法输入；（3）权限/路径拒绝；（4）上游失败时的 AppError 映射。相关用例见后文测试专章。


#### 模块 `py-src/data_formulator/data_loader/superset_data_loader.py`（约 1112 行）

**模块文档**：SupersetLoader — ExternalDataLoader implementation for Apache Superset.  Treats Superset as a hierarchical data source:   dashboard (table_group) → dataset (table)  Authentication is JWT-based (``auth_mode() = "token"``).  Data is fetched via Superset's Chart Data API (``POST /api/v1/chart/data``), which only requires ``datasource access`` permission and automatically applies Row-Level Security (RLS).

符号统计：类 1 个，函数 0 个，模块级常量 0 个。下列说明覆盖全部符号。

##### 类 `SupersetLoader`
- **基类**：ExternalDataLoader。
- **字段标注**：无模块级注解字段（可能使用 dataclass / 动态属性）。
- **方法数**：30。
- **文档字符串**：Treats a Superset instance as a hierarchical data source.  Hierarchy: ``dashboard`` (namespace) → ``dataset`` (table). Datasets not attached to any dashboard appear under a synthetic "All Datasets" namespace at the root level.
  - **`list_params()`**：内部实现细节见源码。 声明连接器表单字段：类型、是否必填、是否敏感、tier（connection/auth/filter），供前端动态渲染。
  - **`auth_paths(cls)`**：内部实现细节见源码。
  - **`infer_auth_path(cls, params)`**：内部实现细节见源码。
  - **`auth_mode()`**：内部实现细节见源码。
  - **`auth_config()`**：内部实现细节见源码。
  - **`delegated_login_config()`**：Return popup-based login config if PLG_SUPERSET_URL is set.
  - **`catalog_hierarchy()`**：内部实现细节见源码。
  - **`__init__(self, params)`**：内部实现细节见源码。
  - **`_do_login(self)`**：内部实现细节见源码。
  - **`_try_sso_exchange(self, sso_token)`**：Best-effort SSO token exchange — silently ignored on failure.
  - **`_is_token_expired(token, buffer_seconds)`**：Check JWT exp claim without hitting Superset API.
  - **`_ensure_token(self)`**：Return a valid access token, refreshing if needed.  Uses JWT exp claim to detect expiry (no API call). Tries refresh token first, then full re-login with password. SSO tokens that expire without a refresh token will rais
  - **`test_connection(self)`**：内部实现细节见源码。
  - **`list_tables(self, table_filter)`**：List datasets grouped under dashboards **and** under "All Datasets".  Includes per-dataset column metadata fetched in parallel via ``/api/v1/dataset/{pk}/column``.  This ensures that catalog cache written on connect alre
  - **`_enrich_columns(self, tables, token)`**：Fetch column metadata for all tables in parallel.  Populates ``metadata["columns"]`` and ``metadata["source_metadata_status"]`` on each table entry in-place.  Also sets ``table_key`` from uuid when available.
  - **`search_catalog(self, query, limit)`**：Search Superset datasets and dashboards as a lightweight tree.
  - **`ls(self, path, filter)`**：内部实现细节见源码。
  - **`_build_dashboard_group_metadata(self, token, dashboard_id)`**：Build tables list for a dashboard table_group node.  Returns tables only.  Filters are fetched lazily via ``get_dashboard_filters()`` when the user actually clicks a dashboard.  This is a **lightweight** call: it fetches
  - **`_build_chart_data_filters(source_filters)`**：Convert *source_filters* to Chart Data API ``filters`` format.  Returns a list of ``{"col": ..., "op": ..., "val": ...}`` dicts understood by ``POST /api/v1/chart/data``.  BETWEEN is split into two conditions (``>=`` + `
  - **`_build_chart_data_orderby(sort_columns, sort_order)`**：Convert sort settings to Chart Data API ``orderby`` format.  Returns ``[["col_name", is_ascending], ...]``.
  - **`get_metadata(self, path)`**：内部实现细节见源码。
  - **`get_column_types(self, source_table)`**：Return source-level column types from Superset dataset detail.  Includes ``is_dttm`` flag which reliably identifies temporal columns regardless of the raw type string, and ``description`` from the column's ``verbose_name
  - **`_build_column_entry(cls, c)`**：Build a standardised column metadata dict from a Superset column record.  Merges ``verbose_name``, ``description``, ``expression``, and the ``extra`` JSON blob into a single ``description`` field so that catalog search c
  - **`_normalize_column_type(column)`**：Normalize Superset column metadata to a standard type category.  Uses ``is_dttm``, ``type_generic``, and raw ``type`` to classify into TEMPORAL / NUMERIC / BOOLEAN / STRING.
  - **`get_column_values(self, source_table, column_name, keyword, limit, offset)`**：Return distinct values for *column_name* in a Superset dataset.  Uses a three-tier fallback strategy (same as 0.6): 1. ``/api/v1/datasource/table/{id}/column/{col}/values/`` 2. ``/api/v1/dataset/distinct/{col}`` 3. SQL `
  - **`fetch_data_as_arrow(self, source_table, import_options)`**：Fetch dataset data via Superset's Chart Data API.  Uses ``POST /api/v1/chart/data`` with ``result_type=samples`` which only requires ``datasource access`` permission (no SQL Lab permission needed) and automatically appli 按 import_options（列投影、过滤、排序、行数上限）从源系统拉数并转为 Arrow Table，再写入工作区 Parquet。
  - **`_convert_temporal_columns(self, col_data, temporal_cols, token, dataset_id)`**：Convert epoch-ms temporal columns to proper Arrow date/timestamp types.  Classification strategy: 1. Dataset detail raw type: explicit DATE → date32 2. Value heuristic: all non-null epoch values are exact midnight → date
  - **`_empty_arrow_table(self, token, dataset_id)`**：Build a 0-row Arrow table preserving column names from metadata.
  - **`list_tables_tree(self, table_filter)`**：Build a **lightweight** catalog tree (names + basic metadata only).  No per-dataset detail/SQL queries at this stage — column lists and sample rows are fetched on demand via ``get-catalog`` with a path or ``preview-data`
  - **`_fetch_all_datasets(self, token)`**：Paginate through all datasets.
  设计约束：公开方法必须返回安全数据（无密钥、无绝对路径）；异常应转换为 AppError 或 ToolResult 可恢复错误，而不是把 Python 堆栈直接交给前端。

模块级测试建议：至少覆盖（1）正常路径；（2）非法输入；（3）权限/路径拒绝；（4）上游失败时的 AppError 映射。相关用例见后文测试专章。


### 目录 `py-src/data_formulator/data_loader/guides`

**目录职责**：该目录承载一组紧密相关的实现单元。

本目录内模块共享同一边界：只通过公开函数与相邻层通信。禁止跨层 import 私有 `_` 符号（测试除外）。新增文件必须补测试，并检查是否触发路径安全、日志脱敏、错误协议三条规则。


#### 模块 `py-src/data_formulator/data_loader/guides/__init__.py`（约 1 行）

**模块文档**：Packaged Markdown connection guides for built-in data loaders.

本文件主要为导出聚合或入口转发，无额外类/函数定义。


### 目录 `py-src/data_formulator/data_operations`

**目录职责**：结构化加载计划模型、仓库、执行器

本目录内模块共享同一边界：只通过公开函数与相邻层通信。禁止跨层 import 私有 `_` 符号（测试除外）。新增文件必须补测试，并检查是否触发路径安全、日志脱敏、错误协议三条规则。


#### 模块 `py-src/data_formulator/data_operations/__init__.py`（约 46 行）

**模块文档**：该文件未写模块级 docstring。其职责由文件名与导出符号定义，修改时应补充文档字符串，避免意图漂移。

本文件主要为导出聚合或入口转发，无额外类/函数定义。


#### 模块 `py-src/data_formulator/data_operations/actions.py`（约 12 行）

**模块文档**：该文件未写模块级 docstring。其职责由文件名与导出符号定义，修改时应补充文档字符串，避免意图漂移。

符号统计：类 0 个，函数 1 个，模块级常量 0 个。下列说明覆盖全部符号。

##### 函数 `build_data_operation_action(operation)`
- **说明**：模块级函数 `build_data_operation_action` 封装可复用算法或路由处理。参数经过校验后进入核心逻辑，返回结构化结果或通过异常表达业务失败。
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

模块级测试建议：至少覆盖（1）正常路径；（2）非法输入；（3）权限/路径拒绝；（4）上游失败时的 AppError 映射。相关用例见后文测试专章。


#### 模块 `py-src/data_formulator/data_operations/discovery.py`（约 373 行）

**模块文档**：该文件未写模块级 docstring。其职责由文件名与导出符号定义，修改时应补充文档字符串，避免意图漂移。

符号统计：类 3 个，函数 3 个，模块级常量 3 个。下列说明覆盖全部符号。

**常量**：

- **常量 `DEFAULT_PROBE_BUDGET`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `ANALYST_PROBE_GUIDANCE`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `STANDALONE_PROBE_GUIDANCE`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

##### 类 `ProbeBudget`
- **基类**：无显式基类。
- **字段标注**：remaining。
- **方法数**：1。
- **职责推断**：该类位于对应模块中，承担 `ProbeBudget` 所表示的领域对象或服务。其方法覆盖构造、序列化、领域操作与错误处理；调用方应通过公开方法交互，避免依赖私有实现细节。
  - **`consume(self)`**：内部实现细节见源码。
  设计约束：公开方法必须返回安全数据（无密钥、无绝对路径）；异常应转换为 AppError 或 ToolResult 可恢复错误，而不是把 Python 堆栈直接交给前端。

##### 类 `ProbeGuidance`
- **基类**：无显式基类。
- **字段标注**：exhausted, success。
- **方法数**：0。
- **职责推断**：该类位于对应模块中，承担 `ProbeGuidance` 所表示的领域对象或服务。其方法覆盖构造、序列化、领域操作与错误处理；调用方应通过公开方法交互，避免依赖私有实现细节。
  设计约束：公开方法必须返回安全数据（无密钥、无绝对路径）；异常应转换为 AppError 或 ToolResult 可恢复错误，而不是把 Python 堆栈直接交给前端。

##### 函数 `ensure_catalogs_current(user_home)`
- **说明**：Apply source policies and return current snapshots, including stale ones.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `ensure_no_auth_catalogs_cached(user_home)`
- **说明**：Backward-compatible alias for callers migrated from catalog bootstrap.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_freshness_payload(snapshot)`
- **说明**：模块级函数 `_freshness_payload` 封装可复用算法或路由处理。参数经过校验后进入核心逻辑，返回结构化结果或通过异常表达业务失败。
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 类 `DataDiscoveryService`
- **基类**：无显式基类。
- **字段标注**：无模块级注解字段（可能使用 dataclass / 动态属性）。
- **方法数**：7。
- **文档字符串**：Read-only catalog discovery shared by data-loading entry points.
  - **`__init__(self, workspace)`**：内部实现细节见源码。
  - **`list_data(self, args)`**：内部实现细节见源码。
  - **`find_data(self, args)`**：内部实现细节见源码。
  - **`describe_data(self, args)`**：内部实现细节见源码。
  - **`resolve_catalog_path(self, source_id, table_key)`**：内部实现细节见源码。
  - **`resolve_load_table(self, source_id, table_key)`**：Resolve model-facing catalog identity into loader and display fields.
  - **`probe_data(self, args, budget, guidance)`**：内部实现细节见源码。
  设计约束：公开方法必须返回安全数据（无密钥、无绝对路径）；异常应转换为 AppError 或 ToolResult 可恢复错误，而不是把 Python 堆栈直接交给前端。

模块级测试建议：至少覆盖（1）正常路径；（2）非法输入；（3）权限/路径拒绝；（4）上游失败时的 AppError 映射。相关用例见后文测试专章。


#### 模块 `py-src/data_formulator/data_operations/executor.py`（约 196 行）

**模块文档**：该文件未写模块级 docstring。其职责由文件名与导出符号定义，修改时应补充文档字符串，避免意图漂移。

符号统计：类 2 个，函数 0 个，模块级常量 0 个。下列说明覆盖全部符号。

##### 类 `DataOperationExecutionResult`
- **基类**：无显式基类。
- **字段标注**：result_table_ids, failed_steps。
- **方法数**：0。
- **职责推断**：该类位于对应模块中，承担 `DataOperationExecutionResult` 所表示的领域对象或服务。其方法覆盖构造、序列化、领域操作与错误处理；调用方应通过公开方法交互，避免依赖私有实现细节。
  设计约束：公开方法必须返回安全数据（无密钥、无绝对路径）；异常应转换为 AppError 或 ToolResult 可恢复错误，而不是把 Python 堆栈直接交给前端。

##### 类 `DataOperationExecutor`
- **基类**：无显式基类。
- **字段标注**：无模块级注解字段（可能使用 dataclass / 动态属性）。
- **方法数**：7。
- **职责推断**：该类位于对应模块中，承担 `DataOperationExecutor` 所表示的领域对象或服务。其方法覆盖构造、序列化、领域操作与错误处理；调用方应通过公开方法交互，避免依赖私有实现细节。
  - **`__init__(self, workspace, loader_resolver)`**：内部实现细节见源码。
  - **`execute(self, operation)`**：内部实现细节见源码。
  - **`_publish_connector_query(self, table_name, step)`**：内部实现细节见源码。
  - **`_find_published_results(self, operation_id, plan_hash)`**：内部实现细节见源码。
  - **`_build_import_options(step)`**：内部实现细节见源码。
  - **`_allocate_table_name(requested_name, used)`**：内部实现细节见源码。
  - **`_resolve_live_loader(source_id)`**：内部实现细节见源码。
  设计约束：公开方法必须返回安全数据（无密钥、无绝对路径）；异常应转换为 AppError 或 ToolResult 可恢复错误，而不是把 Python 堆栈直接交给前端。

模块级测试建议：至少覆盖（1）正常路径；（2）非法输入；（3）权限/路径拒绝；（4）上游失败时的 AppError 映射。相关用例见后文测试专章。


#### 模块 `py-src/data_formulator/data_operations/models.py`（约 411 行）

**模块文档**：该文件未写模块级 docstring。其职责由文件名与导出符号定义，修改时应补充文档字符串，避免意图漂移。

符号统计：类 9 个，函数 5 个，模块级常量 1 个。下列说明覆盖全部符号。

**常量**：

- **常量 `DATA_OPERATION_SCHEMA_VERSION`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

##### 函数 `_new_id()`
- **说明**：模块级函数 `_new_id` 封装可复用算法或路由处理。参数经过校验后进入核心逻辑，返回结构化结果或通过异常表达业务失败。
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_canonical_hash(value)`
- **说明**：模块级函数 `_canonical_hash` 封装可复用算法或路由处理。参数经过校验后进入核心逻辑，返回结构化结果或通过异常表达业务失败。
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_freeze_json(value)`
- **说明**：模块级函数 `_freeze_json` 封装可复用算法或路由处理。参数经过校验后进入核心逻辑，返回结构化结果或通过异常表达业务失败。
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_thaw_json(value)`
- **说明**：模块级函数 `_thaw_json` 封装可复用算法或路由处理。参数经过校验后进入核心逻辑，返回结构化结果或通过异常表达业务失败。
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 类 `DataOperationStatus`
- **基类**：StrEnum。
- **字段标注**：无模块级注解字段（可能使用 dataclass / 动态属性）。
- **方法数**：0。
- **职责推断**：该类位于对应模块中，承担 `DataOperationStatus` 所表示的领域对象或服务。其方法覆盖构造、序列化、领域操作与错误处理；调用方应通过公开方法交互，避免依赖私有实现细节。
  设计约束：公开方法必须返回安全数据（无密钥、无绝对路径）；异常应转换为 AppError 或 ToolResult 可恢复错误，而不是把 Python 堆栈直接交给前端。

##### 类 `OperationFilter`
- **基类**：无显式基类。
- **字段标注**：column, operator, value。
- **方法数**：3。
- **职责推断**：该类位于对应模块中，承担 `OperationFilter` 所表示的领域对象或服务。其方法覆盖构造、序列化、领域操作与错误处理；调用方应通过公开方法交互，避免依赖私有实现细节。
  - **`__post_init__(self)`**：内部实现细节见源码。
  - **`to_dict(self)`**：内部实现细节见源码。
  - **`from_dict(cls, value)`**：内部实现细节见源码。
  设计约束：公开方法必须返回安全数据（无密钥、无绝对路径）；异常应转换为 AppError 或 ToolResult 可恢复错误，而不是把 Python 堆栈直接交给前端。

##### 类 `LoadQueryOrder`
- **基类**：无显式基类。
- **字段标注**：column, direction。
- **方法数**：3。
- **职责推断**：该类位于对应模块中，承担 `LoadQueryOrder` 所表示的领域对象或服务。其方法覆盖构造、序列化、领域操作与错误处理；调用方应通过公开方法交互，避免依赖私有实现细节。
  - **`__post_init__(self)`**：内部实现细节见源码。
  - **`to_dict(self)`**：内部实现细节见源码。
  - **`from_dict(cls, value)`**：内部实现细节见源码。
  设计约束：公开方法必须返回安全数据（无密钥、无绝对路径）；异常应转换为 AppError 或 ToolResult 可恢复错误，而不是把 Python 堆栈直接交给前端。

##### 类 `LoadQuery`
- **基类**：无显式基类。
- **字段标注**：filters, columns, order_by, limit。
- **方法数**：3。
- **文档字符串**：Raw-row subset of the shared SPJQ vocabulary used for loading.
  - **`__post_init__(self)`**：内部实现细节见源码。
  - **`to_dict(self)`**：内部实现细节见源码。
  - **`from_dict(cls, value)`**：内部实现细节见源码。
  设计约束：公开方法必须返回安全数据（无密钥、无绝对路径）；异常应转换为 AppError 或 ToolResult 可恢复错误，而不是把 Python 堆栈直接交给前端。

##### 类 `ConnectorQueryStep`
- **基类**：无显式基类。
- **字段标注**：kind, source_id, table_key, display_name, source_table, source_table_name, query。
- **方法数**：3。
- **职责推断**：该类位于对应模块中，承担 `ConnectorQueryStep` 所表示的领域对象或服务。其方法覆盖构造、序列化、领域操作与错误处理；调用方应通过公开方法交互，避免依赖私有实现细节。
  - **`to_dict(self)`**：内部实现细节见源码。
  - **`to_public_dict(self)`**：内部实现细节见源码。
  - **`from_dict(cls, value)`**：内部实现细节见源码。
  设计约束：公开方法必须返回安全数据（无密钥、无绝对路径）；异常应转换为 AppError 或 ToolResult 可恢复错误，而不是把 Python 堆栈直接交给前端。

##### 函数 `_step_from_dict(value)`
- **说明**：模块级函数 `_step_from_dict` 封装可复用算法或路由处理。参数经过校验后进入核心逻辑，返回结构化结果或通过异常表达业务失败。
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 类 `DataOperationPlan`
- **基类**：无显式基类。
- **字段标注**：label, summary, steps, id, plan_hash。
- **方法数**：5。
- **职责推断**：该类位于对应模块中，承担 `DataOperationPlan` 所表示的领域对象或服务。其方法覆盖构造、序列化、领域操作与错误处理；调用方应通过公开方法交互，避免依赖私有实现细节。
  - **`__post_init__(self)`**：内部实现细节见源码。
  - **`compute_hash(self)`**：内部实现细节见源码。 对规范化 JSON（sort_keys、无空格）做 SHA-256，用于加载计划去重。
  - **`to_dict(self)`**：内部实现细节见源码。
  - **`to_public_dict(self)`**：内部实现细节见源码。
  - **`from_dict(cls, value)`**：内部实现细节见源码。
  设计约束：公开方法必须返回安全数据（无密钥、无绝对路径）；异常应转换为 AppError 或 ToolResult 可恢复错误，而不是把 Python 堆栈直接交给前端。

##### 类 `OperationError`
- **基类**：无显式基类。
- **字段标注**：code, message。
- **方法数**：2。
- **职责推断**：该类位于对应模块中，承担 `OperationError` 所表示的领域对象或服务。其方法覆盖构造、序列化、领域操作与错误处理；调用方应通过公开方法交互，避免依赖私有实现细节。
  - **`to_dict(self)`**：内部实现细节见源码。
  - **`from_dict(cls, value)`**：内部实现细节见源码。
  设计约束：公开方法必须返回安全数据（无密钥、无绝对路径）；异常应转换为 AppError 或 ToolResult 可恢复错误，而不是把 Python 堆栈直接交给前端。

##### 类 `FailedOperationStep`
- **基类**：无显式基类。
- **字段标注**：step_index, display_name, error。
- **方法数**：2。
- **职责推断**：该类位于对应模块中，承担 `FailedOperationStep` 所表示的领域对象或服务。其方法覆盖构造、序列化、领域操作与错误处理；调用方应通过公开方法交互，避免依赖私有实现细节。
  - **`to_dict(self)`**：内部实现细节见源码。
  - **`from_dict(cls, value)`**：内部实现细节见源码。
  设计约束：公开方法必须返回安全数据（无密钥、无绝对路径）；异常应转换为 AppError 或 ToolResult 可恢复错误，而不是把 Python 堆栈直接交给前端。

##### 类 `DataOperation`
- **基类**：无显式基类。
- **字段标注**：reason, plans, description, canvas_title, canvas_summary, id, status, selected_plan_id, result_table_ids, error, failed_steps, superseded_by_operation_id, schema_version。
- **方法数**：4。
- **职责推断**：该类位于对应模块中，承担 `DataOperation` 所表示的领域对象或服务。其方法覆盖构造、序列化、领域操作与错误处理；调用方应通过公开方法交互，避免依赖私有实现细节。
  - **`__post_init__(self)`**：内部实现细节见源码。
  - **`to_dict(self)`**：内部实现细节见源码。
  - **`to_public_dict(self)`**：内部实现细节见源码。
  - **`from_dict(cls, value)`**：内部实现细节见源码。
  设计约束：公开方法必须返回安全数据（无密钥、无绝对路径）；异常应转换为 AppError 或 ToolResult 可恢复错误，而不是把 Python 堆栈直接交给前端。

模块级测试建议：至少覆盖（1）正常路径；（2）非法输入；（3）权限/路径拒绝；（4）上游失败时的 AppError 映射。相关用例见后文测试专章。


#### 模块 `py-src/data_formulator/data_operations/repository.py`（约 295 行）

**模块文档**：该文件未写模块级 docstring。其职责由文件名与导出符号定义，修改时应补充文档字符串，避免意图漂移。

符号统计：类 3 个，函数 1 个，模块级常量 2 个。下列说明覆盖全部符号。

**常量**：

- **常量 `STORE_FILENAME`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `STORE_VERSION`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

##### 类 `DataOperationConflictError`
- **基类**：ValueError。
- **字段标注**：无模块级注解字段（可能使用 dataclass / 动态属性）。
- **方法数**：0。
- **职责推断**：该类位于对应模块中，承担 `DataOperationConflictError` 所表示的领域对象或服务。其方法覆盖构造、序列化、领域操作与错误处理；调用方应通过公开方法交互，避免依赖私有实现细节。
  设计约束：公开方法必须返回安全数据（无密钥、无绝对路径）；异常应转换为 AppError 或 ToolResult 可恢复错误，而不是把 Python 堆栈直接交给前端。

##### 类 `StoredDataOperation`
- **基类**：无显式基类。
- **字段标注**：conversation_id, operation。
- **方法数**：2。
- **职责推断**：该类位于对应模块中，承担 `StoredDataOperation` 所表示的领域对象或服务。其方法覆盖构造、序列化、领域操作与错误处理；调用方应通过公开方法交互，避免依赖私有实现细节。
  - **`to_dict(self)`**：内部实现细节见源码。
  - **`from_dict(cls, value)`**：内部实现细节见源码。
  设计约束：公开方法必须返回安全数据（无密钥、无绝对路径）；异常应转换为 AppError 或 ToolResult 可恢复错误，而不是把 Python 堆栈直接交给前端。

##### 类 `DataOperationRepository`
- **基类**：无显式基类。
- **字段标注**：无模块级注解字段（可能使用 dataclass / 动态属性）。
- **方法数**：13。
- **职责推断**：该类位于对应模块中，承担 `DataOperationRepository` 所表示的领域对象或服务。其方法覆盖构造、序列化、领域操作与错误处理；调用方应通过公开方法交互，避免依赖私有实现细节。
  - **`__init__(self, workspace_path)`**：内部实现细节见源码。
  - **`for_workspace(cls, workspace)`**：内部实现细节见源码。
  - **`create(self, operation)`**：内部实现细节见源码。
  - **`get(self, operation_id)`**：内部实现细节见源码。
  - **`get_awaiting_selection(self, operation_id)`**：内部实现细节见源码。
  - **`select(self, operation_id, plan_id)`**：内部实现细节见源码。
  - **`complete(self, operation_id, result_table_ids)`**：内部实现细节见源码。
  - **`fail(self, operation_id, error)`**：内部实现细节见源码。
  - **`finish(self, operation_id, result_table_ids, failed_steps)`**：内部实现细节见源码。
  - **`_record_execution(self, operation_id)`**：内部实现细节见源码。
  - **`_select_operation(operation, plan_id)`**：内部实现细节见源码。
  - **`_read_unlocked(self)`**：内部实现细节见源码。
  - **`_write_unlocked(self, records)`**：内部实现细节见源码。
  设计约束：公开方法必须返回安全数据（无密钥、无绝对路径）；异常应转换为 AppError 或 ToolResult 可恢复错误，而不是把 Python 堆栈直接交给前端。

##### 函数 `resolve_interaction_response(repository, response)`
- **说明**：模块级函数 `resolve_interaction_response` 封装可复用算法或路由处理。参数经过校验后进入核心逻辑，返回结构化结果或通过异常表达业务失败。
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

模块级测试建议：至少覆盖（1）正常路径；（2）非法输入；（3）权限/路径拒绝；（4）上游失败时的 AppError 映射。相关用例见后文测试专章。


### 目录 `py-src/data_formulator/datalake`

**目录职责**：工作区、Parquet、目录缓存、命名与元数据

本目录内模块共享同一边界：只通过公开函数与相邻层通信。禁止跨层 import 私有 `_` 符号（测试除外）。新增文件必须补测试，并检查是否触发路径安全、日志脱敏、错误协议三条规则。


#### 模块 `py-src/data_formulator/datalake/__init__.py`（约 129 行）

**模块文档**：Data Lake module for Data Formulator.  This module provides a unified data management layer that: - Manages user workspaces with identity-based directories - Stores user-uploaded files as-is (CSV, Excel, TXT, HTML, JSON, PDF) - Stores data from external loaders as parquet via pyarrow - Tracks all data sources in a workspace.yaml metadata file  Example usage:      from data_formulator.datalake import Workspace, save_uploaded_file, write_parquet          # Get or create a workspace for a user     workspace = Workspace("user:123")          # Save an uploaded file     with open("sales.csv", "rb") as f:         metadata = save_uploaded_file(workspace, f.read(), "sales.csv")          # Write a DataFrame as parquet (typically from data loaders)     import pandas as pd     df = pd.DataFrame({"id":

本文件主要为导出聚合或入口转发，无额外类/函数定义。


#### 模块 `py-src/data_formulator/datalake/azure_blob_workspace.py`（约 777 行）

**模块文档**：Azure Blob Storage–backed workspace for the Data Lake.  Drop-in replacement for :class:`Workspace` where every file (data files **and** ``workspace.yaml`` metadata) lives as a blob under::      <container>/<datalake_root>/<sanitized_identity_id>/  Requires ``azure-storage-blob`` (``pip install azure-storage-blob``).  Usage::      from azure.storage.blob import ContainerClient      container = ContainerClient.from_connection_string(conn_str, "my-container")     ws = AzureBlobWorkspace("user:42", container, datalake_root="workspaces")

符号统计：类 1 个，函数 2 个，模块级常量 0 个。下列说明覆盖全部符号。

##### 函数 `get_azure_workspace_scratch_path(blob_prefix, safe_id)`
- **说明**：模块级函数 `get_azure_workspace_scratch_path` 封装可复用算法或路由处理。参数经过校验后进入核心逻辑，返回结构化结果或通过异常表达业务失败。
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_data_cache_ttl()`
- **说明**：Seconds a cached *data* blob may be served without re-validating.  Metadata is always re-validated (TTL 0). Data blobs (parquet) are effectively immutable per table version, so a small TTL lets rapid repeat reads (agent tool loops, UI refreshes) skip Azure entirely. Override with ``AZURE_BLOB_CACHE_TTL_SECONDS``.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 类 `AzureBlobWorkspace`
- **基类**：Workspace。
- **字段标注**：无模块级注解字段（可能使用 dataclass / 动态属性）。
- **方法数**：36。
- **文档字符串**：Workspace backed by Azure Blob Storage.  All files (data + ``workspace.yaml``) are stored as blobs under ``<datalake_root>/<sanitized_identity_id>/`` inside the given container.  Inherits from :class:`Workspace` so it is a drop-in replacement everywhere a ``Workspace`` is expected. Methods that only call other (overridden) methods — ``add_table_metadata``, ``get_table_metadata``, ``list_tables``, 
  - **`__init__(self, identity_id, container_client, datalake_root)`**：Args:     identity_id: Unique user identifier (e.g. ``"user:123"``).     container_client: An ``azure.storage.blob.ContainerClient``         already authenticated and pointing at the target container.     datalake_root: 
  - **`_blob_name(self, filename)`**：Full blob name for *filename* within this workspace.
  - **`_data_blob_key(self, filename)`**：Blob-internal key for a data file (under data/ subdirectory).
  - **`_cache_key(self, filename)`**：Globally-unique key for the disk cache: container + full blob name.
  - **`_get_blob(self, filename)`**：Return a ``BlobClient`` for *filename*.
  - **`_blob_exists(self, filename)`**：内部实现细节见源码。
  - **`_upload_bytes(self, filename, data)`**：Upload *data* to blob.  Returns size in bytes.  Write-through: the disk cache is updated with the new bytes + ETag so this worker serves the fresh copy immediately without a round trip.
  - **`_ensure_cached(self, filename)`**：Return a disk-cache :class:`CacheEntry` for *filename*, fetching or re-validating from Azure only when necessary.  - Fresh within TTL (data blobs only) → served from disk, no Azure call. - Otherwise revalidate via a chea
  - **`_download_bytes(self, filename)`**：内部实现细节见源码。
  - **`_delete_blob(self, filename)`**：内部实现细节见源码。
  - **`_temp_local_copy(self, filename)`**：Yield a local file path containing the blob's data.  Backed by the process-global disk cache: the yielded path points at the cached ``.bin`` file (ETag-validated against Azure), so repeated calls across requests reuse th
  - **`_cleanup_temp_files(self)`**：Remove all cached temp files from disk.
  - **`_cleanup_scratch(self)`**：Remove the local scratch directory (sandbox working files).
  - **`__del__(self)`**：内部实现细节见源码。
  - **`_init_metadata(self)`**：内部实现细节见源码。
  - **`get_metadata(self)`**：内部实现细节见源码。
  - **`save_metadata(self, metadata)`**：内部实现细节见源码。
  - **`invalidate_metadata_cache(self)`**：Force the next get_metadata() to re-read from blob storage.
  - **`_atomic_update_metadata(self, updater)`**：Atomically read → update → write blob-backed metadata.  Uses a per-instance :class:`threading.Lock` to serialise concurrent in-process metadata modifications.  This prevents lost-update races when the frontend sends para
  - **`get_file_path(self, filename)`**：Return the full blob name for a data file.  Data files are stored under the ``data/`` prefix within the workspace, matching the local workspace layout.  .. note::     The return type is ``str`` (a blob path), **not** a l
  - **`file_exists(self, filename)`**：内部实现细节见源码。
  - **`delete_table(self, table_name)`**：内部实现细节见源码。
  - **`delete_tables_by_source_file(self, source_filename)`**：内部实现细节见源码。
  - **`cleanup(self)`**：Delete **all** blobs under this workspace's prefix.
  - **`read_data_as_df(self, table_name)`**：内部实现细节见源码。
  - **`write_parquet_from_arrow(self, table, table_name, compression, source_info)`**：内部实现细节见源码。
  - **`write_parquet(self, df, table_name, compression, source_info)`**：内部实现细节见源码。
  - **`get_parquet_schema(self, table_name)`**：内部实现细节见源码。
  - **`get_parquet_path(self, table_name)`**：Return the full blob name for the parquet file.  .. warning::     Unlike the base class this returns a *blob path* (``str``),     **not** a resolved local ``pathlib.Path``.
  - **`run_parquet_sql(self, table_name, sql)`**：Run a DuckDB SQL query against a parquet table.  Downloads the blob to a temporary local file for the duration of the query, so DuckDB can use its native parquet reader.
  - **`local_dir(self)`**：Download all workspace files to a temporary local directory.  Yields the path to the temp directory.  The directory and its contents are removed when the context manager exits.
  - **`upload_file(self, content, filename)`**：Upload raw file content to the workspace as a data blob.
  - **`download_file(self, filename)`**：Download raw file content from the workspace data blob.
  - **`save_workspace_snapshot(self, dst)`**：Download all workspace blobs (including metadata) to *dst*.
  - **`restore_workspace_snapshot(self, src)`**：Replace all workspace blobs with files from *src* directory.
  - **`__repr__(self)`**：内部实现细节见源码。
  设计约束：公开方法必须返回安全数据（无密钥、无绝对路径）；异常应转换为 AppError 或 ToolResult 可恢复错误，而不是把 Python 堆栈直接交给前端。

模块级测试建议：至少覆盖（1）正常路径；（2）非法输入；（3）权限/路径拒绝；（4）上游失败时的 AppError 映射。相关用例见后文测试专章。


#### 模块 `py-src/data_formulator/datalake/azure_blob_workspace_manager.py`（约 348 行）

**模块文档**：AzureBlobWorkspaceManager — manages multiple workspaces per user on Azure Blob Storage.  Extends WorkspaceManager, overriding storage operations to use Azure Blob instead of the local filesystem. Same interface, different backend.  Layout (blob prefixes):     <datalake_root>/users/<safe_id>/workspaces/<workspace_id>/       workspace_meta.json       workspace.yaml       session_state.json       data/         <table>.parquet

符号统计：类 1 个，函数 0 个，模块级常量 0 个。下列说明覆盖全部符号。

##### 类 `AzureBlobWorkspaceManager`
- **基类**：WorkspaceManager。
- **字段标注**：无模块级注解字段（可能使用 dataclass / 动态属性）。
- **方法数**：22。
- **文档字符串**：Manages workspaces stored as blob prefixes in Azure Blob Storage.  Inherits _safe_id, rename_workspace (raises NotImplementedError for now), and the method signatures from WorkspaceManager.
  - **`__init__(self, container_client, workspaces_blob_prefix)`**：Args:     container_client: Authenticated Azure ContainerClient.     workspaces_blob_prefix: Blob prefix for this user's workspaces,         e.g. "workspaces/users/user_123/workspaces/"
  - **`root(self)`**：Not a filesystem path — returns the blob prefix.
  - **`_ws_prefix(self, workspace_id)`**：Blob prefix for a specific workspace.
  - **`_blob_name(self, workspace_id, filename)`**：Full blob name for a file within a workspace.
  - **`_blob_exists(self, blob_name)`**：内部实现细节见源码。
  - **`_upload_blob(self, blob_name, data)`**：内部实现细节见源码。
  - **`_download_blob(self, blob_name)`**：内部实现细节见源码。
  - **`_delete_blobs_with_prefix(self, prefix)`**：Delete all blobs under a prefix. Returns count deleted.
  - **`_list_workspace_prefixes(self)`**：List unique workspace ID prefixes under the user's workspaces root.
  - **`_upload_meta(self, workspace_id, display_name)`**：Upload a lightweight ``workspace_meta.json`` blob for fast listing.  ``createdAt`` is preserved across uploads — only set on the first upload (or backfilled to the current ``updatedAt`` if missing for a legacy workspace)
  - **`_ensure_meta(self, workspace_id)`**：Return workspace_meta.json content, auto-creating it if missing.  Legacy workspaces may only have ``workspace.yaml`` or ``session_state.json``.  This method infers a display name from ``session_state.json`` when possible
  - **`list_workspaces(self)`**：List all workspaces (newest first).  Reads the lightweight ``workspace_meta.json`` blob (~150 bytes) per workspace.  If a workspace lacks this blob (legacy), it is auto-repaired via :meth:`_ensure_meta`.
  - **`workspace_exists(self, workspace_id)`**：Check if a workspace exists.  A workspace exists if any blob exists under its prefix.
  - **`get_workspace_path(self, workspace_id)`**：Returns the blob prefix (not a filesystem path).
  - **`create_workspace(self, workspace_id)`**：Create a new empty workspace by uploading an initial workspace.yaml.  Returns the workspace blob prefix. If the workspace already exists, returns its prefix without error.
  - **`open_workspace(self, workspace_id, identity_id)`**：Open a workspace and return an AzureBlobWorkspace (or cached variant).
  - **`create_and_open_workspace(self, workspace_id, identity_id)`**：Create a new workspace and return an open workspace instance.
  - **`delete_workspace(self, workspace_id)`**：Delete all blobs under the workspace prefix.
  - **`rename_workspace(self, old_id, new_id)`**：Rename not supported for Azure Blob (no atomic rename for prefixes).
  - **`update_display_name(self, workspace_id, display_name)`**：Update only the displayName in workspace_meta.json blob.
  - **`save_session_state(self, workspace_id, state)`**：Save frontend state to session_state.json blob.  Also updates the lightweight ``workspace_meta.json`` blob used by :meth:`list_workspaces`.
  - **`load_session_state(self, workspace_id)`**：Load frontend state from session_state.json blob.
  设计约束：公开方法必须返回安全数据（无密钥、无绝对路径）；异常应转换为 AppError 或 ToolResult 可恢复错误，而不是把 Python 堆栈直接交给前端。

模块级测试建议：至少覆盖（1）正常路径；（2）非法输入；（3）权限/路径拒绝；（4）上游失败时的 AppError 映射。相关用例见后文测试专章。


#### 模块 `py-src/data_formulator/datalake/blob_disk_cache.py`（约 239 行）

**模块文档**：Process-persistent, ETag-validated local disk cache for Azure Blob reads.  Azure blob workspaces build a *fresh* :class:`AzureBlobWorkspace` on every request, so their per-instance in-memory caches are always cold and every request re-downloads ``workspace.yaml`` and data blobs (parquet files can be many megabytes).  This module provides a single, process-global cache that survives across those short-lived instances (and across requests within a worker container), backed by real files on local disk.  Layout under ``<df_home>/blob_cache/``::      <sha256(key)>.bin        # the blob bytes     <sha256(key)>.meta.json  # {"key", "blob_name", "container", "etag", "size", "cached_at"}  The cache stores ``bytes + etag``.  Freshness (whether a conditional GET is needed) is tracked *in memory per p

符号统计：类 2 个，函数 1 个，模块级常量 2 个。下列说明覆盖全部符号。

**常量**：

- **常量 `CACHE_DIR_NAME`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `_DEFAULT_MAX_BYTES`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

##### 类 `CacheEntry`
- **基类**：无显式基类。
- **字段标注**：key, path, etag, size。
- **方法数**：1。
- **文档字符串**：A cached blob: its bytes live at ``path``, validated by ``etag``.
  - **`read_bytes(self)`**：内部实现细节见源码。
  设计约束：公开方法必须返回安全数据（无密钥、无绝对路径）；异常应转换为 AppError 或 ToolResult 可恢复错误，而不是把 Python 堆栈直接交给前端。

##### 类 `BlobDiskCache`
- **基类**：无显式基类。
- **字段标注**：无模块级注解字段（可能使用 dataclass / 动态属性）。
- **方法数**：13。
- **文档字符串**：Thread-safe on-disk cache of blob bytes keyed by ``container/blob_name``.  Safe to share across threads.  Multiple *processes* (gunicorn workers) share the same directory; writes are atomic (temp + ``os.replace``) so a concurrent reader never sees a half-written file.  In-memory bookkeeping (index, validation timestamps, total size) is per-process — that only affects eviction accounting and TTL fr
  - **`__init__(self, root, max_bytes)`**：内部实现细节见源码。
  - **`_stem(key)`**：内部实现细节见源码。
  - **`_bin_path(self, key)`**：内部实现细节见源码。
  - **`_meta_path(self, key)`**：内部实现细节见源码。
  - **`_load_index(self)`**：Populate the in-memory index by scanning existing meta files.
  - **`get(self, key)`**：Return the cached entry for *key*, or ``None`` if absent.
  - **`is_fresh(self, key, ttl_seconds)`**：Whether *key* was validated within the last ``ttl_seconds``.
  - **`mark_validated(self, key)`**：Record that *key* was just confirmed up-to-date against Azure.
  - **`put(self, key, data, etag)`**：Store *data*/*etag* for *key* and return the resulting entry.
  - **`invalidate(self, key)`**：Remove *key* from the cache (disk + memory).
  - **`_atomic_write(path, data)`**：内部实现细节见源码。
  - **`_drop_locked(self, key)`**：内部实现细节见源码。
  - **`_evict_if_needed_locked(self)`**：内部实现细节见源码。
  设计约束：公开方法必须返回安全数据（无密钥、无绝对路径）；异常应转换为 AppError 或 ToolResult 可恢复错误，而不是把 Python 堆栈直接交给前端。

##### 函数 `get_blob_disk_cache()`
- **说明**：Return the process-global :class:`BlobDiskCache` (created on first use).
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

模块级测试建议：至少覆盖（1）正常路径；（2）非法输入；（3）权限/路径拒绝；（4）上游失败时的 AppError 映射。相关用例见后文测试专章。


#### 模块 `py-src/data_formulator/datalake/catalog_cache.py`（约 716 行）

**模块文档**：Catalog cache — persist lightweight list_tables() results to disk.  Stored as JSON files under ``<workspace_root>/catalog_cache/<source_id>.json``. Used by agents to search available data without live connections.  File format::      {         "source_id": "superset_prod",         "synced_at": "2026-04-28T10:00:00Z",         "tables": [             {                 "table_key": "a1b2c3d4-...",                 "name": "42:monthly_orders",                 "path": ["Sales Dashboard", "monthly_orders"],                 "metadata": { ... }             }         ]     }

符号统计：类 2 个，函数 20 个，模块级常量 3 个。下列说明覆盖全部符号。

**常量**：

- **常量 `CATALOG_CACHE_DIR`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `CATALOG_CACHE_SCHEMA_VERSION`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `LIST_DATA_LIMIT`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

##### 类 `CatalogSnapshot`
- **基类**：无显式基类。
- **字段标注**：source_id, tables, listing_refreshed_at, metadata_refreshed_at, listing_age_seconds, metadata_age_seconds, listing_freshness, metadata_freshness, refresh_kind, refresh_status, last_refresh_attempt_at, last_refresh_error。
- **方法数**：0。
- **职责推断**：该类位于对应模块中，承担 `CatalogSnapshot` 所表示的领域对象或服务。其方法覆盖构造、序列化、领域操作与错误处理；调用方应通过公开方法交互，避免依赖私有实现细节。
  设计约束：公开方法必须返回安全数据（无密钥、无绝对路径）；异常应转换为 AppError 或 ToolResult 可恢复错误，而不是把 Python 堆栈直接交给前端。

##### 函数 `_utc_now()`
- **说明**：模块级函数 `_utc_now` 封装可复用算法或路由处理。参数经过校验后进入核心逻辑，返回结构化结果或通过异常表达业务失败。
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_parse_timestamp(value)`
- **说明**：模块级函数 `_parse_timestamp` 封装可复用算法或路由处理。参数经过校验后进入核心逻辑，返回结构化结果或通过异常表达业务失败。
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_age_and_freshness(value, ttl_seconds, now)`
- **说明**：模块级函数 `_age_and_freshness` 封装可复用算法或路由处理。参数经过校验后进入核心逻辑，返回结构化结果或通过异常表达业务失败。
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 类 `CatalogSearchError`
- **基类**：ValueError。
- **字段标注**：无模块级注解字段（可能使用 dataclass / 动态属性）。
- **方法数**：0。
- **文档字符串**：Raised when a catalog search receives a malformed query (e.g. bad regex).  Agent tools should catch this and surface the message verbatim so the model can correct its query, instead of returning an empty result set.
  设计约束：公开方法必须返回安全数据（无密钥、无绝对路径）；异常应转换为 AppError 或 ToolResult 可恢复错误，而不是把 Python 堆栈直接交给前端。

##### 函数 `_cache_dir(workspace_root)`
- **说明**：模块级函数 `_cache_dir` 封装可复用算法或路由处理。参数经过校验后进入核心逻辑，返回结构化结果或通过异常表达业务失败。
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_cache_jail(workspace_root)`
- **说明**：模块级函数 `_cache_jail` 封装可复用算法或路由处理。参数经过校验后进入核心逻辑，返回结构化结果或通过异常表达业务失败。
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_cache_filename(source_id)`
- **说明**：模块级函数 `_cache_filename` 封装可复用算法或路由处理。参数经过校验后进入核心逻辑，返回结构化结果或通过异常表达业务失败。
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_cache_file(workspace_root, source_id)`
- **说明**：模块级函数 `_cache_file` 封装可复用算法或路由处理。参数经过校验后进入核心逻辑，返回结构化结果或通过异常表达业务失败。
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_atomic_write_payload(workspace_root, source_id, payload)`
- **说明**：模块级函数 `_atomic_write_payload` 封装可复用算法或路由处理。参数经过校验后进入核心逻辑，返回结构化结果或通过异常表达业务失败。
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_merge_catalog_listing(previous_tables, listing_tables)`
- **说明**：模块级函数 `_merge_catalog_listing` 封装可复用算法或路由处理。参数经过校验后进入核心逻辑，返回结构化结果或通过异常表达业务失败。
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `save_catalog(workspace_root, source_id, tables)`
- **说明**：Persist catalog data to disk. Best-effort — errors are logged, not raised.  ``mode="replace"`` stores a fresh source snapshot. ``mode="seed_if_missing"`` only writes when no cache exists, so lightweight list calls cannot downgrade a richer sync-catalog-metadata snapshot.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `record_catalog_refresh_failure(workspace_root, source_id, error)`
- **说明**：Record a safe refresh failure without replacing the last good tables.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_load_catalog_raw(workspace_root, source_id)`
- **说明**：Load raw catalog JSON (including original ``source_id`` key).
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `load_catalog(workspace_root, source_id)`
- **说明**：Load cached catalog. Returns None if not found or corrupted.  In disabled-connectors mode, only admin source_ids (e.g. ``sample_datasets``) are readable — user catalogs on disk are hidden.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `load_catalog_snapshot(workspace_root, source_id)`
- **说明**：Load cached tables with independent listing and metadata freshness.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `delete_catalog(workspace_root, source_id)`
- **说明**：Remove cached catalog file. Best-effort.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `list_cached_sources(workspace_root)`
- **说明**：Return the original source IDs that have a cached catalog.  Each cache file stores the original (un-sanitised) ``source_id`` so that ``mysql:mysql`` round-trips correctly even though its filename stem is ``mysql--mysql``. We prefer that stored value here; consumers (agent context, ``load_catalog``, ``delete_catalog``) all accept the original id and re-apply ``safe_source_id`` internally when touching the disk.  Falls back to the filename stem if a cache file is missing or corrupt.  When external
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_search_python(workspace_root, needle, all_ids, exclude, limit_per_source)`
- **说明**：Structured field search over the on-disk catalog cache.  ``needle`` is always a regex pattern (case-insensitive).  Callers who want literal substring matching should ``re.escape`` first.  Invalid patterns raise :class:`CatalogSearchError`.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `search_catalog_cache(workspace_root, query, source_ids, limit_per_source, exclude_tables)`
- **说明**：Search across cached catalogs for tables matching a regex pattern.  ``query`` is treated as a case-insensitive regex.  Callers passing user-typed keywords should ``re.escape`` the input first.  Invalid patterns raise :class:`CatalogSearchError`.  Returns a flat list of match dicts with fields: ``source_id``, ``table_key``, ``name``, ``description``, ``matched_columns``, ``score``, ``match_reasons``, ``metadata_status``.  ``exclude_pattern``, ``fields``, and ``path_prefix`` further constrain the 
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `list_sources_summary(workspace_root)`
- **说明**：Return a per-source summary suitable for ``list_data()`` with no args.  Each entry: ``{source_id, table_count, is_hierarchical}``.  Sources whose cache file is missing or unreadable are skipped silently — the agent treats the cache as ground truth (see design-docs §8).
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `list_path_children(workspace_root, source_id, path, filter, limit)`
- **说明**：List direct children at a hierarchy level within a source's catalog.  Path semantics: each cached table record has ``path: list[str]``.  The final element is the table's leaf name in the tree view; earlier elements are folder segments.  For a query at depth ``K = len(path)``:  * **Folders** = distinct ``path[K]`` from records with ``len(path) >= K+2``   whose first ``K`` segments equal the input path. * **Tables** = records with ``len(path) == K+1`` whose first ``K`` segments   equal the input p
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

模块级测试建议：至少覆盖（1）正常路径；（2）非法输入；（3）权限/路径拒绝；（4）上游失败时的 AppError 映射。相关用例见后文测试专章。


#### 模块 `py-src/data_formulator/datalake/catalog_refresh.py`（约 159 行）

**模块文档**：该文件未写模块级 docstring。其职责由文件名与导出符号定义，修改时应补充文档字符串，避免意图漂移。

符号统计：类 0 个，函数 4 个，模块级常量 2 个。下列说明覆盖全部符号。

**常量**：

- **常量 `_REFRESH_EXECUTOR`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `_REFRESH_LOCK`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

##### 函数 `_retry_allowed(snapshot, policy)`
- **说明**：模块级函数 `_retry_allowed` 封装可复用算法或路由处理。参数经过校验后进入核心逻辑，返回结构化结果或通过异常表达业务失败。
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_refresh_catalog(workspace_root, source_id, loader, policy)`
- **说明**：模块级函数 `_refresh_catalog` 封装可复用算法或路由处理。参数经过校验后进入核心逻辑，返回结构化结果或通过异常表达业务失败。
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_complete_refresh(key, workspace_root, source_id, loader, policy)`
- **说明**：模块级函数 `_complete_refresh` 封装可复用算法或路由处理。参数经过校验后进入核心逻辑，返回结构化结果或通过异常表达业务失败。
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `ensure_catalog_freshness(workspace_root, source_id)`
- **说明**：Return the current snapshot and safely refresh it when policy permits.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

模块级测试建议：至少覆盖（1）正常路径；（2）非法输入；（3）权限/路径拒绝；（4）上游失败时的 AppError 映射。相关用例见后文测试专章。


#### 模块 `py-src/data_formulator/datalake/ephemeral_workspace.py`（约 184 行）

**模块文档**：TTL-managed local workspaces for anonymous/demo deployments.  ``WORKSPACE_BACKEND=ephemeral`` uses the normal on-disk workspace format, but stores it under a separate root and removes inactive workspaces after a configurable TTL. A global LRU byte cap provides a second storage bound. Durable ``local`` workspaces never pass through this module.

符号统计：类 1 个，函数 9 个，模块级常量 2 个。下列说明覆盖全部符号。

**常量**：

- **常量 `_CLEANUP_LOCK`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `_LAST_CLEANUP_AT`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

##### 函数 `_positive_float_env(name, default)`
- **说明**：模块级函数 `_positive_float_env` 封装可复用算法或路由处理。参数经过校验后进入核心逻辑，返回结构化结果或通过异常表达业务失败。
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `get_ephemeral_root()`
- **说明**：Return the root reserved for TTL-managed ephemeral workspaces.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `get_ephemeral_workspaces_root(identity_id)`
- **说明**：模块级函数 `get_ephemeral_workspaces_root` 封装可复用算法或路由处理。参数经过校验后进入核心逻辑，返回结构化结果或通过异常表达业务失败。
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_workspace_updated_at(workspace_dir)`
- **说明**：模块级函数 `_workspace_updated_at` 封装可复用算法或路由处理。参数经过校验后进入核心逻辑，返回结构化结果或通过异常表达业务失败。
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_directory_size(path)`
- **说明**：模块级函数 `_directory_size` 封装可复用算法或路由处理。参数经过校验后进入核心逻辑，返回结构化结果或通过异常表达业务失败。
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_workspace_directories(root)`
- **说明**：模块级函数 `_workspace_directories` 封装可复用算法或路由处理。参数经过校验后进入核心逻辑，返回结构化结果或通过异常表达业务失败。
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_tombstone_path(root, identity_dir, workspace_id)`
- **说明**：模块级函数 `_tombstone_path` 封装可复用算法或路由处理。参数经过校验后进入核心逻辑，返回结构化结果或通过异常表达业务失败。
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_write_tombstone(root, workspace_dir, reason)`
- **说明**：模块级函数 `_write_tombstone` 封装可复用算法或路由处理。参数经过校验后进入核心逻辑，返回结构化结果或通过异常表达业务失败。
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `cleanup_ephemeral_workspaces()`
- **说明**：Remove expired workspaces, then enforce the global LRU byte cap.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 类 `EphemeralWorkspaceManager`
- **基类**：WorkspaceManager。
- **字段标注**：无模块级注解字段（可能使用 dataclass / 动态属性）。
- **方法数**：5。
- **文档字符串**：Standard local workspace manager with ephemeral retention semantics.
  - **`__init__(self, identity_id)`**：内部实现细节见源码。
  - **`workspace_was_evicted(self, workspace_id)`**：内部实现细节见源码。
  - **`_touch(self, workspace_id)`**：内部实现细节见源码。
  - **`open_workspace(self, workspace_id, identity_id)`**：内部实现细节见源码。
  - **`load_session_state(self, workspace_id)`**：内部实现细节见源码。
  设计约束：公开方法必须返回安全数据（无密钥、无绝对路径）；异常应转换为 AppError 或 ToolResult 可恢复错误，而不是把 Python 堆栈直接交给前端。

模块级测试建议：至少覆盖（1）正常路径；（2）非法输入；（3）权限/路径拒绝；（4）上游失败时的 AppError 映射。相关用例见后文测试专章。


#### 模块 `py-src/data_formulator/datalake/file_manager.py`（约 367 行）

**模块文档**：File manager for user-uploaded files in the Data Lake.  This module handles storing user-uploaded files (CSV, Excel, TXT, HTML, JSON, PDF) as-is in the workspace without conversion.

符号统计：类 0 个，函数 9 个，模块级常量 3 个。下列说明覆盖全部符号。

**常量**：

- **常量 `_TEXT_FILE_TYPES`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `SUPPORTED_EXTENSIONS`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `_TRUSTED_DETECTIONS`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

##### 函数 `normalize_text_encoding(content, file_type)`
- **说明**：Detect encoding of text file content and re-encode as UTF-8.  Only processes text-based file types (csv, txt). Binary formats are returned unchanged.  Strategy:    1. Strip UTF-8 BOM if present.   2. Try strict UTF-8 decode — fast path for the common case.   3. Try GBK — covers the vast majority of non-UTF-8 files      produced by Chinese-locale Excel / Windows.  GBK is a strict      superset of GB2312 and handles GB18030 BMP characters too.   4. Use charset_normalizer for less common encodings 
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `is_supported_file(filename)`
- **说明**：模块级函数 `is_supported_file` 封装可复用算法或路由处理。参数经过校验后进入核心逻辑，返回结构化结果或通过异常表达业务失败。
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `get_file_type(filename)`
- **说明**：Get the file type based on extension.  Args:     filename: Name of the file      Returns:     File type string (e.g., 'csv', 'excel') or None if unsupported
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `compute_file_hash(content)`
- **说明**：Compute MD5 hash of file content.  Args:     content: File content as bytes      Returns:     MD5 hash as hex string
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `sanitize_table_name(name)`
- **说明**：Derive a table name from an upload filename (stem).  Delegates to :func:`data_formulator.datalake.table_names.sanitize_upload_stem_table_name`.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `generate_unique_filename(workspace, desired_filename)`
- **说明**：Generate a unique filename if the desired one already exists.  Args:     workspace: The workspace to check     desired_filename: The desired filename      Returns:     A unique filename (may be the original if it doesn't exist)
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `save_uploaded_file(workspace, file_content, filename, table_name, overwrite)`
- **说明**：Save an uploaded file to the workspace.  The file is stored as-is without conversion. Metadata is added to track the file in the workspace.  Args:     workspace: The workspace to save to     file_content: File content as bytes or file-like object     filename: Original filename (used for extension detection)     table_name: Name to use for the table. If None, derived from filename.     overwrite: If True, overwrite existing file. If False, generate unique name.      Returns:     TableMetadata fo
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `save_uploaded_file_from_path(workspace, source_path, table_name, overwrite)`
- **说明**：Save a file from a local path to the workspace.  Args:     workspace: The workspace to save to     source_path: Path to the source file     table_name: Name to use for the table. If None, derived from filename.     overwrite: If True, overwrite existing file.      Returns:     TableMetadata for the saved file
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `get_file_info(workspace, table_name)`
- **说明**：Get information about an uploaded file.  Args:     workspace: The workspace     table_name: Name of the table      Returns:     Dictionary with file information or None if not found
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

模块级测试建议：至少覆盖（1）正常路径；（2）非法输入；（3）权限/路径拒绝；（4）上游失败时的 AppError 映射。相关用例见后文测试专章。


#### 模块 `py-src/data_formulator/datalake/naming.py`（约 54 行）

**模块文档**：Lightweight ID / filename sanitisation helpers.  No heavy dependencies (no pandas, pyarrow, etc.) so that both :mod:`data_connector` and :mod:`datalake.catalog_cache` can import without pulling in the data stack.  For **table-name** sanitisation see :mod:`datalake.table_names`. For **data-file** sanitisation see :func:`datalake.parquet_utils.safe_data_filename`.

符号统计：类 0 个，函数 1 个，模块级常量 1 个。下列说明覆盖全部符号。

**常量**：

- **常量 `_SAFE_SOURCE_ID_RE`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

##### 函数 `safe_source_id(source_id)`
- **说明**：Sanitise a ``source_id`` into a filesystem-safe, collision-resistant string.  Used wherever a source / connector ID needs to become a filename component (e.g. ``connectors/<id>.json``, ``catalog_cache/<id>.json``).  Rules: * ``/`` and ``\`` → ``_``  (path separators) * ``:``            → ``--`` (Windows-unsafe, and preserves uniqueness so that   ``mysql:prod`` and ``mysql_prod`` map to different filenames) * final value must match ``[A-Za-z0-9._-]+`` and not be ``.`` or ``..``  Examples::      >
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

模块级测试建议：至少覆盖（1）正常路径；（2）非法输入；（3）权限/路径拒绝；（4）上游失败时的 AppError 映射。相关用例见后文测试专章。


#### 模块 `py-src/data_formulator/datalake/parquet_utils.py`（约 257 行）

**模块文档**：Parquet utility functions for the Data Lake.  Pure helper functions for parquet I/O, hashing, column introspection, and name sanitisation.  These utilities have **no dependency on Workspace** and are consumed by Workspace methods that handle metadata bookkeeping.

符号统计：类 0 个，函数 11 个，模块级常量 2 个。下列说明覆盖全部符号。

**常量**：

- **常量 `DEFAULT_COMPRESSION`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `DEFAULT_METADATA_SAMPLE_ROWS`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

##### 函数 `safe_data_filename(filename)`
- **说明**：Unicode-safe filename sanitisation for data files.  Prevents path traversal by extracting the basename while preserving Unicode characters (Chinese, Japanese, Korean, etc.) that ``werkzeug.secure_filename`` would strip.  Use this instead of ``secure_filename`` for any path that stores or reads user data files (parquet, csv, xlsx, …).  Args:     filename: Input filename (may contain path components)  Returns:     Sanitised basename (Unicode preserved, no directory components)  Raises:     ValueEr
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `sanitize_table_name(name)`
- **说明**：Sanitize a string to be a valid workspace / parquet logical table name.  Delegates to :func:`data_formulator.datalake.table_names.sanitize_workspace_parquet_table_name`.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `get_sample_rows_from_arrow(table, limit)`
- **说明**：Get a small sample of rows from an Arrow table as JSON/YAML-safe records.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `df_to_safe_records(df)`
- **说明**：Convert a pandas DataFrame to a list of JSON-safe record dicts.  Uses ``date_format='iso'`` so that datetime columns are serialized as ISO-8601 strings instead of epoch milliseconds.  ``default_handler=str`` provides a safety net for exotic types (Decimal, bytes, etc.).  All code that converts a DataFrame to records for API responses or streaming should call this function rather than using ``df.to_json`` / ``df.to_dict`` directly.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_repair_invalid_unicode(value)`
- **说明**：Replace malformed surrogate code points while preserving valid Unicode.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `normalize_dtype_to_app_type(dtype_str)`
- **说明**：Map a pandas/Arrow dtype string to a standardized App Type label.  The labels are consumed by the frontend ``mapApiTypeToAppType()`` and must stay in sync with the ``Type`` enum in ``src/data/types.ts``.  Returns one of: 'datetime', 'date', 'time', 'duration',                  'integer', 'number', 'boolean', 'string'.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `get_arrow_column_info(table)`
- **说明**：Extract column information from a PyArrow Table.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `get_column_info(df)`
- **说明**：Extract column information from a pandas DataFrame.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `compute_arrow_table_hash(table, sample_rows)`
- **说明**：Compute an MD5 hash representing the Arrow Table content.  Uses row count, column names, and sampled rows for efficiency.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `sanitize_dataframe_for_arrow(df)`
- **说明**：Sanitize a DataFrame for conversion to PyArrow Table.  Handles common issues that cause ArrowTypeError: - Mixed types in object columns (e.g., strings and integers) - Columns with all nulls that have ambiguous type  For object dtype columns, converts all non-null values to strings to ensure consistent typing.  Returns:     A copy of the DataFrame with sanitized columns.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `compute_dataframe_hash(df, sample_rows)`
- **说明**：Compute an MD5 hash representing the DataFrame content.  Uses row count, column names, and sampled rows for efficiency.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

模块级测试建议：至少覆盖（1）正常路径；（2）非法输入；（3）权限/路径拒绝；（4）上游失败时的 AppError 映射。相关用例见后文测试专章。


#### 模块 `py-src/data_formulator/datalake/table_names.py`（约 188 行）

**模块文档**：Single source of truth for table-name sanitisation across the datalake, API, data loaders, and DuckDB SQL helpers.  Different call sites historically used slightly different rules (empty-name fallback, digit prefixes, SQL keywords, allowed punctuation). Use the function that matches the **consumer**:  * :func:`sanitize_workspace_parquet_table_name` — logical parquet/workspace   table keys, HTTP ``create-table`` / :mod:`tables_routes` (lowercase,   ``table_`` prefix for invalid leading characters, empty → ``"table"``). * :func:`sanitize_upload_stem_table_name` — default table name derived from an   **upload filename** (``Path.stem``, empty → ``"_unnamed"``, ``_`` prefix for   leading digits). * :func:`sanitize_external_loader_table_name` — names produced when ingesting   from external DB/AP

符号统计：类 0 个，函数 4 个，模块级常量 0 个。下列说明覆盖全部符号。

##### 函数 `sanitize_workspace_parquet_table_name(name)`
- **说明**：Sanitize a user-provided string to a workspace / parquet logical table name.  Preserves Unicode letters and digits while normalizing whitespace, separators, and punctuation to underscores. Result is lowercased. Leading digit or other non-identifier start → ``table_`` prefix. Empty after cleanup → ``"table"``.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `sanitize_upload_stem_table_name(name)`
- **说明**：Derive a table name from an upload **filename** (extension stripped via :meth:`pathlib.PurePath.stem`).  Empty after cleanup → ``"_unnamed"``. Leading digit → leading ``_``. Result is lowercased. Does not treat ``/`` or ``\`` specially beyond :class:`~pathlib.Path` stem behaviour.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `sanitize_external_loader_table_name(name_as)`
- **说明**：Sanitize a table name for external data-loader ingest (parquet in workspace).  Raises:     ValueError: if ``name_as`` is empty.  Strips common SQL comment/injection fragments, normalizes to Unicode word tokens, prefixes with ``_`` if the name is a SQL keyword or starts with a non-letter (except leading ``_``). Max length 63. Case is preserved.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `sanitize_duckdb_sql_table_name(table_name)`
- **说明**：Sanitize a table name for use as a DuckDB view name (quoted identifier).  Allows ``.`` and ``$`` in the character set. Empty → ``"table"``. Invalid leading character → ``table_`` prefix. Case preserved.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

模块级测试建议：至少覆盖（1）正常路径；（2）非法输入；（3）权限/路径拒绝；（4）上游失败时的 AppError 映射。相关用例见后文测试专章。


#### 模块 `py-src/data_formulator/datalake/workspace.py`（约 1038 行）

**模块文档**：Workspace management for the Data Lake.  Each user has a workspace directory identified by their identity_id. The workspace contains all their data files (uploaded and ingested) plus a workspace.yaml metadata file.

符号统计：类 2 个，函数 7 个，模块级常量 1 个。下列说明覆盖全部符号。

**常量**：

- **常量 `SCRATCH_MAX_BYTES`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

##### 函数 `get_data_formulator_home()`
- **说明**：Get the Data Formulator home directory.  Resolution order: 1. Flask app.config['CLI_ARGS']['data_dir'] (set via --data-dir CLI flag) 2. DATA_FORMULATOR_HOME environment variable 3. Default: ~/.data_formulator
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `get_default_workspace_root()`
- **说明**：Get the default workspace root directory.  Returns DATA_FORMULATOR_HOME / "workspaces".
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `get_user_home(identity_id)`
- **说明**：Return the per-user home directory: DATA_FORMULATOR_HOME/users/<safe_id>/.  Shared helper used by workspace_factory, data_connector, and any code that needs per-user storage paths.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `sanitize_identity_dirname(identity_id)`
- **说明**：Sanitize identity_id for use as a directory name.  Uses ``secure_filename`` to produce a safe single-component name. Raises ``ValueError`` if the result is empty or too long.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_sanitize_identity_id(identity_id)`
- **说明**：Backward-compatible alias for identity directory sanitization.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_configured_scratch_max_bytes()`
- **说明**：Total scratch cap from the server's CLI_ARGS, falling back to the default.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `cleanup_stale_temp_files(workspace_path, max_age_hours)`
- **说明**：Remove stale temporary files from workspace directory.  This handles crash recovery by cleaning up temp files (.temp_*.parquet) that were not properly deleted due to server crashes or unexpected shutdowns.  Args:     workspace_path: Path to the workspace directory     max_age_hours: Remove temp files older than this many hours (default: 24)  Returns:     Number of files cleaned up
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 类 `Workspace`
- **基类**：无显式基类。
- **字段标注**：无模块级注解字段（可能使用 dataclass / 动态属性）。
- **方法数**：36。
- **文档字符串**：Manages a user's workspace directory in the Data Lake.  The workspace contains: - workspace.yaml: Metadata file tracking all data sources - Data files: User uploaded files (CSV, Excel, etc.) and parquet files from data loaders  All files are stored in a single flat directory per user.
  - **`__init__(self, identity_id, root_dir)`**：Initialize a workspace for a user.  Args:     identity_id: Unique identifier for the user (e.g., "user:123" or "browser:abc")     root_dir: Root directory for all workspaces. If None, uses default.               Ignored 
  - **`_sanitize_identity_id(identity_id)`**：Sanitize identity_id for use as a directory name.  Delegates to module-level :func:`_sanitize_identity_id`.
  - **`user_home(self)`**：Per-user home directory (parent of workspaces, catalog_cache, etc.).
  - **`confined_root(self)`**：ConfinedDir jail for the workspace root directory.
  - **`confined_data(self)`**：ConfinedDir jail for the ``data/`` sub-directory.
  - **`confined_scratch(self)`**：ConfinedDir jail for the ``scratch/`` sub-directory.
  - **`prune_scratch(self, max_bytes)`**：Cap the scratch directory's total size via LRU eviction.  Scratch accumulates agent intermediates (execute_python DataFrames, fetch_url payloads, uploads) and is otherwise only cleared when the workspace is deleted. When
  - **`_init_metadata(self)`**：Initialize a new workspace with empty metadata.
  - **`get_file_path(self, filename)`**：Get the full path for a data file in the workspace.  Files are stored under the ``data/`` subdirectory of the workspace.  Uses :func:`safe_data_filename` for Unicode-safe sanitisation: extracts the basename to prevent pa
  - **`file_exists(self, filename)`**：Check if a file exists in the workspace.  Args:     filename: Name of the file      Returns:     True if file exists, False otherwise
  - **`delete_table(self, table_name)`**：Delete a table by name (removes both file and metadata).  Args:     table_name: Name of the table to delete      Returns:     True if table was deleted, False if it didn't exist
  - **`_atomic_update_metadata(self, updater)`**：Atomically read → update → write workspace metadata.  Uses :func:`update_metadata` which holds a **single** file lock across the entire read-modify-write cycle, preventing lost updates when multiple requests modify metad
  - **`get_metadata(self)`**：内部实现细节见源码。
  - **`save_metadata(self, metadata)`**：内部实现细节见源码。
  - **`invalidate_metadata_cache(self)`**：Force the next get_metadata() to re-read from disk.
  - **`add_table_metadata(self, table)`**：Atomically add or update a table entry in workspace metadata.
  - **`get_table_metadata(self, table_name)`**：Look up table metadata, falling back to sanitized name.
  - **`list_tables(self)`**：内部实现细节见源码。
  - **`get_fresh_name(self, name)`**：Generate a unique table name that doesn't conflict with existing tables.  Sanitizes the input name, then checks if it already exists in the workspace. If it does, appends an incrementing numeric suffix (_2, _3, ...) unti
  - **`delete_tables_by_source_file(self, source_filename)`**：Delete all tables whose source filename matches.  Matches against both ``source_file`` (upload origin) and ``filename`` (physical file), so this works whether the table was stored as-is or converted to parquet.  Atomical
  - **`cleanup(self)`**：Remove the entire workspace directory.
  - **`get_relative_data_file_path(self, table_name)`**：Get the filename for a table, suitable for use in generated code.  Returns just the filename (e.g. "sales_data.parquet").  The sandbox ensures the script runs with the workspace as its working directory, so ``pd.read_par
  - **`read_data_as_df(self, table_name)`**：Read a table from the workspace as a pandas DataFrame.  Automatically selects the appropriate reader based on the file's type (stored in metadata). Supports parquet, csv, excel, json, and txt. Falls back to sanitized tab
  - **`write_parquet_from_arrow(self, table, table_name, compression, source_info)`**：Write a PyArrow Table directly to parquet.  This is the preferred path because it avoids pandas conversion.
  - **`write_parquet(self, df, table_name, compression, source_info)`**：Write a pandas DataFrame to parquet.
  - **`get_parquet_schema(self, table_name)`**：Get schema information for a parquet table without reading all data.
  - **`get_parquet_path(self, table_name)`**：Return the resolved filesystem path of the parquet file for *table_name*.
  - **`run_parquet_sql(self, table_name, sql)`**：Run a DuckDB SQL query against a parquet table.  The *sql* string must contain a ``{parquet}`` placeholder which will be replaced with ``read_parquet('<path>')``. Example:  ``SELECT * FROM {parquet} AS t LIMIT 10``  This
  - **`refresh_parquet_from_arrow(self, table_name, table, compression)`**：Refresh a parquet table with new Arrow data.  Returns ``(new_metadata, data_changed)``.
  - **`refresh_parquet(self, table_name, df, compression)`**：Refresh a parquet table with new DataFrame data.
  - **`local_dir(self)`**：Context manager yielding a local directory containing workspace files.  For local workspaces this simply yields ``self._path``. Subclasses (e.g. Azure Blob) override this to download files to a temporary directory that i
  - **`save_workspace_snapshot(self, dst)`**：Copy all workspace files (including metadata) to *dst* directory.  Used by session save / export to capture the full workspace state.
  - **`restore_workspace_snapshot(self, src)`**：Replace all workspace files with the contents of *src* directory.  Used by session load / import to restore a previously saved workspace.
  - **`export_session_zip(self, state)`**：Export current state + workspace as a zip.
  - **`import_session_zip(self, zip_data)`**：Import a zip.  Restores workspace, returns state dict.  Raises ``ValueError`` on invalid zip / missing state.json.
  - **`__repr__(self)`**：内部实现细节见源码。
  设计约束：公开方法必须返回安全数据（无密钥、无绝对路径）；异常应转换为 AppError 或 ToolResult 可恢复错误，而不是把 Python 堆栈直接交给前端。

##### 类 `WorkspaceWithTempData`
- **基类**：无显式基类。
- **字段标注**：无模块级注解字段（可能使用 dataclass / 动态属性）。
- **方法数**：7。
- **文档字符串**：A Workspace wrapper with an in-memory metadata overlay for temporary tables.  Delegates **all** attribute and method access to the wrapped workspace via ``__getattr__``, ensuring the correct backend-specific implementations (e.g. ``AzureBlobWorkspace.read_data_as_df``, ``local_dir``, etc.) are always used.  Does **not** inherit from :class:`Workspace` — this is intentional so that Python's MRO can
  - **`__init__(self, workspace, temp_data)`**：内部实现细节见源码。
  - **`__getattr__(self, name)`**：Delegate attribute access to the wrapped workspace.
  - **`get_table_metadata(self, table_name)`**：内部实现细节见源码。
  - **`list_tables(self)`**：内部实现细节见源码。
  - **`read_data_as_df(self, table_name)`**：Read a table — temp overlay first, then delegate to base.
  - **`__enter__(self)`**：内部实现细节见源码。
  - **`__exit__(self, exc_type, exc_val, exc_tb)`**：内部实现细节见源码。
  设计约束：公开方法必须返回安全数据（无密钥、无绝对路径）；异常应转换为 AppError 或 ToolResult 可恢复错误，而不是把 Python 堆栈直接交给前端。

模块级测试建议：至少覆盖（1）正常路径；（2）非法输入；（3）权限/路径拒绝；（4）上游失败时的 AppError 映射。相关用例见后文测试专章。


#### 模块 `py-src/data_formulator/datalake/workspace_manager.py`（约 521 行）

**模块文档**：WorkspaceManager — manages multiple workspaces per user.  Each workspace is a named folder containing:   - workspace_meta.json: lightweight metadata for fast listing   - workspace.yaml: all table metadata (single file)   - session_state.json: auto-persisted frontend state   - data/: data files (parquet, csv, etc.)  Users can create, list, open, delete, and switch workspaces.

符号统计：类 1 个，函数 1 个，模块级常量 3 个。下列说明覆盖全部符号。

**常量**：

- **常量 `SESSION_STATE_FILENAME`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `WORKSPACE_META_FILENAME`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `_SENSITIVE_FIELDS`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

##### 函数 `_strip_sensitive(state)`
- **说明**：Return a copy of *state* with sensitive / ephemeral fields removed.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 类 `WorkspaceManager`
- **基类**：无显式基类。
- **字段标注**：无模块级注解字段（可能使用 dataclass / 动态属性）。
- **方法数**：20。
- **文档字符串**：Manages the set of workspaces for a single user.  Layout:     <workspaces_root>/       <workspace_id>/         workspace_meta.json         workspace.yaml         session_state.json         data/
  - **`__init__(self, workspaces_root)`**：Args:     workspaces_root: Directory containing all workspaces for one user.                      e.g. ~/.data_formulator/workspaces/<user_id>/
  - **`root(self)`**：内部实现细节见源码。
  - **`_safe_id(workspace_id)`**：Sanitize workspace ID for filesystem use.
  - **`_write_meta(self, workspace_id, display_name)`**：Write a lightweight ``workspace_meta.json`` used by list_workspaces.  ``createdAt`` is preserved across writes — only set on the first write (or backfilled to the current ``updatedAt`` if missing for a legacy workspace).
  - **`_ensure_meta(self, workspace_id)`**：Return the workspace_meta.json content, auto-creating it if missing.  Legacy workspaces (created before workspace_meta.json was introduced) only have ``workspace.yaml`` and/or ``session_state.json``.  This method infers 
  - **`_has_content(self, ws_dir)`**：True when a workspace holds work worth listing.  Used as a safety net alongside the ``provisional`` flag: any path that writes real content without clearing the flag still shows up.
  - **`list_workspaces(self)`**：List all workspaces (newest first).  Reads the lightweight ``workspace_meta.json`` (~150 bytes) per workspace.  If a workspace directory lacks this file (legacy), it is auto-repaired via :meth:`_ensure_meta`.  Provisiona
  - **`workspace_exists(self, workspace_id)`**：Check if a workspace with the given ID exists.  A workspace exists if and only if its directory exists.
  - **`get_workspace_path(self, workspace_id)`**：Get the filesystem path for a workspace.
  - **`move_workspaces_from(self, source_root)`**：Copy all workspaces from *source_root* into this manager's root.  Uses copy-then-delete semantics so that the operation succeeds even when the source files are locked by another process (common on Windows when the anonym
  - **`_merge_workspace(src, dest)`**：Merge data files and metadata from *src* workspace into *dest*.
  - **`delete_all_workspaces(self)`**：Delete every workspace under this manager's root.  Returns the number of entries deleted (or attempted). On Windows, locked files are skipped with a warning.
  - **`create_workspace(self, workspace_id)`**：Create a new empty workspace.  Returns the workspace directory path. Raises ValueError if the workspace already exists.
  - **`open_workspace(self, workspace_id, identity_id)`**：Open an existing workspace and return a Workspace instance.  Args:     workspace_id: Workspace ID (folder name).     identity_id: User identity (passed through to Workspace for compatibility).  Returns:     Workspace ins
  - **`create_and_open_workspace(self, workspace_id, identity_id)`**：Create a new workspace and return an open Workspace instance.
  - **`delete_workspace(self, workspace_id)`**：Delete a workspace and all its contents.  Returns True if the workspace existed and was deleted.
  - **`rename_workspace(self, old_id, new_id)`**：Rename a workspace (change its folder ID).  Returns the new workspace directory path. Raises ValueError if old doesn't exist or new already exists.
  - **`update_display_name(self, workspace_id, display_name)`**：Update the displayName in workspace_meta.json and session_state.json.  Write-through: both files are updated so they stay consistent even when the workspace is not currently open in the frontend.
  - **`save_session_state(self, workspace_id, state)`**：Save frontend state to session_state.json in a workspace.  Sensitive fields are automatically stripped.  Also updates the lightweight ``workspace_meta.json`` used by :meth:`list_workspaces`.
  - **`load_session_state(self, workspace_id)`**：Load frontend state from session_state.json in a workspace.  Returns None if the workspace or state file doesn't exist.
  设计约束：公开方法必须返回安全数据（无密钥、无绝对路径）；异常应转换为 AppError 或 ToolResult 可恢复错误，而不是把 Python 堆栈直接交给前端。

模块级测试建议：至少覆盖（1）正常路径；（2）非法输入；（3）权限/路径拒绝；（4）上游失败时的 AppError 映射。相关用例见后文测试专章。


#### 模块 `py-src/data_formulator/datalake/workspace_metadata.py`（约 631 行）

**模块文档**：Metadata management for the Data Lake workspace.  This module defines the schema and operations for workspace.yaml, which tracks all data sources (uploaded files and data loader ingests).

符号统计：类 6 个，函数 7 个，模块级常量 4 个。下列说明覆盖全部符号。

**常量**：

- **常量 `METADATA_VERSION`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `METADATA_FILENAME`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `LOCK_FILENAME`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `MAX_LOCK_WAIT_SECONDS`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

##### 类 `WorkspaceLock`
- **基类**：无显式基类。
- **字段标注**：无模块级注解字段（可能使用 dataclass / 动态属性）。
- **方法数**：3。
- **文档字符串**：Context manager for acquiring an exclusive lock on workspace metadata. Prevents race conditions when multiple processes/threads modify metadata concurrently. Uses LockFileEx on Windows and fcntl.flock on Unix — both provide whole-file locking.
  - **`__init__(self, workspace_path, timeout)`**：内部实现细节见源码。
  - **`__enter__(self)`**：Acquire exclusive lock with timeout.
  - **`__exit__(self, exc_type, exc_val, exc_tb)`**：Release lock.
  设计约束：公开方法必须返回安全数据（无密钥、无绝对路径）；异常应转换为 AppError 或 ToolResult 可恢复错误，而不是把 Python 堆栈直接交给前端。

##### 函数 `make_json_safe(value)`
- **说明**：Convert a value (possibly containing numpy/pandas/pyarrow scalars) into a JSON/YAML-safe primitive structure.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 类 `ColumnInfo`
- **基类**：无显式基类。
- **字段标注**：name, dtype, description。
- **方法数**：2。
- **文档字符串**：Information about a single column in a table.
  - **`to_dict(self)`**：内部实现细节见源码。
  - **`from_dict(cls, data)`**：内部实现细节见源码。
  设计约束：公开方法必须返回安全数据（无密钥、无绝对路径）；异常应转换为 AppError 或 ToolResult 可恢复错误，而不是把 Python 堆栈直接交给前端。

##### 类 `TableMetadata`
- **基类**：无显式基类。
- **字段标注**：name, source_type, filename, file_type, created_at, content_hash, file_size, loader_type, loader_params, source_table, source_query, import_options, last_synced, row_count, columns, original_name, source_file, description。
- **方法数**：2。
- **文档字符串**：Metadata for a single table/file in the workspace.
  - **`to_dict(self)`**：Convert to dictionary for YAML serialization.
  - **`from_dict(cls, name, data)`**：Create from dictionary (YAML deserialization).
  设计约束：公开方法必须返回安全数据（无密钥、无绝对路径）；异常应转换为 AppError 或 ToolResult 可恢复错误，而不是把 Python 堆栈直接交给前端。

##### 类 `WorkspaceMetadata`
- **基类**：无显式基类。
- **字段标注**：version, created_at, updated_at, tables。
- **方法数**：8。
- **文档字符串**：Metadata for the entire workspace.
  - **`add_table(self, table)`**：Add or update a table in the metadata.
  - **`remove_table(self, name)`**：Remove a table from the metadata. Returns True if removed.
  - **`get_table(self, name)`**：Get metadata for a specific table.
  - **`list_tables(self)`**：List all table names.
  - **`search_tables(self, query, limit)`**：Search workspace tables by keyword across names, descriptions, column names, and column descriptions.  Returns a list of match dicts sorted by relevance (name match first). Each dict: ``{"name", "description", "matched_c
  - **`to_dict(self)`**：Convert to dictionary for YAML serialization.
  - **`from_dict(cls, data)`**：Create from dictionary (YAML deserialization).
  - **`create_new(cls)`**：Create a new empty workspace metadata.
  设计约束：公开方法必须返回安全数据（无密钥、无绝对路径）；异常应转换为 AppError 或 ToolResult 可恢复错误，而不是把 Python 堆栈直接交给前端。

##### 函数 `_read_metadata_file(workspace_path)`
- **说明**：Read and parse workspace.yaml.  **Caller must already hold the lock.**
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_write_metadata_file(workspace_path, metadata)`
- **说明**：Atomically write workspace.yaml.  **Caller must already hold the lock.**
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `load_metadata(workspace_path)`
- **说明**：Load workspace metadata from YAML file with file locking.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `save_metadata(workspace_path, metadata)`
- **说明**：Save workspace metadata to YAML file with atomic write and file locking.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `update_metadata(workspace_path, updater)`
- **说明**：Atomically read → update → write workspace metadata.  The *updater* callback receives the current :class:`WorkspaceMetadata` and should mutate it in place.  The entire read-modify-write is protected by a **single** lock acquisition, preventing the lost-update race condition that occurs when ``load_metadata`` and ``save_metadata`` each acquire their own independent lock.  Returns:     The updated :class:`WorkspaceMetadata` (useful for refreshing caches).
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `metadata_exists(workspace_path)`
- **说明**：Check if workspace metadata file exists.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 类 `ImportedFrom`
- **基类**：无显式基类。
- **字段标注**：source_type, ingested_at, loader_type, params, source_table, source_query, original_name, url, refresh_interval_seconds, dataset_name。
- **方法数**：2。
- **文档字符串**：Provenance of a dataset — how it was originally imported.  Mirrors the ExternalDataLoader model. Credentials are never stored; only safe params (output of get_safe_params()) are persisted.  source_type discriminates the shape:   - data_loader: loader_type + params + source_table/source_query   - upload: original_name   - url: url   - stream: url + refresh_interval_seconds   - example: dataset_name
  - **`to_dict(self)`**：内部实现细节见源码。
  - **`from_dict(cls, data)`**：内部实现细节见源码。
  设计约束：公开方法必须返回安全数据（无密钥、无绝对路径）；异常应转换为 AppError 或 ToolResult 可恢复错误，而不是把 Python 堆栈直接交给前端。

##### 类 `Derivation`
- **基类**：无显式基类。
- **字段标注**：source_tables, description, code, created_at。
- **方法数**：2。
- **文档字符串**：Lineage info for a derived table in a workspace.
  - **`to_dict(self)`**：内部实现细节见源码。
  - **`from_dict(cls, data)`**：内部实现细节见源码。
  设计约束：公开方法必须返回安全数据（无密钥、无绝对路径）；异常应转换为 AppError 或 ToolResult 可恢复错误，而不是把 Python 堆栈直接交给前端。

模块级测试建议：至少覆盖（1）正常路径；（2）非法输入；（3）权限/路径拒绝；（4）上游失败时的 AppError 映射。相关用例见后文测试专章。


### 目录 `py-src/data_formulator/knowledge`

**目录职责**：规则 / 工作流 / data-memory 存储

本目录内模块共享同一边界：只通过公开函数与相邻层通信。禁止跨层 import 私有 `_` 符号（测试除外）。新增文件必须补测试，并检查是否触发路径安全、日志脱敏、错误协议三条规则。


#### 模块 `py-src/data_formulator/knowledge/__init__.py`（约 3 行）

**模块文档**：该文件未写模块级 docstring。其职责由文件名与导出符号定义，修改时应补充文档字符串，避免意图漂移。

本文件主要为导出聚合或入口转发，无额外类/函数定义。


#### 模块 `py-src/data_formulator/knowledge/store.py`（约 727 行）

**模块文档**：Knowledge store — manages user knowledge files and data-source memory.  Each user has a ``knowledge/`` directory under their home with two sub-directories: ``rules`` and ``workflows``.  Every knowledge entry is a Markdown file with YAML front matter.  All file I/O is routed through :class:`ConfinedDir` for path safety.  Directory depth constraints:  - ``rules``: flat — only files directly under ``rules/`` (1 path part) - ``workflows``: one level of sub-directories (up to 2 path parts)

符号统计：类 2 个，函数 3 个，模块级常量 9 个。下列说明覆盖全部符号。

**常量**：

- **常量 `VALID_CATEGORIES`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `DATA_MEMORY_FILE`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `DATA_MEMORY_HARD_MAX`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `DATA_MEMORY_TEMPLATE`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `_MAX_DEPTH`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `_ENGLISH_STOPWORDS`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `_MIN_TOKEN_LEN`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `_CJK_ASCII_RE`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `_FM_PATTERN`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

##### 函数 `_tokenize_query(query)`
- **说明**：Split *query* into meaningful keyword tokens.  1. Space-split the query. 2. For tokens containing **both** CJK and ASCII characters, further    split into CJK segments and ASCII segments so each can match    independently.  E.g. ``"帮我分析ROI"`` → ``["帮我分析", "roi"]``. 3. Filter English stopwords and short ASCII tokens (≤ 2 chars). 4. Non-ASCII tokens (e.g. Chinese phrases) are kept regardless of    length — they participate in whole-substring matching.  When proper Chinese word segmentation is need
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `parse_front_matter(content)`
- **说明**：Parse YAML front matter from *content*.  Returns ``(metadata_dict, body_text)``.  On parse failure the metadata dict is empty and the full content is returned as the body (graceful degradation).
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 类 `KnowledgeItemMeta`
- **基类**：无显式基类。
- **字段标注**：无模块级注解字段（可能使用 dataclass / 动态属性）。
- **方法数**：2。
- **文档字符串**：Type-safe representation of a knowledge file's front matter.  Guarantees all fields are the expected types regardless of what YAML produced.  Construct via ``from_raw(meta_dict, fallback_stem)``.
  - **`__init__(self, title, source, created, description, always_apply, source_workspace_id, source_workspace_name)`**：内部实现细节见源码。
  - **`from_raw(cls, meta, fallback_stem)`**：Build from a raw YAML-parsed dict with type coercion.
  设计约束：公开方法必须返回安全数据（无密钥、无绝对路径）；异常应转换为 AppError 或 ToolResult 可恢复错误，而不是把 Python 堆栈直接交给前端。

##### 函数 `_ensure_front_matter(content, path, category)`
- **说明**：If *content* lacks front matter, prepend a minimal header.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 类 `KnowledgeStore`
- **基类**：无显式基类。
- **字段标注**：无模块级注解字段（可能使用 dataclass / 动态属性）。
- **方法数**：18。
- **文档字符串**：Manages the user's knowledge directory tree.  Usage::      store = KnowledgeStore(user_home)     items = store.list_all("rules")     content = store.read("workflows", "data-cleaning/handle-missing.md")     store.write("rules", "date-format.md", md_content)     store.delete("rules", "date-format.md")     results = store.search("ROI", categories=["rules", "workflows"])
  - **`__init__(self, user_home)`**：内部实现细节见源码。
  - **`read_data_memory(self)`**：Read the user's shared data-source memory, creating it if absent.
  - **`rewrite_data_memory(self, content)`**：Replace the user's shared data-source memory with Markdown text.
  - **`append_data_memory(self, content)`**：Append a durable Markdown note to the user's data-source memory.
  - **`replace_data_memory(self, old_text, new_text)`**：Replace exact text in data memory, returning replacement count.  An empty ``new_text`` deletes the matched text. By default only the first occurrence is replaced; ``replace_all`` updates every match.
  - **`_migrate_experiences_to_workflows(self)`**：Move legacy ``experiences/`` files into ``workflows/`` (one-time).  The feature was renamed from "experiences" to "workflows"; existing users have files under ``knowledge/experiences/``.  Move them so the rename is trans
  - **`_migrate_flat(self)`**：Move any workflows/subdir/file.md → workflows/file.md (one-time migration).
  - **`validate_path(category, relative_path)`**：Validate that *relative_path* conforms to *category* depth rules.  Raises ``ValueError`` on violation.
  - **`_jail(self, category)`**：内部实现细节见源码。
  - **`list_all(self, category)`**：List all knowledge entries in *category*.  Returns a list of dicts with ``title``, ``path``, ``source``, and ``created`` parsed from front matter. For rules, also includes ``description`` and ``alwaysApply``.
  - **`read(self, category, path)`**：Read the full content of a knowledge file.
  - **`write(self, category, path, content)`**：Create or update a knowledge file.  If *content* lacks YAML front matter, a minimal header is prepended. Validates body length and (for rules) description length against :data:`KNOWLEDGE_LIMITS`.
  - **`delete(self, category, path)`**：Delete a knowledge file.
  - **`find_workflow_by_workspace_id(self, workspace_id)`**：Return the workflow entry whose front matter records this workspace id.  Used by the session-scoped distillation flow (design-docs/24) to upsert: when re-distilling the same session, overwrite the same file even if the u
  - **`load_always_apply_rules(self)`**：Load rules with ``alwaysApply=true`` for system prompt injection.  Returns a list of ``{"title": ..., "body": ...}`` dicts. Non-alwaysApply rules are excluded (they are picked up via search). Returns empty list on failur
  - **`format_rules_block(self, rules)`**：Return a formatted prompt block for ``alwaysApply`` rules.  Args:     rules: Pre-loaded rules from :meth:`load_always_apply_rules`.            When *None* (default), calls ``load_always_apply_rules()``            automat
  - **`search(self, query, categories, max_results, table_names)`**：Search across knowledge categories.  Tokenizes *query* into keywords and scores each entry using multi-field weighted matching (title > filename > body). Whole-string exact matches and table-name overlaps receive additio
  - **`_match_score(query, title, stem, body_prefix)`**：Compute a relevance score (0 = no match).  Tokenizes *query*, then scores each token against multiple fields with weights normalised by token count.  Whole-string and table-name bonuses are added on top.  Non-manual sour
  设计约束：公开方法必须返回安全数据（无密钥、无绝对路径）；异常应转换为 AppError 或 ToolResult 可恢复错误，而不是把 Python 堆栈直接交给前端。

模块级测试建议：至少覆盖（1）正常路径；（2）非法输入；（3）权限/路径拒绝；（4）上游失败时的 AppError 映射。相关用例见后文测试专章。


### 目录 `py-src/data_formulator/routes`

**目录职责**：HTTP 蓝图：表、Agent、会话、知识、凭证、日志

本目录内模块共享同一边界：只通过公开函数与相邻层通信。禁止跨层 import 私有 `_` 符号（测试除外）。新增文件必须补测试，并检查是否触发路径安全、日志脱敏、错误协议三条规则。


#### 模块 `py-src/data_formulator/routes/__init__.py`（约 3 行）

**模块文档**：该文件未写模块级 docstring。其职责由文件名与导出符号定义，修改时应补充文档字符串，避免意图漂移。

本文件主要为导出聚合或入口转发，无额外类/函数定义。


#### 模块 `py-src/data_formulator/routes/agents.py`（约 1096 行）

**模块文档**：该文件未写模块级 docstring。其职责由文件名与导出符号定义，修改时应补充文档字符串，避免意图漂移。

符号统计：类 0 个，函数 23 个，模块级常量 1 个。下列说明覆盖全部符号。

**常量**：

- **常量 `PREVIEW_ROW_LIMIT`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

##### 函数 `_get_ui_lang()`
- **说明**：Extract the primary language code from the Accept-Language header.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `get_language_instruction()`
- **说明**：Read the UI language from the Accept-Language header and build the prompt instruction.  mode: "full" for text-heavy agents, "compact" for code-generation agents.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_get_knowledge_store(identity_id)`
- **说明**：Create a KnowledgeStore for the given user, or None on failure.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `preview_data_operation()`
- **说明**：Return bounded display rows for an opaque operation plan.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_with_warnings(gen)`
- **说明**：Wrap an NDJSON generator to flush accumulated stream warnings.  Any code running during chunk generation (e.g. agent helpers) may call :func:`collect_stream_warning`.  This wrapper drains the accumulated warnings before each application chunk so the frontend receives them in chronological order.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_set_cors(response)`
- **说明**：Set CORS headers from server configuration.  By default no ``Access-Control-Allow-Origin`` header is emitted (same-origin only).  To allow cross-origin requests set the ``CORS_ORIGIN`` env-var (e.g. ``CORS_ORIGIN=https://my-embed-host``). Use ``CORS_ORIGIN=*`` only for development / fully trusted networks.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `get_client(model_config, trusted)`
- **说明**：Build a LiteLLM client for *model_config*.  ``trusted`` marks a config that came from the server-side registry rather than from a request body.  Callers that already resolved a config through ``model_registry`` pass ``trusted=True``; everything reached from an HTTP payload must leave it ``False``.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `list_global_models()`
- **说明**：Return all globally configured models instantly, without connectivity checks.  The frontend calls this first to render the model list immediately (with a 'checking' status), then calls /check-available-models to get real statuses.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `check_available_models()`
- **说明**：Return all globally configured models with their connectivity status.  Connectivity checks run in parallel (ThreadPoolExecutor) so the total wall-clock time equals the slowest single model, not the sum of all. Sensitive credentials (api_key) are never sent to the client.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `test_model()`
- **说明**：模块级函数 `test_model` 封装可复用算法或路由处理。参数经过校验后进入核心逻辑，返回结构化结果或通过异常表达业务失败。
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `process_data_on_load_request()`
- **说明**：模块级函数 `process_data_on_load_request` 封装可复用算法或路由处理。参数经过校验后进入核心逻辑，返回结构化结果或通过异常表达业务失败。
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `sort_data_request()`
- **说明**：模块级函数 `sort_data_request` 封装可复用算法或路由处理。参数经过校验后进入核心逻辑，返回结构化结果或通过异常表达业务失败。
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `derive_starter_questions_request()`
- **说明**：Generate a few short, data-tailored starter exploration questions.  Called once when a workspace's set of root tables changes (e.g. after data is loaded). Input: ``input_tables`` (list of {name, columns, sample_rows, description}) and ``model``. Returns ``{"result": [..]}``.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `analyst_streaming()`
- **说明**：Unified AnalystAgent streaming endpoint (design-docs/35 + /36).  The single ``AnalystAgent`` subsumes both data exploration and report writing: it gathers with inspection tools, commits one action per turn (``visualize`` / ``ask_user`` / ``delegate`` / ``write_report``), and streams the report live on the ``report`` channel (same ``text_delta`` event the frontend already routes).  Streams newline-delimited JSON. Terminal events: ``completion`` (the run finished or hit its budget), ``interact`` (
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `request_code_expl()`
- **说明**：模块级函数 `request_code_expl` 封装可复用算法或路由处理。参数经过校验后进入核心逻辑，返回结构化结果或通过异常表达业务失败。
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `refresh_derived_data()`
- **说明**：Re-run Python transformation code with updated input data to refresh a derived table.  Security: The code must have been previously signed by the server (via ``code_signing.sign_result``) when it was first generated by an agent. The frontend must send the original ``code_signature`` back alongside the code.  This endpoint verifies the signature before executing, preventing execution of tampered or injected code.  This endpoint: 1. Verifies the code signature (HMAC-SHA256) 2. Gets input tables fr
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `workspace_name()`
- **说明**：Generate a short display name for the current workspace.  Called after the first agent interaction to auto-name the workspace. Expects: { model: <model_config>, context: { tables: [...], userQuery: "..." } } Returns: { status: "success", data: { display_name: "short name" } }
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `nl_to_filter()`
- **说明**：Translate a natural language filter instruction to structured conditions.  Request body:     model: model config object (same as other agent routes)     columns: [{name, type}, ...]  — the table's column schema     instruction: str — the user's NL filter description  Response:     {status: "success", data: {conditions, sort_columns?, sort_order?, limit?}}
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `classify_chart_intent()`
- **说明**：Classify a chart-prompt as STYLE or DATA.  Used by the encoding-shelf input on Enter to route the prompt to either the chart-restyle agent (visual changes) or the data agent (data shape / chart-type changes). Multilingual by design — keyword heuristics are too brittle for non-English prompts. See agent_simple.classify_chart_intent and the chat discussion in design history.  Request body:     model: model config object     instruction: str — the user's NL prompt  Response:     {status: "success",
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `chart_restyle()`
- **说明**：Apply a natural-language STYLE instruction to a Vega-Lite spec.  Request body:     model: model config object (same shape as other agent routes)     instruction: str — the user's NL style instruction     vlSpec: dict — current Vega-Lite spec (data block already stripped client-side)     chartType: str — chart template label (e.g. "Bar Chart")     dataSample: list[dict] (optional) — first ~10 rows of the underlying table  Response:     On success: {status: "success", data: {vlSpec: <new spec>, ra
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `scratch_upload()`
- **说明**：Upload a file to the workspace scratch/ folder.  Accepts multipart/form-data with a 'file' field. Returns: { status: "success", data: { path, url } }
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `scratch_serve(filename)`
- **说明**：Serve a file from the workspace scratch/ folder.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `data_loading_chat()`
- **说明**：Conversational data loading agent endpoint.  Streams newline-delimited JSON events (SSE-style).
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

模块级测试建议：至少覆盖（1）正常路径；（2）非法输入；（3）权限/路径拒绝；（4）上游失败时的 AppError 映射。相关用例见后文测试专章。


#### 模块 `py-src/data_formulator/routes/credentials.py`（约 74 行）

**模块文档**：REST API for credential management (list / store / delete).  All endpoints are identity-scoped: the current user (from :func:`get_identity_id`) can only access their own stored credentials. Credential *values* are never returned to the frontend — ``/list`` only reveals which source_keys have stored credentials.

符号统计：类 0 个，函数 3 个，模块级常量 0 个。下列说明覆盖全部符号。

##### 函数 `list_credentials()`
- **说明**：List source_keys with stored credentials (no secrets exposed).
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `store_credential()`
- **说明**：Store or update encrypted credentials for a source.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `delete_credential()`
- **说明**：Delete stored credentials for a source.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

模块级测试建议：至少覆盖（1）正常路径；（2）非法输入；（3）权限/路径拒绝；（4）上游失败时的 AppError 映射。相关用例见后文测试专章。


#### 模块 `py-src/data_formulator/routes/demo_stream.py`（约 1169 行）

**模块文档**：Demo data REST APIs for streaming/refresh demos.  Design Philosophy: - Each endpoint returns a COMPLETE dataset (not just a single row) - Datasets are meaningful on their own for analysis/visualization - When refreshed, datasets change over time:   * New rows may be added (accumulating data)   * Existing values may update (latest readings) - This allows tracking trends, changes, and patterns over time  Example Use Cases: - Stock prices: "Last 30 days" grows daily with new data - Earthquakes: All quakes since a start date accumulates  Rate Limiting: - External API routes are rate-limited to prevent abuse - Limits are set per IP address using Flask-Limiter

符号统计：类 0 个，函数 15 个，模块级常量 12 个。下列说明覆盖全部符号。

**常量**：

- **常量 `EARTHQUAKE_RATE_LIMIT`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `WEATHER_RATE_LIMIT`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `YFINANCE_RATE_LIMIT`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `MOCK_RATE_LIMIT`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `WEATHER_CITIES`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `DEFAULT_SYMBOLS`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `SP100_SYMBOLS`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `_SALES_PRODUCTS`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `_SALES_REGIONS`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `_SALES_REGION_WEIGHTS`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `_SALES_CHANNELS`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `_SALES_CHANNEL_WEIGHTS`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

##### 函数 `_set_cors(response)`
- **说明**：Set CORS headers from CORS_ORIGIN env-var (same logic as agent_routes).
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `make_csv_response(rows, filename)`
- **说明**：Convert list of dicts to CSV text response
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `get_earthquakes()`
- **说明**：Earthquakes from USGS. Dataset grows as new quakes occur.  Query params:     - timeframe: 'hour', 'day', 'week', 'month' (default: 'day')     - min_magnitude: Minimum magnitude filter (default: 0)     - max_magnitude: Maximum magnitude filter (optional)     - since: ISO date string - only return quakes after this time     - limit: Maximum number of results (default: 20000, max: 20000)     - use_query_api: 'true' to use query API for more data (default: 'false' for quick summary)  Use case:     -
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `get_weather()`
- **说明**：Current weather for major US cities. Updates every 15 minutes.  Query params:     - cities: Comma-separated list of city names (default: all cities)               Example: cities=Seattle,New York,Los Angeles     - fields: Comma-separated list of fields to include (default: all)               Available: temperature,humidity,wind,precipitation,pressure,cloud_cover  Recommended refresh: 300 seconds
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `get_weather_history()`
- **说明**：Hourly weather history for one or more locations. Dataset grows with each hour.  Query params:     - city: City name (default: Seattle) - one of the WEATHER_CITIES               Can also be comma-separated list: city=Seattle,New York,Los Angeles     - cities: Alternative parameter name for comma-separated list of city names     - days: Number of past days to include (default: 7, max: 92 for archive, 14 for forecast)     - use_archive: 'true' to use historical archive API (for data older than 5 d
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `get_weather_forecast()`
- **说明**：Multi-day weather forecast for US cities. Updates every few hours.  Query params:     - cities: Comma-separated list of city names (default: all cities)               Example: cities=Seattle,New York,Los Angeles     - days: Number of forecast days (default: 7, max: 16)     - hourly: 'true' to get hourly forecast data (default: 'false' for daily)  Use case:     - Compare forecasts across multiple cities     - Track upcoming weather patterns     - Plan based on forecasted conditions     - Use hour
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `get_weather_today()`
- **说明**：Today's current weather for multiple US cities - perfect for comparison.  Query params:     - cities: Comma-separated list of city names (default: all cities)               Example: cities=Seattle,New York,Los Angeles,Miami     - limit: Maximum number of cities to return (default: 20, max: 50)  Use case:     - Compare current weather conditions across US cities     - Visualize temperature, humidity, wind patterns geographically     - Great for maps, bar charts, and comparison visualizations     
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_yf_is_valid(val)`
- **说明**：Check if value is valid (not NaN/None)
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_yf_format_timestamp(date_obj)`
- **说明**：Convert pandas Timestamp or datetime to string
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `get_yfinance_history()`
- **说明**：6-month daily stock price history via yfinance.  Returns daily OHLCV data for the last 6 months.  Query params:     - symbols: comma-separated stock symbols (default: AAPL,MSFT,GOOGL,AMZN,META,NVDA,TSLA)  Example:     /api/demo-stream/yfinance/history?symbols=AAPL,MSFT,GOOGL  Recommended refresh: 3600 seconds (1 hour)
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `get_yfinance_recent()`
- **说明**：Recent intraday stock prices (15-minute intervals) via yfinance.  Returns 15-minute interval data for the last 5 trading days. yfinance typically provides intraday data for the last 5-7 days only.  Query params:     - symbols: comma-separated stock symbols (default: AAPL,MSFT,GOOGL,AMZN,META,NVDA,TSLA)  Example:     /api/demo-stream/yfinance/recent?symbols=AAPL,MSFT,GOOGL  Recommended refresh: 300 seconds (5 minutes) during market hours
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `get_yfinance_financials()`
- **说明**：Key financial metrics snapshot via yfinance for S&P 100 companies.  Returns current financial data for each stock including market cap, P/E ratio, EPS, dividend yield, 52-week range, and more.  Query params:     - symbols: comma-separated stock symbols (default: all S&P 100 companies)  Example:     /api/demo-stream/yfinance/financials     /api/demo-stream/yfinance/financials?symbols=AAPL,MSFT,GOOGL,AMZN  Recommended refresh: 3600 seconds (1 hour)
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_generate_sale_transaction(timestamp)`
- **说明**：Generate a single sale transaction
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `get_live_sales()`
- **说明**：Simulated live sales feed with accumulating transaction history. Data accumulates in memory and maintains a rolling record of the last 1000 transactions. Each refresh may add new transactions and returns the complete accumulated dataset.  Query params:     - limit: Maximum number of records to return (default: 1000, max: 1000)  Recommended refresh: 1-5 seconds
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `get_info()`
- **说明**：List all available demo data endpoints with their parameters
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

模块级测试建议：至少覆盖（1）正常路径；（2）非法输入；（3）权限/路径拒绝；（4）上游失败时的 AppError 映射。相关用例见后文测试专章。


#### 模块 `py-src/data_formulator/routes/knowledge.py`（约 393 行）

**模块文档**：Knowledge management API — CRUD + search + workflow distillation.  All endpoints use ``POST`` with JSON body.  Access is scoped to the current user via ``get_identity_id()`` and confined via ``ConfinedDir``.

符号统计：类 0 个，函数 16 个，模块级常量 2 个。下列说明覆盖全部符号。

**常量**：

- **常量 `_EXP_PREFIX_RE`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `_UNSAFE_FILENAME_CHARS`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

##### 函数 `_get_store()`
- **说明**：模块级函数 `_get_store` 封装可复用算法或路由处理。参数经过校验后进入核心逻辑，返回结构化结果或通过异常表达业务失败。
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_require_json_field(data, field)`
- **说明**：模块级函数 `_require_json_field` 封装可复用算法或路由处理。参数经过校验后进入核心逻辑，返回结构化结果或通过异常表达业务失败。
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `knowledge_limits()`
- **说明**：Return body-length and description limits so the frontend stays in sync.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `data_memory_read()`
- **说明**：Read the current user's shared data-source memory.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `data_memory_append()`
- **说明**：Append a durable note to the current user's data-source memory.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `data_memory_rewrite()`
- **说明**：Replace the current user's data-source memory.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `knowledge_list()`
- **说明**：模块级函数 `knowledge_list` 封装可复用算法或路由处理。参数经过校验后进入核心逻辑，返回结构化结果或通过异常表达业务失败。
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `knowledge_read()`
- **说明**：模块级函数 `knowledge_read` 封装可复用算法或路由处理。参数经过校验后进入核心逻辑，返回结构化结果或通过异常表达业务失败。
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `knowledge_write()`
- **说明**：模块级函数 `knowledge_write` 封装可复用算法或路由处理。参数经过校验后进入核心逻辑，返回结构化结果或通过异常表达业务失败。
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `knowledge_delete()`
- **说明**：模块级函数 `knowledge_delete` 封装可复用算法或路由处理。参数经过校验后进入核心逻辑，返回结构化结果或通过异常表达业务失败。
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `knowledge_search()`
- **说明**：模块级函数 `knowledge_search` 封装可复用算法或路由处理。参数经过校验后进入核心逻辑，返回结构化结果或通过异常表达业务失败。
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `distill_workflow()`
- **说明**：Distill user-visible analysis context into a reusable workflow.  Session-scoped payload (design-docs/24): ``workflow_context`` carries a list of ``threads`` (one per leaf derived table the user has on screen), each with its own chronological ``events`` array. ``workspace_id`` + ``workspace_name`` bind the resulting file to the active session so re-distilling upserts the same file.  Required body fields: ``workflow_context`` and ``model``. Optional: ``user_instruction`` (natural-language focus hi
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_apply_session_front_matter(content, workspace_id, workspace_name)`
- **说明**：Override / inject session-binding fields in the workflow front matter.  - Sets the visible ``title`` to the agent-emitted descriptive   ``subtitle`` (preferred) or the pre-existing ``title``, with any   legacy ``Workflow from <name>: `` prefix stripped. The ``subtitle``   field is removed from the front matter once consumed. - Consumes the agent-emitted short ``filename`` hint (removed from the   front matter) and returns it so the caller can name the file without   using the long descriptive ti
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_strip_workflow_prefix(title)`
- **说明**：模块级函数 `_strip_workflow_prefix` 封装可复用算法或路由处理。参数经过校验后进入核心逻辑，返回结构化结果或通过异常表达业务失败。
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_serialize_front_matter(meta, body)`
- **说明**：Render front matter back to YAML, preserving body verbatim.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_workflow_filename(title)`
- **说明**：Slugify an LLM-supplied name into a clean, safe ``.md`` filename.  Re-distilling a session upserts by ``source_workspace_id`` (see caller), so the file is replaced even when the name changes. ``safe_data_filename`` enforces the security boundary (basename only, no ``.``/``..``); the slug step just keeps separators and reserved chars out so the name is clean and portable. Unicode (e.g. CJK) is preserved.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

模块级测试建议：至少覆盖（1）正常路径；（2）非法输入；（3）权限/路径拒绝；（4）上游失败时的 AppError 映射。相关用例见后文测试专章。


#### 模块 `py-src/data_formulator/routes/logs.py`（约 117 行）

**模块文档**：Server log inspection routes.  Data Formulator persists all server + Python-execution logs to a rotating file under ``<DATA_FORMULATOR_HOME>/logs/data_formulator.log`` (configured in ``app.configure_file_logging``). This is the artifact a user can send when reporting a problem.  Access policy — logs are **server-side only**:  * In **local single-user mode** (``is_local_mode()`` is true) the user *is*   the server operator, so these endpoints expose the log to the UI (view /   tail / download). * In any **hosted / multi-user** deployment these endpoints return   ``ACCESS_DENIED``. The operator reads the file directly on the server host;   end users never see server logs.  Routes:   GET /api/logs/info      — metadata: path, size, existence, local-mode flag   GET /api/logs/tail      — last N 

符号统计：类 0 个，函数 5 个，模块级常量 2 个。下列说明覆盖全部符号。

**常量**：

- **常量 `_MAX_TAIL_LINES`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `_DEFAULT_TAIL_LINES`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

##### 函数 `_require_local_mode()`
- **说明**：Reject the request unless running in local single-user mode.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_log_path()`
- **说明**：Resolve the active log file path (set by configure_file_logging).
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `logs_info()`
- **说明**：Return log file metadata (local mode only).
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `logs_tail()`
- **说明**：Return the last N lines of the current log file (local mode only).
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `logs_download()`
- **说明**：Download the current log file as an attachment (local mode only).
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

模块级测试建议：至少覆盖（1）正常路径；（2）非法输入；（3）权限/路径拒绝；（4）上游失败时的 AppError 映射。相关用例见后文测试专章。


#### 模块 `py-src/data_formulator/routes/model_endpoints.py`（约 94 行）

**模块文档**：Per-user history of non-secret model endpoint configurations.

符号统计：类 0 个，函数 6 个，模块级常量 4 个。下列说明覆盖全部符号。

**常量**：

- **常量 `_FILENAME`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `_MAX_ENTRIES`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `_MAX_FIELD_LENGTH`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `_FIELDS`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

##### 函数 `_history_path(identity_id)`
- **说明**：模块级函数 `_history_path` 封装可复用算法或路由处理。参数经过校验后进入核心逻辑，返回结构化结果或通过异常表达业务失败。
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_sanitize_entry(value)`
- **说明**：模块级函数 `_sanitize_entry` 封装可复用算法或路由处理。参数经过校验后进入核心逻辑，返回结构化结果或通过异常表达业务失败。
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_read_history(path)`
- **说明**：模块级函数 `_read_history` 封装可复用算法或路由处理。参数经过校验后进入核心逻辑，返回结构化结果或通过异常表达业务失败。
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_write_history(path, entries)`
- **说明**：模块级函数 `_write_history` 封装可复用算法或路由处理。参数经过校验后进入核心逻辑，返回结构化结果或通过异常表达业务失败。
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `list_model_endpoints()`
- **说明**：模块级函数 `list_model_endpoints` 封装可复用算法或路由处理。参数经过校验后进入核心逻辑，返回结构化结果或通过异常表达业务失败。
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `remember_model_endpoint()`
- **说明**：模块级函数 `remember_model_endpoint` 封装可复用算法或路由处理。参数经过校验后进入核心逻辑，返回结构化结果或通过异常表达业务失败。
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

模块级测试建议：至少覆盖（1）正常路径；（2）非法输入；（3）权限/路径拒绝；（4）上游失败时的 AppError 映射。相关用例见后文测试专章。


#### 模块 `py-src/data_formulator/routes/sessions.py`（约 374 行）

**模块文档**：Workspace management routes.  All backends expose the same workspace CRUD API. The ephemeral backend selects a TTL-managed local WorkspaceManager in ``workspace_factory``.  Routes:   POST /api/sessions/save        — auto-persist state to active workspace   GET  /api/sessions/list        — list all workspaces   POST /api/sessions/load        — switch to a workspace (open it)   POST /api/sessions/delete      — delete a workspace   POST /api/sessions/create      — create a new workspace   POST /api/sessions/rename      — rename a workspace   POST /api/sessions/update-meta — update display name (lightweight, no full state)   POST /api/sessions/export      — export active workspace as zip   POST /api/sessions/import      — import workspace from zip  Note: URL prefix kept as /api/sessions for fr

符号统计：类 0 个，函数 12 个，模块级常量 0 个。下列说明覆盖全部符号。

##### 函数 `_raise_if_storage_full(exc)`
- **说明**：Convert disk-full writes into a user-facing API error.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `save_session()`
- **说明**：Auto-persist frontend state to the active workspace.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `list_sessions()`
- **说明**：List all workspaces for the current user.  Optional query param ``source_identity`` (e.g. ``browser:<uuid>``) lets an authenticated ``user:`` identity peek at an anonymous identity's workspace list — used by the migration dialog to check whether there is data to import.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `load_session()`
- **说明**：Switch to a workspace (open it) and return its state.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `delete_session()`
- **说明**：Delete a workspace.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `create_workspace_route()`
- **说明**：Create a new workspace.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `rename_workspace_route()`
- **说明**：Rename a workspace (change its folder ID).
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `update_workspace_meta()`
- **说明**：Update workspace display name without writing full session state.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `export_session()`
- **说明**：Export a workspace as a zip.  Body: ``{ "state": {...}, "workspace_id": "session_..." }``  ``workspace_id`` identifies which workspace's files to package. This avoids the need for an ``X-Workspace-Id`` header, allowing export from the landing page where no workspace is active.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `import_session()`
- **说明**：Import a workspace from a zip.  The optional ``workspace_id`` form field specifies the target workspace.  If the workspace doesn't exist yet it is created automatically, so callers can generate a fresh ID client-side. When omitted, falls back to the ``X-Workspace-Id`` header.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `migrate_workspaces()`
- **说明**：Move workspaces from an anonymous browser identity to the current user.  Body: ``{ "source_identity": "browser:<uuid>" }``  Only allowed when the current identity is ``user:*`` and the source is ``browser:*``.  New workspaces are moved; existing ones are merged (new data files + metadata entries added).  The anonymous source workspaces are deleted after a successful move.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `cleanup_anonymous()`
- **说明**：Delete all workspaces belonging to an anonymous browser identity.  Body: ``{ "source_identity": "browser:<uuid>" }``  Used by the "Start Fresh" migration option so the anonymous data does not linger and trigger another migration prompt later.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

模块级测试建议：至少覆盖（1）正常路径；（2）非法输入；（3）权限/路径拒绝；（4）上游失败时的 AppError 映射。相关用例见后文测试专章。


#### 模块 `py-src/data_formulator/routes/tables.py`（约 1267 行）

**模块文档**：该文件未写模块级 docstring。其职责由文件名与导出符号定义，修改时应补充文档字符串，避免意图漂移。

符号统计：类 0 个，函数 36 个，模块级常量 3 个。下列说明覆盖全部符号。

**常量**：

- **常量 `_LARGE_TABLE_THRESHOLD`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `_COLUMN_STATS_LEVELS_LIMIT`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `_CSV_STREAM_CHUNK_ROWS`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

##### 函数 `_get_workspace()`
- **说明**：Get workspace for the current identity.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_should_use_duckdb(workspace, table_name)`
- **说明**：Return True if the table is a large parquet file that benefits from DuckDB.  Small parquet tables are faster to handle with pandas (avoids DuckDB connection overhead and repeated YAML reads).
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_quote_duckdb(col)`
- **说明**：Quote identifier for DuckDB (double quotes, escape internal quotes).
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_quote_lit(value)`
- **说明**：Quote a literal for safe DuckDB SQL interpolation.  Supports str/int/float/bool/None and pandas/datetime values via ``str()``. Numeric values are emitted as-is; strings are single-quoted with internal quotes doubled.  Used by the column-filter WHERE builder (design-doc 31).
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_column_type_map(columns_info)`
- **说明**：``get_parquet_schema`` columns → ``{name: TYPE_UPPER}`` lookup.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_extend_eod_if_timestamp(value, col_type_upper)`
- **说明**：Promote a bare ``YYYY-MM-DD`` upper bound to end-of-day for timestamp columns so a date-only ``<=`` filter is naturally inclusive of that day.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_build_filter_where_duckdb(filters, columns, column_types)`
- **说明**：Build a DuckDB ``WHERE`` clause from the three-op filter vocabulary (design-doc 31): ``range`` / ``in`` / ``contains``. Returns either an empty string or ``" WHERE <clause>"`` ready to splice into a query.  Unknown ops, filters on missing columns, and malformed entries are silently dropped — the route must continue to work even if the frontend sends stale state. An empty ``in`` list collapses to ``WHERE FALSE`` for the field so the user sees an empty grid.  ``search`` is a first-class global qui
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_apply_filters_pandas(df, filters, search)`
- **说明**：Pandas mirror of :func:`_build_filter_where_duckdb`.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_dedup_dataframe_columns(df)`
- **说明**：Remove duplicate columns from a DataFrame, keeping the first occurrence.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_dedup_list(items)`
- **说明**：Remove duplicates from a list while preserving order.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_build_parquet_sample_sql(columns, aggregate_fields_and_functions, select_fields, method, order_by_fields, sample_size, offset, filters, column_types, search)`
- **说明**：Build DuckDB SQL for sampling (and optional aggregation) over parquet. Returns (main_sql, count_sql) where each contains {parquet} placeholder.  When ``filters`` is provided (design-doc 31), a WHERE clause is added on the base table — pre-aggregation in the aggregate branch and pre-``ROW_NUMBER()`` in the non-aggregate branch so the surfaced ``#rowId`` is contiguous over the filtered slice.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_table_metadata_to_source_metadata(meta)`
- **说明**：Convert workspace TableMetadata to API source_metadata dict (for refresh).
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `open_workspace()`
- **说明**：Open the Data Formulator home directory in the system file manager.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `list_tables()`
- **说明**：List all tables in the current workspace (datalake).
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_apply_aggregation_and_sample(df, aggregate_fields_and_functions, select_fields, method, order_by_fields, sample_size, offset, filters, search)`
- **说明**：Apply filters (optional), aggregation (optional), then sample with ordering. Returns (sampled_df, total_row_count_after_aggregation).
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_fetch_column_levels_duckdb(workspace, table_name, column)`
- **说明**：Top-N value/count pairs (count-desc) for a single low-card column.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_safe_levels(levels)`
- **说明**：Run levels (which may contain pandas/numpy scalars) through df_to_safe_records.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `sample_table()`
- **说明**：Sample a table from the workspace. Uses DuckDB for parquet (no full load).
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `get_table_data()`
- **说明**：Get data from a specific table in the workspace. Uses DuckDB for parquet (LIMIT/OFFSET only).
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_read_upload_to_df(content, file_type)`
- **说明**：Parse uploaded file bytes into a DataFrame for parquet conversion.  For Excel files the target sheet is resolved by :func:`_resolve_excel_sheet`.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_resolve_excel_sheet(content, table_name, sheet_hint)`
- **说明**：Pick the correct sheet from an Excel workbook.  Resolution order:  1. **Validated hint** — if the frontend sent *sheet_hint* **and** that    name actually exists in the workbook, use it directly. 2. **Suffix match** — for each real sheet name, check whether    ``table_name`` ends with ``_<sheet_lower>``.  This handles the    common pattern ``query_产品利润_xlsx_sheet1``. 3. **Substring match** — looser: ``sheet_lower in table_name``. 4. **Fallback** — first sheet (index 0).
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `create_table()`
- **说明**：Create a new table from uploaded file or raw data in the workspace.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `parse_file()`
- **说明**：Parse an uploaded file and return data as JSON without saving to workspace.  Used for client-side preview of formats that the browser cannot parse natively (e.g. legacy .xls).
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `sync_table_data()`
- **说明**：Update an existing workspace table's parquet with new row data.  Used when the frontend has fresher data than the workspace (e.g., from stream refresh) and needs to sync it so sandbox code reads the latest data.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `drop_table()`
- **说明**：Drop a table from the workspace.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `upload_db_file()`
- **说明**：No longer used: storage is workspace/datalake, not DuckDB. Kept for API compatibility.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `download_db_file()`
- **说明**：No longer used: storage is workspace/datalake. Kept for API compatibility.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_stream_csv_from_duckdb(workspace, table_name, delimiter)`
- **说明**：Use DuckDB native COPY to export CSV — bypasses pandas entirely.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_stream_csv_from_dataframe(df, delimiter)`
- **说明**：Stream CSV from a pandas DataFrame in chunks to limit memory.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `export_table_csv()`
- **说明**：Export a workspace table as CSV (or TSV) file download.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `reset_db_file()`
- **说明**：Reset the workspace for the current session (removes all tables and files).
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_is_numeric_duckdb_type(col_type)`
- **说明**：Return True if DuckDB/parquet type is numeric for min/max/avg.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `analyze_table()`
- **说明**：Get basic statistics about a table in the workspace. Uses DuckDB for parquet (no full load).  For low-cardinality columns (``unique_count <= _COLUMN_STATS_LEVELS_LIMIT``) also returns ``levels`` and parallel ``level_counts`` arrays so the data- grid column filter popover (design-doc 31) can render a checklist synchronously without a follow-up fetch.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `sanitize_table_name(table_name)`
- **说明**：Sanitize a table name for use in the workspace.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `classify_and_raise_db_error(error)`
- **说明**：Classify a database/workspace error and raise ``AppError``.  **Security rule**: the raised message is *never* derived from ``str(error)``.  Only pre-defined, human-written strings are used. The full exception is logged server-side for debugging.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `sanitize_db_error_message(error)`
- **说明**：Legacy wrapper — prefer ``classify_and_raise_db_error`` for new code.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

模块级测试建议：至少覆盖（1）正常路径；（2）非法输入；（3）权限/路径拒绝；（4）上游失败时的 AppError 映射。相关用例见后文测试专章。


### 目录 `py-src/data_formulator/sandbox`

**目录职责**：代码隔离执行：local / docker / not_a_sandbox

本目录内模块共享同一边界：只通过公开函数与相邻层通信。禁止跨层 import 私有 `_` 符号（测试除外）。新增文件必须补测试，并检查是否触发路径安全、日志脱敏、错误协议三条规则。


#### 模块 `py-src/data_formulator/sandbox/__init__.py`（约 22 行）

**模块文档**：该文件未写模块级 docstring。其职责由文件名与导出符号定义，修改时应补充文档字符串，避免意图漂移。

符号统计：类 0 个，函数 1 个，模块级常量 1 个。下列说明覆盖全部符号。

**常量**：

- **常量 `SANDBOX_OPTIONS`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

##### 函数 `create_sandbox(sandbox)`
- **说明**：Instantiate a sandbox from a config string.  Parameters ---------- sandbox : str     ``"local"`` (default) or ``"docker"``.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

模块级测试建议：至少覆盖（1）正常路径；（2）非法输入；（3）权限/路径拒绝；（4）上游失败时的 AppError 映射。相关用例见后文测试专章。


#### 模块 `py-src/data_formulator/sandbox/base.py`（约 55 行）

**模块文档**：Abstract base class for code-execution sandboxes.  Every sandbox backend must subclass :class:`Sandbox` and implement :meth:`run_python_code`.  The return contract is a dict with:  * ``{'status': 'ok', 'content': <pandas.DataFrame>}``  on success * ``{'status': 'error', 'content': '<error message>'}`` on failure

符号统计：类 1 个，函数 0 个，模块级常量 0 个。下列说明覆盖全部符号。

##### 类 `Sandbox`
- **基类**：ABC。
- **字段标注**：无模块级注解字段（可能使用 dataclass / 动态属性）。
- **方法数**：1。
- **文档字符串**：Base class for sandbox execution backends.
  - **`run_python_code(self, code, workspace, output_variable)`**：Execute a Python script and return the resulting DataFrame.  The script runs with the workspace directory as its working directory (read-only).  Scripts can therefore read files directly via e.g. ``pd.read_csv("file.csv" 在隔离环境执行 Agent 生成代码，捕获指定 output_variable 的 DataFrame，失败时返回安全错误而非堆栈。
  设计约束：公开方法必须返回安全数据（无密钥、无绝对路径）；异常应转换为 AppError 或 ToolResult 可恢复错误，而不是把 Python 堆栈直接交给前端。

模块级测试建议：至少覆盖（1）正常路径；（2）非法输入；（3）权限/路径拒绝；（4）上游失败时的 AppError 映射。相关用例见后文测试专章。


#### 模块 `py-src/data_formulator/sandbox/docker_sandbox.py`（约 263 行）

**模块文档**：Docker-based sandbox for executing Python code in an isolated container.  The workspace directory is mounted **read-only** as the container's working directory so user scripts can read data files via e.g. ``pd.read_csv("file.csv")`` but cannot tamper with the host filesystem. The output DataFrame is serialised to Parquet and read back via a bind-mounted output directory.

符号统计：类 1 个，函数 1 个，模块级常量 2 个。下列说明覆盖全部符号。

**常量**：

- **常量 `DEFAULT_DOCKER_IMAGE`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `DEFAULT_TIMEOUT`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

##### 函数 `_safe_error_response(content, detail)`
- **说明**：模块级函数 `_safe_error_response` 封装可复用算法或路由处理。参数经过校验后进入核心逻辑，返回结构化结果或通过异常表达业务失败。
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 类 `DockerSandbox`
- **基类**：Sandbox。
- **字段标注**：无模块级注解字段（可能使用 dataclass / 动态属性）。
- **方法数**：3。
- **文档字符串**：Execute Python code inside a Docker container.  The workspace directory is bind-mounted **read-only** as the container's working directory.  Scripts read files directly via e.g. ``pd.read_csv("file.csv")``.  Parameters ---------- docker_image : str     Docker image tag to use.  The default ``data-formulator-sandbox``     is built from ``Dockerfile.sandbox`` and has all required packages     pre-in
  - **`__init__(self, docker_image, timeout)`**：内部实现细节见源码。
  - **`run_python_code(self, code, workspace, output_variable)`**：Execute *code* in a Docker container and return the result DataFrame.  The wrapper script runs the user code, then serialises ``output_variable`` to Parquet so the host can read it back.  Returns ------- dict     ``{'sta 在隔离环境执行 Agent 生成代码，捕获指定 output_variable 的 DataFrame，失败时返回安全错误而非堆栈。
  - **`_cleanup(tmpdir)`**：Best-effort recursive removal of *tmpdir*.
  设计约束：公开方法必须返回安全数据（无密钥、无绝对路径）；异常应转换为 AppError 或 ToolResult 可恢复错误，而不是把 Python 堆栈直接交给前端。

模块级测试建议：至少覆盖（1）正常路径；（2）非法输入；（3）权限/路径拒绝；（4）上游失败时的 AppError 映射。相关用例见后文测试专章。


#### 模块 `py-src/data_formulator/sandbox/local_sandbox.py`（约 608 行）

**模块文档**：Local sandbox -- executes Python code in a persistent warm subprocess.  The script runs with the workspace directory as its working directory so user scripts access files via e.g. ``pd.read_csv("sample.csv")``.

符号统计：类 3 个，函数 1 个，模块级常量 0 个。下列说明覆盖全部符号。

##### 函数 `_warm_worker_loop(conn)`
- **说明**：Long-lived child process that pre-imports heavy libraries then waits for code to execute.  Protocol (over *conn*):     Host -> worker:  (code, allowed_objects, workspace_path)           — fresh namespace                  or  (code, allowed_objects, workspace_path, True)     — persistent namespace                  or  "__clear_ns__"                                    — reset persistent namespace                  or  None                                              — terminate     Worker -> host:
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 类 `_WarmWorkerPool`
- **基类**：无显式基类。
- **字段标注**：无模块级注解字段（可能使用 dataclass / 动态属性）。
- **方法数**：6。
- **文档字符串**：Pool of persistent child processes with pre-imported libraries.  Workers are forked once and reuse the same process for multiple calls, avoiding the ~600ms pandas/numpy import overhead each time. A simple LIFO stack ensures thread-safe checkout/return.
  - **`__init__(self, size)`**：内部实现细节见源码。
  - **`_spawn(self)`**：内部实现细节见源码。
  - **`acquire(self)`**：Get a warm worker (process, conn). Spawns one if needed.
  - **`release(self, proc, conn)`**：Return a worker to the pool for reuse.
  - **`discard(self, proc, conn)`**：Discard a broken worker (don't put it back).
  - **`shutdown(self)`**：内部实现细节见源码。
  设计约束：公开方法必须返回安全数据（无密钥、无绝对路径）；异常应转换为 AppError 或 ToolResult 可恢复错误，而不是把 Python 堆栈直接交给前端。

##### 类 `SandboxSession`
- **基类**：无显式基类。
- **字段标注**：无模块级注解字段（可能使用 dataclass / 动态属性）。
- **方法数**：7。
- **文档字符串**：A session that keeps the Python namespace alive across multiple executions.  Use as a context manager inside an agent turn so that consecutive ``explore()`` / ``execute_python()`` calls share variables (like a Jupyter kernel).  The namespace is cleared and the worker returned to the pool on ``close()`` / ``__exit__``.
  - **`__init__(self)`**：内部实现细节见源码。
  - **`execute(self, code, allowed_objects, workspace_path)`**：Run *code* with namespace persisted between calls.  Returns the same dict contract as ``_warm_worker_loop``: ``{"status": "ok", "allowed_objects": {...}}`` or ``{"status": "error", "error_message": "..."}``.
  - **`close(self)`**：Clear the persistent namespace and return the worker to the pool.
  - **`save_namespace(self, save_dir, workspace_path)`**：Serialize user DataFrames and scalars from the persistent namespace to *save_dir* so they can be restored in a future session.  DataFrames are saved as parquet files (by the **host** process, since the worker's audit hoo
  - **`restore_namespace(session, save_dir, workspace_path)`**：Restore previously saved DataFrames and scalars into *session*.  Returns True if restoration succeeded, False if nothing to restore. This is a static helper so it can be called right after creating a new session (the sav
  - **`__enter__(self)`**：内部实现细节见源码。
  - **`__exit__(self)`**：内部实现细节见源码。
  设计约束：公开方法必须返回安全数据（无密钥、无绝对路径）；异常应转换为 AppError 或 ToolResult 可恢复错误，而不是把 Python 堆栈直接交给前端。

##### 类 `LocalSandbox`
- **基类**：Sandbox。
- **字段标注**：无模块级注解字段（可能使用 dataclass / 动态属性）。
- **方法数**：2。
- **文档字符串**：Execute Python code in a persistent warm subprocess.  Uses a pool of pre-warmed child processes with pandas/numpy/duckdb already imported, giving ~1 ms execution overhead per call. Audit hooks in the child block file writes and dangerous operations.
  - **`run_python_code(self, code, workspace, output_variable)`**：Execute *code* and return the result DataFrame.  The script runs with the workspace directory as its working directory so scripts access data files via e.g. ``pd.read_csv("sample.csv")``.  Returns ------- dict     ``{'st 在隔离环境执行 Agent 生成代码，捕获指定 output_variable 的 DataFrame，失败时返回安全错误而非堆栈。
  - **`_run_in_warm_subprocess(code, allowed_objects, workspace_path)`**：Send code to a warm worker from the pool, return the result.
  设计约束：公开方法必须返回安全数据（无密钥、无绝对路径）；异常应转换为 AppError 或 ToolResult 可恢复错误，而不是把 Python 堆栈直接交给前端。

模块级测试建议：至少覆盖（1）正常路径；（2）非法输入；（3）权限/路径拒绝；（4）上游失败时的 AppError 映射。相关用例见后文测试专章。


#### 模块 `py-src/data_formulator/sandbox/not_a_sandbox.py`（约 79 行）

**模块文档**：Unsandboxed main-process executor -- for benchmarking only.  This runs user code directly in the main process with no isolation. It is NOT exposed as a CLI option and should only be used to measure the raw execution overhead baseline in benchmarks.

符号统计：类 1 个，函数 0 个，模块级常量 0 个。下列说明覆盖全部符号。

##### 类 `NotASandbox`
- **基类**：Sandbox。
- **字段标注**：无模块级注解字段（可能使用 dataclass / 动态属性）。
- **方法数**：1。
- **文档字符串**：Execute Python code directly in the main process (no isolation).  For benchmarking only -- measures raw exec() overhead without any subprocess or container overhead.  No security restrictions are applied.
  - **`run_python_code(self, code, workspace, output_variable)`**：内部实现细节见源码。 在隔离环境执行 Agent 生成代码，捕获指定 output_variable 的 DataFrame，失败时返回安全错误而非堆栈。
  设计约束：公开方法必须返回安全数据（无密钥、无绝对路径）；异常应转换为 AppError 或 ToolResult 可恢复错误，而不是把 Python 堆栈直接交给前端。

模块级测试建议：至少覆盖（1）正常路径；（2）非法输入；（3）权限/路径拒绝；（4）上游失败时的 AppError 映射。相关用例见后文测试专章。


### 目录 `py-src/data_formulator/security`

**目录职责**：路径监禁、日志脱敏、代码签名、URL 白名单

本目录内模块共享同一边界：只通过公开函数与相邻层通信。禁止跨层 import 私有 `_` 符号（测试除外）。新增文件必须补测试，并检查是否触发路径安全、日志脱敏、错误协议三条规则。


#### 模块 `py-src/data_formulator/security/__init__.py`（约 3 行）

**模块文档**：该文件未写模块级 docstring。其职责由文件名与导出符号定义，修改时应补充文档字符串，避免意图漂移。

本文件主要为导出聚合或入口转发，无额外类/函数定义。


#### 模块 `py-src/data_formulator/security/code_signing.py`（约 141 行）

**模块文档**：HMAC-based code signing for transformation code.  When the agent generates Python transformation code and the server executes it successfully, the server signs the code with a secret key. The signature is returned to the frontend alongside the code.  When the frontend later sends the code back for re-execution (e.g. during data refresh), the server verifies the signature before running the code.  This prevents a tampered or injected script from being executed by the sandbox.  Secret lifecycle ~~~~~~~~~~~~~~~~ - **Dev mode** (``--dev``): uses a fixed, deterministic key so that   signatures survive reloader restarts and hot-reloads during   development.  This is *not* secure for production. - **Production**: derives the key from Flask's ``app.secret_key``.   For multi-worker deploys (gunicor

符号统计：类 0 个，函数 5 个，模块级常量 2 个。下列说明覆盖全部符号。

**常量**：

- **常量 `_DEV_SECRET`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `MAX_CODE_SIZE`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

##### 函数 `_is_dev_mode()`
- **说明**：Return True if the server was started with ``--dev``.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_get_secret()`
- **说明**：Return the signing secret.  Priority: 1. ``DF_CODE_SIGNING_SECRET`` env-var  (set once, works everywhere) 2. Dev mode → fixed deterministic key (survives reloader restarts) 3. Production → derived from Flask ``app.secret_key`` 4. Fallback for tests / non-Flask callers
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `sign_code(code)`
- **说明**：Compute an HMAC-SHA256 signature over *code*.  Returns the hex-encoded signature string.  The signature covers the raw UTF-8 bytes of *code* — whitespace and encoding matter.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `verify_code(code, signature)`
- **说明**：Return ``True`` if *signature* is a valid HMAC for *code*.  Uses constant-time comparison to prevent timing attacks.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `sign_result(result)`
- **说明**：Add ``code_signature`` to an agent result dict (in-place).  If the result contains a non-empty ``code`` key, a signature is computed and stored under ``code_signature``.  The result dict is returned for convenience.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

模块级测试建议：至少覆盖（1）正常路径；（2）非法输入；（3）权限/路径拒绝；（4）上游失败时的 AppError 映射。相关用例见后文测试专章。


#### 模块 `py-src/data_formulator/security/log_sanitizer.py`（约 240 行）

**模块文档**：Log sanitization utilities for preventing sensitive data leakage.  Provides two layers of defense:  1. **Explicit utilities** — ``sanitize_url``, ``sanitize_params``,    ``redact_token`` — called at logging call-sites for precise control.  2. **SensitiveDataFilter** — a ``logging.Filter`` registered on handlers    as a safety net, automatically redacting patterns that slip through.  Usage::      from data_formulator.security.log_sanitizer import (         sanitize_url, sanitize_params, redact_token,     )      logger.info("Connected to %s", sanitize_url(url))     logger.info("Params: %s", sanitize_params(params))     logger.debug("Token: %s", redact_token(token))  The filter is registered in ``app.py:configure_logging()`` and can be disabled via ``LOG_SANITIZE=false`` for local debugging.

符号统计：类 1 个，函数 6 个，模块级常量 9 个。下列说明覆盖全部符号。

**常量**：

- **常量 `_REDACTED`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `_REDACTED_TOKEN`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `_RE_URL_CREDS`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `_RE_URL_LIKE`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `_SENSITIVE_KEY_NAMES`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `_RE_KEY_VALUE`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `_RE_BEARER`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `_RE_JWT_LIKE`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `_RE_DICT_SENSITIVE`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

##### 函数 `sanitize_url(url)`
- **说明**：Mask credentials embedded in a URL.  ``https://user:s3cret@host/path`` → ``https://user:***@host/path``  Sensitive query parameters such as ``password`` and ``access_token`` are also replaced with ``***``.  Safe to call on URLs without credentials or sensitive query parameters (returns unchanged).
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_is_sensitive_key(key)`
- **说明**：模块级函数 `_is_sensitive_key` 封装可复用算法或路由处理。参数经过校验后进入核心逻辑，返回结构化结果或通过异常表达业务失败。
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_sanitize_url_token(url)`
- **说明**：Sanitize one URL-like token while preserving non-sensitive URLs.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `sanitize_params(params, extra_keys)`
- **说明**：Return a shallow copy of *params* with sensitive values masked.  Keys are matched case-insensitively against ``SENSITIVE_KEYS`` (plus *extra_keys* when provided).
- **算法要点**：按 SENSITIVE_KEYS 把 password/token/api_key 等替换为 ***，作为日志第一道防线。
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `redact_token(token, visible)`
- **说明**：Abbreviate a token to first/last *visible* characters.  Short tokens (≤ 2×visible) are fully masked.  ``"eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.xxx"`` → ``"eyJh...eCJ9"``  (visible=4)
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_apply_patterns(text)`
- **说明**：Run all redaction patterns on *text* and return the sanitized version.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 类 `SensitiveDataFilter`
- **基类**：logging.Filter。
- **字段标注**：无模块级注解字段（可能使用 dataclass / 动态属性）。
- **方法数**：1。
- **文档字符串**：Safety-net filter that auto-redacts sensitive patterns in log messages.  Attach to handlers in ``configure_logging()``::      handler.addFilter(SensitiveDataFilter())  Disable at runtime via ``LOG_SANITIZE=false``.
  - **`filter(self, record)`**：内部实现细节见源码。
  设计约束：公开方法必须返回安全数据（无密钥、无绝对路径）；异常应转换为 AppError 或 ToolResult 可恢复错误，而不是把 Python 堆栈直接交给前端。

模块级测试建议：至少覆盖（1）正常路径；（2）非法输入；（3）权限/路径拒绝；（4）上游失败时的 AppError 映射。相关用例见后文测试专章。


#### 模块 `py-src/data_formulator/security/path_safety.py`（约 136 行）

**模块文档**：Path confinement primitive — prevents path traversal at the API level.  Usage::      jail = ConfinedDir("/tmp/workspace")     safe = jail / "data/sales.parquet"        # OK     jail / "../etc/passwd"                     # raises ValueError     jail.write("data/out.parquet", raw_bytes)  # resolve + mkdir + write

符号统计：类 1 个，函数 0 个，模块级常量 0 个。下列说明覆盖全部符号。

##### 类 `ConfinedDir`
- **基类**：无显式基类。
- **字段标注**：无模块级注解字段（可能使用 dataclass / 动态属性）。
- **方法数**：12。
- **文档字符串**：A directory jail that prevents any path operation from escaping its root.  All path resolution goes through this single chokepoint.  If the resolved path escapes the root, ``ValueError`` is raised immediately.  Thread-safe: instances are immutable after construction; Path.resolve() and is_relative_to() are OS-level and inherently safe for concurrent use.
  - **`__init__(self, root)`**：内部实现细节见源码。
  - **`root(self)`**：The resolved, canonical root directory.
  - **`resolve(self, relative)`**：Resolve *relative* within this jail.  Raises ``ValueError`` if the result would escape the root.  Defence is layered:   1. Reject absolute paths outright.   2. Reject path segments equal to ``..``.   3. Join onto root, c ConfinedDir 核心算法：canonicalize + relative_to(root)，拒绝绝对路径、.. 与符号链接逃逸。
  - **`write(self, relative, data)`**：Resolve, create parent dirs, and write *data* atomically.
  - **`read_text(self, relative, encoding)`**：Read a text file within this jail.
  - **`write_text(self, relative, content, encoding)`**：Write a text file within this jail (auto-creates parent dirs).
  - **`exists(self, relative)`**：Check whether a file/directory exists inside this jail.  Returns ``False`` for paths that would escape the jail instead of raising, so callers can treat traversal as "not found".
  - **`iterdir(self, relative)`**：List immediate children of *relative* (or the root).
  - **`rglob(self, pattern, relative)`**：Recursively glob *pattern* starting from *relative* (or the root).
  - **`unlink(self, relative)`**：Delete a file inside this jail.
  - **`__truediv__(self, relative)`**：Operator overload: ``jail / "sub/path"`` → ``jail.resolve("sub/path")``.
  - **`__repr__(self)`**：内部实现细节见源码。
  设计约束：公开方法必须返回安全数据（无密钥、无绝对路径）；异常应转换为 AppError 或 ToolResult 可恢复错误，而不是把 Python 堆栈直接交给前端。

模块级测试建议：至少覆盖（1）正常路径；（2）非法输入；（3）权限/路径拒绝；（4）上游失败时的 AppError 映射。相关用例见后文测试专章。


#### 模块 `py-src/data_formulator/security/sanitize.py`（约 210 行）

**模块文档**：Shared helpers for sanitizing error messages before they reach the client.

符号统计：类 0 个，函数 5 个，模块级常量 4 个。下列说明覆盖全部符号。

**常量**：

- **常量 `_GENERIC_5XX`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `_GENERIC_502`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `_GENERIC_4XX`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `_LLM_ERROR_GENERIC`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

##### 函数 `_extract_traceback_summary(message)`
- **说明**：Return the final exception line from a Python traceback when possible.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_structured_error_response(code, message, status_code)`
- **说明**：模块级函数 `_structured_error_response` 封装可复用算法或路由处理。参数经过校验后进入核心逻辑，返回结构化结果或通过异常表达业务失败。
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `safe_error_response(exc, status_code)`
- **说明**：Build a sanitized JSON error response for the client.  Strategy by *status_code*:  * **5xx** — never expose exception details; return a fixed generic message   and log the full exception server-side. * **502 (HTTPError from upstream)** — return "Upstream service unavailable". * **4xx** — if *client_message* is provided, use it directly (caller   asserts the message is safe); otherwise select a canned message based   on the upstream HTTP status, falling back to a generic "Bad request".   Exceptio
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `classify_llm_error(exc)`
- **说明**：Return a safe, user-friendly message for an LLM / external-API error.  The function matches ``str(exc)`` against known error patterns and returns a **pre-defined** human-readable message.  No text from the original exception is ever included in the return value.  Falls back to a generic ``"Model request failed"`` for unknown errors. The caller is responsible for logging the full exception server-side.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `sanitize_error_message(error_message)`
- **说明**：Sanitize error messages before sending to client.  Strips stack traces, file paths, API keys, and other potentially sensitive implementation details so that only a human-readable summary is returned to the browser.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

模块级测试建议：至少覆盖（1）正常路径；（2）非法输入；（3）权限/路径拒绝；（4）上游失败时的 AppError 映射。相关用例见后文测试专章。


#### 模块 `py-src/data_formulator/security/url_allowlist.py`（约 104 行）

**模块文档**：URL allowlist for user-provided LLM API base URLs.  When a user adds a custom model via the UI, they can supply an arbitrary ``api_base`` URL.  The server then makes outbound HTTP requests to that URL on behalf of the user.  Without validation this is a **Server-Side Request Forgery (SSRF)** vector — a malicious ``api_base`` could target internal services, cloud metadata endpoints, or private-network hosts.  This module provides a simple allowlist mechanism:  * **Open mode** (default): When ``DF_ALLOWED_API_BASES`` is not set,   *all* URLs are permitted.  This is convenient for local development. * **Enforce mode**: When ``DF_ALLOWED_API_BASES`` is set (comma-separated   glob patterns), only URLs that match at least one pattern are allowed.  Empty / missing ``api_base`` is always permitted

符号统计：类 0 个，函数 3 个，模块级常量 1 个。下列说明覆盖全部符号。

**常量**：

- **常量 `_ENV_KEY`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

##### 函数 `_load_patterns()`
- **说明**：Return the allowlist patterns, or ``None`` for open mode.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_is_allowlist_configured()`
- **说明**：Return ``True`` if an allowlist is active (enforce mode).
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `validate_api_base(api_base)`
- **说明**：Validate *api_base* against the configured allowlist.  - ``None`` or empty string → always allowed (provider default). - Open mode (env var unset) → everything allowed. - Enforce mode → must match at least one pattern.  Raises ``ValueError`` with a user-facing message on rejection.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

模块级测试建议：至少覆盖（1）正常路径；（2）非法输入；（3）权限/路径拒绝；（4）上游失败时的 AppError 映射。相关用例见后文测试专章。


### 目录 `py-src/data_formulator/workflows`

**目录职责**：语义类型到 Vega-Lite 图表的组装算法

本目录内模块共享同一边界：只通过公开函数与相邻层通信。禁止跨层 import 私有 `_` 符号（测试除外）。新增文件必须补测试，并检查是否触发路径安全、日志脱敏、错误协议三条规则。


#### 模块 `py-src/data_formulator/workflows/__init__.py`（约 1 行）

**模块文档**：该文件未写模块级 docstring。其职责由文件名与导出符号定义，修改时应补充文档字符串，避免意图漂移。

本文件主要为导出聚合或入口转发，无额外类/函数定义。


#### 模块 `py-src/data_formulator/workflows/chart_semantics.py`（约 600 行）

**模块文档**：============================================================================= CHART SEMANTICS — Lightweight type resolution for VL spec assembly =============================================================================  Provides semantic-aware type resolution for create_vl_plots.py:   - Type registry (maps semantic types → VL encoding types)   - VL type resolution (nominal / ordinal / temporal / quantitative)   - Ordinal sort order (months, days, quarters)  This is intentionally minimal.  The flint-chart TS library is the canonical source of truth for formatting, color schemes, tick constraints, domain constraints, and other visual refinements. The Python side focuses on getting the structural type decisions right (which directly affect chart shape), and leaves cosmetic details to defa

符号统计：类 2 个，函数 15 个，模块级常量 14 个。下列说明覆盖全部符号。

**常量**：

- **常量 `_UNKNOWN`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `_MAX_TIMESTAMP_SEC`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `_MAX_TIMESTAMP_MS`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `_DATE_PATTERNS`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `_MONTH_FULL`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `_MONTH_ABBR`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `_MONTH_NUM`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `_DOW_FULL`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `_DOW_ABBR`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `_DOW_FULL_SUN`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `_DOW_ABBR_SUN`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `_QUARTER`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `_COMPASS_8`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `_COMPASS_4`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

##### 类 `TypeRegistryEntry`
- **基类**：无显式基类。
- **字段标注**：t0, t1, vis_encodings, agg_role, domain_shape。
- **方法数**：0。
- **职责推断**：该类位于对应模块中，承担 `TypeRegistryEntry` 所表示的领域对象或服务。其方法覆盖构造、序列化、领域操作与错误处理；调用方应通过公开方法交互，避免依赖私有实现细节。
  设计约束：公开方法必须返回安全数据（无密钥、无绝对路径）；异常应转换为 AppError 或 ToolResult 可恢复错误，而不是把 Python 堆栈直接交给前端。

##### 函数 `get_registry_entry(semantic_type)`
- **说明**：Look up a type in the registry. Falls back to UNKNOWN.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `is_registered(semantic_type)`
- **说明**：模块级函数 `is_registered` 封装可复用算法或路由处理。参数经过校验后进入核心逻辑，返回结构化结果或通过异常表达业务失败。
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 类 `ChannelSemantics`
- **基类**：无显式基类。
- **字段标注**：field, semantic_type, vl_type, ordinal_sort_order。
- **方法数**：0。
- **文档字符串**：Resolved semantic decisions for a single channel.
  设计约束：公开方法必须返回安全数据（无密钥、无绝对路径）；异常应转换为 AppError 或 ToolResult 可恢复错误，而不是把 Python 堆栈直接交给前端。

##### 函数 `resolve_vl_type(semantic_type, values)`
- **说明**：Determine the best VL encoding type for a field. Uses semantic type first, then disambiguates with data.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_looks_like_year_integers(values)`
- **说明**：Check if a list of numeric values look like 4-digit year integers.  Returns True when >=80% of non-null values are integers in the plausible year range 1000-2999.  This is used to decide whether integer Year/Decade data should get a temporal axis (with string conversion) rather than being treated as plain quantitative numbers.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_infer_vl_type_from_data(values)`
- **说明**：Infer VL type purely from data values.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_is_likely_timestamp(val)`
- **说明**：Check if a numeric value is likely a unix timestamp (s or ms).
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_timestamp_to_ms(val)`
- **说明**：Convert seconds-epoch to ms-epoch if needed.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_looks_like_date(s)`
- **说明**：Check if a string looks like a date/datetime value.  This is a lightweight heuristic used for type inference. Does NOT match bare 4-digit years ("2020") — those should be handled via semantic type (Year/Decade) to avoid false positives with generic integer IDs.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_try_parse_date(val)`
- **说明**：Try to parse a value as a datetime.  Returns None on failure.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `infer_ordinal_sort_order(semantic_type, values)`
- **说明**：Detect canonical ordinal sort order for months, days, quarters, etc. Returns sorted unique values in canonical order, or None.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_match_sequence(values, sequences)`
- **说明**：模块级函数 `_match_sequence` 封装可复用算法或路由处理。参数经过校验后进入核心逻辑，返回结构化结果或通过异常表达业务失败。
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_expand_to_full_year(val)`
- **说明**：Expand 2-digit year to 4-digit: '98' → '1998', '07' → '2007'.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `convert_temporal_data(data, semantic_types, all_values)`
- **说明**：Convert temporal field values to canonical string representations so that Vega-Lite can parse them correctly.  Mirrors the TS ``convertTemporalData`` function.  This handles: - Year/Decade integers → string ("2015" not 2015, avoids VL unix-ms interpretation) - Unix timestamps → ISO datetime strings - datetime/date objects → ISO strings - pd.Timestamp objects → ISO strings - 2-digit year strings → 4-digit - Any other temporal values → str()  Parameters: - data: list of row dicts (will be cloned) 
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_extract_sem_type(annotation)`
- **说明**：Extract the semantic type string from an annotation (str or dict).
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `resolve_channel_semantics(field_name, semantic_type, channel, mark_type, values, unit, intrinsic_domain)`
- **说明**：Resolve semantic decisions for one (field, channel) pair. This is the main entry point used by create_vl_plots.py.  Focuses on the critical structural decision: VL type + ordinal sort. Formatting, domains, ticks, zero-baseline, color schemes, etc. are left to VL defaults (or the front-end TS library).
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

模块级测试建议：至少覆盖（1）正常路径；（2）非法输入；（3）权限/路径拒绝；（4）上游失败时的 AppError 映射。相关用例见后文测试专章。


#### 模块 `py-src/data_formulator/workflows/create_vl_plots.py`（约 2018 行）

**模块文档**：该文件未写模块级 docstring。其职责由文件名与导出符号定义，修改时应补充文档字符串，避免意图漂移。

符号统计：类 0 个，函数 28 个，模块级常量 2 个。下列说明覆盖全部符号。

**常量**：

- **常量 `CHART_TEMPLATES`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

- **常量 `_BAR_LIKE_CHARTS`**：模块级配置/协议常量，改变它会影响序列化兼容或运行时策略，修改前应检索全部引用并补充回归测试。

##### 函数 `field_metadata_to_semantic_types(field_metadata)`
- **说明**：Convert agent ``field_metadata`` to the ``semantic_types`` dict expected by :func:`assemble_vegailte_chart`.  ``field_metadata`` comes from the LLM's ``refined_goal.field_metadata`` and has the shape::      {         "Revenue": {"semantic_type": "Amount", "unit": "USD"},         "Month":   {"semantic_type": "Month"},     }  The returned dict maps field names to either a plain string (the semantic type name) or a dict with ``type``, ``unit``, and ``intrinsic_domain`` keys — exactly what ``assembl
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `resolve_field_type(series, field_name)`
- **说明**：Resolve the Vega-Lite type for a field.  Priority:   1. Column-name heuristic (catches derived columns like avg_revenue, year, etc.)   2. Pandas dtype detection (fallback)  Parameters: - series: the pandas Series for the field - field_name: column name (used for name heuristics)  Returns one of: 'quantitative', 'nominal', 'ordinal', 'temporal'
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `detect_field_type(series)`
- **说明**：Detect the appropriate Vega-Lite field type for a pandas Series. Returns one of: 'quantitative', 'nominal', 'ordinal', 'temporal'
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `coerce_field_type(chart_type, channel, detected_type)`
- **说明**：Return the Vega-Lite type that should actually be used for this (chart_type, channel) combination.  If no override is needed the originally detected type is returned unchanged.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `get_chart_template(chart_type)`
- **说明**：Find a chart template by its full name (e.g. "Scatter Plot").
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `create_chart_spec(df, fields, chart_type)`
- **说明**：Assign fields to appropriate visualization channels based on their data types and chart type.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `fields_to_encodings(df, chart_type, fields)`
- **说明**：Assign fields to appropriate visualization channels based on their data types and chart type.  Parameters: - df: pandas DataFrame containing the data - chart_type: string matching one of the chart types in CHART_TEMPLATES (e.g. "Scatter Plot") - fields: list of column names to assign to channels  Returns: - dict: mapping of channel names to encoding objects with "field" and "type" properties ("nominal", "quantitative", "temporal")
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `assemble_vegailte_chart(df, chart_type, encodings, max_nominal_values, config, semantic_types)`
- **说明**：Assemble a Vega-Lite chart specification from a dataframe, chart type, and encodings.  Parameters: - df: pandas DataFrame containing the data - chart_type: string matching one of the chart types in CHART_TEMPLATES - encodings: dict mapping channel names to encoding objects with "field" property   Examples:   - Simple: {"x": {"field": "field1"}, "y": {"field": "field2"}}   - With aggregation: {"x": {"field": "category"}, "y": {"field": "sales", "aggregate": "mean"}} - max_nominal_values: maximum 
- **算法要点**：字段→通道→Vega-Lite spec，再按图表类型做 post-process（lollipop、waterfall、radar 等）。
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_build_initial_spec(chart_type, template, df, encodings, config)`
- **说明**：Build the initial VL spec skeleton before encoding channels are applied.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_post_process_chart(spec, chart_type, df, encodings, config)`
- **说明**：Apply chart-type-specific post-processing after encodings are set.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_post_process_lollipop(spec, df, encodings, config)`
- **说明**：Lollipop: rule from 0 + circle at value. Both layers share positional encodings.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_post_process_regression(spec, encodings)`
- **说明**：Regression: scatter layer + regression trend line.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_post_process_ranged_dot(spec, encodings)`
- **说明**：Ranged dot plot: line + point layers; detail links line segments.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_post_process_candlestick(spec, df, encodings, config)`
- **说明**：Candlestick: rule (wick) + bar (body) with conditional coloring.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_post_process_waterfall(spec, df, encodings, config)`
- **说明**：Waterfall: cumulative bar chart with positive/negative coloring.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_post_process_density(spec, encodings, config)`
- **说明**：Density plot: kernel density transform.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_post_process_radar(spec, df, encodings, config)`
- **说明**：Radar chart: entirely client-computed polar projection.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_post_process_pyramid(spec, df, encodings)`
- **说明**：Pyramid: hconcat of two mirrored bar panels split by color field.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_post_process_streamgraph(spec, encodings, config)`
- **说明**：Streamgraph: stacked area with center baseline.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_post_process_bump(spec, encodings)`
- **说明**：Bump chart: reversed y-axis so rank 1 is at top.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_post_process_strip(spec, df, encodings, config)`
- **说明**：Strip plot: jittered points along a categorical axis.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_post_process_rose(spec, df, encodings, config)`
- **说明**：Rose (Nightingale) chart: polar bar using arc mark.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_apply_semantic_encoding(encoding_obj, cs, channel, chart_type, config)`
- **说明**：Apply semantic-aware enhancements to a VL encoding object.  Kept intentionally minimal — only ordinal sort order (months, days). Formatting, domains, ticks, zero-baseline, etc. are left to VL defaults or handled by the front-end TS library.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_apply_spec_quality(spec, table_data, df, chart_type)`
- **说明**：Post-processing quality improvements for VL specs.  Mirrors the TS ``vlApplyLayoutToSpec`` logic in instantiate-spec.ts. Returns (potentially filtered) table_data.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_apply_chart_config(spec, chart_type, config)`
- **说明**：Apply optional config overrides to a Vega-Lite spec.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `_get_top_values(df, field_name, unique_values, channel, spec, max_values)`
- **说明**：Get top values for nominal fields with many entries.
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `vl_spec_to_png(spec, output_path, scale)`
- **说明**：Convert a Vega-Lite specification to a PNG image.  Parameters: - spec: Vega-Lite specification dictionary - output_path: Optional path to save the PNG file - scale: Scale factor for higher resolution (default 1.0)  Returns: - bytes: PNG image data  Requires: pip install vl-convert-python
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

##### 函数 `spec_to_base64(spec, scale)`
- **说明**：Convert a Vega-Lite specification to a base64 encoded PNG string.  Parameters: - spec: Vega-Lite specification dictionary - width: Optional width in pixels (defaults to spec width or 400) - height: Optional height in pixels (defaults to spec height or 300) - scale: Scale factor for higher resolution (default 2.0 for 2x resolution)  Returns: - str: Base64 encoded PNG data (data:image/png;base64,...)  Requires: pip install vl-convert-python
- **调用约定**：禁止在日志中打印原始凭证；路径参数必须经 ConfinedDir；面向用户的字符串应走 i18n 或 message_code，而不是硬编码单一语言。

模块级测试建议：至少覆盖（1）正常路径；（2）非法输入；（3）权限/路径拒绝；（4）上游失败时的 AppError 映射。相关用例见后文测试专章。


## 5. 前端功能与函数设计

前端是分析师的操作面。设计原则：

- **单一 Redux 会话**：除 knowledge 面板与若干缓存外，会话状态集中在 dfSlice。
- **服务端权威表数据**：inputTables 只存快照元数据，行在 preview cache。
- **流式消费**：所有 Agent 端点走 streamRequest，禁止手写 EventSource SSE 前缀解析。
- **图表双路径**：交互编辑用 Vega/Flint；线程缩略图用 ChartRenderService 无头渲染。

### 5.1 路由与壳

`/` 与 `/app` 进入 DataFormulatorFC；`/auth/callback` 处理 OIDC PKCE；`/about` 产品说明。
AppShell 提供工作区选择、模型按钮、语言、上传对话框入口、鉴权按钮。

### 5.2 主界面三分栏

- 左/中：Data Thread（卡片网格：表、草稿、图、文本回合）。
- 中/右：Canvas（VisualizationView 或表 DataView 或报告编辑器）。
- 侧栏：DataSourceSidebar（连接器目录、会话、知识）。

Allotment 分栏宽度由 LayoutProvider 按容器测量，layout.ts 给出密度与图表拉伸上限。

### 5.3 Data Thread 状态机

1. 用户提交问题 → createDraftNode(status=running)。
2. 流事件更新 runningPlan、interaction。
3. visualize 结果 → 插入派生表与 Chart，promoteDraft。
4. ask_user → status=clarifying，AgentPausePanel。
5. 错误 → status=error，卡片展示可恢复信息。
6. 用户从历史卡片分支 → 新 DraftNode.parentNodeId 指向该节点。

### 5.4 数据加载对话状态机

queueDataLoadingTask → DataLoadingChat 开流 → LoadPlanCard/ConnectorFormCard 交互 →
confirmTableLoad → loadTable thunk → 可选择继续对话（canContinue）。

### 5.5 前端模块字典


### 前端目录 `src`

该目录下的 TypeScript/TSX 文件构成 UI 或客户端基础设施。组件必须使用 `t()`/`i18n.t()` 输出用户可见文本；thunk 与非组件工具不得调用 React hooks。


#### `src/icons.tsx`（约 286 行）

本文件导出 19 个符号：connectorSortOrder, getConnectorIcon, DatabaseViewIcon, default, default, default, default, default, default, GenericDBIcon, RelationalDBIcon, BooleanIcon, NumericalIcon, StringIcon, DateIcon, DateTimeIcon, TimeIcon, DurationIcon, UnknownIcon。
设计职责：作为该路径所表明的 UI 单元或工具模块，负责把 Redux 状态映射为可视控件，或把用户操作映射为 API/thunk。交互过程中的加载、空态、错误态必须可区分。若涉及 Agent 流，必须能被 AbortSignal 取消，并在 unmount 时停止写入已卸载组件。
状态约束：不得在组件本地复制一份“即将过期”的表全量行；需要行数据时通过 tableResolution/preview cache 物化。图表变更应走 dfActions 以便缩略图服务订阅。
测试对应：`tests/frontend/unit` 中同名或相近测试文件覆盖选择器、迁移、错误码、关键组件渲染契约。新增 props 时同步更新这些测试。


#### `src/index.css`（约 91 行）

本文件导出 0 个符号：（无 export 或仅为副作用入口）。
设计职责：作为该路径所表明的 UI 单元或工具模块，负责把 Redux 状态映射为可视控件，或把用户操作映射为 API/thunk。交互过程中的加载、空态、错误态必须可区分。若涉及 Agent 流，必须能被 AbortSignal 取消，并在 unmount 时停止写入已卸载组件。
状态约束：不得在组件本地复制一份“即将过期”的表全量行；需要行数据时通过 tableResolution/preview cache 物化。图表变更应走 dfActions 以便缩略图服务订阅。
测试对应：`tests/frontend/unit` 中同名或相近测试文件覆盖选择器、迁移、错误码、关键组件渲染契约。新增 props 时同步更新这些测试。


#### `src/index.tsx`（约 29 行）

本文件导出 0 个符号：（无 export 或仅为副作用入口）。
设计职责：作为该路径所表明的 UI 单元或工具模块，负责把 Redux 状态映射为可视控件，或把用户操作映射为 API/thunk。交互过程中的加载、空态、错误态必须可区分。若涉及 Agent 流，必须能被 AbortSignal 取消，并在 unmount 时停止写入已卸载组件。
状态约束：不得在组件本地复制一份“即将过期”的表全量行；需要行数据时通过 tableResolution/preview cache 物化。图表变更应走 dfActions 以便缩略图服务订阅。
测试对应：`tests/frontend/unit` 中同名或相近测试文件覆盖选择器、迁移、错误码、关键组件渲染契约。新增 props 时同步更新这些测试。


#### `src/mui.d.ts`（约 8 行）

本文件导出 0 个符号：（无 export 或仅为副作用入口）。
设计职责：作为该路径所表明的 UI 单元或工具模块，负责把 Redux 状态映射为可视控件，或把用户操作映射为 API/thunk。交互过程中的加载、空态、错误态必须可区分。若涉及 Agent 流，必须能被 AbortSignal 取消，并在 unmount 时停止写入已卸载组件。
状态约束：不得在组件本地复制一份“即将过期”的表全量行；需要行数据时通过 tableResolution/preview cache 物化。图表变更应走 dfActions 以便缩略图服务订阅。
测试对应：`tests/frontend/unit` 中同名或相近测试文件覆盖选择器、迁移、错误码、关键组件渲染契约。新增 props 时同步更新这些测试。


#### `src/types.d.ts`（约 30 行）

本文件导出 0 个符号：（无 export 或仅为副作用入口）。
设计职责：作为该路径所表明的 UI 单元或工具模块，负责把 Redux 状态映射为可视控件，或把用户操作映射为 API/thunk。交互过程中的加载、空态、错误态必须可区分。若涉及 Agent 流，必须能被 AbortSignal 取消，并在 unmount 时停止写入已卸载组件。
状态约束：不得在组件本地复制一份“即将过期”的表全量行；需要行数据时通过 tableResolution/preview cache 物化。图表变更应走 dfActions 以便缩略图服务订阅。
测试对应：`tests/frontend/unit` 中同名或相近测试文件覆盖选择器、迁移、错误码、关键组件渲染契约。新增 props 时同步更新这些测试。


### 前端目录 `src/api`

该目录下的 TypeScript/TSX 文件构成 UI 或客户端基础设施。组件必须使用 `t()`/`i18n.t()` 输出用户可见文本；thunk 与非组件工具不得调用 React hooks。


#### `src/api/knowledgeApi.ts`（约 192 行）

本文件导出 16 个符号：KnowledgeCategory, KnowledgeItem, KnowledgeLimits, KnowledgeSearchResult, readDataMemory, appendDataMemory, rewriteDataMemory, fetchKnowledgeLimits, listKnowledge, readKnowledge, writeKnowledge, deleteKnowledge, searchKnowledge, DistillWorkflowResult, SessionWorkflowContext, distillSessionWorkflow。
设计职责：作为该路径所表明的 UI 单元或工具模块，负责把 Redux 状态映射为可视控件，或把用户操作映射为 API/thunk。交互过程中的加载、空态、错误态必须可区分。若涉及 Agent 流，必须能被 AbortSignal 取消，并在 unmount 时停止写入已卸载组件。
状态约束：不得在组件本地复制一份“即将过期”的表全量行；需要行数据时通过 tableResolution/preview cache 物化。图表变更应走 dfActions 以便缩略图服务订阅。
测试对应：`tests/frontend/unit` 中同名或相近测试文件覆盖选择器、迁移、错误码、关键组件渲染契约。新增 props 时同步更新这些测试。


### 前端目录 `src/app`

该目录下的 TypeScript/TSX 文件构成 UI 或客户端基础设施。组件必须使用 `t()`/`i18n.t()` 输出用户可见文本；thunk 与非组件工具不得调用 React hooks。


#### `src/app/App.tsx`（约 1779 行）

本文件导出 3 个符号：toolName, AppFCProps, AppFC。
设计职责：作为该路径所表明的 UI 单元或工具模块，负责把 Redux 状态映射为可视控件，或把用户操作映射为 API/thunk。交互过程中的加载、空态、错误态必须可区分。若涉及 Agent 流，必须能被 AbortSignal 取消，并在 unmount 时停止写入已卸载组件。
状态约束：不得在组件本地复制一份“即将过期”的表全量行；需要行数据时通过 tableResolution/preview cache 物化。图表变更应走 dfActions 以便缩略图服务订阅。
测试对应：`tests/frontend/unit` 中同名或相近测试文件覆盖选择器、迁移、错误码、关键组件渲染契约。新增 props 时同步更新这些测试。


#### `src/app/AuthButton.tsx`（约 173 行）

本文件导出 1 个符号：AuthButton。
设计职责：作为该路径所表明的 UI 单元或工具模块，负责把 Redux 状态映射为可视控件，或把用户操作映射为 API/thunk。交互过程中的加载、空态、错误态必须可区分。若涉及 Agent 流，必须能被 AbortSignal 取消，并在 unmount 时停止写入已卸载组件。
状态约束：不得在组件本地复制一份“即将过期”的表全量行；需要行数据时通过 tableResolution/preview cache 物化。图表变更应走 dfActions 以便缩略图服务订阅。
测试对应：`tests/frontend/unit` 中同名或相近测试文件覆盖选择器、迁移、错误码、关键组件渲染契约。新增 props 时同步更新这些测试。


#### `src/app/IdentityMigrationDialog.tsx`（约 156 行）

本文件导出 2 个符号：MigrationDialogProps, IdentityMigrationDialog。
设计职责：作为该路径所表明的 UI 单元或工具模块，负责把 Redux 状态映射为可视控件，或把用户操作映射为 API/thunk。交互过程中的加载、空态、错误态必须可区分。若涉及 Agent 流，必须能被 AbortSignal 取消，并在 unmount 时停止写入已卸载组件。
状态约束：不得在组件本地复制一份“即将过期”的表全量行；需要行数据时通过 tableResolution/preview cache 物化。图表变更应走 dfActions 以便缩略图服务订阅。
测试对应：`tests/frontend/unit` 中同名或相近测试文件覆盖选择器、迁移、错误码、关键组件渲染契约。新增 props 时同步更新这些测试。


#### `src/app/LayoutProvider.tsx`（约 234 行）

本文件导出 6 个符号：DensityPreference, LayoutContextValue, LayoutProvider, useLayout, useContainerSize, useSettledValue。
设计职责：作为该路径所表明的 UI 单元或工具模块，负责把 Redux 状态映射为可视控件，或把用户操作映射为 API/thunk。交互过程中的加载、空态、错误态必须可区分。若涉及 Agent 流，必须能被 AbortSignal 取消，并在 unmount 时停止写入已卸载组件。
状态约束：不得在组件本地复制一份“即将过期”的表全量行；需要行数据时通过 tableResolution/preview cache 物化。图表变更应走 dfActions 以便缩略图服务订阅。
测试对应：`tests/frontend/unit` 中同名或相近测试文件覆盖选择器、迁移、错误码、关键组件渲染契约。新增 props 时同步更新这些测试。


#### `src/app/OidcCallback.tsx`（约 119 行）

本文件导出 1 个符号：OidcCallback。
设计职责：作为该路径所表明的 UI 单元或工具模块，负责把 Redux 状态映射为可视控件，或把用户操作映射为 API/thunk。交互过程中的加载、空态、错误态必须可区分。若涉及 Agent 流，必须能被 AbortSignal 取消，并在 unmount 时停止写入已卸载组件。
状态约束：不得在组件本地复制一份“即将过期”的表全量行；需要行数据时通过 tableResolution/preview cache 物化。图表变更应走 dfActions 以便缩略图服务订阅。
测试对应：`tests/frontend/unit` 中同名或相近测试文件覆盖选择器、迁移、错误码、关键组件渲染契约。新增 props 时同步更新这些测试。


#### `src/app/agentInteractionPolicy.ts`（约 7 行）

本文件导出 1 个符号：shouldAutoFocusGeneratedChart。
设计职责：作为该路径所表明的 UI 单元或工具模块，负责把 Redux 状态映射为可视控件，或把用户操作映射为 API/thunk。交互过程中的加载、空态、错误态必须可区分。若涉及 Agent 流，必须能被 AbortSignal 取消，并在 unmount 时停止写入已卸载组件。
状态约束：不得在组件本地复制一份“即将过期”的表全量行；需要行数据时通过 tableResolution/preview cache 物化。图表变更应走 dfActions 以便缩略图服务订阅。
测试对应：`tests/frontend/unit` 中同名或相近测试文件覆盖选择器、迁移、错误码、关键组件渲染契约。新增 props 时同步更新这些测试。


#### `src/app/apiClient.ts`（约 317 行）

本文件导出 7 个符号：ApiError, StreamEvent, ApiRequestError, parseApiResponse, parseStreamLine, apiRequest, assertDownloadResponseOk。
设计职责：作为该路径所表明的 UI 单元或工具模块，负责把 Redux 状态映射为可视控件，或把用户操作映射为 API/thunk。交互过程中的加载、空态、错误态必须可区分。若涉及 Agent 流，必须能被 AbortSignal 取消，并在 unmount 时停止写入已卸载组件。
状态约束：不得在组件本地复制一份“即将过期”的表全量行；需要行数据时通过 tableResolution/preview cache 物化。图表变更应走 dfActions 以便缩略图服务订阅。
测试对应：`tests/frontend/unit` 中同名或相近测试文件覆盖选择器、迁移、错误码、关键组件渲染契约。新增 props 时同步更新这些测试。


#### `src/app/chartCache.ts`（约 136 行）

本文件导出 8 个符号：ChartCacheEntry, getCachedChart, setCachedChart, invalidateChart, clearCache, getChartPngDataUrl, downscaleImageForAgent, computeCacheKey。
设计职责：作为该路径所表明的 UI 单元或工具模块，负责把 Redux 状态映射为可视控件，或把用户操作映射为 API/thunk。交互过程中的加载、空态、错误态必须可区分。若涉及 Agent 流，必须能被 AbortSignal 取消，并在 unmount 时停止写入已卸载组件。
状态约束：不得在组件本地复制一份“即将过期”的表全量行；需要行数据时通过 tableResolution/preview cache 物化。图表变更应走 dfActions 以便缩略图服务订阅。
测试对应：`tests/frontend/unit` 中同名或相近测试文件覆盖选择器、迁移、错误码、关键组件渲染契约。新增 props 时同步更新这些测试。


#### `src/app/chartRecommendation.ts`（约 113 行）

本文件导出 2 个符号：resolveRecommendedChart, resolveChartFields。
设计职责：作为该路径所表明的 UI 单元或工具模块，负责把 Redux 状态映射为可视控件，或把用户操作映射为 API/thunk。交互过程中的加载、空态、错误态必须可区分。若涉及 Agent 流，必须能被 AbortSignal 取消，并在 unmount 时停止写入已卸载组件。
状态约束：不得在组件本地复制一份“即将过期”的表全量行；需要行数据时通过 tableResolution/preview cache 物化。图表变更应走 dfActions 以便缩略图服务订阅。
测试对应：`tests/frontend/unit` 中同名或相近测试文件覆盖选择器、迁移、错误码、关键组件渲染契约。新增 props 时同步更新这些测试。


#### `src/app/clarification.ts`（约 119 行）

本文件导出 3 个符号：NormalizedClarification, normalizeClarifyEvent, formatClarificationResponses。
设计职责：作为该路径所表明的 UI 单元或工具模块，负责把 Redux 状态映射为可视控件，或把用户操作映射为 API/thunk。交互过程中的加载、空态、错误态必须可区分。若涉及 Agent 流，必须能被 AbortSignal 取消，并在 unmount 时停止写入已卸载组件。
状态约束：不得在组件本地复制一份“即将过期”的表全量行；需要行数据时通过 tableResolution/preview cache 物化。图表变更应走 dfActions 以便缩略图服务订阅。
测试对应：`tests/frontend/unit` 中同名或相近测试文件覆盖选择器、迁移、错误码、关键组件渲染契约。新增 props 时同步更新这些测试。


#### `src/app/connectorFormPersistence.ts`（约 16 行）

本文件导出 1 个符号：stripConnectorPrefillFromEntries。
设计职责：作为该路径所表明的 UI 单元或工具模块，负责把 Redux 状态映射为可视控件，或把用户操作映射为 API/thunk。交互过程中的加载、空态、错误态必须可区分。若涉及 Agent 流，必须能被 AbortSignal 取消，并在 unmount 时停止写入已卸载组件。
状态约束：不得在组件本地复制一份“即将过期”的表全量行；需要行数据时通过 tableResolution/preview cache 物化。图表变更应走 dfActions 以便缩略图服务订阅。
测试对应：`tests/frontend/unit` 中同名或相近测试文件覆盖选择器、迁移、错误码、关键组件渲染契约。新增 props 时同步更新这些测试。


#### `src/app/connectorNames.ts`（约 39 行）

本文件导出 1 个符号：deriveConnectorDisplayName。
设计职责：作为该路径所表明的 UI 单元或工具模块，负责把 Redux 状态映射为可视控件，或把用户操作映射为 API/thunk。交互过程中的加载、空态、错误态必须可区分。若涉及 Agent 流，必须能被 AbortSignal 取消，并在 unmount 时停止写入已卸载组件。
状态约束：不得在组件本地复制一份“即将过期”的表全量行；需要行数据时通过 tableResolution/preview cache 物化。图表变更应走 dfActions 以便缩略图服务订阅。
测试对应：`tests/frontend/unit` 中同名或相近测试文件覆盖选择器、迁移、错误码、关键组件渲染契约。新增 props 时同步更新这些测试。


#### `src/app/dfSlice.tsx`（约 2761 行）

本文件导出 22 个符号：generateFreshChart, SSEMessage, ServerConfig, ModelConfig, FocusedId, DEFAULT_ROW_LIMIT, ClientConfig, GeneratedReport, DataFormulatorState, fetchFieldSemanticType, generateStarterQuestions, fetchColumnStats, fetchCodeExpl, fetchGlobalModelList, fetchAvailableModels, dataFormulatorSlice, selectTableIds, selectRefreshConfigs, dfSelectors, getDataFieldItems, dfActions, dataFormulatorReducer。
设计职责：作为该路径所表明的 UI 单元或工具模块，负责把 Redux 状态映射为可视控件，或把用户操作映射为 API/thunk。交互过程中的加载、空态、错误态必须可区分。若涉及 Agent 流，必须能被 AbortSignal 取消，并在 unmount 时停止写入已卸载组件。
状态约束：不得在组件本地复制一份“即将过期”的表全量行；需要行数据时通过 tableResolution/preview cache 物化。图表变更应走 dfActions 以便缩略图服务订阅。
测试对应：`tests/frontend/unit` 中同名或相近测试文件覆盖选择器、迁移、错误码、关键组件渲染契约。新增 props 时同步更新这些测试。


#### `src/app/displayRowsCache.ts`（约 44 行）

本文件导出 3 个符号：DisplayRowsEntry, displayRowsCache, computeDisplayRowsCacheKey。
设计职责：作为该路径所表明的 UI 单元或工具模块，负责把 Redux 状态映射为可视控件，或把用户操作映射为 API/thunk。交互过程中的加载、空态、错误态必须可区分。若涉及 Agent 流，必须能被 AbortSignal 取消，并在 unmount 时停止写入已卸载组件。
状态约束：不得在组件本地复制一份“即将过期”的表全量行；需要行数据时通过 tableResolution/preview cache 物化。图表变更应走 dfActions 以便缩略图服务订阅。
测试对应：`tests/frontend/unit` 中同名或相近测试文件覆盖选择器、迁移、错误码、关键组件渲染契约。新增 props 时同步更新这些测试。


#### `src/app/errorCodes.ts`（约 72 行）

本文件导出 2 个符号：ERROR_CODE_I18N_MAP, getErrorMessage。
设计职责：作为该路径所表明的 UI 单元或工具模块，负责把 Redux 状态映射为可视控件，或把用户操作映射为 API/thunk。交互过程中的加载、空态、错误态必须可区分。若涉及 Agent 流，必须能被 AbortSignal 取消，并在 unmount 时停止写入已卸载组件。
状态约束：不得在组件本地复制一份“即将过期”的表全量行；需要行数据时通过 tableResolution/preview cache 物化。图表变更应走 dfActions 以便缩略图服务订阅。
测试对应：`tests/frontend/unit` 中同名或相近测试文件覆盖选择器、迁移、错误码、关键组件渲染契约。新增 props 时同步更新这些测试。


#### `src/app/errorHandler.ts`（约 123 行）

本文件导出 3 个符号：extractErrorMessage, HandleApiErrorOptions, handleApiError。
设计职责：作为该路径所表明的 UI 单元或工具模块，负责把 Redux 状态映射为可视控件，或把用户操作映射为 API/thunk。交互过程中的加载、空态、错误态必须可区分。若涉及 Agent 流，必须能被 AbortSignal 取消，并在 unmount 时停止写入已卸载组件。
状态约束：不得在组件本地复制一份“即将过期”的表全量行；需要行数据时通过 tableResolution/preview cache 物化。图表变更应走 dfActions 以便缩略图服务订阅。
测试对应：`tests/frontend/unit` 中同名或相近测试文件覆盖选择器、迁移、错误码、关键组件渲染契约。新增 props 时同步更新这些测试。


#### `src/app/identity.ts`（约 121 行）

本文件导出 8 个符号：IdentityType, Identity, UserInfo, generateUUID, getBrowserId, clearBrowserId, resolveIdentity, getIdentityKey。
设计职责：作为该路径所表明的 UI 单元或工具模块，负责把 Redux 状态映射为可视控件，或把用户操作映射为 API/thunk。交互过程中的加载、空态、错误态必须可区分。若涉及 Agent 流，必须能被 AbortSignal 取消，并在 unmount 时停止写入已卸载组件。
状态约束：不得在组件本地复制一份“即将过期”的表全量行；需要行数据时通过 tableResolution/preview cache 物化。图表变更应走 dfActions 以便缩略图服务订阅。
测试对应：`tests/frontend/unit` 中同名或相近测试文件覆盖选择器、迁移、错误码、关键组件渲染契约。新增 props 时同步更新这些测试。


#### `src/app/inputTablePreviewCache.ts`（约 49 行）

本文件导出 6 个符号：INPUT_TABLE_PREVIEW_ROW_LIMIT, getInputTablePreview, setInputTablePreview, invalidateInputTablePreview, clearInputTablePreviewCache, replaceInputTablePreviews。
设计职责：作为该路径所表明的 UI 单元或工具模块，负责把 Redux 状态映射为可视控件，或把用户操作映射为 API/thunk。交互过程中的加载、空态、错误态必须可区分。若涉及 Agent 流，必须能被 AbortSignal 取消，并在 unmount 时停止写入已卸载组件。
状态约束：不得在组件本地复制一份“即将过期”的表全量行；需要行数据时通过 tableResolution/preview cache 物化。图表变更应走 dfActions 以便缩略图服务订阅。
测试对应：`tests/frontend/unit` 中同名或相近测试文件覆盖选择器、迁移、错误码、关键组件渲染契约。新增 props 时同步更新这些测试。


#### `src/app/intentClassifier.ts`（约 60 行）

本文件导出 2 个符号：ChartPromptIntent, classifyChartIntent。
设计职责：作为该路径所表明的 UI 单元或工具模块，负责把 Redux 状态映射为可视控件，或把用户操作映射为 API/thunk。交互过程中的加载、空态、错误态必须可区分。若涉及 Agent 流，必须能被 AbortSignal 取消，并在 unmount 时停止写入已卸载组件。
状态约束：不得在组件本地复制一份“即将过期”的表全量行；需要行数据时通过 tableResolution/preview cache 物化。图表变更应走 dfActions 以便缩略图服务订阅。
测试对应：`tests/frontend/unit` 中同名或相近测试文件覆盖选择器、迁移、错误码、关键组件渲染契约。新增 props 时同步更新这些测试。


#### `src/app/layout.ts`（约 542 行）

本文件导出 42 个符号：Density, WidthClass, HeightClass, MIN_SUPPORTED, WIDTH_BREAKPOINTS, HEIGHT_BREAKPOINTS, DENSITY_SCALE, LayoutTokens, REFERENCE, resolveWidthClass, resolveHeightClass, densityForWidthClass, threadColumnsForWidthClass, maxThreadColumnsForWidthClass, defaultThreadColumns, layoutFor, stripPaddingLeft, SCROLLBAR_ALLOWANCE, COLUMN_FIT_TOLERANCE, threadPaneWidthFor, threadStripWidthFor, fittableThreadColumnsFor, textVar, iconVar, buttonVar, gridSizeCaps, CHART_STRETCH_STEPS, chartStretchCeiling, CHART_SIZE_STOPS, DEFAULT_CHART_SIZE_STOP_INDEX, chartSizeStopIndex, defaultChartSizeStop, DIALOG_VIEWPORT_MARGIN, dialogHeight, dialogWidth, ShellBudget, minimumShellBudget, sidebarFitsExpanded, maxThreadColumnsForWidth, COMFORTABLE_CANVAS。
设计职责：作为该路径所表明的 UI 单元或工具模块，负责把 Redux 状态映射为可视控件，或把用户操作映射为 API/thunk。交互过程中的加载、空态、错误态必须可区分。若涉及 Agent 流，必须能被 AbortSignal 取消，并在 unmount 时停止写入已卸载组件。
状态约束：不得在组件本地复制一份“即将过期”的表全量行；需要行数据时通过 tableResolution/preview cache 物化。图表变更应走 dfActions 以便缩略图服务订阅。
测试对应：`tests/frontend/unit` 中同名或相近测试文件覆盖选择器、迁移、错误码、关键组件渲染契约。新增 props 时同步更新这些测试。


#### `src/app/loadableState.ts`（约 48 行）

本文件导出 7 个符号：LoadableStatus, LoadableState, idleLoadable, loadingLoadable, successLoadable, errorLoadable, getLoadableErrorMessage。
设计职责：作为该路径所表明的 UI 单元或工具模块，负责把 Redux 状态映射为可视控件，或把用户操作映射为 API/thunk。交互过程中的加载、空态、错误态必须可区分。若涉及 Agent 流，必须能被 AbortSignal 取消，并在 unmount 时停止写入已卸载组件。
状态约束：不得在组件本地复制一份“即将过期”的表全量行；需要行数据时通过 tableResolution/preview cache 物化。图表变更应走 dfActions 以便缩略图服务订阅。
测试对应：`tests/frontend/unit` 中同名或相近测试文件覆盖选择器、迁移、错误码、关键组件渲染契约。新增 props 时同步更新这些测试。


#### `src/app/oidcConfig.ts`（约 227 行）

本文件导出 11 个符号：OidcEndpointMetadata, OidcConfig, AuthInfo, isBackendAuth, getAuthInfo, getOidcConfig, getUserManager, getAccessToken, getOidcUser, _resetForTesting, _setUserManagerForTesting。
设计职责：作为该路径所表明的 UI 单元或工具模块，负责把 Redux 状态映射为可视控件，或把用户操作映射为 API/thunk。交互过程中的加载、空态、错误态必须可区分。若涉及 Agent 流，必须能被 AbortSignal 取消，并在 unmount 时停止写入已卸载组件。
状态约束：不得在组件本地复制一份“即将过期”的表全量行；需要行数据时通过 tableResolution/preview cache 物化。图表变更应走 dfActions 以便缩略图服务订阅。
测试对应：`tests/frontend/unit` 中同名或相近测试文件覆盖选择器、迁移、错误码、关键组件渲染契约。新增 props 时同步更新这些测试。


#### `src/app/restyle.ts`（约 327 行）

本文件导出 8 个符号：buildEmbeddedDataForChart, buildSpecForRestyle, buildDataContext, RestyleResult, callRestyleAgent, makeVariant, sanitizeConfigUI, applyVariantConfigUI。
设计职责：作为该路径所表明的 UI 单元或工具模块，负责把 Redux 状态映射为可视控件，或把用户操作映射为 API/thunk。交互过程中的加载、空态、错误态必须可区分。若涉及 Agent 流，必须能被 AbortSignal 取消，并在 unmount 时停止写入已卸载组件。
状态约束：不得在组件本地复制一份“即将过期”的表全量行；需要行数据时通过 tableResolution/preview cache 物化。图表变更应走 dfActions 以便缩略图服务订阅。
测试对应：`tests/frontend/unit` 中同名或相近测试文件覆盖选择器、迁移、错误码、关键组件渲染契约。新增 props 时同步更新这些测试。


#### `src/app/stateMigrations.ts`（约 357 行）

本文件导出 2 个符号：DF_STATE_VERSION, migrateState。
设计职责：作为该路径所表明的 UI 单元或工具模块，负责把 Redux 状态映射为可视控件，或把用户操作映射为 API/thunk。交互过程中的加载、空态、错误态必须可区分。若涉及 Agent 流，必须能被 AbortSignal 取消，并在 unmount 时停止写入已卸载组件。
状态约束：不得在组件本地复制一份“即将过期”的表全量行；需要行数据时通过 tableResolution/preview cache 物化。图表变更应走 dfActions 以便缩略图服务订阅。
测试对应：`tests/frontend/unit` 中同名或相近测试文件覆盖选择器、迁移、错误码、关键组件渲染契约。新增 props 时同步更新这些测试。


#### `src/app/store.ts`（约 54 行）

本文件导出 3 个符号：AppDispatch, persistor, store。
设计职责：作为该路径所表明的 UI 单元或工具模块，负责把 Redux 状态映射为可视控件，或把用户操作映射为 API/thunk。交互过程中的加载、空态、错误态必须可区分。若涉及 Agent 流，必须能被 AbortSignal 取消，并在 unmount 时停止写入已卸载组件。
状态约束：不得在组件本地复制一份“即将过期”的表全量行；需要行数据时通过 tableResolution/preview cache 物化。图表变更应走 dfActions 以便缩略图服务订阅。
测试对应：`tests/frontend/unit` 中同名或相近测试文件覆盖选择器、迁移、错误码、关键组件渲染契约。新增 props 时同步更新这些测试。


#### `src/app/tableResolution.ts`（约 61 行）

本文件导出 5 个符号：workspaceTableIdOf, AnalystTableRef, toAnalystTableRef, materializeInputTablePreview, materializeTables。
设计职责：作为该路径所表明的 UI 单元或工具模块，负责把 Redux 状态映射为可视控件，或把用户操作映射为 API/thunk。交互过程中的加载、空态、错误态必须可区分。若涉及 Agent 流，必须能被 AbortSignal 取消，并在 unmount 时停止写入已卸载组件。
状态约束：不得在组件本地复制一份“即将过期”的表全量行；需要行数据时通过 tableResolution/preview cache 物化。图表变更应走 dfActions 以便缩略图服务订阅。
测试对应：`tests/frontend/unit` 中同名或相近测试文件覆盖选择器、迁移、错误码、关键组件渲染契约。新增 props 时同步更新这些测试。


#### `src/app/tableThunks.ts`（约 390 行）

本文件导出 6 个符号：LoadTablePayload, LoadTableResult, resolveDatabaseImportLimit, loadTable, buildDictTableFromWorkspace, hasLocalOnlyAncestor。
设计职责：作为该路径所表明的 UI 单元或工具模块，负责把 Redux 状态映射为可视控件，或把用户操作映射为 API/thunk。交互过程中的加载、空态、错误态必须可区分。若涉及 Agent 流，必须能被 AbortSignal 取消，并在 unmount 时停止写入已卸载组件。
状态约束：不得在组件本地复制一份“即将过期”的表全量行；需要行数据时通过 tableResolution/preview cache 物化。图表变更应走 dfActions 以便缩略图服务订阅。
测试对应：`tests/frontend/unit` 中同名或相近测试文件覆盖选择器、迁移、错误码、关键组件渲染契约。新增 props 时同步更新这些测试。


#### `src/app/tokens.ts`（约 269 行）

本文件导出 16 个符号：borderColor, sidebarEdge, DividerBorderStyle, ComponentBorderStyle, ViewBorderStyle, shadow, transition, floatingPillSx, conversationWidth, radius, AppPaletteEntry, AppPalette, palettes, defaultPaletteKey, paletteKeys, bgAlpha。
设计职责：作为该路径所表明的 UI 单元或工具模块，负责把 Redux 状态映射为可视控件，或把用户操作映射为 API/thunk。交互过程中的加载、空态、错误态必须可区分。若涉及 Agent 流，必须能被 AbortSignal 取消，并在 unmount 时停止写入已卸载组件。
状态约束：不得在组件本地复制一份“即将过期”的表全量行；需要行数据时通过 tableResolution/preview cache 物化。图表变更应走 dfActions 以便缩略图服务订阅。
测试对应：`tests/frontend/unit` 中同名或相近测试文件覆盖选择器、迁移、错误码、关键组件渲染契约。新增 props 时同步更新这些测试。


#### `src/app/useAutoSave.tsx`（约 121 行）

本文件导出 2 个符号：getSerializableState, useAutoSave。
设计职责：作为该路径所表明的 UI 单元或工具模块，负责把 Redux 状态映射为可视控件，或把用户操作映射为 API/thunk。交互过程中的加载、空态、错误态必须可区分。若涉及 Agent 流，必须能被 AbortSignal 取消，并在 unmount 时停止写入已卸载组件。
状态约束：不得在组件本地复制一份“即将过期”的表全量行；需要行数据时通过 tableResolution/preview cache 物化。图表变更应走 dfActions 以便缩略图服务订阅。
测试对应：`tests/frontend/unit` 中同名或相近测试文件覆盖选择器、迁移、错误码、关键组件渲染契约。新增 props 时同步更新这些测试。


#### `src/app/useDataRefresh.tsx`（约 653 行）

本文件导出 2 个符号：useDataRefresh, useDerivedTableRefresh。
设计职责：作为该路径所表明的 UI 单元或工具模块，负责把 Redux 状态映射为可视控件，或把用户操作映射为 API/thunk。交互过程中的加载、空态、错误态必须可区分。若涉及 Agent 流，必须能被 AbortSignal 取消，并在 unmount 时停止写入已卸载组件。
状态约束：不得在组件本地复制一份“即将过期”的表全量行；需要行数据时通过 tableResolution/preview cache 物化。图表变更应走 dfActions 以便缩略图服务订阅。
测试对应：`tests/frontend/unit` 中同名或相近测试文件覆盖选择器、迁移、错误码、关键组件渲染契约。新增 props 时同步更新这些测试。


#### `src/app/useKnowledgeStore.ts`（约 201 行）

本文件导出 2 个符号：KnowledgeCategoryState, useKnowledgeStore。
设计职责：作为该路径所表明的 UI 单元或工具模块，负责把 Redux 状态映射为可视控件，或把用户操作映射为 API/thunk。交互过程中的加载、空态、错误态必须可区分。若涉及 Agent 流，必须能被 AbortSignal 取消，并在 unmount 时停止写入已卸载组件。
状态约束：不得在组件本地复制一份“即将过期”的表全量行；需要行数据时通过 tableResolution/preview cache 物化。图表变更应走 dfActions 以便缩略图服务订阅。
测试对应：`tests/frontend/unit` 中同名或相近测试文件覆盖选择器、迁移、错误码、关键组件渲染契约。新增 props 时同步更新这些测试。


#### `src/app/useWorkspaceAutoName.tsx`（约 98 行）

本文件导出 2 个符号：isUntitledWorkspaceName, useWorkspaceAutoName。
设计职责：作为该路径所表明的 UI 单元或工具模块，负责把 Redux 状态映射为可视控件，或把用户操作映射为 API/thunk。交互过程中的加载、空态、错误态必须可区分。若涉及 Agent 流，必须能被 AbortSignal 取消，并在 unmount 时停止写入已卸载组件。
状态约束：不得在组件本地复制一份“即将过期”的表全量行；需要行数据时通过 tableResolution/preview cache 物化。图表变更应走 dfActions 以便缩略图服务订阅。
测试对应：`tests/frontend/unit` 中同名或相近测试文件覆盖选择器、迁移、错误码、关键组件渲染契约。新增 props 时同步更新这些测试。


#### `src/app/utils.tsx`（约 678 行）

本文件导出 17 个符号：getUrls, SourceTableRef, CONNECTOR_ACTION_URLS, CONNECTOR_URLS, fetchWithIdentity, getAgentLanguage, translateBackend, translateBackendOptions, usePrevious, computeContentHash, runCodeOnInputListsInVM, extractFieldsFromEncodingMap, prepVisTable, assembleVegaChart, hashCode, resolveRecommendedChart, resolveChartFields。
设计职责：作为该路径所表明的 UI 单元或工具模块，负责把 Redux 状态映射为可视控件，或把用户操作映射为 API/thunk。交互过程中的加载、空态、错误态必须可区分。若涉及 Agent 流，必须能被 AbortSignal 取消，并在 unmount 时停止写入已卸载组件。
状态约束：不得在组件本地复制一份“即将过期”的表全量行；需要行数据时通过 tableResolution/preview cache 物化。图表变更应走 dfActions 以便缩略图服务订阅。
测试对应：`tests/frontend/unit` 中同名或相近测试文件覆盖选择器、迁移、错误码、关键组件渲染契约。新增 props 时同步更新这些测试。


#### `src/app/workspaceDB.ts`（约 129 行）

本文件导出 3 个符号：TableIndexEntry, WorkspaceEntry, workspaceDB。
设计职责：作为该路径所表明的 UI 单元或工具模块，负责把 Redux 状态映射为可视控件，或把用户操作映射为 API/thunk。交互过程中的加载、空态、错误态必须可区分。若涉及 Agent 流，必须能被 AbortSignal 取消，并在 unmount 时停止写入已卸载组件。
状态约束：不得在组件本地复制一份“即将过期”的表全量行；需要行数据时通过 tableResolution/preview cache 物化。图表变更应走 dfActions 以便缩略图服务订阅。
测试对应：`tests/frontend/unit` 中同名或相近测试文件覆盖选择器、迁移、错误码、关键组件渲染契约。新增 props 时同步更新这些测试。


#### `src/app/workspaceService.ts`（约 298 行）

本文件导出 13 个符号：WorkspaceSummary, onWorkspaceListChanged, WorkspaceLoadSupersededError, listWorkspaces, loadWorkspace, deleteWorkspace, updateWorkspaceMeta, saveWorkspaceState, exportWorkspace, importWorkspace, deleteTableFromWorkspace, deleteTablesFromWorkspace, isWorkspaceReadOnly。
设计职责：作为该路径所表明的 UI 单元或工具模块，负责把 Redux 状态映射为可视控件，或把用户操作映射为 API/thunk。交互过程中的加载、空态、错误态必须可区分。若涉及 Agent 流，必须能被 AbortSignal 取消，并在 unmount 时停止写入已卸载组件。
状态约束：不得在组件本地复制一份“即将过期”的表全量行；需要行数据时通过 tableResolution/preview cache 物化。图表变更应走 dfActions 以便缩略图服务订阅。
测试对应：`tests/frontend/unit` 中同名或相近测试文件覆盖选择器、迁移、错误码、关键组件渲染契约。新增 props 时同步更新这些测试。


### 前端目录 `src/components`

该目录下的 TypeScript/TSX 文件构成 UI 或客户端基础设施。组件必须使用 `t()`/`i18n.t()` 输出用户可见文本；thunk 与非组件工具不得调用 React hooks。


#### `src/components/AnvilLoader.tsx`（约 116 行）

本文件导出 2 个符号：AnvilLoaderProps, AnvilLoader。
设计职责：作为该路径所表明的 UI 单元或工具模块，负责把 Redux 状态映射为可视控件，或把用户操作映射为 API/thunk。交互过程中的加载、空态、错误态必须可区分。若涉及 Agent 流，必须能被 AbortSignal 取消，并在 unmount 时停止写入已卸载组件。
状态约束：不得在组件本地复制一份“即将过期”的表全量行；需要行数据时通过 tableResolution/preview cache 物化。图表变更应走 dfActions 以便缩略图服务订阅。
测试对应：`tests/frontend/unit` 中同名或相近测试文件覆盖选择器、迁移、错误码、关键组件渲染契约。新增 props 时同步更新这些测试。


#### `src/components/CatalogTree.tsx`（约 245 行）

本文件导出 9 个符号：CatalogTreeNode, collectNamespaceIds, mergeChildrenAtPath, appendChildrenAtPath, findNodeByPath, StyledTreeItem, CountBadge, RenderCatalogTreeOptions, renderCatalogTreeItems。
设计职责：作为该路径所表明的 UI 单元或工具模块，负责把 Redux 状态映射为可视控件，或把用户操作映射为 API/thunk。交互过程中的加载、空态、错误态必须可区分。若涉及 Agent 流，必须能被 AbortSignal 取消，并在 unmount 时停止写入已卸载组件。
状态约束：不得在组件本地复制一份“即将过期”的表全量行；需要行数据时通过 tableResolution/preview cache 物化。图表变更应走 dfActions 以便缩略图服务订阅。
测试对应：`tests/frontend/unit` 中同名或相近测试文件覆盖选择器、迁移、错误码、关键组件渲染契约。新增 props 时同步更新这些测试。


#### `src/components/ChartTemplates.tsx`（约 157 行）

本文件导出 6 个符号：CHART_ICONS, CHART_TEMPLATES, getChartTemplate, getChartChannels, channels, channelGroups。
设计职责：作为该路径所表明的 UI 单元或工具模块，负责把 Redux 状态映射为可视控件，或把用户操作映射为 API/thunk。交互过程中的加载、空态、错误态必须可区分。若涉及 Agent 流，必须能被 AbortSignal 取消，并在 unmount 时停止写入已卸载组件。
状态约束：不得在组件本地复制一份“即将过期”的表全量行；需要行数据时通过 tableResolution/preview cache 物化。图表变更应走 dfActions 以便缩略图服务订阅。
测试对应：`tests/frontend/unit` 中同名或相近测试文件覆盖选择器、迁移、错误码、关键组件渲染契约。新增 props 时同步更新这些测试。


#### `src/components/ComponentType.tsx`（约 687 行）

本文件导出 56 个符号：FieldSource, FieldItem, duplicateField, ROOTLESS_THREAD_ID, Trigger, Actor, ClarificationOption, ClarificationQuestion, ClarificationResponse, DelegateTarget, InteractionEntry, DeriveStatus, LoadedTableNode, PendingClarification, DraftNode, ThreadNode, TextTurn, DataCleanTableOutput, DataCleanBlock, ChatAttachment, InlineTablePreview, CodeExecution, PendingTableLoad, LoadPlanCandidate, LoadPlan, ConnectorFormPrompt, ConnectorFormArtifact, FormArtifact, ChatMessage, DataSourceType, DataSourceConfig, InputTableSource, InputTableColumn, FieldSemanticsInfo, TableSemanticsInfo, InputTableSnapshot, InputTablePreview, InputTable, DictTable, TableNode。
设计职责：作为该路径所表明的 UI 单元或工具模块，负责把 Redux 状态映射为可视控件，或把用户操作映射为 API/thunk。交互过程中的加载、空态、错误态必须可区分。若涉及 Agent 流，必须能被 AbortSignal 取消，并在 unmount 时停止写入已卸载组件。
状态约束：不得在组件本地复制一份“即将过期”的表全量行；需要行数据时通过 tableResolution/preview cache 物化。图表变更应走 dfActions 以便缩略图服务订阅。
测试对应：`tests/frontend/unit` 中同名或相近测试文件覆盖选择器、迁移、错误码、关键组件渲染契约。新增 props 时同步更新这些测试。


#### `src/components/ConnectorFormCard.tsx`（约 393 行）

本文件导出 1 个符号：ConnectorFormCard。
设计职责：作为该路径所表明的 UI 单元或工具模块，负责把 Redux 状态映射为可视控件，或把用户操作映射为 API/thunk。交互过程中的加载、空态、错误态必须可区分。若涉及 Agent 流，必须能被 AbortSignal 取消，并在 unmount 时停止写入已卸载组件。
状态约束：不得在组件本地复制一份“即将过期”的表全量行；需要行数据时通过 tableResolution/preview cache 物化。图表变更应走 dfActions 以便缩略图服务订阅。
测试对应：`tests/frontend/unit` 中同名或相近测试文件覆盖选择器、迁移、错误码、关键组件渲染契约。新增 props 时同步更新这些测试。


#### `src/components/ConnectorTablePreview.tsx`（约 738 行）

本文件导出 8 个符号：ColumnMeta, PreviewFilter, SourceFilter, ConnectorTablePreviewProps, inferInputType, defaultOperatorForType, coerceFilters, ConnectorTablePreview。
设计职责：作为该路径所表明的 UI 单元或工具模块，负责把 Redux 状态映射为可视控件，或把用户操作映射为 API/thunk。交互过程中的加载、空态、错误态必须可区分。若涉及 Agent 流，必须能被 AbortSignal 取消，并在 unmount 时停止写入已卸载组件。
状态约束：不得在组件本地复制一份“即将过期”的表全量行；需要行数据时通过 tableResolution/preview cache 物化。图表变更应走 dfActions 以便缩略图服务订阅。
测试对应：`tests/frontend/unit` 中同名或相近测试文件覆盖选择器、迁移、错误码、关键组件渲染契约。新增 props 时同步更新这些测试。


#### `src/components/DataOperationCard.tsx`（约 99 行）

本文件导出 1 个符号：DataOperationCard。
设计职责：作为该路径所表明的 UI 单元或工具模块，负责把 Redux 状态映射为可视控件，或把用户操作映射为 API/thunk。交互过程中的加载、空态、错误态必须可区分。若涉及 Agent 流，必须能被 AbortSignal 取消，并在 unmount 时停止写入已卸载组件。
状态约束：不得在组件本地复制一份“即将过期”的表全量行；需要行数据时通过 tableResolution/preview cache 物化。图表变更应走 dfActions 以便缩略图服务订阅。
测试对应：`tests/frontend/unit` 中同名或相近测试文件覆盖选择器、迁移、错误码、关键组件渲染契约。新增 props 时同步更新这些测试。


#### `src/components/DndTypes.ts`（约 17 行）

本文件导出 2 个符号：CATALOG_TABLE_ITEM, CatalogTableDragItem。
设计职责：作为该路径所表明的 UI 单元或工具模块，负责把 Redux 状态映射为可视控件，或把用户操作映射为 API/thunk。交互过程中的加载、空态、错误态必须可区分。若涉及 Agent 流，必须能被 AbortSignal 取消，并在 unmount 时停止写入已卸载组件。
状态约束：不得在组件本地复制一份“即将过期”的表全量行；需要行数据时通过 tableResolution/preview cache 物化。图表变更应走 dfActions 以便缩略图服务订阅。
测试对应：`tests/frontend/unit` 中同名或相近测试文件覆盖选择器、迁移、错误码、关键组件渲染契约。新增 props 时同步更新这些测试。


#### `src/components/FunComponents.tsx`（约 78 行）

本文件导出 4 个符号：WritingPencil, ShimmerText, WritingIndicator, ThinkingBufferEffect。
设计职责：作为该路径所表明的 UI 单元或工具模块，负责把 Redux 状态映射为可视控件，或把用户操作映射为 API/thunk。交互过程中的加载、空态、错误态必须可区分。若涉及 Agent 流，必须能被 AbortSignal 取消，并在 unmount 时停止写入已卸载组件。
状态约束：不得在组件本地复制一份“即将过期”的表全量行；需要行数据时通过 tableResolution/preview cache 物化。图表变更应走 dfActions 以便缩略图服务订阅。
测试对应：`tests/frontend/unit` 中同名或相近测试文件覆盖选择器、迁移、错误码、关键组件渲染契约。新增 props 时同步更新这些测试。


#### `src/components/LoadPlanCard.tsx`（约 495 行）

本文件导出 3 个符号：PresentedLoadCandidate, buildLoadQueryImportOptions, LoadPlanCard。
设计职责：作为该路径所表明的 UI 单元或工具模块，负责把 Redux 状态映射为可视控件，或把用户操作映射为 API/thunk。交互过程中的加载、空态、错误态必须可区分。若涉及 Agent 流，必须能被 AbortSignal 取消，并在 unmount 时停止写入已卸载组件。
状态约束：不得在组件本地复制一份“即将过期”的表全量行；需要行数据时通过 tableResolution/preview cache 物化。图表变更应走 dfActions 以便缩略图服务订阅。
测试对应：`tests/frontend/unit` 中同名或相近测试文件覆盖选择器、迁移、错误码、关键组件渲染契约。新增 props 时同步更新这些测试。


#### `src/components/MarkdownEditor.tsx`（约 100 行）

本文件导出 1 个符号：MarkdownEditor。
设计职责：作为该路径所表明的 UI 单元或工具模块，负责把 Redux 状态映射为可视控件，或把用户操作映射为 API/thunk。交互过程中的加载、空态、错误态必须可区分。若涉及 Agent 流，必须能被 AbortSignal 取消，并在 unmount 时停止写入已卸载组件。
状态约束：不得在组件本地复制一份“即将过期”的表全量行；需要行数据时通过 tableResolution/preview cache 物化。图表变更应走 dfActions 以便缩略图服务订阅。
测试对应：`tests/frontend/unit` 中同名或相近测试文件覆盖选择器、迁移、错误码、关键组件渲染契约。新增 props 时同步更新这些测试。


#### `src/components/ResizeHandle.tsx`（约 115 行）

本文件导出 2 个符号：ResizeHandleProps, ResizeHandle。
设计职责：作为该路径所表明的 UI 单元或工具模块，负责把 Redux 状态映射为可视控件，或把用户操作映射为 API/thunk。交互过程中的加载、空态、错误态必须可区分。若涉及 Agent 流，必须能被 AbortSignal 取消，并在 unmount 时停止写入已卸载组件。
状态约束：不得在组件本地复制一份“即将过期”的表全量行；需要行数据时通过 tableResolution/preview cache 物化。图表变更应走 dfActions 以便缩略图服务订阅。
测试对应：`tests/frontend/unit` 中同名或相近测试文件覆盖选择器、迁移、错误码、关键组件渲染契约。新增 props 时同步更新这些测试。


#### `src/components/RotatingTextBlock.tsx`（约 49 行）

本文件导出 1 个符号：RotatingTextBlock。
设计职责：作为该路径所表明的 UI 单元或工具模块，负责把 Redux 状态映射为可视控件，或把用户操作映射为 API/thunk。交互过程中的加载、空态、错误态必须可区分。若涉及 Agent 流，必须能被 AbortSignal 取消，并在 unmount 时停止写入已卸载组件。
状态约束：不得在组件本地复制一份“即将过期”的表全量行；需要行数据时通过 tableResolution/preview cache 物化。图表变更应走 dfActions 以便缩略图服务订阅。
测试对应：`tests/frontend/unit` 中同名或相近测试文件覆盖选择器、迁移、错误码、关键组件渲染契约。新增 props 时同步更新这些测试。


#### `src/components/ScrollFade.tsx`（约 114 行）

本文件导出 4 个符号：SCROLL_FADE, useScrollFade, ScrollFadeEdge, ScrollFadeContainer。
设计职责：作为该路径所表明的 UI 单元或工具模块，负责把 Redux 状态映射为可视控件，或把用户操作映射为 API/thunk。交互过程中的加载、空态、错误态必须可区分。若涉及 Agent 流，必须能被 AbortSignal 取消，并在 unmount 时停止写入已卸载组件。
状态约束：不得在组件本地复制一份“即将过期”的表全量行；需要行数据时通过 tableResolution/preview cache 物化。图表变更应走 dfActions 以便缩略图服务订阅。
测试对应：`tests/frontend/unit` 中同名或相近测试文件覆盖选择器、迁移、错误码、关键组件渲染契约。新增 props 时同步更新这些测试。


#### `src/components/TablePreviewRow.tsx`（约 152 行）

本文件导出 3 个符号：TablePreviewData, TablePreviewRowProps, TablePreviewRow。
设计职责：作为该路径所表明的 UI 单元或工具模块，负责把 Redux 状态映射为可视控件，或把用户操作映射为 API/thunk。交互过程中的加载、空态、错误态必须可区分。若涉及 Agent 流，必须能被 AbortSignal 取消，并在 unmount 时停止写入已卸载组件。
状态约束：不得在组件本地复制一份“即将过期”的表全量行；需要行数据时通过 tableResolution/preview cache 物化。图表变更应走 dfActions 以便缩略图服务订阅。
测试对应：`tests/frontend/unit` 中同名或相近测试文件覆盖选择器、迁移、错误码、关键组件渲染契约。新增 props 时同步更新这些测试。


#### `src/components/VirtualizedCatalogTree.tsx`（约 531 行）

本文件导出 2 个符号：VirtualizedCatalogTreeProps, VirtualizedCatalogTree。
设计职责：作为该路径所表明的 UI 单元或工具模块，负责把 Redux 状态映射为可视控件，或把用户操作映射为 API/thunk。交互过程中的加载、空态、错误态必须可区分。若涉及 Agent 流，必须能被 AbortSignal 取消，并在 unmount 时停止写入已卸载组件。
状态约束：不得在组件本地复制一份“即将过期”的表全量行；需要行数据时通过 tableResolution/preview cache 物化。图表变更应走 dfActions 以便缩略图服务订阅。
测试对应：`tests/frontend/unit` 中同名或相近测试文件覆盖选择器、迁移、错误码、关键组件渲染契约。新增 props 时同步更新这些测试。


#### `src/components/filterFormat.ts`（约 69 行）

本文件导出 3 个符号：FILTER_OPERATOR_SYMBOLS, formatFilterOperator, formatFilterChipLabel。
设计职责：作为该路径所表明的 UI 单元或工具模块，负责把 Redux 状态映射为可视控件，或把用户操作映射为 API/thunk。交互过程中的加载、空态、错误态必须可区分。若涉及 Agent 流，必须能被 AbortSignal 取消，并在 unmount 时停止写入已卸载组件。
状态约束：不得在组件本地复制一份“即将过期”的表全量行；需要行数据时通过 tableResolution/preview cache 物化。图表变更应走 dfActions 以便缩略图服务订阅。
测试对应：`tests/frontend/unit` 中同名或相近测试文件覆盖选择器、迁移、错误码、关键组件渲染契约。新增 props 时同步更新这些测试。


### 前端目录 `src/data`

该目录下的 TypeScript/TSX 文件构成 UI 或客户端基础设施。组件必须使用 `t()`/`i18n.t()` 输出用户可见文本；thunk 与非组件工具不得调用 React hooks。


#### `src/data/column.ts`（约 37 行）

本文件导出 1 个符号：Column。
设计职责：作为该路径所表明的 UI 单元或工具模块，负责把 Redux 状态映射为可视控件，或把用户操作映射为 API/thunk。交互过程中的加载、空态、错误态必须可区分。若涉及 Agent 流，必须能被 AbortSignal 取消，并在 unmount 时停止写入已卸载组件。
状态约束：不得在组件本地复制一份“即将过期”的表全量行；需要行数据时通过 tableResolution/preview cache 物化。图表变更应走 dfActions 以便缩略图服务订阅。
测试对应：`tests/frontend/unit` 中同名或相近测试文件覆盖选择器、迁移、错误码、关键组件渲染契约。新增 props 时同步更新这些测试。


#### `src/data/table.ts`（约 73 行）

本文件导出 1 个符号：ColumnTable。
设计职责：作为该路径所表明的 UI 单元或工具模块，负责把 Redux 状态映射为可视控件，或把用户操作映射为 API/thunk。交互过程中的加载、空态、错误态必须可区分。若涉及 Agent 流，必须能被 AbortSignal 取消，并在 unmount 时停止写入已卸载组件。
状态约束：不得在组件本地复制一份“即将过期”的表全量行；需要行数据时通过 tableResolution/preview cache 物化。图表变更应走 dfActions 以便缩略图服务订阅。
测试对应：`tests/frontend/unit` 中同名或相近测试文件覆盖选择器、迁移、错误码、关键组件渲染契约。新增 props 时同步更新这些测试。


#### `src/data/types.ts`（约 156 行）

本文件导出 11 个符号：Type, TypeList, CoerceType, TestType, isBoolean, isNumber, isDate, getDType, testType, mapApiTypeToAppType, isTemporalType。
设计职责：作为该路径所表明的 UI 单元或工具模块，负责把 Redux 状态映射为可视控件，或把用户操作映射为 API/thunk。交互过程中的加载、空态、错误态必须可区分。若涉及 Agent 流，必须能被 AbortSignal 取消，并在 unmount 时停止写入已卸载组件。
状态约束：不得在组件本地复制一份“即将过期”的表全量行；需要行数据时通过 tableResolution/preview cache 物化。图表变更应走 dfActions 以便缩略图服务订阅。
测试对应：`tests/frontend/unit` 中同名或相近测试文件覆盖选择器、迁移、错误码、关键组件渲染契约。新增 props 时同步更新这些测试。


#### `src/data/utils.ts`（约 328 行）

本文件导出 14 个符号：readFileText, loadTextDataWrapper, createTableFromText, createTableFromFromObjectArray, inferTypeFromValueArray, refineTemporalType, convertTypeToDtype, coerceValueArrayFromTypes, coerceValueFromTypes, computeUniqueValues, tupleEqual, resolveExcelCellValue, loadBinaryDataWrapper, exportTableToDsv。
设计职责：作为该路径所表明的 UI 单元或工具模块，负责把 Redux 状态映射为可视控件，或把用户操作映射为 API/thunk。交互过程中的加载、空态、错误态必须可区分。若涉及 Agent 流，必须能被 AbortSignal 取消，并在 unmount 时停止写入已卸载组件。
状态约束：不得在组件本地复制一份“即将过期”的表全量行；需要行数据时通过 tableResolution/preview cache 物化。图表变更应走 dfActions 以便缩略图服务订阅。
测试对应：`tests/frontend/unit` 中同名或相近测试文件覆盖选择器、迁移、错误码、关键组件渲染契约。新增 props 时同步更新这些测试。


### 前端目录 `src/dataOperations`

该目录下的 TypeScript/TSX 文件构成 UI 或客户端基础设施。组件必须使用 `t()`/`i18n.t()` 输出用户可见文本；thunk 与非组件工具不得调用 React hooks。


#### `src/dataOperations/models.ts`（约 230 行）

本文件导出 12 个符号：DATA_OPERATION_SCHEMA_VERSION, JsonValue, DataOperationStatus, OperationFilter, LoadQuery, ConnectorQueryStepSummary, DataOperationStepSummary, DataOperationPlan, OperationError, FailedOperationStep, DataOperation, parseDataOperation。
设计职责：作为该路径所表明的 UI 单元或工具模块，负责把 Redux 状态映射为可视控件，或把用户操作映射为 API/thunk。交互过程中的加载、空态、错误态必须可区分。若涉及 Agent 流，必须能被 AbortSignal 取消，并在 unmount 时停止写入已卸载组件。
状态约束：不得在组件本地复制一份“即将过期”的表全量行；需要行数据时通过 tableResolution/preview cache 物化。图表变更应走 dfActions 以便缩略图服务订阅。
测试对应：`tests/frontend/unit` 中同名或相近测试文件覆盖选择器、迁移、错误码、关键组件渲染契约。新增 props 时同步更新这些测试。


### 前端目录 `src/i18n`

该目录下的 TypeScript/TSX 文件构成 UI 或客户端基础设施。组件必须使用 `t()`/`i18n.t()` 输出用户可见文本；thunk 与非组件工具不得调用 React hooks。


#### `src/i18n/index.ts`（约 33 行）

本文件导出 0 个符号：（无 export 或仅为副作用入口）。
设计职责：作为该路径所表明的 UI 单元或工具模块，负责把 Redux 状态映射为可视控件，或把用户操作映射为 API/thunk。交互过程中的加载、空态、错误态必须可区分。若涉及 Agent 流，必须能被 AbortSignal 取消，并在 unmount 时停止写入已卸载组件。
状态约束：不得在组件本地复制一份“即将过期”的表全量行；需要行数据时通过 tableResolution/preview cache 物化。图表变更应走 dfActions 以便缩略图服务订阅。
测试对应：`tests/frontend/unit` 中同名或相近测试文件覆盖选择器、迁移、错误码、关键组件渲染契约。新增 props 时同步更新这些测试。


#### `src/i18n/vega-locale.ts`（约 50 行）

本文件导出 1 个符号：syncVegaLocale。
设计职责：作为该路径所表明的 UI 单元或工具模块，负责把 Redux 状态映射为可视控件，或把用户操作映射为 API/thunk。交互过程中的加载、空态、错误态必须可区分。若涉及 Agent 流，必须能被 AbortSignal 取消，并在 unmount 时停止写入已卸载组件。
状态约束：不得在组件本地复制一份“即将过期”的表全量行；需要行数据时通过 tableResolution/preview cache 物化。图表变更应走 dfActions 以便缩略图服务订阅。
测试对应：`tests/frontend/unit` 中同名或相近测试文件覆盖选择器、迁移、错误码、关键组件渲染契约。新增 props 时同步更新这些测试。


### 前端目录 `src/i18n/locales`

该目录下的 TypeScript/TSX 文件构成 UI 或客户端基础设施。组件必须使用 `t()`/`i18n.t()` 输出用户可见文本；thunk 与非组件工具不得调用 React hooks。


#### `src/i18n/locales/index.ts`（约 8 行）

本文件导出 2 个符号：en, zh。
设计职责：作为该路径所表明的 UI 单元或工具模块，负责把 Redux 状态映射为可视控件，或把用户操作映射为 API/thunk。交互过程中的加载、空态、错误态必须可区分。若涉及 Agent 流，必须能被 AbortSignal 取消，并在 unmount 时停止写入已卸载组件。
状态约束：不得在组件本地复制一份“即将过期”的表全量行；需要行数据时通过 tableResolution/preview cache 物化。图表变更应走 dfActions 以便缩略图服务订阅。
测试对应：`tests/frontend/unit` 中同名或相近测试文件覆盖选择器、迁移、错误码、关键组件渲染契约。新增 props 时同步更新这些测试。


### 前端目录 `src/i18n/locales/en`

该目录下的 TypeScript/TSX 文件构成 UI 或客户端基础设施。组件必须使用 `t()`/`i18n.t()` 输出用户可见文本；thunk 与非组件工具不得调用 React hooks。


#### `src/i18n/locales/en/index.ts`（约 27 行）

本文件导出 0 个符号：（无 export 或仅为副作用入口）。
设计职责：作为该路径所表明的 UI 单元或工具模块，负责把 Redux 状态映射为可视控件，或把用户操作映射为 API/thunk。交互过程中的加载、空态、错误态必须可区分。若涉及 Agent 流，必须能被 AbortSignal 取消，并在 unmount 时停止写入已卸载组件。
状态约束：不得在组件本地复制一份“即将过期”的表全量行；需要行数据时通过 tableResolution/preview cache 物化。图表变更应走 dfActions 以便缩略图服务订阅。
测试对应：`tests/frontend/unit` 中同名或相近测试文件覆盖选择器、迁移、错误码、关键组件渲染契约。新增 props 时同步更新这些测试。


### 前端目录 `src/i18n/locales/zh`

该目录下的 TypeScript/TSX 文件构成 UI 或客户端基础设施。组件必须使用 `t()`/`i18n.t()` 输出用户可见文本；thunk 与非组件工具不得调用 React hooks。


#### `src/i18n/locales/zh/index.ts`（约 27 行）

本文件导出 0 个符号：（无 export 或仅为副作用入口）。
设计职责：作为该路径所表明的 UI 单元或工具模块，负责把 Redux 状态映射为可视控件，或把用户操作映射为 API/thunk。交互过程中的加载、空态、错误态必须可区分。若涉及 Agent 流，必须能被 AbortSignal 取消，并在 unmount 时停止写入已卸载组件。
状态约束：不得在组件本地复制一份“即将过期”的表全量行；需要行数据时通过 tableResolution/preview cache 物化。图表变更应走 dfActions 以便缩略图服务订阅。
测试对应：`tests/frontend/unit` 中同名或相近测试文件覆盖选择器、迁移、错误码、关键组件渲染契约。新增 props 时同步更新这些测试。


### 前端目录 `src/views`

该目录下的 TypeScript/TSX 文件构成 UI 或客户端基础设施。组件必须使用 `t()`/`i18n.t()` 输出用户可见文本；thunk 与非组件工具不得调用 React hooks。


#### `src/views/About.tsx`（约 254 行）

本文件导出 1 个符号：About。
设计职责：作为该路径所表明的 UI 单元或工具模块，负责把 Redux 状态映射为可视控件，或把用户操作映射为 API/thunk。交互过程中的加载、空态、错误态必须可区分。若涉及 Agent 流，必须能被 AbortSignal 取消，并在 unmount 时停止写入已卸载组件。
状态约束：不得在组件本地复制一份“即将过期”的表全量行；需要行数据时通过 tableResolution/preview cache 物化。图表变更应走 dfActions 以便缩略图服务订阅。
测试对应：`tests/frontend/unit` 中同名或相近测试文件覆盖选择器、迁移、错误码、关键组件渲染契约。新增 props 时同步更新这些测试。


#### `src/views/AgentChatInput.tsx`（约 601 行）

本文件导出 2 个符号：AgentChatInputProps, AgentChatInput。
设计职责：作为该路径所表明的 UI 单元或工具模块，负责把 Redux 状态映射为可视控件，或把用户操作映射为 API/thunk。交互过程中的加载、空态、错误态必须可区分。若涉及 Agent 流，必须能被 AbortSignal 取消，并在 unmount 时停止写入已卸载组件。
状态约束：不得在组件本地复制一份“即将过期”的表全量行；需要行数据时通过 tableResolution/preview cache 物化。图表变更应走 dfActions 以便缩略图服务订阅。
测试对应：`tests/frontend/unit` 中同名或相近测试文件覆盖选择器、迁移、错误码、关键组件渲染契约。新增 props 时同步更新这些测试。


#### `src/views/AgentPausePanel.tsx`（约 673 行）

本文件导出 2 个符号：ClarificationPanel, ExplanationPanel。
设计职责：作为该路径所表明的 UI 单元或工具模块，负责把 Redux 状态映射为可视控件，或把用户操作映射为 API/thunk。交互过程中的加载、空态、错误态必须可区分。若涉及 Agent 流，必须能被 AbortSignal 取消，并在 unmount 时停止写入已卸载组件。
状态约束：不得在组件本地复制一份“即将过期”的表全量行；需要行数据时通过 tableResolution/preview cache 物化。图表变更应走 dfActions 以便缩略图服务订阅。
测试对应：`tests/frontend/unit` 中同名或相近测试文件覆盖选择器、迁移、错误码、关键组件渲染契约。新增 props 时同步更新这些测试。


#### `src/views/AgentRulesDialog.tsx`（约 393 行）

本文件导出 1 个符号：AgentRulesDialog。
设计职责：作为该路径所表明的 UI 单元或工具模块，负责把 Redux 状态映射为可视控件，或把用户操作映射为 API/thunk。交互过程中的加载、空态、错误态必须可区分。若涉及 Agent 流，必须能被 AbortSignal 取消，并在 unmount 时停止写入已卸载组件。
状态约束：不得在组件本地复制一份“即将过期”的表全量行；需要行数据时通过 tableResolution/preview cache 物化。图表变更应走 dfActions 以便缩略图服务订阅。
测试对应：`tests/frontend/unit` 中同名或相近测试文件覆盖选择器、迁移、错误码、关键组件渲染契约。新增 props 时同步更新这些测试。


#### `src/views/AgentToyIcon.tsx`（约 138 行）

本文件导出 3 个符号：AgentToyVariant, AgentToyIcon, AnimatedAgentToyIcon。
设计职责：作为该路径所表明的 UI 单元或工具模块，负责把 Redux 状态映射为可视控件，或把用户操作映射为 API/thunk。交互过程中的加载、空态、错误态必须可区分。若涉及 Agent 流，必须能被 AbortSignal 取消，并在 unmount 时停止写入已卸载组件。
状态约束：不得在组件本地复制一份“即将过期”的表全量行；需要行数据时通过 tableResolution/preview cache 物化。图表变更应走 dfActions 以便缩略图服务订阅。
测试对应：`tests/frontend/unit` 中同名或相近测试文件覆盖选择器、迁移、错误码、关键组件渲染契约。新增 props 时同步更新这些测试。


#### `src/views/ChartQuickConfig.tsx`（约 417 行）

本文件导出 2 个符号：ChartQuickConfigProps, ChartQuickConfig。
设计职责：作为该路径所表明的 UI 单元或工具模块，负责把 Redux 状态映射为可视控件，或把用户操作映射为 API/thunk。交互过程中的加载、空态、错误态必须可区分。若涉及 Agent 流，必须能被 AbortSignal 取消，并在 unmount 时停止写入已卸载组件。
状态约束：不得在组件本地复制一份“即将过期”的表全量行；需要行数据时通过 tableResolution/preview cache 物化。图表变更应走 dfActions 以便缩略图服务订阅。
测试对应：`tests/frontend/unit` 中同名或相近测试文件覆盖选择器、迁移、错误码、关键组件渲染契约。新增 props 时同步更新这些测试。


#### `src/views/ChartRenderService.tsx`（约 360 行）

本文件导出 1 个符号：ChartRenderService。
设计职责：作为该路径所表明的 UI 单元或工具模块，负责把 Redux 状态映射为可视控件，或把用户操作映射为 API/thunk。交互过程中的加载、空态、错误态必须可区分。若涉及 Agent 流，必须能被 AbortSignal 取消，并在 unmount 时停止写入已卸载组件。
状态约束：不得在组件本地复制一份“即将过期”的表全量行；需要行数据时通过 tableResolution/preview cache 物化。图表变更应走 dfActions 以便缩略图服务订阅。
测试对应：`tests/frontend/unit` 中同名或相近测试文件覆盖选择器、迁移、错误码、关键组件渲染契约。新增 props 时同步更新这些测试。


#### `src/views/ChartUtils.tsx`（约 65 行）

本文件导出 0 个符号：（无 export 或仅为副作用入口）。
设计职责：作为该路径所表明的 UI 单元或工具模块，负责把 Redux 状态映射为可视控件，或把用户操作映射为 API/thunk。交互过程中的加载、空态、错误态必须可区分。若涉及 Agent 流，必须能被 AbortSignal 取消，并在 unmount 时停止写入已卸载组件。
状态约束：不得在组件本地复制一份“即将过期”的表全量行；需要行数据时通过 tableResolution/preview cache 物化。图表变更应走 dfActions 以便缩略图服务订阅。
测试对应：`tests/frontend/unit` 中同名或相近测试文件覆盖选择器、迁移、错误码、关键组件渲染契约。新增 props 时同步更新这些测试。


#### `src/views/ChartVariantStrip.tsx`（约 636 行）

本文件导出 2 个符号：ChartVariantStripProps, ChartVariantStrip。
设计职责：作为该路径所表明的 UI 单元或工具模块，负责把 Redux 状态映射为可视控件，或把用户操作映射为 API/thunk。交互过程中的加载、空态、错误态必须可区分。若涉及 Agent 流，必须能被 AbortSignal 取消，并在 unmount 时停止写入已卸载组件。
状态约束：不得在组件本地复制一份“即将过期”的表全量行；需要行数据时通过 tableResolution/preview cache 物化。图表变更应走 dfActions 以便缩略图服务订阅。
测试对应：`tests/frontend/unit` 中同名或相近测试文件覆盖选择器、迁移、错误码、关键组件渲染契约。新增 props 时同步更新这些测试。


#### `src/views/ChartifactDialog.tsx`（约 308 行）

本文件导出 2 个符号：convertToChartifact, openChartifactViewer。
设计职责：作为该路径所表明的 UI 单元或工具模块，负责把 Redux 状态映射为可视控件，或把用户操作映射为 API/thunk。交互过程中的加载、空态、错误态必须可区分。若涉及 Agent 流，必须能被 AbortSignal 取消，并在 unmount 时停止写入已卸载组件。
状态约束：不得在组件本地复制一份“即将过期”的表全量行；需要行数据时通过 tableResolution/preview cache 物化。图表变更应走 dfActions 以便缩略图服务订阅。
测试对应：`tests/frontend/unit` 中同名或相近测试文件覆盖选择器、迁移、错误码、关键组件渲染契约。新增 props 时同步更新这些测试。


#### `src/views/ChatDialog.tsx`（约 450 行）

本文件导出 4 个符号：GroupHeader, GroupItems, ChatDialogProps, ChatDialog。
设计职责：作为该路径所表明的 UI 单元或工具模块，负责把 Redux 状态映射为可视控件，或把用户操作映射为 API/thunk。交互过程中的加载、空态、错误态必须可区分。若涉及 Agent 流，必须能被 AbortSignal 取消，并在 unmount 时停止写入已卸载组件。
状态约束：不得在组件本地复制一份“即将过期”的表全量行；需要行数据时通过 tableResolution/preview cache 物化。图表变更应走 dfActions 以便缩略图服务订阅。
测试对应：`tests/frontend/unit` 中同名或相近测试文件覆盖选择器、迁移、错误码、关键组件渲染契约。新增 props 时同步更新这些测试。


#### `src/views/ColumnFilterPopover.tsx`（约 632 行）

本文件导出 5 个符号：RangeFilter, InFilter, ContainsFilter, ColumnFilter, ColumnFilterPopover。
设计职责：作为该路径所表明的 UI 单元或工具模块，负责把 Redux 状态映射为可视控件，或把用户操作映射为 API/thunk。交互过程中的加载、空态、错误态必须可区分。若涉及 Agent 流，必须能被 AbortSignal 取消，并在 unmount 时停止写入已卸载组件。
状态约束：不得在组件本地复制一份“即将过期”的表全量行；需要行数据时通过 tableResolution/preview cache 物化。图表变更应走 dfActions 以便缩略图服务订阅。
测试对应：`tests/frontend/unit` 中同名或相近测试文件覆盖选择器、迁移、错误码、关键组件渲染契约。新增 props 时同步更新这些测试。


#### `src/views/DBTableManager.tsx`（约 1248 行）

本文件导出 1 个符号：DataLoaderForm。
设计职责：作为该路径所表明的 UI 单元或工具模块，负责把 Redux 状态映射为可视控件，或把用户操作映射为 API/thunk。交互过程中的加载、空态、错误态必须可区分。若涉及 Agent 流，必须能被 AbortSignal 取消，并在 unmount 时停止写入已卸载组件。
状态约束：不得在组件本地复制一份“即将过期”的表全量行；需要行数据时通过 tableResolution/preview cache 物化。图表变更应走 dfActions 以便缩略图服务订阅。
测试对应：`tests/frontend/unit` 中同名或相近测试文件覆盖选择器、迁移、错误码、关键组件渲染契约。新增 props 时同步更新这些测试。


#### `src/views/DataFormulator.tsx`（约 1200 行）

本文件导出 1 个符号：DataFormulatorFC。
设计职责：作为该路径所表明的 UI 单元或工具模块，负责把 Redux 状态映射为可视控件，或把用户操作映射为 API/thunk。交互过程中的加载、空态、错误态必须可区分。若涉及 Agent 流，必须能被 AbortSignal 取消，并在 unmount 时停止写入已卸载组件。
状态约束：不得在组件本地复制一份“即将过期”的表全量行；需要行数据时通过 tableResolution/preview cache 物化。图表变更应走 dfActions 以便缩略图服务订阅。
测试对应：`tests/frontend/unit` 中同名或相近测试文件覆盖选择器、迁移、错误码、关键组件渲染契约。新增 props 时同步更新这些测试。


#### `src/views/DataFrameTable.tsx`（约 273 行）

本文件导出 2 个符号：DataFrameTableProps, DataFrameTable。
设计职责：作为该路径所表明的 UI 单元或工具模块，负责把 Redux 状态映射为可视控件，或把用户操作映射为 API/thunk。交互过程中的加载、空态、错误态必须可区分。若涉及 Agent 流，必须能被 AbortSignal 取消，并在 unmount 时停止写入已卸载组件。
状态约束：不得在组件本地复制一份“即将过期”的表全量行；需要行数据时通过 tableResolution/preview cache 物化。图表变更应走 dfActions 以便缩略图服务订阅。
测试对应：`tests/frontend/unit` 中同名或相近测试文件覆盖选择器、迁移、错误码、关键组件渲染契约。新增 props 时同步更新这些测试。


#### `src/views/DataLoadingChat.tsx`（约 2013 行）

本文件导出 1 个符号：DataLoadingChat。
设计职责：作为该路径所表明的 UI 单元或工具模块，负责把 Redux 状态映射为可视控件，或把用户操作映射为 API/thunk。交互过程中的加载、空态、错误态必须可区分。若涉及 Agent 流，必须能被 AbortSignal 取消，并在 unmount 时停止写入已卸载组件。
状态约束：不得在组件本地复制一份“即将过期”的表全量行；需要行数据时通过 tableResolution/preview cache 物化。图表变更应走 dfActions 以便缩略图服务订阅。
测试对应：`tests/frontend/unit` 中同名或相近测试文件覆盖选择器、迁移、错误码、关键组件渲染契约。新增 props 时同步更新这些测试。


#### `src/views/DataSourceSidebar.tsx`（约 2544 行）

本文件导出 1 个符号：DataSourceSidebar。
设计职责：作为该路径所表明的 UI 单元或工具模块，负责把 Redux 状态映射为可视控件，或把用户操作映射为 API/thunk。交互过程中的加载、空态、错误态必须可区分。若涉及 Agent 流，必须能被 AbortSignal 取消，并在 unmount 时停止写入已卸载组件。
状态约束：不得在组件本地复制一份“即将过期”的表全量行；需要行数据时通过 tableResolution/preview cache 物化。图表变更应走 dfActions 以便缩略图服务订阅。
测试对应：`tests/frontend/unit` 中同名或相近测试文件覆盖选择器、迁移、错误码、关键组件渲染契约。新增 props 时同步更新这些测试。


#### `src/views/DataThread.tsx`（约 3405 行）

本文件导出 3 个符号：ThinkingStepsBanner, ThinkingBanner, DataThread。
设计职责：作为该路径所表明的 UI 单元或工具模块，负责把 Redux 状态映射为可视控件，或把用户操作映射为 API/thunk。交互过程中的加载、空态、错误态必须可区分。若涉及 Agent 流，必须能被 AbortSignal 取消，并在 unmount 时停止写入已卸载组件。
状态约束：不得在组件本地复制一份“即将过期”的表全量行；需要行数据时通过 tableResolution/preview cache 物化。图表变更应走 dfActions 以便缩略图服务订阅。
测试对应：`tests/frontend/unit` 中同名或相近测试文件覆盖选择器、迁移、错误码、关键组件渲染契约。新增 props 时同步更新这些测试。


#### `src/views/DataThreadCards.tsx`（约 384 行）

本文件导出 1 个符号：BuildTableCardProps。
设计职责：作为该路径所表明的 UI 单元或工具模块，负责把 Redux 状态映射为可视控件，或把用户操作映射为 API/thunk。交互过程中的加载、空态、错误态必须可区分。若涉及 Agent 流，必须能被 AbortSignal 取消，并在 unmount 时停止写入已卸载组件。
状态约束：不得在组件本地复制一份“即将过期”的表全量行；需要行数据时通过 tableResolution/preview cache 物化。图表变更应走 dfActions 以便缩略图服务订阅。
测试对应：`tests/frontend/unit` 中同名或相近测试文件覆盖选择器、迁移、错误码、关键组件渲染契约。新增 props 时同步更新这些测试。


#### `src/views/DataView.tsx`（约 373 行）

本文件导出 2 个符号：FreeDataViewProps, FreeDataViewFC。
设计职责：作为该路径所表明的 UI 单元或工具模块，负责把 Redux 状态映射为可视控件，或把用户操作映射为 API/thunk。交互过程中的加载、空态、错误态必须可区分。若涉及 Agent 流，必须能被 AbortSignal 取消，并在 unmount 时停止写入已卸载组件。
状态约束：不得在组件本地复制一份“即将过期”的表全量行；需要行数据时通过 tableResolution/preview cache 物化。图表变更应走 dfActions 以便缩略图服务订阅。
测试对应：`tests/frontend/unit` 中同名或相近测试文件覆盖选择器、迁移、错误码、关键组件渲染契约。新增 props 时同步更新这些测试。


#### `src/views/EncodingBox.tsx`（约 766 行）

本文件导出 4 个符号：LittleConceptCardProps, LittleConceptCard, EncodingBoxProps, EncodingBox。
设计职责：作为该路径所表明的 UI 单元或工具模块，负责把 Redux 状态映射为可视控件，或把用户操作映射为 API/thunk。交互过程中的加载、空态、错误态必须可区分。若涉及 Agent 流，必须能被 AbortSignal 取消，并在 unmount 时停止写入已卸载组件。
状态约束：不得在组件本地复制一份“即将过期”的表全量行；需要行数据时通过 tableResolution/preview cache 物化。图表变更应走 dfActions 以便缩略图服务订阅。
测试对应：`tests/frontend/unit` 中同名或相近测试文件覆盖选择器、迁移、错误码、关键组件渲染契约。新增 props 时同步更新这些测试。


#### `src/views/EncodingShelfCard.tsx`（约 1179 行）

本文件导出 7 个符号：ConfigSlider, EncodingShelfCardProps, renderTextWithEmphasis, TriggerCard, StylePreset, STYLE_PRESETS, EncodingShelfCard。
设计职责：作为该路径所表明的 UI 单元或工具模块，负责把 Redux 状态映射为可视控件，或把用户操作映射为 API/thunk。交互过程中的加载、空态、错误态必须可区分。若涉及 Agent 流，必须能被 AbortSignal 取消，并在 unmount 时停止写入已卸载组件。
状态约束：不得在组件本地复制一份“即将过期”的表全量行；需要行数据时通过 tableResolution/preview cache 物化。图表变更应走 dfActions 以便缩略图服务订阅。
测试对应：`tests/frontend/unit` 中同名或相近测试文件覆盖选择器、迁移、错误码、关键组件渲染契约。新增 props 时同步更新这些测试。


#### `src/views/EncodingShelfThread.tsx`（约 103 行）

本文件导出 2 个符号：EncodingShelfThreadProps, EncodingShelfThread。
设计职责：作为该路径所表明的 UI 单元或工具模块，负责把 Redux 状态映射为可视控件，或把用户操作映射为 API/thunk。交互过程中的加载、空态、错误态必须可区分。若涉及 Agent 流，必须能被 AbortSignal 取消，并在 unmount 时停止写入已卸载组件。
状态约束：不得在组件本地复制一份“即将过期”的表全量行；需要行数据时通过 tableResolution/preview cache 物化。图表变更应走 dfActions 以便缩略图服务订阅。
测试对应：`tests/frontend/unit` 中同名或相近测试文件覆盖选择器、迁移、错误码、关键组件渲染契约。新增 props 时同步更新这些测试。


#### `src/views/ExampleSessions.tsx`（约 195 行）

本文件导出 4 个符号：ExampleSession, fetchExampleSessions, exampleSessions, ExampleSessionCard。
设计职责：作为该路径所表明的 UI 单元或工具模块，负责把 Redux 状态映射为可视控件，或把用户操作映射为 API/thunk。交互过程中的加载、空态、错误态必须可区分。若涉及 Agent 流，必须能被 AbortSignal 取消，并在 unmount 时停止写入已卸载组件。
状态约束：不得在组件本地复制一份“即将过期”的表全量行；需要行数据时通过 tableResolution/preview cache 物化。图表变更应走 dfActions 以便缩略图服务订阅。
测试对应：`tests/frontend/unit` 中同名或相近测试文件覆盖选择器、迁移、错误码、关键组件渲染契约。新增 props 时同步更新这些测试。


#### `src/views/ExplComponents.tsx`（约 394 行）

本文件导出 5 个符号：ConceptExplanationItem, ConceptExplCardsProps, ConceptExplCards, extractConceptExplanations, CodeExplanationCard。
设计职责：作为该路径所表明的 UI 单元或工具模块，负责把 Redux 状态映射为可视控件，或把用户操作映射为 API/thunk。交互过程中的加载、空态、错误态必须可区分。若涉及 Agent 流，必须能被 AbortSignal 取消，并在 unmount 时停止写入已卸载组件。
状态约束：不得在组件本地复制一份“即将过期”的表全量行；需要行数据时通过 tableResolution/preview cache 物化。图表变更应走 dfActions 以便缩略图服务订阅。
测试对应：`tests/frontend/unit` 中同名或相近测试文件覆盖选择器、迁移、错误码、关键组件渲染契约。新增 props 时同步更新这些测试。


#### `src/views/InteractionEntryCard.tsx`（约 756 行）

本文件导出 11 个符号：getStepIconComponent, PlanStepsView, CompactMarkdown, renderFieldHighlights, stripFieldMarkers, InteractionEntryCardProps, InteractionEntryCard, ResolvedConversationCardProps, ResolvedConversationCard, getEntryGutterIcon, getDefaultGutterIcon。
设计职责：作为该路径所表明的 UI 单元或工具模块，负责把 Redux 状态映射为可视控件，或把用户操作映射为 API/thunk。交互过程中的加载、空态、错误态必须可区分。若涉及 Agent 流，必须能被 AbortSignal 取消，并在 unmount 时停止写入已卸载组件。
状态约束：不得在组件本地复制一份“即将过期”的表全量行；需要行数据时通过 tableResolution/preview cache 物化。图表变更应走 dfActions 以便缩略图服务订阅。
测试对应：`tests/frontend/unit` 中同名或相近测试文件覆盖选择器、迁移、错误码、关键组件渲染契约。新增 props 时同步更新这些测试。


#### `src/views/KnowledgePanel.tsx`（约 644 行）

本文件导出 1 个符号：KnowledgePanel。
设计职责：作为该路径所表明的 UI 单元或工具模块，负责把 Redux 状态映射为可视控件，或把用户操作映射为 API/thunk。交互过程中的加载、空态、错误态必须可区分。若涉及 Agent 流，必须能被 AbortSignal 取消，并在 unmount 时停止写入已卸载组件。
状态约束：不得在组件本地复制一份“即将过期”的表全量行；需要行数据时通过 tableResolution/preview cache 物化。图表变更应走 dfActions 以便缩略图服务订阅。
测试对应：`tests/frontend/unit` 中同名或相近测试文件覆盖选择器、迁移、错误码、关键组件渲染契约。新增 props 时同步更新这些测试。


#### `src/views/LocalInstallUpgradePanel.tsx`（约 299 行）

本文件导出 1 个符号：LocalInstallUpgradePanel。
设计职责：作为该路径所表明的 UI 单元或工具模块，负责把 Redux 状态映射为可视控件，或把用户操作映射为 API/thunk。交互过程中的加载、空态、错误态必须可区分。若涉及 Agent 流，必须能被 AbortSignal 取消，并在 unmount 时停止写入已卸载组件。
状态约束：不得在组件本地复制一份“即将过期”的表全量行；需要行数据时通过 tableResolution/preview cache 物化。图表变更应走 dfActions 以便缩略图服务订阅。
测试对应：`tests/frontend/unit` 中同名或相近测试文件覆盖选择器、迁移、错误码、关键组件渲染契约。新增 props 时同步更新这些测试。


#### `src/views/LogViewerDialog.tsx`（约 325 行）

本文件导出 1 个符号：LogViewerDialog。
设计职责：作为该路径所表明的 UI 单元或工具模块，负责把 Redux 状态映射为可视控件，或把用户操作映射为 API/thunk。交互过程中的加载、空态、错误态必须可区分。若涉及 Agent 流，必须能被 AbortSignal 取消，并在 unmount 时停止写入已卸载组件。
状态约束：不得在组件本地复制一份“即将过期”的表全量行；需要行数据时通过 tableResolution/preview cache 物化。图表变更应走 dfActions 以便缩略图服务订阅。
测试对应：`tests/frontend/unit` 中同名或相近测试文件覆盖选择器、迁移、错误码、关键组件渲染契约。新增 props 时同步更新这些测试。


#### `src/views/MessageSnackbar.tsx`（约 353 行）

本文件导出 2 个符号：Message, MessageSnackbar。
设计职责：作为该路径所表明的 UI 单元或工具模块，负责把 Redux 状态映射为可视控件，或把用户操作映射为 API/thunk。交互过程中的加载、空态、错误态必须可区分。若涉及 Agent 流，必须能被 AbortSignal 取消，并在 unmount 时停止写入已卸载组件。
状态约束：不得在组件本地复制一份“即将过期”的表全量行；需要行数据时通过 tableResolution/preview cache 物化。图表变更应走 dfActions 以便缩略图服务订阅。
测试对应：`tests/frontend/unit` 中同名或相近测试文件覆盖选择器、迁移、错误码、关键组件渲染契约。新增 props 时同步更新这些测试。


#### `src/views/ModelSelectionDialog.tsx`（约 821 行）

本文件导出 1 个符号：ModelSelectionButton。
设计职责：作为该路径所表明的 UI 单元或工具模块，负责把 Redux 状态映射为可视控件，或把用户操作映射为 API/thunk。交互过程中的加载、空态、错误态必须可区分。若涉及 Agent 流，必须能被 AbortSignal 取消，并在 unmount 时停止写入已卸载组件。
状态约束：不得在组件本地复制一份“即将过期”的表全量行；需要行数据时通过 tableResolution/preview cache 物化。图表变更应走 dfActions 以便缩略图服务订阅。
测试对应：`tests/frontend/unit` 中同名或相近测试文件覆盖选择器、迁移、错误码、关键组件渲染契约。新增 props 时同步更新这些测试。


#### `src/views/MultiTablePreview.tsx`（约 234 行）

本文件导出 2 个符号：MultiTablePreviewProps, MultiTablePreview。
设计职责：作为该路径所表明的 UI 单元或工具模块，负责把 Redux 状态映射为可视控件，或把用户操作映射为 API/thunk。交互过程中的加载、空态、错误态必须可区分。若涉及 Agent 流，必须能被 AbortSignal 取消，并在 unmount 时停止写入已卸载组件。
状态约束：不得在组件本地复制一份“即将过期”的表全量行；需要行数据时通过 tableResolution/preview cache 物化。图表变更应走 dfActions 以便缩略图服务订阅。
测试对应：`tests/frontend/unit` 中同名或相近测试文件覆盖选择器、迁移、错误码、关键组件渲染契约。新增 props 时同步更新这些测试。


#### `src/views/OperatorCard.tsx`（约 65 行）

本文件导出 2 个符号：OperatorCardProp, OperatorCard。
设计职责：作为该路径所表明的 UI 单元或工具模块，负责把 Redux 状态映射为可视控件，或把用户操作映射为 API/thunk。交互过程中的加载、空态、错误态必须可区分。若涉及 Agent 流，必须能被 AbortSignal 取消，并在 unmount 时停止写入已卸载组件。
状态约束：不得在组件本地复制一份“即将过期”的表全量行；需要行数据时通过 tableResolution/preview cache 物化。图表变更应走 dfActions 以便缩略图服务订阅。
测试对应：`tests/frontend/unit` 中同名或相近测试文件覆盖选择器、迁移、错误码、关键组件渲染契约。新增 props 时同步更新这些测试。


#### `src/views/ReactTable.tsx`（约 212 行）

本文件导出 2 个符号：ColumnDef, CustomReactTable。
设计职责：作为该路径所表明的 UI 单元或工具模块，负责把 Redux 状态映射为可视控件，或把用户操作映射为 API/thunk。交互过程中的加载、空态、错误态必须可区分。若涉及 Agent 流，必须能被 AbortSignal 取消，并在 unmount 时停止写入已卸载组件。
状态约束：不得在组件本地复制一份“即将过期”的表全量行；需要行数据时通过 tableResolution/preview cache 物化。图表变更应走 dfActions 以便缩略图服务订阅。
测试对应：`tests/frontend/unit` 中同名或相近测试文件覆盖选择器、迁移、错误码、关键组件渲染契约。新增 props 时同步更新这些测试。


#### `src/views/RefreshDataDialog.tsx`（约 556 行）

本文件导出 2 个符号：RefreshDataDialogProps, RefreshDataDialog。
设计职责：作为该路径所表明的 UI 单元或工具模块，负责把 Redux 状态映射为可视控件，或把用户操作映射为 API/thunk。交互过程中的加载、空态、错误态必须可区分。若涉及 Agent 流，必须能被 AbortSignal 取消，并在 unmount 时停止写入已卸载组件。
状态约束：不得在组件本地复制一份“即将过期”的表全量行；需要行数据时通过 tableResolution/preview cache 物化。图表变更应走 dfActions 以便缩略图服务订阅。
测试对应：`tests/frontend/unit` 中同名或相近测试文件覆盖选择器、迁移、错误码、关键组件渲染契约。新增 props 时同步更新这些测试。


#### `src/views/ReportView.tsx`（约 780 行）

本文件导出 1 个符号：ReportView。
设计职责：作为该路径所表明的 UI 单元或工具模块，负责把 Redux 状态映射为可视控件，或把用户操作映射为 API/thunk。交互过程中的加载、空态、错误态必须可区分。若涉及 Agent 流，必须能被 AbortSignal 取消，并在 unmount 时停止写入已卸载组件。
状态约束：不得在组件本地复制一份“即将过期”的表全量行；需要行数据时通过 tableResolution/preview cache 物化。图表变更应走 dfActions 以便缩略图服务订阅。
测试对应：`tests/frontend/unit` 中同名或相近测试文件覆盖选择器、迁移、错误码、关键组件渲染契约。新增 props 时同步更新这些测试。


#### `src/views/SelectableDataGrid.tsx`（约 849 行）

本文件导出 2 个符号：ColumnDef, SelectableDataGrid。
设计职责：作为该路径所表明的 UI 单元或工具模块，负责把 Redux 状态映射为可视控件，或把用户操作映射为 API/thunk。交互过程中的加载、空态、错误态必须可区分。若涉及 Agent 流，必须能被 AbortSignal 取消，并在 unmount 时停止写入已卸载组件。
状态约束：不得在组件本地复制一份“即将过期”的表全量行；需要行数据时通过 tableResolution/preview cache 物化。图表变更应走 dfActions 以便缩略图服务订阅。
测试对应：`tests/frontend/unit` 中同名或相近测试文件覆盖选择器、迁移、错误码、关键组件渲染契约。新增 props 时同步更新这些测试。


#### `src/views/SessionDistill.tsx`（约 734 行）

本文件导出 7 个符号：SessionThread, BuildSessionResult, findSessionWorkflow, collectSessionThreads, buildSessionWorkflowContext, SessionDistillDialogProps, SessionDistillDialog。
设计职责：作为该路径所表明的 UI 单元或工具模块，负责把 Redux 状态映射为可视控件，或把用户操作映射为 API/thunk。交互过程中的加载、空态、错误态必须可区分。若涉及 Agent 流，必须能被 AbortSignal 取消，并在 unmount 时停止写入已卸载组件。
状态约束：不得在组件本地复制一份“即将过期”的表全量行；需要行数据时通过 tableResolution/preview cache 物化。图表变更应走 dfActions 以便缩略图服务订阅。
测试对应：`tests/frontend/unit` 中同名或相近测试文件覆盖选择器、迁移、错误码、关键组件渲染契约。新增 props 时同步更新这些测试。


#### `src/views/SimpleChartRecBox.tsx`（约 2629 行）

本文件导出 1 个符号：SimpleChartRecBox。
设计职责：作为该路径所表明的 UI 单元或工具模块，负责把 Redux 状态映射为可视控件，或把用户操作映射为 API/thunk。交互过程中的加载、空态、错误态必须可区分。若涉及 Agent 流，必须能被 AbortSignal 取消，并在 unmount 时停止写入已卸载组件。
状态约束：不得在组件本地复制一份“即将过期”的表全量行；需要行数据时通过 tableResolution/preview cache 物化。图表变更应走 dfActions 以便缩略图服务订阅。
测试对应：`tests/frontend/unit` 中同名或相近测试文件覆盖选择器、迁移、错误码、关键组件渲染契约。新增 props 时同步更新这些测试。


#### `src/views/SourceTableShelf.tsx`（约 982 行）

本文件导出 2 个符号：SHELF_VISIBLE_LIMIT, SourceTableShelf。
设计职责：作为该路径所表明的 UI 单元或工具模块，负责把 Redux 状态映射为可视控件，或把用户操作映射为 API/thunk。交互过程中的加载、空态、错误态必须可区分。若涉及 Agent 流，必须能被 AbortSignal 取消，并在 unmount 时停止写入已卸载组件。
状态约束：不得在组件本地复制一份“即将过期”的表全量行；需要行数据时通过 tableResolution/preview cache 物化。图表变更应走 dfActions 以便缩略图服务订阅。
测试对应：`tests/frontend/unit` 中同名或相近测试文件覆盖选择器、迁移、错误码、关键组件渲染契约。新增 props 时同步更新这些测试。


#### `src/views/TestPanel.tsx`（约 89 行）

本文件导出 3 个符号：TestPanelProps, TestPanelState, TestPanel。
设计职责：作为该路径所表明的 UI 单元或工具模块，负责把 Redux 状态映射为可视控件，或把用户操作映射为 API/thunk。交互过程中的加载、空态、错误态必须可区分。若涉及 Agent 流，必须能被 AbortSignal 取消，并在 unmount 时停止写入已卸载组件。
状态约束：不得在组件本地复制一份“即将过期”的表全量行；需要行数据时通过 tableResolution/preview cache 物化。图表变更应走 dfActions 以便缩略图服务订阅。
测试对应：`tests/frontend/unit` 中同名或相近测试文件覆盖选择器、迁移、错误码、关键组件渲染契约。新增 props 时同步更新这些测试。


#### `src/views/TiptapReportEditor.tsx`（约 791 行）

本文件导出 3 个符号：TiptapReportEditorProps, InspectStep, TiptapReportEditor。
设计职责：作为该路径所表明的 UI 单元或工具模块，负责把 Redux 状态映射为可视控件，或把用户操作映射为 API/thunk。交互过程中的加载、空态、错误态必须可区分。若涉及 Agent 流，必须能被 AbortSignal 取消，并在 unmount 时停止写入已卸载组件。
状态约束：不得在组件本地复制一份“即将过期”的表全量行；需要行数据时通过 tableResolution/preview cache 物化。图表变更应走 dfActions 以便缩略图服务订阅。
测试对应：`tests/frontend/unit` 中同名或相近测试文件覆盖选择器、迁移、错误码、关键组件渲染契约。新增 props 时同步更新这些测试。


#### `src/views/UnifiedDataUploadDialog.tsx`（约 2639 行）

本文件导出 7 个符号：UploadTabType, LocalFolderPanel, DataLoadMenuProps, DataLoadMenu, UnifiedDataUploadDialogProps, UnifiedDataUploadDialog, type ConnectorInstance。
设计职责：作为该路径所表明的 UI 单元或工具模块，负责把 Redux 状态映射为可视控件，或把用户操作映射为 API/thunk。交互过程中的加载、空态、错误态必须可区分。若涉及 Agent 流，必须能被 AbortSignal 取消，并在 unmount 时停止写入已卸载组件。
状态约束：不得在组件本地复制一份“即将过期”的表全量行；需要行数据时通过 tableResolution/preview cache 物化。图表变更应走 dfActions 以便缩略图服务订阅。
测试对应：`tests/frontend/unit` 中同名或相近测试文件覆盖选择器、迁移、错误码、关键组件渲染契约。新增 props 时同步更新这些测试。


#### `src/views/ViewUtils.tsx`（约 187 行）

本文件导出 6 个符号：DENSE_MENU_SLOT_PROPS, groupConceptItems, getIconFromType, formatCellValue, getColumnAlign, getIconFromDtype。
设计职责：作为该路径所表明的 UI 单元或工具模块，负责把 Redux 状态映射为可视控件，或把用户操作映射为 API/thunk。交互过程中的加载、空态、错误态必须可区分。若涉及 Agent 流，必须能被 AbortSignal 取消，并在 unmount 时停止写入已卸载组件。
状态约束：不得在组件本地复制一份“即将过期”的表全量行；需要行数据时通过 tableResolution/preview cache 物化。图表变更应走 dfActions 以便缩略图服务订阅。
测试对应：`tests/frontend/unit` 中同名或相近测试文件覆盖选择器、迁移、错误码、关键组件渲染契约。新增 props 时同步更新这些测试。


#### `src/views/VisualizationView.tsx`（约 1968 行）

本文件导出 7 个符号：VisPanelProps, VisPanelState, ChartEditorFC, VisualizationViewFC, generateChartSkeleton, getDataTable, checkChartAvailability。
设计职责：作为该路径所表明的 UI 单元或工具模块，负责把 Redux 状态映射为可视控件，或把用户操作映射为 API/thunk。交互过程中的加载、空态、错误态必须可区分。若涉及 Agent 流，必须能被 AbortSignal 取消，并在 unmount 时停止写入已卸载组件。
状态约束：不得在组件本地复制一份“即将过期”的表全量行；需要行数据时通过 tableResolution/preview cache 物化。图表变更应走 dfActions 以便缩略图服务订阅。
测试对应：`tests/frontend/unit` 中同名或相近测试文件覆盖选择器、迁移、错误码、关键组件渲染契约。新增 props 时同步更新这些测试。


#### `src/views/dataLoadingSuggestions.ts`（约 210 行）

本文件导出 6 个符号：DataLoadingSuggestion, SuggestionPayload, BuildSuggestionsArgs, buildDataLoadingSuggestions, DataLoadingQuickAction, buildDataLoadingQuickActions。
设计职责：作为该路径所表明的 UI 单元或工具模块，负责把 Redux 状态映射为可视控件，或把用户操作映射为 API/thunk。交互过程中的加载、空态、错误态必须可区分。若涉及 Agent 流，必须能被 AbortSignal 取消，并在 unmount 时停止写入已卸载组件。
状态约束：不得在组件本地复制一份“即将过期”的表全量行；需要行数据时通过 tableResolution/preview cache 物化。图表变更应走 dfActions 以便缩略图服务订阅。
测试对应：`tests/frontend/unit` 中同名或相近测试文件覆盖选择器、迁移、错误码、关键组件渲染契约。新增 props 时同步更新这些测试。


#### `src/views/threadLayout.ts`（约 56 行）

本文件导出 7 个符号：CARD_WIDTH, CARD_GAP, PANEL_PADDING, MAX_THREAD_COLUMNS, COLUMN_FIT_TOLERANCE, threadPaneWidth, fittableThreadColumns。
设计职责：作为该路径所表明的 UI 单元或工具模块，负责把 Redux 状态映射为可视控件，或把用户操作映射为 API/thunk。交互过程中的加载、空态、错误态必须可区分。若涉及 Agent 流，必须能被 AbortSignal 取消，并在 unmount 时停止写入已卸载组件。
状态约束：不得在组件本地复制一份“即将过期”的表全量行；需要行数据时通过 tableResolution/preview cache 物化。图表变更应走 dfActions 以便缩略图服务订阅。
测试对应：`tests/frontend/unit` 中同名或相近测试文件覆盖选择器、迁移、错误码、关键组件渲染契约。新增 props 时同步更新这些测试。


#### `src/views/workflowContext.ts`（约 319 行）

本文件导出 7 个符号：MESSAGE_CONTENT_LIMIT, TOOL_ARGS_LIMIT, SAMPLE_ROW_COUNT, TOOL_USES_CODE_FONT, isLeafDerivedTable, buildLeafEvents, buildDistillModelConfig。
设计职责：作为该路径所表明的 UI 单元或工具模块，负责把 Redux 状态映射为可视控件，或把用户操作映射为 API/thunk。交互过程中的加载、空态、错误态必须可区分。若涉及 Agent 流，必须能被 AbortSignal 取消，并在 unmount 时停止写入已卸载组件。
状态约束：不得在组件本地复制一份“即将过期”的表全量行；需要行数据时通过 tableResolution/preview cache 物化。图表变更应走 dfActions 以便缩略图服务订阅。
测试对应：`tests/frontend/unit` 中同名或相近测试文件覆盖选择器、迁移、错误码、关键组件渲染契约。新增 props 时同步更新这些测试。


## 5A. 核心算法详细说明（实现级）

### A. AnalystAgent.run

输入：input_tables、user_question、focused_thread、other_threads、trajectory、scratch_files、charts 等。
输出：NDJSON 事件生成器。

步骤：

1. `_build_system_prompt`：身份、tools vs actions 合同、技能目录、规则、语言指令。core SKILL.md 追加在后。
2. `_build_initial_messages`：轻量表上下文 + 焦点线程叙事 + 外围线程摘要 + 用户问题 + 图像。
3. 若 resume：`_rehydrate_loaded_skills` 扫描消息开头 `[SKILL LOADED: name]`。
4. `_tool_loop`：LiteLLM 流式 completion，合并增量 tool_calls。
5. 分区：registry.action_names() 中的为 action，其余为 inspection（含 load_skill）。
6. inspection：`handle_tool` 或内置 execute_python / inspect_source_data / load_skill。
7. action：`_dispatch_skill_action` → skill.handle_action 生成器 → `_route_skill_events`。
8. visualize 内部：`_run_visualize_code` 沙箱跑转换 → `create_chart_spec`/`assemble` → register_run_chart。
9. 停止条件：无 tool_call；或 ask_user interact；或预算耗尽。

复杂度：受 max_iterations 与每次 LLM 延迟主导，本地代码执行不是瓶颈（warm worker ~1ms）。

### B. DataOperationExecutor

按 plan.steps 顺序：resolve_live_loader(source_id) → fetch_data_as_arrow(query) → write_parquet → 记录 provenance。
单步失败记入 FailedOperationStep，整体状态 partially_loaded 或 failed。表名经 table_names.sanitize 保证 DuckDB 可引用。

### C. Catalog 搜索

对每个 CatalogNode 拼接 path 标签、列名、描述；CJK 用字符级 bigram + 空白分词；打分后截断。
未命中缓存则按策略决定是否阻塞同步。

### D. 语义类型推断

规则优先于统计：列名匹配（year、lat、country）、值域检测（时间戳范围、日期正则）、基数（高基数倾向 identifier）。
结果写入 tableSemantics，供编码通道推荐与 VL type 选择。ordinal 可触发 SortDataAgent 补序。

### E. Vega 后处理族

create_vl_plots 中每个 `_post_process_*` 针对一类图的 Flint/Vega 缺陷做补丁：例如 waterfall 需要中间计算列，
radar 需要极坐标变换，density 需要 bin。算法保持 spec 可序列化，避免嵌入函数。

### F. 编码探测

FileManager 对文本样本尝试 utf-8-sig、utf-8、gbk、gb18030、shift_jis、latin1、cp1251 等，
用可打印比例与解码异常次数打分。结果全部转 UTF-8 再解析。有 test_file_manager_encoding 覆盖。

### G. Token 刷新

TokenStore：若 access 过期且有 refresh，调用 IdP token endpoint；失败则标记服务断开而不清 SSO，
以便用户重新 delegated login。并发请求靠 session 锁避免重复刷新。

### H. 工作区 LRU（ephemeral）

定期扫描 last_access，超过 TTL 或总字节超 MAX 则删最久未用。被删 id 再访问返回 WORKSPACE_EXPIRED，
前端应提示“会话已回收”而不是 TABLE_NOT_FOUND。

### I. 图表意图分类

intentClassifier / SimpleAgents.classify_chart_intent：区分“改样式”与“改数据/编码”。
前者走 restyle，后者走 Analyst visualize，避免用错工具造成编码被清空。

### J. 内容哈希

table_hash / computeContentHash：对列名+采样行规范化后哈希，用于刷新检测与缓存键，避免无变化重渲染。


## 6. 功能规格说明书（按用户可感知功能）

### 6.1 工作区与会话

功能：创建、重命名、加载、删除、导出/导入 zip、匿名身份迁移。
后端：`routes/sessions.py`。前端：WorkspaceMenu、useAutoSave、workspaceService。
失败：磁盘满 → STORAGE_FULL；过期 ephemeral → WORKSPACE_EXPIRED。

### 6.2 本地文件上传与解析

支持 CSV/TSV/JSON/Excel/Parquet。FileManager 做编码探测（GBK 等转 UTF-8）、表名消毒。
parse-file 给预览，create-table 落 Parquet。同名文件有 replace source 语义（有专门路由测试）。

### 6.3 连接器

管理型（env/YAML）与用户创建型（user::identity::id）。连接、断开、目录树、搜索、预览、导入、刷新、列值枚举。
敏感字段不回显。多用户关闭 local_folder。

### 6.4 数据加载 Agent

独立 NDJSON 对话，工具含 list/find/describe/probe 与连接器描述。可提出 LoadPlan 或 DataOperation。
用户确认后才真正 import。这避免 Agent 在未授权范围拉走整库。

### 6.5 Analyst 分析

自然语言提问、附加图片与 scratch 文件、可视化、澄清、写报告。技能按需加载。
max_iterations / max_repair_attempts 防止死循环。代码修复循环只针对可视化执行失败。

### 6.6 图表编码与风格

拖拽字段到通道；Auto 推荐用 flint vlRecommendEncodings；风格变体由 ChartRestyleAgent 生成可控 VL 补丁；
主题来自 flint THEME_PRESETS。

### 6.7 报告

Tiptap 编辑，嵌入图表引用，流式生成时展示 inspectionSteps。Chartifact 导出便携 Markdown。

### 6.8 知识库

规则 always_apply 或按检索注入 Analyst system prompt。工作流可由 SessionDistill 从当前线程提炼。
data-memory 记录“这个连接器里销售表叫什么”这类长期记忆。

### 6.9 模型管理

全局模型只读；用户模型可增删改测。DISABLE_CUSTOM_MODELS 时隐藏自定义。
api_base 受 DF_ALLOWED_API_BASES 约束。

### 6.10 认证

匿名 browser、本机 local、OIDC 前端 PKCE、OIDC 后端 confidential、GitHub OAuth、Azure EasyAuth。
Kusto 另有委托登录弹窗。

### 6.11 刷新与实时

表可配置刷新间隔；demo-stream 提供公共 CSV/天气/yfinance 示例。派生表可按签名代码重跑。

### 6.12 日志与诊断

本地模式可打开 LogViewerDialog。ReasoningLogger 按会话写 JSONL，含 TTL 清理与脱敏。
AgentDiagnostics 在开发态附加 payload。

## 7. 质量属性设计

- **安全性**：见 security 模块与测试（229 个后端安全用例量级）。
- **可维护性**：dev-guides 编号文档固化跨切约定。
- **国际化**：en/zh 双文件同步；Agent 输出语言随 UI。
- **性能**：沙箱预热 ~1ms；目录虚拟化；图表缓存；Arrow 列投影。
- **可观测性**：滚动日志 + 推理日志；统一 request 错误码。
- **可移植性**：Python≥3.11，Wheel 含前端 dist；桌面 PyInstaller。


## 7. 测试用例设计总册

测试策略：后端 pytest + 前端 vitest。根 conftest 的 `_isolate_env` 在每个用例前后恢复 os.environ，
避免开发者 .env 泄漏。Docker 数据库用例在 tests/database-dockers，默认 CI 可不跑依赖服务的部分。
下列清单覆盖仓库内全部自动化测试文件及其用例名称。每个用例都应能从名称推断断言意图；
若不能，视为命名缺陷，应在后续补测时改名而不是删测。

### 7.1 后端 pytest

#### `tests/backend/agents/test_agent_diagnostics.py`（295 行，24 个 test_* ，类 6）

测试目标：验证 `backend/agents/agent_diagnostics.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
测试类：TestSharedFields、TestForError、TestForResponse、TestForJsonOnly、TestSchemaCompatibility、TestMultipleInstances。
- **`test_for_error_has_shared_keys`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。
- **`test_for_response_has_shared_keys`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_for_json_only_has_shared_keys`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_agent_name_propagated`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_timestamp_is_iso8601`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_model_info_preserved`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_prompt_components_complete`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_llm_request_message_count`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_contains_error_field`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。
- **`test_no_parsing_or_execution_sections`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_top_level_sections`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_llm_response_fields`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_parsing_fields`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_execution_fields`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_execution_error_fields_present_when_failed`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。
- **`test_performance_rounding`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_has_llm_response_and_performance`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_no_parsing_or_execution_sections`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_defaults_when_no_kwargs`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_for_response_key_set`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_for_error_key_set`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。
- **`test_for_json_only_key_set`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_different_agent_names`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_prompt_isolation`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。

#### `tests/backend/agents/test_agent_language.py`（235 行，38 个 test_* ，类 6）

测试目标：验证 `backend/agents/agent_language.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
测试类：TestLanguageRegistry、TestBuildLanguageInstructionEnglish、TestBuildLanguageInstructionFull、TestBuildLanguageInstructionCompact、TestBuildLanguageInstructionUnknown、TestInjectLanguageInstruction。
- **`test_english_in_registry`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_common_languages_present`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_display_names_are_non_empty_strings`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_default_language_is_en`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_extra_rules_values_are_strings`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_extra_rules_codes_are_subset_of_display_names`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_english_returns_empty_string`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_english_compact_also_returns_empty`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_empty_string_defaults_to_en_returns_empty`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_none_coerced_to_default_returns_empty`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_whitespace_only_returns_empty`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_case_insensitive_en`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_non_english_returns_non_empty`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_result_contains_language_marker`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_result_contains_display_name`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_full_mode_is_default`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_full_mode_mentions_user_visible_fields`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_full_mode_mentions_internal_fields`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_zh_extra_rules_injected`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_ja_extra_rules_injected`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_lang_without_extra_rules_has_no_extra_block`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_compact_returns_non_empty_for_non_english`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_compact_contains_language_marker`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_compact_shorter_than_full`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_compact_mentions_display_instruction`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_compact_instructs_english_for_code`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_compact_zh_extra_rules_present`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_unknown_code_returns_non_empty`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_unknown_code_uses_raw_code_as_display_name`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_empty_instruction_is_noop`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_non_empty_instruction_appended_by_default`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_instruction_appended_after_base`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_marker_found_inserts_before_marker`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_marker_not_found_falls_back_to_append`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_marker_at_start_of_string_not_inserted`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_original_prompt_is_preserved_in_output`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_marker_insertion_preserves_rest_of_prompt`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_round_trip_with_build_and_inject`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。

#### `tests/backend/agents/test_agent_utils_sql_table_names.py`（49 行，4 个 test_* ，类 0）

测试目标：验证 `backend/agents/agent_utils_sql_table_names.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
- **`test_sql_sanitize_preserves_safe_ascii_symbols`**：断言日志或响应中的密钥、URL 口令、堆栈被脱敏。
- **`test_sql_sanitize_replaces_spaces_and_hyphens_for_ascii_names`**：断言日志或响应中的密钥、URL 口令、堆栈被脱敏。
- **`test_sql_sanitize_preserves_unicode_identifier`**：断言日志或响应中的密钥、URL 口令、堆栈被脱敏。
- **`test_create_duckdb_views_supports_unicode_view_names`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。

#### `tests/backend/agents/test_analyst_connector_skill.py`（126 行，4 个 test_* ，类 0）

测试目标：验证 `backend/agents/analyst_connector_skill.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
- **`test_list_and_describe_connectors`**：断言连接器 CRUD、可见性隔离或表单字段。
- **`test_propose_connection_requires_listing_first`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_propose_connection_emits_prefilled_canvas_form_without_echo`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_local_folder_is_available_when_registered`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。

#### `tests/backend/agents/test_analyst_scratch_files.py`（59 行，3 个 test_* ，类 1）

测试目标：验证 `backend/agents/analyst_scratch_files.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
测试类：TestScratchFileInjection。
- **`test_scratch_note_injected`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_no_note_without_files`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_file_bytes_not_inlined`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。

#### `tests/backend/agents/test_client_image_strip.py`（238 行，17 个 test_* ，类 5）

测试目标：验证 `backend/agents/client_image_strip.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
测试类：TestStripImageBlocks、TestStripImagesFromMessages、TestIsImageDeserializeError、TestGetCompletionRetryLitellm、TestGetCompletionRetryOpenAI。
- **`test_removes_image_url_items`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_keeps_all_when_no_images`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_returns_empty_list_when_all_images`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_passthrough_non_list`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_preserves_non_dict_items`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_strips_images_from_multimodal_message`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_does_not_mutate_original`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_handles_plain_text_messages`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_preserves_non_dict_messages`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_detects_known_patterns`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_ignores_unrelated_errors`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。
- **`test_case_insensitive`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_retries_on_image_error`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。
- **`test_raises_unrelated_error`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。
- **`test_no_retry_on_success`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_retries_on_image_error`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。
- **`test_raises_unrelated_error`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。

#### `tests/backend/agents/test_client_utils.py`（499 行，57 个 test_* ，类 12）

测试目标：验证 `backend/agents/client_utils.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
测试类：TestModelNamePrefixing、TestOllamaApiBaseNormalisation、TestAzureCredentialSelection、TestStripImageBlocks、TestStripImagesFromMessages、TestIsImageDeserializeError、TestMessagesContainImages、TestFromConfig、TestExtractJsonObjects、TestMatchToolFromObj、TestSalvageToolCallsFromContent、TestMatchToolWireFormats。
- **`test_gemini_prefix_added_when_missing`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_gemini_prefix_not_doubled`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_anthropic_prefix_added_when_missing`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_anthropic_prefix_not_doubled`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_ollama_prefix_added_when_missing`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_ollama_prefix_not_doubled`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_openai_model_prefixed`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_trailing_slash_stripped`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_trailing_api_stripped`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_trailing_api_slash_stripped`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_non_api_suffix_preserved`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_default_base_when_none`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_desktop_keyless_model_uses_azure_cli_credential`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_blank_api_version_is_not_defaulted`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_explicit_api_version_is_preserved`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_api_base_is_required`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_string_content_unchanged`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_image_url_blocks_removed`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_non_image_blocks_preserved`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_mixed_list_with_non_dict_preserved`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_all_images_removed_returns_empty_list`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_system_message_unchanged`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_image_blocks_removed_from_user_message`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_text_blocks_preserved_in_user_message`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_original_messages_not_mutated`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_non_dict_messages_preserved`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_image_url_expected_text_detected`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_unknown_variant_image_url_detected`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_unrelated_error_not_detected`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。
- **`test_empty_string_not_detected`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_partial_match_image_url_without_expected`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_upstream_failure_with_images_detected`**：断言 NDJSON 事件类型、预检错误或警告刷新。
- **`test_upstream_failure_without_images_not_detected`**：断言 NDJSON 事件类型、预检错误或警告刷新。
- **`test_unsupported_image_message_with_images_detected`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_multimodal_message_detected`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_text_only_messages_not_detected`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_creates_client_from_dict`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_strips_whitespace_from_values`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_optional_fields_absent_when_empty`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_gemini_prefix_applied_via_from_config`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_extracts_single_object`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_ignores_braces_inside_strings`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_extracts_object_from_markdown_fence`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_no_object_returns_empty`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_explicit_wrapper_name_and_arguments`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_bare_visualize_args_match_visualize_not_execute`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_bare_execute_args_match_execute`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_ask_user_shape`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_nested_action_wrapper_shape`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_nested_tool_wrapper_shape`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_non_matching_object_returns_none`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_salvages_visualize_action_from_content`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_does_not_touch_response_with_native_tool_calls`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_plain_text_answer_left_untouched`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_no_tools_is_noop`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_openai_tool_calls_array_in_content`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_stringified_arguments_are_parsed`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。

#### `tests/backend/agents/test_context.py`（22 行，1 个 test_* ，类 0）

测试目标：验证 `backend/agents/context.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
- **`test_focused_context_includes_text_turn_and_loading_decision`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。

#### `tests/backend/agents/test_core_chart_contract.py`（49 行，2 个 test_* ，类 0）

测试目标：验证 `backend/agents/core_chart_contract.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
- **`test_visualize_schema_requires_title_and_exposes_subtitle`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_visualize_handler_forwards_title_and_subtitle`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。

#### `tests/backend/agents/test_data_loading_chat_images.py`（59 行，3 个 test_* ，类 0）

测试目标：验证 `backend/agents/data_loading_chat_images.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
- **`test_convert_message_keeps_instruction_before_image_attachment`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_convert_message_ignores_empty_image_attachment`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_build_system_prompt_accepts_workspace_table_name_strings`**：断言工作区创建、打开、迁移、过期或元数据原子更新。

#### `tests/backend/agents/test_data_loading_discovery_tools.py`（774 行，54 个 test_* ，类 9）

测试目标：验证 `backend/agents/data_loading_discovery_tools.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
测试类：TestListData、TestFindData、TestDescribeData、TestProposeLoadPlan、TestNormalizeLoadQueryFilters、TestBuildSystemPromptConnectorSummary、TestDataMemoryTools、TestProbeData、TestConnectorTools。
- **`test_connection`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_bootstraps_uncached_zero_auth_connector`**：断言身份解析、登录回调、未授权拒绝或 token 刷新行为。
- **`test_no_args_returns_sources_summary`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_no_user_home_returns_empty_sources`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_source_id_at_root`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_source_id_with_path_drills_into_folder`**：断言路径监禁：越狱输入被拒绝，合法相对路径可解析。
- **`test_filter_narrows_tables`**：断言导入/显示行数上限被执行。
- **`test_invalid_path_type_returns_error`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。
- **`test_bootstraps_uncached_zero_auth_connector`**：断言身份解析、登录回调、未授权拒绝或 token 刷新行为。
- **`test_empty_query_returns_error`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。
- **`test_searches_catalog_with_regex`**：断言目录缓存、同步、搜索或进度 API 的正确性。
- **`test_scope_with_source_id`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_scope_with_path_prefix`**：断言路径监禁：越狱输入被拒绝，合法相对路径可解析。
- **`test_scope_workspace_skips_catalog`**：断言目录缓存、同步、搜索或进度 API 的正确性。
- **`test_bad_regex_returns_error`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。
- **`test_no_match_returns_note_and_valid_sources`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_delegates_to_handle_read_catalog_metadata`**：断言目录缓存、同步、搜索或进度 API 的正确性。
- **`test_missing_params_still_calls_with_empty_strings`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_preserves_all_option_groups`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_empty_options_returns_empty_action`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_resolves_superset_dataset_id_from_catalog`**：断言目录缓存、同步、搜索或进度 API 的正确性。
- **`test_strips_wildcards_and_upgrades_eq_to_ilike`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_strips_wildcards_from_like`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_like_without_wildcards_upgraded_to_ilike`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_eq_without_wildcards_stays_eq`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_symbol_operators_mapped`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_contains_mapped_to_ilike`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_is_null_no_value`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_empty_wildcard_only_value_skipped`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_invalid_operator_falls_back_to_eq`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_non_list_returns_empty`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_missing_column_skipped`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_includes_connector_summary_when_sources_exist`**：断言连接器 CRUD、可见性隔离或表单字段。
- **`test_shows_none_when_no_sources`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_graceful_when_user_home_missing`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_includes_current_date_and_time`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_tools_are_exposed`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_read_append_and_rewrite`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_read_defaults_to_one_hundred_lines_and_pages`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_read_rejects_invalid_regex`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_unavailable_without_knowledge_store`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_prompt_labels_memory_as_stale_and_requires_verification`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_missing_ids_returns_error`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。
- **`test_unknown_table_key_returns_error`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。
- **`test_budget_exhaustion_returns_error`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。
- **`test_resolves_path_and_delegates_to_loader`**：断言路径监禁：越狱输入被拒绝，合法相对路径可解析。
- **`test_not_connected_source_returns_error`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。
- **`test_loader_error_result_passes_through`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。
- **`test_list_connectors_returns_available`**：断言连接器 CRUD、可见性隔离或表单字段。
- **`test_describe_connector_returns_full_detail`**：断言连接器 CRUD、可见性隔离或表单字段。
- **`test_describe_connector_unknown_type`**：断言连接器 CRUD、可见性隔离或表单字段。
- **`test_propose_connection_requires_list_first`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_propose_connection_emits_connect_form_action`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_propose_connection_unknown_type`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。

#### `tests/backend/agents/test_data_loading_skill.py`（405 行，11 个 test_* ，类 0）

测试目标：验证 `backend/agents/data_loading_skill.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
- **`test_registry_exposes_discovery_tools_only_after_skill_load`**：断言技能注册、工具可见性或 propose 拒绝重复加载。
- **`test_proposal_persists_executable_plan_and_emits_display_only_pause`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_narration_is_the_response_shown_to_the_user`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_invalid_proposal_returns_recoverable_observation`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_proposal_does_not_require_plan_descriptions`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_minimal_proposal_resolves_table_fields_from_catalog`**：断言目录缓存、同步、搜索或进度 API 的正确性。
- **`test_canonical_proposal_does_not_add_canvas_prose`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_proposal_rejects_exact_query_already_loaded_in_workspace`**：断言工作区创建、打开、迁移、过期或元数据原子更新。
- **`test_discovery_parameter_contract_matches_standalone_agent`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_skill_uses_shared_catalog_discovery`**：断言目录缓存、同步、搜索或进度 API 的正确性。
- **`test_probe_budget_is_shared_within_run_and_isolated_between_runs`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。

#### `tests/backend/agents/test_duckdb_notes_prompt.py`（36 行，2 个 test_* ，类 0）

测试目标：验证 `backend/agents/duckdb_notes_prompt.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
- **`test_duckdb_notes_mentions_non_ascii_double_quoting`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_duckdb_notes_mentions_identifier_quoting_rule`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。

#### `tests/backend/agents/test_generate_data_summary.py`（107 行，5 个 test_* ，类 1）

测试目标：验证 `backend/agents/generate_data_summary.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
测试类：TestInlineRowsFallback。
- **`test_workspace_table_uses_parquet`**：断言工作区创建、打开、迁移、过期或元数据原子更新。
- **`test_derived_table_falls_back_to_inline_rows`**：断言导入/显示行数上限被执行。
- **`test_no_workspace_no_rows_shows_unavailable`**：断言工作区创建、打开、迁移、过期或元数据原子更新。
- **`test_inline_rows_shows_in_memory_path`**：断言路径监禁：越狱输入被拒绝，合法相对路径可解析。
- **`test_inline_rows_sample_size_respected`**：断言导入/显示行数上限被执行。

#### `tests/backend/agents/test_model_registry.py`（157 行，11 个 test_* ，类 3）

测试目标：验证 `backend/agents/model_registry.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
测试类：TestModelDiscovery、TestPublicListingSecurity、TestCustomProvider。
- **`test_discovers_all_enabled_providers`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_total_model_count`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_empty_env_yields_no_models`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_skips_provider_without_models`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_skips_disabled_provider`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_no_api_key_in_public_info`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_public_fields_are_complete`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_full_config_contains_api_key`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_custom_provider_uses_explicit_endpoint`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_builtin_provider_uses_own_name_as_endpoint`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_custom_provider_defaults_to_openai_endpoint`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。

#### `tests/backend/agents/test_provenance_models.py`（105 行，7 个 test_* ，类 2）

测试目标：验证 `backend/agents/provenance_models.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
测试类：TestImportedFrom、TestDerivation。
- **`test_data_loader_roundtrip`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_upload_roundtrip`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_url_roundtrip`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_stream_roundtrip`**：断言 NDJSON 事件类型、预检错误或警告刷新。
- **`test_paste_roundtrip`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_example_roundtrip`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_roundtrip`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。

#### `tests/backend/agents/test_reasoning_content_helpers.py`（103 行，11 个 test_* ，类 2）

测试目标：验证 `backend/agents/reasoning_content_helpers.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
测试类：TestAttachReasoningContent、TestAccumulateReasoningContent。
- **`test_present`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_absent`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_none_value_not_attached`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_empty_string_attached`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_first_chunk`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_subsequent_chunks`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_no_attr`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_no_attr_preserves_accumulator`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_none_value_no_change`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_empty_string_delta_no_change`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_multiple_chunks_sequence`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。

#### `tests/backend/agents/test_reasoning_logger.py`（423 行，31 个 test_* ，类 8）

测试目标：验证 `backend/agents/reasoning_logger.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
测试类：TestLogFileCreation、TestOffMode、TestOnMode、TestVerboseMode、TestSecurityConstraints、TestExpiredLogCleanup、TestContextManager、TestNullReasoningLogger。
- **`test_creates_file_in_correct_directory`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_jsonl_format_each_line_parseable`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_each_line_has_step_type_and_ts`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_close_makes_file_complete`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_close_is_idempotent`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_rotates_to_new_date_directory`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_off_creates_no_file`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_off_case_insensitive`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_on_writes_structured_summary`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_on_strips_messages_defensively`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_on_auto_fields_cannot_be_overridden`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_default_is_off`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_verbose_writes_full_content`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_verbose_sanitizes_api_key`**：断言日志或响应中的密钥、URL 口令、堆栈被脱敏。
- **`test_verbose_sanitizes_password_in_list`**：断言日志或响应中的密钥、URL 口令、堆栈被脱敏。
- **`test_verbose_sanitizes_top_level_kwargs`**：断言日志或响应中的密钥、URL 口令、堆栈被脱敏。
- **`test_verbose_case_insensitive`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_no_api_key_in_log`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_no_connection_string_in_log`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_confined_dir_prevents_traversal`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_old_directories_removed`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_cleanup_ignores_non_date_dirs`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_cleanup_failure_does_not_raise`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_cleanup_nonexistent_dir_does_not_raise`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_cleanup_runs_in_background`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_exception_still_closes_file`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_off_mode_context_manager_safe`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_log_is_noop`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_close_is_noop`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_context_manager`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_level_is_off`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。

#### `tests/backend/agents/test_semantic_types.py`（349 行，34 个 test_* ，类 11）

测试目标：验证 `backend/agents/semantic_types.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
测试类：TestAllSemanticTypes、TestIsMeasureType、TestIsTimeseriesType、TestIsCategoricalType、TestIsOrdinalType、TestIsGeoType、TestIsNonMeasureNumeric、TestIsSignedMeasure、TestGetVlType、TestInferVlTypeFromName、TestGenerateSemanticTypesPrompt。
- **`test_no_duplicates`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_includes_known_types`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_every_vl_map_key_is_in_all_types`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_vl_map_values_are_valid`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_all_types_covered_in_semantic_categories`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_measure_types_return_true`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_non_measure_types_return_false`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_unknown_string_returns_false`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_timeseries_types_return_true`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_non_timeseries_return_false`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_categorical_types_return_true`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_non_categorical_return_false`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_ordinal_types_return_true`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_non_ordinal_return_false`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_geo_types_return_true`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_non_geo_return_false`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_non_measure_numerics_return_true`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_others_return_false`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_signed_measures_return_true`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_non_signed_return_false`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_vl_type_mapping`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_unknown_type_returns_none`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_temporal_names`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_ordinal_names`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_quantitative_names`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_nominal_names`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_no_signal_names_return_none`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_temporal_takes_priority_over_ordinal`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_case_insensitive`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_returns_non_empty_string`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_contains_category_headers`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_contains_type_names`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_contains_guidelines`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_all_registered_types_appear_in_prompt`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。

#### `tests/backend/agents/test_sort_data_agent.py`（200 行，10 个 test_* ，类 2）

测试目标：验证 `backend/agents/sort_data_agent.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
测试类：TestInputConstruction、TestResponseParsing。
- **`test_input_key_is_values_not_value`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_input_name_is_preserved`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_input_values_are_preserved`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_unicode_values_serialised_correctly`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_valid_json_block_in_content_returns_ok`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_json_wrapped_in_text_is_extracted`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_unparseable_content_returns_error_status`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。
- **`test_multiple_choices_produce_multiple_candidates`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_agent_field_is_set`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_dialog_includes_system_and_user_and_assistant`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。

#### `tests/backend/agents/test_tool_path_safety.py`（196 行，13 个 test_* ，类 4）

测试目标：验证 `backend/agents/tool_path_safety.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
测试类：TestToolReadFile、TestToolListDirectory、TestToolWriteFile、TestPreviewScratchFiles。
- **`test_read_valid_file`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_traversal_blocked`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_absolute_path_blocked`**：断言路径监禁：越狱输入被拒绝，合法相对路径可解析。
- **`test_nonexistent_file`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_empty_path_blocked`**：断言路径监禁：越狱输入被拒绝，合法相对路径可解析。
- **`test_list_valid_directory`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_list_root_directory`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_traversal_blocked`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_nonexistent_directory`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_write_valid_file`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_traversal_sanitized_and_confined`**：断言日志或响应中的密钥、URL 口令、堆栈被脱敏。
- **`test_traversal_blocked`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_valid_scratch_file`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。

#### `tests/backend/agents/test_workflow_distill.py`（571 行，26 个 test_* ，类 4）

测试目标：验证 `backend/agents/workflow_distill.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
测试类：TestExtractContextSummary、TestRunWithMockedLLM、TestWorkflowFilename、TestDistillEndpoint。
- **`test_renders_each_event_type`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_empty_events_returns_marker`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_user_content_is_not_displaycontent`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_skips_non_dict_events`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_create_table_basic`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_create_chart_without_encoding`**：断言 visualize 契约、标题或编码。
- **`test_renders_multi_thread_with_headers`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_produces_valid_markdown`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_fallback_front_matter_added`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_retries_once_when_body_too_long`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_retry_asks_for_slack_under_limit`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_hard_trims_when_retry_still_over_limit`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_no_retry_when_body_within_limit`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_language_instruction_injected_into_system_prompt`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_language_code_zh_injects_chinese_instruction`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_language_code_en_no_extra_instruction`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_derives_from_title`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_fallback_when_title_blank`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_rejects_path_traversal`**：断言路径监禁：越狱输入被拒绝，合法相对路径可解析。
- **`test_strips_reserved_and_control_chars`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_missing_context_returns_error`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。
- **`test_missing_model_returns_error`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。
- **`test_missing_events_returns_error`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。
- **`test_missing_events_field_returns_error`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。
- **`test_successful_distill`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_category_hint_creates_subdir`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。

#### `tests/backend/auth/sso_provider_contracts/__init__.py`（2 行，0 个 test_* ，类 0）

测试目标：验证 `backend/auth/sso_provider_contracts/__init__.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
该文件可能是 fixture、benchmark 或辅助模块，不含 test_* 函数。

#### `tests/backend/auth/sso_provider_contracts/provider_fixtures.py`（255 行，0 个 test_* ，类 0）

测试目标：验证 `backend/auth/sso_provider_contracts/provider_fixtures.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
该文件可能是 fixture、benchmark 或辅助模块，不含 test_* 函数。

#### `tests/backend/auth/sso_provider_contracts/test_oauth_provider_contracts.py`（179 行，5 个 test_* ，类 1）

测试目标：验证 `backend/auth/sso_provider_contracts/oauth_provider_contracts.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
测试类：TestGitHubOAuthContract。
- **`test_login_redirect_matches_github_oauth_contract`**：断言身份解析、登录回调、未授权拒绝或 token 刷新行为。
- **`test_callback_exchanges_code_and_stores_github_session`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_callback_fetches_primary_email_when_github_user_email_is_private`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_callback_rejects_invalid_github_oauth_state`**：断言身份解析、登录回调、未授权拒绝或 token 刷新行为。
- **`test_callback_rejects_missing_github_access_token`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。

#### `tests/backend/auth/sso_provider_contracts/test_oidc_provider_contracts.py`（342 行，6 个 test_* ，类 1）

测试目标：验证 `backend/auth/sso_provider_contracts/oidc_provider_contracts.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
测试类：TestMainstreamOIDCProviderContracts。
- **`test_discovery_metadata_resolves_provider_endpoints`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_jwks_jwt_validation_accepts_provider_claim_shape`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_jwks_jwt_validation_accepts_missing_optional_email_claim`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_userinfo_fallback_accepts_provider_userinfo_response`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_backend_gateway_uses_provider_authorization_endpoint`**：断言身份解析、登录回调、未授权拒绝或 token 刷新行为。
- **`test_backend_gateway_exchanges_code_and_stores_userinfo`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。

#### `tests/backend/auth/test_auth.py`（139 行，16 个 test_* ，类 2）

测试目标：验证 `backend/auth/auth.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
测试类：TestValidateIdentityValue、TestGetIdentityId。
- **`test_valid_uuid`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_valid_email`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_strips_whitespace`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_empty_raises`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_whitespace_only_raises`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_too_long_raises`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_path_separator_rejected`**：断言路径监禁：越狱输入被拒绝，合法相对路径可解析。
- **`test_shell_metachar_rejected`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_control_chars_rejected`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_azure_principal_returns_user_prefix`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_browser_identity_returns_browser_prefix`**：断言导入/显示行数上限被执行。
- **`test_client_cannot_spoof_user_prefix`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_azure_header_takes_priority_over_browser`**：断言导入/显示行数上限被执行。
- **`test_missing_all_headers_raises`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_malformed_azure_header_rejected`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_browser_identity_strips_prefix`**：断言导入/显示行数上限被执行。

#### `tests/backend/auth/test_auth_info_endpoint.py`（91 行，4 个 test_* ，类 1）

测试目标：验证 `backend/auth/auth_info_endpoint.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
测试类：TestAuthInfoEndpoint。
- **`test_anonymous_mode_returns_none_action`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_oidc_provider_returns_frontend_action`**：断言发现文档、JWT/JWKS、callback、logout 与 AUTH_MODE。
- **`test_github_provider_returns_redirect_action`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_azure_provider_returns_transparent_action`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。

#### `tests/backend/auth/test_auth_provider_chain.py`（302 行，27 个 test_* ，类 9）

测试目标：验证 `backend/auth/auth_provider_chain.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
测试类：TestProviderDiscovery、TestInitAuth、TestProviderDispatch、TestAnonymousMode、TestSSOToken、TestAuthenticationErrorPropagation、TestAllPhase1ProvidersDiscovered、TestOIDCViaInitAuth、TestIdentityRegexExpansion。
- **`test_azure_easyauth_is_discovered`**：断言身份解析、登录回调、未授权拒绝或 token 刷新行为。
- **`test_get_provider_class_returns_class`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_unknown_provider_returns_none`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_no_env_var_stays_anonymous`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_explicit_anonymous_stays_anonymous`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_azure_easyauth_activates`**：断言身份解析、登录回调、未授权拒绝或 token 刷新行为。
- **`test_unknown_provider_logs_error_stays_none`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。
- **`test_allow_anonymous_false`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_provider_authenticated_returns_user_prefix`**：断言身份解析、登录回调、未授权拒绝或 token 刷新行为。
- **`test_provider_miss_falls_back_to_anonymous`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_provider_miss_no_anonymous_raises`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_browser_identity_works`**：断言导入/显示行数上限被执行。
- **`test_prefixed_identity_stripped`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_missing_header_raises`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_spoofed_user_prefix_forced_to_browser`**：断言导入/显示行数上限被执行。
- **`test_anonymous_mode_returns_none`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_provider_without_token_returns_none`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_authentication_error_becomes_value_error`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。
- **`test_oidc_discovered`**：断言发现文档、JWT/JWKS、callback、logout 与 AUTH_MODE。
- **`test_github_discovered`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_azure_easyauth_discovered`**：断言身份解析、登录回调、未授权拒绝或 token 刷新行为。
- **`test_oidc_activates_when_configured`**：断言发现文档、JWT/JWKS、callback、logout 与 AUTH_MODE。
- **`test_oidc_disabled_without_env_vars`**：断言发现文档、JWT/JWKS、callback、logout 与 AUTH_MODE。
- **`test_oidc_no_bearer_falls_back_to_anonymous`**：断言发现文档、JWT/JWKS、callback、logout 与 AUTH_MODE。
- **`test_oidc_style_sub_claims_accepted`**：断言发现文档、JWT/JWKS、callback、logout 与 AUTH_MODE。
- **`test_path_separator_still_rejected`**：断言路径监禁：越狱输入被拒绝，合法相对路径可解析。
- **`test_space_still_rejected`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。

#### `tests/backend/auth/test_azure_cli.py`（16 行，1 个 test_* ，类 0）

测试目标：验证 `backend/auth/azure_cli.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
- **`test_expose_azure_cli_adds_executable_directory_to_path`**：断言路径监禁：越狱输入被拒绝，合法相对路径可解析。

#### `tests/backend/auth/test_azure_easyauth_provider.py`（94 行，9 个 test_* ，类 2）

测试目标：验证 `backend/auth/azure_easyauth_provider.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
测试类：TestAzureEasyAuthProviderMetadata、TestAzureEasyAuthAuthenticate。
- **`test_name`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_enabled_always_true`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_get_auth_info_action`**：断言身份解析、登录回调、未授权拒绝或 token 刷新行为。
- **`test_principal_header_present`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_principal_header_with_display_name`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_empty_display_name_becomes_none`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_no_header_returns_none`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_strips_whitespace_from_user_id`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_raw_token_is_none`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。

#### `tests/backend/auth/test_credential_vault.py`（157 行，18 个 test_* ，类 6）

测试目标：验证 `backend/auth/credential_vault.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
测试类：TestStoreAndRetrieve、TestUserIsolation、TestDelete、TestListSources、TestEncryptionKeyMismatch、TestEdgeCases。
- **`test_round_trip`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_retrieve_missing_returns_none`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_overwrite`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_different_users_isolated`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_cross_user_retrieve_returns_none`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_browser_identity_works`**：断言导入/显示行数上限被执行。
- **`test_delete_removes_credential`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_delete_nonexistent_is_noop`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_delete_one_source_keeps_others`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_empty_initially`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_lists_stored_sources`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_list_after_delete`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_list_isolated_per_user`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_wrong_key_returns_none`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_invalid_key_raises`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_complex_credentials`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_unicode_credentials`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_empty_credentials_dict`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。

#### `tests/backend/auth/test_credential_vault_factory.py`（171 行，8 个 test_* ，类 4）

测试目标：验证 `backend/auth/credential_vault_factory.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
测试类：TestExplicitKey、TestAutoGeneratedKey、TestFactoryReturnsNone、TestSingleton。
- **`test_env_key_takes_priority`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_default_type_is_local`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_auto_generates_key_when_no_env`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_reuses_existing_key_file`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_auto_generated_vault_is_functional`**：断言凭证加解密、按用户隔离、列表不返回明文。
- **`test_unknown_type_returns_none`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_multiple_calls_return_same_instance`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_singleton_is_cached`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。

#### `tests/backend/auth/test_flask_session_config.py`（50 行，5 个 test_* ，类 2）

测试目标：验证 `backend/auth/flask_session_config.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
测试类：TestFlaskSecretKey、TestSessionConfig。
- **`test_uses_env_secret_key`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_falls_back_to_random_when_env_missing`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_permanent_session_lifetime_is_one_year`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_session_cookie_httponly`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_session_cookie_samesite`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。

#### `tests/backend/auth/test_github_oauth_provider.py`（131 行，11 个 test_* ，类 3）

测试目标：验证 `backend/auth/github_oauth_provider.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
测试类：TestGitHubProviderMetadata、TestGitHubAuthenticate、TestGitHubGateway。
- **`test_name`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_enabled_with_both_vars`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_disabled_without_client_id`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_disabled_without_client_secret`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_get_auth_info_is_redirect`**：断言身份解析、登录回调、未授权拒绝或 token 刷新行为。
- **`test_session_with_github_user`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_empty_session_returns_none`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_wrong_provider_in_session_returns_none`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_missing_provider_key_returns_none`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_callback_missing_code_redirects`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_logout_returns_json_ok`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。

#### `tests/backend/auth/test_kusto_oauth_gateway.py`（148 行，4 个 test_* ，类 0）

测试目标：验证 `backend/auth/kusto_oauth_gateway.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
- **`test_login_uses_cluster_scope_and_pkce`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_login_rejects_untrusted_cluster_urls`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_callback_exchanges_code_and_posts_token_to_opener`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_callback_state_cannot_be_replayed`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。

#### `tests/backend/auth/test_oidc_gateway.py`（441 行，29 个 test_* ，类 8）

测试目标：验证 `backend/auth/oidc_gateway.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
测试类：TestOIDCLogin、TestOIDCCallback、TestOIDCStatus、TestOIDCLogout、TestAutoDetection、TestClearServiceToken、TestSaveDelegatedToken、TestAuthServiceStatus。
- **`test_login_redirects_to_authorize_url`**：断言身份解析、登录回调、未授权拒绝或 token 刷新行为。
- **`test_login_sets_state_in_session`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_login_disabled_without_secret`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_login_fails_without_authorize_url`**：断言身份解析、登录回调、未授权拒绝或 token 刷新行为。
- **`test_login_redirect_uri_uses_auth_callback`**：断言身份解析、登录回调、未授权拒绝或 token 刷新行为。
- **`test_callback_rejects_missing_code`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_callback_rejects_invalid_state`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_callback_success_stores_tokens_and_redirects`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_callback_token_exchange_failure`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_callback_disabled_without_secret`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_callback_access_denied_returns_access_denied_error`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。
- **`test_callback_access_denied_clears_oauth_state`**：断言身份解析、登录回调、未授权拒绝或 token 刷新行为。
- **`test_callback_idp_server_error_returns_token_exchange_failed`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。
- **`test_callback_access_denied_preserves_existing_sso_session`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_status_authenticated`**：断言身份解析、登录回调、未授权拒绝或 token 刷新行为。
- **`test_status_not_authenticated`**：断言身份解析、登录回调、未授权拒绝或 token 刷新行为。
- **`test_status_frontend_mode`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_logout_clears_session`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_logout_does_not_delete_vault_credentials`**：断言凭证加解密、按用户隔离、列表不返回明文。
- **`test_secret_present_implies_backend`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_no_secret_implies_frontend`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_auth_mode_overrides_auto_detection`**：断言身份解析、登录回调、未授权拒绝或 token 刷新行为。
- **`test_auth_mode_backend_without_secret`**：断言身份解析、登录回调、未授权拒绝或 token 刷新行为。
- **`test_clear_token_removes_from_session`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_clear_nonexistent_token_is_ok`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_save_token_success`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_save_token_missing_fields`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_save_token_missing_system_id`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_service_status_returns_dict`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。

#### `tests/backend/auth/test_oidc_provider.py`（276 行，18 个 test_* ，类 4）

测试目标：验证 `backend/auth/oidc_provider.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
测试类：TestOIDCProviderMetadata、TestOIDCAuthenticate、TestOIDCAuthErrors、TestOIDCSkip。
- **`test_name`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_enabled_with_config`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_disabled_without_issuer`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_disabled_without_client_id`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_get_auth_info_action_is_frontend`**：断言身份解析、登录回调、未授权拒绝或 token 刷新行为。
- **`test_default_scopes_include_offline_access`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_custom_scopes_override_defaults`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_valid_jwt_returns_auth_result`**：断言身份解析、登录回调、未授权拒绝或 token 刷新行为。
- **`test_numeric_sub_is_rejected`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_expired_token_raises`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_wrong_issuer_raises`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_wrong_audience_raises`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_wrong_key_raises`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_missing_sub_claim_raises`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_no_authorization_header`**：断言身份解析、登录回调、未授权拒绝或 token 刷新行为。
- **`test_non_bearer_authorization`**：断言身份解析、登录回调、未授权拒绝或 token 刷新行为。
- **`test_empty_bearer_token`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_no_jwks_client_returns_none`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。

#### `tests/backend/auth/test_token_store.py`（393 行，34 个 test_* ，类 9）

测试目标：验证 `backend/auth/token_store.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
测试类：TestStoreServiceToken、TestStoreSSO、TestExpiry、TestGetAccess、TestRefresh、TestSSOExchange、TestGetAuthStatus、TestAvailableStrategies、TestResolveEnv。
- **`test_store_and_retrieve_via_session`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_store_overwrites_previous`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_clear_service_token`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_clear_sso_exchange_token_blocks_auto_reconnect`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_store_service_token_allows_auto_reconnect_again`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_clear_session_tokens_preserves_vault`**：断言凭证加解密、按用户隔离、列表不返回明文。
- **`test_store_sso_tokens`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_get_sso_token_backend_mode`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_get_sso_token_backend_expired_no_refresh`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_get_sso_token_frontend_mode`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_is_expired_true`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_is_expired_false`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_is_expired_missing_key`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_returns_none_when_no_auth_config`**：断言身份解析、登录回调、未授权拒绝或 token 刷新行为。
- **`test_returns_cached_token`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_skips_expired_cache_tries_refresh`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_falls_through_to_vault`**：断言凭证加解密、按用户隔离、列表不返回明文。
- **`test_returns_none_when_all_fail`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_refresh_success`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_refresh_no_token_url`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_refresh_http_failure`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_sso_exchange_success`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_sso_exchange_no_sso_token`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_sso_exchange_no_exchange_url`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_sso_exchange_skips_when_blocked`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_returns_status_for_configured_systems`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_returns_unauthorized_for_missing_token`**：断言身份解析、登录回调、未授权拒绝或 token 刷新行为。
- **`test_sso_exchange_available`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_delegated_popup`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_manual_credentials`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_oauth2_redirect`**：断言身份解析、登录回调、未授权拒绝或 token 刷新行为。
- **`test_resolve_env`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_resolve_env_missing`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_resolve_env_empty_key`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。

#### `tests/backend/benchmarks/benchmark_sandbox.py`（232 行，0 个 test_* ，类 0）

测试目标：验证 `backend/benchmarks/benchmark_sandbox.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
该文件可能是 fixture、benchmark 或辅助模块，不含 test_* 函数。

#### `tests/backend/benchmarks/benchmark_workspace.py`（530 行，0 个 test_* ，类 0）

测试目标：验证 `backend/benchmarks/benchmark_workspace.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
该文件可能是 fixture、benchmark 或辅助模块，不含 test_* 函数。

#### `tests/backend/data/test_all_loader_verification.py`（231 行，13 个 test_* ，类 5）

测试目标：验证 `backend/data/all_loader_verification.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
测试类：TestAllLoaderCatalogHierarchies、TestScopePinningAllLoaders、TestAuthModes、TestStaticMethods、TestDataConnectorWrapping。
- **`test_all_loaders_have_hierarchy`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_hierarchies_match_expected`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_last_level_is_importable`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_pinning_removes_level`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_no_pinning_returns_full_hierarchy`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_default_auth_mode_is_connection`**：断言身份解析、登录回调、未授权拒绝或 token 刷新行为。
- **`test_superset_uses_token_mode`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_all_loaders_have_list_params`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_all_loaders_have_auth_instructions`**：断言身份解析、登录回调、未授权拒绝或 token 刷新行为。
- **`test_all_loaders_have_required_host_or_identifier`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_rate_limit_returns_dict_or_none`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_all_loaders_can_be_wrapped`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_all_loaders_blueprints_have_all_routes`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。

#### `tests/backend/data/test_atomic_metadata_update.py`（53 行，1 个 test_* ，类 0）

测试目标：验证 `backend/data/atomic_metadata_update.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
- **`test_concurrent_add_table_no_lost_updates`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。

#### `tests/backend/data/test_catalog_cache.py`（692 行，49 个 test_* ，类 11）

测试目标：验证 `backend/data/catalog_cache.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
测试类：TestSaveLoadCatalog、TestDeleteCatalog、TestListCachedSources、TestSearchCatalogCache、TestStructuredFieldSearch、TestListSourcesSummary、TestListPathChildren、TestSearchCatalogCacheExtended、TestConnectorConnectCatalogSave、TestCatalogCacheSyncedAt、TestSearchReturnsTableKey。
- **`test_save_creates_directory_and_file`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_load_returns_saved_tables`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_load_returns_none_for_missing`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_save_overwrites_existing`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_source_id_with_special_chars`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_legacy_synced_at_populates_both_freshness_clocks`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_listing_and_metadata_have_independent_freshness`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_listing_refresh_preserves_enriched_metadata`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_refresh_failure_preserves_last_good_tables`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_atomic_replace_failure_preserves_existing_catalog`**：断言目录缓存、同步、搜索或进度 API 的正确性。
- **`test_delete_removes_file`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_delete_nonexistent_is_silent`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_delete_rejects_symlink_escape`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_returns_source_ids`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_returns_empty_for_missing_dir`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_returns_canonical_id_with_colon`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_falls_back_to_stem_when_source_id_missing`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_search_by_table_name`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_search_by_description`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_search_by_column_name`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_search_excludes_imported_tables`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_search_returns_empty_for_no_match`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_search_respects_limit_per_source`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_table_name_match_reports_table_name_reason`**：断言报告流式字段或工具语法剥离。
- **`test_table_description_match`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_column_name_match`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_column_description_match`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_no_match_returns_empty`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_exclude_tables_drops_matches`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_search_catalog_cache_end_to_end`**：断言目录缓存、同步、搜索或进度 API 的正确性。
- **`test_regex_query_alternation`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_flat_and_hierarchical`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_empty_when_no_cache`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_root_lists_folders_and_top_level_tables`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_drill_into_folder`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_filter_narrows_results`**：断言导入/显示行数上限被执行。
- **`test_missing_source_returns_empty`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_truncation_includes_hint`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_regex_alternation_matches_two_tables`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_exclude_pattern_filters_out_matches`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_path_prefix_scopes_search`**：断言路径监禁：越狱输入被拒绝，合法相对路径可解析。
- **`test_fields_restricts_search_surface`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_bad_regex_raises_catalog_search_error`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。
- **`test_connect_saves_catalog_to_user_home`**：断言目录缓存、同步、搜索或进度 API 的正确性。
- **`test_connection`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_save_writes_synced_at`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_load_catalog_ignores_synced_at`**：断言目录缓存、同步、搜索或进度 API 的正确性。
- **`test_python_search_includes_table_key`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_python_search_empty_table_key`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。

#### `tests/backend/data/test_catalog_refresh.py`（109 行，4 个 test_* ，类 0）

测试目标：验证 `backend/data/catalog_refresh.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
- **`test_disconnected_source_serves_stale_without_refresh`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_missing_connected_source_refreshes_synchronously`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_stale_refresh_is_deduplicated_and_serves_stale`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_recent_failure_suppresses_retry_and_preserves_tables`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。

#### `tests/backend/data/test_catalog_search_loaders.py`（95 行，3 个 test_* ，类 0）

测试目标：验证 `backend/data/catalog_search_loaders.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
- **`test_postgresql_search_catalog_returns_lightweight_tree`**：断言目录缓存、同步、搜索或进度 API 的正确性。
- **`test_mysql_search_catalog_returns_lightweight_tree`**：断言目录缓存、同步、搜索或进度 API 的正确性。
- **`test_superset_search_catalog_returns_dataset_and_dashboard_matches`**：断言目录缓存、同步、搜索或进度 API 的正确性。

#### `tests/backend/data/test_catalog_sync_base.py`（137 行，15 个 test_* ，类 4）

测试目标：验证 `backend/data/catalog_sync_base.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
测试类：TestSyncCatalogMetadataDefault、TestEnsureTableKeys、TestSourceMetadataStatusConstants、TestCatalogErrorCodes。
- **`test_returns_list_tables_results`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_passes_table_filter`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_empty_tables`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_existing_table_key_preserved`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_fallback_to_source_name`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_fallback_to_name`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_fallback_to_name_no_metadata`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_empty_list`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_does_not_overwrite_existing_key`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_synced_value`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_not_synced_value`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_partial_value`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_unavailable_value`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_catalog_sync_timeout`**：断言目录缓存、同步、搜索或进度 API 的正确性。
- **`test_catalog_not_found`**：断言目录缓存、同步、搜索或进度 API 的正确性。

#### `tests/backend/data/test_connector_directory_storage.py`（285 行，11 个 test_* ，类 4）

测试目标：验证 `backend/data/connector_directory_storage.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
测试类：TestSafeSourceId、TestPersistAndRemove、TestLoadUserSpecs、TestLoadConnectorsIntegration。
- **`test_connection`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_sanitises_special_chars`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_persist_creates_directory_and_json`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_persist_overwrites_existing`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_remove_deletes_file`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_remove_nonexistent_is_silent`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_remove_rejects_symlink_escape`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_loads_from_json_directory`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_loads_multiple_connectors`**：断言连接器 CRUD、可见性隔离或表单字段。
- **`test_returns_empty_when_no_connectors`**：断言连接器 CRUD、可见性隔离或表单字段。
- **`test_loads_user_connectors_from_directory`**：断言连接器 CRUD、可见性隔离或表单字段。

#### `tests/backend/data/test_connector_errors.py`（74 行，4 个 test_* ，类 0）

测试目标：验证 `backend/data/connector_errors.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
- **`test_classify_connector_error`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。
- **`test_http_401_maps_to_connector_auth_failed`**：断言身份解析、登录回调、未授权拒绝或 token 刷新行为。
- **`test_http_500_with_lost_connection_maps_to_connection_failed`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_azure_sql_firewall_denial_exposes_safe_client_ip`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。

#### `tests/backend/data/test_data_connector_config.py`（618 行，26 个 test_* ，类 7）

测试目标：验证 `backend/data/data_connector_config.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
测试类：TestResolveEnvRefs、TestLoadConnectorsYaml、TestEnvVarParsing、TestLoadAdminSpecs、TestUserConnectorPersistence、TestLoadConnectors、TestRegisterConnectedSources。
- **`test_connection`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_resolves_env_var`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_missing_env_var_becomes_empty`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_non_env_ref_passed_through`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_load_valid_file`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_returns_empty_for_missing_file`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_returns_empty_for_bad_yaml`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_parse_env_sources_basic`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_parse_env_sources_multiple`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_parse_env_sources_missing_type_skipped`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_parse_env_sources_default_name`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_load_from_connectors_yaml`**：断言连接器 CRUD、可见性隔离或表单字段。
- **`test_env_overrides_yaml`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_multiple_instances_same_type`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_env_ref_resolution_in_yaml_params`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_save_and_load_user_connectors`**：断言连接器 CRUD、可见性隔离或表单字段。
- **`test_load_user_specs_returns_empty_if_no_file`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_loads_user_connectors_on_first_call`**：断言连接器 CRUD、可见性隔离或表单字段。
- **`test_does_not_overwrite_admin_connectors`**：断言连接器 CRUD、可见性隔离或表单字段。
- **`test_second_call_is_noop`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_user_connectors_are_scoped_by_identity`**：断言连接器 CRUD、可见性隔离或表单字段。
- **`test_create_connector_persists_only_non_auth_params`**：断言身份解析、登录回调、未授权拒绝或 token 刷新行为。
- **`test_registers_blueprints`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_skips_unknown_loader_type`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_logs_disabled_loaders`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_frontend_config_in_sources`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。

#### `tests/backend/data/test_data_connector_framework.py`（908 行，55 个 test_* ，类 10）

测试目标：验证 `backend/data/data_connector_framework.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
测试类：TestSharedRouteRegistration、TestFrontendConfig、TestConnectorList、TestAuthRoutes、TestCatalogRoutes、TestDataRoutes、TestErrorHandling、TestIdentityIsolation、TestScopePinning、TestHelpers。
- **`test_connection`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_connection`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_connection`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_shared_routes_registered`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_frontend_config_structure`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_pinned_params_excluded_from_form`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_hierarchy_included`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_pinned_source_effective_hierarchy`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_pinned_params_do_not_expose_auth_or_sensitive_values`**：断言身份解析、登录回调、未授权拒绝或 token 刷新行为。
- **`test_sso_auto_connect_respects_session_block`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_connect_success`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_connect_merges_default_params`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_connect_bad_host_returns_error`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。
- **`test_connect_fails_when_test_connection_fails`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_get_status_returns_structured_connection_error`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。
- **`test_delete_connector_clears_status`**：断言连接器 CRUD、可见性隔离或表单字段。
- **`test_rename_connector_preserves_stable_id`**：断言连接器 CRUD、可见性隔离或表单字段。
- **`test_disconnect_connector_clears_loader_and_credentials`**：断言连接器 CRUD、可见性隔离或表单字段。
- **`test_status_connected`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_status_not_connected`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_status_not_connected_after_no_connect`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_safe_params_exclude_password`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_ls_root`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_ls_returns_hierarchy`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_ls_drill_down_to_tables`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_ls_with_filter`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_ls_not_connected_returns_error`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。
- **`test_catalog_metadata`**：断言目录缓存、同步、搜索或进度 API 的正确性。
- **`test_catalog_tree`**：断言目录缓存、同步、搜索或进度 API 的正确性。
- **`test_catalog_tree_with_filter`**：断言目录缓存、同步、搜索或进度 API 的正确性。
- **`test_search_catalog_route`**：断言目录缓存、同步、搜索或进度 API 的正确性。
- **`test_search_catalog_empty_query_returns_empty_tree`**：断言目录缓存、同步、搜索或进度 API 的正确性。
- **`test_search_catalog_not_connected_returns_error`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。
- **`test_preview`**：断言预览行数、缓存或替换语义。
- **`test_preview_missing_source_table`**：断言预览行数、缓存或替换语义。
- **`test_import_requires_source_table`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_import_success`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_refresh_requires_table_name`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_column_values_success`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_column_values_missing_source_table`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_column_values_missing_column_name`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_column_values_with_keyword`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_column_values_not_connected`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_sanitize_error_connection_refused`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。
- **`test_sanitize_error_permission`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。
- **`test_sanitize_error_invalid_params`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。
- **`test_sanitize_error_unknown`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。
- **`test_error_does_not_leak_internal_details`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。
- **`test_different_identities_have_separate_loaders`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_delete_does_not_affect_other_user`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_pinned_database_skips_database_level`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_pinned_scope_in_connect_response`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_node_to_dict`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_resolve_env_refs`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_resolve_env_refs_missing`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。

#### `tests/backend/data/test_data_connector_vault.py`（462 行，23 个 test_* ，类 6）

测试目标：验证 `backend/data/data_connector_vault.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
测试类：TestVaultHelpers、TestConnectStoresCredentials、TestDeleteCredentials、TestAutoReconnect、TestNoVaultFallback、TestIdentityIsolation。
- **`test_connection`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_vault_store_and_retrieve`**：断言凭证加解密、按用户隔离、列表不返回明文。
- **`test_vault_retrieve_when_empty`**：断言凭证加解密、按用户隔离、列表不返回明文。
- **`test_vault_delete`**：断言凭证加解密、按用户隔离、列表不返回明文。
- **`test_vault_unavailable_returns_false`**：断言凭证加解密、按用户隔离、列表不返回明文。
- **`test_has_stored_credentials`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_vault_exception_is_caught`**：断言凭证加解密、按用户隔离、列表不返回明文。
- **`test_connect_does_not_auto_persist`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_persist_credentials_stores_in_vault`**：断言凭证加解密、按用户隔离、列表不返回明文。
- **`test_connect_via_route_stores_in_vault`**：断言凭证加解密、按用户隔离、列表不返回明文。
- **`test_connect_via_route_persist_false`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_connect_persist_false_clears_old_vault_entry`**：断言凭证加解密、按用户隔离、列表不返回明文。
- **`test_delete_credentials_clears_vault`**：断言凭证加解密、按用户隔离、列表不返回明文。
- **`test_require_loader_auto_reconnects`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_auto_reconnect_cleans_stale_creds`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_auto_reconnect_exception_cleans_stale_creds`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_status_reports_stored_credentials`**：断言报告流式字段或工具语法剥离。
- **`test_auth_status_not_connected_no_vault`**：断言身份解析、登录回调、未授权拒绝或 token 刷新行为。
- **`test_connect_without_vault`**：断言凭证加解密、按用户隔离、列表不返回明文。
- **`test_connect_route_without_vault_not_persisted`**：断言凭证加解密、按用户隔离、列表不返回明文。
- **`test_require_loader_no_vault_raises`**：断言凭证加解密、按用户隔离、列表不返回明文。
- **`test_different_users_separate_vault_entries`**：断言凭证加解密、按用户隔离、列表不返回明文。
- **`test_delete_only_affects_own_user`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。

#### `tests/backend/data/test_df_to_safe_records.py`（93 行，9 个 test_* ，类 3）

测试目标：验证 `backend/data/df_to_safe_records.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
测试类：TestDatetimeSerialization、TestMixedTypes、TestEdgeCases。
- **`test_datetime_column_returns_iso_string`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_datetime_with_time_component`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_nat_becomes_null`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_int_string_datetime_mixed`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_float_with_nan`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_empty_dataframe`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_empty_dataframe_no_columns`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_default_handler_catches_exotic_types`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_malformed_unicode_is_replaced_without_mutating_input`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。

#### `tests/backend/data/test_ephemeral_workspace.py`（136 行，4 个 test_* ，类 0）

测试目标：验证 `backend/data/ephemeral_workspace.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
- **`test_ephemeral_matches_local_workspace_lifecycle_without_retention`**：断言工作区创建、打开、迁移、过期或元数据原子更新。
- **`test_factory_uses_same_lazy_workspace_contract`**：断言工作区创建、打开、迁移、过期或元数据原子更新。
- **`test_cleanup_expires_only_ephemeral_workspaces`**：断言工作区创建、打开、迁移、过期或元数据原子更新。
- **`test_cleanup_lru_evicts_oldest_workspace`**：断言工作区创建、打开、迁移、过期或元数据原子更新。

#### `tests/backend/data/test_excel_fixture_parsing.py`（24 行，1 个 test_* ，类 0）

测试目标：验证 `backend/data/excel_fixture_parsing.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
- **`test_manual_xls_fixture_can_be_parsed`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。

#### `tests/backend/data/test_external_data_loader_table_names.py`（37 行，6 个 test_* ，类 0）

测试目标：验证 `backend/data/external_data_loader_table_names.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
- **`test_external_loader_sanitize_rejects_empty_name`**：断言日志或响应中的密钥、URL 口令、堆栈被脱敏。
- **`test_external_loader_sanitize_prefixes_sql_keywords`**：断言日志或响应中的密钥、URL 口令、堆栈被脱敏。
- **`test_external_loader_sanitize_truncates_overlong_names`**：断言日志或响应中的密钥、URL 口令、堆栈被脱敏。
- **`test_external_loader_sanitize_preserves_pure_chinese_name`**：断言日志或响应中的密钥、URL 口令、堆栈被脱敏。
- **`test_external_loader_sanitize_normalizes_mixed_unicode_name`**：断言日志或响应中的密钥、URL 口令、堆栈被脱敏。
- **`test_external_loader_sanitize_applies_safe_prefix_without_losing_unicode`**：断言日志或响应中的密钥、URL 口令、堆栈被脱敏。

#### `tests/backend/data/test_file_manager_encoding.py`（228 行，21 个 test_* ，类 0）

测试目标：验证 `backend/data/file_manager_encoding.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
- **`test_utf8_passthrough`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_ascii_passthrough`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_empty_content`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_utf8_bom_stripped`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_utf8_bom_stripped_for_txt`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_gbk_converted_to_utf8`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_gb18030_bmp_converted_to_utf8`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_gbk_txt_converted_to_utf8`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_excel_type_not_converted`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_json_type_not_converted`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_parquet_type_not_converted`**：断言 Arrow/Parquet 往返、dtype 与表名安全。
- **`test_gbk_multicolumn_csv`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_gbk_medium_content`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_gbk_large_content`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_shift_jis_with_halfwidth_katakana`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_pure_kanji_shift_jis_decoded_as_gbk_is_known_tradeoff`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_euckr_decoded_as_gbk_is_known_tradeoff`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_euckr_with_large_content`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_latin1_french_converted_to_utf8`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_latin1_german_decoded_as_gbk_is_known_tradeoff`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_windows1251_russian_converted_to_utf8`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。

#### `tests/backend/data/test_file_manager_table_names.py`（83 行，15 个 test_* ，类 0）

测试目标：验证 `backend/data/file_manager_table_names.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
- **`test_preserves_pure_chinese_name`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_preserves_japanese_name`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_preserves_korean_name`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_preserves_cyrillic_name`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_preserves_mixed_unicode_and_ascii`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_collapses_consecutive_underscores`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_normalizes_spaces_and_hyphens`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_normalizes_special_chars_to_single_underscore`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_strips_file_extension`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_empty_name_returns_unnamed`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_dotfile_name_treated_as_stem`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_only_special_chars_returns_unnamed`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_digit_prefix_gets_underscore`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_result_is_lowercase`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_no_leading_trailing_underscores`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。

#### `tests/backend/data/test_json_chinese_serialization.py`（216 行，17 个 test_* ，类 5）

测试目标：验证 `backend/data/json_chinese_serialization.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
测试类：TestJsonEnsureAsciiBasic、TestAgentMessageSerialization、TestStreamEventSerialization、TestWorkspaceStateSerialization、TestEdgeCases。
- **`test_non_ascii_string_preserved`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_ascii_only_string_unchanged`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_mixed_ascii_and_chinese`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_goal_with_chinese_description_should_be_readable`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_chart_spec_with_chinese_field_names`**：断言 visualize 契约、标题或编码。
- **`test_nested_chinese_values_all_preserved`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_ok_event_with_chinese_content`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_error_event_with_chinese_message`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。
- **`test_roundtrip_preserves_chinese`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_state_with_chinese_table_names`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_roundtrip_state_preserves_all_fields`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_default_str_handles_non_serializable_types`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_empty_string`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_none_value`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_emoji_preserved`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_mixed_languages`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_special_json_chars_still_escaped`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。

#### `tests/backend/data/test_loader_auth_paths.py`（78 行，5 个 test_* ，类 0）

测试目标：验证 `backend/data/loader_auth_paths.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
- **`test_multi_auth_loaders_expose_expected_paths`**：断言身份解析、登录回调、未授权拒绝或 token 刷新行为。
- **`test_auth_path_fields_do_not_overlap`**：断言身份解析、登录回调、未授权拒绝或 token 刷新行为。
- **`test_s3_default_credentials_do_not_require_access_keys`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_s3_access_key_path_requires_both_keys`**：断言路径监禁：越狱输入被拒绝，合法相对路径可解析。
- **`test_superset_only_exposes_sso_when_configured`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。

#### `tests/backend/data/test_local_folder_loader.py`（313 行，31 个 test_* ，类 2）

测试目标：验证 `backend/data/local_folder_loader.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
测试类：TestConfinedDir、TestLocalFolderDataLoader。
- **`test_resolve_valid_relative_path`**：断言路径监禁：越狱输入被拒绝，合法相对路径可解析。
- **`test_reject_absolute_path`**：断言路径监禁：越狱输入被拒绝，合法相对路径可解析。
- **`test_reject_dotdot_traversal`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_reject_dotdot_in_middle`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_reject_empty_path`**：断言路径监禁：越狱输入被拒绝，合法相对路径可解析。
- **`test_symlink_escape_rejected`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_write_creates_parents`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_resolve_with_mkdir_parents`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_repr`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_list_params`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_test_connection_valid_dir`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_test_connection_nonexistent_dir`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_list_tables_recursive`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_list_tables_non_recursive`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_list_tables_with_filter`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_list_tables_with_file_pattern`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_list_tables_path_hierarchy`**：断言路径监禁：越狱输入被拒绝，合法相对路径可解析。
- **`test_fetch_csv`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_fetch_tsv`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_fetch_parquet`**：断言 Arrow/Parquet 往返、dtype 与表名安全。
- **`test_fetch_jsonl`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_fetch_subdirectory_file`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_fetch_with_size_limit`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_fetch_path_traversal_rejected`**：断言路径监禁：越狱输入被拒绝，合法相对路径可解析。
- **`test_fetch_unsupported_type_rejected`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_metadata_parquet`**：断言 Arrow/Parquet 往返、dtype 与表名安全。
- **`test_metadata_csv`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_ls_root`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_ls_subdirectory`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_ls_with_filter`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_catalog_hierarchy`**：断言目录缓存、同步、搜索或进度 API 的正确性。

#### `tests/backend/data/test_max_import_rows.py`（68 行，7 个 test_* ，类 2）

测试目标：验证 `backend/data/max_import_rows.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
测试类：TestMaxImportRowsCap、TestLoaderImportsMaxImportRows。
- **`test_max_import_rows_constant_value`**：断言导入/显示行数上限被执行。
- **`test_size_over_limit_is_capped`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_size_under_limit_is_preserved`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_no_size_defaults_to_max`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_size_exactly_at_limit`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_size_negative_treated_as_given`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_loader_has_max_import_rows`**：断言导入/显示行数上限被执行。

#### `tests/backend/data/test_normalize_dtype.py`（97 行，8 个 test_* ，类 1）

测试目标：验证 `backend/data/normalize_dtype.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
测试类：TestNormalizeDtypeToAppType。
- **`test_datetime_types`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_date_types`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_time_types`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_duration_types`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_integer_types`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_float_types`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_boolean_types`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_string_fallback`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。

#### `tests/backend/data/test_parquet_utils_table_names.py`（32 行，5 个 test_* ，类 0）

测试目标：验证 `backend/data/parquet_utils_table_names.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
- **`test_parquet_sanitize_keeps_output_non_empty_for_dangerous_input`**：断言日志或响应中的密钥、URL 口令、堆栈被脱敏。
- **`test_parquet_sanitize_keeps_ascii_names_lowercase`**：断言日志或响应中的密钥、URL 口令、堆栈被脱敏。
- **`test_parquet_sanitize_preserves_pure_chinese_table_name`**：断言日志或响应中的密钥、URL 口令、堆栈被脱敏。
- **`test_parquet_sanitize_preserves_unicode_when_name_starts_with_digit`**：断言日志或响应中的密钥、URL 口令、堆栈被脱敏。
- **`test_parquet_sanitize_normalizes_separators_without_losing_unicode`**：断言日志或响应中的密钥、URL 口令、堆栈被脱敏。

#### `tests/backend/data/test_phase5_agent_metadata.py`（395 行，26 个 test_* ，类 7）

测试目标：验证 `backend/data/phase5_agent_metadata.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
测试类：TestWorkspaceSearch、TestCatalogCache、TestFieldSummaryWithDescription、TestSummarySystemDescription、TestCatalogMetadataLookups、TestMergeSourceMetadataEmptyClear、TestProgressiveContext。
- **`test_match_table_name`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_match_table_description`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_match_column_name`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_match_column_description`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_empty_query_returns_empty`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_no_match`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_respects_limit`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_save_and_load`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_load_nonexistent_returns_none`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_delete_catalog`**：断言目录缓存、同步、搜索或进度 API 的正确性。
- **`test_search_catalog_cache`**：断言目录缓存、同步、搜索或进度 API 的正确性。
- **`test_search_excludes_imported_tables`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_no_description`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_with_description`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_with_verbose_name`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_with_expression`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_with_all_metadata`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_only_source_description`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_uses_system_description_from_workspace_metadata`**：断言工作区创建、打开、迁移、过期或元数据原子更新。
- **`test_attached_metadata_is_ignored_if_present`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_lookups_use_workspace_user_home`**：断言工作区创建、打开、迁移、过期或元数据原子更新。
- **`test_lookups_graceful_without_user_home`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_empty_description_clears_existing`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_missing_key_preserves_existing`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_few_tables_includes_samples`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_many_tables_include_bounded_samples`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。

#### `tests/backend/data/test_plugin_scanner.py`（282 行，13 个 test_* ，类 0）

测试目标：验证 `backend/data/plugin_scanner.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
- **`test_scanner_disabled_in_hosted_mode`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_scanner_enabled_in_local_mode`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_scanner_opt_in_overrides_hosted_gate`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_missing_dependency_recorded_with_pip_hint`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_no_subclass_recorded_in_disabled`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_broken_plugin_does_not_leak_sys_modules`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_plugin_overriding_builtin_is_rejected`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_duplicate_plugin_keys_are_rejected`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_multiple_subclasses_registers_first_alphabetically`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_missing_plugin_dir_is_silent`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_empty_plugin_dir_is_silent`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_plugin_dir_defaults_to_data_formulator_home`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_df_plugin_dir_overrides_data_formulator_home`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。

#### `tests/backend/data/test_safe_data_filename.py`（28 行，3 个 test_* ，类 0）

测试目标：验证 `backend/data/safe_data_filename.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
- **`test_chinese_filename_preserves_unicode`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_path_traversal_strips_directory_components`**：断言路径监禁：越狱输入被拒绝，合法相对路径可解析。
- **`test_empty_filename_raises_valueerror`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。

#### `tests/backend/data/test_source_metadata.py`（532 行，35 个 test_* ，类 10）

测试目标：验证 `backend/data/source_metadata.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
测试类：TestColumnInfoDescription、TestTableMetadataColumnDescriptions、TestMergeSourceMetadata、TestIngestMetadataEnrichment、TestGetColumnTypesDefault、TestInferSourceMetadataStatus、TestCatalogTreeMetadataStatus、TestRefreshPreservesImportOptions、TestFormatImportOptions、TestSourceMetadataImportOptions。
- **`test_from_dict_without_description`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_from_dict_with_description`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_to_dict_omits_none_description`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_to_dict_includes_description_when_present`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_roundtrip_with_description`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_roundtrip_without_description`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_serialize_columns_with_descriptions`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_deserialize_old_metadata_no_column_description`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_merges_table_description`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_merges_column_descriptions`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_no_crash_on_empty_source_meta`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_no_crash_when_columns_none`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_empty_description_clears_existing`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_missing_key_preserves_existing`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_ingest_enriches_metadata_on_success`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_ingest_succeeds_when_metadata_fails`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_preserves_table_description`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_returns_empty_when_no_metadata`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_none_metadata_returns_unavailable`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_empty_metadata_returns_unavailable`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_columns_without_descriptions_returns_synced`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_columns_with_description_returns_synced`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_both_table_and_column_metadata_returns_synced`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_explicit_status_overrides_inference`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_no_columns_key_returns_partial_with_desc`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_empty_columns_returns_partial`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_tree_leaf_has_status_injected`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_explicit_status_preserved_in_tree`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_import_options_retained_after_refresh`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_empty_returns_empty`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_sort_only`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_filters_and_limit`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_full_options`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_import_options_in_response`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_import_options_none_when_absent`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。

#### `tests/backend/data/test_superset_catalog_sync.py`（362 行，18 个 test_* ，类 5）

测试目标：验证 `backend/data/superset_catalog_sync.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
测试类：TestGetDatasetColumns、TestListTablesUuidPassthrough、TestSupersetSyncCatalogMetadata、TestListTablesIncludesColumns、TestBuildColumnEntryExtra。
- **`test_calls_correct_endpoint`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_returns_empty_on_empty_result`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_fallback_to_detail_on_http_error`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。
- **`test_propagates_error_when_both_endpoints_fail`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。
- **`test_uuid_and_description_in_metadata`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_no_uuid_when_missing`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_enriches_columns_and_sets_table_key`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_column_fetch_failure_marks_unavailable`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_empty_columns_marks_partial`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_table_key_fallback_without_uuid`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_list_tables_includes_columns`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_list_tables_column_failure_marks_unavailable`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_certification_extra`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_warning_markdown_extra`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_empty_extra_no_effect`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_null_extra_no_effect`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_invalid_json_extra_no_crash`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_extra_combined_with_other_fields`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。

#### `tests/backend/data/test_superset_smart_filter.py`（499 行，52 个 test_* ，类 9）

测试目标：验证 `backend/data/superset_smart_filter.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
测试类：TestNormalizeColumnType、TestBuildChartDataFilters、TestBuildChartDataOrderby、TestGetColumnTypes、TestGetColumnValues、TestFetchDataAsArrow、TestSupersetURLResolution、TestValidateParams、TestClassifyConnectorError。
- **`test_classification`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_is_dttm_takes_precedence`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_empty_filters`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_eq_filter`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_neq_filter`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_comparison_operators`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_in_filter`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_not_in_filter`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_between_splits_into_two`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_is_null_filter`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_is_not_null_filter`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_like_filter`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_ilike_filter`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_invalid_operator_skipped`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_missing_column_skipped`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_in_with_empty_list_skipped`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_between_with_wrong_length_skipped`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_multiple_filters`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_non_dict_filter_skipped`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_empty_sort_columns`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_ascending`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_descending`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_defaults_to_ascending`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_multiple_columns`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_skips_invalid_columns`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_returns_normalized_types`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_invalid_source_table`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_api_failure_returns_empty`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_tier1_datasource_api`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_tier2_dataset_distinct`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_tier3_chart_data_fallback`**：断言 visualize 契约、标题或编码。
- **`test_invalid_source_table`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_keyword_filtering`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_has_more_pagination`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_limit_clamped`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_deduplication`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_all_tiers_fail_returns_empty`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_passes_sort_to_chart_data_api`**：断言 visualize 契约、标题或编码。
- **`test_passes_filters_and_sort`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_empty_result_returns_column_schema`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_empty_result_fallback_to_metadata`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_invalid_source_table_raises`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_url_from_params`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_url_from_env_fallback`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_url_missing_raises`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_sso_token_with_env_url`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_all_required_present_passes`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_missing_required_raises_with_names`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_skip_auth_tier_ignores_auth_params`**：断言身份解析、登录回调、未授权拒绝或 token 刷新行为。
- **`test_empty_string_treated_as_missing`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_connector_param_error_preserves_message`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。
- **`test_generic_required_error_passes_detail`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。

#### `tests/backend/data/test_sync_catalog_api.py`（316 行，7 个 test_* ，类 1）

测试目标：验证 `backend/data/sync_catalog_api.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
测试类：TestSyncCatalogMetadataEndpoint。
- **`test_connection`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_returns_tree_and_summary`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_partial_sync_returns_message_code`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_full_sync_returns_complete_message_code`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_timeout_returns_catalog_sync_timeout`**：断言目录缓存、同步、搜索或进度 API 的正确性。
- **`test_missing_connector_returns_error`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。
- **`test_sync_then_catalog_tree_preserves_column_metadata_in_cache`**：断言目录缓存、同步、搜索或进度 API 的正确性。

#### `tests/backend/data/test_sync_catalog_cross_db.py`（438 行，9 个 test_* ，类 2）

测试目标：验证 `backend/data/sync_catalog_cross_db.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
测试类：TestPostgreSQLSyncCatalogMetadata、TestMSSQLSyncCatalogMetadata。
- **`test_single_db_delegates_to_list_tables`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_multi_db_iterates_all_databases`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_multi_db_skips_failing_database`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_multi_db_table_filter_applied`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_single_db_delegates_to_list_tables`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_list_tables_groups_tables_and_views`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_ls_browses_tables_and_views_directories`**：断言导入/显示行数上限被执行。
- **`test_multi_db_iterates_all_databases`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_multi_db_skips_failing_database`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。

#### `tests/backend/data/test_table_name_contracts.py`（24 行，3 个 test_* ，类 0）

测试目标：验证 `backend/data/table_name_contracts.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
- **`test_route_sanitize_should_not_turn_pure_chinese_name_into_placeholder`**：断言日志或响应中的密钥、URL 口令、堆栈被脱敏。
- **`test_route_sanitize_should_keep_unicode_and_apply_safe_prefix_if_needed`**：断言日志或响应中的密钥、URL 口令、堆栈被脱敏。
- **`test_route_sanitize_should_delegate_to_parquet_sanitizer_for_ascii_name`**：断言日志或响应中的密钥、URL 口令、堆栈被脱敏。

#### `tests/backend/data/test_unicode_table_name_sanitization.py`（51 行，3 个 test_* ，类 0）

测试目标：验证 `backend/data/unicode_table_name_sanitization.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
- **`test_parquet_sanitize_table_name_should_preserve_unicode`**：断言日志或响应中的密钥、URL 口令、堆栈被脱敏。
- **`test_external_loader_sanitize_should_preserve_unicode`**：断言日志或响应中的密钥、URL 口令、堆栈被脱敏。
- **`test_sql_sanitize_should_preserve_unicode_identifiers`**：断言日志或响应中的密钥、URL 口令、堆栈被脱敏。

#### `tests/backend/data/test_workspace_fresh_names.py`（30 行，3 个 test_* ，类 0）

测试目标：验证 `backend/data/workspace_fresh_names.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
- **`test_workspace_get_fresh_name_appends_numeric_suffix_for_ascii_name`**：断言工作区创建、打开、迁移、过期或元数据原子更新。
- **`test_workspace_get_fresh_name_preserves_unicode_and_suffixes`**：断言工作区创建、打开、迁移、过期或元数据原子更新。
- **`test_workspace_get_fresh_name_applies_safe_prefix_without_losing_unicode`**：断言工作区创建、打开、迁移、过期或元数据原子更新。

#### `tests/backend/data/test_workspace_manager.py`（537 行，37 个 test_* ，类 6）

测试目标：验证 `backend/data/workspace_manager.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
测试类：TestWorkspaceLifecycle、TestSessionState、TestOpenWorkspace、TestWorkspaceMigrationOps、TestLegacyWorkspaceAutoRepair、TestEmptyWorkspaceVisibility。
- **`test_list_empty`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_create_workspace`**：断言工作区创建、打开、迁移、过期或元数据原子更新。
- **`test_create_duplicate_raises`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_list_workspaces`**：断言工作区创建、打开、迁移、过期或元数据原子更新。
- **`test_workspace_exists`**：断言工作区创建、打开、迁移、过期或元数据原子更新。
- **`test_workspace_exists_is_directory_based`**：断言工作区创建、打开、迁移、过期或元数据原子更新。
- **`test_delete_workspace`**：断言工作区创建、打开、迁移、过期或元数据原子更新。
- **`test_delete_nonexistent`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_rename_workspace`**：断言工作区创建、打开、迁移、过期或元数据原子更新。
- **`test_rename_nonexistent_raises`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_rename_to_existing_raises`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_save_and_load`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_sensitive_fields_stripped`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_load_nonexistent`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_update_display_name_patches_session_state`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_update_display_name_skips_missing_session_state`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_save_to_nonexistent_workspace_raises`**：断言工作区创建、打开、迁移、过期或元数据原子更新。
- **`test_overwrite_session_state`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_open_workspace_returns_workspace`**：断言工作区创建、打开、迁移、过期或元数据原子更新。
- **`test_open_nonexistent_raises`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_create_and_open_workspace`**：断言工作区创建、打开、迁移、过期或元数据原子更新。
- **`test_write_data_in_workspace`**：断言工作区创建、打开、迁移、过期或元数据原子更新。
- **`test_session_state_persists_with_data`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_move_workspaces_from_merges_existing_and_cleans_source`**：断言工作区创建、打开、迁移、过期或元数据原子更新。
- **`test_delete_all_workspaces_removes_dirs_and_files`**：断言工作区创建、打开、迁移、过期或元数据原子更新。
- **`test_move_workspaces_from_succeeds_when_source_locked`**：断言工作区创建、打开、迁移、过期或元数据原子更新。
- **`test_delete_all_workspaces_skips_locked_entries`**：断言工作区创建、打开、迁移、过期或元数据原子更新。
- **`test_legacy_workspace_with_only_yaml_appears_in_list`**：断言工作区创建、打开、迁移、过期或元数据原子更新。
- **`test_legacy_workspace_with_only_session_state_appears_in_list`**：断言工作区创建、打开、迁移、过期或元数据原子更新。
- **`test_legacy_workspace_with_empty_dir_appears_in_list`**：断言工作区创建、打开、迁移、过期或元数据原子更新。
- **`test_workspace_exists_consistent_with_create`**：断言工作区创建、打开、迁移、过期或元数据原子更新。
- **`test_move_legacy_workspace_auto_repairs_meta`**：断言工作区创建、打开、迁移、过期或元数据原子更新。
- **`test_provisional_workspace_is_hidden`**：断言工作区创建、打开、迁移、过期或元数据原子更新。
- **`test_naming_promotes_a_provisional_workspace`**：断言工作区创建、打开、迁移、过期或元数据原子更新。
- **`test_content_written_outside_save_is_still_listed`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_workspace_with_tables_is_visible`**：断言工作区创建、打开、迁移、过期或元数据原子更新。
- **`test_zero_count_workspace_is_visible`**：断言工作区创建、打开、迁移、过期或元数据原子更新。

#### `tests/backend/data/test_workspace_path_safety.py`（49 行，5 个 test_* ，类 1）

测试目标：验证 `backend/data/workspace_path_safety.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
测试类：TestWorkspacePathTraversal。
- **`test_normal_identity_succeeds`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_dotdot_identity_sanitized_safely`**：断言日志或响应中的密钥、URL 口令、堆栈被脱敏。
- **`test_slash_identity_sanitized_safely`**：断言日志或响应中的密钥、URL 口令、堆栈被脱敏。
- **`test_empty_identity_rejected`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_path_must_be_under_root`**：断言路径监禁：越狱输入被拒绝，合法相对路径可解析。

#### `tests/backend/data/test_workspace_scratch.py`（61 行，2 个 test_* ，类 0）

测试目标：验证 `backend/data/workspace_scratch.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
- **`test_prune_scratch_uses_backend_neutral_confined_directory`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_deleting_azure_workspace_removes_local_scratch`**：断言工作区创建、打开、迁移、过期或元数据原子更新。

#### `tests/backend/data/test_workspace_source_file_ops.py`（59 行，3 个 test_* ，类 0）

测试目标：验证 `backend/data/workspace_source_file_ops.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
- **`test_deletes_all_tables_from_matching_source`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_does_not_affect_other_source_files`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_returns_empty_when_no_match`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。

#### `tests/backend/data_loader/test_auth_paths.py`（42 行，3 个 test_* ，类 0）

测试目标：验证 `backend/data_loader/auth_paths.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
- **`test_mysql_declares_password_auth_path`**：断言身份解析、登录回调、未授权拒绝或 token 刷新行为。
- **`test_mysql_validation_materializes_defaults`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_mssql_sorted_fetch_places_order_by_after_top`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。

#### `tests/backend/data_loader/test_clickhouse_loader.py`（452 行，21 个 test_* ，类 0）

测试目标：验证 `backend/data_loader/clickhouse_loader.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
- **`test_static_contract_and_sensitive_password`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_constructor_maps_secure_read_only_options`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_constructor_normalizes_none_optional_parameters`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_constructor_preserves_falsy_non_none_parameters`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_connection_failure_redacts_password`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_omitted_port_follows_transport_default`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_fetch_enforces_row_cap_and_quotes_identifiers`**：断言导入/显示行数上限被执行。
- **`test_fetch_rejects_invalid_sizes`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_fetch_parameterizes_filters_and_quotes_sort_columns`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_fetch_rejects_raw_sql_and_dangerous_identifiers`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_list_tables_returns_arrow_metadata_and_excludes_catalog_details`**：断言目录缓存、同步、搜索或进度 API 的正确性。
- **`test_pinned_database_catalog_and_lazy_browsing`**：断言目录缓存、同步、搜索或进度 API 的正确性。
- **`test_probe_cannot_escape_pinned_database`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_get_metadata_returns_empty_when_a_query_fails`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_close_and_connection_check`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_cast_function_covers_lossy_types`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_fetch_projects_lossy_columns`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_fetch_projects_within_explicit_column_list`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_fetch_keeps_plain_star_without_lossy_columns`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_fetch_survives_column_type_lookup_failure`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_metadata_sample_projects_lossy_columns`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。

#### `tests/backend/data_loader/test_databricks_connection.py`（134 行，7 个 test_* ，类 0）

测试目标：验证 `backend/data_loader/databricks_connection.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
- **`test_token_is_default_auth_path_without_oauth`**：断言身份解析、登录回调、未授权拒绝或 token 刷新行为。
- **`test_sign_in_path_appears_when_oauth_configured`**：断言身份解析、登录回调、未授权拒绝或 token 刷新行为。
- **`test_token_path_requires_access_token`**：断言路径监禁：越狱输入被拒绝，合法相对路径可解析。
- **`test_hierarchy_is_catalog_schema_table`**：断言目录缓存、同步、搜索或进度 API 的正确性。
- **`test_effective_hierarchy_pins_provided_catalog`**：断言目录缓存、同步、搜索或进度 API 的正确性。
- **`test_resolve_source_table_variants`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_live_unity_catalog_roundtrip`**：断言目录缓存、同步、搜索或进度 API 的正确性。

#### `tests/backend/data_loader/test_kusto_connection.py`（178 行，11 个 test_* ，类 0）

测试目标：验证 `backend/data_loader/kusto_connection.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
- **`test_connection_uses_direct_sdk_probe`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_connection_returns_false_when_live_probe_fails`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_database_is_required`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_service_principal_path_requires_complete_credentials`**：断言路径监禁：越狱输入被拒绝，合法相对路径可解析。
- **`test_ambient_path_does_not_require_service_principal_fields`**：断言路径监禁：越狱输入被拒绝，合法相对路径可解析。
- **`test_microsoft_sign_in_is_default_when_oauth_is_configured`**：断言身份解析、登录回调、未授权拒绝或 token 刷新行为。
- **`test_ambient_is_default_when_oauth_is_not_configured`**：断言身份解析、登录回调、未授权拒绝或 token 刷新行为。
- **`test_connector_manifest_preserves_root_oauth_url`**：断言身份解析、登录回调、未授权拒绝或 token 刷新行为。
- **`test_delegated_credential_refreshes_expired_token`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_legacy_complete_service_principal_infers_path`**：断言路径监禁：越狱输入被拒绝，合法相对路径可解析。
- **`test_database_options_are_loaded_only_on_demand`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。

#### `tests/backend/data_loader/test_probe.py`（435 行，29 个 test_* ，类 7）

测试目标：验证 `backend/data_loader/probe.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
测试类：TestCompileProbeSql、TestProbeViaDuckDB、TestProbeUnavailable、TestProbeViaSql、TestSqlDialects、TestKustoKql、TestMongoPipeline。
- **`test_sample_projection`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_sample_all_columns`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_count`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_group_by_count_order`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_count_distinct`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_filter_applied`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_invalid_agg_op_raises`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_count_distinct_without_column_raises`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_count`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_distinct_values_with_frequency`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_filter_applied_locally_even_when_loader_ignores_it`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_date_range`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_sample_projection`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_output_capped_at_probe_max_rows`**：断言导入/显示行数上限被执行。
- **`test_scan_cap_marks_approximate`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_empty_path_errors`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。
- **`test_base_probe_reports_unavailable`**：断言报告流式字段或工具语法剥离。
- **`test_compiles_native_sql_and_returns_exact`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_filter_compiles_into_where`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_invalid_query_returns_error_without_executing`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。
- **`test_mssql_top_and_brackets`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_mysql_backtick_and_emulated_ilike`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_bigquery_backtick_path_relation`**：断言路径监禁：越狱输入被拒绝，合法相对路径可解析。
- **`test_summarize_by_pipeline`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_projection_and_take`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_invalid_agg_raises`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_group_pipeline_with_distinct`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_between_match`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_invalid_agg_raises`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。

#### `tests/backend/data_operations/test_executor.py`（169 行，5 个 test_* ，类 0）

测试目标：验证 `backend/data_operations/executor.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
- **`test_executor_materializes_bounded_table_with_provenance`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_executor_publishes_source_descriptions`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_executor_keeps_successful_tables_when_later_step_fails`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_executor_allocates_distinct_fresh_names`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_executor_recovers_published_tables_without_refetching`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。

#### `tests/backend/data_operations/test_models.py`（184 行，10 个 test_* ，类 0）

测试目标：验证 `backend/data_operations/models.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
- **`test_operation_round_trip_preserves_identity_and_status`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_operation_round_trip_preserves_discovery_presentation`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_plan_hash_depends_on_executable_steps_not_display_text`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_plan_rejects_tampered_serialized_hash`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_operation_rejects_unknown_selected_plan`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_models_are_immutable`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_load_query_rejects_multiple_order_clauses`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_nested_filter_values_cannot_change_after_hashing`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_action_factory_owns_the_versioned_wire_envelope`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_load_query_requires_positive_limit_and_valid_order`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。

#### `tests/backend/data_operations/test_repository.py`（186 行，10 个 test_* ，类 0）

测试目标：验证 `backend/data_operations/repository.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
- **`test_repository_round_trips_full_executable_operation`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_create_supersedes_only_same_conversation`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_select_is_idempotent_and_rejects_conflicts`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_interaction_response_returns_trusted_plan_label`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_elaborate_requires_an_awaiting_operation`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_execution_transitions_are_atomic_and_idempotent`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_failed_execution_records_typed_error`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。
- **`test_repository_lives_under_ephemeral_workspace_scratch`**：断言工作区创建、打开、迁移、过期或元数据原子更新。
- **`test_finish_records_partial_result`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_select_rejects_superseded_operation`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。

#### `tests/backend/errors/__init__.py`（1 行，0 个 test_* ，类 0）

测试目标：验证 `backend/errors/__init__.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
该文件可能是 fixture、benchmark 或辅助模块，不含 test_* 函数。

#### `tests/backend/errors/test_api_error_protocol_contract.py`（127 行，6 个 test_* ，类 2）

测试目标：验证 `backend/errors/api_error_protocol_contract.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
测试类：TestTablesErrorProtocol、TestStreamingErrorProtocol。
- **`test_download_db_file_returns_structured_business_error`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。
- **`test_export_csv_missing_table_returns_structured_error`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。
- **`test_export_csv_invalid_delimiter_returns_structured_error`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。
- **`test_db_error_classification_defaults_to_http_200`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。
- **`test_stream_preflight_error_uses_json_error_envelope`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。
- **`test_analyst_streaming_emits_top_level_type_events`**：断言 NDJSON 事件类型、预检错误或警告刷新。

#### `tests/backend/errors/test_error_handler.py`（509 行，51 个 test_* ，类 9）

测试目标：验证 `backend/errors/error_handler.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
测试类：TestClassifyAndWrapLlmError、TestStreamErrorEvent、TestRegisterErrorHandlers、TestRequestIdMiddleware、TestStreamWarningEvent、TestCollectAndFlushStreamWarnings、TestJsonOk、TestStreamPreflightError、TestErrorCodeHttpStatusMapping。
- **`test_auth_error`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。
- **`test_rate_limit`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_context_too_long`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_model_not_found`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_timeout`**：断言超时分类为可重试的 LLM/目录错误。
- **`test_service_error_502`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。
- **`test_content_filter`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_access_denied`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_bad_request`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_unknown_error`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。
- **`test_message_never_contains_original_exception`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_detail_contains_original_for_logging`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_output_is_valid_ndjson_line`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_error_structure`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。
- **`test_token_absent_from_error_event`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。
- **`test_raw_exception_wrapped`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_unicode_safe`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_app_error_returns_200_with_error_body`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。
- **`test_app_error_retryable`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。
- **`test_app_error_debug_mode_includes_detail`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。
- **`test_unexpected_error_returns_500`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。
- **`test_unexpected_error_debug_includes_safe_detail`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。
- **`test_413_returns_unified_format`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_api_404_returns_json`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_response_has_json_content_type`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_response_has_request_id_header`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_client_provided_request_id_is_echoed`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_request_id_is_uuid_when_not_provided`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_output_is_valid_ndjson_line`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_warning_structure`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_detail_included`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_message_code_included`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_unicode_safe`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_no_warnings_returns_empty`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_collect_then_flush`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_flush_clears_accumulator`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_outside_request_context_is_noop`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_basic_success`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_none_data`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_custom_status_code`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_top_level_body_is_strict`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_status_field_is_success_not_ok`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_always_returns_http_200`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_error_envelope_structure`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。
- **`test_content_type_is_json_not_ndjson`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_debug_mode_includes_detail`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_auth_codes_have_non_200_http_status`**：断言身份解析、登录回调、未授权拒绝或 token 刷新行为。
- **`test_auth_http_statuses_are_valid`**：断言身份解析、登录回调、未授权拒绝或 token 刷新行为。
- **`test_non_auth_app_error_returns_200`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。
- **`test_auth_app_error_returns_non_200`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。
- **`test_unknown_code_defaults_to_200`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。

#### `tests/backend/errors/test_errors.py`（163 行，14 个 test_* ，类 4）

测试目标：验证 `backend/errors/errors.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
测试类：TestErrorCode、TestAppErrorConstruction、TestAppErrorToDict、TestAppErrorSubclass。
- **`test_error_code_exists`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。
- **`test_code_values_are_strings`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_basic_construction`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_custom_status_code`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_detail_and_retry`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_inherits_from_exception`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_can_be_raised_and_caught`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_to_dict_without_detail`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_to_dict_excludes_detail_by_default`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_to_dict_includes_detail_when_requested`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_to_dict_include_detail_no_op_when_detail_is_none`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_to_dict_with_retry_true`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_subclass_preserves_interface`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_subclass_caught_as_app_error`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。

#### `tests/backend/knowledge/__init__.py`（1 行，0 个 test_* ，类 0）

测试目标：验证 `backend/knowledge/__init__.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
该文件可能是 fixture、benchmark 或辅助模块，不含 test_* 函数。

#### `tests/backend/knowledge/test_knowledge_store.py`（575 行，71 个 test_* ，类 12）

测试目标：验证 `backend/knowledge/knowledge_store.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
测试类：TestDataMemory、TestListAll、TestRead、TestWrite、TestDelete、TestValidatePath、TestFrontMatter、TestSearch、TestLoadAlwaysApplyRules、TestFormatRulesBlock、TestTokenizeQuery、TestMatchScore。
- **`test_read_creates_reserved_markdown_file`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_persists_across_store_instances_for_same_user`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_append_and_rewrite`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_replace_exact_text_and_delete`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_replace_rejects_missing_match`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_rejects_empty_append_and_oversized_rewrite`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_lists_rules`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_lists_workflows_in_subdirs`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_empty_category_returns_empty`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_front_matter_title_fallback_to_stem`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_reads_content`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_read_nonexistent_raises`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_creates_new_file`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_updates_existing_file`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_auto_adds_front_matter`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_preserves_existing_front_matter`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_writes_workflows_in_subdir`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_deletes_file`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_delete_nonexistent_raises`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_rules_flat_file_ok`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_rules_subdir_rejected`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_workflows_one_subdir_ok`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_workflows_two_subdirs_rejected`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_skills_rejected_as_invalid`**：断言技能注册、工具可见性或 propose 拒绝重复加载。
- **`test_non_md_extension_rejected`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_invalid_category_rejected`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_empty_path_rejected`**：断言路径监禁：越狱输入被拒绝，合法相对路径可解析。
- **`test_traversal_blocked_by_confined_dir`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_valid_front_matter`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_no_front_matter_degrades`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_invalid_yaml_degrades`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_search_by_title`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_search_by_filename`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_search_by_body`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_empty_query_returns_empty`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_no_match_returns_empty`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_max_results_limit`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_search_filters_by_category`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_search_skips_always_apply_rules`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_search_returns_non_always_apply_rules`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_title_match_ranks_higher`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_search_case_insensitive`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_partial_token_match_finds_results`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_table_names_boost`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_non_manual_source_discounted`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_loads_always_apply_rules`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_skips_non_always_apply`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_default_always_apply_is_true`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_empty_body_skipped`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_graceful_on_empty_store`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_auto_load_and_format`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_pre_loaded_rules`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_empty_returns_empty_string`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_empty_list_returns_empty_string`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_block_starts_with_newlines`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_english_basic`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_english_stopwords_filtered`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_short_ascii_filtered`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_pure_chinese_kept_as_single_token`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_mixed_cjk_ascii_split`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_mixed_cjk_ascii_with_spaces`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_underscore_preserved`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_empty_query`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_all_stopwords_returns_empty`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_single_token_title_hit`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_partial_tokens_accumulate`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_whole_string_bonus`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_source_discount`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_table_names_boost`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_no_match_returns_zero`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_cjk_mixed_query_matches`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。

#### `tests/backend/routes/__init__.py`（1 行，0 个 test_* ，类 0）

测试目标：验证 `backend/routes/__init__.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
该文件可能是 fixture、benchmark 或辅助模块，不含 test_* 函数。

#### `tests/backend/routes/test_agent_diagnostics_wiring.py`（80 行，3 个 test_* ，类 1）

测试目标：验证 `backend/routes/agent_diagnostics_wiring.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
测试类：TestDataLoadAgentWiring。
- **`test_run_attaches_diagnostics`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_run_parse_failure_still_has_diagnostics`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_init_backward_compatible_without_model_info`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。

#### `tests/backend/routes/test_analyst_data_operation_flow.py`（227 行，3 个 test_* ，类 0）

测试目标：验证 `backend/routes/analyst_data_operation_flow.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
- **`test_operation_preview_is_bounded_and_display_only`**：断言预览行数、缓存或替换语义。
- **`test_selected_operation_executes_without_model_turn`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_expired_operation_resumes_analyst_for_rediscovery`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。

#### `tests/backend/routes/test_create_table_replace_source.py`（122 行，3 个 test_* ，类 0）

测试目标：验证 `backend/routes/create_table_replace_source.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
- **`test_overwrite_same_table_name`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_replace_source_removes_old_tables`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_fewer_sheets_after_replace_no_orphans`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。

#### `tests/backend/routes/test_create_table_xls_upload.py`（176 行，5 个 test_* ，类 0）

测试目标：验证 `backend/routes/create_table_xls_upload.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
- **`test_upload_xls_creates_table_and_returns_columns`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_upload_xls_preserves_chinese_column_names`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_upload_xls_table_name_sanitized_for_unicode`**：断言日志或响应中的密钥、URL 口令、堆栈被脱敏。
- **`test_upload_xls_rejects_missing_table_name`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_list_tables_returns_sample_rows_for_xls`**：断言导入/显示行数上限被执行。

#### `tests/backend/routes/test_credential_routes.py`（192 行，11 个 test_* ，类 4）

测试目标：验证 `backend/routes/credential_routes.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
测试类：TestStoreEndpoint、TestListEndpoint、TestDeleteEndpoint、TestUserIsolation。
- **`test_store_success`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_store_missing_fields`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_store_no_vault_returns_error`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。
- **`test_list_empty`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_list_after_store`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_list_no_vault_returns_empty`**：断言凭证加解密、按用户隔离、列表不返回明文。
- **`test_delete_success`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_delete_missing_key`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_delete_no_vault_returns_error`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。
- **`test_different_users_see_own_credentials`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_delete_only_affects_own`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。

#### `tests/backend/routes/test_credentials_contract.py`（125 行，8 个 test_* ，类 3）

测试目标：验证 `backend/routes/credentials_contract.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
测试类：TestListCredentials、TestStoreCredential、TestDeleteCredential。
- **`test_no_vault_returns_empty_sources`**：断言凭证加解密、按用户隔离、列表不返回明文。
- **`test_vault_returns_sources`**：断言凭证加解密、按用户隔离、列表不返回明文。
- **`test_no_vault_returns_service_unavailable`**：断言凭证加解密、按用户隔离、列表不返回明文。
- **`test_missing_fields_returns_invalid_request`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_success_returns_source_key`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_no_vault_returns_service_unavailable`**：断言凭证加解密、按用户隔离、列表不返回明文。
- **`test_missing_source_key_returns_invalid_request`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_success_returns_source_key`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。

#### `tests/backend/routes/test_csv_encoding_roundtrip.py`（155 行，7 个 test_* ，类 3）

测试目标：验证 `backend/routes/csv_encoding_roundtrip.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
测试类：TestWorkspaceRoundTrip、TestParseFileEndpoint、TestCreateAndGetTable。
- **`test_gbk_csv_columns_are_chinese`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_gbk_csv_cell_values_are_correct`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_utf8_csv_still_works`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_utf8_bom_csv_columns_correct`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_gbk_csv_parse_returns_chinese_columns`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_gbk_csv_parse_returns_correct_rows`**：断言导入/显示行数上限被执行。
- **`test_gbk_upload_then_get_returns_chinese`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。

#### `tests/backend/routes/test_data_loaders_discovery.py`（215 行，5 个 test_* ，类 0）

测试目标：验证 `backend/routes/data_loaders_discovery.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
- **`test_plugin_appears_in_discovery_endpoint`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_builtin_loader_marked_as_builtin`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_mysql_auth_path_surfaces_in_discovery`**：断言身份解析、登录回调、未授权拒绝或 token 刷新行为。
- **`test_display_name_default_titlecases_registry_key`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_plugins_block_surfaces_loaded_and_rejected`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。

#### `tests/backend/routes/test_data_loading_chat_route.py`（159 行，4 个 test_* ，类 3）

测试目标：验证 `backend/routes/data_loading_chat_route.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
测试类：TestDataLoadingChatValidation、TestDataLoadingChatSuccess、TestDataLoadingChatErrors。
- **`test_non_json_request_returns_error`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。
- **`test_messages_forwarded_to_agent`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_image_messages_are_forwarded`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_agent_exception_streams_error_event`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。

#### `tests/backend/routes/test_knowledge_contract.py`（126 行，9 个 test_* ，类 4）

测试目标：验证 `backend/routes/knowledge_contract.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
测试类：TestKnowledgeLimits、TestKnowledgeList、TestKnowledgeRead、TestKnowledgeSearch。
- **`test_returns_limits_in_success_envelope`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_missing_category_returns_error`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。
- **`test_invalid_category_returns_error`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。
- **`test_success_returns_items`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_missing_fields_returns_error`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。
- **`test_file_not_found_returns_error`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。
- **`test_success_returns_content`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_success_returns_results`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_invalid_categories_type_returns_error`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。

#### `tests/backend/routes/test_knowledge_routes.py`（456 行，28 个 test_* ，类 7）

测试目标：验证 `backend/routes/knowledge_routes.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
测试类：TestDataMemory、TestKnowledgeList、TestKnowledgeRead、TestKnowledgeWrite、TestKnowledgeDelete、TestKnowledgeSearch、TestDistillWorkflow。
- **`test_read_creates_user_memory`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_append_then_rewrite`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_rejects_non_string_content`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_list_empty`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_list_with_entries`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_list_invalid_category`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_list_missing_category`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_read_existing`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_read_nonexistent`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_read_traversal_rejected`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_write_creates_file`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_write_updates_file`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_write_traversal_rejected`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_write_non_md_rejected`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_delete_existing`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_delete_nonexistent`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_search_returns_results`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_search_empty_query`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_search_invalid_category`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_search_filters_by_category`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_distill_workflow_from_context`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_distill_workflow_llm_timeout_returns_structured_error`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。
- **`test_distill_workflow_missing_context`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_distill_workflow_missing_threads`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_distill_workflow_missing_workspace`**：断言工作区创建、打开、迁移、过期或元数据原子更新。
- **`test_distill_session_uses_descriptive_title`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_distill_session_upserts_existing_workspace_file`**：断言工作区创建、打开、迁移、过期或元数据原子更新。
- **`test_distill_session_strips_legacy_title_prefix`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。

#### `tests/backend/routes/test_list_global_models_api.py`（91 行，5 个 test_* ，类 1）

测试目标：验证 `backend/routes/list_global_models_api.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
测试类：TestListGlobalModelsEndpoint。
- **`test_returns_all_configured_models`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_response_has_required_fields`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_no_api_key_in_response`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_all_models_marked_global`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_empty_env_returns_empty_list`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。

#### `tests/backend/routes/test_parse_file_endpoint.py`（89 行，4 个 test_* ，类 0）

测试目标：验证 `backend/routes/parse_file_endpoint.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
- **`test_parse_xls_returns_sheet_data`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_parse_file_rejects_missing_file`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_parse_file_rejects_unsupported_format`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_parse_csv_via_endpoint`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。

#### `tests/backend/routes/test_same_basename_upload.py`（273 行，7 个 test_* ，类 3）

测试目标：验证 `backend/routes/same_basename_upload.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
测试类：TestPreviewIdVsWorkspaceIdMismatch、TestSameBasenameDifferentExtension、TestFileArrayMismatch。
- **`test_preview_id_differs_from_workspace_name`**：断言工作区创建、打开、迁移、过期或元数据原子更新。
- **`test_orphan_cleanup_would_wrongly_remove_on_reupload`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_both_tables_exist_after_upload`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_both_readable_after_upload`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_reupload_csv_does_not_destroy_other_table`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_replace_source_only_affects_same_source_file`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_index_fallback_sends_wrong_file`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。

#### `tests/backend/routes/test_sample_table_pagination.py`（135 行，8 个 test_* ，类 1）

测试目标：验证 `backend/routes/sample_table_pagination.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
测试类：TestSampleTablePagination。
- **`test_no_offset_returns_first_page`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_offset_skips_rows`**：断言导入/显示行数上限被执行。
- **`test_offset_with_desc_sort`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_offset_with_desc_sort_page2`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_offset_beyond_total_returns_empty`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_total_row_count_consistent_across_pages`**：断言导入/显示行数上限被执行。
- **`test_backward_compat_no_offset`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_pages_cover_all_rows_without_overlap`**：断言导入/显示行数上限被执行。

#### `tests/backend/routes/test_session_export_import.py`（204 行，8 个 test_* ，类 2）

测试目标：验证 `backend/routes/session_export_import.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
测试类：TestExportSession、TestImportSession。
- **`test_export_uses_workspace_id_from_body`**：断言工作区创建、打开、迁移、过期或元数据原子更新。
- **`test_export_rejects_missing_workspace_id`**：断言工作区创建、打开、迁移、过期或元数据原子更新。
- **`test_export_rejects_missing_state`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_export_returns_error_for_unknown_workspace`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。
- **`test_import_creates_workspace_when_not_existing`**：断言工作区创建、打开、迁移、过期或元数据原子更新。
- **`test_import_opens_existing_workspace`**：断言工作区创建、打开、迁移、过期或元数据原子更新。
- **`test_import_falls_back_to_active_workspace`**：断言工作区创建、打开、迁移、过期或元数据原子更新。
- **`test_import_rejects_missing_file`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。

#### `tests/backend/routes/test_session_routes_migration.py`（124 行，5 个 test_* ，类 3）

测试目标：验证 `backend/routes/session_routes_migration.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
测试类：TestSaveSessionRoute、TestMigrateRoute、TestCleanupAnonymousRoute。
- **`test_save_session_reports_storage_full`**：断言报告流式字段或工具语法剥离。
- **`test_migrate_moves_and_cleans_source`**：断言状态或身份迁移后数据仍完整。
- **`test_migrate_rejects_non_user`**：断言状态或身份迁移后数据仍完整。
- **`test_cleanup_success`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_cleanup_rejects_non_user`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。

#### `tests/backend/routes/test_upload_parquet_conversion.py`（324 行，17 个 test_* ，类 6）

测试目标：验证 `backend/routes/upload_parquet_conversion.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
测试类：TestParquetConversion、TestMultiSheetEnglish、TestMultiSheetChinese、TestMultiSheetMixed、TestReplaceSourceWithParquet、TestMetadataIntegrity。
- **`test_csv_upload_stored_as_parquet`**：断言 Arrow/Parquet 往返、dtype 与表名安全。
- **`test_xlsx_upload_stored_as_parquet`**：断言 Arrow/Parquet 往返、dtype 与表名安全。
- **`test_gbk_csv_converted_correctly`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_upload_orders_sheet_via_suffix`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_upload_returns_sheet_via_suffix`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_upload_both_sheets_coexist`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_sheet_hint_overrides_inference`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_invalid_sheet_hint_falls_back`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_upload_sales_sheet`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_upload_profit_sheet`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_chinese_sheet_hint`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_upload_summary_sheet`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_upload_detail_sheet_chinese`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_upload_q1_sheet`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_load_all_three_sheets`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_replace_source_removes_parquet_tables`**：断言 Arrow/Parquet 往返、dtype 与表名安全。
- **`test_source_file_and_original_name_recorded`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。

#### `tests/backend/routes/test_workspace_name_api.py`（86 行，2 个 test_* ，类 1）

测试目标：验证 `backend/routes/workspace_name_api.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
测试类：TestWorkspaceNameEndpoint。
- **`test_workspace_name_uses_selected_model_payload`**：断言工作区创建、打开、迁移、过期或元数据原子更新。
- **`test_workspace_name_requires_model`**：断言工作区创建、打开、迁移、过期或元数据原子更新。

#### `tests/backend/security/test_code_signing.py`（94 行，13 个 test_* ，类 2）

测试目标：验证 `backend/security/code_signing.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
测试类：TestSignVerifyRoundTrip、TestSignResult。
- **`test_valid_signature_accepted`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_tampered_code_rejected`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_tampered_signature_rejected`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_empty_code_returns_empty_sig`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_empty_code_verify_returns_false`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_empty_signature_verify_returns_false`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_whitespace_matters`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_unicode_code`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_signature_is_hex_string`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_adds_signature_when_code_present`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_no_signature_when_code_empty`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_no_signature_when_code_missing`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_returns_result_for_chaining`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。

#### `tests/backend/security/test_confined_dir_extended.py`（176 行，26 个 test_* ，类 7）

测试目标：验证 `backend/security/confined_dir_extended.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
测试类：TestReadText、TestWriteText、TestExists、TestIterdir、TestRglob、TestUnlink、TestExistingApiRegression。
- **`test_reads_existing_file`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_reads_utf8_by_default`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_traversal_raises`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_nonexistent_file_raises`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_creates_new_file`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_creates_parent_dirs`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_returns_resolved_path`**：断言路径监禁：越狱输入被拒绝，合法相对路径可解析。
- **`test_traversal_raises`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_existing_file_returns_true`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_missing_file_returns_false`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_traversal_returns_false`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_directory_returns_true`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_lists_root_contents`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_lists_subdirectory`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_empty_directory_returns_empty`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_traversal_raises`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_finds_matching_files`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_no_match_returns_empty`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_rglob_in_subdirectory`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_deletes_existing_file`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_traversal_raises`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_nonexistent_raises`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_resolve_normal`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_resolve_traversal_raises`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_write_bytes`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_truediv_operator`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。

#### `tests/backend/security/test_confined_dir_migration.py`（247 行，23 个 test_* ，类 3）

测试目标：验证 `backend/security/confined_dir_migration.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
测试类：TestWorkspaceConfinedProperties、TestAgentToolsUseConfinedProperties、TestScratchRoutesConfinedMigration。
- **`test_confined_root_is_confineddir`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_confined_data_is_confineddir`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_confined_scratch_is_confineddir`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_confined_root_points_to_workspace_path`**：断言路径监禁：越狱输入被拒绝，合法相对路径可解析。
- **`test_confined_data_points_to_data_subdir`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_confined_scratch_points_to_scratch_subdir`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_confined_root_rejects_traversal`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_confined_data_rejects_traversal`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_confined_scratch_rejects_traversal`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_get_file_path_uses_confined_data`**：断言路径监禁：越狱输入被拒绝，合法相对路径可解析。
- **`test_get_file_path_traversal_sanitized`**：断言路径监禁：越狱输入被拒绝，合法相对路径可解析。
- **`test_data_dir_created`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_scratch_dir_created`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_read_file_traversal_blocked`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_write_file_traversal_blocked`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_list_directory_traversal_blocked`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_preview_scratch_traversal_blocked`**：断言预览行数、缓存或替换语义。
- **`test_scratch_serve_normal`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_scratch_serve_traversal_rejected`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_scratch_serve_nonexistent`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_scratch_upload_normal`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_scratch_upload_traversal_sanitized`**：断言日志或响应中的密钥、URL 口令、堆栈被脱敏。
- **`test_scratch_upload_no_file_returns_error`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。

#### `tests/backend/security/test_docker_sandbox_path.py`（147 行，7 个 test_* ，类 1）

测试目标：验证 `backend/security/docker_sandbox_path.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
测试类：TestOutputPathValidation。
- **`test_traversal_via_slashes_neutralized`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_traversal_via_dotdot_neutralized`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_empty_output_variable_rejected`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_normal_output_variable_not_rejected`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_output_variable_with_separator`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_docker_stderr_returns_sanitized_diagnostics`**：断言日志或响应中的密钥、URL 口令、堆栈被脱敏。
- **`test_start_exception_returns_generic_content`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。

#### `tests/backend/security/test_global_model_security.py`（265 行，17 个 test_* ，类 3）

测试目标：验证 `backend/security/global_model_security.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
测试类：TestGetClientGlobalResolution、TestSharedErrorSanitization、TestClassifyLlmError。
- **`test_global_model_gets_real_api_key`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_user_model_keeps_own_credentials`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_global_claim_for_unregistered_id_is_rejected`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_global_claim_cannot_bypass_the_api_base_allowlist`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_user_model_api_base_is_still_validated`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_resolving_a_global_model_does_not_mutate_the_registry`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_sanitize_redacts_api_key_patterns`**：断言日志或响应中的密钥、URL 口令、堆栈被脱敏。
- **`test_sanitize_truncates_long_messages`**：断言日志或响应中的密钥、URL 口令、堆栈被脱敏。
- **`test_sanitize_escapes_html`**：断言日志或响应中的密钥、URL 口令、堆栈被脱敏。
- **`test_auth_error_401`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。
- **`test_auth_error_invalid_key`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。
- **`test_rate_limit_429`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_context_length`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_model_not_found`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_timeout`**：断言超时分类为可重试的 LLM/目录错误。
- **`test_unknown_error_generic_fallback`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。
- **`test_never_includes_raw_exception_text`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。

#### `tests/backend/security/test_local_folder_deployment.py`（74 行，4 个 test_* ，类 1）

测试目标：验证 `backend/security/local_folder_deployment.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
测试类：TestLocalFolderDeploymentRestriction。
- **`test_local_mode_keeps_local_folder`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_multi_user_mode_disables_local_folder`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_ephemeral_mode_disables_local_folder`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_create_connector_rejects_disabled_type`**：断言连接器 CRUD、可见性隔离或表单字段。

#### `tests/backend/security/test_log_sanitizer.py`（324 行，41 个 test_* ，类 5）

测试目标：验证 `backend/security/log_sanitizer.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
测试类：TestSanitizeUrl、TestSanitizeParams、TestRedactToken、TestApplyPatterns、TestSensitiveDataFilter。
- **`test_url_with_password`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_url_without_credentials`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_url_with_port`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_empty_string`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_non_url_string`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_s3_url_no_creds`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_multiple_urls_in_string`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_url_with_sensitive_query_params`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_url_without_sensitive_query_params_preserves_query`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_masks_password`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_masks_multiple_keys`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_case_insensitive`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_nested_dict`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_does_not_mutate_original`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_extra_keys`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_empty_dict`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_connection_string`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_long_token`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_short_token_fully_masked`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_empty_token`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_custom_visible`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_boundary_length`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_url_credentials`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_url_query_credentials`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_key_value_password`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_key_value_api_key`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_key_value_with_colon`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_bearer_token`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_bare_jwt`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_dict_repr_with_password`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_dict_repr_double_quotes`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_no_false_positive`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_no_false_positive_on_normal_url`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_mixed_patterns`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_filter_redacts_password_in_message`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_filter_redacts_url_creds`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_filter_percent_style_args`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_filter_disabled_by_env`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_filter_always_returns_true`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_filter_handles_bad_format_args`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_filter_sanitizes_exc_text`**：断言日志或响应中的密钥、URL 口令、堆栈被脱敏。

#### `tests/backend/security/test_sandbox.py`（516 行，34 个 test_* ，类 6）

测试目标：验证 `backend/security/sandbox.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
测试类：TestCreateSandbox、TestLocalSandbox、TestDockerSandbox、TestDockerSandboxNoDocker、TestSandboxSession、TestSandboxSessionSaveRestore。
- **`test_local`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_docker`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_default_is_local`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_simple_transform`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_derived_columns`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_read_csv_from_workspace`**：断言工作区创建、打开、迁移、过期或元数据原子更新。
- **`test_read_parquet_from_workspace`**：断言工作区创建、打开、迁移、过期或元数据原子更新。
- **`test_parquet_modules_preimported`**：断言 Arrow/Parquet 往返、dtype 与表名安全。
- **`test_write_to_workdir_blocked`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_syntax_error`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。
- **`test_runtime_error`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。
- **`test_non_dataframe_output`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_duckdb_sql`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_simple_transform`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_derived_columns`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_read_csv_from_workspace`**：断言工作区创建、打开、迁移、过期或元数据原子更新。
- **`test_syntax_error`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。
- **`test_runtime_error`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。
- **`test_non_dataframe_output`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_json_usage`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_duckdb_sql`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_missing_docker_returns_error`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。
- **`test_variable_persists_across_calls`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_dataframe_persists`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_close_clears_namespace`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_context_manager`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_error_does_not_break_session`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。
- **`test_backward_compat_3tuple`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_save_and_restore_dataframe`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_save_scalars_only`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_save_empty_namespace_returns_false`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_restore_missing_dir_returns_false`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_restore_does_not_clobber_existing_vars`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_multiple_dataframes`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。

#### `tests/backend/security/test_sandbox_security.py`（158 行，10 个 test_* ，类 3）

测试目标：验证 `backend/security/sandbox_security.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
测试类：TestLocalSandboxFileWriteBlocked、TestLocalSandboxProcessExecBlocked、TestDockerSandboxSecurity。
- **`test_open_write_blocked`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_csv_write_blocked`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_os_system`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_os_popen`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_os_execvp`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_os_spawnlp`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_os_kill`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_os_via_sys_modules`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_os_putenv`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_workspace_readonly`**：断言工作区创建、打开、迁移、过期或元数据原子更新。

#### `tests/backend/security/test_sanitize.py`（126 行，16 个 test_* ，类 6）

测试目标：验证 `backend/security/sanitize.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
测试类：TestApiKeyRedaction、TestPathRedaction、TestStackTraceStripping、TestHtmlEscaping、TestTruncation、TestEdgeCases。
- **`test_api_key_equals`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_api_token_equals`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_generic_token_equals`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_password_equals`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_no_false_positive_on_normal_text`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_unix_home_path`**：断言路径监禁：越狱输入被拒绝，合法相对路径可解析。
- **`test_unix_opt_path`**：断言路径监禁：越狱输入被拒绝，合法相对路径可解析。
- **`test_windows_path`**：断言路径监禁：越狱输入被拒绝，合法相对路径可解析。
- **`test_tmp_path`**：断言路径监禁：越狱输入被拒绝，合法相对路径可解析。
- **`test_traceback_removed`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_file_line_references_stripped`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_script_tag_escaped`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_long_message_truncated`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_short_message_not_truncated`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_empty_string`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_unicode_error`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。

#### `tests/backend/security/test_scratch_serve.py`（81 行，5 个 test_* ，类 1）

测试目标：验证 `backend/security/scratch_serve.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
测试类：TestScratchServePathSafety。
- **`test_normal_file_served`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_path_traversal_returns_error`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。
- **`test_dotdot_single_returns_error`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。
- **`test_nonexistent_file_returns_error`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。
- **`test_response_uses_send_file_not_send_from_directory`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。

#### `tests/backend/security/test_startup_safety.py`（67 行，3 个 test_* ，类 1）

测试目标：验证 `backend/security/startup_safety.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
测试类：TestStartupSafetyChecks。
- **`test_multi_user_no_sandbox_logs_critical`**：断言代码在隔离环境执行，危险操作被拦截或容器路径安全。
- **`test_local_mode_no_warning`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_multi_user_with_docker_sandbox_no_warning`**：断言代码在隔离环境执行，危险操作被拦截或容器路径安全。

#### `tests/backend/security/test_superset_bridge_security.py`（108 行，4 个 test_* ，类 2）

测试目标：验证 `backend/security/superset_bridge_security.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
测试类：TestSupersetBridgeOriginValidation、TestSupersetBridgeTemplate。
- **`test_allows_known_data_formulator_origins`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_rejects_untrusted_or_non_origin_values`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_allows_env_configured_origin`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_script_payload_escapes_script_breakout`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。

#### `tests/backend/security/test_url_allowlist.py`（178 行，26 个 test_* ，类 6）

测试目标：验证 `backend/security/url_allowlist.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
测试类：TestOpenMode、TestEnforceMode、TestEmptyBaseAlwaysAllowed、TestCaseInsensitive、TestPatternLoading、TestGlobEdgeCases。
- **`test_any_url_allowed`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_private_ip_allowed_in_open_mode`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_localhost_allowed_in_open_mode`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_empty_base_allowed`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_none_base_allowed`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_openai_allowed`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_azure_wildcard_allowed`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_ollama_localhost_allowed`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_unlisted_url_rejected`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_private_ip_rejected`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_internal_network_rejected`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_localhost_wrong_port_rejected`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_none_allowed_in_enforce_mode`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_empty_string_allowed_in_enforce_mode`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_uppercase_url_matches`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_uppercase_pattern_matches`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_unset_returns_none`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_empty_string_returns_none`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_whitespace_only_returns_none`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_comma_separated_parsed`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_whitespace_trimmed`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_subdomain_wildcard`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_deep_subdomain_wildcard`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_no_path_still_matches_with_slash`**：断言路径监禁：越狱输入被拒绝，合法相对路径可解析。
- **`test_exact_domain_no_trailing_slash_no_match`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_pattern_without_slash_star_matches_bare_domain`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。

#### `tests/backend/test_desktop_single_instance.py`（70 行，3 个 test_* ，类 0）

测试目标：验证 `backend/desktop_single_instance.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
- **`test_second_instance_signals_primary`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_unrelated_port_occupant_is_not_treated_as_existing_instance`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_activate_window_restores_and_shows_window`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。

#### `tests/backend/test_model_endpoints.py`（96 行，5 个 test_* ，类 0）

测试目标：验证 `backend/model_endpoints.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
- **`test_sanitize_entry_keeps_only_non_secret_fields`**：断言日志或响应中的密钥、URL 口令、堆栈被脱敏。
- **`test_history_round_trip_and_deduplication`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_invalid_history_is_treated_as_empty`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_history_file_contains_no_unrecognized_fields`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_api_isolates_history_by_identity_and_drops_keys`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。

#### `tests/backend/test_startup_spinner.py`（56 行，2 个 test_* ，类 0）

测试目标：验证 `backend/startup_spinner.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
- **`test_spinner_output_is_cp1252_safe`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_desktop_standard_streams_replace_unencodable_output`**：断言 NDJSON 事件类型、预检错误或警告刷新。

#### `tests/conftest.py`（40 行，0 个 test_* ，类 0）

测试目标：验证 `conftest.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
该文件可能是 fixture、benchmark 或辅助模块，不含 test_* 函数。

#### `tests/database-dockers/bigquery/test_bigquery_loader.py`（365 行，15 个 test_* ，类 3）

测试目标：验证 `database-dockers/bigquery/bigquery_loader.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
测试类：TestBigQueryEmulatorProbe、TestBigQueryDataLoader、TestBigQueryDataLoaderStatic。
- **`test_bq_emulator_available_false_quickly_on_closed_port`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_list_tables`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_list_tables_with_filter`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_list_tables_specific_dataset`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_fetch_data_as_arrow_from_table`**：断言导入/显示行数上限被执行。
- **`test_fetch_data_respects_size`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_fetch_data_invalid_table_raises`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_ingest_table_to_workspace`**：断言工作区创建、打开、迁移、过期或元数据原子更新。
- **`test_ingest_table_auto_name`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_ingest_products_table`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_ingest_orders_table`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_ingest_sanitizes_table_name`**：断言日志或响应中的密钥、URL 口令、堆栈被脱敏。
- **`test_get_table_info_from_datalake`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_list_params`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_auth_instructions`**：断言身份解析、登录回调、未授权拒绝或 token 刷新行为。

#### `tests/database-dockers/cosmosdb/seed_data.py`（146 行，0 个 test_* ，类 0）

测试目标：验证 `database-dockers/cosmosdb/seed_data.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
该文件可能是 fixture、benchmark 或辅助模块，不含 test_* 函数。

#### `tests/database-dockers/cosmosdb/test_cosmosdb_loader.py`（336 行，22 个 test_* ，类 2）

测试目标：验证 `database-dockers/cosmosdb/cosmosdb_loader.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
测试类：TestCosmosDBDataLoader、TestCosmosDBDataLoaderStatic。
- **`test_list_tables`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_list_tables_with_filter`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_list_tables_specific_container`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_list_tables_row_count`**：断言导入/显示行数上限被执行。
- **`test_fetch_data_as_arrow`**：断言导入/显示行数上限被执行。
- **`test_fetch_data_respects_size`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_ingest_table_to_workspace`**：断言工作区创建、打开、迁移、过期或元数据原子更新。
- **`test_ingest_nested_documents_flattened`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_ingest_with_arrays_flattened`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_ingest_sanitizes_table_name`**：断言日志或响应中的密钥、URL 口令、堆栈被脱敏。
- **`test_get_table_info_from_datalake`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_connection_close`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_context_manager`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_flatten_document`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_convert_special_types`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_catalog_hierarchy`**：断言目录缓存、同步、搜索或进度 API 的正确性。
- **`test_ls_containers`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_test_connection`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_get_metadata`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_list_params`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_auth_instructions`**：断言身份解析、登录回调、未授权拒绝或 token 刷新行为。
- **`test_flatten_strips_cosmos_metadata`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。

#### `tests/database-dockers/mongodb/test_mongodb_loader.py`（282 行，17 个 test_* ，类 2）

测试目标：验证 `database-dockers/mongodb/mongodb_loader.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
测试类：TestMongoDBDataLoader、TestMongoDBDataLoaderStatic。
- **`test_list_tables`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_list_tables_with_filter`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_list_tables_specific_collection`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_list_tables_row_count`**：断言导入/显示行数上限被执行。
- **`test_fetch_data_as_arrow`**：断言导入/显示行数上限被执行。
- **`test_fetch_data_respects_size`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_ingest_table_to_workspace`**：断言工作区创建、打开、迁移、过期或元数据原子更新。
- **`test_ingest_nested_documents_flattened`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_ingest_with_arrays_flattened`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_ingest_sanitizes_table_name`**：断言日志或响应中的密钥、URL 口令、堆栈被脱敏。
- **`test_get_table_info_from_datalake`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_connection_close`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_context_manager`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_flatten_document`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_convert_special_types`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_list_params`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_auth_instructions`**：断言身份解析、登录回调、未授权拒绝或 token 刷新行为。

#### `tests/database-dockers/mysql/test_mysql_datalake.py`（149 行，3 个 test_* ，类 1）

测试目标：验证 `database-dockers/mysql/mysql_datalake.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
测试类：TestMySQLDataLake。
- **`test_connect_and_list_tables`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_ingest_table_into_datalake`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_get_table_info_from_datalake`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。

#### `tests/database-dockers/mysql/test_mysql_loader.py`（225 行，10 个 test_* ，类 2）

测试目标：验证 `database-dockers/mysql/mysql_loader.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
测试类：TestMySQLDataLoader、TestMySQLDataLoaderStatic。
- **`test_list_tables`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_list_tables_with_filter`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_fetch_data_as_arrow_from_table`**：断言导入/显示行数上限被执行。
- **`test_fetch_data_respects_size`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_ingest_table_to_workspace`**：断言工作区创建、打开、迁移、过期或元数据原子更新。
- **`test_ingest_products_table`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_get_table_info_from_datalake`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_list_params`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_auth_instructions`**：断言身份解析、登录回调、未授权拒绝或 token 刷新行为。
- **`test_fetch_data_as_arrow_uses_source_filters`**：断言导入/显示行数上限被执行。

#### `tests/database-dockers/postgres/test_postgresql_loader.py`（373 行，20 个 test_* ，类 2）

测试目标：验证 `database-dockers/postgres/postgresql_loader.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
测试类：TestPostgreSQLDataLoader、TestPostgreSQLDataLoaderStatic。
- **`test_list_tables`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_list_tables_with_filter`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_fetch_data_as_arrow_from_table`**：断言导入/显示行数上限被执行。
- **`test_fetch_data_respects_size`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_ingest_table_to_workspace`**：断言工作区创建、打开、迁移、过期或元数据原子更新。
- **`test_ingest_products_table`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_get_table_info_from_datalake`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_list_params`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_auth_instructions`**：断言身份解析、登录回调、未授权拒绝或 token 刷新行为。
- **`test_connect_forces_utf8_client_encoding`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_resolve_source_table_three_parts`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_resolve_source_table_three_parts_same_db`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_resolve_source_table_two_parts`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_resolve_source_table_one_part`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_ls_table_nodes_include_source_name`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_ls_schema_filters_postgresql_temp_schemas`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_ls_table_level_supports_limit_offset`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_source_filter_helper_compiles_postgres_operators`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_fetch_data_as_arrow_uses_source_filters`**：断言导入/显示行数上限被执行。
- **`test_secondary_connection_forces_utf8_client_encoding`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。

#### `tests/database-dockers/superset/sample_data.py`（232 行，0 个 test_* ，类 0）

测试目标：验证 `database-dockers/superset/sample_data.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
该文件可能是 fixture、benchmark 或辅助模块，不含 test_* 函数。

#### `tests/database-dockers/superset/superset_config.py`（144 行，0 个 test_* ，类 0）

测试目标：验证 `database-dockers/superset/superset_config.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
该文件可能是 fixture、benchmark 或辅助模块，不含 test_* 函数。

#### `tests/database-dockers/superset/test_superset_data_connector.py`（467 行，16 个 test_* ，类 5）

测试目标：验证 `database-dockers/superset/superset_data_connector.py` 所对应的生产模块。每个用例独立、不依赖执行顺序。失败时禁止为了变绿而改断言，应先分类（测试错 / 实现错 / 规格变）。
测试类：TestSupersetAuth、TestSupersetCatalog、TestSupersetData、TestSupersetTokenRefresh、TestSupersetFrontendConfig。
- **`test_connect_success`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_connect_bad_credentials`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_auth_mode_is_token`**：断言身份解析、登录回调、未授权拒绝或 token 刷新行为。
- **`test_disconnect_and_status`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_ls_root_lists_dashboards_and_all_datasets`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_ls_dashboard_lists_its_datasets`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_ls_all_datasets`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_ls_with_filter`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_catalog_metadata`**：断言目录缓存、同步、搜索或进度 API 的正确性。
- **`test_list_tables_flat`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_preview`**：断言预览行数、缓存或替换语义。
- **`test_import`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_connect_with_expired_token_triggers_refresh`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_config_structure`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_pinned_url`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`test_hierarchy_is_dashboard_dataset`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。

### 7.2 前端 vitest

#### `tests/frontend/setup.ts`（0 个 it）

describe 分组：
这些用例运行在 jsdom 中，验证选择器、迁移、组件渲染与 API 客户端契约，不启动真实 Flask。

#### `tests/frontend/unit/app/AuthButton.test.tsx`（1 个 it）

describe 分组：AuthButton backend logout
这些用例运行在 jsdom 中，验证选择器、迁移、组件渲染与 API 客户端契约，不启动真实 Flask。
- **`clears backend session and switches persisted identity to browser`**：断言导入/显示行数上限被执行。

#### `tests/frontend/unit/app/IdentityMigrationDialog.test.tsx`（7 个 it）

describe 分组：Anonymous user logs in and sees migration dialog； User clicks ； User clicks 
这些用例运行在 jsdom 中，验证选择器、迁移、组件渲染与 API 客户端契约，不启动真实 Flask。
- **`shows the dialog when anonymous workspaces exist`**：断言工作区创建、打开、迁移、过期或元数据原子更新。
- **`auto-closes when no anonymous workspaces exist`**：断言工作区创建、打开、迁移、过期或元数据原子更新。
- **`does NOT call cleanup-anonymous (anonymous data preserved)`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`does NOT call migrate endpoint`**：断言状态或身份迁移后数据仍完整。
- **`never shows `**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`navigates to home page`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`calls migrate endpoint and shows importing state`**：断言状态或身份迁移后数据仍完整。

#### `tests/frontend/unit/app/LayoutProvider.test.tsx`（6 个 it）

describe 分组：LayoutProvider； shell allocation at the floor
这些用例运行在 jsdom 中，验证选择器、迁移、组件渲染与 API 客户端契约，不启动真实 Flask。
- **`classifies the minimum supported viewport as compact and short`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`leaves a standard desktop at the reference layout`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`publishes the scale to CSS so stylesheets can follow`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`scales button geometry with spacious layouts`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`seats one thread column and still clears the canvas minimum`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`gives a wide screen more columns without starving the canvas`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。

#### `tests/frontend/unit/app/OidcCallback.test.tsx`（3 个 it）

describe 分组：OidcCallback error handling
这些用例运行在 jsdom 中，验证选择器、迁移、组件渲染与 API 客户端契约，不启动真实 Flask。
- **`redirects to /?auth_error=access_denied when IdP returns error`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。
- **`redirects with encoded error for other IdP error values`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。
- **`proceeds with signinRedirectCallback when no error param`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。

#### `tests/frontend/unit/app/agentInteractionPolicy.test.ts`（1 个 it）

describe 分组：agent interaction policy
这些用例运行在 jsdom 中，验证选择器、迁移、组件渲染与 API 客户端契约，不启动真实 Flask。
- **`keeps generated chart auto-focus disabled while the user is viewing a chart`**：断言 visualize 契约、标题或编码。

#### `tests/frontend/unit/app/agentMetadataTimeout.test.ts`（8 个 it）

describe 分组：agent metadata thunks
这些用例运行在 jsdom 中，验证选择器、迁移、组件渲染与 API 客户端契约，不启动真实 Flask。
- **`does not add a frontend timeout to semantic type requests`**：断言超时分类为可重试的 LLM/目录错误。
- **`retries semantic type inference once for transient model errors`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。
- **`does not retry non-retryable semantic type errors`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。
- **`does not add a frontend timeout to code explanation requests`**：断言超时分类为可重试的 LLM/目录错误。
- **`shows a warning when global model list loading fails`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`ignores aborted global model list requests`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`clears testing model status when connectivity check fails`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`ignores aborted connectivity checks`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。

#### `tests/frontend/unit/app/apiClient.test.ts`（37 个 it）

describe 分组：ApiRequestError； parseApiResponse； new unified format； Phase 2 unified format； strict protocol validation； parseStreamLine； apiRequest； assertDownloadResponseOk； streamRequest
这些用例运行在 jsdom 中，验证选择器、迁移、组件渲染与 API 客户端契约，不启动真实 Flask。
- **`should carry apiError and httpStatus`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。
- **`isRetryable returns true when retry flag is set`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`isRetryable returns false by default`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`isAuthError returns true for auth codes`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。
- **`isAuthError returns false for non-auth codes`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。
- **`should parse success response`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`should throw ApiRequestError on error response`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。
- **`should include detail when present`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`should parse status:`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`should throw on error with HTTP 4xx status`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。
- **`should throw on error with HTTP 5xx status`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。
- **`should reject legacy error_message field`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。
- **`should reject legacy message field`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`should reject status:`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`should reject legacy result field`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`should parse a normal event`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`should parse an error event`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。
- **`should parse a done event`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`should return null for empty lines`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`should return null for malformed JSON`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`should handle unicode content`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`should reject legacy status-wrapped events`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`should reject legacy status error wrapper`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。
- **`should return data on 200 + status:`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`should reject 200 + status:`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`should throw ApiRequestError on 400 + structured error`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。
- **`should throw ApiRequestError on 502 + structured error`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。
- **`should reject malformed 200 + status:`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`should throw HTTP_ERROR on non-JSON error response`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。
- **`should throw PARSE_ERROR on 200 with non-JSON body`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。
- **`should pass options to fetchWithIdentity`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`throws ApiRequestError for structured JSON error bodies`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。
- **`throws HTTP_ERROR for non-JSON transport failures`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。
- **`allows successful non-JSON download responses`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`yields top-level type events`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`throws on 200 application/json preflight error`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。
- **`rejects 200 application/json success for stream endpoints`**：断言 NDJSON 事件类型、预检错误或警告刷新。

#### `tests/frontend/unit/app/chartInsightContract.test.ts`（5 个 it）

describe 分组：chart insight contract
这些用例运行在 jsdom 中，验证选择器、迁移、组件渲染与 API 客户端契约，不启动真实 Flask。
- **`invalidates insight text when channel or aggregation changes`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`passes title and subtitle through Flint assembly`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`applies a Flint theme preset to the assembled base chart`**：断言 visualize 契约、标题或编码。
- **`does not let Flint mutate frozen chart properties while applying theme defaults`**：断言 visualize 契约、标题或编码。
- **`preserves the dark canvas supplied by the Power BI theme`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。

#### `tests/frontend/unit/app/clarification.test.ts`（9 个 it）

describe 分组：clarification helpers
这些用例运行在 jsdom 中，验证选择器、迁移、组件渲染与 API 客户端契约，不启动真实 Flask。
- **`normalizes structured clarify events and translates backend codes`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`defaults responseType to free_text when no options are provided`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`defaults responseType to single_choice when options are provided`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`accepts bare-string options`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`preserves opaque option values separately from translated labels`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`rejects clarify events without questions`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`formats single response as just the answer`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`formats multiple selections with 1-based indices`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`appends freeform text on its own line after selections`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。

#### `tests/frontend/unit/app/connectorFormPersistence.test.ts`（2 个 it）

describe 分组：connector form persistence
这些用例运行在 jsdom 中，验证选择器、迁移、组件渲染与 API 客户端契约，不启动真实 Flask。
- **`removes transient prefills from standalone chat messages`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`removes transient prefills from generalized form artifacts`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。

#### `tests/frontend/unit/app/dfSelectors.test.ts`（10 个 it）

describe 分组：dfSelectors.getActiveModel
这些用例运行在 jsdom 中，验证选择器、迁移、组件渲染与 API 客户端契约，不启动真实 Flask。
- **`should return the selected model when it exists`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`should fall back to the first model when selectedModelId does not match`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`should return undefined when the models array is empty`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`should return undefined when models is empty even with a selectedModelId`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`should return the first model when selectedModelId is undefined`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`should find a model in globalModels by selectedModelId`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`should prefer exact match in globalModels over first user model`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`should fall back to first globalModel when no id matches and models is empty`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`should fall back to globalModel (first in combined array) over user model`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`should handle undefined globalModels gracefully`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。

#### `tests/frontend/unit/app/dfSliceTableCollections.test.ts`（11 个 it）

describe 分组：split table collections； text artifact canvas ownership
这些用例运行在 jsdom 中，验证选择器、迁移、组件渲染与 API 客户端契约，不启动真实 Flask。
- **`stores a Flint theme and returns an active custom variant to the base chart`**：断言 visualize 契约、标题或编码。
- **`stores inferred field semantics separately from physical metadata`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`automatically migrates legacy tables when state is loaded`**：断言状态或身份迁移后数据仍完整。
- **`drops the retired mini agent setting from loaded state`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`stores input metadata without rows and tracks derived tables separately`**：断言导入/显示行数上限被执行。
- **`stores an authored parent edge when a report is finalized`**：断言身份解析、登录回调、未授权拒绝或 token 刷新行为。
- **`preserves a draft parent edge when promoting its result`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`repairs authored child edges when a text turn is removed`**：断言身份解析、登录回调、未授权拒绝或 token 刷新行为。
- **`removes a terminal response`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`removes loaded-table references with their shelf table`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`reparents authored children when a derived table is removed`**：断言身份解析、登录回调、未授权拒绝或 token 刷新行为。

#### `tests/frontend/unit/app/errorCodes.test.ts`（9 个 it）

describe 分组：ERROR_CODE_I18N_MAP； getErrorMessage
这些用例运行在 jsdom 中，验证选择器、迁移、组件渲染与 API 客户端契约，不启动真实 Flask。
- **`should have mappings for all major error codes`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。
- **`should map to errors.* i18n keys`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。
- **`should return translated message for known code`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`should return translated message for LLM rate limit`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`should fall back to backend message for unknown code`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`should fall back to backend message when i18n key has no translation`**：断言中英资源键齐全或语言注入提示词正确。
- **`should use TABLE_NOT_FOUND translation`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`should use STORAGE_FULL translation`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`should use CONNECTOR_AUTH_FAILED translation`**：断言身份解析、登录回调、未授权拒绝或 token 刷新行为。

#### `tests/frontend/unit/app/errorHandler.test.ts`（18 个 it）

describe 分组：handleApiError； ApiRequestError handling； AbortError handling； plain Error handling； silent option； callback options； RTK serialized error handling； extractErrorMessage
这些用例运行在 jsdom 中，验证选择器、迁移、组件渲染与 API 客户端契约，不启动真实 Flask。
- **`should dispatch addMessages for an ApiRequestError`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。
- **`should include detail in the dispatched message`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`should include request_id in the dispatched message detail`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`should include diagnostics with the full apiError`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。
- **`should silently ignore AbortError`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。
- **`should dispatch addMessages for a plain Error`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。
- **`should handle non-Error values`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。
- **`should not dispatch when silent is true`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`should call onAuth for auth errors and skip dispatch`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。
- **`should call onRetryable for retryable errors and skip dispatch`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。
- **`should dispatch normally when callback is not provided for matching error`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。
- **`should extract message from RTK serialized error object`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。
- **`should not produce [object Object] for serialized errors`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。
- **`should extract message from ApiRequestError`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。
- **`should extract message from plain Error`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。
- **`should extract message from RTK serialized error (plain object with .message)`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。
- **`should fallback to String() for unknown values`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`should not return [object Object] for objects with message`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。

#### `tests/frontend/unit/app/fetchWithIdentity.test.ts`（9 个 it）

describe 分组：fetchWithIdentity； Bearer token attachment； 401 retry with silent renew
这些用例运行在 jsdom 中，验证选择器、迁移、组件渲染与 API 客户端契约，不启动真实 Flask。
- **`should attach Authorization header when OIDC token is available`**：断言身份解析、登录回调、未授权拒绝或 token 刷新行为。
- **`should not attach Authorization header in anonymous mode`**：断言身份解析、登录回调、未授权拒绝或 token 刷新行为。
- **`should always attach X-Identity-Id header`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`preserves an explicit workspace instead of the active workspace`**：断言工作区创建、打开、迁移、过期或元数据原子更新。
- **`should not modify headers for non-API URLs`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`should retry once after silent renew on 401`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`should return 401 when silent renew fails`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`should not retry when no UserManager is available`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`should not retry on non-401 errors`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。

#### `tests/frontend/unit/app/getAccessToken.test.ts`（5 个 it）

describe 分组：getAccessToken
这些用例运行在 jsdom 中，验证选择器、迁移、组件渲染与 API 客户端契约，不启动真实 Flask。
- **`returns token when user exists and not expired`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`returns null when no user stored`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`calls signinSilent when token is expired and returns refreshed token`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`returns null when token is expired and signinSilent fails`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`returns null when token is expired and signinSilent returns null`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。

#### `tests/frontend/unit/app/i18nLocales.test.ts`（1 个 it）

describe 分组：i18n locale bundles
这些用例运行在 jsdom 中，验证选择器、迁移、组件渲染与 API 客户端契约，不启动真实 Flask。
- **`keeps Simplified Chinese translation keys aligned with English`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。

#### `tests/frontend/unit/app/inputTablePreviewCache.test.ts`（1 个 it）

describe 分组：input table preview cache
这些用例运行在 jsdom 中，验证选择器、迁移、组件渲染与 API 客户端契约，不启动真实 Flask。
- **`bounds rows and invalidates stale content versions`**：断言导入/显示行数上限被执行。

#### `tests/frontend/unit/app/layout.test.ts`（44 个 it）

describe 分组：reference layout； density scaling； size classes； type scale； canvas content sizing； default thread columns； column capacity as the window resizes； chart size stops； minimum screen budget
这些用例运行在 jsdom 中，验证选择器、迁移、组件渲染与 API 客户端契约，不启动真实 Flask。
- **`is the app as hardcoded today`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`leaves geometry untouched at reference density`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`reproduces the thread pane widths the Allotment snaps to`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`round-trips pane width back to column count`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`keeps real headroom below each snap point`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`never claims more columns than the strip can draw`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`still counts columns when zoom reports a sub-pixel pane width`**：断言报告流式字段或工具语法剥离。
- **`uses designed type stops, not a multiplied reference`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`keeps the ramp rhythm at every density`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`moves every stop up as density increases`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`scales lengths but never counts`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`yields whole pixels`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`asks for more thread columns as width grows`**：断言导入/显示行数上限被执行。
- **`scales density up with the screen, and only down on small ones`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`exposes a CSS variable for every token`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`seeds index.css with the reference values`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`never gives less room than the old fixed caps`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`gives a big canvas materially more table`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`bounds the table so it cannot run away on a 4K screen`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`raises the chart ceiling with the room, within bounds`**：断言 visualize 契约、标题或编码。
- **`holds steady through a drag instead of changing every pixel`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`never moves backwards as the canvas widens`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`starts at two once the screen affords it, even with one thread`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`stays at one where the shell cannot seat two`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`follows the content once it needs more than two`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`never offers more than the width class allows`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`keeps two columns on a mid-size laptop`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`drops to one column once a second would squeeze the canvas`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`lets a measured split open a second column the width class would veto`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`leaves the canvas comfortable at every count it suggests`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`treats a missing container as no constraint, not as a cap`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`never decreases as the split container grows`**：断言导入/显示行数上限被执行。
- **`always leaves the canvas its minimum`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`steps exactly at the pane snap points`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`puts the authored size on a stop, so `1` is always reachable`**：断言身份解析、登录回调、未授权拒绝或 token 刷新行为。
- **`increases monotonically`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`snaps a persisted free-slider factor to the closest stop`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`grows the suggested size with the screen`**：断言导入/显示行数上限被执行。
- **`suggests the authored size on a standard screen`**：断言身份解析、登录回调、未授权拒绝或 token 刷新行为。
- **`only ever suggests a real stop, so the slider can show it`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`fits MIN_SUPPORTED at the densities small screens actually get`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`does not pretend comfortable density fits a floor viewport`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`clamps any requested density to what the viewport can seat`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`cannot fit an expanded sidebar at the floor — which is why it rails`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。

#### `tests/frontend/unit/app/loadableState.test.ts`（4 个 it）

describe 分组：loadableState
这些用例运行在 jsdom 中，验证选择器、迁移、组件渲染与 API 客户端契约，不启动真实 Flask。
- **`preserves previous data while entering loading`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`marks successful empty data with the empty status`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`extracts API error messages for error state`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。
- **`starts as idle without data`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。

#### `tests/frontend/unit/app/oidcConfig.test.ts`（3 个 it）

describe 分组：getAuthInfo
这些用例运行在 jsdom 中，验证选择器、迁移、组件渲染与 API 客户端契约，不启动真实 Flask。
- **`unwraps the unified API success envelope`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`keeps legacy flat auth info responses compatible`**：断言身份解析、登录回调、未授权拒绝或 token 刷新行为。
- **`returns null when auth info is unavailable`**：断言身份解析、登录回调、未授权拒绝或 token 刷新行为。

#### `tests/frontend/unit/app/rehydrateBackfill.test.ts`（3 个 it）

describe 分组：rehydrating a payload that predates a collection
这些用例运行在 jsdom 中，验证选择器、迁移、组件渲染与 API 客户端契约，不启动真实 Flask。
- **`backfills a missing array so consumers can read .length`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`leaves existing collections untouched`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`replaces a non-array value with an empty array`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。

#### `tests/frontend/unit/app/stateMigrations.test.ts`（5 个 it）

describe 分组：state migrations
这些用例运行在 jsdom 中，验证选择器、迁移、组件渲染与 API 客户端契约，不启动真实 Flask。
- **`applies the single v3 split, semantic extraction, and legacy cleanup`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`normalizes partial pre-release states into v3 without duplicates`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`upgrades an already split pre-release state to v4`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`moves loaded-table thread edges into reference nodes`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`unifies authored table, draft, and report edges on parentNodeId`**：断言身份解析、登录回调、未授权拒绝或 token 刷新行为。

#### `tests/frontend/unit/app/tableLoadsInFlight.test.ts`（5 个 it）

describe 分组：tableLoadsInFlight
这些用例运行在 jsdom 中，验证选择器、迁移、组件渲染与 API 客户端契约，不启动真实 Flask。
- **`starts at zero`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`counts a load as in flight until it settles`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`clears the counter when a load fails`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`tracks concurrent loads independently`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`never drops below zero when a settle arrives after a state reset`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。

#### `tests/frontend/unit/app/tableResolution.test.ts`（6 个 it）

describe 分组：workspaceTableIdOf； toAnalystTableRef
这些用例运行在 jsdom 中，验证选择器、迁移、组件渲染与 API 客户端契约，不启动真实 Flask。
- **`uses the workspace table id for workspace-backed inputs`**：断言工作区创建、打开、迁移、过期或元数据原子更新。
- **`uses the materialized copy for connector-backed inputs`**：断言连接器 CRUD、可见性隔离或表单字段。
- **`falls back to the entry id when a connector input is not materialized`**：断言连接器 CRUD、可见性隔离或表单字段。
- **`sends the workspace id, the user-facing name, and the snapshot schema`**：断言工作区创建、打开、迁移、过期或元数据原子更新。
- **`omits row_count when the snapshot has no known count`**：断言导入/显示行数上限被执行。
- **`never carries preview rows`**：断言导入/显示行数上限被执行。

#### `tests/frontend/unit/app/tableThunks.test.ts`（6 个 it）

describe 分组：resolveDatabaseImportLimit； buildDictTableFromWorkspace
这些用例运行在 jsdom 中，验证选择器、迁移、组件渲染与 API 客户端契约，不启动真实 Flask。
- **`does not treat an intentional query limit as safety truncation`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`applies the safety cap to unbounded and oversized imports`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`preserves column descriptions in metadata`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`omits description when not provided by backend`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`uses table-level loader description as DictTable.description`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`works with no descriptions at all`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。

#### `tests/frontend/unit/app/useAutoSave.test.tsx`（2 个 it）

describe 分组：useAutoSave
这些用例运行在 jsdom 中，验证选择器、迁移、组件渲染与 API 客户端契约，不启动真实 Flask。
- **`notifies the frontend when auto-save fails`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`strips connector form prefills from workspace snapshots`**：断言工作区创建、打开、迁移、过期或元数据原子更新。

#### `tests/frontend/unit/app/useWorkspaceAutoName.test.tsx`（3 个 it）

describe 分组：useWorkspaceAutoName
这些用例运行在 jsdom 中，验证选择器、迁移、组件渲染与 API 客户端契约，不启动真实 Flask。
- **`recognizes the workspace placeholder name`**：断言工作区创建、打开、迁移、过期或元数据原子更新。
- **`calls the workspace name API with the selected server-managed model payload`**：断言工作区创建、打开、迁移、过期或元数据原子更新。
- **`does not auto-name a custom workspace name`**：断言工作区创建、打开、迁移、过期或元数据原子更新。

#### `tests/frontend/unit/app/workspaceService.test.ts`（7 个 it）

describe 分组：ephemeral workspace recovery； local workspace parity
这些用例运行在 jsdom 中，验证选择器、迁移、组件渲染与 API 客户端契约，不启动真实 Flask。
- **`stores a row-free browser snapshot after a successful server save`**：断言导入/显示行数上限被执行。
- **`loads an expired server workspace from its browser snapshot as read-only`**：断言工作区创建、打开、迁移、过期或元数据原子更新。
- **`returns only the server workspace list without consulting recovery storage`**：断言工作区创建、打开、迁移、过期或元数据原子更新。
- **`propagates a missing-workspace error without consulting recovery storage`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。
- **`saves only to the server without creating a recovery snapshot`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`hydrates previews against the workspace being loaded`**：断言工作区创建、打开、迁移、过期或元数据原子更新。
- **`rejects a load superseded by a newer workspace switch`**：断言工作区创建、打开、迁移、过期或元数据原子更新。

#### `tests/frontend/unit/components/ConnectorTablePreview.test.tsx`（2 个 it）

describe 分组：ConnectorTablePreview source metadata
这些用例运行在 jsdom 中，验证选择器、迁移、组件渲染与 API 客户端契约，不启动真实 Flask。
- **`shows the source table description directly`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`uses descriptions on table headers without restoring the old metadata panel`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。

#### `tests/frontend/unit/components/DataOperationCard.test.tsx`（2 个 it）

describe 分组：DataOperationCard
这些用例运行在 jsdom 中，验证选择器、迁移、组件渲染与 API 客户端契约，不启动真实 Flask。
- **`renders immutable plan alternatives without execution controls`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`shows partial status and failed step names`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。

#### `tests/frontend/unit/components/LoadPlanCard.test.tsx`（4 个 it）

describe 分组：LoadPlanCard； buildLoadQueryImportOptions
这些用例运行在 jsdom 中，验证选择器、迁移、组件渲染与 API 客户端契约，不启动真实 Flask。
- **`presents connector and scratch candidates through one selection flow`**：断言连接器 CRUD、可见性隔离或表单字段。
- **`does not fetch a preview for a scratch-only plan`**：断言预览行数、缓存或替换语义。
- **`converts the canonical load query to connector import options`**：断言连接器 CRUD、可见性隔离或表单字段。
- **`bounds previews without changing the requested load limit`**：断言预览行数、缓存或替换语义。

#### `tests/frontend/unit/components/filterFormat.test.ts`（4 个 it）

describe 分组：formatFilterChipLabel
这些用例运行在 jsdom 中，验证选择器、迁移、组件渲染与 API 客户端契约，不启动真实 Flask。
- **`uses label-value language for equality and membership`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`uses a compact range instead of query syntax`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`spells out operators whose meaning matters`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`keeps familiar comparison symbols`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。

#### `tests/frontend/unit/data/coerceDate.test.ts`（13 个 it）

describe 分组：coerceDate； coerceDateTime； coerceTime； coerceDuration
这些用例运行在 jsdom 中，验证选择器、迁移、组件渲染与 API 客户端契约，不启动真实 Flask。
- **`should return null for null input`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`should return null for undefined input`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`should return null for empty string`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`should convert Date object to date-only ISO string`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`should pass through string date values unchanged`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`should pass through numeric timestamps unchanged`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`should return null for null/undefined/empty`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`should convert Date object to full ISO string`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`should pass through ISO datetime strings unchanged`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`should return null for null/undefined/empty`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`should pass through time strings unchanged`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`should return null for null/undefined/empty`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`should pass through duration values unchanged`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。

#### `tests/frontend/unit/data/resolveExcelCellValue.test.ts`（16 个 it）

describe 分组：resolveExcelCellValue
这些用例运行在 jsdom 中，验证选择器、迁移、组件渲染与 API 客户端契约，不启动真实 Flask。
- **`should return null for null`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`should return null for undefined`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`should return string as-is`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`should return number as-is`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`should return boolean as-is`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`should return empty string as-is`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`should convert Date to ISO string`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`should join richText segments`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`should handle richText with missing text fields`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`should extract text from hyperlink object`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`should fall back to hyperlink URL when text is empty`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`should resolve formula result (primitive)`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`should resolve formula result (Date)`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`should return null for formula with undefined result`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`should return null for error cell value`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。
- **`should stringify unknown objects`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。

#### `tests/frontend/unit/data/typeInference.test.ts`（34 个 it）

describe 分组：testDate (strict YYYY-MM-DD)； testDateTime； testTime； testDuration； inferTypeFromValueArray； mapApiTypeToAppType
这些用例运行在 jsdom 中，验证选择器、迁移、组件渲染与 API 客户端契约，不启动真实 Flask。
- **`should accept ISO date strings`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`should reject datetime strings (should be DateTime)`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`should reject pure year numbers`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`should reject time-only strings`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`should reject non-string values`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`should handle whitespace trimming`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`should accept ISO datetime strings`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`should accept Date objects`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`should reject date-only strings`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`should reject time-only strings`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`should reject non-date strings`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`should accept time strings`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`should accept time with timezone`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`should reject date strings`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`should reject datetime strings`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`should reject non-strings`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`should accept ISO 8601 duration strings`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`should reject bare `**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`should reject non-duration strings`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`should infer Boolean for boolean values`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`should infer Integer for whole numbers`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`should infer Number for decimal numbers`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`should infer Date for YYYY-MM-DD values`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`should infer DateTime for YYYY-MM-DDTHH:mm:ss values`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`should infer Time for HH:mm:ss values`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`should infer Duration for ISO duration values`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`should NOT mis-identify pure year numbers as Date`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`should fall back to String for mixed types`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`should skip nulls and empty strings during inference`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`should return Boolean (first candidate) for empty array`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`should return Boolean (first candidate) for all-null array`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`should map standardized labels`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`should handle legacy pandas dtype strings`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`should be case-insensitive`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。

#### `tests/frontend/unit/dataOperations/models.test.ts`（6 个 it）

describe 分组：parseDataOperation
这些用例运行在 jsdom 中，验证选择器、迁移、组件渲染与 API 客户端契约，不启动真实 Flask。
- **`maps the versioned wire contract to a detached view model`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`parses optional discovery canvas presentation`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`rejects an unsupported schema version`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`rejects a selected plan outside the operation`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`rejects malformed plan hashes`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`parses partial results and structured step failures`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。

#### `tests/frontend/unit/views/ClarificationPanel.test.tsx`（8 个 it）

describe 分组：ClarificationPanel
这些用例运行在 jsdom 中，验证选择器、迁移、组件渲染与 API 客户端契约，不启动真实 Flask。
- **`selects an operation plan without submitting until Continue is clicked`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`offers only the loading options, leaving other requests to the chat input`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`submits a single-choice question immediately when an option is clicked`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`records partial selections via onSelectAnswer without submitting`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`renders an inline input under a free-text question and submits it tagged to that question`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`lets a single-choice question take a typed answer instead of a chip`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`supersedes a selected option when the user types a custom answer`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`records a typed answer live and submits it on Enter`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。

#### `tests/frontend/unit/views/DataFrameTable.test.tsx`（3 个 it）

describe 分组：DataFrameTable
这些用例运行在 jsdom 中，验证选择器、迁移、组件渲染与 API 客户端契约，不启动真实 Flask。
- **`renders column headers`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`adds dotted underline to headers with descriptions`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`does not set native title when columnDescriptions is provided for that col`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。

#### `tests/frontend/unit/views/DataLoadingChat.test.tsx`（1 个 it）

describe 分组：DataLoadingChat canvas
这些用例运行在 jsdom 中，验证选择器、迁移、组件渲染与 API 客户端契约，不启动真实 Flask。
- **`opens Python-produced tables in the unified right-side load plan`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。

#### `tests/frontend/unit/views/DataSourceSidebar.test.tsx`（3 个 it）

describe 分组：DataSourceSidebar
这些用例运行在 jsdom 中，验证选择器、迁移、组件渲染与 API 客户端契约，不启动真实 Flask。
- **`leaves loading state when catalog fetch fails`**：断言目录缓存、同步、搜索或进度 API 的正确性。
- **`returns to the landing state without creating an empty workspace`**：断言工作区创建、打开、迁移、过期或元数据原子更新。
- **`shows recently modified sessions first and can switch to creation order`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。

#### `tests/frontend/unit/views/SessionDistill.test.tsx`（8 个 it）

describe 分组：buildSessionWorkflowContext； findSessionWorkflow
这些用例运行在 jsdom 中，验证选择器、迁移、组件渲染与 API 客户端契约，不启动真实 Flask。
- **`returns null when no threads are supplied`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`packs every supplied thread into the payload, in order`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`counts steps as create_table events across all threads`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`drops tool-call events when over the byte budget`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`drops oldest threads when payload still exceeds budget after lighter trimming`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`returns the entry whose sourceWorkspaceId matches`**：断言工作区创建、打开、迁移、过期或元数据原子更新。
- **`returns undefined when nothing matches`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`returns undefined when workspaceId is empty`**：断言工作区创建、打开、迁移、过期或元数据原子更新。

#### `tests/frontend/unit/views/experienceContext.test.ts`（14 个 it）

describe 分组：isLeafDerivedTable； buildLeafEvents； buildDistillModelConfig
这些用例运行在 jsdom 中，验证选择器、迁移、组件渲染与 API 客户端契约，不启动真实 Flask。
- **`returns true for a derived table with no children`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`returns false for a derived table that has children`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`returns false for non-derived tables`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`returns null for non-derived table`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`returns null when no user-originated message exists in chain`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`builds a flat event timeline from a single-step chain`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`emits one create_table per derived step in a multi-step chain`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`skips the deleted middle table (chain re-parented)`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`emits create_chart paired with create_table when the step has a chart`**：断言 visualize 契约、标题或编码。
- **`drops error-role and empty interaction entries`**：断言错误码、HTTP 状态或错误 envelope 符合统一协议，且不泄漏内部细节。
- **`sends raw code (not a code shape summary)`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`includes columns, row_count, and sample_rows on create_table`**：断言导入/显示行数上限被执行。
- **`preserves global model identity so the backend can resolve server credentials`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`preserves user model api_base for custom endpoints`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。

#### `tests/frontend/unit/views/formatCellValue.test.ts`（16 个 it）

describe 分组：formatCellValue
这些用例运行在 jsdom 中，验证选择器、迁移、组件渲染与 API 客户端契约，不启动真实 Flask。
- **`should return empty string for null/undefined`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`should format numbers with locale separators`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`should not add separators for non-measure numeric semantics`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`should keep separators for measure numeric semantics`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`should format booleans as strings`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`should pass through plain strings`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`should work without dataType parameter`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`should format Date type with locale date`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`should handle invalid date gracefully`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`should format DateTime type with locale datetime`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`should handle invalid datetime gracefully`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`should format Time type with locale time`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`should handle invalid time gracefully`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`should format Duration from milliseconds`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`should not over-format sub-second Duration values`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`should pass through non-numeric Duration as string`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。

#### `tests/frontend/unit/views/safeCellRender.test.tsx`（13 个 it）

describe 分组：safeCellRender – inline pattern (ReactTable / SelectableDataGrid)； formatFn – DataLoadingThread format callback
这些用例运行在 jsdom 中，验证选择器、迁移、组件渲染与 API 客户端契约，不启动真实 Flask。
- **`should render string values directly`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`should render number values directly`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`should render null without crashing`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`should render undefined without crashing`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`should render boolean as string`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`should safely render a Date object as string`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`should safely render a plain object as string`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`should safely render an array as string`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`should pass through string values`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`should pass through number values`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`should pass through null`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`should convert Date object to string`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。
- **`should convert arbitrary object to string`**：针对该名称所描述的行为做正向或反向断言：给定夹具输入，输出结构、边界值或异常类型必须与规格一致。回归时若失败，应对照最近的协议/模型变更，而不是放宽断言。

### 7.3 测试设计原则在本仓库中的体现
- 协议契约测试（errors/test_api_error_protocol_contract.py）锁定 HTTP 200 + envelope。
- 安全测试独立成包，覆盖沙箱逃逸、日志脱敏、路径穿越、全局模型密钥不泄漏。
- 连接器框架测试用假 loader，不要求真实数据库；database-dockers 做真协议回归。
- 前端对 i18n、OIDC、identity 迁移有专项，防止“只在中文环境坏”或“登录后工作区丢失”。


## 8. 设计决策记录（摘要）

1. **为什么业务错误用 HTTP 200**：流式端点一旦 flush 了 200 就无法改状态码；统一后代理不会把“表不存在”当 5xx 报警。
2. **为什么技能是目录包而不是提示词拼接**：可测试、可按需加载、工具 schema 与文档同居，减少漂移。
3. **为什么加载计划不可变**：让用户审的是 Agent 提议，而不是一个可随意改的 SQL 编辑器；便于 hash 去重。
4. **为什么 input table 行不进 Redux persist**：避免 localStorage/IndexedDB 被百万行撑爆，刷新后从服务器 sample。
5. **为什么 local sandbox 是默认**：开发体验与 CI 速度；docker 作为高隔离选项。
6. **为什么 TokenStore 用服务端 session**：cookie 4KB 装不下 SSO+多服务 token；flask-session 文件后端可扩展。
7. **为什么图表走 Flint 而不是手写 Vega**：压缩规格、统一主题与推荐，减少 Agent 生成非法 spec。

## 9. 风险与缓解

| 风险 | 缓解 |
|------|------|
| 模型乱调用 action | 外壳强制 one-action-per-turn；schema 校验 |
| 提示词注入读盘 | ConfinedDir + 沙箱 audit |
| SSRF 打内网 | url_allowlist + 私网 IP 拒绝 + api_base 白名单 |
| 超大目录拖死 UI | 虚拟树 + 缓存 + 搜索而非全量展开 |
| 会话密钥丢失 | FLASK_SECRET_KEY 必须持久化，迁移清单已写 |
| 弱模型 JSON 失败 | salvage + unpack + repair 循环 |
| 多用户数据串租户 | identity 前缀路径 + 连接器可见性过滤 |

## 10. 本说明书的维护

源码变更若修改：错误协议、流事件、Skill 工具、DataOperation schema、身份格式、行数上限，
必须同步更新本文件对应章节与测试专章。目录级函数清单可由 AST 再生，但算法叙述需要人工审校。
