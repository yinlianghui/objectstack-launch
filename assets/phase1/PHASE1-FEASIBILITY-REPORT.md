# 阶段 1 · 可行性确认报告

> 状态:**三道闸门全部实测完成,并完成后续三项验证 + 事实核对。**
> 方法:真起环境。克隆 `objectstack-ai/hotcrm`(公开仓库,当前 `HEAD`,app 版本 2.1.0),本地 `pnpm install && pnpm dev`(端口 4001),分别在 **better-sqlite3** 与 **sqlite-wasm(StackBlitz 同款驱动)** 两种驱动下跑通,用无头 Chromium 登录后驱动真实 UI / API 取证。全部结论均来自实测,不含猜测。
> 关键背景:本仓库 `yinlianghui/objectstack-launch` 只是发布工作区(只有 README + 本报告),**不含任何 hotcrm/ObjectUI 代码**;代码在 `objectstack-ai/*`。

---

## 0. 一句话结论

**环境能起、界面漂亮、种子数据真实,写权限也通 —— 但头号爆点「Wow #1(加字段→AI 用上)」开箱演示不出来:AI 服务连包都没装(`@objectstack/service-ai` 不在依赖里),且必须自备 LLM Key;运行时改元数据还要额外开 `OS_METADATA_WRITABLE`。可 100% 免配置现场演示的是「Executive 仪表盘 + Sales Pipeline 看板(51 条种子商机)」这类静态但极上镜的界面。** 详见分级结论。

---

## 三道闸门

### 闸门 A · 写权限 —— 🟢 绿灯(已验证)
- 新建 `assets/phase1/.gitkeep`,commit `054f661` 已成功 `push` 到 `claude/objectstack-phase1-verify-wfnrwl`。**写权限 OK,无需再动 GitHub App 授权。**

### 闸门 B · 环境起得来 —— 🟢 绿灯(已验证)
- 工具链匹配:Node **v22.22.2**、pnpm **10.33.0**、磁盘充足。`pnpm install` 约 **2 分 18 秒**(含编译 better-sqlite3 原生模块)。`pnpm dev` 起在 **4001**,36 个插件加载完成,`/api/v1/health` = 200。
- 构建产物真实:**15 objects / 307 fields / 17 flows / 11 actions / 2 agents / 6 skills**。
- Dev 登录(日志打印):`admin@objectos.ai` / `admin123`(应用需登录,非自动进)。
- 种子数据加载正常并已截图取证:
  - `03-dashboard.png` — Executive Overview:YTD 营收 2,528,600、营收趋势面积图、行业分布环形图、KPI(5 Accounts / 5 Contacts / 21 Leads)。
  - `04-pipeline-kanban.png` — Sales Pipeline 看板,**51 条真实商机**分列 PROSPECTING/QUALIFICATION/NEEDS ANALYSIS/PROPOSAL(Globex、Acme、Initech、Wayne 等)。
  - `08-case-record.png`(CASE-00003 服务工单)、`11-lead-new-field.png`(Lead 记录页)等。
- ⚠️ 说明:**未走 StackBlitz**。StackBlitz 是浏览器内云 IDE,本无头环境无法真正驱动它跑 dev server;改用等价的本地 `pnpm dev`,并额外用 `OS_DATABASE_DRIVER=sqlite-wasm` 复现了 StackBlitz 的 WASM 驱动路径(见 R2)。

### 闸门 C · AI 能不能出效果 —— 🔴 红灯 / 需自备 Key + 装包(已验证,如实)
不配 `OPENAI_API_KEY` 时,Sales Copilot 的真实表现(实测):
- `GET/POST /api/v1/ai/chat` → **404 `{"error":{"message":"AI service is not configured"}}`**
- `GET /api/v1/ai/agents` → 200 但 **`{"agents":[]}`**(agent 根本没挂上)
- **没有内置的免 Key demo/mock 交互模式。** 唯一的 "mock" 是 `mock_llm:'echo'`,那只是 `@objectstack/runtime/ai/testing` 的**单元测试**确定性桩,不接入 chat UI。
- **更硬的坑:`@objectstack/service-ai` 根本没被 hotcrm 列为依赖**(不在 `package.json`、不在 lockfile、node_modules 里没有)。所以**即使配了 Key,当前这份 clone 也加载不出 AI 服务** —— CLI `importFromHost('@objectstack/service-ai')` 直接失败。

