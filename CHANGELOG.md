# 更新日志 / CHANGELOG

本仓库 = **CPA（CLIProxyAPI）+ CPAMP（CPA-Manager-Plus）自研二改**的部署与维护记录。
每条按日期倒序排列；「版本」指镜像标签与补丁基线，上游基线随升级前移。

镜像命名：CPA `eceasy/cli-proxy-api:<whitelist-vX.Y.Z[-norm1][-norm2][-gpt6]>`；CPAMP `seakee/cpa-manager-plus:<whitelist-vN>`。
补丁基线、文件数与重放校验见 [`patches/README.md`](patches/README.md)，部署与回滚步骤见 [`docs/deployment.md`](docs/deployment.md)。

---

## 2026-09-26 — 升级 CPA v7.3.18 + CPAMP v1.14.1，修复「思考强度 max 被夹成 high」（正式标签 `whitelist-v7.3.18-norm2-gpt6` / `whitelist-v4`，已部署生产）

### 修复

- **思考强度透传（本轮触发问题）**：pi-web 侧选「max」经 CPA 到上游后变成 `high`。根因：CPA 对未显式声明 `thinking:` 的 `openai-compatibility` 模型注入写死的默认档位 `[low,medium,high]`，客户端发来的 `xhigh`/`max` 被 `clampLevel` 夹到 `high`（两处硬编码：`buildOpenAICompatibilityConfigModels`、`compileOpenAICompatibleModelCapabilities`）。
  - **上游 v7.3.18 未修复**（`config.example.yaml` 把它写成预期行为，只提供逐模型显式声明 `thinking.levels` 的迂回方式），因此修进本补丁：默认档位扩为 `[low,medium,high,xhigh,max]`，模型显式声明仍优先。
  - 生产实测（usage 库）：`buddy/deepseek-v4.1-flash` 发 `max` → 记录 `max`；发 `xhigh` → 记录 `xhigh`；修复前这两个值均记录为 `high`。
- **白名单条目删不掉（升级过程中发现的新 bug）**：上游 `SaveConfigPreserveComments` 只对写死的几个 mapping 做 prune（`oauth-excluded-models` / `oauth-model-alias` / `oauth-request-scoped-errors`），未列入的 mapping 会**保留原文件里的旧键**。因此经管理端点删除一条 `api-key-models` 只改内存、文件里那一条会留下并在下次 reload 复活。补一行 `pruneMappingToGeneratedKeys(..., "api-key-models")`，并补 2 例回归测试（去掉修复即失败，已验证）。
- **编辑已有 key 时白名单改动会被丢弃**：上游 v1.14.1 的「key 未改名」分支是纯别名流程且 early-return；另外该分支要求别名非空，没有别名的 key 存不了。面板侧改为：别名仅在变更时校验，白名单持久化前置，别名未变也照常落盘并提示。

### 新增

- **CPA `GET/PUT/PATCH/DELETE /v0/management/api-key-models`**：把 per-key 白名单纳入管理 API（对齐上游对 `api-keys` 的新架构）。PATCH 是声明式幂等语义——非空列表=设置，空列表=取消限制；每次变更走 `h.persist` 触发既有 reload 链路，即时生效。
- **CPAMP 白名单改为管理端点即时落盘**：新增 `apps/web/src/services/api/apiKeyModels.ts`，弹窗勾完模型点一次「保存」即同时落 `api-keys` 与 `api-key-models`；候选列表优先 `/v0/management/models` 全量目录，失败回退 `/v1/models` 探针并提示「列表可能不完整」。
- 回归测试：CPA 侧 `internal/config` 2 例 + `management` 10 例 + 思考档位 4 例；CPAMP 侧新增 8 例 editor 用例 + 10 例服务用例（四个 locale 键集一致性由既有 `ConfigPage.test.ts` 守卫）。

### 变更

