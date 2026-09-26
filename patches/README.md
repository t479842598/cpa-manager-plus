# Patches（自研 per-key 模型白名单补丁）

上游 CPA / CPAMP 均不支持「某个 API key 只能访问部分模型」，本目录的两个补丁实现了该功能的完整链路。

## 当前基线

| 补丁 | 上游基线 | 文件数 | 校验状态 |
|---|---|---|---|
| `cpa-whitelist.patch` | `router-for-me/CLIProxyAPI` **v7.3.18** | 30（白名单 14 + 消息归一化 2 + 流式收尾修复 2 + GPT-6 自动身份头 4 + **思考档位默认值 3** + **api-key-models 管理端点 2** + **config.yaml prune 1 + 测试 2**） | 干净 v7.3.18 worktree 上 `git apply --check`（plain，无需 `--3way`，无空白告警）+ 30/30 文件逐字节一致 + `go build ./...` 退出码 0 + `go test ./...` 97 包全过 + 沙箱灰度 23/23 |
| `cpamp-whitelist.patch` | `seakee/CPA-Manager-Plus` **v1.14.1** | 9（2 个新文件：`apiKeyModels.ts` + 其测试） | 干净 v1.14.1 worktree 上 `git apply --check`（plain）+ 9/9 文件逐字节一致 + `tsc --noEmit` + `vitest`(4174) + 仓库级测试(257) + `vite build` 全通过 |

> ### 2026-09-26 升级基线 **CPA v7.2.159 → v7.3.18** + **CPAMP v1.12.5 → v1.14.1**（616 个上游提交）
>
> 触发：pi-web 侧选「max」思考强度经 CPA 后变成 `high`。上游 v7.3.18 **未修复**该问题
> （`config.example.yaml` 明确把它写成预期行为：未声明 `thinking:` 的模型默认注入
> `[low,medium,high]`，超出档位夹到 high）；因此修复做进本补丁，并把 CPA/CPAMP 一并升到最新。
>
> **本次补丁新增：**
> 1. **思考档位默认值修复**（`sdk/cliproxy/service_models.go`、`sdk/cliproxy/auth/api_key_model_capabilities.go`）：默认注入档位由 `[low,medium,high]` 扩为 `[low,medium,high,xhigh,max]`，客户端显式发送的 `max`/`xhigh` 原样透传；模型显式声明的 `thinking.levels` 仍然优先。沙箱实测 `max→max`、`xhigh→xhigh`，`high/medium/low` 不变。
> 2. **`/v0/management/api-key-models` 管理端点**（GET/PUT/PATCH/DELETE）：面板可以把「加 key」与「配白名单」都走管理 API 一步落盘，不再整份重写 `config.yaml`（对齐上游 v1.14.1 对 `api-keys` 的新架构）。PATCH 是声明式幂等语义：非空列表=设置，空列表=取消限制。
> 3. **`config.yaml` prune 修复**：上游 `SaveConfigPreserveComments` 只对白名单里的 mapping 做 prune（`oauth-excluded-models` 等），**未列入的 mapping 会保留原文件里的旧键**——因此通过管理端点**删除**一条 `api-key-models` 只会改内存、文件里那一条会留下来，并在下次 reload 复活。新增 `pruneMappingToGeneratedKeys(..., "api-key-models")` 一行修复，并补 2 例回归测试（去掉修复即失败，已验证）。
>
> **CPAMP 补丁本次是重写而非平移**：上游 v1.14.1 把 API key 的增删改整体改成「管理端点即时落盘」
> （`onPersistApiKeyMutation` + `apiKeysApi`，并有 source-dirty 守卫、canonical 回读、快照刷新），
> 旧补丁依赖的「可视化配置一步提交（`onRequestCommit` / `commitVisualChangesNow`）」在新架构里已不成立。
> 现在白名单走 CPA 侧新端点，因此 **`VisualConfigEditor.tsx`、`ConfigPage.tsx`、`useVisualConfig.ts`、
> `types/visualConfig.ts` 四个文件不再需要改动**，`api-key-models` 也不进入可视化值（整份保存时
> 作为未知键原样保留），补丁面反而更小。
>
> 同时修掉一个上游 v1.14.1 的行为缺口：改动前的「编辑已有 key（未改名）」分支是**纯别名**流程，
> 白名单改动会被它的 early-return 丢掉；且该分支要求别名非空，没有别名的 key 根本存不了。
> 现在先做别名校验（仅在别名变更时）、再持久化白名单，别名未变时也照常落盘并提示。
>
> ⚠️ **v7.3.18 上游自带一个失败测试**：`TestOpenAICompatExecutorToolResultContentByInputModalities`（4 个子用例）在**未打补丁的干净基线**上同样失败（上一轮基线 v7.2.159 起就如此，属上游测试与实现不同步），**非本补丁引入**，本项目不代为修复。