**要让 AI 出效果的最简步骤(诚实版上手话术素材):**
1. 给 hotcrm 补依赖:`pnpm add @objectstack/service-ai`。⚠️ registry 上最新是 **10.3.0**,而 hotcrm 其它 `@objectstack/*` 都是 **^14.7.0** —— **主版本落后,存在兼容风险,需先验证能装能跑**。
2. 提供 LLM Provider Key(运行时自动检测):环境变量 `OPENAI_API_KEY`(或 `ANTHROPIC_API_KEY` / `GOOGLE_GENERATIVE_AI_API_KEY` / `AI_GATEWAY_MODEL`);**或**在 Setup → Settings 里填(加密存储)。两个 agent 都写死 `provider: 'openai', model: 'gpt-4'`,所以走 `OPENAI_API_KEY` 最省事(换 provider 要改 `*.agent.ts`)。

> 结论:**核心爆点的 AI 半边无法零配置开箱复现**,社区 clone 后不装包 + 不配 Key 一定看不到 Copilot。

---

## 后续验证

### 1. 头号爆点「Wow #1」实测 —— ⚠️ 底层机制为真,但开箱演示不出来
按 `live-data.skill.ts` 思路真操作:给 `crm_lead` 加自定义字段 `pain_point`(改元数据)→ 看 AI 是否认到并用上。

- **"活元数据"底层机制是真的(免 Key 可证):** 开启 `OS_METADATA_WRITABLE=object,field` 后,通过元数据 API `PUT /api/v1/meta/object/crm_lead` 加字段 **成功(HTTP 200,~47–74ms)**,紧接着 `describe_object` 等价的 `GET meta` **~18ms** 即读到新字段(`present: true`)。即"改一处 schema、秒级 live 可见"机制成立、延迟够短、够上镜。
- **但三个前置坑决定它开箱拍不出来:**
  1. **运行时加字段默认被禁**:不设 `OS_METADATA_WRITABLE` 时,加字段 → **403 `[not_overridable] ... allowOrgOverride=false`**。hotcrm 的对象是"代码包提供"的,默认不允许运行时改元数据(与"运营免部署加字段"的叙事直接冲突,必须显式开环境变量才行)。
  2. **AI 消费半边跑不了**:如闸门 C,没 Key + 没 `service-ai` 包,"让 Copilot 按新字段查询/汇总"这一步演不了。
  3. **新字段不会自动出现在 UI 记录页**:字段进了 schema(AI/数据层可见),但记录页布局由 view/page 元数据决定,**不自动收录新字段**(实测 `11-lead-new-field.png` 展开"空字段"后仍无 Pain Point)。
- **判定:Wow #1 按脚本(加字段→AI 用上)属于 ❌ 开箱演示不出来 / ⚠️ 强条件**。要拍必须同时满足:装 `service-ai`(且解决版本兼容)+ 自备 OpenAI Key + 开 `OS_METADATA_WRITABLE`。

### 2. 演示避坑(R2)· action 沙箱白名单 —— 已在 sqlite-wasm 下逐个实测
调用契约:`POST /api/v1/actions/{objectName|global}/{action}`,body `{recordId, params}`(action 体里的 `input` = `params`)。在 **sqlite-wasm(QuickJS WASM 沙箱)** 下逐个实测:

**✅ 安全可上镜(实测成功 200):**
- `mark_primary`(标记主联系人)
- `send_email`(发邮件 / 写活动)
- `log_call`(记录通话)
- `log_meeting`(记录会议)

**❌ / ⚠️ 避开(实测失败):**
- `clone_opportunity` —— **ValidationError:crm_account / amount / close_date 必填**。注:代码里"已知会崩"的 emscripten 崩溃**已被 workaround 规避**(现在不崩了),但 workaround 本身功能坏了(最小 insert 违反必填约束),**仍会报错、不可上镜**。
- `escalate_case` / `close_case` / `mass_update_stage` —— **`FORBIDDEN: insufficient privileges to update`**。经交叉验证:dev admin 直接用数据 API 更新这些对象**都能成功(200)**,所以这是 **action 沙箱执行主体特有的权限门**(不是 admin 缺权限),开箱状态下这三个动作会当场报错。
- `create_campaign` —— ValidationError(crm_campaign 必填),需正确的 campaign 关联,风险高。
- `export_csv` —— 唯一用裸 `ctx.api.object(x).find()`(即"marshalling certain row shapes"崩溃模式)的动作;经 API 路径其 `objectName` 被 URL 段强制成 `global`,无法直接跑到真实表,**wasm 下的 marshalling 崩溃风险未能证伪,建议避开**。

> 补充:本轮测试中**没有复现任何 emscripten/QuickJS 硬崩溃**;文档点名的 `CloneOpportunityAction` 崩溃在当前源码里已被 workaround 替换成"校验报错"。演示脚本请只用上面 4 个 ✅ 动作。

