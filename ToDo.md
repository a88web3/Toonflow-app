# Toonflow 生产改造施工清单

> 唯一施工入口。来源：`docs/TOONFLOW_PRODUCTION_AUDIT.md`（2026-11-15）。
> 初始状态仅为审查规划，**不是实施证据**。每个任务完成后由执行者更新本文件的状态、真实命令/人工验收结果、commit SHA 和恢复记录；不得把计划文字改成 COMPLETED。
> Rolling Batch：仅施工所有前置依赖已 `COMPLETED` 的 `READY` 项；每批最多 2 个相互独立任务。遇到外部服务、合同、密钥、生产数据或验收不确定性，标为 `BLOCKED` 并停止，不绕过。

## 状态总览

| 阶段 | READY | IN_PROGRESS | BLOCKED | COMPLETED |
| --- | ---: | ---: | ---: | ---: |
| P0 | 3 | 0 | 0 | 0 |
| P1 | 2 | 0 | 0 | 0 |
| P2 | 0 | 0 | 4 | 0 |
| P3 | 0 | 0 | 4 | 0 |
| P4 | 0 | 0 | 3 | 0 |
| P5 | 0 | 0 | 3 | 0 |
| P6 | 0 | 0 | 2 | 0 |
| **合计** | **5** | **0** | **16** | **0** |

## 通用施工规则

- 每任务一个独立 OpenCode session；不要把任务拆分外的重构、依赖升级或基础设施修改混入。
- 禁止重启/修改 Mac Studio MLX、OpenCode 4096，禁止调用付费媒体 API，除非具体任务明确授权并记录。
- 不删除 OSS/数据库/用户素材；任何迁移必须先备份、验证恢复、再取得明确确认。
- 所有新增密钥使用引用或安全存储；不得提交到仓库、日志、截图或本文件。
- `必须执行的测试` 仅列验收时应运行的已有命令/手动验证；执行者只可记录真实结果。

---

## P0：安全、数据保护与生产阻断

### P0-001 — 服务监听、CORS 与入口隔离

- 状态：`READY`
- 所属模块：Server security
- 优先级：P0 / 风险：Critical
- 任务目标：使 standalone 3000 默认只对 loopback 可达；建立显式、可审计的外部入口策略与严格 origin allowlist。
- 当前问题：`apps/server/src/app.ts` 使用 `cors()`；`apps/server/src/index.ts` 用 `app.listen(realPort)` 未显式 host。未启用多租户时 API 无用户认证。
- 涉及文件：`apps/server/src/app.ts`、`apps/server/src/index.ts`、部署文档（新增/更新由执行者确定）。
- 实施范围：仅网络暴露、CORS、启动配置与安全文档；不得改变 MLX/OpenCode 端口或自动暴露公网。
- 前置依赖：无。
- 验收标准：默认模式仅 `127.0.0.1/::1`；允许来源为显式配置且拒绝未知 Origin；受控外部入口有认证边界说明；原桌面本地流程手测仍可用。
- 必须执行的测试：已有 server typecheck/build；本机 curl/浏览器验证本地成功、非允许 Origin 被拒绝、非 loopback 无监听（不扫描外网）。
- 交付物：代码、部署运行手册、安全验证记录。
- Git commit 证据：待完成填写 SHA。
- 失败恢复方式：回退本任务 commit；恢复前确认监听未意外暴露。

### P0-002 — 密钥、插件与供应商代码准入基线

- 状态：`READY`
- 所属模块：Configuration / plugin security
- 优先级：P0 / 风险：Critical
- 任务目标：定义并实现最小可行的密钥脱敏、导出禁令、插件/供应商来源准入与审计事件边界。
- 当前问题：API key 存在 settings/provider 配置；插件和供应商为本机代码执行面，`apps/server/src/utils/plugins/*` 和 provider 安装入口没有生产签名/allowlist 策略。
- 涉及文件：`apps/server/src/utils/conf/index.ts`、`apps/server/src/utils/plugins/*`、`apps/server/src/routes/providers/media/*`、相关设置 UI/文档。
- 实施范围：安全存储适配设计、日志脱敏、默认禁止不可信远程 provider/plugin、管理员显式批准流；不迁移真实密钥、不删除用户插件。
- 前置依赖：无。
- 验收标准：日志/接口/导出不回传明文密钥；未批准来源不能安装/执行；批准、来源、哈希和操作者可追溯；已有内置 provider 不被破坏。
- 必须执行的测试：已有 typecheck/build；使用临时假密钥和临时插件做安装拒绝/批准/脱敏手测。
- 交付物：实施说明、威胁模型、准入台账格式。
- Git commit 证据：待完成填写 SHA。
- 失败恢复方式：回退本任务 commit；保留现有配置文件与用户插件，不执行清理。

