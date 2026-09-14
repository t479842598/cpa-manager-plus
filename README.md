# CPA-Manager-Plus 部署项目

LLM API 聚合网关部署与维护项目。

- **CPA (CLIProxyAPI)**：LLM API 聚合代理网关（端口 8317）——当前 **v7.2.159 + 白名单 + 消息归一化 + 流式收尾修复（whitelist-v7.2.159-norm2）**
- **CPAMP (CPA-Manager-Plus)**：CPA 管理面板 + 可观测仪表盘（端口 18317）——当前 **v1.12.5-whitelist-v3**（上游基线 v1.12.5，有意不升 v1.12.6）
- **域名**：https://api.274747.xyz
- **服务器**：Oracle 193.123.167.208（Ubuntu 24.04 ARM64 / 2H12G）

## 目录结构

```
cpa-manager-plus/
├── README.md                 # 本文件
├── src/
│   ├── cli-proxy-api/        # CPA 源码（v7.2.159 + 白名单补丁，与生产构建逐文件一致）
│   ├── cpa-manager-plus/     # CPAMP 源码（v1.12.5 + 白名单补丁，与生产构建逐文件一致）
│   └── backup/               # 生产镜像原始完整源码归档（历史锚点）
├── patches/
│   ├── README.md             # ⚠️ 重放/重导出补丁必读（含踩坑说明）
│   ├── cpa-whitelist.patch   # CPA 补丁（22 文件 = 白名单 14 + 归一化 2 + 收尾修复 2 + GPT-6 自动身份头 4）
│   └── cpamp-whitelist.patch # CPAMP 白名单补丁（9 文件）
├── backup-remote/            # 服务器备份的异地副本
└── docs/
    └── deployment.md         # 部署与运维记录（含升级/回滚/备份路径）
```

## 核心功能二：OpenAI 消息归一化（自研扩展，norm1）

上游 CPA 对 OpenAI→OpenAI 请求**原样透传**（只改 model 字段），而 b.ai 等严格上游（Rust serde / DeepSeek thinking）会拒收 Codex/Claude 风格的消息形态，导致 `400001 unknown variant developer` 与 `400 reasoning_content must be passed back`。本项目在 `ConvertOpenAIRequestToOpenAI` 转发前做归一化：

- `developer` role → `system`
- assistant 的 `thinking`/`redacted_thinking`/`reasoning` content 块 → 抽成顶层 `reasoning_content`，可见内容折叠回纯文本
- 每条 assistant 消息保证带 `reasoning_content` 字段（缺则补空串）

纯字符串消息零拷贝原样返回，不影响其它上游。

## 核心功能三：OpenAI 流式收尾修复（自研扩展，norm2）

部分 OpenAI 兼容上游（实测 FB/deepseek-v4-flash 经 freebuff2api 链路，近 7 天 53 次）会**完整推完正文与 tool_calls 参数后，直接以 `[DONE]` 或干净 EOF 收尾，却从不发送带 `finish_reason` 的终止 chunk**。CPA 的 OpenAI→OpenAI 响应转换是纯透传，残缺流原样转发给客户端；pi-ai 等严格客户端据此抛 `Stream ended without finish_reason`（CPA 侧因已正常记账而显示成功，造成"上游成功、客户端失败"的错位）。

修复在 `openai_compat_executor.go` 的流式循环：跟踪上游是否发过非空 `finish_reason`、是否推过 `tool_calls` delta、是否有过任何 choice 内容；当流**干净结束**（收到 `[DONE]` 或 EOF 补发 `[DONE]`）且满足"有内容但缺收尾帧"时，在 `[DONE]` 之前合成一个标准 `chat.completion.chunk`（`finish_reason` 按有无 tool_calls 取 `tool_calls`/`stop`，复用上游 id/model），再交给既有翻译器转成各客户端格式。空流不伪造终止事件；上游已带收尾帧时零改动，不重复注入。

- 回归测试 `openai_compat_executor_finish_test.go` 覆盖 5 场景：缺收尾帧补 stop、tool_calls 补 tool_calls、已有收尾帧不重复、EOF 无 DONE 也补、空流不补。