### 3. ObjectUI 自定义 React 组件「逃生舱」查证 —— 没找到实锤
查了 hotcrm 文档(`ui-extensions.mdx` 只讲 action/view/page/dashboard,无自定义组件入口)、`src/interfaces/`(空)、以及已安装的 ObjectUI/console 框架包:
- **没有**找到"写并注册你自己的 React 组件"(类 Salesforce LWC)的一等公民机制/API/文档。
- 找到的相关物,都不是那回事:
  - `<Block type="...">` —— "render any **registered** component by type",是在声明式目录里调**内置已注册**组件的逃生舱,不是自定义代码。
  - `customCss` —— 原始 CSS 逃生舱,官方标注 **"Discouraged"**。
  - `customComponents` —— 一个能力**标志位(默认 false**,描述 "Supports custom UI components/widgets"),只说明"平台层面存在这个概念",但**没有任何"怎么注册自定义 React 组件"的机制或文档**。
- **判定:无实锤 → 素材里不要写"需要时也能插自定义 React 组件"这句。**

---

## 事实核对结论

- **License:真冲突,未解决,素材暂不写具体 license 名。**
  - `LICENSE` 文件 = **MIT**(Copyright 2024 HotCRM Team)。
  - README 徽章 + **源码文件头 128/128 全部 = Apache-2.0**(Copyright 2025 ObjectStack)。
  - 二者法律上实质不同(Apache-2.0 有专利授权 / NOTICE 要求)。GitHub 的 license 探测以顶层 `LICENSE` 文件为准(即会显示 MIT),但代码头一致声称 Apache-2.0 —— **这是真实矛盾,必须由维护者拍板**,确认前所有素材不出现具体 license 名。
- **版本号:以 `2.1.0` 为准。**
  - `package.json` = `objectstack.config.ts`(app.version)= `CHANGELOG` 顶栏 = **2.1.0**(2026-07-14),三处一致,是权威值。
  - README 徽章 `1.0.0` 是**过时**的,应改。(阶段 0 记的 `2.0.0` 是上一版,现已升到 2.1.0。)
  - 注:`/api/v1/health` 返回的 `"version":"1.0.0"` 是平台/API 版本,非应用版本,别混。
- **端口:4001**(package.json dev 脚本 + README + 实测一致)。
- **数量口径(以构建输出为准):** 15 objects / 307 fields / 17 flows / 11 actions / 2 agents / 6 skills。README 写的 "10 flows / 10 actions" 与构建输出(17 / 11)有出入,以构建为准。

---

## 钩子场景最终建议

1. **若能接受"诚实版有条件"** → 仍可用 **Wow #1(活元数据)**,但脚本必须诚实标注前置:装 `@objectstack/service-ai`(先解决 10.x vs 14.x 版本兼容)+ 自备 `OPENAI_API_KEY` + 开 `OS_METADATA_WRITABLE`。这条能拍出"改一处 schema、秒级生效"的科幻感,但**不是 clone 就能复现**,病毒传播力打折。
2. **若要 100% 免配置、clone 即复现的现场爆点** → 建议主推**已验证、零配置、极上镜**的静态界面:**Executive 仪表盘 + Sales Pipeline 看板(51 条种子商机)**(证据 `03`、`04`)。免 Key、免装包、免环境变量,今天就能拍。
3. **一个折中的"开发者向"免 Key 证明** → 只拍 Wow #1 的**元数据层**:`PUT` 加字段 → `describe_object` ~20ms 读到新字段(纯 API/终端演示),证明"schema 是活的",规避 AI Key 依赖。科幻感弱于完整版,但真实、可复现。

> 演示动作只用白名单四件:`mark_primary` / `send_email` / `log_call` / `log_meeting`。避开 `clone_opportunity`、`escalate_case`、`close_case`、`mass_update_stage`、`create_campaign`、`export_csv`。

---

## 分级汇总

| 项 | 结论 | 证据 / 条件 |
|---|---|---|
| ✅ 写权限 | 已验证 | commit `054f661` 已 push |
| ✅ 环境 + 种子界面 | 已验证,可现场演示 | `03-dashboard.png`、`04-pipeline-kanban.png`(51 条) |
| ✅ 安全 action 4 件 | 已验证 | sqlite-wasm 下 200 成功 |
| ✅ 活元数据机制(API 层) | 已验证 | PUT 加字段 200 ~50ms,describe ~18ms |
| ⚠️ Wow #1 完整版(加字段→AI 用) | 有条件 | 需 service-ai 包 + Key + `OS_METADATA_WRITABLE` |
| ⚠️ 运行时改元数据 | 有条件 | 默认 403,须开 `OS_METADATA_WRITABLE` |
| ❌ AI Copilot 开箱 | 演示不出 | 无包 + 无 Key → 404 `AI service is not configured` |
| ❌ 危险 action | 演示不出 | clone/escalate/close/mass_update/create_campaign/export_csv 均报错 |
| ❌ 自定义 React 组件逃生舱 | 无实锤,不写 | 仅有 `<Block>`/`customCss`/能力标志位 |

**报告完，按流程停下,等待人类确认后再进阶段 2。**
