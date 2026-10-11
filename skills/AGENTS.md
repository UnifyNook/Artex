# skills/

渗透 agent 的技能包目录（不是 Go 代码）。每个技能一个子目录，结构约定：

```
<skill-name>/
├── SKILL.md        # 必需，YAML frontmatter：name / description（必需）；allowed-tools 可选
├── reference.md    # 可选，详细参考
└── scripts/        # 可选，可执行脚本
```

frontmatter 遵循 agentskills.io 格式；server 侧会解析并允许 UI 安全改写部分字段（server/server_mgmt.go 的 rewriteSkillFrontmatter），新技能缺 name/description 会导致加载异常。

## 现有技能

- `api-recon/`：前端 API 侦察（明确禁止漏洞利用与真实登录）；scripts 含 Python 分析脚本与 Electron 系 runtime_harvest（无凭据 stub/mock，只录 outbound）。
- `scopesentry/`：ScopeSentry MCP 平台管理指南。
- `playwright-cli/`：浏览器自动化用法，references/ 下九篇细分文档。

## 部署与修改

- 路径解析：`ARTEX_SKILL_DIR` > config `skill_dir` > `BaseDir/skills`；Dockerfile COPY 进镜像，compose 又以 `./skills` 绑定挂载覆盖 `/app/skills`——**改技能内容不需要重建镜像**。
- 技能可经 UI 上传 zip（server/skill_zip.go 校验并落盘），上传的技能与内置技能同源管理。
