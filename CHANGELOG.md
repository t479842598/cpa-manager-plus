# 更新日志 / CHANGELOG

本仓库 = **CPA（CLIProxyAPI）+ CPAMP（CPA-Manager-Plus）自研二改**的部署与维护记录。
每条按日期倒序排列；「版本」指镜像标签与补丁基线，上游基线随升级前移。

镜像命名：CPA `eceasy/cli-proxy-api:<whitelist-vX.Y.Z[-norm1][-norm2][-gpt6]>`；CPAMP `seakee/cpa-manager-plus:<whitelist-vN>`。
补丁基线、文件数与重放校验见 [`patches/README.md`](patches/README.md)，部署与回滚步骤见 [`docs/deployment.md`](docs/deployment.md)。

---

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
