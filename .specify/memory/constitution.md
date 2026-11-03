<!--
同步影响报告 / Sync Impact Report
=================================
版本变更: (未批准模板, 无版本) -> 1.0.0
变更性质: 初始批准。依据 docs/11-Dr.BIM研发前关键详细设计基线_V1.0.md
  (V1.0, 生效 2026-10-23) 首次建立项目宪章。
修改的原则: 无旧标题可重命名, 核心原则 I 至 XII 均为首次生成
新增章节: Core Principles; 适用范围与系统层级; 实施制品与合规门禁; Governance
删除章节: 无 (仅移除模板示例注释)
遗留 TODO: 无
说明: 本报告为临时评审材料, 提交修订后的宪章前应删除本注释。
-->

# Dr.BIM Constitution

## Core Principles

### I. 唯一写入 Owner（NON-NEGOTIABLE）

每个事实（聚合）有且仅有一个 Command Owner 和一个事实事件生产者，全部登记在唯一 Owner
注册表中（完整清单以基线文档第 3 章为准）。核心映射：Account/Session→Identity；
Organization/Workspace/Membership→Organization；Contract/Subscription/PlanDefinition→Entitlement；
资源账户、客户额度账本、平台用量与计量事件→Billing/Metering；Project→MS-01；
DrawingRevision/FileSecurityResult→MS-02；ParseRun/ResultVersion→MS-03；
KnowledgeRelease/EvidenceBundle→MS-04；DecisionCase/DecisionRecord→MS-05；
ModelingPlan/ExecutionSpecification→MS-06；WorkflowRun/Task/ResourceLease→MS-07；
EdgeDevice/BridgeTask/DeviceLease→MS-08；ExecutionRun/Receipt/LocalModelExecutionRecord/
ModelRevision/ArtifactRevision→MS-09；DeliveryPackage/SignOff/LearningCandidate→MS-10；
AuditRecord→Audit Sink。

- 同一事实 MUST NOT 存在第二个写入口或第二个事实事件生产者；新增任何事实前 MUST 先在
  注册表登记。
- 读模型可以复制字段，但 MUST 标记来源、事件版本和 `data_as_of`，MUST NOT 反向写入来源聚合。
- 注册表条目的编号与含义 MUST NOT 互换；一个产品可以组合多个 MS，一个 MS 也可以服务多个
  产品，但事实归属不变。
- 理由：多写入方会破坏版本链、审计与计费事实的可信性，是全系统一致性的根基。

### II. 显式命令与事件契约

所有跨模块协作 MUST 通过显式命令和事件表达，并在命令/事件目录中登记唯一处理者、关键前置
条件和成功事实。

- 每个命令有唯一处理者；前置条件不满足时 MUST 产生显式拒绝事实（如 `UploadAdmissionRejected`），
  MUST NOT 静默忽略或隐式重试。
- 事件统一使用过去式命名，最小信封包含：eventId、eventType、schemaVersion、producer、
  aggregateType、aggregateId、aggregateVersion、organizationId、projectId、authorityCellId、
  occurredAt、correlationId、causationId、dataClassification、payloadRef/hash。
- 消息正文、大文件、图纸、模型和完整建模计划 MUST NOT 直接放入消息；MUST 使用受权限保护的
  对象引用加内容哈希。
- 事件 MUST NOT 被用于要求上游回写；同名事实事件 MUST NOT 由多个生产者共用。

### III. 状态机与不可变修订

关键聚合（DrawingRevision、ParseRun、PreviewBuild、ModelingPlan、WorkflowRun、DeliveryPackage）
的状态转换 MUST 显式建模，且只有其 Owner 能推进状态。

- 新内容 = 新修订：DrawingRevision、ParseCandidateRevision、ResultVersion、ModelingPlan 等
  MUST 以新增修订/版本演进，MUST NOT 原地覆盖；旧修订进入 `Superseded` 并保留关系。
- 解析业务状态（MS-03）与调度任务状态（MS-07）MUST 通过 ID 映射关联，MUST NOT 共享一张表。
- 失败与未知结果显式化：Edge 失联、Revit 崩溃或保存结果不确定时 MUST 进入
  `OutcomeUnknown -> Reconciling`，MUST NOT 臆测为成功或失败。
- `SecurityTrusted` 之前禁止任何业务/技术解析；`QuotaSettled + DrawingRevisionCreated`
  之前禁止 ParseRun。
- PreviewBuild 幂等键为 `resultVersionId + previewProfileVersion`；`Degraded` MUST NOT 冒充
  完整预览；PreviewArtifact 只读引用 ResultVersion，MUST NOT 作为建模计划或交付签发的
  权威输入。
