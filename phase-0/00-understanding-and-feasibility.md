# 阶段 0 · 纯理解与可行性初判报告

> 状态：**只读完成，未起任何环境**（无 StackBlitz / 无本地 dev server / 无 API 调用）。
> 数据来源：objectstack.ai 官网、`objectstack-ai/hotcrm`、`objectstack-ai/docs` 的源码与文档（只读）。
> 阅读方法：因会话仅授权 `yinlianghui/objectstack-launch`，三个 ObjectStack 仓库通过 GitHub 公开原文（raw / 页面）只读获取，未克隆、未运行。

---

## 0. 一句话结论（给赶时间的人）

**核心叙事的第一支柱（AI 原生后端）成立，且证据比总纲想象的更硬；但第二支柱（"完全自由的纯 React 前端"）与三个真实仓库的实际实现和官方叙事直接冲突——ObjectUI 官方定位是"服务端驱动 / 元数据投影"的渲染引擎（自称"类 Salesforce 的元数据驱动架构"），恰恰是总纲要我们对立的那类东西。这需要人类拍板重构叙事，否则一旦社区 clone 打脸，可信度会崩。** 另有 3 个会直接击穿"现场演示"的可行性风险（AI 需外部 OpenAI Key、server action 在 WASM 沙箱有已知崩溃、license/版本号事实性错乱）。

---

## 1. 我理解的核心叙事（用我自己的话）

ObjectStack 想干的事，本质是把"一个企业应用"从"一堆手写代码"压缩成"一份可被机器读懂的元数据协议"，然后让三个部件各司其职：

- **ObjectQL（协议 / 数据层）**：一份 schema 同时定义数据模型、API、校验、关系。它是"单一事实源"。
- **ObjectOS（引擎 / 控制层）**：像内核一样启动、加载驱动与应用，并强制所有请求过一遍鉴权 / 权限 / 审计 / 工作流。官方一句话是 "Governed runtime for AI-written apps"。
- **ObjectUI（渲染 / 视图层）**：**不"写"界面，而是把 ObjectQL 的 schema"投影"成界面**（server-driven UI），底层用 React + Tailwind + Shadcn 渲染。

真正性感的点在于：**因为整个应用是元数据，AI 就能"读懂并驾驭"它**——不是"AI 帮你写代码"，而是"AI 直接把你的业务模型当作可操作对象"。你今天给某个对象加一个字段，AI 下一句话就能用上这个字段，因为它每次都重新读 live schema。这是"这就是未来"的真实落点。

> **与总纲的关键差异（现在就要标红）**：总纲把"哇"押在两根柱子——(1) AI 原生后端 +（2）**你完全掌控的、纯手写 React 前端**，并强调"不受低代码平台受限 UI 的天花板限制"。但代码与官方文档显示，ObjectStack 的 UI 哲学恰恰是 **"Intent over Implementation"——用声明式数据、而非命令式代码来定义界面**。第二根柱子在真实仓库里不成立（详见 §3）。

---

## 2. 实锤 A ·「AI 原生后端」——成立，但机制与总纲描述不同

### 2.1 总纲的说法需要修正

总纲原话："hotcrm 中**每个实体都有一个 `*.action.ts`，它同时也是一个 AI tool**"。

**核对代码后，这个表述不准确，有三处偏差：**

1. 文件名是 `*.actions.ts`（复数），**不是每个实体一个**——`src/actions/` 下只有 5 个：`case.actions.ts`、`contact.actions.ts`、`global.actions.ts`、`lead.actions.ts`、`opportunity.actions.ts`。
2. 这些 `Action` 来自 `@objectstack/spec/**ui**`，是**界面动作**（挂在 `record_header` / `list_toolbar` / `list_item` 等 UI 位置），**不是** AI tool。例如 `src/actions/lead.actions.ts`：

```typescript
// src/actions/lead.actions.ts
import type { Action } from '@objectstack/spec/ui';
import { P } from '@objectstack/spec';

export const ConvertLeadAction: Action = {
  name: 'convert_lead',
  label: 'Convert Lead',
  objectName: 'crm_lead',
  type: 'flow',              // ← 委托给 src/flows 里的 lead_conversion 流程
  target: 'lead_conversion',
  locations: ['record_header', 'list_item'],   // ← 这是 UI 按钮的位置
  visible: P`record.status == "qualified" && record.is_converted == false`,
  confirmText: 'Are you sure you want to convert this lead?',
  refreshAfter: true,
};
```