### P0-003 — 数据保护、备份与恢复基线

- 状态：`READY`
- 所属模块：Data operations
- 优先级：P0 / 风险：Critical
- 任务目标：为 settings、mediaTasks、workspaces、assets 与可选多租户数据库建立可验证的备份/恢复操作规范和只读盘点工具。
- 当前问题：任务/素材/画布分散在文件系统；无备份、保留、恢复演练证据。
- 涉及文件：`apps/server/src/utils/conf/index.ts`、`apps/server/src/utils/media/tasks.ts`、`apps/server/src/utils/workspace/*`、运维文档/脚本目录（不触及真实数据）。
- 实施范围：安全的备份 manifest、校验、恢复到临时目录的流程和 runbook；不删除、不迁移真实生产文件。
- 前置依赖：无。
- 验收标准：针对人工创建的临时数据集可完成备份、哈希校验、隔离目录恢复和读取验证；定义 RPO/RTO 与保留负责人。
- 必须执行的测试：临时目录端到端备份/恢复演练；已有 typecheck（如改动 TS）。
- 交付物：runbook、恢复演练记录、数据分类表。
- Git commit 证据：待完成填写 SHA。
- 失败恢复方式：仅删除执行者创建的临时演练目录；真实数据只读不动。

## P1：架构稳定性与基础验证

### P1-001 — 结构化日志、健康与任务可观测性

- 状态：`READY`
- 所属模块：Server observability
- 优先级：P1 / 风险：High
- 任务目标：提供无密钥/无内容泄露的 requestId、任务状态、错误分类、健康和最小指标，覆盖文本与媒体请求。
- 当前问题：`apps/server/src/app.ts` 主要使用 console/morgan；媒体任务和 SSE 无统一追踪/告警视图。
- 涉及文件：`apps/server/src/app.ts`、`apps/server/src/routes/ai/generate.ts`、`apps/server/src/utils/media/tasks.ts`、相关 route 与文档。
- 实施范围：可关闭的结构化日志、健康端点、指标契约、trace 传播；不接入外部遥测 SaaS，除非用户另行批准。
- 前置依赖：P0-001、P0-002。
- 验收标准：单次文本/模拟媒体错误可通过 requestId 关联；日志没有 token、API key、prompt 正文或二进制；健康检查不产生写入。
- 必须执行的测试：已有 typecheck/build；本地临时请求与日志审查；错误路径手测。
- 交付物：日志字段说明、健康/指标说明、证据样本（脱敏）。
- Git commit 证据：待完成填写 SHA。
- 失败恢复方式：回退本任务 commit；禁用新增日志输出开关。

### P1-002 — 生产领域模型与迁移 ADR

- 状态：`READY`
- 所属模块：Production data architecture
- 优先级：P1 / 风险：High
- 任务目标：完成可施工的数据库选型、schema、迁移和对象存储 ADR；不接入或写入生产数据库。
- 当前问题：画布 JSON 和 `mediaTasks` JSON 不是项目/剧集/镜头/审批/产物/成本的事务事实源；SQLite/MySQL 多租户 schema 不覆盖这些实体。
- 涉及文件：新增 `docs/adr/*`、schema 设计文档；后续实际实现另开任务。
- 实施范围：设计 Project/Episode/ScriptVersion/Character/Asset/Shot/TaskAttempt/Approval/Render/Cost/AuditEvent 的标识、版本、保留、迁移、幂等和访问边界。
- 前置依赖：P0-003。
- 验收标准：数据模型能支持单镜头重试、产物不可变版本、审批回滚、项目隔离和恢复；选择 SQLite/数据库的明确适用边界。
- 必须执行的测试：架构评审；不得连接生产 DB。
- 交付物：ADR、ERD、迁移/回滚模板、实施分解更新。
- Git commit 证据：待完成填写 SHA。
- 失败恢复方式：文档任务，回退 commit 即可。

## P2：本地模型及媒体模型集成（前置验收未完成，暂阻塞）

### P2-001 — Qwen/MLX 受控 provider profile

