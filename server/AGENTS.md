# server/

后端 API 层与运行时编排：`Server`（server.go）持有 Manager 与 Engine，装配 JWT、LLM provider、scheduler/notifier 等并注册全部路由；`Manager`（manager.go）托管任务生命周期与进程级资源（PG 连接、AssetStore、流量代理、拦截器）；`Engine`（engine.go）驱动事件式探索循环。上层入口见 `cmd/artex/main.go`（退出码 0/75 协议）。

## 关键概念

- **Task**：一次渗透任务，独享 ExplorationStore（探索图），共享全局 AssetStore（资产图）。
- **SSE**：Broadcaster 每任务内存 pub/sub，落库与推送同点发布；断线用 Last-Event-ID/since 游标补洞；订阅缓冲满则丢事件，故必须**先 Subscribe 再回放历史**。
- **拦截审批**：agent 工具调用按规则进 pending 队列等 `/api/intercept/pending/{id}/decide`；未命中规则可交 LLM 兜底裁判（wireInterceptReviewer）。

## 代码约定

- 路由用 Go 1.22 ServeMux（`"METHOD /api/xxx/{id}"`），集中注册于 `Handler()`；`/api/*` 统一包 CORS + 鉴权中间件，`/api/auth/*` 与 `/api/health` 豁免。
- 鉴权：JWT HS256，key 位于 `BaseDir/jwt.key` 自动生成；token 取自 Bearer / cookie `artex_token` / `?token=`（SSE 用）；单用户模型，bcrypt 密码存 PG settings。
- DTO：`dto.go` 专职 DB → 前端形状映射，契约另一端是 `web/src/lib/types.ts`；请求体用 handler 内联匿名 struct + `decode()`。
- 错误：`writeJSON` / `writeErr` 返回 `{"error": 中文消息}`；**读库失败不得当“未设置”处理**（鉴权相关一律 503）。
- DB 访问全部经 db 包（`m.pg` / `t.Store` / `m.Assets()`），server 层**无裸 SQL**。
- webui 嵌入：`webui_embed.go` 带 `//go:build embedui` + `//go:embed all:webui/dist`（`all:` 必须保留，`_next` 是下划线目录）；`webui_stub.go` 是无 tag 时的 404 回退。

## 检查清单

- Task 可变字段（Status/Paused/Queued 等）发布后必须经 lifecycleSnapshot / updateLifecycle 更新，禁止直读直写。
- seed / 初始化路径中不得 Notify（会绕过 engine.Run 门控）。
- 删除或控制任务必须走 beginTaskOperation / decInflight 屏障。
- 改 LLM profile 后必须 invalidateProfileAgents 清 provider/ChatAgent 缓存（每 profile 单例 provider 即单限流器）。
- 测试走 httptest + 真实 `Handler()`；`testmain_test.go` 的 TestMain 抢 PG advisory lock 7337741002，使本包与 db/agent/evidence 在共享库上串行；PG 不可用则 Skipf，不允许因环境缺失而 Fail。
