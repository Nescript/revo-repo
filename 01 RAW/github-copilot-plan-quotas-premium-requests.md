# GitHub Copilot 各订阅套餐的用量额度：AI Credits（现行）与 premium requests（旧制）

研究日期：2026-08-30
研究目标：查清 GitHub Copilot 各套餐（Free / Pro / Pro+ / Business / Enterprise，及 2026 年新增的 Max）的每月用量额度；明确计量单位是"请求次数"还是"token"；梳理模型乘数、超额付费规则、agent 类功能的计量方式，以及额度政策的变更时间线。

## 研究范围与判断方式

- 只使用官方一手来源：docs.github.com（GitHub Docs）、github.blog（官方博客与 Changelog）、github.com/features/copilot/plans（官方定价页）。
- 结论分级：
  - **【官方事实】**：官方文档/公告明确写明的内容，逐条附来源链接。
  - **【归纳】**：基于多处官方内容归纳的解释。
  - **【待确认】**：官方来源之间不一致、或现行文档未给出确切数字的事项，集中在文末列出。
- 重要背景：GitHub 已于 **2026-06-01** 将全部套餐从"premium request（按请求次数）"计费切换为"GitHub AI Credits（按 token 用量）"计费。本报告以现行 AI Credits 体系为主线，旧制 premium requests 单独成节说明（目前仅对少数遗留按年订阅者适用）。

## 核心结论

