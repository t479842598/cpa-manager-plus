# 部署与运维记录

## 服务器信息

- **主机**: Oracle 193.123.167.208（Ubuntu 24.04.4 LTS, aarch64, 2H12G）
- **SSH**: root / 55eb0737e7b442848d9f51abda729dbf
- **Docker**: 29.7.2 / Compose v5.5.0
- **Nginx**: 1.24.0（Ubuntu apt，sites-enabled 结构）

## 服务架构

```
客户端 → https://api.274747.xyz (nginx:443)
              ├── /v1/*  → CPA (127.0.0.1:8317)     ← LLM API 代理
              └── /*     → CPAMP (127.0.0.1:18317)  ← 管理面板
```

| 服务 | 容器名 | 镜像 | 端口 |
|---|---|---|---|
| CPA | cli-proxy-api | eceasy/cli-proxy-api:whitelist-v7.2.159-norm2 | 127.0.0.1:8317 |
| CPAMP | cpa-manager-plus | seakee/cpa-manager-plus:whitelist-v3 | 127.0.0.1:18317 |

> 当前版本：**CPA v7.2.159 + 白名单 + 消息归一化（norm1）+ 流式收尾修复（norm2）**（2026-09-13 由 v7.2.157 升级，含 35 个上游提交；详见下方时间线）+ **CPAMP v1.12.5-whitelist-v3**（上游基线仍 v1.12.5，**有意不升**；实测 v1.12.12 补丁有 16 个冲突块且 v1.12.6 加密迁移带 fail-closed，原因见「升级 CPAMP 到 v1.12.6」）。旧镜像 `:whitelist-v7.2.157-norm2` / `:whitelist-v7.2.145-norm2` / `:whitelist-v7.2.145-norm1` / `:whitelist-v7.2.145` / `:whitelist` / `:latest` 全部保留作回滚锚点。

部署目录：`/opt/cpa/`（`compose.yaml`、`cli-config.yaml`、`secrets/`）

## 部署时间线

### 2026-08-26 首次部署
1. 安装 Docker（官方 apt 源，Ubuntu noble ARM64）+ 启动 Docker
2. 安装 acme.sh（crontab 每日 4 次自动续签：5:33/11:33/17:33/23:33）
3. Docker Compose 部署 CPA + CPAMP
4. nginx 反代 `api.274747.xyz`（含 SSE 流式优化、client_max_body_size 100m）
5. acme.sh 签发 Let's Encrypt 证书（api.274747.xyz，2026-08-26 ~ 2026-11-24）
6. 配置 CPAMP 管理员密钥

### 2026-08-26 问题修复
- **502 / connection refused**：config.yaml 格式错误（嵌套 `server.port` → 扁平 `port: 8317`）
- **config.yaml read-only**：Docker 挂载 `:ro` 移除，改为单文件挂载 `/opt/cpa/cli-config.yaml → /CLIProxyAPI/config.yaml`
- **YAML 解析失败**：清理测试字节残留
- **nginx 413**：`client_max_body_size 100m`

### 2026-08-27 自研功能：per-key 模型白名单
CPA/CPAMP 原生不支持 per-key 模型限制，通过改源码实现：
- CPA：`api-key-models` 映射字段 + 认证 context 传递 + 路由前检查（403 model_not_allowed）
- CPAMP：API Key 编辑弹窗新增模型白名单多选勾选（搜索框 + 全选）
- 源码与 patch 备份在本仓库（`src/`、`patches/`）

### 2026-08-29 备份修复 + 白名单完整版 + 升级 CPA v7.2.145

**① 修复残缺备份（严重隐患）**：`patches/cpa-whitelist.patch` 当初用裸 `git diff` 导出，
漏掉了 `sdk/api/handlers/handlers_routing.go` 里的 `enforceModelWhitelist` 方法定义
（35 行 + 1 个 import），导致按 README 流程重建时 `go build` 直接失败
（`h.enforceModelWhitelist undefined`），备份无法重建生产镜像。完整源码当时只存在于
服务器 `/tmp/cli-proxy-api`（非 git 目录，`/tmp` 一清就没）。
- 已从服务器取回并归档整份源码：`src/backup/cpa-src-whitelist-full-v7.2.143.tar.gz`
- 重新导出补丁（`git add -A && git diff --cached`），CPA 补丁从 8 文件补全到 14 文件
- 旧残缺补丁保留为 `patches/cpa-whitelist.patch.bak-incomplete`
- `patches/README.md` 已写明「必须先 `git add -A`、禁止裸 `git diff`、生成后必须编译校验」

**② 白名单做成完整版**：
- CPA 新增 `GET /v0/management/models`（management key 认证）返回**全量**模型目录，不受白名单影响
- CPAMP 模型候选改走该端点。旧实现用 `api-keys[0]` 探 `/v1/models`，而 `/v1/models` 按 key 过滤，
  一旦第一个 key 受限，面板就只能看到它那十几个模型、无法给正在编辑的 key 授予其它模型；
  新端点失败时回退旧逻辑并提示「列表可能不完整」。**无需改 nginx、无需改 CPAMP 后端**
  （CPAMP 的 `ProxyManagement` 对 `/v0/management/*` 是通用透传，只校验路径前缀）
- CPAMP 弹窗「保存」**一步落盘**：加 key + 勾模型只需点一次，直接写 `config.yaml` 并生效，
  无需再去点配置页顶部「保存配置」
- 删除 key 同样一步落盘，并新增危险操作确认（含「删除最后一个 key 会让网关无 api-keys」的专门提示）
- 回归测试：CPA 10 例（403 拦截 / `/v1/models` 过滤 / 全量端点）、CPAMP 5 例（YAML 往返 / 一步写入 / 清空删除 / 畸形行）

**③ 升级 CPA v7.2.143 → v7.2.145**（13 commits）。对本部署的有效收益是 `7bc16ee3`：
多候选轮询在凭证进冷却/被摘除后 ready-view 游标位置错乱——生产有 12 个模型名跨 provider
重名（`glm-5.3-flash`×3、`deepseek-v4-flash-0731`×3 等），且 `futureppo`/`nvidia` 频繁 429/500
触发冷却，正好命中该路径。其余 12 个 commit 都是 claude/codex/gemini 协议层、HTTPS 代理、
OAuth auth 与 home mode，本部署无对应流量（近 72h `/v1/messages`、`/v1/responses`、`/v1beta`
均为 0 次）故无关。

