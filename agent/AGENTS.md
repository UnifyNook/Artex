# agent/

在 Norma agentcore SDK 之上装配渗透多 agent 编排。**边界**：会话循环、compaction、预算收尾（Settlement）、transcript、todo 全由 Norma 提供，本包只做装配与领域工具；import 限于 Norma 的 agentcore / llm / permission / tool / transcript / harness / compaction。

## 角色

- **MainAgent**：人在环路接口（add_hint / add_intent / steer_work / set_goals / set_constraints）。
- **Planner**：唯一意图生成者，事件驱动唤醒（每次唤醒全新会话），读态势 → prove_goal 判目标 → 一次 add_intent 批量（≤4）。
- **Worker**：领取一条意图、真实工具执行、写回两图后即停，多 goroutine 并发。
- **ChatAgent**：通用会话；retester 的漏洞复测也走它。

## 双图工具集

`ToolSet`（ts=探索图 store，as=资产图 store）把图暴露为 LLM 工具：图态势类（graph_overview / list_findings / node_detail…）、意图与证明类（add_intent / prove_goal / record_fact / report_finding…）、资产写入类在 tools_insert.go（insert_assets / add_task_scope / list_untested_assets…）、跨 worker 过程检索（search_all_worker_traces / get_worker_trace）。按角色组装 PlannerTools / WorkerTools / MainAgentTools（worker 刻意无 list_facts 与 MultiEdit/Glob/Grep）。

## 机制与约定

- **意图领取**靠 `ExplorationStore.ClaimIntent`（db/exploration.go）的单条原子 UPDATE（open → running），失败试下一条；**不要加进程内锁**。
- **prompt**：promptcatalog.go 把内置正文（段 A）作 Go 模板种子进 `agent_prompts` 表，PromptOverride 可覆盖，渲染失败回退默认；段 B（traffic 块）/ 段 C（产物目录规约）由代码固定追加，不可被覆盖删掉。planner 每轮把触发事件 + 预取图 overview 拼进 user 消息，system 只留静态正文（利于提示缓存）；worker 把意图 + 锚定资产写 system 尾部防 compaction 压掉。
- **工具目录**经 toolcatalog 双向注册：BuiltinToolSeeds 播种 `tools` 表，DB 可覆盖描述 / schema / 默认入参并按 agent 绑定过滤；handler 永远在代码层，未绑定的工具对所有 agent 丢弃（goal_met 默认不绑）。
- ToolResolve / ToolAugment / PromptOverride / FindingRecorder 是 server 注入的包级钩子；测试中为 nil 也必须能跑（nil store 安全）。所有工具经 guardPanic（assembly.go）包裹，panic 转工具错误防进程崩。
- `task_id` 不暴露给模型，由 SetTaskID 程序权威赋值；开关类配置用 resolver func 每轮读取，免重建 agent。
- **transcript session id 规则** `exp<N>-<role>`（planner / worker-i<M> / main）被 llmrec.parseSession 依赖；goals 拆解必须显式挂 session id（网关按头路由，缺失 400）。
- capture.go 把 SDK 事件流合并为 db.Activity（delta 合并、usage、result），全 agent 共用并喂 intercept 审计。

## 测试

`testmain_test.go` 抢 PG advisory lock 7337741002 与其他包串行；无 PG 时 Skip 而非 Fail；无模型的单测用 fake `llm.Provider`（iter.Seq2 流）。