> 2026-09-13 升级基线 **v7.2.157 → v7.2.159**（补丁内容未变，仅基线前移）。新基线下实测：`git apply` plain 成功、18 文件、`go build ./...` 退出码 0；干净 v7.2.159 worktree 重放同样通过。上游 v7.2.157→v7.2.159 的 152 个改动文件中**仅 2 个**与本补丁重叠（`internal/api/server_management.go` 新增插件配额路由、`sdk/api/handlers/openai/openai_handlers.go` 改用 `h.WriteModelListResponse`），**均可自动合并**（白名单过滤分支与上游新 API 共存）。旧基线补丁存为 `cpa-whitelist.patch.bak-pre-v72159`。
>
> ⚠️ **v7.2.159 上游自带一个失败测试**：`TestOpenAICompatExecutorToolResultContentByInputModalities`（4 个子用例）在**未打补丁的干净基线**上同样失败（已做对照实验），属上游测试与实现不同步，**非本补丁引入**，本项目不代为修复。
>
> 2026-09-11 升级基线 **v7.2.145 → v7.2.157**。本次升级本身修复了 ZCode 报 `Model request failed / text part msg_…_0 not found` 的根因：上游 v7.2.146（`6c6473f8`）把 Responses 转换器的空 `tool_calls` 数组判断从 `tcs.IsArray()` 改为 `tcs.IsArray() && len(tcs.Array()) > 0`，此前空数组会在第一个正文 delta 后就误发 `response.output_item.done`，随后继续发 500+ delta 导致客户端判为「text part not found」。**该修复属上游代码，不在本补丁内。**
>
> 2026-08-29 起补丁在 whitelist 之外新增 **OpenAI 消息归一化**（`openai_openai_request.go` 的 `normalizeOpenAIChatMessages`：developer→system、thinking 块→reasoning_content、补空 reasoning_content），解决 b.ai 等严格上游的 400。旧版（仅白名单）存为 `cpa-whitelist.patch.bak-pre-norm1`。
>
> 2026-08-30 起补丁再新增 **OpenAI 流式收尾修复**（`openai_compat_executor.go` 合成缺失的 `finish_reason` 终止 chunk，解决 pi-ai `Stream ended without finish_reason`）。
>
> ⚠️ **norm2 曾漏出补丁**：该修复部署于 2026-08-30，但直到 2026-09-11 才补进 `cpa-whitelist.patch`（旧补丁 16 文件、`grep openai_compat_executor` = 0）。期间任何按旧流程重放补丁再构建镜像的操作都会**静默丢掉 norm2**。旧补丁存为 `cpa-whitelist.patch.bak-16file-pre-upgrade-v72157` 留作教训对照。
>
> 2026-09-14 补丁再新增 **GPT-6 家族自动 Codex 身份头**（4 文件：`codex_executor_request.go` 新增 `isGPT6FamilyModel` / `isOfficialCodexBaseURL` / `hasOperatorHeader` / `applyGPT6CodexIdentityHeaders`，在 `codex_executor_execute.go`（流式 + `/responses/compact`）与 `codex_executor_stream.go` 调用；`gpt6_codex_identity_test.go` **新增** 8 例回归测试）。背景：anyrouter 这类转发站按 `Originator` 分流，`codex_exec` 身份可用、`codex-tui`（CPA 默认 cloaking 值）报 `400 invalid codex request`。此前靠每个 provider 手写 `headers` + 关 `disable-codex-cloaking` 解决，装到别人机器上容易漏配。自动规则只在**非官方上游**（`base-url` 不含 `chatgpt.com` / `openai.com`）生效，且让位于 provider `headers` 与 `models.json` 的 `config.override_header`。18→22 文件，验证：干净 v7.2.159 worktree `git apply --check` plain 通过 + `go build ./...` 0 + `-run GPT6` 8 例 PASS（上游自带的 `TestOpenAICompatExecutorToolResultContentByInputModalities` 失败已用 0 改动干净 worktree 对照确认，非本补丁引入）。
| `cpamp-whitelist.patch` | `seakee/CPA-Manager-Plus` **v1.14.1** | 9（2 个新文件） | 干净 v1.14.1 worktree 上 `git apply --check`（plain）+ 9/9 逐字节一致 + `tsc --noEmit` + `vitest`(4174) + 仓库级测试(257) + `vite build` |