执行顺序（**备份先行**）：
```bash
# 1) 全量备份：停 CPAMP 让 WAL 落盘 → 整套打包卷 → 复制配置/数据/secrets → MANIFEST + sha256
/opt/cpa/backups/upgrade-v7.2.145-20260828T185106Z/
#   含 cli-config.yaml / compose.yaml / secrets/ / data/ / cpamp-volume.tar.gz / MANIFEST.txt
#   备份可用性已验证：PRAGMA integrity_check=ok、46 张表、usage_events 6899 行、data.key 44B
#   异地副本已 scp 回本机 cpa-manager-plus/backup-remote/

# 2) 构建（源码 = 本机已验证的 src/cli-proxy-api，已含补丁）
cd /tmp/cpa145build
docker build --build-arg VERSION=v7.2.145 --build-arg COMMIT=d9cea890 \
  -t eceasy/cli-proxy-api:whitelist-v7.2.145 .

# 3) 灰度：8318 起 canary 容器，6 项验证全过才切
docker run -d --name cpa-canary -p 127.0.0.1:8318:8317 \
  -v /opt/cpa/cli-config.yaml:/CLIProxyAPI/config.yaml:ro -v /opt/cpa/data:/app/data \
  --network cpa_cpa-net eceasy/cli-proxy-api:whitelist-v7.2.145

# 4) 切换（compose 已留 compose.yaml.bak-pre-v7.2.145）
sed -i 's|image: eceasy/cli-proxy-api:whitelist$|image: eceasy/cli-proxy-api:whitelist-v7.2.145|' /opt/cpa/compose.yaml
docker compose up -d --no-deps cli-proxy-api && docker rm -f cpa-canary

# 5) CPAMP 重建（基线仍 v1.12.5，只换补丁）：v2 → v3，每次切换前都再做一次卷备份
/opt/cpa/backups/pre-cpamp-whitelist-v2-20260828T190520Z/
/opt/cpa/backups/pre-cpamp-whitelist-v3-20260828T192801Z/
```

**灰度 6 项验证结果**（全部通过）：不受限 key 模型数新旧一致 47；受限 key 一致 10；
白名单内 200；白名单外 403 `model_not_allowed`；无 management key 访问新端点 401；
带 management key 返回全量 47（含受限 key 看不到的 `FB/*`）。

**生产复测**：公网 `https://api.274747.xyz` 受限 key 只见 10 个、越权 403；
浏览器端到端验证「加载模型 → 候选 47 全量 → 勾 2 个 → 添加 → 一次点击即写入
`api-keys` 与 `api-key-models` 两处 → toast 配置已保存并生效」；删除走确认弹窗、
确认后两处同时清空、原有 key 不受影响；升级后 `usage_events` 6899 → 6901 继续写入，
CPA/CPAMP 日志无 panic/fatal。

### 2026-08-29 消息归一化修复（norm1）

**背景**：`BAI/*` 模型（b.ai，Rust serde 网关 + DeepSeek thinking 上游）对请求消息格式严格校验。客户端（Codex/Claude 风格）发出的 `developer` role、assistant `thinking` content 块、缺失 `reasoning_content` 的 assistant 消息被 CPA 原样透传，触发 400001 `unknown variant developer` 与 400 `reasoning_content must be passed back`。

**修复**：`internal/translator/openai/openai/chat-completions/openai_openai_request.go` 新增 `normalizeOpenAIChatMessages()`（developer→system、thinking 块→reasoning_content、补空 reasoning_content），已在本地编译 + translator/SDK 全量测试通过。

**部署**：Docker 构建 `eceasy/cli-proxy-api:whitelist-v7.2.145-norm1` → 8318 canary（三个失败场景 + 白名单 6 项回归全过）→ compose 换标签 `:whitelist-v7.2.145-norm1` 重启 → 公网复测通过。补丁 14→16 文件已重导出（旧版 `patches/cpa-whitelist.patch.bak-pre-norm1`）。

### 2026-08-30 OpenAI 流式收尾修复（norm2）

**背景**：pi-web 报 `Stream ended without finish_reason`（近 7 天 53 次，全在 `CPA-Free/FB/deepseek-v4-flash`，另有 CPA 各通道零星 1-2 次）。排查：失败请求正文与 tool_calls 参数**完整送达**、usage 也收到，唯独缺带 `finish_reason` 的终止 chunk；curl 按 pi 真实请求形态（带 tools / `thinking:{type:enabled}` / `reasoning_effort:max` / `stream_options`）复测 13 次全部正常，确认是上游 freebuff2api 间歇性在流末尾丢收尾帧。CPA 的 OpenAI→OpenAI 响应转换纯透传，残缺流原样转发；而 CPA 控制台因已正常记账显示"成功"，形成"上游成功、客户端失败"的错位。

**修复**：`internal/runtime/executor/openai_compat_executor.go` 流式循环新增收尾跟踪（`sawFinishReason`/`sawToolCallDelta`/`sawStreamContent` + 复用上游 id/model），在 `[DONE]`（含 EOF 补发路径）之前，若"有内容但缺收尾帧"则合成一个标准 `chat.completion.chunk`（`finish_reason` 按有无 tool_calls 取 `tool_calls`/`stop`），再交既有翻译器转各客户端格式。空流不伪造、上游已带收尾帧时零改动不重复注入。新增 `openAIStreamTerminalInfo` / `synthesizeOpenAIStreamFinish` 辅助函数。

**测试**：`openai_compat_executor_finish_test.go` 5 场景全过（缺收尾帧补 stop / tool_calls 补 tool_calls / 已有收尾帧不重复 / EOF 无 DONE 也补 / 空流不补）；本地 `go build` + executor/translator 全量测试通过。

**部署**：源码打包传服务器 `/tmp/cpa-norm2-build` → Docker 构建 `eceasy/cli-proxy-api:whitelist-v7.2.145-norm2`（COMMIT=norm2-20260830）→ 8318 canary 灰度（流式收尾一致、模型列表 57=57、非流式正常、日志零 panic/fatal）→ compose 换标签 `:whitelist-v7.2.145-norm2` 重启（备份 `compose.yaml.bak-pre-v7.2.145-norm2`）→ 公网 `https://api.274747.xyz` 带 tools+thinking 复测通过（`finish_reason:"tool_calls"` + `[DONE]`）。旧 `:whitelist-v7.2.145-norm1` 保留作回滚锚点。

### 2026-08-31 故障修复：改 CPA 管理密钥后面板「登录即闪退」→ 全站 403

**现象**：在面板的配置文件编辑器里改掉 `remote-management.secret-key` 后，用新密钥登面板登不上；用旧面板密钥能登进去但立即被弹回登录页；十几分钟后所有页面全报 403。

**根因（两把钥匙 + 启动时读一次 + 防爆破封禁）**：
- 面板登录框那个「管理密钥」是 **CPAMP admin key**（存 SQLite `settings.admin_credential_v1`），与 **CPA Management Key** 是两套独立凭证，改 CPA 密钥不会改面板密码——所以新密钥登不上、旧面板密钥能登。
- CPA 侧 `PUT /v0/management/config.yaml` 后**热加载即时生效**（日志 `remote-management.secret-key: updated`），并把明文重哈希为 bcrypt 落盘；而 CPAMP 只在**容器启动时**读一次 `/opt/cpa/secrets/cpa_management_key`（且 `ResolveSetupWithSource` 中 env/secret 文件**优先级高于 SQLite**）→ 面板拿着旧密钥请求 CPA → **401**。
- 前端 axios 拦截器对 401 派发全局 `unauthorized` → `useAuthStore.logout()` → **闪退回登录页**。
- CPA 管理接口防爆破：同 IP 连 5 次认证失败即**封 30 分钟**（`AuthenticateManagementKey`，`maxFailures=5`/`banDuration=30min`），采集器每几秒重试很快把 `172.19.0.3` 打进封禁；封禁检查在密钥校验**之前**，所以之后连正确密钥也一律 **403**。

**修复**：`printf > /opt/cpa/secrets/cpa_management_key` 写入新密钥（原地截断写，保 inode）→ `docker restart cpa-manager-plus`（重读密钥）→ `docker restart cli-proxy-api`（清内存态封禁）。CPA 侧不动，尊重用户改密钥的本意。