- **CPA 基线 v7.2.159 → v7.3.18**；补丁 22 → 30 文件。白名单 / norm1 / norm2 / gpt6 四组既有改动全部保留；上游改过的 3 个 codex 文件（`codex_executor_execute.go` / `codex_executor_request.go` / `codex_executor_stream.go`）已手工合并（上游新增的 `applyCodexRoutingHint` 与我们的 `applyGPT6CodexIdentityHeaders` 共存），并顺带修掉 `handlers.go` / `openai_handlers.go` / `apikey_metadata.go` 的 gofmt 问题。
- **CPAMP 基线 v1.12.5 → v1.14.1**（跨 616 个上游提交），补丁按新架构重写：不再需要改 `VisualConfigEditor.tsx` / `ConfigPage.tsx` / `useVisualConfig.ts` / `types/visualConfig.ts`，`api-key-models` 不进入可视化值（整份保存时作为未知键原样保留），补丁面反而从 9 文件（含 4 个可视化层文件）缩到 9 文件（其中 2 个新文件）。
- 镜像：CPA `whitelist-v7.3.18-norm2-gpt6`（ID `daf220703bda`）、CPAMP `whitelist-v4`（ID `fef8b46b8956`），均在服务器原生构建。旧镜像 `whitelist-v7.2.159-norm2-gpt6` / `whitelist-v3` 保留作回滚锚点。
- 自定义配置（`/opt/cpa/cli-config.yaml`）零改动：`api-key-models`、`payload.default`、`routing.session-affinity`、`codex.disable-codex-cloaking`、`transient-error-cooldown-seconds: 2` 全部随文件保留。

### 部署与验证

- 全量备份 `/opt/cpa/backups/upgrade-v7.3.18-cpamp-v1.14.1-20260926T093448Z/`（卷含 `usage.sqlite` 522MB + `data.key` + `usage-imports/`，`integrity_check=ok`、46 表、`usage_events` 83131 行），异地副本已 scp 回 `backup-remote/` 且 5 个 sha256 与服务器逐字节一致。
- 灰度（三层：生产配置只读 canary 8318 → 配置副本沙箱 8319 → 卷副本 CPAMP canary 18318/18319）：沙箱 **23/23 过**（含 `max→max`/`xhigh→xhigh` 透传、白名单写/删/落盘/prune、热重载生效、复原）；生产只读 7/8（1 项不适用：生产 5 个 key 全部受限，无「不受限 key」可测）；CPAMP 迁移在卷副本上跑通（83k 事件派生表重建完成，无报错）。
- 切换顺序：**先 CPA 后 CPAMP**（面板的新端点依赖 CPA 侧存在）。已按灰度结果切换并向旧镜像留锚。
- 生产复测：`/v1/models` 受限 key 28 个、越权 `403 model_not_allowed`、`GET /v0/management/api-key-models` 200 / 无 key 401、公网 HTTPS 三项通过、CPA/CPAMP 日志 panic/fatal **0**。
- 白名单**写路径**在生产上做了可逆验证（追加→生效→复原）：追加后条目落盘且该模型放行，复原后文件与内存均与初始一致、越权重新 403；生产配置零残留（与写测试前逐项相等）。

- **离线维护（迁移收尾，曾被漏做后补齐）**：CPAMP v1.14.1 的迁移把 12 个查询索引与 1 个清理任务**延后到离线命令**（启动日志 `[derived-migration] deferred index preparation indexes=12 … command=cleanup-derived`）。切换当天我误把迁移日志当作收尾完成，面板随后持续显示「数据库升级维护尚未完成 · 性能降级」。已按面板指引补做：停 CPAMP → 清理前快照（`/opt/cpa/backups/pre-cleanup-derived-20260926T110050Z/`，92 MB，sha256 `3cfde22f…`）→ `docker compose run --rm --no-deps cpa-manager-plus cleanup-derived --db-path /data/usage.sqlite` → 启 CPAMP。
  实测：`creating index …` 12 个（含重建 legacy 表上的 3 个过期索引）、`Derived cleanup completed: jobs=1 processed_rows=0 prepared_indexes=12`；`/status` 的 `databaseMaintenance` 变为 `{required:false, performanceDegraded:false, deferredIndexes:0, offlineJobs:0, reasons:[]}`（横幅消失）、`integrity_check=ok`、`usage_events` 持续增长（83131 → 83390）、collector `deadLetters=0`、CPA/CPAMP 健康检查全 200。
  ⚠️ **下次升级 CPAMP 必查**：升级后 `curl -H "Authorization: Bearer <admin>" /status` 看 `databaseMaintenance.required` 是否为 `false`；为 `true` 就说明还有 deferred 索引/离线任务没做。