- 已签发交付包不可修改：变更 MUST 生成替代包并保留关系；IFC 只有在合同、Exchange Profile、
  权限和签发满足时进入正式包。
- 理由：显式状态机与守卫让每个禁令可被自动化测试验证，替代对运行时行为的猜测。

### IV. G1 人工确认与不可变 ResultVersion（NON-NEGOTIABLE）

正式结果 MUST 经过 G1 人工确认门禁：红色阻断项与黄色注意项均清零后才能提交确认。

- `ConfirmationCard` MUST 引用候选对象、二维证据位置、来源通道、置信度、进入原因和影响
  范围，MUST NOT 复制脱离来源的数据。
- 确认、修改、丢弃、人工补录逐项留痕（操作者、时间、原值、新值、原因码、版本）；批量操作
  逐项留痕并生成汇总事件；丢弃 MUST 携带原因码。
- MS-05 拥有 G1 决定与 `ConfirmationSetSnapshot`；MS-03 MUST NOT 修改人工决定，只消费匹配
  的确认集发布 ResultVersion。颜色和阈值来自 `ConfirmationPolicyVersion`，MUST NOT 写死。
- `ConfirmationSetSnapshot` 是过闸时全量构件判定值的不可变快照（含绿色自动接受项与全部
  人工决定），MUST NOT 退化为"问题项列表"。
- `ResultVersion` MUST 绑定完整版本链：drawingRevisionId、parseRunId、candidateRevisionId、
  confirmationSetSnapshotId、productLineCode、schemaVersion、unitSystem、coordinateSystem、
  parserVersion、modelVersion、ruleVersion、knowledgeReleaseId、contentHash。
- `Completed` MUST 在同一事务中生成不可变 ResultVersion 与 `ResultVersionPublished` Outbox
  事件；任一步失败保持可重试的 `PublishingResult` 或进入对账，MUST NOT 显示完成但无版本。
- G1 提交使用 `candidateRevisionId + decisionCaseVersion + idempotencyKey` 并发控制；冲突
  返回明确版本差异，MUST NOT 后写覆盖；迟到消息按
  `runId + attempt + seq + aggregateVersion` 判定，MUST NOT 覆盖新尝试。

### V. 客户端能力与本地建模边界

- Web 与 Mobile 共享同一云端 API、权限、项目状态和业务能力；两者均无本地 BIM 执行能力，
  MUST NOT 连接 Edge、MCP 或 BIM 进程，MUST NOT 创建、下发或执行本地建模任务。
- 正式建模任务只能在 Windows Desktop 完成用户、项目、ResultVersion、设备、软件版本、模板、
  插件和本地保存位置校验后创建；Web 和 Mobile MUST NOT 调用计划创建、冻结或启动接口。
- 两种入口（`Web/Mobile 确认结果 -> Desktop 打开同一项目继续`；`Desktop 登录/建项/上传 ->
  云端安全检查与解析 -> 预览调整确认 -> 直接继续本地建模`）MUST 在 `ResultVersion` 处汇合；
  MUST NOT 复制项目、重复上传权威文件或生成平行版本链。
- SaaS 模式下上传与解析仍由 Hosting Cell 处理，仅 BIM 原生建模在用户本机执行；Customer
  Local 模式可由 Customer Local Cell 及本地 Worker 处理前半段。
- `drbim://` 仅用于唤起客户端并定位项目，MUST NOT 承载完整任务载荷。
- 静态计划校验 MUST NOT 依赖某一台当前在线设备；设备、许可证、插件、磁盘和目录权限在
  启动前预检。输入 ResultVersion、软件兼容范围、模板/族库 Release 或计划规则变化时生成
  新计划版本，MUST NOT 原位修改 Frozen 计划。
- 本地绝对保存路径、Windows 用户名和本机敏感目录只保留在 Desktop/Edge 本地，MUST NOT
  写入云端计划；`LocalModelExecutionRecord` 记录不透明本地模型标识、模型指纹、版本与
  `UniqueId` 映射，MUST NOT 保存绝对本地路径或 RVT 正文。
- `ExecutionSucceeded` 仅证明本地执行经回执与对账确认；`ArtifactAvailable` 才证明平台持有
  用户授权上传的文件。未上传 RVT 时，平台可以展示任务进度、完成状态和回执摘要，但
  MUST NOT 声称已存储、完整检查或可以在线预览最新 RVT。