**验证（全过）**：直连 CPA / 经面板 / 公网三条链路均 **200**；CPA 侧 CPAMP 流量重启后 90s **123×200**（仅 1 条 403 是重启前最后一条）；不受限客户端 key 仍 57 个模型；受限 key 白名单外 **403 model_not_allowed**、白名单内 **200**；受限/不受限可见模型数 **11/57** 仍一致。

**取证要点**：现网 bcrypt hash 可用 `bcrypt.checkpw` 与候选明文逐个比对定位（Aug-27 备份的 hash 命中旧值，两次互为印证）——**忘了新密钥时可以从 hash 反向比对候选值找回**。此处不记录具体候选值。

**同类先例**：2026-08-27 曾发生「secret-key 被二次哈希 → 踢下线 + 封禁 30 分钟」，属同一类故障的两种触发方式（那次是误哈希，这次是有意改值但没同步面板）。

**后续（同日 12:43）——用户要求把密钥恢复原值**：确认本次改动属于误操作后已回滚。
- 回滚走的是用户当时改它的同一条链路：`GET /v0/management/config.yaml` → 仅替换 `secret-key` 一行为原文档明文 → `PUT /v0/management/config.yaml`（返回 `{"ok":true,"changed":["config"]}`），CPA 自动重新哈希并落盘 + 热加载；再同步 `/opt/cpa/secrets/cpa_management_key` 并 `docker restart cpa-manager-plus`。
- 备份：`/opt/cpa/backups/cli-config.yaml.bak-pre-keyrestore-20260831-044238` 与 `cpa_management_key.bak-pre-keyrestore-20260831-044238`。
- 复测：三条链路均 200；面板端点 6/6 全 200；不受限 key 57 / 受限 key 11 模型；白名单外 403 `model_not_allowed`、白名单内 200；现网 hash 经 `bcrypt.checkpw` 反查确认命中的正是当时的原值。
- 面板侧未再触发封禁（本次错密钥窗口只凑足 2 次 401，未达 5 次阈值）。
- 旁注：日志里 `POST /v1/chat/completions` 的 403（来自 193.123.167.208 自身，即 CPA-Free 回环渠道）是**白名单正常拦截**，今日共 2 次且其中一次发生在本次改动之前，与密钥无关。

### 2026-09-11 升级 CPA v7.2.145 → v7.2.157 + 修复 ZCode Responses 流顺序报错

**触发**：ZCode 报 `Model request failed / Turn execution failed / reason=unknown`（provider `CPA-Free` → `api.274747.xyz`，模型 `buddy/deepseek-v4.1-flash`，走 `/v1/responses`）。最内层 cause 为 `UnknownError: text part msg_<hex>_0 not found`。

**根因（实测复现）**：CPA 的 Responses 转换器 `internal/translator/openai/openai/responses/openai_openai-responses_response.go` 用
`if tcs := delta.Get("tool_calls"); tcs.Exists() && tcs.IsArray()` 判工具调用，而**空数组 `[]` 也满足 `IsArray()`**；上游 OpenAI 兼容中继每个 chunk 都带 `"tool_calls":[]`，于是第一个正文 delta 之后就误发 `response.output_text.done` / `content_part.done` / `output_item.done`，随后剩余正文仍以 delta 继续发出。客户端（ZCode 内置 AI SDK）在 `output_item.done` 时删除文本 part，再收到同 id 的 delta 即抛 `text part ... not found`。**短回复（单 chunk）看不出异常，输出一长必崩**——这正是"偶发"的真相。

**上游已修**：v7.2.146 提交 `6c6473f8`（"ignore empty tool calls array in responses translation"，closes #5333）把该条件改为 `... && tcs.IsArray() && len(tcs.Array()) > 0`。**属上游修复，无需自研补丁。**

**顺带修复的隐患（重要）**：`patches/cpa-whitelist.patch` 长期停留在 **16 文件**、`grep openai_compat_executor` = **0**——2026-08-30 部署的流式收尾修复（norm2）与其回归测试从未被导出进补丁。任何按旧流程重放补丁再建镜像的操作都会**静默丢掉 norm2**。本次先补齐为 **18 文件**，并把 `patches/README.md` 的重导出命令从 `git diff --cached` 改为 `git add -A && git diff HEAD`，同时加入「文件数 + 关键字 + 真实回放编译」三重校验。旧补丁存为 `cpa-whitelist.patch.bak-16file-pre-upgrade-v72157`。

**升级流程（备份先行 → 灰度 → 切换）**：

```bash
# 1) 全量备份（停 CPAMP 让 WAL 落盘 → 整套卷 tar → 配置/secrets/data → MANIFEST+sha256 → 异地副本）
/opt/cpa/backups/upgrade-v7.2.157-20260911T110129Z/
#   含 cli-config.yaml / compose.yaml / secrets/ / data/ / cpamp-volume.tar.gz / MANIFEST.txt
#   校验：PRAGMA integrity_check=ok、data.key 44B、整卷含 usage.sqlite + usage-imports/
#   异地副本已 scp 回本机 cpa-manager-plus/backup-remote/upgrade-v7.2.157-20260911T110129Z/
#   sha256 与服务器 MANIFEST 逐字节一致（cf9d7d9c…）

# 2) 服务器构建
#    源码 = 本机 src/cli-proxy-api（v7.2.157 + 18 文件补丁），打包上传 /tmp/cpa157build
docker build --build-arg VERSION=v7.2.157 --build-arg COMMIT=09a29bd3 \
  --build-arg BUILD_DATE=2026-09-11T11:08:53Z \
  -t eceasy/cli-proxy-api:whitelist-v7.2.157-norm2 .

# 3) 8318 灰度
docker run -d --name cpa-canary -p 127.0.0.1:8318:8317 \
  -v /opt/cpa/cli-config.yaml:/CLIProxyAPI/config.yaml -v /opt/cpa/data:/app/data \
  --network cpa_cpa-net eceasy/cli-proxy-api:whitelist-v7.2.157-norm2

# 4) 切换（compose 已留 compose.yaml.bak-pre-v7.2.157-norm2）
sed -i 's|whitelist-v7.2.145-norm2|whitelist-v7.2.157-norm2|' /opt/cpa/compose.yaml
cd /opt/cpa && docker compose up -d --no-deps cli-proxy-api && docker rm -f cpa-canary
```

**灰度 + 生产验证结果（全过）**：
- 版本：`CLIProxyAPI Version: v7.2.157, Commit: 09a29bd3`
- `/v1/models` 不受限 key **67** 个；受限 key **6** 个；受限 key 白名单外 `403 model_not_allowed`、白名单内非 403
- `/v0/management/models` 带 management key **200**、无 key **401**
- **Responses 流顺序回归（本次核心）**：同一会长输出请求，**生产 v7.2.145 报 425 处顺序违规，切 v7.2.157 后 0 处**；公网侧 482 delta / 0 违规；ZCode 原始场景（reasoning=max + 长输出）635 delta / 0 违规且正常收到 `response.completed`
- 流式 chat：358 chunk、含 `[DONE]` 与 `finish_reason`；非流式 200（1.28s）
- 日志零 panic/fatal；CPAMP 面板（`whitelist-v3` 未动）`/health` 200、经管理密钥访问 CPA 管理接口 200
- 旧镜像全部保留作回滚锚点（`:whitelist-v7.2.145-norm2` 等），未执行 `docker image prune`