- 状态：`BLOCKED`
- 所属模块：AI integration
- 优先级：P2 / 风险：High
- 任务目标：把现有稳定 MLX Qwen 作为受控 OpenAI-compatible provider，建立模型能力/限额/并发/超时与非付费验收记录。
- 当前问题：通用 provider 可配置，但无 Qwen 专属 capability profile 或稳定性证据。
- 涉及文件：`apps/server/src/utils/ai/provider.ts`、`models.ts`、设置 UI、集成文档。
- 实施范围：Toonflow 侧配置与测试；绝不修改/重启 Studio MLX 或 OpenCode。
- 前置依赖：P0-001、P0-002、P1-001；用户提供允许调用现有模型的验收窗口。
- 风险等级：High。
- 验收标准：模型发现、流式、取消、JSON schema、工具调用、长上下文、并发拒绝均有真实脱敏证据；错误不泄密。
- 必须执行的测试：只读 `/v1/models` 和经批准的小型非付费请求矩阵。
- 交付物：profile、能力矩阵、运行手册。
- Git commit 证据：待完成填写 SHA。
- 失败恢复方式：移除 Toonflow profile，不改 Studio 服务。

### P2-002 — ComfyUI 图片 provider 与一致性资产合同

- 状态：`BLOCKED`
- 所属模块：Image generation
- 优先级：P2 / 风险：High
- 任务目标：新增明确版本的 ComfyUI adapter、workflow 输入 schema、异步队列/恢复和角色参考资产合同。
- 当前问题：源码未见 ComfyUI 实现；README 文案不能作为功能证据。
- 涉及文件：新增 provider、`packages/providers/types.d.ts`（仅必要时）、媒体任务/节点设置、文档。
- 实施范围：只接入明确的本地或隔离测试 ComfyUI 实例；禁止安装/启动/改动 Mac Studio 现有模型服务。
- 前置依赖：P0-002、P1-001、P1-002；经批准的 ComfyUI API/workflow/硬件容量基线。
- 风险等级：High。
- 验收标准：文本/参考图/角色 ID 的输入输出可复现；submit/poll/resume/cancel；20 次连续测试的成功率和内存证据；失败不重复扣费/提交。
- 必须执行的测试：隔离实例的模拟与真实小图测试；不得使用生产素材。
- 交付物：adapter、workflow schema、基准记录、隐私说明。
- Git commit 证据：待完成填写 SHA。
- 失败恢复方式：禁用 provider；保留任务证据，不清理用户素材。

### P2-003 — 主视频供应商能力合同

- 状态：`BLOCKED`
- 所属模块：Video generation
- 优先级：P2 / 风险：High
- 任务目标：选择一个已合同化视频供应商，形成模型/时长/分辨率/参考/价格/重试/数据条款的能力合同与验收。
- 当前问题：三套内置适配器存在，但无生产供应商选择、SLA、配额或成本实测。
- 涉及文件：`packages/providers/src/media/*`、媒体设置、运维文档。
- 实施范围：优先复用已有适配器；未经批准不得调用付费 API。
- 前置依赖：P0-002、P1-001；用户批准供应商、测试预算、账号与条款。
- 风险等级：High。
- 验收标准：异步 submit/poll/resume/cancel、超时/失败分类、费用快照、输出完整性、供应商故障人工接管均有证据。
- 必须执行的测试：经批准的最小付费/沙盒矩阵。
- 交付物：能力合同、供应商准入记录、成本基线。
- Git commit 证据：待完成填写 SHA。
- 失败恢复方式：停止新提交、保留 taskId/账单证据、按供应商流程处理。

### P2-004 — TTS/ASR 隐私与角色音色方案

- 状态：`BLOCKED`
- 所属模块：Audio
- 优先级：P2 / 风险：High
- 任务目标：选定可商用 TTS/ASR，建立角色音色授权、台词输入、时间码和数据流政策。
- 当前问题：浏览器 ASR 可能上传音频；没有角色声音资产、字幕产物或授权记录。
- 涉及文件：`packages/asr/*`、`audioGenerationNode`、新增生产领域服务/文档。
- 实施范围：先设计和隔离验证；不处理真实敏感语音。
- 前置依赖：P0-002、P1-002；法务/供应商选择。
- 风险等级：High。
- 验收标准：输出带可追溯 voice/license/version；生成 SRT/ASS 的时码质量可人工复核；敏感数据去向可说明。
- 必须执行的测试：经批准的合成语音样本验证。
- 交付物：音频能力合同、授权台账、验收样例。
- Git commit 证据：待完成填写 SHA。
- 失败恢复方式：禁用自动 ASR/TTS，继续人工字幕/配音流程。