### 回滚

```bash
cd /opt/cpa
# CPA 单独回退
sed -i 's|whitelist-v7.3.18-norm2-gpt6|whitelist-v7.2.159-norm2-gpt6|' compose.yaml && docker compose up -d --no-deps cli-proxy-api
# CPAMP 单独回退（需连同数据卷一起回退：v1.14.1 已把连接配置迁入加密 SQLite）
sed -i 's|whitelist-v4|whitelist-v3|' compose.yaml && docker compose up -d --no-deps cpa-manager-plus
```

> 完整时间线（含备份路径、构建命令、灰度明细）见 [`docs/deployment.md`](docs/deployment.md)；补丁基线与文件清单见 [`patches/README.md`](patches/README.md)。


### 变更

- `/opt/cpa/cli-config.yaml`：`transient-error-cooldown-seconds` **15 → 2**（本轮唯一的运行时改动，镜像与补丁均未变）。
- 机制：CPA 对上游瞬时错误（408/500/502/503/504）会冷却该 provider 的凭证，**冷却窗口内的新请求不发起上游调用、直接返回 `503 auth_unavailable`**（CLIProxyAPI issue #2261 / #4787 同类现象）。2026-09-10 已从默认 60s 下调到 15s，本次实测 15s 仍然过长。
- 触发证据（2026-09-16 19:24，由 freebuff2api-go 侧报「调用模型 503」反查）：
  - `19:24:40` CPA → Cloudflare（`104.21.22.113:443`）连接被 reset：`read: connection reset by peer`；
  - `19:24:42` / `19:24:48` 两次 `503 | 75ms / 71ms`（耗时说明**未发上游请求**）；同一时段 freebuff 源站 nginx access.log **无对应 POST 记录**、服务日志零 ERROR；
  - `19:24:57` 冷却结束，恢复 200。
- 选 2s 的理由：社区实测值（2s 让 Agent 的重试节奏落在冷却窗口外）；免费 provider 的账号级轮换由下游网关（freebuff2api-go，189 账号池）自行处理，CPA 这层不需要长冷却。

### 验证

- `cd /opt/cpa && docker compose restart cli-proxy-api`：启动日志 `API server started successfully on: :8317`、`21 clients (1 Codex keys + 20 OpenAI-compat)`，无 panic / fatal。
- 公网复测：`https://api.274747.xyz/v1/chat/completions` 调 `FB/deepseek-v4-flash` → **200 / 7.06s**，正文 `COOL2OK`。
- 改动为一行数值 + 注释，其余 provider / 白名单 / headers 配置零变化。

### 备注（自定义资产，必须随配置一起迁移）

- 该值是**宿主机挂载文件**里的自定义值（`/opt/cpa/cli-config.yaml` → 容器 `/CLIProxyAPI/config.yaml`），**不在镜像内**。重建 `/opt/cpa`、换机器、或从旧备份还原配置时**必须重新应用本值**，否则会退回上游默认行为（默认值不等于 2s）。
- 改动前备份：`/opt/cpa/cli-config.yaml.bak.20260916-113442`。
- 回滚：`cp /opt/cpa/cli-config.yaml.bak.20260916-113442 /opt/cpa/cli-config.yaml && cd /opt/cpa && docker compose restart cli-proxy-api`。

