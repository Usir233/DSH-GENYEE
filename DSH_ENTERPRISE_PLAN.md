# DSH 企业版改造工作文档

> 用途：交给 Claude Code，作为在 `deepseek-ai/deepseek-harness` 仓库上讨论与实施改造的起点。
> 基线：主干 `@deepseek-ai/dsh-root` `0.2.2-alpha.2`（2026-10-28 拉取）。上游仍是开发者预览版，开始工作前先 `git pull` 并核对下文路径是否仍然存在。

---

## 1. 背景与目标

**我们是谁**：一家顾问咨询公司的开发团队。

**要做什么**：基于 DeepSeek Harness（下称 DSH）做一个可交付给企业客户的 agent 工具，配套 Tencent WeKnora 作为知识库，配套一个 AI 网关做密钥管理。

**交付模式**：每个客户一套**私有化部署**，部署在客户内网，不做 SaaS 多租户。

**核心问题**：很多客户没有开通付费 agent 产品，也没有统一的模型采购与管控。我们要给他们：
1. 员工可用的桌面 agent（基于 DSH 官方桌面端 `apps/desktop`）
2. 集中的模型密钥管理：客户自有厂商 Key（BYOK）只存在网关里，员工只拿虚拟 Key
3. 管理员可强制执行的策略：员工本机无法绕过
4. 审计与用量可追溯，数据不外发给 DeepSeek

**非目标**：
- 不深度 fork DSH 核心包；能用插件解决的绝不改上游代码
- 不在本仓库实现 AI 网关本体（网关是独立服务，可基于 LiteLLM Proxy / One API / New API 二次开发）
- 不改 WeKnora 源码，原样部署 + 官方 `@wxg-prc-cpg/dsh-weknora` 插件接入

---

## 2. 目标架构

```
客户内网
├── 员工桌面端（DSH 薄壳 fork + 企业组合包）
│     └─ 只持有短期虚拟 Key，经 SSO 换取
├── AI 网关（独立服务，不在本仓库）
│     └─ 真实厂商 Key（BYOK）、虚拟 Key、额度、审计、模型路由
├── WeKnora 知识库（原版部署）
├── 企业 SSO（企业微信 / 飞书 / LDAP / OIDC）
└── 本地模型（Ollama / vLLM，敏感数据专用路由）

外部：模型厂商 API（客户自付）、我们的发布仓库（交付包 + 版本矩阵）
```

---

## 3. 改造原则

DSH 是 Cordis "万物皆插件"架构。profile 由组合包（bundle）的 patch 层叠加而成，树外组合包可通过 `dsh plugin --profile <name> add <package>` 安装。

改动按成本分四层，**永远优先选更低的层**：

| 层 | 内容 | 是否改上游代码 |
|---|---|---|
| L1 配置 | 网关路由、关闭/重定向遥测 | 否 |
| L2 企业组合包 | 凭据提供方、策略、审计、预装插件 | 否（独立 npm 包） |
| L3 桌面薄壳 fork | 品牌、更新源、登录页、预置 profile | 是，仅 `apps/desktop` |
| L4 上游 PR | 管理员锁定配置层 | 是，争取合入上游 |

---

## 4. 已核实的事实（附路径）

### 4.1 配置 patch 的应用顺序 —— 关键安全缺口

`packages/boot/app-boot/src/profile-context.ts` 中 `readProfilePatches()` 的顺序：

1. 组合包 patch（`readBundlePatches`）
2. profile 自己的 `cordis.patch.yml`
3. **home 级 `~/.dsh/cordis.patch.yml`**（用户可写）
4. 命令行 overlays（`context.overlays`）
5. 遥测 patch（由 `DSH_TELEMETRY_DISABLED` 决定）

**结论**：我们的企业组合包在第 1 层，员工在第 3 层写一个 patch 就能覆盖企业策略。目前没找到管理员只读/锁定层。→ 见工作流 L4。

### 4.2 模型路由可直接接自建网关

`packages/llm/llm-pi-ai/README.zh.md`：`providers` 字典支持手工声明的 OpenAI 兼容网关（`api`、`baseURL`、`models`、`compat`）。`apiKeyEnv` 是凭据引用名，按请求经凭据 seam 解析，配置文件不含密钥。设置变更下一次请求生效，无需重启。

### 4.3 凭据是可替换的 seam

- `packages/credentials/credentials/src/types.ts`：`CredentialRef`、`CredentialKey`、`ApiKeyRecord`、`GrantRecord`、`CredentialInfo`
- `packages/credentials/credentials-local/`：默认实现，本机私有 YAML，优先于环境变量
- `packages/credentials/authorization/`：插件拥有的"向人询问以获取凭据"流程
- `packages/credentials/deepseek-account*`：DeepSeek 账号登录，企业版应不加载
- 子系统参考：`docs/subsystems/credentials.zh.md`