### VI. 模块依赖与调用规则

模块依赖方向固定：

```text
Identity/Organization/Entitlement/Billing
                 |
                 v
MS-01 -> MS-02 -> MS-03 -> MS-05 -> MS-06 -> MS-07 -> MS-08 -> MS-09 -> MS-10
                    |        ^
                    v        |
                  MS-04 -----+
```

- 允许：同进程短查询、命令验证与事务外编排；跨部署单元同步短调用与能力查询；单一 Owner
  的 Command 消息；事实发布、读模型与后续流程的 Event 消息；大文件/计划/结果/证据的对象
  引用。
- 禁止：调用对方 Repository 或共享可变实体；长任务等待、文件正文传输与循环调用；广播给
  多个竞争业务 Owner；用事件要求上游回写；永久公开 URL 或无哈希引用。
- CI MUST 阻断反向项目引用、跨模块 Repository 引用和循环依赖。
- 理由：依赖规则是少数能被机器强制执行的架构约束，必须进 CI 而非仅靠评审。

### VII. 部署身份与安全基线

每个部署单元 MUST 持有唯一身份：Web/BFF 工作负载身份、Core API 工作负载身份、Worker 身份、
Knowledge 身份、Bridge 身份、Desktop/Edge 设备证书加用户委托、MCP Server 本机服务身份加
一次性任务令牌、BIM 插件签名插件身份加宿主进程上下文。

- Web/BFF MUST NOT 读取业务数据库；MCP Server 仅调用白名单工具且 MUST NOT 持有长期用户
  Token；BIM 插件仅在目标软件事务内执行有限命令；Desktop/Edge 仅接收匹配设备、项目、租约
  的任务。
- 同一 Core API 内的 MS 模块间不要求 mTLS；跨进程、跨主机、跨 Cell 和设备通信 MUST 使用
  TLS/mTLS，并结合用户/任务委托、ABAC、签名、nonce、时间戳和幂等键。
- 首版威胁模型清单（Web/BFF、文件入口、PageAgent、AWS AEC、Edge/MCP/插件、更新链、
  Customer Local）的边界与首版控制 MUST 落地并有测试引用；新增边界 MUST 先补充威胁条目
  再实现。
- 制品更新链 MUST 包含 SBOM、哈希、签名、Staging、灰度和回滚；Customer Local MUST 具备
  签名 Release、有效期、离线能力清单和漂移检测。

### VIII. 写模型幂等、租约与对账

每条写模型命令 MUST 携带完整控制字段集：workflowRunId、bridgeTaskId、attemptId、commandId、
idempotencyKey、leaseId、fencingToken、sequenceNumber。

- 相同 `idempotencyKey` 已完成时，插件或 Edge MUST 返回既有回执，MUST NOT 重复创建或修改
  构件；`attemptId` 仅标识技术执行尝试，MUST NOT 替代业务幂等键。
- 过期 `fencingToken` 回执只归档，MUST NOT 推进状态；`DispatchBridgeTask` MUST 校验设备
  能力、DeviceLease 和 fencingToken。
- 排队状态可立即取消；已派发未执行由 Edge 确认取消；运行中进入
  `CancelRequested -> Cancelling`，在事务批次安全点停止。当前未提交事务可以回滚；已提交
  事务批次保留并在回执中列出，MUST NOT 承诺整项任务自动回滚，后续清理由补偿计划或人工
  处理完成。
- Edge 失联、Revit 崩溃或保存结果不确定时进入 `OutcomeUnknown`；同一写任务 MUST NOT 直接
  改派到另一设备，必须等待 Edge 本地账本、插件回查和模型指纹对账后再决定结果；对账前
  不能换节点重派同一写任务。
- 理由：分布式写模型的重复构件与丢失写入只能靠幂等键、租约和对账消除，任何"尽力而为"
  都会破坏交付可信度。

### IX. AEC 知识治理

知识运行 MUST 遵循以下三条路径之一；四种检索方式共享
`knowledgeId/chunkId/releaseId/policyTag/sourceId` 标识，但 MUST NOT 要求所有知识位于同一
物理 ArcadeDB 实例。

- 在线公共/法域：知识位于 AWS AEC Core ArcadeDB；MS-04 发送最小查询上下文，返回引用和
  证据；客户项目正文 MUST NOT 作为默认上下文上传。
- 在线企业/项目私有：知识位于 Authority Cell 私有 ArcadeDB 分区/检索端点；本地过滤和召回，
  仅按策略发送最小证据参与融合；MUST NOT 先全库召回再由 LLM 过滤。
