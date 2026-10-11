# notify/

漏洞发现的 IM / 邮件推送适配层，**叶子包：实现仅依赖标准库，禁止引入 db/server 等上层依赖**。

## 约定

- 六渠道：钉钉、飞书（含 Lark）、企业微信、通用 Webhook、Telegram Bot、SMTP 邮件；channel.go 用**显式字面量注册表**（非 init 自注册）——新增渠道漏注册会在编译期暴露，保持该模式。
- `Channel` 接口实现必须**无状态**：凭据只从 cfg（`map[string]any`，对应 `notification_channels.config` JSONB）传入，不落结构体字段。
- `Send` 返回实际送达条数（防截断丢件）；`PermanentError` 标记不可重试错误。
- 文案长度是**字节**不是字符：企微 4096 字节硬上限，截断必须走 render.go 的 UTF-8 边界截断。
- mask.go 的凭据掩码回显分离 SecretKeys / DestinationKeys，防凭据外泄，勿合并。
- event.go 的 events.snapshot 契约冗余存事发当时的漏洞字段（推送不回查库），新增字段注意同步该快照。
