# Mac Studio 模型互斥加载与自动恢复：候选方案

> 日期：2026-11-15；状态：`RESEARCHED` / `ARCHITECTURE_PROPOSED`。本文件没有停止、重启、卸载、下载或调用任何模型，也没有修改 MLX Serve、LaunchDaemon、OpenCode、端口或模型配置。它不是对现有 Studio 服务的运行验证。

## 1. 目标、非目标和结论

目标是在**未来获得明确授权和独立验证后**，让 Mac mini 上的 Toonflow 控制平面把 Mac Studio 64GB 统一内存作为单一、可租约的重推理资源：文本批次使用现有 Qwen；媒体批次在文本排空并确认 Qwen 已卸载后运行；媒体结束后恢复**完全相同**的 Qwen 服务配置。

非目标：本方案不让图片/视频与 Qwen 并发，不修改稳定 :8000 的模型参数/路径/端口，不以 `SIGSTOP` 冒充卸载，不把“进程名消失”当作统一内存已释放，也不允许 Toonflow 直接 kill Studio 上的任意 PID。

**结论**：可设计为可恢复的互斥调度，但当前不能宣称可用。最大的未知是当前 `com.mlx-serve.8000` 的实际二进制、版本、启动参数和支持的生命周期 API。服务名 `mlx-serve` 不是能力合同。必须先完成只读能力盘点和隔离测试；在此之前，最安全策略仍是 Qwen 常驻 + 本地图片/视频不启动 + 云端/动态漫画作为将来可选路径。

## 2. MLX Serve 生命周期调查

### 2.1 已知资料与不可假设事项