3. **真正让 AI 能读懂 / 操作业务模型的机制是 `agent → skill → tool`，不是 action。**

### 2.2 真正的实锤（更硬，建议改用这个作为爆点）

**Agent 定义**（`src/agents/sales-copilot.agent.ts`，逐字）：

```typescript
export const SalesCopilotAgent = defineAgent({
  name: 'sales_copilot',
  label: 'Sales Copilot',
  instructions: `... CRITICAL: HotCRM's schema is alive — admins add,
modify and remove fields at any time. Before answering any question
about a record, call \`describe_object\` (via the Live Data skill) to
see the CURRENT fields. Never assume the schema you saw on a previous
turn is still accurate. If you spot a field you don't recognise, use
it — that's almost certainly what the user wants you to surface.`,
  model: { provider: 'openai', model: 'gpt-4', temperature: 0.6, maxTokens: 2000 },
  skills: ['live_data', 'lead_qualification', 'email_drafting',
           'revenue_forecasting', 'customer_360'],
  knowledge: { topics: [...], indexes: ['sales_knowledge'] },
});
```

**Skill 把平台内建的数据工具接给 AI**（`src/skills/live-data.skill.ts`，逐字节选）——注意注释里官方自己把它标成 "Wow #1"：

```typescript
/**
 * Live Data — the foundation of HotCRM's "Wow #1" demo.
 * ... This is what makes "add a field, AI uses it 30s later"
 * actually work — `describe_object` re-reads metadata on every call.
 */
export const LiveDataSkill = defineSkill({
  name: 'live_data',
  instructions: `... 1. **Always** call \`describe_object\` for the
relevant object FIRST, even if you think you remember the fields.
The schema can change between turns ...`,
  tools: [
    'describe_object',   // 读 live 元数据
    'list_objects',
    'query_records',
    'get_record',
    'aggregate_data',
  ],
  triggerPhrases: ['show me', 'how many', 'list', 'count', 'sum', 'top', ...],
  permissions: ['crm:read'],
});
```

**结论**：AI 原生这件事**是真的、可讲、且有官方自认的 demo（"Wow #1"）**。最有说服力的现场效果就是它自己写的那句："**加一个字段，AI 30 秒后就会用上它**"——因为 `describe_object` 每次都重读元数据。**建议把爆点从"每个 action 都是 AI tool"改成"你的业务模型是活的元数据，AI 每一步都重新读它、并直接查询 / 聚合 / 操作它"。**

> 附带修正：总纲 §6 称"AI agent 治理经 **MCP tools**"。代码里看到的是 skill 里声明的 `tools`（平台内建的 `describe_object` / `query_records` 等），**未在代码 / 文档中确认这些是 MCP 协议工具**。使用前请让维护者确认是否真走 MCP，否则不要在脚本里写"MCP"。

---

## 3. 实锤 B ·「自由 React 前端 + 元数据绑定」——**与真实实现冲突，这是最大风险**

总纲第二支柱："**前端你完全掌控：纯 React 微页面**……不受低代码平台的丑 UI 和天花板限制。"

**核对结果：这个描述与 hotcrm 代码、docs、官网三处都对不上。**

### 3.1 hotcrm 的"前端"全是声明式元数据，不是手写 React

- `src/pages/*.page.ts`、`src/views/*.view.ts` 全是 `.ts` **元数据配置**，不是 `.tsx` 组件。
- `src/interfaces/` 目录（最可能放自定义组件的地方）里**只有一个空的 `index.ts`**。
- 页面用 `page:card` / `record:highlights` / `three-column` 这类**声明式组件类型**拼装，明确对标 Airtable Interfaces / Salesforce Lightning。逐字看 `src/pages/account_workbench.page.ts` 的作者注释：

```typescript
/**
 * Account Workbench — ADR-0047 interface page (Airtable Interfaces parity).
 * ...
 */