## 核心功能四：GPT-6 家族自动 Codex 身份头（自研扩展，gpt6）

部分 Codex 转发站（如 anyrouter）**按 `Originator` 分流**：`codex_exec` 身份可用，`codex-tui`（CPA 默认 cloaking 值）会报 `400 invalid codex request` / `500 get_channel_failed`。原先必须给每个 provider 手写 `headers` **并**关掉 `codex.disable-codex-cloaking`（见下节），装到别人机器上很容易漏配。

补丁现在内置「识别到 GPT-6 家族模型 → 自动改用 Codex CLI 身份头」：

- 命中模型：`gpt-6`、`gpt-6-astra`、`gpt-6.0` 等（含 `团队/gpt-6-astra` 前缀名与 `gpt-6-astra(high)` 思考后缀）
- 自动设置 `Originator: codex_exec` + `User-Agent: codex_exec/0.154.0 (Mac OS 26.7.0; arm64) dumb (codex_exec; 0.154.0)`，并在缺失时补 `Session_id`
- **只在非官方上游生效**：`base-url` 指向 `chatgpt.com` / `openai.com`（或留空走默认官方后端）时一律不干预，官方账号照旧用 `codex-tui` 身份
- **不覆盖显式配置**：provider `headers:` 里写过的 `Originator` / `User-Agent` 优先；`models.json` 的 `config.override_header` 也优先（应用顺序在自动规则之后）
- 客户端自带的 `Originator`（如 ZCode 的 `codex_cli_rs`）在命中时会被改写为 `codex_exec`，不会漏到上游

实现位置：`internal/runtime/executor/codex_executor_request.go`（`isGPT6FamilyModel` / `applyGPT6CodexIdentityHeaders`），在 `codex_executor_execute.go`（流式 + `/responses/compact`）与 `codex_executor_stream.go` 的 `applyCodexHeaders` 之后调用。回归测试 `gpt6_codex_identity_test.go` 8 例覆盖：家族识别、前缀/后缀名、官方后端不干预、覆盖 cloaking、丢弃客户端 Originator、让位于 provider `headers`、让位于 `config.override_header`、空输入不 panic。

> 限制：WebSocket 传输（`codex-api-key` 的 `websockets: true`）未接这条自动规则，走 SSE 的常规 `/responses` 路径已覆盖。

## 已修复：Responses 流事件顺序（上游 v7.2.146 修复，2026-09-11 升级生效）

ZCode 报 `Model request failed`（`/v1/responses` + 长输出）的根因在 CPA 的 Responses 转换器 `openai_openai-responses_response.go`：判断工具调用用的是
`delta.Get("tool_calls").IsArray()`，而**空数组 `[]` 也满足该条件**。上游中继每个 chunk 都带 `"tool_calls":[]`，于是第一个正文 delta 后就误发
`response.output_item.done`，随后剩余正文继续以 delta 发出；客户端在 done 时删掉文本 part，再收同 id 的 delta 即抛 `text part … not found`。

上游 **v7.2.146**（`6c6473f8`，closes #5333）改为 `… && len(tcs.Array()) > 0` 修复。本项目 2026-09-11 由 v7.2.145 升级到 **v7.2.157** 生效——
实测同一会长输出请求：升级前生产报 **425** 处顺序违规，升级后 **0** 处。**此修复属上游代码，不在本项目补丁内，升级上游版本即获得。**

## 核心功能：per-key 模型白名单（自研扩展）

上游 CPA/CPAMP **原生不支持**「某个 API key 只能访问部分模型」。本项目通过改源码实现了该功能：

- **CPA 后端**：config 新增 `api-key-models` 映射字段（`{key: [模型列表]}`），认证时把白名单挂到请求 context，模型路由前检查，不在白名单返回 `403 model_not_allowed`
- **`/v1/models` 同步过滤**：受限 key 只能看到自己白名单内的模型，「看得见」与「调得动」严格一致
- **`GET /v0/management/models`**（新增）：management key 认证，返回**全量**模型目录且不受白名单影响，供面板列候选模型；经 CPAMP 现有通用透传可达，**无需改 nginx**
- **CPAMP 前端**：API Key 弹窗内多选勾选模型（带搜索/全选/清空），候选来自全量端点；**加 key + 配白名单一步落盘**，无需再点顶部「保存配置」；删除 key 同样一步落盘并带危险操作确认
- **回归测试**：CPA 10 例 + CPAMP 5 例，覆盖 403 拦截、列表过滤、全量端点、YAML 往返与一步写入