## P3：完整短剧生产流水线

### P3-001 — 项目、剧集、脚本与资产版本领域服务

- 状态：`BLOCKED`
- 所属模块：Production domain
- 优先级：P3 / 风险：High
- 任务目标：实现经 ADR 批准的项目/剧集/剧本/角色/场景/资产不可变版本与审批状态。
- 当前问题：画布 JSON 不提供生产版本、隔离或审批审计。
- 涉及文件：由 P1-002 ADR 确定的新服务、迁移、API 与最小 UI。
- 实施范围：先最小 vertical slice；不迁移真实项目。
- 前置依赖：P1-002、P2-001、P2-004。
- 风险等级：High。
- 验收标准：一个项目可拥有多集、版本、审批和引用冻结；画布可关联而非替代领域记录。
- 必须执行的测试：迁移到临时数据库、API/手工 UI 流程、回滚验证。
- 交付物：schema、迁移、API 契约、操作说明。
- Git commit 证据：待完成填写 SHA。
- 失败恢复方式：回滚迁移（仅临时/非生产）；保留旧画布不破坏。

### P3-002 — 结构化分镜与连续性合同

- 状态：`BLOCKED`
- 所属模块：Storyboard
- 优先级：P3 / 风险：High
- 任务目标：将文字分镜转为 Shot schema，强制关联角色、服装、场景、道具、台词、运镜、时长和参考版本。
- 当前问题：`storyboardTeam` 只产生可审阅文本，未形成领域对象或执行依赖。
- 涉及文件：`packages/teams/storyboardTeam/*`、新 schema/service、画布适配器。
- 实施范围：复用现有 team 提示词/continuity 知识，不把 Agent 输出直接当批准生产指令。
- 前置依赖：P3-001、P2-001。
- 风险等级：High。
- 验收标准：每镜头可独立批准/返工/重试；跨镜角色和场景引用稳定；AI 输出经过 schema 校验。
- 必须执行的测试：固定样例剧本的人工审阅与 schema 失败路径。
- 交付物：shot schema、导入/审阅 UI、连续性报告。
- Git commit 证据：待完成填写 SHA。
- 失败恢复方式：保留原文本分镜，禁止写入生产镜头状态。

### P3-003 — 镜头资产生成与人工审核闭环

- 状态：`BLOCKED`
- 所属模块：Image/video production
- 优先级：P3 / 风险：High
- 任务目标：把批准的 shot 变为可追溯图片/视频生成请求、候选产物、人工选择与冻结版本。
- 当前问题：当前节点能生成媒体，但没有项目级任务、候选管理、审批或成本归集。
- 涉及文件：媒体节点适配层、P3 数据服务、任务/审批 UI。
- 实施范围：单项目/单镜头 MVP；不做批量自动调度。
- 前置依赖：P2-002 或 P2-003（按所选媒介）、P3-002、P4-001。
- 风险等级：High。
- 验收标准：重试不会误覆盖批准资产；每输出有输入哈希、provider taskId、成本与审批记录；可人工中止/恢复。
- 必须执行的测试：隔离项目中的一镜头端到端验收。
- 交付物：生成记录、审批 UI、恢复 runbook。
- Git commit 证据：待完成填写 SHA。
- 失败恢复方式：任务标记失败/待人工；保留已产生候选，不删除。

### P3-004 — 确定性成片合成与字幕导出

- 状态：`BLOCKED`
- 所属模块：Render/export
- 优先级：P3 / 风险：High
- 任务目标：按 immutable render manifest 以 FFmpeg 合成视频、音频、字幕并导出指定母版。
- 当前问题：只有通用 FFmpeg 工具和局部处理；没有 timeline、concat、字幕烧录、渲染任务或成片版本。
- 涉及文件：`packages/ffmpeg/*`、`packages/tools/ffmpeg/*`、新 render service/worker、生产数据服务。
- 实施范围：一个明确分辨率/帧率/响度规范；不覆盖原始片段。
- 前置依赖：P3-001、P3-002、P2-004、P4-001。
- 风险等级：High。
- 验收标准：相同 manifest 产出同一可验证结果；有音视频时长/编码/字幕检查；失败可恢复、不覆盖原件。
- 必须执行的测试：小型临时媒体 fixture 的手动渲染与 ffprobe 验收。
- 交付物：manifest schema、render worker、导出说明。
- Git commit 证据：待完成填写 SHA。
- 失败恢复方式：取消 worker、保留 manifest/日志，删除仅执行者生成的临时输出。