**旁注**：公网 `https://api.274747.xyz/healthz` 返回 404 属**既有 nginx 路由**（`/healthz` 只在 `/v1/` location 下转发给 CPA，公网 `/` 走 CPAMP），升级前后一致；CPA 本地 `127.0.0.1:8317/healthz` 为 200，与本次升级无关。

**回滚**：
```bash
cd /opt/cpa
sed -i 's|image: eceasy/cli-proxy-api:whitelist-v7.2.157-norm2|image: eceasy/cli-proxy-api:whitelist-v7.2.145-norm2|' compose.yaml
docker compose up -d --no-deps cli-proxy-api
```

### 2026-09-13 升级 CPA v7.2.157 → v7.2.159 + 补丁基线迁移 + 凭证脱敏

**触发**：跟进上游版本。v7.2.158（27 项）+ v7.2.159（4 项）共 **35 个提交**，含流式与翻译器多处修复：`reasoning` delta 先于 content 处理、Cloudflare 520-526 视为瞬时故障、Gemini 函数调用配对与 thought tokens 计入用量、Claude CAQS 签名校验、Kimi K2.8 模型定义、codex user-agent 0.154.0。

**升级前验证（隔离 worktree，零污染）**：
- 18 文件补丁在 pristine v7.2.159 上 `git apply` **plain 成功**（无需 `--3way`）
- 上游 v7.2.157→v7.2.159 改动 152 个文件，与本项目补丁**仅 2 个重叠**，均可自动合并：
  - `internal/api/server_management.go` —— 上游新增 7 行插件配额路由（`/plugins/:id/quota`、`/quota/providers` 等）
  - `sdk/api/handlers/openai/openai_handlers.go` —— 上游把 `c.JSON(...)` 换成 `h.WriteModelListResponse(c, h.HandlerType(), ...)`，白名单过滤分支包裹该调用后仍成立
- `go build ./...` 退出码 0、`go vet ./...` 0、白名单 6 例（`-run Whitelist`）全 PASS、norm2 收尾 5 例全 PASS
- **对照实验**：`TestOpenAICompatExecutorToolResultContentByInputModalities` 在**未打补丁的干净 v7.2.159** 上同样失败（4 个子用例）→ 属上游自带缺陷（测试与实现不同步），**非本项目引入，不代为修复**（修了只会扩大与上游的差异面）

**备份**：`/opt/cpa/backups/upgrade-v7.2.159-20260913T111213Z/`
- 含 `cpamp-volume.tar.gz`（38 MB，整套卷：`usage.sqlite` 296 MB + `data.key` + `usage-imports/`）、`cli-config.yaml`、`compose.yaml`、`secrets/`、`data/`、`MANIFEST.txt`（全文件 sha256）
- 校验：`PRAGMA integrity_check = ok`、`quick_check = ok`、`data.key` 44 字节
- 异地副本：`backup-remote/upgrade-v7.2.159-20260913T111213Z/`，`cpamp-volume.tar.gz` / `cli-config.yaml` / `compose.yaml` 三方 sha256 与服务器**逐字节一致**（`ce6a9c6b…` / `6145c8ff…` / `c7fb0b57…`）
- 说明：停容器后 SQLite 干净 checkpoint，卷内 `-wal` / `-shm` 已不存在——**这是正常现象**，不是备份遗漏
- ⚠️ 备份含明文密钥与 `data.key`，**不得纳入 git 或对外分享**（已由根目录 `.gitignore` 排除）

**构建**（服务器，源码包 `/tmp/cpa159build`）：
```bash
docker build --build-arg VERSION=v7.2.159 --build-arg COMMIT=ac02da6c \
  --build-arg BUILD_DATE=2026-09-13T11:20:00Z \
  -t eceasy/cli-proxy-api:whitelist-v7.2.159-norm2 .
```
启动日志确认：`CLIProxyAPI Version: v7.2.159, Commit: ac02da6c, BuiltAt: 2026-09-13T11:20:00Z`

**8318 灰度结果（8 项全过）**：

| 验证项 | 期望 | 实测 |
|---|---|---|
| `/v1/models` 不受限 key | 全量 | 28 |
| `/v1/models` 受限 key | 白名单内 | 6（13 条白名单 ∩ 可用模型） |
| 白名单外模型 | 403 | **403 `model_not_allowed`** |
| 白名单内模型 | 非 403 | 通过白名单（400 为上游余额不足，非拦截） |
| `/v0/management/models` 带 key | 200 | **200 / 69 个模型**（全量，绕白名单） |
| 同端点无 key | 401 | 401 |
| 流式 chat | 含 `[DONE]`+`finish_reason` | FB / buddy / BAI / u1s1 四模型均 200 |
| **Responses 流顺序** | 0 违规 | **0 违规**（FB 759 事件、buddy 364 事件，均正常收 `response.completed`） |

**切换**（2026-09-13）：
```bash
cp /opt/cpa/compose.yaml /opt/cpa/compose.yaml.bak-pre-v7.2.159-norm2
sed -i 's|whitelist-v7.2.157-norm2|whitelist-v7.2.159-norm2|' /opt/cpa/compose.yaml
cd /opt/cpa && docker compose up -d --no-deps cli-proxy-api
docker rm -f cpa-canary
```

**生产复测（全过）**：版本 `v7.2.159`；不受限 28 / 受限 4（**等于该 key 白名单条数**）；白名单外 **403 `model_not_allowed`**；management **200 / 401**；**Responses 流顺序 0 违规**（746、484 事件两条）；公网 `api.274747.xyz` models 28 + 白名单外 403；CPAMP `/health` 本地与公网均 200；`panic`/`fatal` **0**；旧镜像 `:whitelist-v7.2.157-norm2` 保留。

**补丁基线迁移**：`v7.2.157 → v7.2.159`，工作树已切到 v7.2.159 + 18 文件补丁；重导出后**三重校验**通过（`diff --git` = 18、`normalizeOpenAIChatMessages` ×3、`synthesizeOpenAIStreamFinish` ×3），并在干净 v7.2.159 worktree 上重放 + `go build` 通过。旧补丁存 `patches/cpa-whitelist.patch.bak-pre-v72159`。

**回滚**：
```bash
cd /opt/cpa
sed -i 's|whitelist-v7.2.159-norm2|whitelist-v7.2.157-norm2|' compose.yaml
docker compose up -d --no-deps cli-proxy-api
```

**顺带完成的凭证脱敏（2026-09-13）**：README 与本文档曾把 CPAMP 管理员密钥、CPA Management Key、客户端 API key 明文写在正文里。排查结论——本项目**不是 git 仓库**（无 commit / 无 remote / 从未外传）、iCloud 开发笔记库 grep 命中 0 文件，**没有外泄**；但明文记录属不当做法，已全部改为「位置指引」并加根目录 `.gitignore` 预排除 `backup-remote/` 等敏感路径。同时修正：README 以往记录的 Management Key 值实测已返回 401（失效），当前有效值以 `/opt/cpa/secrets/cpa_management_key` 为准。