> ⚠️ **补丁基线已前移到 CPA v7.3.18 / CPAMP v1.14.1**（2026-09-26）。旧基线补丁分别留作
> `cpa-whitelist.patch.bak-pre-v7318`（v7.2.159 基线，22 文件）与 `cpamp-whitelist.patch.bak-pre-v1141`
> （v1.12.5 基线，9 文件）。升级时上游 v7.2.159→v7.3.18 与 CPAMP v1.12.5→v1.14.1 的改动里，
> CPA 侧与本补丁重叠的 3 个 codex 文件需要手工合并（已合并并在下方「文件」表反映）；CPAMP 侧按新架构重写。

## 升级上游时的重放步骤

```bash
# CPA
cd src/cli-proxy-api
git fetch --tags origin
git checkout --detach v<新版本>
git clean -fd                      # 关键：清掉上一次补丁留下的未跟踪文件
git apply --3way ../../patches/cpa-whitelist.patch
go build ./... && go vet ./... && go test ./... -race -count=1

# CPAMP
cd src/cpa-manager-plus
git fetch --tags origin
git checkout --detach v<新版本>
git clean -fd
git apply ../../patches/cpamp-whitelist.patch
npm ci && npm run type-check && npm --workspace apps/web run build
```

`git apply` 失败时先试 `git apply --3way`；仍失败则手工移植，改完**按下面的规则重新生成补丁**。

## 重新生成补丁（务必照做，别用裸 git diff）

```bash
git add -A                 # 必须先跑：纳入新文件，也把未暂存改动并入 index
git diff HEAD > ../../patches/<name>.patch
```

**必须先 `git add -A`**。本补丁包含上游不存在的新文件（`sdk/access/apikey_metadata.go`、
`internal/api/handlers/management/models.go`、以及 4 个测试文件）。裸 `git diff` 只输出
**已跟踪且已暂存**的改动，会静默丢掉未跟踪的新文件与未暂存的改动——这两点都踩过坑：

> **坑一（新文件被丢）**：旧版 `cpa-whitelist.patch` 用裸 `git diff` 导出，漏掉了
> `sdk/api/handlers/handlers_routing.go` 里的 `enforceModelWhitelist` 方法定义
> （该文件上游已存在、但改动是在非 git 目录里做的，从未被导出）。结果补丁打上后
> `go build` 直接失败：`h.enforceModelWhitelist undefined (type *BaseAPIHandler has no field
> or method ...)`，备份无法重建生产镜像。2026-08-29 已修复并补齐。
>
> **坑二（未暂存改动被丢）**：2026-08-30 的流式收尾修复（norm2，
> `internal/runtime/executor/openai_compat_executor.go`）与它的回归测试
> `openai_compat_executor_finish_test.go` 都没进补丁，而**补丁在改动落地后从未重新导出**，
> 于是补丁长期停留在 16 文件、静默缺少 norm2。2026-09-11 升级到 v7.2.157 时才补齐为 18 文件。