### 配置示例

```yaml
api-keys:
  - sk-unrestricted-example   # 不限制模型
  - sk-whitelisted-key        # 受限 key（下面配置白名单）

api-key-models:
  sk-whitelisted-key:
    - deepseek-v4-flash
    - gpt-5.6-luna
```

## 关键凭证

> 🔐 **本文档不记录任何明文凭证**（2026-09-13 起）。所有密钥只存在服务器上，需要时按「位置」一列去取。
> 旧版 README 曾把管理员密钥与 API key 明文写在表格里——虽然本项目不是 git 仓库、从未外传，但明文记录仍属不当做法，已全部移除。

| 项目 | 位置 / 说明 |
|---|---|
| CPAMP 面板 | https://api.274747.xyz |
| CPAMP 管理员密钥（登录框标签写「管理密钥」，实为 admin key） | 服务器 `/opt/cpa/compose.yaml` 的 `CPA_MANAGER_ADMIN_KEY` 环境变量 |
| CPA Management Key（面板代理调用 CPA 的上游钥匙） | 服务器 `/opt/cpa/secrets/cpa_management_key`（明文，44 字节）；`cli-config.yaml` 里只存它的 bcrypt hash |
| 客户端 API key | 服务器 `/opt/cpa/cli-config.yaml` 的 `api-keys` 段 |
| 服务器 SSH | root@193.123.167.208（密钥认证；密码见运维记录，亦不写入本文件） |

取密钥的通用命令：

```bash
ssh root@193.123.167.208 'cat /opt/cpa/secrets/cpa_management_key'          # CPA Management Key
ssh root@193.123.167.208 'grep CPA_MANAGER_ADMIN_KEY /opt/cpa/compose.yaml'  # 面板管理员密钥
```

> ⚠️ **这是两把互不相干的钥匙**：登录面板用的是 CPAMP 管理员密钥（存 SQLite），CPA Management Key 是面板代理调用 CPA 时用的上游钥匙。
> 改 CPA Management Key **不会**改面板登录密码；且必须同步面板侧的 secret 文件并重启容器，否则面板会被 CPA 判定为爆破、封 IP 30 分钟（全站 403）。
> 正确流程与故障排查见 [`docs/deployment.md`](docs/deployment.md) 的「修改 CPA 管理密钥」一节。

> ⚠️ **旧记录已过期**：README 以往记录的 Management Key 值 `86b836b1…` 在 2026-09-13 实测返回 401，**已失效**。当前有效值一律以 `/opt/cpa/secrets/cpa_management_key` 为准。

> ⚠️ **本地备份目录含明文密钥**：`backup-remote/` 下有完整 `cli-config.yaml`（含全部 API key）与含 `data.key` 的加密卷包，这是备份的必需内容，**切勿将其纳入 git 或对外分享**；根目录 `.gitignore` 已预先将其排除。

## 特殊上游：anyrouter / gpt-6-astra —— 必须走 Codex 协议（2026-09-13 适配）

anyrouter（`anyrouter.top`）是 **Claude Code / Codex 转发站，不支持通用的 OpenAI chat/completions 协议**。「本站直接接入官方 Claude Code 转发，无法转发非 Claude Code 的 API 流量」——官网原文。

> **2026-09-14 起自动生效**：补丁新增 GPT-6 家族自动 Codex 身份头（见「核心功能四」）——`gpt-6-astra` 这类模型即使不配 `headers`、不开 `disable-codex-cloaking`，也会自动使用 `codex_exec` 身份。下面的手工配置仍然有效，且优先级高于自动规则（用于非 GPT-6 命名或需要别的身份时）。

实测（同一令牌 `gpt-6-astra`）：