export const AccountWorkbenchPage: Page = {
  type: 'list',
  object: 'crm_account',
  // Interface pages carry no regions — the list surface is generated
  // from `interfaceConfig` (ADR-0047), not composed from components.
  regions: [],
  interfaceConfig: { source: 'crm_account', sourceView: 'all_accounts', ... },
};
```

**"generated from `interfaceConfig`, not composed from components" —— 这就是低代码 / server-driven 的做法，不是"你写自由 React"。**

### 3.2 官方文档把这条哲学写死了

- `docs/.../concepts/architecture.mdx`：ObjectUI 是 **"Projection Engine"**、**"does not contain hardcoded forms"**、**"server-driven UI"**。
- `docs/.../concepts/core-values.mdx`：头号价值主张是 **"Intent over Implementation — application logic defined by declarative data, not imperative code"**；并把 **React Hook Form 当作"旧世界的问题"来批判**（即 ObjectStack 要取代的东西）。
- `docs/.../creator-layer/` 只有 `index / sdk / cli` 三页，**没有任何"写自定义 React 组件"的入口文档**。
- 官网 / 官方 GitHub 描述：ObjectUI "**server-driven UI rendering that adapts to ObjectQL's schema, defining interfaces in JSON metadata and rendering with React + Tailwind + Shadcn**"；整体自我定位 "**Metadata Driven Architecture (similar to Salesforce)**"，主打 "**reduces code to 1%**"。

**即：React + Shadcn 是 ObjectUI 的"渲染底座"，不是"给应用开发者自由手写的层"。开发者写的是 JSON/TS 元数据。** hotcrm README/官网说的 "Shadcn UX" 指的是**平台内建 UI 好看**，不等于"你自由写 React"。

### 3.3 为什么这是红线级问题

1. 总纲把 ObjectStack 定位成"低代码受限 UI 的对立面（你有完全自由的 React）"。但真实的 ObjectStack **就是一个（AI 原生的、更强的）元数据驱动平台**，血统上更接近 Salesforce / Steedos（官网关联项目 `steedos/steedos-platform` 明写 "Powered by ObjectStack"）。**把它说成"自由 React、非低代码"，方向是反的。**
2. 一旦开发者按我们的话去 clone hotcrm 找"自由 React 微页面"，只会看到 `*.page.ts` 元数据 + server-driven 渲染——**"不受低代码天花板限制"这句会当场被打脸**，且连带拖垮第一支柱的可信度（红线 #2、#3）。
3. 官方自己主打的 "reduces code to 1%" 正是总纲红线 #1 明令**不许当主卖点**的"行数对比"。我们要主动避开它，而不是无意中滑进去。

### 3.4 我的建议（供人类决策，不擅自改叙事）

把双支柱叙事改成**诚实但依然很"哇"的版本**：

> **"你的整个企业应用第一次变成一份 AI 能读懂、能治理、能实时操作的活元数据——模型、权限、流程、UI 全部由同一份 schema 投影出来；改一处，AI 和界面同一秒都跟着变。"**

即：**主卖点 = "AI 原生 + 单一元数据事实源 + 强治理运行时（ObjectOS）"**；UI 的卖点从"自由 React"改成"**声明式元数据一处定义、React/Shadcn 自动渲染出企业级好看的界面，且天然可被 AI 读写**"。这既是真的、可复现，又避开了"行数对比"和"自由 React"两个雷。**是否采纳，请人类拍板。**

---

## 4. 最适合做「钩子」的可复现小场景（候选）

结合真实能力（尤其 `live_data` skill 自认的 "Wow #1"），给两个具体候选：

**候选 A（首选）·「活元数据：加个字段，AI 立刻会用」**
- 剧本：在跑起来的 hotcrm 里，给某对象（如 Lead）加一个自定义字段（如 `pain_point`）→ 不重启、不改代码 → 直接问 Sales Copilot："列出所有 lead 的 pain point / 按 pain point 汇总" → AI 因为每次都 `describe_object`，当场就用上这个刚加的字段。
- 为什么强：这是官方代码注释里点名的 "Wow #1"，科幻感 = "软件被 AI 真正读懂"，且**不依赖前端自由度**（绕开了第二支柱的坑）。
- ⚠️ 前置条件：**需要接一个真实 LLM（hotcrm 现在写死 OpenAI GPT-4，需 API Key）**、需要种子数据、且"加字段"的入口要确认是否在 StackBlitz 零配置下可用（见 §5）。

**候选 B（备选）·「一句自然语言，AI 查 + 聚合业务数据」**
- 剧本：对 Sales Copilot 说"本季度赢单金额 top 5 的机会是哪些、总额多少"→ 它 `describe_object` + `query_records` + `aggregate_data` 直接给出并回链 record ID。
- 为什么强：纯读、最容易稳定复现；缺点是"AI 会查数据"没有 A 那么科幻。

> 注：总纲设想的"从 0 现场生长一个自定义富 UI 的 todo 小应用"**风险较高**——因为 UI 是声明式元数据（不是随手写 React），"现场生长一个漂亮自定义 UI"要走 `*.page.ts` / `interfaceConfig` 那一套，步骤不短、也不如"AI 用上新字段"直觉。**建议钩子聚焦 AI×活元数据，而不是"现场手搓 UI"。**

---

## 5. 可行性风险初判（针对"现场演示"）

按"会不会击穿演示"从高到低：

### 🔴 R1 · AI 演示需要外部 LLM API Key（击穿 "Wow #1" 的开箱复现）
`sales-copilot.agent.ts` 写死 `model: { provider: 'openai', model: 'gpt-4' }`。**任何 clone hotcrm 的社区成员，不配 `OPENAI_API_KEY` 就跑不出 AI 效果。** 这与"短、快、可复现、clone 就能复现"的病毒传播要求直接冲突。
→ **待人类 / 维护者确认**：StackBlitz badge 环境是否预置了可用的模型接入？是否有免 Key 的 demo 模式或代理？若没有，钩子片必须诚实标注"需自备 OpenAI Key"，或改用不依赖 LLM 的场景。

### 🔴 R2 · server action 在 WASM 沙箱有已知崩溃（clone 打脸风险）
`src/actions/opportunity.actions.ts` 的 `CloneOpportunityAction` 里作者留了注释：**"Under the QuickJS WASM sandbox this pattern triggers an emscripten 'memory access out of bounds' fault when marshalling certain row shapes."** 说明 server action 跑在 QuickJS WASM 沙箱里，某些数据形状会崩。
→ StackBlitz（sqlite-wasm + WASM 沙箱）录屏时，**要避开会崩的动作**；演示脚本里用到的每一个按钮 / action 都必须先在目标环境实测过（红线 #2）。

### 🟠 R3 · StackBlitz 冷启动 & 首屏
`.stackblitzrc`：`installDependencies: false` + `startCommand: "pnpm install && pnpm dev"`，`OS_DATABASE_DRIVER: sqlite-wasm`。README 自认首次约 **60s**。录屏需处理这段等待（剪辑 / 预热 / 加载态转场）。整体可支撑演示，但不是"秒开"。

### 🟠 R4 · 事实性错乱（红线 #5，上线前必须核对）
- **License 冲突（已核实，确有其事）**：`LICENSE` 文件是 **MIT**（"MIT License, Copyright (c) 2024 HotCRM Team"），但 README 徽章和**每个源码文件头**都写 **"Apache-2.0"**。→ 真实 license 待维护者确认，**素材里在确认前不许出现具体 license 名**。
- **版本号冲突**：README 写 `Version 1.0.0`，但 `package.json` 与 `objectstack.config.ts` 都是 **`2.0.0`**。
- **端口**：总纲 §6 写本地运行在 `http://localhost:3000`，**实际是 `4001`**（`package.json` dev 脚本 + README 均为 4001）。截图 / 脚本里别再写 3000。
- **数量口径**：总纲 §4 写"6 workflow"，README 实为 **10 automation flows**、10 server actions、**6 AI skills**（skill 与 flow 是两码事，别混）。对象 15、agent 2、语言 4（en/zh-CN/ja/es）与总纲一致，已核对属实。