1. **各套餐额度（现行，2026-06-01 起）**：额度以 GitHub AI Credits 计量（1 credit = $0.01 USD）。Copilot Pro（$10/月）含 1,500 credits/月（base 1,000 + flex 500）；Copilot Pro+（$39/月）含 7,000（3,900 + 3,100）；Copilot Max（$100/月）含 20,000（10,000 + 10,000）；Copilot Business（$19/人/月）含 1,900 credits/人/月；Copilot Enterprise（$39/人/月）含 3,900 credits/人/月（组织内 pooling）；Copilot Free 有"一定额度"的 AI Credits（官方未公布具体数字）+ 每月 2,000 次代码补全上限。[Plans for GitHub Copilot](https://docs.github.com/en/copilot/get-started/plans)、[Usage-based billing for individuals](https://docs.github.com/en/copilot/concepts/billing/usage-based-billing-for-individuals)、[Usage-based billing for organizations and enterprises](https://docs.github.com/en/copilot/concepts/billing/usage-based-billing-for-organizations-and-enterprises)
2. **请求制还是 token 制**：**现行体系按 token 计量**。每次交互消耗 input / output / cached（部分模型还有 cache write）token，按各模型"每 1M tokens"的官方定价折算成 AI Credits 扣减。代码补全（code completions）和 next edit suggestions 不消耗 credits，所有付费套餐内无限使用。旧制（premium requests，按请求次数 × 模型乘数）已于 2026-06-01 被取代，仅适用于仍留在按年付费旧计划的 Pro/Pro+ 订阅者。[GitHub Copilot is moving to usage-based billing](https://github.blog/news-insights/company-news/github-copilot-is-moving-to-usage-based-billing/)、[Models and pricing for GitHub Copilot](https://docs.github.com/en/copilot/reference/copilot-billing/models-and-pricing)
3. **模型乘数（model multipliers）是旧制概念，现行体系已不存在**。旧制下每次交互消耗"1 次 premium request × 模型乘数"；乘数 0x（如 GPT-4.1、GPT-4o）的准确含义是：在付费套餐内用这些模型进行 chat / agent mode 交互**不扣减额度、等效无限使用**（仍受速率限制）。现行体系没有乘数，也不再有"0x 无限用"的模型——成本完全由所选模型的 token 单价决定；旧的"额度用完后回落到低成本模型"（fallback）机制也已取消。[Update to GitHub Copilot consumptive billing experience (2025-06-18)](https://github.blog/changelog/2025-06-18-update-to-github-copilot-consumptive-billing-experience/)、[Model multipliers for annual plans on request-based billing (legacy)](https://docs.github.com/en/copilot/reference/copilot-billing/request-based-billing-legacy/model-multipliers-for-annual-plans)、[GitHub Copilot is moving to usage-based billing](https://github.blog/news-insights/company-news/github-copilot-is-moving-to-usage-based-billing/)

## 现行体系：GitHub AI Credits（2026-06-01 起）

### 计量规则【官方事实】

- AI Credits 是 Copilot 用量的计费单位，**1 AI credit = $0.01 USD**。[GitHub Copilot billing 概念页](https://docs.github.com/en/billing/concepts/product-billing/github-copilot-billing)
- 每次 Copilot 交互消耗 token：input tokens（发送给模型的内容）、output tokens（模型生成的内容）、cached tokens（模型复用的上下文），部分模型另有 cache write 成本；每种 token 按所用模型的单价计费，总额折算为 AI Credits。[Models and pricing for GitHub Copilot](https://docs.github.com/en/copilot/reference/copilot-billing/models-and-pricing)
- 所有模型价格以"每 1M tokens"标价，超额用量同样按此 per-token 费率以 credits 结算。[Models and pricing for GitHub Copilot](https://docs.github.com/en/copilot/reference/copilot-billing/models-and-pricing)
- **代码补全与 next edit suggestions 不按 AI Credits 计费，所有付费套餐内保持无限使用**（继续沿用原有的计数机制）。[Models and pricing for GitHub Copilot](https://docs.github.com/en/copilot/reference/copilot-billing/models-and-pricing)、[GitHub Copilot Plans & pricing](https://github.com/features/copilot/plans)
- 消耗 AI Credits 的功能包括：Copilot Chat、agent mode、code review、Copilot cloud agent（coding agent）、Copilot CLI、Copilot Spaces、Spark、Copilot Apps 及第三方 coding agents。[GitHub Copilot Plans & pricing](https://github.com/features/copilot/plans)、[Usage-based billing for organizations and enterprises](https://docs.github.com/en/copilot/concepts/billing/usage-based-billing-for-organizations-and-enterprises)
- 个人套餐的 credits 分两部分：**base credits**（与订阅价格等额，固定不变）+ **flex allotment**（额外的浮动额度，GitHub 可根据模型定价与效率变化调整）。[Usage-based billing for individuals](https://docs.github.com/en/copilot/concepts/billing/usage-based-billing-for-individuals)
- 组织/企业套餐中，每个席位贡献的 credits 在 billing entity 层面**池化（pooled）**共享；未用完的 credits 不结转，每月 1 日 00:00 UTC 重置。[Usage-based billing for organizations and enterprises](https://docs.github.com/en/copilot/concepts/billing/usage-based-billing-for-organizations-and-enterprises)

### 各套餐额度对照表（现行）【官方事实】

| 套餐 | 价格（月付） | 每月包含 AI Credits | 代码补全 | 备注 |
| --- | --- | --- | --- | --- |
| Copilot Free | 免费 | 官方仅称"an allowance"，未公布具体数字【待确认】 | 上限 2,000 次/月 | 仅 auto model selection；agent 能力受限 |
| Copilot Student | 免费（验证后） | 一定额度 AI Credits | 无限 | 仅 auto model selection；不含第三方 agents |
| Copilot Pro | $10/月 | base 1,000 + flex 500 = **1,500** | 无限 | 部分模型可用 |
| Copilot Pro+ | $39/月 | base 3,900 + flex 3,100 = **7,000** | 无限 | 可用 premium 模型 |
| Copilot Max | $100/月 | base 10,000 + flex 10,000 = **20,000** | 无限 | 2026-06-01 随新计费推出，面向高强度 agent 工作流 |
| Copilot Business | $19/人/月 | **1,900**/人/月（池化） | 无限 | 2026-06-01 至 2026-09-01 过渡期促销为 3,000/人/月 |
| Copilot Enterprise | $39/人/月 | **3,900**/人/月（池化） | 无限 | 仅限 GitHub Enterprise Cloud；过渡期促销为 7,000/人/月 |

来源：[Plans for GitHub Copilot](https://docs.github.com/en/copilot/get-started/plans)、[Usage-based billing for individuals](https://docs.github.com/en/copilot/concepts/billing/usage-based-billing-for-individuals)、[Usage-based billing for organizations and enterprises](https://docs.github.com/en/copilot/concepts/billing/usage-based-billing-for-organizations-and-enterprises)、[GitHub Copilot licenses](https://docs.github.com/en/billing/concepts/product-billing/github-copilot-licenses)、[GitHub Copilot Plans & pricing](https://github.com/features/copilot/plans)、[Updates to GitHub Copilot billing and plans (2026-06-01)](https://github.blog/changelog/2026-06-01-updates-to-github-copilot-billing-and-plans/)

### 模型 per-token 定价（节选，每 1M tokens）【官方事实】

| 模型 | Input | Cached input | Cache write | Output |
| --- | --- | --- | --- | --- |
| GPT-5 mini | $0.25 | $0.025 | — | $2.00 |
| GPT-5.4（≤272K） | $2.50 | $0.25 | — | $15.00 |
| GPT-5.5（≤272K） | $5.00 | $0.50 | — | $30.00 |
| Claude Haiku 4.5 | $1.00 | $0.10 | $1.25 | $5.00 |
| Claude Sonnet 4 / 4.5 / 4.6 | $3.00 | $0.30 | $3.75 | $15.00 |
| Claude Opus 4.5–4.8 / Opus 5 | $5.00 | $0.50 | $6.25 | $25.00 |
| Gemini 3.1 Pro（≤200K） | $2.00 | $0.20 | — | $12.00 |

完整定价表（含长上下文档位、促销价）见 [Models and pricing for GitHub Copilot](https://docs.github.com/en/copilot/reference/copilot-billing/models-and-pricing)。

### 超额付费规则（现行）【官方事实】

- 额度用完后：
  - **个人用户**：可升级套餐（只补差价，当月已用量计入新套餐额度），或设置"additional usage"美元预算继续按 $0.01/credit 付费使用；个人用户的超额总量可能因用量模式、账单历史、账户验证状态被限制。通过 GitHub Mobile（iOS/Android）订阅的用户无法购买额外 credits。
  - **组织/企业**：超额付费（"AI credits paid usage"策略）**默认开启**，超额部分按 $0.01/credit 向组织/企业收费；管理员可在 AI Controls 设置中显式关闭，关闭后额度耗尽即停用至下个账期。
- 预算控制层级：user-level budget（ULB，限制个人总消耗，**永远硬性停止**，$0 预算立即封锁）、cost center 预算、organization 预算、enterprise 预算（后三者只限制池子耗尽后的 metered 超额部分）；"Stop usage when budget limit is reached" 开关**默认关闭**，不设则会持续产生费用。预算以美元设置，按 1 credit = $0.01 折算。

来源：[Usage-based billing for individuals](https://docs.github.com/en/copilot/concepts/billing/usage-based-billing-for-individuals)、[Budgets for usage-based billing](https://docs.github.com/en/copilot/concepts/billing/budgets-for-usage-based-billing)、[Usage-based billing for organizations and enterprises](https://docs.github.com/en/copilot/concepts/billing/usage-based-billing-for-organizations-and-enterprises)、[Updates to GitHub Copilot billing and plans (2026-06-01)](https://github.blog/changelog/2026-06-01-updates-to-github-copilot-billing-and-plans/)

### Agent 类功能的计量（现行）【官方事实】

- Chat、agent mode、code review、Copilot cloud agent、Copilot CLI 等均按 token 消耗 AI Credits，不再有"每会话 1 次请求"的固定计价。[GitHub Copilot Plans & pricing](https://github.com/features/copilot/plans)
- **Copilot code review 双重计费**：token 消耗以 AI Credits 计费（模型由系统自动选择且不公开，单次成本不可预估）；同时其 agentic 基础设施消耗 **GitHub Actions 分钟数**（私有仓库从套餐包含的 Actions 分钟中扣除，超出按标准 Actions 费率计费；公共仓库 Actions 分钟免费）。Actions 分钟归属仓库/企业，credits 归属发起 review 的用户或 PR 作者。[Models and pricing for GitHub Copilot](https://docs.github.com/en/copilot/reference/copilot-billing/models-and-pricing)、[Copilot code review will start consuming GitHub Actions minutes on June 1, 2026](https://github.blog/changelog/2026-04-27-github-copilot-code-review-will-start-consuming-github-actions-minutes-on-june-1-2026/)
- Copilot cloud agent 运行在 GitHub Actions 上，旧制下明确同时消耗 Actions 分钟 + premium requests；现行体系下官方文档明确其消耗 AI Credits，但是否仍并行消耗 Actions 分钟，现行文档未明确复述【待确认】。[Overview of request-based billing (legacy)](https://docs.github.com/en/copilot/reference/copilot-billing/request-based-billing-legacy/github-copilot-premium-requests)

## 旧制：premium requests（legacy，2025-06-18 至 2026-06-01，仅遗留按年订阅者适用）

### 适用人群【官方事实】

legacy 文档系列仅适用于：**2026-06-01 之后仍留在按年（annual）付费计划上的 Copilot Pro / Pro+ 订阅者**。这些用户可继续使用 premium request 计费直到年度计划到期；到期后自动降级为 Copilot Free，也可提前取消（按剩余价值比例退款）或升级为月付套餐。月付 Pro/Pro+ 用户及 Business/Enterprise 均已于 2026-06-01 切换到 AI Credits。[What changed with Copilot billing (legacy)](https://docs.github.com/en/copilot/reference/copilot-billing/request-based-billing-legacy/what-changed-with-billing)、[GitHub Copilot request-based billing for annual plan subscribers (legacy)](https://docs.github.com/en/copilot/reference/copilot-billing/request-based-billing-legacy)

### premium request 的定义【官方事实】

- "一次请求"指任何让 Copilot 执行任务的交互：在 chat 窗口发送一条 prompt、触发一次 Copilot 响应，都算一次请求。
- 对 agent 类功能，**只有用户发送的 prompt 计入 premium requests**；Copilot 自主执行的动作（如 tool calls）不计。例如在 Copilot CLI 中使用 `/plan` 计 1 次 premium request。
- 使用 premium 模型的交互才扣额度；部分模型有乘数，一次交互可能按多倍扣减（官方举例：高级推理模型可能按 5× 或 20× 计）。
- 额度每月 1 日 00:00 UTC 重置。

来源：[Requests in GitHub Copilot (legacy)](https://docs.github.com/en/copilot/reference/copilot-billing/request-based-billing-legacy/copilot-requests)、[Overview of request-based billing (legacy)](https://docs.github.com/en/copilot/reference/copilot-billing/request-based-billing-legacy/github-copilot-premium-requests)

### 旧制各套餐额度【官方事实】

| 套餐 | 每月 premium requests | 超额单价 |
| --- | --- | --- |
| Copilot Free | 50 次（另含 2,000 次代码补全） | 不可购买 |
| Copilot Pro（$10/月） | 300 次 | $0.04/次 |
| Copilot Pro+（$39/月） | 1,500 次 | $0.04/次 |
| Copilot Business（$19/人/月） | 300 次/人 | $0.04/次 |
| Copilot Enterprise（$39/人/月） | 1,000 次/人 | $0.04/次 |

- Pro / Pro+ 数字及 $0.04/次：[Requests in GitHub Copilot (legacy)](https://docs.github.com/en/copilot/reference/copilot-billing/request-based-billing-legacy/copilot-requests)
- Business 300 / Enterprise 1,000 及发布时间：[Announcing GitHub Copilot Pro+ (2025-04-04)](https://github.blog/changelog/2025-04-04-announcing-github-copilot-pro/)、[GitHub Copilot agent mode activated](https://github.blog/news-insights/product-news/github-copilot-agent-mode-activated/)
- Copilot Free 的 50 次聊天/高级请求 + 2,000 次补全：[Announcing GitHub Copilot Free (2024-12-18)](https://github.blog/changelog/2024-12-18-announcing-github-copilot-free/)、[GitHub Copilot Plans & pricing](https://github.com/features/copilot/plans)
- 旧制下付费套餐的代码补全无限；使用 GPT-4.1 和 GPT-4o 进行 agent mode 与 chat 交互**无限**（0x 乘数），所有模型仍受速率限制：[Update to GitHub Copilot consumptive billing experience (2025-06-18)](https://github.blog/changelog/2025-06-18-update-to-github-copilot-consumptive-billing-experience/)
- 超额默认关闭（默认 spending limit 为 $0），需在账单设置中设定预算后才能按 $0.04/次购买额外请求：[Update to GitHub Copilot consumptive billing experience (2025-06-18)](https://github.blog/changelog/2025-06-18-update-to-github-copilot-consumptive-billing-experience/)

### 旧制模型乘数（2026-06-01 起对遗留按年订阅者生效的调整表，节选）【官方事实】

| 模型 | 旧乘数 | 2026-06-01 起新乘数 |
| --- | --- | --- |
| GPT-4o / GPT-4o mini | 0 | 0.33 |
| GPT-4.1 | 0 | 1 |
| GPT-5 mini | 0 | 0.33 |
| GPT-5.1 / GPT-5.1-Codex | 1 | 3 |
| GPT-5.3-Codex | 1 | 6 |
| GPT-5.4 | 1 | 6 |
| GPT-5.5 | — | 57 |
| Claude Haiku 4.5 | 0.33 | 0.33 |
| Claude Sonnet 4.5 | 1 | 6 |
| Claude Opus 4.5 | 3 | 15 |
| Claude Opus 4.6 / 4.7 / 4.8 | 3 / 7.5 / — | 27 |
| Gemini 2.5 Pro | 1 | 1 |
| Gemini 3 Pro / 3.1 Pro | 1 | 6 |

- 0x 乘数的含义：使用该模型的交互不扣减 premium request 额度，即在付费套餐内无限使用（受速率限制）。2026-06-01 的调整表已将所有 0x 模型改为 ≥0.33，即遗留按年订阅者不再有任何"免费"模型。
- 另：自 2026-06-01 起，Copilot code review 的乘数为 13（每次 PR review 或 IDE 内 review 扣 13 次 premium requests）。

来源：[Model multipliers for annual plans on request-based billing (legacy)](https://docs.github.com/en/copilot/reference/copilot-billing/request-based-billing-legacy/model-multipliers-for-annual-plans)、[Models and pricing for GitHub Copilot](https://docs.github.com/en/copilot/reference/copilot-billing/models-and-pricing)、[Requests in GitHub Copilot (legacy)](https://docs.github.com/en/copilot/reference/copilot-billing/request-based-billing-legacy/copilot-requests)

### 旧制下 agent 类功能的计量【官方事实】

- Copilot coding agent（cloud agent）：自 2025-07-10 起，**每个 session 固定消耗 1 次 premium request**（无论修改多少文件）；session 指给 Copilot 下达一个任务或将 issue 指派给 Copilot。同时照常消耗 GitHub Actions 分钟数（与账号的 Actions 免费分钟额度共享）。[GitHub Copilot coding agent now uses one premium request per session (2025-07-10)](https://github.blog/changelog/2025-07-10-github-copilot-coding-agent-now-uses-one-premium-request-per-session/)、[Overview of request-based billing (legacy)](https://docs.github.com/en/copilot/reference/copilot-billing/request-based-billing-legacy/github-copilot-premium-requests)
- 旧制的 premium request 用量按 SKU 归属：Copilot premium requests（Chat、CLI、Code Review、Extensions、Spaces）、Spark premium requests、Copilot cloud agent premium requests，可分别设 SKU 级预算。[Overview of request-based billing (legacy)](https://docs.github.com/en/copilot/reference/copilot-billing/request-based-billing-legacy/github-copilot-premium-requests)

## 政策变更时间线【官方事实】

| 时间 | 变更 |
| --- | --- |
| 2024-12-18 | 推出 Copilot Free：每月 2,000 次代码补全 + 50 次聊天消息。[Changelog](https://github.blog/changelog/2024-12-18-announcing-github-copilot-free/) |
| 2025-04-04 | 发布 Copilot Pro+（$39/月），公布 premium request 额度：Pro 300、Pro+ 1,500、Business 300、Enterprise 1,000；Pro/Pro+ 自 5 月 5 日、Business/Enterprise 自 5 月 12–19 日陆续生效。[Changelog](https://github.blog/changelog/2025-04-04-announcing-github-copilot-pro/) |
| 2025-06-18 | premium request 计费在 GitHub.com 全部付费套餐正式强制执行（GHE.com 为 2025-08-01）；计数器清零；超额默认关闭（$0 spending limit）；明确 GPT-4.1/GPT-4o 在付费套餐内 0x 无限使用。[Changelog](https://github.blog/changelog/2025-06-18-update-to-github-copilot-consumptive-billing-experience/)、[Requests in GitHub Copilot (legacy)](https://docs.github.com/en/copilot/reference/copilot-billing/request-based-billing-legacy/copilot-requests) |
| 2025-07-10 | coding agent 改为每 session 固定 1 次 premium request（此前按用量浮动）。[Changelog](https://github.blog/changelog/2025-07-10-github-copilot-coding-agent-now-uses-one-premium-request-per-session/) |
| 2026 年上半年 | GitHub 宣布全部套餐将于 2026-06-01 切换为 usage-based billing（AI Credits，按 token 计量），套餐价格不变。[The GitHub Blog](https://github.blog/news-insights/company-news/github-copilot-is-moving-to-usage-based-billing/) |
| 2026-04-27 | 预告 Copilot code review 自 2026-06-01 起额外消耗 GitHub Actions 分钟数。[Changelog](https://github.blog/changelog/2026-04-27-github-copilot-code-review-will-start-consuming-github-actions-minutes-on-june-1-2026/) |
| 2026-06-01 | **usage-based billing 全面生效**：PRU 被 AI Credits 取代；推出 Copilot Max（$100/月）；user-level budgets GA；月付 Pro/Pro+ 自动迁移，按年 Pro/Pro+ 可选择留在旧制直至到期（乘数表同日上调）；存量 Business/Enterprise 客户 6–8 月享受促销额度（3,000 / 7,000 credits 每人，至 2026-09-01）。[Changelog](https://github.blog/changelog/2026-06-01-updates-to-github-copilot-billing-and-plans/)、[Usage-based billing for organizations and enterprises](https://docs.github.com/en/copilot/concepts/billing/usage-based-billing-for-organizations-and-enterprises) |

## 待确认 / 可能变动事项

1. **Copilot Free 的现行 AI Credits 具体额度**：docs.github.com 现行文档仅称 Free 有"an allowance of GitHub AI Credits"，未给出数字；而官方定价页 FAQ 仍写"Free 用户限 2,000 次补全 + 50 次聊天请求（含 Copilot Edits）"，两处口径可能不同步（FAQ 可能是旧表述）。Free 的聊天额度在现行体系下的确切数字待确认。
2. **Copilot cloud agent 在现行体系下是否仍消耗 GitHub Actions 分钟数**：官方仅明确 code review 自 2026-06-01 起双重计费；cloud agent 同样运行在 Actions 上，但现行文档未明确复述其 Actions 分钟消耗规则。
3. **flex allotment 的可变性**：个人套餐总额度中 flex 部分（Pro 500、Pro+ 3,100、Max 10,000）官方明确会随"AI 经济变化"调整，本文数字为研究日期时点的快照。
4. **Business/Enterprise 促销额度已于 2026-09-01 结束**（研究日期 2026-08-30 时仍在促销期内），之后回落到 1,900 / 3,900 credits 每人。
5. **旧制 Business/Enterprise 额度（300 / 1,000）**出自 2025-04-04 官方公告；现行 legacy 文档只保留 Pro/Pro+ 的说明，Business/Enterprise 的旧制细节页面已随切换下线。
6. **usage-based billing 官方博客公告的确切发布日期**：本次检索未抓取到该博客页面的发布日期元数据（内容显示其早于 2026-04-27 的 code review 预告）。
7. 各模型 per-token 定价与促销价（如 Gemini Flash 促销至 2026-12-31、GPT-5.6 Sol 促销至 2026-09-03）有时效性，引用时请以 [Models and pricing for GitHub Copilot](https://docs.github.com/en/copilot/reference/copilot-billing/models-and-pricing) 当时内容为准。