| 协议路径 | 结果 |
|---|---|
| `GET /v1/models` | **200**，列表里有 `gpt-6-astra`（该令牌只可见这 1 个模型，`owned_by: custom`） |
| `POST /v1/chat/completions` | **404** `当前 API 不支持所选模型 gpt-6-astra` |
| `POST /v1/messages`（Anthropic，x-api-key / Bearer 均试） | **404** 同上 |
| `POST /v1/responses`（Codex） | **通**（需真 Codex 身份头） |

### ⚠️ 关键：`Originator` 必须是 `codex_exec`（2026-09-13 实测）

ZCode 报 `invalid codex request (400, provider_code=invalid_responses_request)` 的根因：
**anyrouter 按 `Originator` 分流，`codex-tui` 身份的通道不可用**（表现为 `400 invalid codex request` 或 `500 get_channel_failed`）。

对照实验（同一 body，**只改请求头**）：

| 请求头 | 结果 |
|---|---|
| `Originator: codex-tui` + codex-tui UA | **500** `get_channel_failed` / **400** `invalid codex request` |
| `Originator: codex_exec` + codex_exec UA | **200** ✓ |

#### 为什么先前的 headers 配置不生效（关键陷阱）

仅加 `headers: {Originator: codex_exec}` **不够** —— `internal/runtime/executor/codex_executor_request.go` 的调用顺序是：

```go
util.ApplyCustomHeadersFromAttrs(r, attrs, ginHeaders)  // ① 先应用我们配的 headers
applyCodexCloakingHeaders(r.Header, cfg)                // ② 然后被 cloaking 无条件覆盖！
```

```go
func applyCodexCloakingHeaders(headers http.Header, cfg *config.Config) {
    if headers == nil || cfg == nil || cfg.Codex.DisableCodexCloaking {
        return                                    // ← 只有开关打开才跳过
    }
    headers.Set("User-Agent", codexUserAgent)     // 强制覆盖
    headers.Set("Originator", codexOriginator)    // 强制覆盖为 codex-tui
}
```

**必须同时关闭 cloaking**，否则自定义 `Originator` 被顶掉。

#### 完整修复（两项都要）

```yaml
codex:
  disable-codex-cloaking: true      # 关键：否则 headers 被 cloaking 覆盖

codex-api-key:
  - api-key: <anyrouter 令牌>
    base-url: https://anyrouter.top/v1
    headers:
      Originator: codex_exec
      User-Agent: codex_exec/0.154.0 (Mac OS 26.7.0; arm64) dumb (codex_exec; 0.154.0)
    models:
      - name: gpt-6-astra
        alias: ""
```

#### 验证：配置能强制覆盖客户端自带的头

ZCode 会带自己的 `Originator`，但实测 **配置优先**（用捕获容器验证 CPA 实际发出的头）：

| 客户端发的头 | CPA 实际发出 |
|---|---|
| 无 | `Originator: codex_exec` ✓ |
| `Originator: codex_cli_rs` + `User-Agent: zcode/1.0` | 仍是 **`Originator: codex_exec`** ✓ |

> 排查方法：起一个捕获服务（**必须是加入 `cpa_cpa-net` 网络的容器**）。注意 **CPA 在容器内，`127.0.0.1` 是容器自身回环**；宿主 IP（`172.19.0.1`）从容器也不可达（Oracle iptables），只能同网络容器。

### 接入方式：放 `codex-api-key` 段，不是 `openai-compatibility`

`openai-compatibility` 段的上游路径在 executor 里**写死为 `/chat/completions`**，必然 404。正确做法是放进 **`codex-api-key`** 段——CPA 会走 `{base-url}/responses` 并自动附加 Codex 身份头（`Originator: codex_exec`、`Session-Id`、`X-Codex-Turn-Metadata` 等）：

```yaml
codex-api-key:
  - api-key: <anyrouter 令牌>
    base-url: https://anyrouter.top/v1
    models:
      - name: gpt-6-astra
        alias: ""
```

> CPA 的 codex 模型注册表（`internal/registry/models/models.json`）**已原生包含 `gpt-6-astra`**（在 `codex-team` / `codex-plus` / `codex-pro` 段），因此**无需改任何代码**，纯配置即可接入。

### 验证结果（生产，2026-09-13）