## 2026-09-14 — GPT-6 家族自动 Codex 身份头（正式标签 `whitelist-v7.2.159-norm2-gpt6`，已部署生产）

### 新增

- **识别到 GPT-6 家族模型即自动使用 Codex CLI 身份头**（`Originator: codex_exec` + `codex_exec` UA，缺失时补 `Session_id`），无需再给每个 provider 手写 `headers`、也无需关掉 `codex.disable-codex-cloaking`。
  - 目的：anyrouter 这类转发站按 `Originator` 分流，`codex_exec` 身份可用、`codex-tui`（CPA 默认 cloaking 值）会报 `400 invalid codex request` / `500 get_channel_failed`。此前靠手工配置解决，**别人装我这份二改 CPA 时极易漏配**，故做进代码。
  - 命中范围：`gpt-6`、`gpt-6-astra`、`gpt-6.0` 等（含 `teamA/gpt-6-astra` 前缀名与 `gpt-6-astra(high)` 思考后缀）。
  - 生效边界：只对**非官方上游**生效（`base-url` 指向 `chatgpt.com` / `openai.com` 或留空走默认官方后端时不干预，官方账号照旧 `codex-tui`）；provider `headers:` 里显式写过的 `Originator` / `User-Agent` 优先；`models.json` 的 `config.override_header` 也优先。
  - 客户端自带 `Originator`（如 ZCode 的 `codex_cli_rs`）在命中时会被改写，不会漏到上游。
- 新增回归测试 `gpt6_codex_identity_test.go` 8 例（家族识别 / 前缀与后缀名 / 官方后端不干预 / 覆盖 cloaking / 丢弃客户端 `Originator` / 让位于 provider `headers` / 让位于 `config.override_header` / 空输入不 panic）。

### 变更

- `patches/cpa-whitelist.patch` 由 18 文件 → **22 文件**（新增 3 个改动文件 + 1 个测试文件）。
- 未改任何现有行为：非 GPT-6 模型、官方后端、已显式配置 headers 的场景一律零改动。

### 验证

- 干净 v7.2.159 worktree 重放补丁：`git apply --check` plain 通过、`go build ./...` 退出码 0、`go test ./internal/runtime/executor/ -run GPT6` 8 例全 PASS、`go vet` 无输出。
- 已知失败测试 `TestOpenAICompatExecutorToolResultContentByInputModalities`（上游自带、4 子用例）已用**零改动干净 worktree** 对照复现，确认非本次引入。
- **抓包 A/B（同一份「无 headers + cloaking 开启」配置）**：旧镜像发 `Originator: codex-tui`，新镜像发 `Originator: codex_exec`；同容器内非 GPT-6 的 `gpt-5.4-codex` 仍发 `codex-tui`（反向对照）。

### 部署（2026-09-14 生产已切换）

- 全量备份 `upgrade-v7.2.159-norm2-gpt6-20260914T031936Z`（含 CPAMP 卷 39 MB + 配置 + secrets + data + MANIFEST/sha256），异地副本三方 sha256 与服务器逐字节一致；`integrity_check`/`quick_check` 均 ok。
- 服务器 arm64 构建 `eceasy/cli-proxy-api:whitelist-v7.2.159-norm2-gpt6`（`Commit: gpt6-20260914`）。
- 8318 灰度与切换前逐项对照：四个客户端 key 模型数 6/4/8/5 不变、白名单外 403、management 200/401、流式 `[DONE]`+`finish_reason`、`panic`/`fatal` 0。
- 切换后复测：`healthz` 200、CPAMP `/health` 本地与公网 200、公网 `天机阁/gpt-5.6-sol` 200；旧镜像保留作回滚锚点。
- 详细记录见 [`docs/deployment.md`](docs/deployment.md) 的 2026-09-14 条目。

### 已知限制