### 🟡 R5 · 技术栈基线（供移交 / 复现环境对齐）
Node **>= 22**、pnpm **10.33**、`@objectstack/*` 依赖 **v13.0.0**、ObjectQL（无裸 SQL）、`@objectstack/spec` 校验元数据。本地复现需 pnpm。这些不阻塞，但移交包要写清。

---

## 6. 需要人类拍板的三件事（阻塞进入阶段 1）

1. **叙事方向**：第二支柱"自由 React 前端"与真实实现冲突。是否采纳 §3.4 的重构版叙事（主卖点转向"AI 原生 + 活元数据 + 强治理运行时"，UI 卖点转向"声明式元数据→React/Shadcn 自动渲染企业级 UI"）？
2. **AI 演示的可复现性**：hotcrm 的 AI 写死 OpenAI GPT-4 需 Key。StackBlitz 里能否零配置跑出 AI 效果？没有的话，钩子片走"需自备 Key"诚实版，还是换不依赖 LLM 的场景？
3. **License**：MIT vs Apache-2.0 到底哪个是真的？（在此确认前，所有素材不写具体 license。）

---

## 7. 阶段 0 结束

以上为纯理解与可行性初判，**全程只读、未起任何环境**。按流程在此**停下**，等待人类就 §6 三件事确认叙事方向后，再取阶段 1 提示词进入"小步起环境验证"。