### 4.4 审批与权限

- `packages/interaction/user-approval/src/index.ts`：`ApprovalPolicy = 'ask' | 'never'`；应答者（answerer）组合成瀑布，无应答或抛错一律 fail closed
- `packages/interaction/permission-presets/`：把沙箱模式与审批策略捆成一个用户可选预设
- 子系统参考：`docs/subsystems/approval.zh.md`、`docs/subsystems/permission-presets.zh.md`

### 4.5 遥测与外发数据

- `packages/client/product-analytics/`：桌面端默认开启产品埋点（OS、CPU、内存、模型选择、插件开关等；不含提示词、回复、API key）。`product-analytics.enabled` 可关，`DSH_PRODUCT_ANALYTICS_OTLP_URL` 可重定向
- `DSH_TELEMETRY_DISABLED`：任意非空值退出遥测（见 4.1）
- `packages/llm/plugin-package-inventory-deepseek/`：向**官方 DeepSeek 请求**附带启用的插件包清单
- 桌面端还有 `crash-report.ts`、`operation-report*.ts`，**需要审查它们发往哪里**
- 子系统参考：`docs/subsystems/otel.zh.md`

### 4.6 桌面壳中写死、必须 fork 的部分（`apps/desktop/src/`）

| 关注点 | 文件 |
|---|---|
| 更新通道与检查 | `update-channel.ts`、`update-coordinator.ts` |
| 强制更新策略 | `mandatory-update-policy.ts`、`mandatory-update-window.ts` |
| 欢迎页/登录 | `welcome-window.ts`、`welcome-backend.ts`、`welcome-api.ts`、`client/welcome.tsx` |
| 账号后端 | `account-backend.ts` |
| 崩溃与操作上报 | `crash-report.ts`、`operation-report*.ts` |
| 图标与品牌 | `resources/icon*.{png,svg}`、`resources/tray-windows.ico`、`installer/assets/` |
| 打包与签名 | `apps/desktop/windows-signing/README.zh.md`、`apps/desktop/publishing/` |

### 4.7 组合包的版本约束

`packages/boot/app-boot/README.zh.md`：组合包可在 `package.json` 声明 `engines.dsh` 版本范围，不兼容时启动器跳过该组合包并列入 `skippedBundles`。可用 `dsh.bundle.platforms` 限定平台。这就是我们版本矩阵的落地手段。

### 4.8 品牌约束

`BRAND_GUIDELINES.zh.md`："DeepSeek Harness" 是深度求索的注册商标，项目名不得直接使用；建议用 "DSH" 缩写命名；不得让人误以为有官方背书。→ 产品可命名为 "DSH Enterprise" 一类，图标全部替换。

### 4.9 许可证

DSH 与 WeKnora 均为 MIT。保留上游 `LICENSE` 与 `THIRD_PARTY_NOTICES.md`。

### 4.10 DeepSeek 官方《限速与隔离》机制与原生扩展插槽核查（2026-10-28 补充）

