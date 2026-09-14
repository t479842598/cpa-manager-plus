# Patches（自研 per-key 模型白名单补丁）

上游 CPA / CPAMP 均不支持「某个 API key 只能访问部分模型」，本目录的两个补丁实现了该功能的完整链路。

## 当前基线

| 补丁 | 上游基线 | 文件数 | 校验状态 |
|---|---|---|---|
| `cpa-whitelist.patch` | `router-for-me/CLIProxyAPI` **v7.2.159** | 22（白名单 14 + 消息归一化 2 + 流式收尾修复 2 + GPT-6 自动身份头 4） | 干净 v7.2.159 worktree 上 `git apply --check`（plain，无需 `--3way`）+ 重放 `go build ./...` 退出码 0 + `-run GPT6` 8 例全过；原基线 v7.2.157 下曾 `go vet` + `go test -race` 全通过 |

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
| `cpamp-whitelist.patch` | `seakee/CPA-Manager-Plus` **v1.12.5** | 9（含 1 个测试文件） | `git apply --check` + `tsc --noEmit` + `vitest`(2533) + `vite build` 全通过 |

> ⚠️ **CPAMP 基线仍是 v1.12.5**（生产未升级到 v1.12.6）。升级到 v1.12.6 时本补丁需要重放，见文末。

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
# 1) 文件数应等于预期（CPA 当前 22）
grep -c '^diff --git' ../../patches/cpa-whitelist.patch

# 2) 关键修复不得缺失（norm1 / norm2 / gpt6 各自的存在性）
grep -c normalizeOpenAIChatMessages ../../patches/cpa-whitelist.patch   # 期望 >= 1
grep -c synthesizeOpenAIStreamFinish ../../patches/cpa-whitelist.patch  # 期望 >= 1
grep -c applyGPT6CodexIdentityHeaders ../../patches/cpa-whitelist.patch # 期望 >= 1

# 3) 在干净的目标 tag 上真实回放一次（最能发现问题）
git worktree add /tmp/patchcheck --detach v<目标版本>
cd /tmp/patchcheck && git apply --check ../../patches/cpa-whitelist.patch \
  && go build ./... && go test ./... -race -count=1
```

> 经验：只做 `git apply --check` 不够——它只验证补丁能贴上，**不验证补丁内容是否完整**。
> 补丁漏文件时 `apply --check` 照样通过。必须加上「文件数 + 关键字 + 能编译」三重校验。

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
| `internal/api/server_management.go` | 注册上述 management 路由 |
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

| 文件 | 改动 |
|---|---|
| `apps/web/src/types/visualConfig.ts` | 新增 `apiKeyModelsText` 字段 |
| `apps/web/src/hooks/useVisualConfig.ts` | `api-key-models` 的解析与序列化（`key=m1,m2` 行格式） |
| `apps/web/src/components/config/ApiKeysCardEditor.tsx` | 模型白名单多选（搜索/全选/清空）；候选改走 management 端点并带回退提示；保存后请求一步落盘 |
| `apps/web/src/components/config/VisualConfigEditor.tsx` | 透传 `modelsText` 与 `onRequestCommit` |
| `apps/web/src/features/config/ConfigPage.tsx` | `commitVisualChangesNow`：跳过 diff 预览直接写 `config.yaml` 并回读刷新基线 |
| `apps/web/src/hooks/useVisualConfigApiKeyWhitelist.test.ts` | **新增**：白名单 YAML 往返 / 一步写入 / 清空删除 / 畸形行忽略 共 5 例 |
| `apps/web/src/i18n/locales/{en,zh-CN,zh-TW}.json` | 白名单与提示文案（zh-TW 已补齐此前缺失的 11 个 key） |

### 一步保存

在 API Key 弹窗里加 key + 勾模型，点弹窗「保存」即写入 `config.yaml` 并生效，
不需要再去点配置页顶部的「保存配置」。实现方式：弹窗保存后回调
`onRequestCommit()`，`ConfigPage` 在同一个 effect 里（此时 visual state 已提交）
执行 `fetchConfigYaml → applyVisualChangesToYaml → saveConfigYaml → 回读`，
并用 ref 防止重入。

副作用需知：这一步会连同配置页上**其它已改未保存**的可视化字段一起落盘
（与顶部「保存」的语义一致）。

## 升级 CPAMP 到 v1.12.6 的注意事项

v1.12.6 改动了本补丁触及的 `ConfigPage.tsx` 与三个 locale 文件。实测：
`git apply` 直接打会在 locale 上报错，`--3way` 可干净应用，`tsc` 与 `vite build` 均通过。
当前补丁已消除「文件末尾换行」这一伪冲突源，重放阻力比早期小。

但 v1.12.6 会把 legacy 的 `CPA_UPSTREAM_URL` + `CPA_MANAGEMENT_KEY_FILE` 迁移进加密
SQLite，且带 fail-closed 逻辑，升级前必须把 `usage.sqlite` + `-wal` + `-shm` + `data.key`
当**一整套文件集**备份。

## 备份位置

- `src/cli-proxy-api/` — **v7.2.159** + 本补丁（22 文件），与生产构建逐文件一致
- `src/cpa-manager-plus/` — v1.12.5 + 本补丁，与生产构建逐文件一致
- `src/backup/cpa-src-whitelist-full-v7.2.145-norm2.tar.gz` — 升级前 v7.2.145 完整源码（含白名单+norm1+norm2，历史锚点）
- `src/backup/cpa-src-whitelist-full-v7.2.143.tar.gz` — 生产镜像 `:whitelist` 的原始完整源码（更早锚点）
- `patches/cpa-whitelist.patch.bak-pre-v72159` — v7.2.157 基线版补丁（基线迁移前，内容等价但上下文对齐旧 tag）
- `patches/cpa-whitelist.patch.bak-16file-pre-upgrade-v72157` — 缺 norm2 的 16 文件旧补丁（教训对照）
- `patches/cpa-whitelist.patch.bak-incomplete` — 更早的残缺补丁（教训对照）