生成后**必须跑下面的完整性校验**，能编译且文件数符合预期才算补丁完整：

```bash
# 1) 文件数应等于预期（CPA 当前 30 / CPAMP 当前 9）
grep -c '^diff --git' ../../patches/cpa-whitelist.patch     # 期望 30
grep -c '^diff --git' ../../patches/cpamp-whitelist.patch   # 期望 9

# 2) 关键修复不得缺失（白名单 / norm1 / norm2 / gpt6 / 思考档位 / 新端点 / prune）
grep -c normalizeOpenAIChatMessages ../../patches/cpa-whitelist.patch      # 期望 >= 1
grep -c synthesizeOpenAIStreamFinish ../../patches/cpa-whitelist.patch     # 期望 >= 1
grep -c applyGPT6CodexIdentityHeaders ../../patches/cpa-whitelist.patch    # 期望 >= 1
grep -c GetAPIKeyModels ../../patches/cpa-whitelist.patch                  # 期望 >= 1
grep -c 'pruneMappingToGeneratedKeys.*api-key-models' ../../patches/cpa-whitelist.patch  # 期望 >= 1
grep -c apiKeyModelsApi ../../patches/cpamp-whitelist.patch                # 期望 >= 1

# 3) 在干净的目标 tag 上真实回放一次（最能发现问题）
git worktree add /tmp/patchcheck --detach v<目标版本>
cd /tmp/patchcheck && git apply --check ../../patches/cpa-whitelist.patch \
  && go build ./... && go test ./... -race -count=1
```

> 经验：只做 `git apply --check` 不够——它只验证补丁能贴上，**不验证补丁内容是否完整**。
> 补丁漏文件时 `apply --check` 照样通过。必须加上「文件数 + 关键字 + 能编译」三重校验。
> 另：回放后建议再逐文件 `cmp` 与工作树对比（本次升级就是这么做的：30/30 与 9/9 逐字节一致）。

## cpa-whitelist.patch（CPA 后端）