### 2026-09-13 修复 anyrouter / gpt-6-astra 长期 404（改用 Codex 协议接入）

**触发**：日志里长期存在 `404 | upstream execution failed: provider=openai-compatible-anyrouter model=gpt-6-astra ... "当前 API 不支持所选模型 gpt-6-astra"`（首次发现于 2026-09-13 09:56）。用户提供 anyrouter 官网快速开始文档（Claude Code + Codex 两种接入说明），并反馈『备用 API 端点直连可调用、经 CPA 就不行』。

**排查与实测（分两件事，结论不同）**：

1. **备用端点 `pmpjfbhq.cn-nb1.rainapp.top` 不可用，与 CPA 无关**：本机与服务器双侧实测，所有路径（`/`、`/v1`、`/api`、`/health`、`/v1/models`、`/v1/messages`、`/v1/responses`）**全部返回 `404 page not found`**（Go 默认 404）。DNS 解析正常（→ `cn-nb1.website.rainapp.top`，国内 IP）、TCP 443 可连，但服务端无任何 API 路由。CNAME 名里的 `website` 说明它是网站节点，不是 API 节点。

2. **`gpt-6-astra` 真实协议受限** —— 同令牌下逐协议实测：

| 协议路径 | 结果 |
|---|---|
| `GET /v1/models` | **200**，列表含 `gpt-6-astra`（该令牌只可见这 1 个模型） |
| `POST /v1/chat/completions` | **404** `当前 API 不支持所选模型 gpt-6-astra` |
| `POST /v1/messages`（Anthropic） | **404** 同上 |
| `POST /v1/responses`（Codex） | 通（curl 手摸格式报 `invalid codex request`，但**不报模型不支持**） |

**根因**：CPA 把 anyrouter 配在 `openai-compatibility` 段，而该段上游路径在 `internal/runtime/executor/openai_compat_executor.go` 里**写死为 `/chat/completions`**（第 413 行）——而 anyrouter 对 `gpt-6-astra` 只有 Codex 协议通道可用。**配置段选错，不是 CPA bug。**

**解决路径（零代码改动）**：

- 用 **Codex CLI 0.154.0 实跑通**（`CODEX_HOME` 隔离到 `/tmp/codex-anyrouter`，不碰用户 `~/.codex`）：`config.toml` 配 `wire_api = "responses"` + **`env_key`（认证必须走环境变量，`auth.json` 里的 `OPENAI_API_KEY` 无效，实测报 401 `未提供令牌`）**；本机直连 anyrouter 返回 000（不可达），**必须走代理 127.0.0.1:7897**（实测 `proxy=200`）。
- 用本地录制服务器（`/tmp/codex-capture-server.py`，127.0.0.1:8899）**抓到真实请求**：请求头含 `Originator: codex_exec`、`Session-Id`、`Thread-Id`、`X-Codex-Turn-Metadata`、`X-Openai-Internal-Codex-Responses-Lite: true`；请求体顶层为 `{model, input[], tool_choice, parallel_tool_calls, reasoning{effort,context}, store, stream, include[], prompt_cache_key, text{verbosity}, client_metadata}`，`input[]` 首项为 `type: additional_tools`。归档于 `backup-remote/anyrouter-codex-migration-20260913T131438Z/codex-request-captured.txt`。
- **源码调研发现 CPA 本就有解**：配置段 **`codex-api-key`**（`internal/config/config_types.go` 的 `CodexKey`）支持自定义 `base-url` + `models`；`codex-executor` 走 `{base-url}/responses`（`codex_executor_execute.go:78`），且 `applyCodexHeaders` 会**自动补上官方 Codex 身份头**；其模型注册表 `models.json` 的 `codex-team`/`codex-plus`/`codex-pro` 段**已原生包含 `gpt-6-astra`**。→ **无需改代码。**

**变更（只影响 anyrouter 这一个 provider）**：

```diff
 openai-compatibility:
-  - name: anyrouter
-    base-url: https://anyrouter.top/v1
-    api-key-entries:
-      - api-key: sk-dctWQ...VD7M
-    models:
-      - name: gpt-6-astra
-        alias: ""
-        input-modalities: [text, image]
-        output-modalities: [text]
 # （其余 21 个 provider 未动）
+
+codex-api-key:
+  - api-key: sk-dctWQ...VD7M
+    base-url: https://anyrouter.top/v1
+    models:
+      - name: gpt-6-astra
+        alias: ""
```

**验证结果（生产，全过）**：重启日志 `1 Codex keys + 21 OpenAI-compat`；`/v1/chat/completions` 非流式 **200**（3.0s，返回 `ok`）；流式 **200**（18 chunks，含 `[DONE]` + `finish_reason`）；公网 `api.274747.xyz` **200**（2.2s）；`/v1/responses` 入口 **200**。

**后续修复（同日 22:30）：ZCode 报 `invalid codex request` —— `Originator` 头必须是 `codex_exec`**

首次切到 `codex-api-key` 后，我自己的 curl 测试（非流式、流式、公网）全过；但用户在 ZCode 里调用报：

```
Turn execution failed  provider_code=invalid_responses_request  model=gpt-6-astra
status=400  reason=invalid_request  retryable=false
invalid codex request (request id: ...)
```

**排查：抓 CPA 实际发出的请求。** 起了一个记录服务器，把 `codex-api-key` 的 `base-url` 临时指过去。踩了三个坑：

1. `127.0.0.1:8899` —— **CPA 在容器内，127.0.0.1 是容器自己的 loopback**（`connection refused`）
2. `172.19.0.1:8899`（docker 网关/宿主） —— **`no route to host`**（Oracle Cloud iptables 阻止容器访问宿主，DOCKER-USER 链为空也仍不通）
3. 最终用 **alpine 容器加入 `cpa_cpa-net` 网络**跑捕获服务（`apk add python3`），才抓成功

**对比结果（同一 body，只改头）：**

| 请求头 | 结果 |
|---|---|
| `Originator: codex-tui` + `User-Agent: codex-tui/0.154.0 ...`（**CPA 默认 cloaking 值**） | **500** `get_channel_failed`（或 400 `invalid codex request`） |
| `Originator: codex_exec` + `User-Agent: codex_exec/0.154.0 ...` | **200** ✓ |

→ **anyrouter 根据 `Originator` 分流，`codex-tui` 身份的通道不可用**（与「本站无法转发非 Claude Code 的 API 流量」一致，它只开放 Codex CLI 身份）。

**修复（只改配置，不动代码）：**

仅加 `headers` **不够**。源码 `codex_executor_request.go` 的调用顺序是：

```go
util.ApplyCustomHeadersFromAttrs(r, attrs, ginHeaders)  // ① 先应用自定义 headers
applyCodexCloakingHeaders(r.Header, cfg)                // ② 再被 cloaking 无条件覆盖为 codex-tui
```

且 `applyCodexCloakingHeaders` 只在 `cfg.Codex.DisableCodexCloaking` 为真时提前返回。
**必须同时关闭 cloaking**：

```yaml
codex:
  disable-codex-cloaking: true

codex-api-key:
  - api-key: sk-dctWQ...VD7M
    base-url: https://anyrouter.top/v1
    headers:
      Originator: codex_exec
      User-Agent: codex_exec/0.154.0 (Mac OS 26.7.0; arm64) dumb (codex_exec; 0.154.0)
    models:
      - name: gpt-6-astra
        alias: ""
```

