# traffic/

记录型 MITM 代理：内嵌 go-mitmproxy（CA 在 `dir/_ca`），默认 127.0.0.1:8788（cmd `-proxy` flag）。agent 的 Bash/HTTP 出口流量在此全程留痕。

## 存储（本包是全仓唯一 SQLite 用户）

- 每个交换写 `dir/_index/index.sqlite`（exchanges / exchange_bodies / blob_refs + FTS5 trigram contentless 全文索引）；FTS 不可用时降级为元数据搜索。
- 正文进内容寻址 `dir/_blobs`（>256KB 溢出到 blob、预览截 8KB）；`busy_timeout` 写进 DSN；后台 reclaim 回收索引空间。
- MITM 失败的 host **fail-open 透传**（不拦截、不记录）——这是刻意行为，勿改成阻断。
- 上游代理支持热切换（http/https/socks5）；探测开关每请求实时求值。

## 其他

- `archive.go` 负责任务归档快照（manifest + blob 流式拷贝）。
- 与漏洞的证据绑定（bind_finding_traffic / traffic_refs）见 `docs/漏洞流量证据.md`；证据的不可清理持久化属于 evidence 包，本包数据可清理。