| 文件 | 改动 |
|---|---|
| `internal/config/sdk_config.go` | 新增 `APIKeyModels map[string][]string`，yaml 字段 `api-key-models` |
| `internal/access/config_access/provider.go` | 认证时把 key 的白名单放进 `Result.Metadata["model-whitelist"]` |
| `internal/api/server_middleware.go` | 把白名单写入请求 context（`sdkaccess.WithAuthMetadata`） |
| `sdk/access/apikey_metadata.go` | **新增**：context key、`WithAuthMetadata`、`ModelWhitelistFromContext`、`MetadataFromContext` |
| `sdk/api/handlers/handlers_routing.go` | `enforceModelWhitelist`：模型不在白名单 → `403 model_not_allowed` |
| `sdk/api/handlers/handlers.go` | 认证 context 传递 + 路由前检查 |
| `sdk/api/handlers/handlers_execution.go` | 非流式执行前调 `enforceModelWhitelist` |
| `sdk/api/handlers/handlers_stream.go` | 流式执行前调 `enforceModelWhitelist` |
| `sdk/api/handlers/openai/openai_handlers.go` | `GET /v1/models` 按白名单过滤，受限 key 只看到自己的模型 |
| `internal/api/handlers/management/models.go` | **新增**：`GET /v0/management/models` 返回**全量**模型目录（不受白名单影响） |
| `internal/api/handlers/management/config_api_key_models.go` | **新增（v7.3.18）**：`GET/PUT/PATCH/DELETE /v0/management/api-key-models` —— 白名单的读/整表替换/单条声明式同步/单条删除；统一 trim+去重、空列表视为「无限制」（不写空映射），每次变更走 `h.persist` 触发同一条 reload 链路 |
| `internal/api/handlers/management/config_api_key_models_test.go` | **新增（v7.3.18）**：10 例（GET 空对象 / PATCH trim+去重+落盘 / 空列表删除 / 未知 key 幂等 / 缺 key 400 / PUT 整表替换与拖尾丢弃 / DELETE 未知 404 / 中文模型名原样往返） |
| `internal/config/config_yaml.go` | **prune 修复（v7.3.18）**：`SaveConfigPreserveComments` 的 prune 列表补上 `api-key-models`（上游原本只 prune `oauth-excluded-models` / `oauth-model-alias` / `oauth-request-scoped-errors`）——否则经管理端点删除的白名单条目只改内存，文件里的旧键会在下次 reload 复活 |
| `internal/config/config_api_key_models_save_test.go` | **新增（v7.3.18）**：2 例（删除条目后文件里真的消失、清空后整个映射消失）；**去掉修复即失败**已验证 |
| `sdk/cliproxy/service_models.go` | **思考档位默认值（v7.3.18）**：`buildOpenAICompatibilityConfigModels` 未声明 `thinking:` 时注入的默认档位由 `[low,medium,high]` 扩为 `[low,medium,high,xhigh,max]` |
| `sdk/cliproxy/auth/api_key_model_capabilities.go` | **同上**：`compileOpenAICompatibleModelCapabilities` 同一个默认档位表 |
| `sdk/cliproxy/openai_compat_thinking_defaults_test.go` | **新增（v7.3.18）**：默认档位含 max/xhigh + 模型显式 `thinking.levels` 仍优先 |
| `sdk/cliproxy/auth/openai_compat_thinking_defaults_test.go` | **新增（v7.3.18）**：能力编译路径同上断言 |
| `internal/api/server_management.go` | 注册上述 management 路由（`/models` + `/api-key-models` 四方法） |
| `sdk/api/handlers/handlers_model_whitelist_test.go` | **新增**：403 拦截回归测试（放行/拒绝/不受限/空白名单/nil 边界） |
| `sdk/api/handlers/openai/openai_models_whitelist_test.go` | **新增**：`/v1/models` 过滤回归测试 |
| `internal/api/handlers/management/models_test.go` | **新增**：全量端点不受白名单影响 + 字段投影测试 |
| `internal/translator/openai/openai/chat-completions/openai_openai_request.go` | **消息归一化**：转发 openai-compat 上游前 developer→system、assistant thinking 块抽成 reasoning_content、每条 assistant 补 reasoning_content（DeepSeek thinking 模式要求），纯字符串消息零拷贝原样返回 |
| `internal/translator/openai/openai/chat-completions/openai_openai_request_test.go` | **新增**：归一化回归测试 5 例（developer→system / thinking 折叠 / 补空 rc / user image 不动 / 零拷贝保持） |
| `internal/runtime/executor/openai_compat_executor.go` | **流式收尾修复（norm2）**：跟踪上游是否发过非空 `finish_reason` / 是否推过 `tool_calls` delta / 是否有过 choice 内容；流干净结束（`[DONE]` 或 EOF 补发 `[DONE]`）且「有内容但缺收尾帧」时，在 `[DONE]` 之前合成标准 `chat.completion.chunk`（`finish_reason` 取 `tool_calls`/`stop`，复用上游 id/model），解决 pi-ai `Stream ended without finish_reason`。新增 `openAIStreamTerminalInfo` / `synthesizeOpenAIStreamFinish` 辅助函数 |
| `internal/runtime/executor/openai_compat_executor_finish_test.go` | **新增**：收尾合成回归测试 5 例（缺收尾补 stop / tool_calls 补 tool_calls / 已有收尾不重复 / EOF 无 DONE 也补 / 空流不补） |
| `internal/runtime/executor/codex_executor_request.go` | **GPT-6 自动身份头（gpt6）**：新增 `isGPT6FamilyModel`（`gpt-6` / `gpt-6-*` / `gpt-6.*`，含 `team/gpt-6-astra` 前缀与思考后缀）、`isOfficialCodexBaseURL`、`hasOperatorHeader`、`applyGPT6CodexIdentityHeaders` —— 命中 GPT-6 家族且上游非官方时强制 `Originator: codex_exec` + `codex_exec` UA（缺失时补 `Session_id`），让位于 provider `headers` 与 `config.override_header` |
| `internal/runtime/executor/codex_executor_execute.go` | 流式与 `/responses/compact` 两条路径在 `applyCodexHeaders` 之后调用 `applyGPT6CodexIdentityHeaders` |
| `internal/runtime/executor/codex_executor_stream.go` | 流式执行路径同步调用 `applyGPT6CodexIdentityHeaders` |
| `internal/runtime/executor/gpt6_codex_identity_test.go` | **新增**：GPT-6 身份头回归测试 8 例（家族识别 / 前缀与后缀名 / 官方后端不干预 / 覆盖 cloaking / 丢弃客户端 `Originator` / 让位于 provider headers / 让位于 `config.override_header` / 空输入不 panic） |

