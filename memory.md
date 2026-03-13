# Request-Response 对话记忆（交接版）

此文件用于在多轮对话中交接当前已确认的决策、未确认的冲突点与下一步精确动作。把该文件直接提供给新 AI 时，AI 能立即进入状态并按既定流程推进“一个冲突点一轮”的确认。

## 一、总体目标
- 把 `full-stack-fastapi-template` 作为工程底座，叠加 `request_response` 协议契约层，作为以后所有 request_response 模板化生成的基础。

## 二、已确认（不可更改，直接执行）

### 1) 响应协议（已确认）
- 格式：统一响应壳（双层）。示例：
	- 成功：
		{
			"success": true,
			"code": "OK",
			"message": "success",
			"request_id": "req-...",
			"ts": 1710000000100,
			"data": { ...业务数据... }
		}
	- 失败：
		{
			"success": false,
			"code": "NOT_FOUND",
			"message": "...",
			"request_id": "req-...",
			"ts": 1710000000100,
			"error": { "details": {...} }
		}
- 说明：错误也必须同壳，并且不直接把 FastAPI 的 `detail` 文本透出给客户端（由异常映射表转为标准 code + message）。

### 2) HTTP 状态码策略（已确认）
- 保留 HTTP 状态码语义（用于网关、代理、监控），同时响应体也包含业务 `code`。这是“双轨并行”策略。

### 3) request_id（已确认）
- 来源：优先透传 `X-Request-ID` 请求头；若缺失后端自动生成；两者并存并记录在响应壳与日志中。

### 4) 分层与模型拆分（已确认）
- 强制存在 `service/` 层，路由层仅做参数校验与上下文注入，业务逻辑放 `service`。
- 模型按用途拆为三类并放不同文件夹：
	- `entities/`：数据库持久化模型（SQLModel/ORM，含关系/约束）
	- `schemas/request/`：入参校验模型（Pydantic/SQLModel 用于请求）
	- `schemas/response/`：出参模型（用于构造统一响应壳内 data）

## 三、完整冲突点清单（供后续逐项确认）

下面每一项都是需要逐一决策的冲突点。每轮只处理第一个未确认的冲突点。每个条目包含：冲突描述、影响范围、可选方案与建议、参考模板文件路径（相对 workspace）。

1) 响应协议（已确认）
	- 描述：模板当前是标准 REST（直接返回资源，错误为 HTTPException detail），与统一壳冲突。已确认采用统一双层壳并保持 HTTP 状态码语义并行。
	- 参考：`full-stack-fastapi-template/backend/app/api/routes/*.py`，`full-stack-fastapi-template/backend/app/models.py`。

2) 分层结构（已确认）
	- 描述：模板把逻辑散在路由与 `crud`，指南要求独立 `service` 层与 domain 分离。已确认按指南拆分。
	- 参考：`full-stack-fastapi-template/backend/app/`。

3) 数据库默认策略（待确认）
	- 描述：指南建议“若未指定数据库，优先生成内存/文件版本以可运行”，模板默认 Postgres + Alembic。若强制 Postgres，会增加运行门槛。影响本地开发体验、CI 与生成器模板。
	- 可选方案：
		A. 默认生成两套配置：开发（SQLite/in-memory）+ 生产（Postgres）。（推荐）
		B. 一律使用 Postgres（模板原样）。
		C. 一律使用 in-memory（只适用于极小 demo）。
	- 参考：`full-stack-fastapi-template/backend/app/core/config.py`、`compose.yml`、`alembic/`。

4) 统一错误码与异常映射（待确认）
	- 描述：模板当前用 `HTTPException(detail=...)` 与不同 status codes；指南需要错误码表（OK/INVALID_ARGUMENT/UNAUTHORIZED/...）并将所有异常映射到该表。影响：日志、告警、前端错误处理。
	- 可选方案：
		A. 在全局异常处理器中把 HTTPException/ValidationError 映射为业务 code（推荐）。
		B. 保留现状，仅在新生成的接口中使用映射。
	- 参考：`full-stack-fastapi-template/backend/app/main.py`、路由中 `raise HTTPException(...)`。