## P4：批量调度与失败恢复

### P4-001 — 持久化 DAG 队列与任务租约

- 状态：`BLOCKED`
- 所属模块：Orchestration
- 优先级：P4 / 风险：Critical
- 任务目标：建立项目级 DAG、任务状态机、幂等键、租约、优先级、最大并发、重试/死信和人工接管。
- 当前问题：`mediaTasks` 只恢复单次媒体任务，没有跨节点依赖和多进程安全调度。
- 涉及文件：新 orchestrator 服务/数据库 schema/worker API；仅复用现有 mediaTasks adapter。
- 实施范围：先单 worker、单项目受控队列；不复制 ACC 架构。
- 前置依赖：P1-002、P1-001、P3-001。
- 风险等级：Critical。
- 验收标准：服务崩溃/重复投递/worker 丢失后任务不会双执行或丢失；依赖不满足不会提前运行；有 operator retry/cancel。
- 必须执行的测试：临时 DB 的故障注入、重启、幂等、并发上限、死信手测。
- 交付物：状态机、worker contract、恢复 runbook、指标。
- Git commit 证据：待完成填写 SHA。
- 失败恢复方式：停止 worker、冻结新任务、按租约过期和审计记录人工恢复。

### P4-002 — 多项目/多集生产看板

- 状态：`BLOCKED`
- 所属模块：Operations UI
- 优先级：P4 / 风险：Medium
- 任务目标：可查看项目/剧集/镜头 DAG、状态、阻塞、审批、重试和预计成本。
- 当前问题：现有画布是单项目创作 UI，缺少运营进度视图。
- 涉及文件：Web、production API、orchestrator query API。
- 实施范围：只读看板先行，再实现操作按钮。
- 前置依赖：P4-001、P3-001。
- 风险等级：Medium。
- 验收标准：看板与数据库状态一致；无权限用户不泄露项目/成本；可定位每个失败任务。
- 必须执行的测试：两项目隔离的手工验收。
- 交付物：看板、权限矩阵、操作说明。
- Git commit 证据：待完成填写 SHA。
- 失败恢复方式：关闭新看板入口，不改变任务事实数据。

### P4-003 — 并发、配额与供应商熔断策略

- 状态：`BLOCKED`
- 所属模块：Resource control
- 优先级：P4 / 风险：High
- 任务目标：按 provider/model/project 设置并发、速率、预算、超时、熔断和人工降级。
- 当前问题：无全局并发控制；不可假设 Mac Studio 可并行承受文字与视频工作负载。
- 涉及文件：orchestrator、provider adapters、成本服务、部署配置。
- 实施范围：策略配置和可观测性；不改变 MLX 服务参数。
- 前置依赖：P2-001、P2-003、P4-001。
- 风险等级：High。
- 验收标准：超限任务排队而非提交；供应商失败触发熔断；Qwen 并发严格遵守批准配置。
- 必须执行的测试：隔离/模拟供应商压力测试，禁止生产付费轰炸。
- 交付物：策略表、告警阈值、演练记录。
- Git commit 证据：待完成填写 SHA。
- 失败恢复方式：将策略恢复到保守并发 0/1，停止新媒体提交。

## P5：质量、成本与运营

### P5-001 — 自动技术 QC 与人工质量门

- 状态：`BLOCKED`
- 所属模块：Quality control
- 优先级：P5 / 风险：High
- 任务目标：对产物检查时长、编码、黑帧、静音、音量、字幕安全区、参考一致性，并设置人工批准门。
- 当前问题：`frameAnalysisNode` 可分析视频，但不是交付 QC，且无阈值/拦截决策。
- 涉及文件：`frameAnalysisNode` 复用层、新 QC service、render/orchestrator。
- 实施范围：先技术 QC；角色/风格一致性采用人工复核并记录，不承诺模型评分正确性。
- 前置依赖：P3-004、P4-001。
- 风险等级：High。
- 验收标准：QC 报告可复现；失败 render 不能自动发布；人工覆写有理由/操作者审计。
- 必须执行的测试：已知合格/不合格临时媒体样例。
- 交付物：QC schema、阈值、报告 UI。
- Git commit 证据：待完成填写 SHA。
- 失败恢复方式：切换为人工审核，不阻断已批准旧产物。