> norm2 与上游 v7.2.157 无冲突：该版本 executor 里 `finish_reason` 出现 0 次，上游新增的是 Responses 格式下的「缺 `[DONE]` 即判失败」，与 norm2 的「补 `finish_reason` 终止帧」互补，因此 norm2 **未过时、仍需保留**（upstream 仅在 claude/gemini 翻译器里各自做了等价兜底，不覆盖 openai→openai）。

### 为什么需要 `/v0/management/models`

面板配置白名单时要列出候选模型。旧实现用 `api-keys[0]` 去探 `/v1/models`，而 `/v1/models`
是**按 key 过滤**的——一旦第一个 key 恰好是受限 key，面板就只能看到它那十几个模型，
无法给正在编辑的 key 授予其它模型。新端点直接读 registry，永远返回全量；CPAMP 的
`ProxyManagement` 对 `/v0/management/*` 是通用透传（只校验路径前缀、无路径白名单），
所以**无需改 nginx、无需改 CPAMP 后端**。

## cpamp-whitelist.patch（CPAMP 前端）

基线 **v1.14.1**。9 个文件，其中 2 个是新增。

| 文件 | 改动 |
|---|---|
| `apps/web/src/services/api/apiKeyModels.ts` | **新增**：`apiKeyModelsApi` —— `list()`（读白名单映射，严格校验响应形状）、`setForKey(key, models)`（声明式 PATCH，空列表=取消限制）、`deleteForKey(key)`、`listModelCatalogue()`（走 `GET /v0/management/models` 拿**全量**目录）、`probeClientModels(key)`（回退探 `/v1/models`，调用方需标注「列表可能不完整」） |
| `apps/web/src/services/api/apiKeyModels.test.ts` | **新增**：10 例（空/驼峰字段/缺字段/非数组拒绝 / trim 与空条目丢弃 / PATCH 载荷 / 空列表走 PATCH 而非 DELETE / key URL 编码 / 全量目录去重投影 / 畸形载荷空列表） |
| `apps/web/src/services/api/index.ts` | 导出新服务 |
| `apps/web/src/components/config/ApiKeysCardEditor.tsx` | 模型白名单多选 UI（搜索 / 全选可见 / 清空 / 计数 / 加载按钮）；挂载时读白名单映射并在编辑弹窗预勾选；保存时紧接 `onPersistApiKeyMutation` 之后持久化白名单（改名时同步丢弃旧 key 条目），失败则是「key 已存但白名单未存」的 partial-success 告警；删除 key 后同步清理其白名单条目；候选列表优先 management 全量端点，回退探针并提示不完整 |
| `apps/web/src/components/config/ApiKeysCardEditor.test.tsx` | 在既有 31 例之外新增 8 例（预勾选 / 一步保存 / 清空 / 改名迁移 / 删除清理 / partial success / 回退提示 / 别名未变时不误写） |
| `apps/web/src/i18n/locales/{en,zh-CN,zh-TW,ru}.json` | 白名单与提示文案 14 个 key（**四个 locale 键集完全一致**，有 `ConfigPage.test.ts` 守卫） |