**验证“配置能否覆盖客户端自带的头”**（用捕获容器看 CPA 实际发出的头，而不是看客户端响应）：

| 客户端发的 | CPA 实际发出 |
|---|---|
| 无 | `Originator: codex_exec` ✓ |
| `Originator: codex_cli_rs` + `User-Agent: zcode/1.0` | 仍是 **`Originator: codex_exec`** ✓ |

**修复后终测（全过）**：`/v1/responses` 无额外头 **200**、模拟 ZCode 自带头 **200**、公网 `api.274747.xyz/v1/responses` **200**，三例均收到 `response.completed`；捕获容器与临时文件已清理（残留 0）。

**另外开启的调试开关**：`request-log: true`（详细上游请求日志，排查期保留）。备份 `cli-config.yaml.bak-pre-requestlog-*`、`cli-config.yaml.bak-pre-nocloak-*`。

**本机 Codex CLI 持久配置**：

**本机 Codex CLI 持久配置**：

```
~/.codex-anyrouter/config.toml   # model=gpt-6-astra, base_url=https://anyrouter.top/v1, wire_api="responses", env_key="ANYROUTER_API_KEY"
~/.codex-anyrouter/api_key       # 令牌，chmod 600
~/.zshrc                         # codex-anyrouter() 函数（已备份 .zshrc.bak-*）
```

函数内容：注入 `CODEX_HOME` + 从文件读 `ANYROUTER_API_KEY` + 三个代理环境变量，然后透传参数给 `codex`。

持久化验证：`codex-anyrouter exec --skip-git-repo-check "reply with exactly: ok"` → 返回 `ok`（15,215 tokens）。

**踩坑与约定**：
- 用 `CODEX_HOME` 隔离，**不碰用户真实 `~/.codex`**（那里有大量会话/日志库）
- 令牌存 `api_key` 文件（600），**不写进 shell 配置与命令历史**
- 本机直连 anyrouter 不可达（`000`），必须走代理 `127.0.0.1:7897`
- 临时验证目录 `/tmp/codex-anyrouter` 已清理

**已知限制**：上游偶发 `当前模型 gpt-6-astra 负载已经达到上限，请稍后重试`（`code: get_channel_failed`，HTTP 500，日志 provider=`codex`）——anyrouter 侧容量限制，重试即可。

**回滚**：
```bash
cd /opt/cpa
cp backups/cli-config.yaml.bak-pre-codex-anyrouter-<时间戳> cli-config.yaml
docker restart cli-proxy-api
```

### 2026-09-13 故障：`payload:` 段 YAML 写错 → cli-proxy-api 崩溃循环、全站 502

**症状（23:04 起）**：

- 客户端调用全部 `502 Bad Gateway`（nginx/1.24.0）
- cpa-manager-plus 面板报 `dial tcp: lookup cli-proxy-api on 127.0.0.11:53: server misbehaving`，`/v0/management/*` 接口全部 502
- `docker ps`：`cli-proxy-api  Restarting (0)`（exit 0 → `restart: unless-stopped` 无限重启）

**根因链（一条错误配置引发的两个表象）**：

1. 当日切 anyrouter 后追加的 `payload:` 段写错——`default:` 被注释掉，规则数组直接挂在 `payload` 下，尾部还多一个悬空 `-`：

```yaml
payload:
#   default:
    - models: ...
    -
```

   启动即报 `failed to load config: yaml: unmarshal errors: line 475: cannot unmarshal !!seq into config.PayloadConfig`，进程退出。
2. 容器不存在 → 同网络内 cpa-manager-plus（`CPA_UPSTREAM_URL=http://cli-proxy-api:8317`）解析不到服务名，Docker 内嵌 DNS（127.0.0.11）返回 SERVFAIL → 面板侧 `server misbehaving`。**两者同源，修好 CPA 即同时消失。**

**正确 schema**（源码 `internal/config/config_types.go` 的 `PayloadConfig`，v7.2.159）：

| 字段 | 类型 | 语义 |
|---|---|---|
| `payload.default` / `default-raw` | `[]PayloadRule` | 仅当字段缺失时写入（raw = 值按原始 JSON 片段处理） |
| `payload.override` / `override-raw` | `[]PayloadRule` | 无条件覆盖 |
| `payload.filter` | `[]PayloadFilterRule` | 按 JSON 路径删除字段 |

`PayloadRule = { models: [{name, protocol, ...}], params: {json.path: value} }`；`PayloadModelRule` 支持 `name`（通配，如 `gpt-*`）、`protocol`、`from-protocol`、`headers`、`match` / `not-match` / `exist` / `not-exist` 条件。

**修复（`cli-config.yaml` 尾部，两处）**：

```yaml
payload:
  default:                              # ① 解开注释、缩进归位
    - models:
        - name: gpt-6-astra
          protocol: responses
      params:
        prompt_cache_key: "anyrouter-gpt6-astra"
                                        # ② 删除尾部悬空 "-"
```

```bash
docker restart cli-proxy-api
```

坏配置留档：`/opt/cpa/backups/cli-config.yaml.bak-broken-payload-20260913-151245`。

**验证（全过）**：日志 `API server started successfully on: :8317` + `21 clients (1 Codex keys + 20 OpenAI-compat)`；本地 `127.0.0.1:8317` 和公网 `api.274747.xyz` 的 `FB/deepseek-v4-flash` 调用均 **200**；面板 `/v0/management/config` 由 502 → **200**。

**经验（改配置前先校验）**：

```bash
# YAML 语法层（挡缩进/注释类错误）
docker run --rm -v /opt/cpa/cli-config.yaml:/c.yaml python:3-slim \
  python -c "import yaml;yaml.safe_load(open('/c.yaml'));print('yaml ok')"
```

- YAML 通过 ≠ schema 通过（本次是 `payload` 类型不匹配，语法层能过、schema 层必挂）——**最终以重启后的启动日志为准**。
- CPA 有 config file watcher 会热加载，但**进程已退出时热加载无从谈起**；写坏配置 = 直接整机断服。
- 排查口诀：nginx 502 + 面板 `server misbehaving` 同时出现 = **先看 `docker ps` 里 cli-proxy-api 是不是 Restarting**，再看它的日志首行报错（配置解析失败会直接打印行号）。

### 2026-09-14 部署 GPT-6 自动 Codex 身份头（`whitelist-v7.2.159-norm2` → `whitelist-v7.2.159-norm2-gpt6`）

**触发**：补丁新增「GPT-6 家族自动 Codex 身份头」（Spec `05-00-gpt6-auto-codex-identity`，ADR `scene:01-cpa-gateway/0001`），解决「别人安装二改 CPA 少配 headers / 少关 cloaking 就复现 anyrouter 400」的分发型配置依赖。补丁 18 → 22 文件，其余三类自研功能（白名单 / norm1 / norm2）不变。