- anyrouter 的 `gpt-6-astra` 当天持续 `500 get_channel_failed`（上游容量），切换前后一致；连续失败后 CPA 会短暂 `503 auth_unavailable`（凭证冷却），约 1 分钟后自动恢复。
- 裸 `gpt-6-astra` 目前仅对 `api-keys` 中的 `tang1234` 放行（其白名单含该项），四个 `sk-*` 客户端 key 未包含，故 403 —— 配置现状，非本次改动引入。
- WebSocket 传输（`websockets: true`）未接本规则。

---

## 2026-09-13 — anyrouter / gpt-6-astra 改用 Codex 协议接入（配置修复，零代码改动）

### 修复

- **`gpt-6-astra` 长期 404**：根因是配置段选错 —— anyrouter 被放在 `openai-compatibility` 段，而该段上游路径在 executor 里写死 `/chat/completions`，anyrouter 对 `gpt-6-astra` 只提供 Codex（`/responses`）通道。改放 **`codex-api-key`** 段即通。
- **ZCode 报 `invalid codex request`（400）**：anyrouter 按 `Originator` 分流，`codex-tui` 通道不可用。修复需**同时**做两件事：`codex.disable-codex-cloaking: true`（否则 `applyCodexCloakingHeaders` 会无条件把 `Originator` 覆写回 `codex-tui`）+ provider `headers` 指定 `Originator: codex_exec` 与配套 UA。
  - 教训：只加 `headers` 不看实际报文会误判「已修好」——必须抓 CPA 实际发出的请求（捕获服务需加入 `cpa_cpa-net` 网络，容器内 `127.0.0.1` 与宿主 IP 均不可达）。

### 变更

- `cli-config.yaml`：anyrouter 从 `openai-compatibility` 迁到 `codex-api-key`，新增 `codex.disable-codex-cloaking: true` 与 `headers`；其余 21 个 provider 未动。改前备份 `cli-config.yaml.bak-pre-codex-anyrouter-20260913-131233`。
- 本机新增 Codex CLI 直连持久配置 `~/.codex-anyrouter/`（`CODEX_HOME` 隔离，令牌 600 权限文件，`.zshrc` 里 `codex-anyrouter` 函数走代理 `127.0.0.1:7897`）。

### 验证（生产全过）

- `/v1/chat/completions` 非流式 **200**（3.0s）；流式 **200**（18 chunks，含 `[DONE]` + `finish_reason`）；`/v1/responses` **200**；公网 `https://api.274747.xyz` **200**。
- 模拟 ZCode 自带 `Originator` / UA 的请求 **200**（配置优先于客户端头）。

### 已知限制

- 上游偶发 `当前模型 gpt-6-astra 负载已经达到上限`（`code: get_channel_failed`，HTTP 500，provider 显示 `codex`）—— anyrouter 侧容量限制，重试即可。
- 传闻中的「备用端点 `pmpjfbhq.cn-nb1.rainapp.top` 直连可用」经本机 + 服务器双侧实测**全部 404**（无任何 API 路由），证伪。

---

## 2026-09-13 — CPA 升级 v7.2.157 → v7.2.159 + 补丁基线迁移 + 凭证脱敏

### 变更

- CPA 基线前移到 **v7.2.159**（补丁内容不变，仅上下文对齐新 tag），上游 v7.2.157→v7.2.159 共 152 个改动文件中**仅 2 个**与本补丁重叠（`internal/api/server_management.go` 插件配额路由、`sdk/api/handlers/openai/openai_handlers.go` 改用 `h.WriteModelListResponse`），均可自动合并。
- 完整备份（配置 + secrets + data + CPAMP 卷 + MANIFEST/sha256）并回传异地副本 `backup-remote/upgrade-v7.2.159-20260913T111213Z/`。
- 仓库内清理明文凭证（README/docs 不再记录密钥值，只留路径与操作步骤）。

### 修复