- 完全离线：签名 KnowledgeRelease 加本地私有知识；本地全文/向量/图能力按已安装包降级
  运行；MUST NOT 伪装成最新公共知识，MUST NOT 离线回传项目正文。

### X. 角色、职责与商业权益基线

- `RoleBinding` MUST NOT 以保存签发范围替代 `ProjectDutyAssignment`；ReviewerDuty 与
  SignatoryDuty MUST 绑定项目、产品、专业、阶段、范围和有效期；SignatoryDuty 还 MUST
  绑定冻结交付对象并执行增强认证。
- 服务端 MUST NOT 仅依据前端菜单、Token 中的历史角色字符串或"管理员"名称放行审核和签发；
  职责撤销、到期、账号停用、项目成员移除或职责范围变化 MUST 发布高优先级失效事件，使
  Gateway、BFF、AI 会话和待办队列立即刷新授权上下文。
- 第一阶段一般用户（ProjectOwner + Engineer）允许操作/参数确认及 Desktop 建模，MUST NOT
  伪造独立 ReviewDecision 或 DeliverySignOff。
- 实名门禁：`IdentityVerificationCase` 独立于 Account；上传、可计费任务、购买/加购、开票、
  敏感导出和签发均在服务端检查 `verified` 且未过期。
- 商业政策 `CommercialPolicyVersion/PlanDefinition` 使用
  `draft -> approved -> published -> retired`；未发布版本 MUST NOT 进入页面、订单、权限和
  测试。订阅状态使用
  `pending_activation -> active -> grace_period -> suspended -> expired/revoked`；
  `autoRenew=false` 仅表示不续订；到期可生成 `locked_readonly` 投影。
- 一般用户席位、BIM 厂商许可、Edge 设备并发三类数量分别建模和校验，MUST NOT 共用
  `seat_count`；产品权益用稳定 `product_code` 逐项授权。
- 客户额度的预留、消费、释放和冲正写 `CustomerQuotaLedgerEntry`，相同业务幂等键 MUST NOT
  重复扣减；平台用量写 `PlatformUsageRecord`，MUST NOT 自动扣客户额度。
- 计量与账务只能追加/冲正，MUST NOT 修改项目、任务和交付事实；未决商业规则（是否试用、
  单档/多档套餐、无限/固定席位标记）MUST 保持 `Pending Decision`，MUST NOT 硬编码默认答案。

### XI. 容量、恢复与制品就绪

- 容量按 Web/API、文件/解析、AI/知识、BIM 执行四个模型推导；"5千/1万/10万/100万用户"仅为
  场景标签；在获得活跃率、项目量和任务率之前 MUST NOT 承诺具体硬件；Golden 样本压测结果
  写入版本化 Capacity Profile。
- 恢复顺序固定：KMS/证书/时间源 -> PostgreSQL 与对象存储 -> RabbitMQ 与 Outbox/Inbox ->
  Core API 与 Gateway -> MS-07 调度和 MS-08 Bridge -> Edge 在线账本与执行对账 -> MS-09 执行
  回执与 LocalModelExecutionRecord 登记 -> 可选 Artifact 上传、安全检查和校核 -> MS-10 签发、
  交付和外部连接器 -> ArcadeDB/派生索引重建或恢复。
- RPO/RTO 在恢复演练前均为 `Candidate`，MUST NOT 写入客户合同；恢复后 MUST 先核对聚合
  版本、Outbox 游标、对象哈希、在途租约、ExecutionReceipt 和已签发包，再开放写流量。
- 在《实施制品与合规门禁》所列制品进入仓库并被 CI 解析之前，架构可指导开发骨架和 POC，
  但 MUST NOT 宣称全部模块达到 L4 研发就绪。

### XII. 需求准入守门

任何产品新增需求 MUST 先回答以下问题，无法回答时保持 `Under Review`，MUST NOT 直接进入
开发任务：

1. 修改哪个产品线或内部能力？
2. 影响哪个权威对象和唯一 Owner？
3. 新增 Command 还是已有 Event 的消费者？
4. 是否改变状态机、权限动作、数据区域或交付责任？
5. 是否需要新部署单元，还是仅新增模块/Adapter？
6. 是否影响现有 Revision、RunTrace、审计和迁移？

产品分析事件还 MUST 额外回答：属于哪个阶段、是否仍在 20% 事件类型上限内、为何不能直接
使用领域事实/审计/RunTrace/计费或可观测数据、唯一生产方是谁、何时关闭、历史数据如何删除
或匿名化；任一项没有答案时，事件保持 `candidate/disabled`。