5) 请求追踪与结构化日志（已确认 request_id 规则，但还需细化日志字段）
	- 描述：指南要求结构化日志与 request_id 埋点。需决定日志库（标准 logging/structlog/loguru）与字段名（request_id、user_id、latency、code）。
	- 可选方案：
		A. 使用标准 `logging` + JSONFormatter（简单、兼容）。
		B. 使用 `structlog`（更灵活，推荐长期使用）。
	- 参考：`full-stack-fastapi-template/backend/app/backend_pre_start.py`（当前使用 logging）。

6) 健康检查路径与探针契约（待确认）
	- 描述：指南示例为 `GET /health`，模板为 `/api/v1/utils/health-check/`。需决定统一路径与返回壳（健康探针通常要求最简返回）。
	- 可选方案：
		A. 保持探索器友好的最简健康端点（单独例外，不包装壳）。
		B. 强制健康接口也返回统一壳（但需兼顾探针兼容）。
	- 参考：`full-stack-fastapi-template/backend/app/api/routes/utils.py`。

7) 测试策略（待确认）
	- 描述：需决定生成器输出的测试集合：契约测试（验证壳）、接口单元测试、集成测试、以及 E2E（Playwright）。
	- 建议：最低每生成功能包含一个契约测试与一个接口测试；复杂项目加入集成与 E2E。
	- 参考：`full-stack-fastapi-template/backend/tests/`、`frontend/tests/`。

8) 鉴权与会话策略（待确认）
	- 描述：模板使用 OAuth2PasswordBearer + JWT；指南默认使用 Bearer Token。需决定是否保留 OAuth2 路由兼容或只用简化的 Bearer JWT。影响文档与客户端生成。
	- 可选方案：
		A. 保留 OAuth2 兼容路由（`/login/access-token`）并标准化响应壳。
		B. 仅使用简化 Bearer JWT 发行接口。
	- 参考：`full-stack-fastapi-template/backend/app/api/routes/login.py`、`app/core/security.py`。

9) OpenAPI / 文档展示（待确认）
	- 描述：统一壳会使 OpenAPI 文档展示业务模型被包裹。需决定如何在 docs 中保持可读性（例如：在 schema 中展示 data 的类型并注解壳）。
	- 可选方案：
		A. 在 OpenAPI 中展示统一壳（完整但包裹）。
		B. 在文档注释中强调实际返回通常包裹在 `data` 中，并在示例里展开。

10) 部署/监控与状态码依赖（待确认）
	- 描述：Traefik、Sentry 等组件依赖 HTTP 状态码做健康判断、错误采样。若只返回 200，会导致监控失效。已确认保留状态码，但仍需检查各部署脚本对状态码的假设。
	- 参考：`compose.yml`、`deployment.md`、模板 README。

11) 前端客户端生成与接口契约（待确认）
	- 描述：模板自带自动生成前端客户端 SDK（TypeScript）。需决定 SDK 是否按`data`字段拆解或保留原始壳。
	- 推荐：SDK 提供两套访问方式：`raw`（返回完整壳）与 `unwrap`（直接返回 data）。

12) 邮件与外部服务配置（待确认）
	- 描述：模板默认包含 SMTP、Mailcatcher 等配置。需决定是否在生成项目中保留这些模块或按 opt-in 方式接入。

## 四、对后续 AI 的明确指令（如何继续）

- 每轮只处理清单中第一个未完成冲突点（见上方编号顺序）。
- 对每个冲突点，必须提供：
	1. 冲突简述（1-2 行）
	2. 至少两个备选方案（含优缺点）
	3. 推荐方案与理由（若无偏好，标注为“需用户选择”）
	4. 关键实现要点（代码位置、需要修改/新增文件清单）
	5. 一次性可执行的变更示例（小 patch 示例或生成器模板变更要点）

## 五、下一步建议（已写入待办）
- 当前待办已列出所有冲突点（见 `TODO` 列表）。下一步由 AI 发起冲突点 3（数据库默认策略）确认轮。

## 六、如何把这个文件给另一个 AI
- 直接把 `memory.md` 内容提供给新 AI；开场提示它：
	- "复述：已完成冲突点 1、2。现在只讨论冲突点 3（数据库默认策略）。请给出两个备选方案并推荐一个。" 

-- 结束 --