**升级前验证（干净 worktree + 抓包对照，零污染）**：
- 22 文件补丁在 pristine v7.2.159 worktree 上 `git apply --check` plain 通过 → `go build ./...` 退出码 0 → `go test -run GPT6` **8 例全 PASS**；上游自带失败用例 `TestOpenAICompatExecutorToolResultContentByInputModalities` 用**零改动**干净 worktree 对照复现，确认非本次引入。
- **抓包 A/B（决定性证据）**：用同一份「无 provider headers + cloaking 默认开启」的临时配置，把 `codex-api-key` 的 `base-url` 指向加入 `cpa_cpa-net` 的捕获容器，对照同一请求在旧/新镜像下 CPA 实际发出的头：

| 镜像 | CPA 实际发出 |
|---|---|
| `whitelist-v7.2.159-norm2` | `Originator: codex-tui` / UA `codex-tui/0.154.0 …` |
| `whitelist-v7.2.159-norm2-gpt6` | **`Originator: codex_exec`** / UA `codex_exec/0.154.0 …` |

- **反向对照（同容器）**：非 GPT-6 的 Codex 模型 `gpt-5.4-codex` 仍发 `Originator: codex-tui`（零改动），同容器的 `gpt-6-astra` 发 `codex_exec` —— 证明规则按模型家族精准生效。
- 注：`cpa-capture` 服务与临时配置（含生产 api-key）用完立即销毁删除，服务器残留 0。

**备份**：`/opt/cpa/backups/upgrade-v7.2.159-norm2-gpt6-20260914T031936Z/`
- 含 `cpamp-volume.tar.gz`（39 MB）、`cli-config.yaml`、`compose.yaml`、`secrets/`、`data/`、`MANIFEST.txt`（sha256）
- 校验：`PRAGMA integrity_check = ok`、`quick_check = ok`、`data.key` 44 字节
- 异地副本：`backup-remote/upgrade-v7.2.159-norm2-gpt6-20260914T031936Z/`，`cpamp-volume.tar.gz`（`24d5d452…`）/ `cli-config.yaml`（`63d84542…`）/ `compose.yaml`（`245e7fcf…`）与服务器 MANIFEST **逐字节一致**

**构建**（服务器，源码包 `/tmp/cpa-gpt6build`，源码含 22 文件补丁）：
```bash
docker build --build-arg VERSION=v7.2.159 --build-arg COMMIT=gpt6-20260914 \
  --build-arg BUILD_DATE=2026-09-14T03:25:00Z \
  -t eceasy/cli-proxy-api:whitelist-v7.2.159-norm2-gpt6 .
```
启动日志：`CLIProxyAPI Version: v7.2.159, Commit: gpt6-20260914` + `21 clients (1 Codex keys + 20 OpenAI-compat)`。

**8318 灰度（生产配置，与切换前逐项对照，全部一致）**：

| 验证项 | 切换前基线 | 灰度（新镜像） | 切换后 |
|---|---|---|---|
| `/v1/models` 四个客户端 key | 6 / 4 / 8 / 5 | 6 / 4 / 8 / 5 | 6 / 4 / 8 / 5 |
| 白名单外模型 | 403 `model_not_allowed` | 403 | 403 |
| 白名单内模型 | 200 | 200 | 200（`天机阁/gpt-5.6-sol` 公网 200） |
| `/v0/management/models` 带 key / 无 key | 200 / 401 | 200 / 401 | 200 / 401 |
| 流式 chat（`FB/deepseek-v4-flash`） | 200，含 `[DONE]`+`finish_reason` | 同 | 同 |
| 裸 `gpt-6-astra`（不在任何 sk- key 白名单） | 403 | 403 | 403 |
| `panic` / `fatal` | 0 | 0 | 0 |

**切换**：
```bash
cd /opt/cpa
cp -a compose.yaml compose.yaml.bak-pre-v7.2.159-norm2-gpt6
sed -i 's|eceasy/cli-proxy-api:whitelist-v7.2.159-norm2$|eceasy/cli-proxy-api:whitelist-v7.2.159-norm2-gpt6|' compose.yaml
docker compose up -d --no-deps cli-proxy-api
```

**生产复测**：版本 `Commit: gpt6-20260914`；`healthz` 200；CPAMP `/health` 本地与公网均 200；四个 key 模型数与切换前一致；白名单外 403；公网 `api.274747.xyz` 的 `天机阁/gpt-5.6-sol` **200**；日志 `panic`/`fatal` 0。旧镜像 `eceasy/cli-proxy-api:whitelist-v7.2.159-norm2`（`f072d687e8ea`）保留作回滚锚点。

**已知限制 / 观察**：
- anyrouter 的 `gpt-6-astra` 当天持续返回 `500 get_channel_failed`（「负载已经达到上限」），**切换前后一致**，属上游容量问题，与本次改动无关。连续失败后 CPA 会短暂出现 `503 auth_unavailable`（凭证进入冷却），等待约 1 分钟后自动恢复为上游原报错。
- 裸 `gpt-6-astra` 目前只对 `api-keys` 里的 `tang1234` 开放（该 key 的白名单含 `gpt-6-astra`）；四个 `sk-*` 客户端 key 未包含它，故一律 403 —— 这是白名单配置现状，不是本次改动引入。

**回滚**：
```bash
cd /opt/cpa
sed -i 's|eceasy/cli-proxy-api:whitelist-v7.2.159-norm2-gpt6$|eceasy/cli-proxy-api:whitelist-v7.2.159-norm2|' compose.yaml
docker compose up -d --no-deps cli-proxy-api
```

### 回滚

```bash
# CPA 退回带归一化修复前的 v7.2.145（白名单版）
cd /opt/cpa
sed -i 's|image: eceasy/cli-proxy-api:whitelist-v7.2.145-norm1$|image: eceasy/cli-proxy-api:whitelist-v7.2.145|' compose.yaml
docker compose up -d --no-deps cli-proxy-api

# 或退回 v7.2.143 基线镜像
cd /opt/cpa
sed -i 's|image: eceasy/cli-proxy-api:whitelist-v7.2.145-norm1$|image: eceasy/cli-proxy-api:whitelist|' compose.yaml
docker compose up -d --no-deps cli-proxy-api

# 或直接恢复切换前的 compose.yaml
cp /opt/cpa/backups/upgrade-v7.2.145-20260828T185106Z/compose.yaml /opt/cpa/compose.yaml
docker compose up -d

# 若需恢复面板数据（必须先 docker stop cpa-manager-plus）
docker run --rm -v cpa_cpa-manager-plus-data:/data -v <备份目录>:/backup:ro \
  alpine sh -c 'rm -f /data/* && tar xzf /backup/cpamp-volume.tar.gz -C /data'
```
旧镜像 `eceasy/cli-proxy-api:whitelist`、`:latest` 与 `seakee/cpa-manager-plus:whitelist`、
`:whitelist-v2` 均保留，勿执行 `docker image prune`。

## 常见运维操作

### 查看服务状态
```bash
docker ps | grep -E "cli-proxy-api|cpa-manager-plus"
curl -sS http://127.0.0.1:18317/health
```

### 查看 CPA 日志
```bash
docker logs cli-proxy-api --tail 50
```