| 入口 | 结果 |
|---|---|
| `/v1/chat/completions`（非流式） | **200**，3.0s，返回 `ok` |
| `/v1/chat/completions`（流式） | **200**，18 chunks，含 `[DONE]` + `finish_reason` |
| `/v1/responses`（ZCode 路径，带 tools+instructions） | **200** |
| `/v1/responses`（模拟 ZCode 自带 `Originator`/`UA`） | **200**（配置强制覆盖生效） |
| 公网 `https://api.274747.xyz/v1/responses` | **200** |

客户端侧统一用 OpenAI 协议调 CPA 即可，协议转换由网关完成。

### 已知限制

上游偶发 **`当前模型 gpt-6-astra 负载已经达到上限，请稍后重试`**（`code: get_channel_failed`，HTTP 500）——属 anyrouter 侧容量限制，**非网关问题**，重试即可。

### 本机 Codex CLI 直连（持久配置，2026-09-13）

除了走网关，也可以在本机直接用 Codex CLI 连 anyrouter（绕过 CPA，适合直接验证上游或临时排查）。已做成持久配置：

```
~/.codex-anyrouter/
├── config.toml   # model=gpt-6-astra, wire_api="responses", env_key="ANYROUTER_API_KEY"
└── api_key       # 令牌，权限 600（不写进 shell 配置）
```

`~/.zshrc` 里追加了同名函数（用函数而非 alias，便于注入多个环境变量 + 透传参数）：

```bash
codex-anyrouter() {
  CODEX_HOME="$HOME/.codex-anyrouter" \
  ANYROUTER_API_KEY="$(cat "$HOME/.codex-anyrouter/api_key" 2>/dev/null)" \
  HTTPS_PROXY=http://127.0.0.1:7897 HTTP_PROXY=http://127.0.0.1:7897 ALL_PROXY=http://127.0.0.1:7897 \
  codex "$@"
}
```

用法：`codex-anyrouter`（交互）或 `codex-anyrouter exec "问题"`（非交互）。

**为什么要 `CODEX_HOME` 隔离**：用户真实配置在 `~/.codex`（含会话/日志/记忆库），直接改会破坏现有环境。
**为什么要代理**：本机直连 `anyrouter.top` 返回 `000`（不可达），必须走 `127.0.0.1:7897`（实测 `proxy=200`）。
**为什么 key 单独存文件**：避免明文令牌出现在 shell 配置与命令历史里。

### 备用端点不可用

官方公告提到的备用端点 `https://pmpjfbhq.cn-nb1.rainapp.top` 实测**所有路径**（`/`、`/v1`、`/api`、`/health`、`/v1/models`、`/v1/messages`）均返回 `404 page not found`（Go 默认 404）。其 CNAME 指向 `cn-nb1.website.rainapp.top`，是**网站节点而非 API 节点**，当前不可用于接口调用。

## 升级时保留白名单功能（重要）

本项目对上游源码做了改动，**升级前务必先打补丁**，避免功能丢失：

```bash
# 1. 在新检出的源码上应用 patch
cd <源码目录>
git apply ../patches/cpa-whitelist.patch      # CPA
git apply ../patches/cpamp-whitelist.patch    # CPAMP

# 2. 构建前硬门禁：必须编译 + 测试全过，能编译才算补丁完整
go build ./...                                # CPA
npm run type-check && npm run build           # CPAMP

# 3. 重新构建镜像、灰度验证、再部署
```

> ⚠️ **重新导出补丁时必须先 `git add -A`，并用 `git diff HEAD`（不要用 `git diff --cached`）**。
> 补丁包含上游不存在的新文件，裸 `git diff` 会静默丢掉它们——历史上就因此漏掉 `handlers_routing.go` 的方法定义，
> 导致备份无法重建生产镜像。另外**改动落地后必须重新导出补丁**：norm2 流式收尾修复曾因未重新导出而长期缺失
> （补丁停留在 16 文件，直到 2026-09-11 升级 v7.2.157 才补齐为 18 文件）。详见 [`patches/README.md`](patches/README.md)。
>
> 生产部署流程（备份先行、灰度验证、回滚命令）见 [`docs/deployment.md`](docs/deployment.md)。