- **`payload:` 段 YAML 写错 → cli-proxy-api 崩溃循环、全站 502**：恢复正确缩进后重启，并补充「配置改动前先 `docker run --rm -v ... image -t` 做 YAML 语法层校验」的门禁。

---

## 2026-09-11 — CPA 升级 v7.2.145 → v7.2.157 + 修复 ZCode Responses 流顺序报错

### 修复

- **ZCode `Model request failed` / `text part msg_…_0 not found`**：根因在 CPA 的 Responses 转换器 `openai_openai-responses_response.go` —— 判断工具调用用 `delta.Get("tool_calls").IsArray()`，**空数组 `[]` 也满足**，于是第一个正文 delta 之后就误发 `response.output_item.done`，客户端删掉文本 part 再收同 id delta 即抛错。上游 **v7.2.146**（`6c6473f8`）改为 `… && len(tcs.Array()) > 0`，本次升级生效：同一会长输出请求升级前 **425** 处顺序违规 → 升级后 **0** 处。
  - 该修复属上游代码，不在本补丁内。

### 变更

- 补丁基线随升级从 v7.2.157 前移；**norm2 的流式收尾修复与它的测试此前从未重新导出补丁**（补丁长期停在 16 文件、静默缺 norm2），本次补齐为 18 文件。旧补丁存为 `cpa-whitelist.patch.bak-16file-pre-upgrade-v72157` 留作教训对照。

---

## 2026-08-31 — 故障修复：改 CPA 管理密钥后面板「登录即闪退」→ 全站 403

### 修复

- 根因是**两把钥匙 + 启动时只读一次 + 防爆破封禁**：面板登录框用 CPAMP admin key（SQLite），与 CPA Management Key 是两套凭证；CPA 侧改 `remote-management.secret-key` 后热加载生效，而 CPAMP 只在容器启动时读一次密钥文件 → 面板持旧密钥请求 CPA 得 401 → 前端全局登出闪回登录页 → 采集器重试 5 次触发 CPA 的「同 IP 失败 5 次封 30 分钟」，此后连正确密钥也一律 403。
- 处理：`printf` 原地截断写密钥文件（保 inode）→ 重启 CPAMP（重读密钥）→ 重启 CPA（清内存态封禁）。
- 同日用户确认属误操作，走面板配置接口 + 同步密钥文件回滚原值。
- 取证要点：现网 bcrypt hash 可用 `bcrypt.checkpw` 与候选明文比对定位，遗忘的密钥可以从 hash 反查找回（仓库不记录具体值）。

---

## 2026-08-30 — OpenAI 流式收尾修复（norm2）

### 修复

- 部分 OpenAI 兼容上游**推完正文与 tool_calls 参数后直接以 `[DONE]` 或干净 EOF 收尾，从不下发带 `finish_reason` 的终止 chunk**，而 CPA 的 OpenAI→OpenAI 响应转换是纯透传 → pi-ai 抛 `Stream ended without finish_reason`（近 7 天 53 次），CPA 侧因已正常记账而显示成功，形成「上游成功、客户端失败」的错位。
- 修复：流式循环跟踪上游是否发过非空 `finish_reason` / 是否推过 `tool_calls` delta / 是否有过 choice 内容；流干净结束（`[DONE]` 或 EOF 补发 `[DONE]`）且「有内容但缺收尾帧」时，在 `[DONE]` 之前合成标准 `chat.completion.chunk`（`finish_reason` 取 `tool_calls`/`stop`，复用上游 id/model）再交既有翻译器。**空流不伪造终止事件；上游已带收尾帧时零改动。**
- 回归测试 5 例：缺收尾帧补 stop / tool_calls 补 tool_calls / 已有收尾帧不重复 / EOF 无 DONE 也补 / 空流不补。

---

## 2026-08-29 — 消息归一化修复（norm1）+ 白名单完整版 + 升级 CPA v7.2.145

### 修复