### 更新 CPA（保留白名单功能！）
```bash
# 0. 先确认补丁是最新的（改动落地后必须重导出，否则会静默丢功能）
cd <本仓库>/src/cli-proxy-api
git add -A && git diff HEAD > ../../patches/cpa-whitelist.patch
grep -c '^diff --git' ../../patches/cpa-whitelist.patch          # 期望 18
grep -c synthesizeOpenAIStreamFinish ../../patches/cpa-whitelist.patch  # 期望 >=1（norm2 在位）

# 1. 检出目标 tag 并打补丁（补丁基线当前为 v7.2.157，详见 patches/README.md）
cd /tmp/cpa-build
git fetch --tags origin
git checkout --detach v<新版本> && git clean -fd
git apply --3way /path/to/patches/cpa-whitelist.patch

# 2. 构建前硬门禁：编译 + vet + 测试必须全过，禁止绕过
go build ./... && go vet ./... && go test ./... -race -count=1
docker build --build-arg VERSION=v<新版本> --build-arg COMMIT=<sha> \
  -t eceasy/cli-proxy-api:whitelist-v<新版本>-norm2 .

# 3. 灰度（8318）跑完验证项后，再改 compose.yaml 的 image 并重启
#    验证项含：版本号 / 不受限与受限模型数 / 白名单外 403 / management 401+200
#              / Responses 流顺序 0 违规 / 流式与非流式 chat / 日志零 panic
cd /opt/cpa && docker compose up -d --no-deps cli-proxy-api
```

### 更新 CPAMP（保留白名单 UI！）
```bash
# 1. 应用 patch（基线 v1.12.5；升 v1.12.6 需 --3way，见 patches/README.md）
git checkout --detach v<新版本> && git clean -fd
git apply /path/to/patches/cpamp-whitelist.patch

# 2. 构建前硬门禁
cd apps/web && npm ci && npm run type-check && npx vitest run && npm run build

# 3. 多阶段 Dockerfile（前端 Vite + Go）；切换前必须先整套备份 CPAMP 卷
docker build -f Dockerfile.manager-server --build-arg VERSION=v<版本> \
  -t seakee/cpa-manager-plus:whitelist-vN .
cd /opt/cpa && docker compose up -d --no-deps cpa-manager-plus
```

### 升级 CPAMP 到 v1.12.6（暂缓中，升级前必读）
v1.12.6 会把 legacy 的 `CPA_UPSTREAM_URL` + `CPA_MANAGEMENT_KEY_FILE`（**当前生产正是这种配置**）
迁移进加密 SQLite，并新增 fail-closed 逻辑：迁移判定为损坏/冲突时**拒绝启动、不创建替代密钥**，
需手工跑 `store-cpa-connection --repair-conflict`。官方要求把 `usage.sqlite` + `-wal` + `-shm` +
`data.key` 当**一整套文件集**备份。收益偏安全收敛（management key 不再下发浏览器）与配额展示，
因此本次有意暂缓，等 v1.12.7 或确认迁移稳定后再评估。

### 证书续签（已自动）
```bash
crontab -l | grep acme    # 应为每日 4 次
```

### 修改 CPA 管理密钥（remote-management.secret-key）—— 必须同步面板，否则全站 403

面板代理调用 CPA 时用的是**容器启动时**读入的密钥（`CPA_MANAGEMENT_KEY_FILE` → `/opt/cpa/secrets/cpa_management_key`），
而 CPA 侧改密钥会**热加载立即生效**，两边一旦不一致：面板连续 401 → 触发 CPA 防爆破（同 IP 连 5 次失败封 30 分钟）→ **全站 403**。

```bash
# 1) 原地截断写（单文件 bind mount 跟 inode，用 sed -i 换 inode 会导致容器里仍看到旧内容）
printf "%s\n" "新密钥" > /opt/cpa/secrets/cpa_management_key

# 2) 重启面板重读密钥 + 重启 CPA 清除内存态 IP 封禁
docker restart cpa-manager-plus cli-proxy-api

# 3) 验证（三条都应为 200）
curl -s -o /dev/null -w "%{http_code}\n" -H "Authorization: Bearer 新密钥" http://127.0.0.1:8317/v0/management/config
curl -s -o /dev/null -w "%{http_code}\n" -H "Authorization: Bearer $CPA_MANAGER_ADMIN_KEY" http://127.0.0.1:18317/v0/management/config
curl -s -o /dev/null -w "%{http_code}\n" -H "Authorization: Bearer $CPA_MANAGER_ADMIN_KEY" https://api.274747.xyz/v0/management/config
```

> 说明：CPA 会把 `secret-key` 的明文自动哈希为 bcrypt 并写回 `cli-config.yaml`（`config_load.go`），所以改完后文件里看到的是 `$2a$10$…` 而不是明文，**这是正常的，不要手动改回去**。
> 如果忘了新密钥，可把 `secret-key` 重写为一个已知明文后 `docker restart cli-proxy-api`（会重新哈希并落盘），再按上面步骤同步面板侧。

### 修改 CPAMP 面板登录密钥（admin key）

> ⚠️ **改 `compose.yaml` 里的 `CPA_MANAGER_ADMIN_KEY` 是无效的**（旧版本文档写错了）。
> 该环境变量只在**首次启动、SQLite 里还没有凭证时**播种一次（`bootstrap.ensureAdminCredential` 发现已有凭证就直接返回），
> 改完重启后登录密钥**仍然是旧值**。要换密钥必须走下面的重置命令。

```bash
# 面板登录密钥存在 SQLite，用内置子命令重置（会直接改 settings.admin_credential_v1）
docker stop cpa-manager-plus
docker run --rm \
  -v cpa_cpa-manager-plus-data:/data \
  -e USAGE_DB_PATH=/data/usage.sqlite \
  seakee/cpa-manager-plus:whitelist-v3 \
  reset-admin-key --admin-key '新面板登录密钥'
docker start cpa-manager-plus
```

> 卷名以 `docker volume ls | grep manager-plus` 为准（当前为 `cpa_cpa-manager-plus-data`）。
> 该命令需先停服务，否则报 `acquire admin reset lock; stop Manager Server and retry: manager database process lock is already held for /data/usage.sqlite`（实测：服务在跑时该重置会被锁直接拒绍，**不会** partially 改坏凭证）。详见源码仓 `docs/reset-admin-key.zh-CN.md`。

## 关键配置

### /opt/cpa/cli-config.yaml（CPA 配置）
```bash
# 面板登录密钥从 compose 环境变量取（不在本文档明文记录）
CPA_MANAGER_ADMIN_KEY=$(ssh root@193.123.167.208 "grep -oP '(?<=CPA_MANAGER_ADMIN_KEY=).*' /opt/cpa/compose.yaml")
```
```yaml
host: ""
port: 8317
remote-management:
  allow-remote: true
  secret-key: "$2a$10$…"   # bcrypt hash；明文存放于 /opt/cpa/secrets/cpa_management_key，本文档不记录该值
usage-statistics-enabled: true
api-keys:
  - <客户端 key，见服务器配置>
api-key-models:       # 白名单（自研字段，可省略）
  sk-xxx:
    - deepseek-v4-flash
```

### nginx（/etc/nginx/sites-available/api.274747.xyz）
- 80 → 301 HTTPS；443 ssl → /v1/ 转 CPA:8317，/ 转 CPAMP:18317
- `client_max_body_size 100m`
- SSE 流式优化：proxy_buffering off / 600s 超时 / websocket upgrade

### acme.sh 证书
- 路径：`/etc/nginx/ssl/api.274747.xyz.{crt,key}`
- reloadcmd：`nginx -s reload`
- 每日 4 次自动检查续签