参考官方文档：[限速与隔离 | DeepSeek API Docs](https://api-docs.deepseek.com/zh-cn/quick_start/rate_limit)，结合仓库源码核查得出以下核心事实：

1. **账号级并发与 `user_id` 多租户隔离**：
   - DeepSeek 并发按**账号粒度**计算（与创建多少个 API Key 无关），默认 `deepseek-flash` 2500 并发、`deepseek-v4-pro` 500 并发，企业可提工单免费扩容。
   - 同一企业主账号下，通过传递 `user_id`（需满足正则 `[a-zA-Z0-9\-_]+`，最大长度 512，不含明文隐私，如 `emp_<hash>`）由 DeepSeek 官方服务端直接完成：
     - **KVCache 隔离**：不同员工之间实现 KVCache 隐私管理隔离；
     - **内容安全隔离**：区分企业内不同员工身份，避免单人违规导致全公司账号连坐；
     - **调度与单用户并发隔离**：即使企业扩容了账号总并发，DeepSeek 仍限制单个 `user_id`（含**空 `user_id`**）上限为 `deepseek-flash` 2500 / `deepseek-v4-pro` 500。若不传 `user_id`，全公司员工将挤在同一个“空 `user_id`”桶内触发 HTTP 429。
2. **DSH 原生 DeepSeek 协议与扩展插槽（无需改核心包）**：
   - **原生协议**：DSH 调用 DeepSeek 官方模型的一等公民适配器是 `@deepseek-ai/dsh-llm-deepseek`（配合 `@deepseek-ai/dsh-llm-deepseek-api-key`），底层使用 **`https://api.deepseek.com/anthropic`（Anthropic Messages `/v1/messages` + `/v1/files` 协议）**，而非 `llm-pi-ai` 的 OpenAI 协议。这样才能保留对话中途系统提示词增量更新（`systemPromptUpdate: in-history`）、中途工具动态增删不破 KV Cache（`mid-conversation-tool-changes-2026-07-01`）和思考签名回放。
   - **免填 Key 凭据插槽**：`packages/llm/llm-deepseek-api-key/src/index.ts` 在每次请求时调用 `ctx.credentials.resolve(ref)`。企业插件接管 `ctx.credentials` 后，员工端无需在界面手动填写任何 API Key。
   - **自动注入 `metadata.user_id` 插槽**：`packages/llm/deepseek-llm-api-extensions/src/index.ts` 提供 `ctx.deepseekLlmApiExtensions.register(field, provider)`，且 `packages/llm/llm-deepseek/src/wire-types.ts` 的 `WireRequest` 默认不含 `metadata` 字段。企业插件只需注册：
     ```ts
     ctx.deepseekLlmApiExtensions.register('metadata', {
       prepare: () => ({ value: { user_id: currentEmployeeId } }),
     })
     ```
     即可在每次发往 DeepSeek 的请求体顶层自动附加 `"metadata": { "user_id": "emp_xxx" }`。
3. **10 分钟 `: keep-alive` 保活机制**：
   - DeepSeek 流式请求排队时持续返回 SSE 注释 `: keep-alive`（最长 10 分钟）。
   - `packages/llm/llm-deepseek/src/sse.ts` 已通过 `EventSourceParserStream({ onComment: activity })` 原生支持心跳续期；若经过任何中间转发层，上游 `read_timeout` 必须 `>= 600s` 且原样透传 SSE 注释行（关闭缓冲）。

---

## 5. 核心方案演进：轻量级「企业管理员 Key 授权管控插件」（替代重型第三方网关）

基于 4.10 的事实，既然 DeepSeek 官方 API 已原生支持账号级高并发与 `user_id` 隔离，我们可以**省去额外部署重型第三方 AI 网关（如 LiteLLM / OneAPI）**，直接编写一个 **DSH 企业管控插件**（分「管理员管控端」与「员工客户端」），实现员工免填 Key、管理员集中授权/断权、请求高速直达。

以下两种落地模式待进一步权衡选择：

### 模式 A：纯控制面授权下发（员工端内存驻留 Key + 直连 DeepSeek 官方）
- **运行机制**：
  1. **管理员端插件**：保管企业真实的 DeepSeek API Key，维护员工授权白名单与 `user_id` 映射。
  2. **员工端插件**：启动时向管理员端认证获取授权，将 Key **仅保存在内存中**（通过 `ctx.credentials` 提供给 `llm-deepseek-api-key`，绝不落盘到 `~/.dsh/`，UI 上也不展示），并通过 `ctx.deepseekLlmApiExtensions` 自动注入该员工的 `"metadata": { "user_id": "emp_xxx" }`。
  3. **断权机制**：员工端插件定时/每次会话前向管理员端做轻量心跳校验；管理员在后台禁用某员工后，员工端立即清空内存中的 Key。
- **优势**：⚡ **速度极致最快**（100% 员工本机直连 DeepSeek 官方 API，0 中转延迟，管理员端无流量压力）。
- **边界**：对普通员工完全隐蔽；但若员工为技术人员并在本机抓包或 dump 进程内存，理论上可提取出真实 Key（可通过定期轮换 Key 缓解）。

### 模式 B：管理员 DSH 插件内置 1ms 换头透传（真实 Key 永不出管理员机器）
- **运行机制**：
  1. **管理员端插件**：利用 DSH 宿主自带的 `ctx.webserver` 暴露内网轻量透传路由（如 `/enterprise/anthropic/v1/*`）。
  2. **员工端插件**：本地只保存管理员颁发的「员工专属授权码（Token）」，并将 `llm-deepseek` 的 `baseURL` 指向管理员 DSH 实例。
  3. **转发与断权**：管理员端收到请求后，仅做 `< 1ms` 的授权码校验，将请求头 `x-api-key` 替换为企业真实 Key、向请求体注入 `"metadata": { "user_id": "emp_xxx" }`，随后直接流式 `pipe` 给 DeepSeek 官方。管理员禁用员工授权码即可**秒级断权**。
- **优势**：🔒 **真实 Key 绝对安全**（永不离开管理员机器），同时依然无需部署任何外部第三方网关软件，全套仅靠 DSH 插件闭环。
- **边界**：模型流式响应经过内网管理员节点一次轻量转发（内网延迟约 1ms，需保持管理员节点在线）。

---

## 6. 工作流

### L1：配置（先做，验证可行性）

- [ ] 写一份企业 profile 的 `cordis.patch.yml` 样例：
  - 声明 `llm-pi-ai` 网关路由，`apiKeyEnv` 指向企业凭据名
  - 关闭 `product-analytics`，或指向客户 OTLP
  - 禁用 `llm-deepseek-account`、`plugin-package-inventory-deepseek` 等官方账号相关行
- [ ] 用 `dsh --dump-config` 确认生效配置
- [ ] 本地起一个 LiteLLM Proxy 作为假网关，跑通一次对话

### L2：企业组合包 `@<org>/dsh-enterprise`

放在**独立仓库**，不进上游 monorepo。包含：

- [ ] **凭据提供方 `credentials-enterprise`**
  - 替代 `credentials-local` 注册 `ctx.credentials`
  - 流程：OIDC 设备码登录 → 向网关换短期虚拟 Key → 内存或短期缓存
  - 虚拟 Key 过期自动续；网关吊销后下一次请求失败并引导重新登录
  - 复用 `authorization` seam 呈现登录交互
- [ ] **策略插件 `policy-enterprise`**
  - 注册一个企业策略 answerer，排在交互式 answerer 之前，按策略直接拒绝（禁止目录、禁止外网 fetch、禁止某些工具）
  - 只暴露客户批准的权限预设
  - 策略从网关拉取（管理员在后台配置），本地只缓存
- [ ] **审计插件 `audit-enterprise`**：会话关键事件推到网关或客户 OTel
- [ ] 预装 `@wxg-prc-cpg/dsh-weknora`
- [ ] `package.json` 声明 `engines.dsh` 精确范围

### L3：桌面薄壳 fork

原则：fork 只改 `apps/desktop`，并尽量把差异做成可配置项而不是改逻辑。

- [ ] 品牌替换（名称、图标、关于面板、安装器素材）
- [ ] 更新源指向客户内网
- [ ] 欢迎页：去掉 DeepSeek 账号与 API Key 入口，换成企业 SSO
- [ ] 打包时预置企业 profile 与组合包
- [ ] 审查并关闭/重定向崩溃与操作上报
- [ ] Windows / macOS 代码签名

### L4：上游 PR —— 管理员锁定配置层（managed policy layer）

- [ ] 设计：从系统级只读路径读取（如 Windows `%ProgramData%\dsh\managed.patch.yml`、macOS `/Library/Application Support/dsh/managed.patch.yml`、Linux `/etc/dsh/managed.patch.yml`），在 `readProfilePatches()` 中**排在 home patch 与 overlays 之后**
- [ ] 语义待定：是"最后覆盖"，还是"锁定某些字段使低层无法修改"（后者更安全，但侵入更大）
- [ ] 与 HMR（`packages/boot/hmr/`）、插件管理器写入、`--dump-config` 的交互
- [ ] 先在 fork 里实现验证，再按 `CONTRIBUTING.zh.md` 提 PR；被拒则作为最小补丁长期维护

---

## 7. 待讨论的开放问题

0. **企业管理员 Key 授权管控插件选型（见第 5 节）**：采用「模式 A（管理员插件下发 Key 到员工端内存，员工端直连官方 API）」还是「模式 B（管理员 DSH 插件内置 1ms 换头透传，真实 Key 不出管理员机器）」？
1. 管理员锁定层用"覆盖"还是"字段锁定"语义？
2. 虚拟 Key / 授权缓存有效期多长？离线时允许继续用多久？
3. 策略是在客户端执行（插件），还是管理员转发节点/网关侧兜底执行（模型请求层面），还是两者都要？
4. 员工能否自行安装插件？如果允许，需要白名单还是签名校验？
5. 本地模型路由的触发条件：按项目、按知识库，还是按数据密级标签？
6. 跟进上游的节奏：每个上游版本都跟，还是只跟季度稳定点？

---

## 8. 风险

- 上游处于 alpha，Cordis 接口与 `engines.dsh` 语义可能破坏性变化
- 桌面壳 fork 越大，合并上游越痛；任何能放进插件的都不要放进壳
- 代码归属：在公司为公司项目开发的代码，开源前需公司书面同意
- 客户数据合规：确保默认配置下没有任何数据发往 DeepSeek 或我们公司

---

## 9. 建议的第一次 Claude Code 会话

在本机 clone 上游后，把本文件放到仓库外或 `docs/` 下，然后对 Claude Code 说：

> 读 DSH_ENTERPRISE_PLAN.md。先不改代码。
> 1. 核对第 4 节所有路径与结论在当前主干上是否仍然成立，列出变化。
> 2. 深入读 `packages/boot/app-boot/src/profile-context.ts`、`profile.ts` 和 `docs/subsystems/credentials.zh.md`，给出 L4 管理员锁定层和 L2 凭据提供方的接口设计草案，以及各自要改/新增的文件清单。
> 3. 把第 7 节的开放问题逐个给出你的建议。