- **b.ai 等严格上游 400**（`400001 unknown variant developer`、`reasoning_content must be passed back`）：CPA 对 OpenAI→OpenAI 请求原样透传，Codex/Claude 风格的 `developer` role、assistant `thinking` content 块、缺 `reasoning_content` 的 assistant 消息会被上游拒收。新增 `normalizeOpenAIChatMessages()`：`developer`→`system`、thinking 块抽成顶层 `reasoning_content`、每条 assistant 补空 `reasoning_content`；纯字符串消息零拷贝原样返回。
- **残缺备份（严重隐患）**：`cpa-whitelist.patch` 当初用裸 `git diff` 导出，漏掉 `sdk/api/handlers/handlers_routing.go` 里的 `enforceModelWhitelist` 方法定义，按 README 流程重建会 `go build` 直接失败。已从服务器取回完整源码归档并重新导出（14 文件版）。旧残缺补丁存为 `cpa-whitelist.patch.bak-incomplete`。
- 教训已写进 `patches/README.md`：**必须先 `git add -A`、禁止裸 `git diff`、生成后必须做文件数 + 关键字 + 真实回放编译三重校验**。

### 新增

- **白名单完整版**：CPA 新增 `GET /v0/management/models`（management key 认证）返回**全量**模型目录，不受白名单影响；CPAMP 模型候选改走该端点（旧实现用 `api-keys[0]` 探 `/v1/models`，首个 key 受限时面板看不到其它模型），失败时回退旧逻辑并提示。
- CPAMP 弹窗「保存」**一步落盘**：加 key + 勾模型只需点一次，直接写 `config.yaml` 并生效；删除 key 同样一步落盘并带危险操作确认。
- 回归测试：CPA 10 例（403 拦截 / `/v1/models` 过滤 / 全量端点）、CPAMP 5 例（YAML 往返 / 一步写入 / 清空删除 / 畸形行）。

### 变更

- 升级 CPA **v7.2.143 → v7.2.145**。对本部署的有效收益是 `7bc16ee3`：多候选轮询在凭证进冷却/被摘除后 ready-view 游标位置错乱 —— 生产有 12 个模型名跨 provider 重名（`glm-5.3-flash`×3、`deepseek-v4-flash-0731`×3 等）且 `futureppo`/`nvidia` 频繁 429/500 触发冷却，正好命中该路径。
- CPAMP 基线仍 v1.12.5，只换补丁：`whitelist-v2` → `whitelist-v3`，每次切换前都做整套卷备份。

---

## 2026-08-27 — 自研功能：per-key 模型白名单

### 新增

- 上游 CPA/CPAMP 原生不支持「某个 API key 只能访问部分模型」，改源码实现完整链路：
  - **CPA 后端**：config 新增 `api-key-models` 映射（`{key: [模型列表]}`），认证时把白名单挂到请求 context，模型路由前检查，不在白名单返回 **403 `model_not_allowed`**。
  - **`/v1/models` 同步过滤**：受限 key 只能看到自己白名单内的模型，「看得见」与「调得动」严格一致。
  - **CPAMP 前端**：API Key 编辑弹窗新增模型白名单多选（搜索框 + 全选）。
- 补丁与源码归档入本仓库（`patches/`、`src/`），`src/backup/` 保存生产镜像的原始完整源码作历史锚点。

### 修复

- 保存白名单后受限 key 被踢下线、面板版本号显示不正确等问题。

---

## 2026-08-26 — 首次部署

- 服务器（Oracle `193.123.167.208`，Ubuntu 24.04 ARM64 / 2H12G）安装 Docker + acme.sh（每日 4 次自动续签）→ Compose 部署 CPA（8317）与 CPAMP（18317）→ nginx 反代 `api.274747.xyz`（含 SSE 流式优化、`client_max_body_size 100m`）→ 签发 Let's Encrypt 证书 → 配置管理员密钥。
- 当日排障：config.yaml 格式错误导致 502 / connection refused；`:ro` 挂载导致配置只读；YAML 解析失败；nginx 413。
