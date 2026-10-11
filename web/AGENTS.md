# web/

Next.js ^16（App Router）+ React 19 + TypeScript strict（别名 `@/*` → src/*）+ Tailwind v4 + shadcn/radix-ui + Biome；**包管理器 npm**（package-lock.json）。静态导出（`NEXT_EXPORT=1` 时 `output:"export"`）产出 `web/out/`，由 Go `go:embed` 内嵌为单二进制。

## 脚本

`dev`（next dev，/api 反代到后端）、`build`、`build:static`（静态导出）、`check` / `check:fix`（Biome lint+format）。

## 结构与约定

- `lib/api.ts` 是**唯一 API 客户端**：`http<T>()` 封装 fetch("/api"+path)，自动附 localStorage `artex_token` 为 Bearer，401 清 token 跳 /login；端点以 `api.xxx` 命名集中导出。
- `lib/types.ts` 集中领域类型，字段 **snake_case 与后端 JSON 对齐**，调用层做 camelCase ↔ snake_case 映射；全局偏好用 zustand（stores/preferences），其余以局部 state 为主。
- Biome：2 空格、双引号、分号、行宽 120；`useSortedClasses`、`noImportCycles`、`useFilenamingConvention` 为 error 级；import 按 react → next → 包 → @别名 → 相对路径排序。
- `src/components/ui` 与 `src/components/calendar` 是 shadcn 生成件（Biome 已排除），**勿手改**。
- 无 i18n 框架，文案中文硬编码。

## 静态导出约束（重要）

- 不能用 SSR 特性：Server Actions、next/headers、middleware；proxy.ts 在导出模式下失效，登录守卫由 `(main)/layout.tsx` 客户端实现。
- 几乎全部组件标 `"use client"`；读 cookie 需判 `typeof document === "undefined"`（prerender 期）。
- 改动要进入单二进制：`npm run build:static` → 同步 `web/out/` 到 `server/webui/dist/` → 后端带 `-tags embedui` 编译（build.sh 已自动化；`ARTEX_SKIP_FRONTEND=1` 复用旧 dist）。

## 环境与 Mock

- dev 下 `/api` 经 next.config rewrites 反代到 `AUTOPENTEST_API`（默认 http://localhost:8787）；**SSE 必须直连 :8787**（Next 反代会缓冲流式响应），token 以 `?token=` 附带。
- `NEXT_PUBLIC_MOCK=1` 时 `http()` 短路进 `lib/mock/handler.ts` 的静态路由表（未命中返回安全默认值），用于无后端预览。
- Go nil slice 序列化为 null，前端统一用 `arr()` 兜底。