> **与旧版（v1.12.5 基线）补丁的区别**：旧版依赖「可视化配置一步提交」链（`types/visualConfig.ts` 的
> `apiKeyModelsText`、`useVisualConfig.ts` 的解析/序列化、`VisualConfigEditor.tsx` 的 props 透传、
> `ConfigPage.tsx` 的 `commitVisualChangesNow`，以及 `useVisualConfigApiKeyWhitelist.test.ts`），
> 上游 v1.14.1 把 API key 改成管理端点即时落盘后这条链已不成立——现在白名单自己走
> `/v0/management/api-key-models`，**上述 5 个文件不再需要改动**，补丁面更小、与上游的耦合也更低。

### 一步保存（F-04）

在 API Key 弹窗里加 key + 勾模型，点弹窗「保存」即可：`api-keys` 走上游自己的
`onPersistApiKeyMutation`（管理端点），`api-key-models` 紧跟着走 CPA 新端点，两者都是即时落盘，
不需要再点配置页顶部的「保存配置」。

⚠️ **顺序要求**：CPA 必须先升到带 `/v0/management/api-key-models` 的版本（本补丁 v7.3.18），
否则面板侧的读/写会 404。部署时先切 CPA、再切 CPAMP。

## 升级 CPAMP 的注意事项（v1.12.5 → v1.14.1 实测）

v1.12.5 → v1.14.1 跨了 616 个上游提交，本补丁触及的 `ApiKeysCardEditor.tsx` 被上游大改
（+408/-137，改成管理端点即时落盘），四个 locale 文件也都有改动。实测：`git apply --3way`
在 `ApiKeysCardEditor.tsx` 上有 7 个冲突块，其余文件可自动合并；本次是按新架构重写该文件。

迁移风险已实测可控：v1.12.6+ 会把 legacy 的 `CPA_UPSTREAM_URL` + `CPA_MANAGEMENT_KEY_FILE`
迁移进加密 SQLite，且 fail-closed。**升级前必须把 `usage.sqlite` + `-wal` + `-shm` + `data.key`
当一整套文件集备份**；本次用「卷副本起 canary」的方式在生产前先跑通迁移（实测：迁移与派生表
重建全部完成，无报错）。

另有一个已观察到的安全行为：两个 CPAMP 实例指向同一 `usage.sqlite` 时，后启动的那个会
以 `manager database process lock is already held` 退出（`Exited (1)`）——这是有意的进程锁，
不是缺陷；起 canary 时请各用一份数据副本。

## 备份位置

- `src/cli-proxy-api/` — **v7.3.18** + 本补丁（30 文件），与生产构建逐文件一致
- `src/cpa-manager-plus/` — **v1.14.1** + 本补丁（9 文件），与生产构建逐文件一致
- `src/backup/cpa-src-whitelist-full-v7.2.145-norm2.tar.gz` — 升级前 v7.2.145 完整源码（含白名单+norm1+norm2，历史锚点）
- `src/backup/cpa-src-whitelist-full-v7.2.143.tar.gz` — 生产镜像 `:whitelist` 的原始完整源码（更早锚点）
- `patches/cpa-whitelist.patch.bak-pre-v7318` — v7.2.159 基线版补丁（22 文件，包括 thinking/api-key-models/prune 三组新改动之前的状态）
- `patches/cpa-whitelist.patch.bak-26file-pre-apikeymodels` — 本次升级中途态：已是 v7.3.18 基线 26 文件、但还没加 api-key-models 端点与 prune 修复
- `patches/cpamp-whitelist.patch.bak-pre-v1141` — v1.12.5 基线版前端补丁（9 文件，旧的可视化配置一步提交架构）
- `patches/cpa-whitelist.patch.bak-pre-v72159` — v7.2.157 基线版补丁（基线迁移前，内容等价但上下文对齐旧 tag）
- `patches/cpa-whitelist.patch.bak-16file-pre-upgrade-v72157` — 缺 norm2 的 16 文件旧补丁（教训对照）
- `patches/cpa-whitelist.patch.bak-incomplete` — 更早的残缺补丁（教训对照）