## 适用范围与系统层级

本宪章约束第一阶段 Web SaaS、Core API、异步 Worker、AEC 知识运行时、Bridge、
Desktop/Edge/MCP、Revit 插件及公共平台服务。后续 Tekla、Rhino、Blender 和数字孪生 MUST
复用同一对象与契约原则，MUST NOT 复制一套平行平台。

系统层级编码如下；编号含义与用途 MUST NOT 互换。一个产品可以组合多个 MS，一个 MS 也可以
服务多个产品：

| 层级 | 编码 | 用途 | 示例 |
|---|---|---|---|
| 业务产品线 | PL | 导航、报价、合同、客户理解 | PL-04 Tekla RC、PL-05 Tekla Rebar |
| 稳定产品代码 | product_code | API、数据库、事件、权限、权益、路由和统计 | `tekla_rc`、`tekla_rebar` |
| 附加能力 | AO | 依附主产品的增值能力 | AO-01 智能识图、AO-02 Rhino/Grasshopper |
| 外部集成 | EI | 受控连接器 | Trimble Connect、其他 CDE |
| 限界上下文 | BC | 业务语言和聚合边界 | 图纸资产、建模规划 |
| 逻辑服务模块 | MS | Owner、接口和状态边界 | MS-01 至 MS-10 |
| 内部流程步骤 | ST | RunTrace 阶段 | ST-01 至 ST-15 |
| 部署单元 | DU | 实际进程、容器或设备身份 | Core API、Parse Worker、Edge |

## 实施制品与合规门禁

以下制品是宪章原则的可执行载体，当前状态均为 Planned：

| 制品 | 最小范围 |
|---|---|
| OpenAPI | 身份、组织、项目、上传、解析、确认、计划、任务、回执、交付 |
| AsyncAPI/事件 Schema | 核心命令和事件、重试、乱序、迟到策略 |
| Edge/MCP Protobuf | 配对、能力、租约、派发、进度、取消、回执、幂等、对账和可选上传授权 |
| 错误目录 YAML | code、HTTP/gRPC、retryable、messageKey、Owner、版本 |
| 数据字典与迁移 | 字段类型、唯一约束、索引、RLS、Outbox/Inbox、保留 |
| 状态矩阵 | 命令、前态、后态、守卫、Owner、事件、补偿 |
| 权限动作目录 | Action、资源、角色、ABAC 条件、增强认证 |
| 产品分析事件目录与 Schema | 当前启用白名单、完整候选池版本、20% 自动校验、Owner、权威生产方、字段分级、留存、指标映射、启停审批和删除策略 |
| 架构依赖测试 | 允许依赖 DAG、禁止 Repository 跨模块和循环依赖 |
| 威胁模型与安全测试 | 威胁边界、Owner、控制、残余风险和测试引用 |

合规门禁：

- 上述制品进入仓库并被 CI 解析之前，MUST NOT 对外宣称全部模块达到 L4 研发就绪。
- CI MUST 执行架构依赖测试；命令/事件目录、状态矩阵和权限动作目录 MUST 与核心原则 I 的
  Owner 注册表保持一致，冲突时以本宪章为准。
- 详细设计依据以 `docs/11-Dr.BIM研发前关键详细设计基线_V1.0.md`（V1.0，生效 2026-10-23）
  为基线；基线与本宪章冲突时，按 Governance 的修订流程处理。

## Governance

- 本宪章 MUST 约束所有 specify、plan、tasks 和实现工作；与其他实践冲突时以本宪章为准。
  产品统一基线定义客户可见范围、角色动作、三端体验、任务表现和商业政策；本宪章将其映射
  为唯一 Owner、状态、命令、事件、部署身份和恢复路径，两者互补。
- 修订流程：修订提案 MUST 说明动机与影响范围（哪些 Owner、状态机、契约、部署单元受影响），
  经批准后更新本文件并生成同步影响报告；MUST 附带迁移计划或明确的过渡说明。
- 版本策略遵循语义化版本：MAJOR = 原则删除或不兼容重定义；MINOR = 新增原则/章节或实质性
  扩展；PATCH = 澄清、措辞与非语义修正。
- 合规审查：所有 PR/评审 MUST 核对与宪章原则的一致性；复杂度引入 MUST 说明理由；偏离项
  MUST 记录并限期纠正；CI 依赖规则与制品解析是持续合规的机器检查手段。

**Version**: 1.0.0 | **Ratified**: 2026-11-03 | **Last Amended**: 2026-11-03