| 服务实现可能性 | 公开资料 | 对“主 Qwen 卸载”的含义 | 当前状态 |
| --- | --- | --- | --- |
| `mlx_lm.server`（MLX 官方语言服务器） | [MLX Local AI 文档](https://docs.mlx.app/local/overview) 说明一条命令服务一个模型；[SERVER.md](https://github.com/ml-explore/mlx-lm/blob/main/mlx_lm/SERVER.md) 记录的 `draft_model: null` 是卸载**draft**模型 | 未找到官方记录的主模型通用 `/unload` 端点；主模型生命周期应按进程启动/退出设计，不能假定热卸载 | `RESEARCHED`；不等于本机实际实现。 |
| 第三方 `raspoli/mlx-serve` | [官方仓库/API](https://github.com/raspoli/mlx-serve) 声称有 `/v1/load-model`、`/v1/unload-model` | 若当前二进制、版本、认证和端点经本机只读检查确认，可作为候选原生卸载路径；仍须验证内存、并发和故障行为 | `COMPATIBLE_ON_PAPER`，未确认当前服务。 |
| 其他名为“MLX Serve”的封装或自定义 LaunchDaemon | 服务名不能识别实现 | 不能安全推断 API、PID 关系、KeepAlive 行为或配置重放方式 | `UNVERIFIED`。 |

因此，对问题“现有 MLX Serve 是否支持不停止进程卸载模型”的唯一诚实答案是：**尚未验证；若它是官方 `mlx_lm.server`，公开文档没有主模型热卸载合同；若它是特定支持 unload 的实现，必须按锁定版本在本机验证。** 不可发送探测性 unload 请求到稳定 :8000。

### 2.2 LaunchDaemon/KeepAlive 的约束

现有稳定服务被描述为 system-domain LaunchDaemon，`RunAtLoad`、`KeepAlive` 已配置。一个简单 `kill` 或退出主进程可能触发 launchd 立即重启；而 `SIGSTOP` 会保留进程、连接和统一内存，不能释放模型。未来的 controller 必须只使用经批准的服务所有者 API 或明确的 launchd 控制流程，绝不能在 Toonflow worker 中散落 shell kill 命令。

要实现安全的进程级切换，建议在**单独批准的基础设施改造**中完成以下二选一，不是本轮动作：

1. **原生生命周期模式（优先）**：经过验证的服务提供受认证的 load/unload/status API，固定的 :8000 ingress 在模型切换期间保持，controller 只调用该服务的支持接口。
2. **受控 child-process 模式（备选）**：一个 Studio 本机 supervisor 成为模型进程唯一所有者；它保存已批准的 Qwen 启动 manifest，启动/等待/退出模型 child。稳定 :8000 对外由受控 proxy 保持，或由 supervisor 自己绑定。此模式需要把现有 LaunchDaemon 所有权迁移给 supervisor，并取得用户明确授权；不能在现有 KeepAlive 服务旁“多启动一个”。

在任何模式下，Qwen 的模型 ID/本地路径、量化、端口 :8000、启动参数、权限与健康检查样例必须从只读的 `QwenServiceManifest` 获取并以 hash 锁定。媒体任务无权编辑该 manifest。配置 hash 不匹配时，恢复必须 `BLOCKED`，而不是猜测参数。

## 3. 推荐架构（`PROPOSED`）

```
Mac mini / Toonflow control plane
  ├─ Production scheduler ───────────── durable DB: jobs, leases, state, audit
  ├─ Text ingress policy (accept/drain/retry) ────────┐
  └─ StudioResourceClient (mutual-TLS/token, idempotency key) │
                                                               ▼
Mac Studio (future, separate approved supervisor)
  ├─ Resource Controller: state machine + host lock + watchdog
  ├─ QwenServiceManifest (read-only, hashed) ─────── Qwen child / native lifecycle
  ├─ MediaServiceManifest(s) (pinned, separately approved) ─ Media child
  ├─ local status socket (not public) / structured event journal
  └─ :8000 compatibility ingress → only healthy Qwen when TEXT_ACTIVE
```

**职责分离**：Mini 的 scheduler 决定何时申请媒体时隙；Studio 的 controller 是唯一能改变模型生命周期的主体。controller 不读剧本、不保存 API key、不选择模型；它只接受已批准 manifest ID、不可重复的切换事务 ID 和时间上限。现有稳定 Qwen 继续由当前服务管理，直至另行批准迁移；本方案不要求、也不授权现在部署 controller。

## 4. 生命周期状态机与互斥锁

### 4.1 主状态

```
TEXT_ACTIVE
  → TEXT_DRAINING
  → TEXT_UNLOADED
  → MEDIA_LOADING
  → MEDIA_RUNNING
  → MEDIA_UNLOADING
  → TEXT_RESTORING
  → TEXT_ACTIVE

任一状态 ─→ RECOVERY_REQUIRED ─→ (reconcile) TEXT_RESTORING 或 SAFE_OFFLINE
```

| 状态 | 允许动作 | 退出条件 / 超时 | 禁止动作 |
| --- | --- | --- | --- |
| `TEXT_ACTIVE` | 接受经入口代理的文本任务；媒体任务排队 | 获得媒体批次 lease 后进入 draining | 启动重媒体模型。 |
| `TEXT_DRAINING` | 拒绝新文本（可重试响应）；等待已登记请求完成 | inflight=0 且持久化；超时则取消切换，回 `TEXT_ACTIVE` | 强杀仍在生成的文本、卸载 Qwen。 |
| `TEXT_UNLOADED` | 校验 Qwen 停止/主模型卸载证据 | 资源释放门通过后加载媒体 | 接受文本，或假定退出即已释放。 |
| `MEDIA_LOADING` | 启动已批准的单一媒体 manifest | 健康检查+容量门通过；失败进入恢复 | 同时启动两个媒体模型或 Qwen。 |
| `MEDIA_RUNNING` | 只执行该批次已冻结的媒体任务 | 批次完成/取消/超时 | 接受 Qwen 任务、绕开资源锁。 |
| `MEDIA_UNLOADING` | 先结束媒体队列，再按支持机制释放 | 无活跃 job、退出/卸载和释放证据 | 直接把 Qwen 与媒体重叠。 |
| `TEXT_RESTORING` | 用 QwenServiceManifest 原样恢复 | 模型列表、健康、确定性 probe 成功 | 先宣布可用或加载未知配置。 |
| `RECOVERY_REQUIRED` | 拒绝新重任务，写告警、执行只读发现/受控 reconcile | 唯一运行模型被识别，manifest hash 匹配 | 自动猜测、并发重启、删除审计记录。 |

状态、转移原因、actor、manifest hash、lease、时间戳、inflight 计数、进程/原生 API 证据、内存快照及错误分类必须写到 Mini 上持久数据库和 Studio 本地 append-only journal。Mini 不可用时 Studio 不能自行切换模型；Studio 重启时不应依据易失内存宣布任何状态。

### 4.2 锁、幂等与避免重复启动

- **全局 Studio heavy-resource lease**：只允许一个 controller 事务持有，带 fencing token 和 TTL；过期 lease 不能由旧 holder 写新状态。
- **模型槽锁**：`TEXT_SLOT` 与 `MEDIA_SLOT` 是互斥锁，必须在启动进程前原子获取、在进程退出/原生确认后释放。
- **切换事务 ID**：`switchId`/idempotency key 贯穿 Mini 与 Studio；重复请求返回同一状态，不会第二次启动。
- **任务冻结**：进入 `TEXT_DRAINING` 后不接收新 text job；媒体批次在进入 `MEDIA_LOADING` 前固定输入和预算，避免任务持续涌入延长停机。
- **看门狗**：controller 自己的 PID/锁/监听 socket 也须被检查；只有一个 controller instance 可以绑定其本地控制 socket。

## 5. 排空、恢复和崩溃回滚

### 5.1 不误杀文本任务

要排空必须先让所有使用 Qwen 的调用走可观测 ingress 或登记 API；否则 controller 不知道是否有直接客户端、流式连接或后台 job。切换前执行：停止取新 text 队列 → ingress 返回“暂不可用、可重试” → 等待已登记请求完成/客户端取消 → 到达批准的 drain deadline。deadline 到达仍有请求时，**默认取消本次媒体切换并恢复文本接受**；不能杀 Qwen 以换取媒体优先级。仅在未来明确的管理员策略和已通知调用者下，才可有单独的 request cancellation 路径。

其他项目若直连 :8000，会绕过该保护并在切换期失败。因此先决条件是盘点所有依赖者，把它们迁移到 ingress 或接受维护窗口；在未完成前，此方案为 `NO-GO`。

### 5.2 确认统一内存的释放

macOS 的“空闲内存”不是充分或稳定的卸载证明；系统会缓存内存。Apple 的 Activity Monitor 说明内存压力综合 free、swap、wired 和 file cache。[Apple 文档](https://support.apple.com/en-euro/guide/activity-monitor/-actmntr1004/mac)。未来验收应组合证据：

1. 服务的原生 unload/status 或 child 退出已确认；不只看端口不可达。
2. 无 Qwen/媒体 worker 存活，且端口/进程树符合预期。
3. 切换前/后记录 memory pressure、swap、wired/compressed、目标进程内存和 GPU/系统指标；压力恢复到预定义安全区且无持续 swap 增长。
4. 启动目标模型后连续生成样例成功；若不能加载，则前一模型未充分释放或容量不足，记录为失败。

这些指标能证明“达到可运行门”，不能数学上证明全部 cache 立即归零。`SIGSTOP` 绝不满足第 1 条。

### 5.3 恢复顺序

媒体加载、运行或卸载任一步失败时：停止接受新媒体 → 保留任务状态和错误 → 尝试按 manifest 卸载已知媒体 → 验证无重媒体进程 → 用不变的 Qwen manifest 恢复 → 完成 `/v1/models`、健康 probe 和一条固定非敏感文本测试 → 才将 TEXT ingress 标为 active。恢复失败进入 `SAFE_OFFLINE`/`RECOVERY_REQUIRED` 并告警；不得自动改量化、模型路径、端口、上下文或重试无限次。

Studio 重启后的 reconcile：读取 Mini DB 与 Studio journal → 检查实际服务/进程/端口 → 若没有可证明的 Qwen 健康态，保持 `RECOVERY_REQUIRED`，由经批准的恢复程序按原 manifest 恢复；若媒体批次中断，任务回到 `UNKNOWN_REMOTE_STATE`，先查询媒体 provider/产物 hash 再决定续跑，不能直接重扣费用。

## 6. 批量生产调度策略

频繁逐镜头切换会把加载、卸载、健康检查和风险放大。建议把流水线拆为有人工门的批次：

```
TEXT BATCH: Series/Season/Episode/ShotSpec（Qwen）
  → review + asset freeze
IMAGE BATCH: 所有已批准关键帧/角色/场景（一个图片模型）
  → review + shot selection
VIDEO BATCH: 仅高运动且已批准的 I2V 镜头（一个视频模型或云端）
  → audio/subtitle/render batch（Mini/轻量资源）
```

默认滚动窗口为 1–3 集，而不是把 100 集一次性投递。调度器按模型切换代价、任务数、预算、到期时间、人工审批和媒体输入是否齐全进行 batching；每批次声明 `maxWallClock`、`maxAttempts`、`maxCost` 和 `returnToTextDeadline`。任何媒体失败都只影响对应镜头/批次，不重跑整季。

## 7. 将来实测计划（需单独授权）

前提：选择具体服务实现与 pinned 版本；记录当前 Qwen manifest；确认全依赖方、维护窗口、备份、回滚 runbook；无敏感资产；有物理/远程控制台可恢复。**以下每次只变更一个变量，且不能由 OpenCode 自动执行。**

| 步骤 | 目的 | 必录证据 | 失败处理 |
| --- | --- | --- | --- |
| V0 只读盘点 | 确认二进制、版本、LaunchDaemon plist、实际参数、支持 API 和依赖方 | manifest hash、服务所有者、端口/health、功能矩阵 | 不明则停止，不探测 unload。 |
| V1 文本基线 | 建立未切换 Qwen 的健康/延迟/内存参考 | models、固定 prompt、inflight、系统内存指标 | 不稳定则不研究切换。 |
| V2 排空 | 验证拒绝新请求而完成已登记文本 | drain 时长、请求结果、无误杀 | 超时即回 `TEXT_ACTIVE`。 |
| V3 Qwen 卸载 | 只验证经确认支持的生命周期方式 | 卸载耗时、进程/端口/内存压力/swap | 立即按原 manifest 恢复，失败升级人工。 |
| V4 图片单任务 | 验证一个批准图片模型的加载、单图、卸载 | 加载/生成/卸载时间、峰值内存、产物 hash | 卸载→Qwen 恢复；不得继续批量。 |
| V5 视频单任务 | 验证一个批准视频模型的 5 秒 I2V | 同上 + 时序 QC、取消与超时 | 失败回滚，不改 Qwen 参数。 |
| V6 循环稳定性 | 连续多次 text→media→text 切换 | 成功率、泄漏趋势、恢复耗时、依赖方影响 | 任一漂移/恢复失败即 No-Go。 |

## 8. Go/No-Go 标准

| 门 | Go | No-Go |
| --- | --- | --- |
| 生命周期能力 | 当前服务实现、原生 API 或 child supervisor 行为已锁定并有回滚 | 只能靠 kill/SIGSTOP、服务/API 不明、KeepAlive 行为不明。 |
| 文本保护 | 所有调用方可排空，drain 不丢/不误杀任务，Qwen 复原后固定文本 probe 通过 | 存在绕过 ingress 的未知依赖方，或恢复改变了 manifest。 |
| 资源 | 卸载/加载后压力、安全 swap 和目标模型单任务均在批准阈值内 | 持续黄/红 memory pressure、swap 增长、任一模型 OOM/卡死。 |
| 媒体生产 | 生成、取消、恢复、可追溯输入/输出和商业许可都通过 | 只有一次样片、没有错误处理/API/许可证据。 |
| 批量 | 资源 lease、fencing、崩溃 reconcile、成本/时限都通过 | 同时模型、重复启动、任务孤儿、不能恢复。 |

任何 No-Go 的默认动作是：停止媒体切换、恢复或保持稳定 Qwen 服务、将媒体任务留在队列/转为经批准云端方案；不得强制重启、升级 MLX、自动更换模型或修改 Qwen 配置。

## 9. 候选任务（不写入 ToDo.md）

| 候选 ID | 优先级 | 范围与独立验收 | 前置依赖 | 执行/决策边界 |
| --- | --- | --- | --- | --- |
| CAND-RESOURCE-001 | P0 | 只读服务能力盘点与 QwenServiceManifest：二进制/版本/启动参数/launchd/health/依赖方，不含控制命令。验收：能力矩阵和 hash 化基线经人工确认。 | P0-001、P0-002 | OpenCode 可收集只读证据；人工批准任何系统控制。 |
| CAND-RESOURCE-002 | P1 | 资源 controller ADR：原生 unload vs child-process supervisor、固定 ingress、权限、journal/DB、恢复/报警。验收：威胁模型、状态机和回滚 runbook 获批。 | CAND-RESOURCE-001、P1-001、P1-002 | OpenCode 可起草；**人工**选择模式与基础设施迁移。 |
| CAND-RESOURCE-003 | P1 | 排空/lease/幂等/切换模拟器和契约测试，无 Studio 控制。验收：状态转换、超时、重复请求、崩溃恢复测试。 | CAND-RESOURCE-002、P4-001 | 适合 OpenCode；不涉及 MLX 服务。 |
| CAND-RESOURCE-004 | P2 | 经批准的隔离生命周期基准 runbook 与采集器。验收：V0–V3 可在维护窗口人工执行并产出数据；无自动切换。 | CAND-RESOURCE-001～003、P0-004、P1-003 | **人工操作**；OpenCode 只可在批准后实现非破坏性记录工具。 |
| CAND-RESOURCE-005 | P2 | 图片/视频单任务切换验收（V4/V5）和 Qwen 回归。验收：满足 Go 门或明确 No-Go。 | CAND-RESOURCE-004、P2-005、P2-006/P2-007 | **人工授权/执行**；不得由 worker 自动下载/启动模型。 |

这些是候选任务，不改变现有 `ToDo.md` 的 28 项或状态。应在用户批准后与 P0-004、P1-003、P2-005～P2-007、P4-001 去重映射。

## 10. 对账增补：可观测性硬缺口与强制规则（2026-11-15，CodexQ 复核）

本节把 §1–§2 的谨慎判断升级为**源码级证据**，并确立一条不可绕过的强制规则。完整对账见 [架构对账报告](./TOONFLOW_ARCHITECTURE_RECONCILIATION.md) §8。

### 10.1 「能否判断其他项目正在使用 Qwen」——答案是否定的

| 判据 | 源码/环境事实 | 状态 |
| --- | --- | --- |
| Toonflow 是否统计自己的在途文本请求 | **否**。`apps/server/src/utils/ai/`（`provider.ts`、`models.ts`、`billingUsage.ts`、`modelContextLimits.ts`、`files.ts`、`requestFiles.ts`）与 `apps/server/src/routes/ai/` 全文 grep `inflight\|concurren\|semaphore\|activeRequests\|rateLimit` **零命中**；`packages/multiTenant` 亦零命中 | `SOURCE_VERIFIED`（缺失） |
| Toonflow 能否观测其他客户端 | **不能**。OpenCode / codexm / codexg 及其他脚本按基础设施约定直连 `http://192.168.0.10:8000/v1`，**完全绕过 Toonflow**；Toonflow 既不是反向代理也不是网关 | `SOURCE_VERIFIED`（架构事实） |
| `/v1/models` 能否当负载指标 | **不能**。`apps/server/src/utils/ai/models.ts:12-26` 的 `fetchProviderModels` 只是 30s 超时的存活/清单探测，不返回在途请求数、KV 占用或内存状态 | `SOURCE_VERIFIED` |
| Toonflow 能否排空**自己**的任务 | **部分可以**。`apps/server/src/agent/runtime/sessions.ts:80-101`：`activeRuns` 在请求进入时即登记，`stopAgentRuns(cwd, runIds)` 可 abort 并等待结束；`activeSessions` 按工作区键索引。**但**仅进程内有效（代码注释：「多进程部署时需共享会话所有权」「进程退出后不恢复任务」），且**不覆盖**直接调用 `/api/ai/generate` 的非 Agent 文本请求 | `SOURCE_VERIFIED` |
| 是否有全局资源锁 | **没有**。唯一锁是进程内按路径的 `lockWorkspaceFiles`（`apps/server/src/utils/workspace/files.ts:133`），语义是文件写入互斥，不是模型资源互斥 | `SOURCE_VERIFIED`（缺失） |

### 10.2 强制规则（优先级高于任何吞吐优化）

> **在同时满足以下三项之前，禁止任何自动卸载 Qwen 的行为；模型切换一律进入人工审批 + 维护窗口流程。**
>
> 1. Studio 侧存在**全客户端可见**的在途请求计数（或所有客户端改走统一 ingress），使「空闲」可被客观判定；
> 2. `QwenServiceManifest`（模型 ID/本地路径/量化/端口 `:8000`/启动参数/健康检查样例）已只读盘点并 hash 锁定，且经验证的生命周期方式（原生 load/unload API，或受批准的本机 supervisor 作为唯一进程所有者）已确认可用；
> 3. 用户对该次切换给出**具体**授权（时间窗、范围、回滚条件）。

违反任一项时的默认动作：保持 Qwen 常驻，媒体任务留在持久队列，或转经人工批准的云端路径。**不得**以 `kill`（LaunchDaemon `KeepAlive` 会立即重启）或 `SIGSTOP`（保留进程与统一内存，不释放模型）冒充卸载。

### 10.3 对验证关卡的修订

- `V2 排空` 的前置条件修订为：**ADD-003（Studio 在途可观测性或统一 ingress）必须先通过**。在此之前，`V2` 只能在「人工确认全部客户端已停止使用」的维护窗口内做演练，不得作为自动切换的依据。
- `V0 只读盘点` 的输出必须包含 `QwenServiceManifest` 的 hash、LaunchDaemon plist 事实、`:8000` 的实际实现与支持的 API，以及**当前已知依赖方清单**（哪些项目/服务直连）。
- 新增 `V2.5 恢复后健康 probe`：`/v1/models` + 一条固定 prompt 的流式补全 + 首 token 延迟记录；probe 失败即视为恢复失败并触发最高优先级告警。

### 10.4 与 Toonflow 侧的对接边界

Toonflow（Mini 侧）只做**租约客户端**：申请 `studio-media` 全局资源锁（带 fencing token）、提交带租约与超时的单作业、接收状态与指标、在异常时把媒体任务退回持久队列。**所有进程生命周期控制必须在 Studio 本机受批准的 supervisor 内完成**，不得从 Toonflow worker 发起 shell kill 或直接调用未验证的 unload 端点。这与 §2.2 的两种模式一致，本节只是把「谁有权控制进程」写成硬边界。

`RECONCILED`
`AUTO_MODEL_UNLOAD_FORBIDDEN`

## 11. 参考资料

- [MLX Local AI / mlx_lm.server](https://docs.mlx.app/local/overview) 与 [官方 SERVER.md](https://github.com/ml-explore/mlx-lm/blob/main/mlx_lm/SERVER.md)：一命令一模型、已记录 API 范围和 draft unload 语义。
- [mlx-serve API 文档](https://github.com/raspoli/mlx-serve)：第三方实现的 load/unload 候选，不证明本机正在使用它。
- [Apple Activity Monitor memory pressure](https://support.apple.com/en-euro/guide/activity-monitor/-actmntr1004/mac)：内存压力/交换等指标解释。
- [本地媒体可行性](./LOCAL_AI_MEDIA_FEASIBILITY.md) 与 [连续短剧架构](./AI_SERIES_PRODUCTION_ARCHITECTURE.md)：媒体和 Toonflow 领域边界。
- [架构对账报告](./TOONFLOW_ARCHITECTURE_RECONCILIATION.md) §8：本节可观测性结论的完整证据链与状态机设计。

`RESEARCH_COMPLETE`

`MODEL_SWITCHING_UNVERIFIED_ON_DEVICE`

`IMPLEMENTATION_NOT_STARTED`