### P5-002 — 成本账本与预算控制

- 状态：`BLOCKED`
- 所属模块：FinOps
- 优先级：P5 / 风险：High
- 任务目标：按项目/剧集/镜头/供应商记录估算、预占、实付、重试和存储成本，并在预算超限时阻断。
- 当前问题：多租户钱包存在，但不是创作成本归集；provider 价格很多是展示/未验证信息。
- 涉及文件：production data、provider adapter、billing/gateway 适配、运营 UI。
- 实施范围：不改变真实钱包语义，先建立只读/影子账本再切换。
- 前置依赖：P2-003、P3-003、P4-001。
- 风险等级：High。
- 验收标准：每个付费 task 有可审计账单证据；预算阻断幂等；估算与实付区别明确。
- 必须执行的测试：模拟账单、重复回调、失败/部分交付结算。
- 交付物：成本模型、报表、预算策略。
- Git commit 证据：待完成填写 SHA。
- 失败恢复方式：回退为只读影子账本，禁止自动扣费决策。

### P5-003 — 素材生命周期、授权与审计

- 状态：`BLOCKED`
- 所属模块：Governance
- 优先级：P5 / 风险：High
- 任务目标：为素材、声纹、模型输出、许可证、保留期、删除审批和数据主体请求建立可审计管理。
- 当前问题：全局 assets 和工作区文件无生产级引用计数、授权元数据、保留或删除控制。
- 涉及文件：production data、asset service、运维/法务文档。
- 实施范围：先元数据与审批，不执行批量删除。
- 前置依赖：P0-003、P3-001。
- 风险等级：High。
- 验收标准：每个发布资产可定位来源/许可证/项目引用；清理仅生成候选清单，执行需显式批准。
- 必须执行的测试：临时资产的引用、保留、审批和恢复演练。
- 交付物：授权台账、生命周期策略、审计查询。
- Git commit 证据：待完成填写 SHA。
- 失败恢复方式：关闭自动清理，保留所有数据。

## P6：性能与生产验收

### P6-001 — 容量、故障与恢复演练

- 状态：`BLOCKED`
- 所属模块：Production readiness
- 优先级：P6 / 风险：High
- 任务目标：基于真实但脱敏的工作负载测量吞吐、延迟、失败率、磁盘增长、恢复时间和成本。
- 当前问题：没有容量/SLO/灾难恢复实测；不得用设备配置推断。
- 涉及文件：部署配置、观测、runbook、压测资产（临时且不可提交敏感素材）。
- 实施范围：隔离环境；不打扰 MLX/OpenCode 稳定生产服务。
- 前置依赖：P4-001、P4-003、P5-001、P5-002、P5-003。
- 风险等级：High。
- 验收标准：达成预先批准 SLO；演练 worker 失效、服务重启、供应商超时、磁盘阈值和备份恢复。
- 必须执行的测试：批准的容量/故障矩阵。
- 交付物：容量报告、SLO、恢复证据、剩余风险接受书。
- Git commit 证据：待完成填写 SHA。
- 失败恢复方式：停止演练、回收临时资源、保留证据后降级人工流程。

### P6-002 — 发布门禁与商业上线评审

- 状态：`BLOCKED`
- 所属模块：Release governance
- 优先级：P6 / 风险：High
- 任务目标：建立安全、版权/隐私、质量、成本、恢复、供应商与运营负责人共同签署的上线门禁。
- 当前问题：无生产发布标准和责任矩阵。
- 涉及文件：发布文档、CI/部署配置（如获批准）、运营手册。
- 实施范围：先定义门禁；不自动发布。
- 前置依赖：P6-001。
- 风险等级：High。
- 验收标准：每次上线有版本、迁移、回滚、备份、SLO、法务与负责人证据；未满足门禁不能发布。
- 必须执行的测试：发布/回滚桌面演练。
- 交付物：release checklist、RACI、签署记录模板。
- Git commit 证据：待完成填写 SHA。
- 失败恢复方式：保持当前版本和人工运营流程。

## 启动条件

- 当前结论：`NOT_READY_FOR_OPENCODE`。
- 变更为 `READY_FOR_OPENCODE` 的最低条件：`P0-001`、`P0-002`、`P0-003`、`P1-001`、`P1-002` 全部为 `COMPLETED`，且每项有真实验收与 commit 证据；随后只能开始相应依赖已满足的一项 P2 任务。
