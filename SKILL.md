---
name: just-do-it
description: >
  以「我」为中心的行动引擎。接收任意输入（会议纪要、逐字稿、口述待办、聊天记录等），
  按固定五步流程处理：⓪ 元意图识别 → ① 文档上下文扫描 → ② 拆出完整 todo 列表
  → ③ 识别跟我有关的 → ④ 模糊的询问我确认 → ⑤ 我的事：能做直接做掉，不能做帮我记待办并提醒。
  触发词：just do it、帮我整理一下、处理这个会议纪要、帮我把跟我有关的做了、
  我接下来要做什么、会议纪要、逐字稿、待办清单、整理一下这些事情。
---

# Just Do It · 以「我」为中心的行动引擎

> 五步走：元意图判断 → 文档扫描建立上下文 → 拆 → 识别 → 确认 → 做完或记下来

---

## 第零步：元意图识别（最优先执行）

**在做任何事之前，先判断用户的输入属于哪种模式：**

| 模式 | 判断依据 | 处理方式 |
|------|---------|---------|
| **执行模式** | 输入是待处理的内容（纪要、口述、聊天记录等） | 进入正常五步流程 |
| **讨论模式** | 输入是在谈论/评价/改进这个 skill 本身 | 切换到「skill 讨论模式」，不拆 todo |
| **查询模式** | 用户在问「我接下来要做什么」「有什么待办」 | 读取 hi-todos，汇总展示 |

**判断「讨论模式」的信号：**
- 主语是「just do it」「这个 skill」「你」（指 Agent）
- 含有「改进」「问题」「建议」「缺少」「应该」等评价词
- 内容是对功能/行为的反馈，而不是需要执行的事项

**讨论模式下的处理：**
```
1. 明确告知：「我理解你是在讨论 just-do-it 的改进，而不是在给我任务」
2. 整理用户的反馈点
3. 询问：「要我直接修改 skill 吗？还是先讨论一下？」
```

> ⚠️ 绝对不要把「对 skill 的评价」当成 todo 拆解。

---

## 谁是「我」（运行时动态获取，三级短路）

**按优先级短路获取，命中即停，不继续往下走：**

### 优先级 1：环境变量（最快，零成本）
```bash
# 直接读取，无需任何调用
XHS_USER_EMAIL       → email / emailPrefix
XHS_USER_NAME        → userName
XHS_USER_RED_NAME    → redName
```
如果以上变量都有值 → **直接使用，跳过 2 和 3**

### 优先级 2：Memory 文件缓存（快，无网络）
```
~/.openclaw/workspace/skills/just-do-it/memory/preferences.json
→ 读取 identity 字段：{ email, userName, redName, emailPrefix }
```
如果 identity 字段完整且 `identity.fetchedAt` 在 24 小时内 → **直接使用，跳过 3**

### 优先级 3：CLI 实时查询（最慢，冷启动兜底）
```bash
# 仅在以上两级都未命中时才执行
pnpm dlx @xhs/hi-workspace-cli@0.2.5 search:me
# → { "xhsContactId": "xxx@xiaohongshu.com" }

pnpm dlx @xhs/hi-workspace-cli@0.2.5 search:employee --query "<上一步邮箱>"
# → { userName, redName, accountId, ... }
```
查询成功后 → **立即写入 Memory 文件**（供下次优先级 2 命中）：
```json
{
  "identity": {
    "email": "xxx@xiaohongshu.com",
    "emailPrefix": "xxx",
    "userName": "<真实姓名>",
    "redName": "<花名>",
    "fetchedAt": "<ISO timestamp>"
  }
}
```

### 兜底
如果三级都失败 → 询问用户「你的花名或邮箱是什么？」手动填入并缓存。

---

**「我」的识别字段（无论哪级获取到）：**
- `email`：完整邮箱（如 `xxx@xiaohongshu.com`）
- `userName`：中文全名（如 `<真实姓名>`）
- `redName`：薯名/花名（如 `<花名>`）
- `emailPrefix`：邮箱 @ 前缀（如 `<邮箱前缀>`）

**识别「跟我有关」时，匹配以上所有字段。**

---

## 流程总览

```
输入任意内容
    ↓
【第零步】元意图识别 → 是在谈 skill？→ 讨论模式（不继续）
    ↓（执行模式）
【第一步】文档上下文扫描（可选，视输入复杂度决定）
    ↓
【第二步】拆出完整 Todo 列表
    ↓
【第二·五步】执行前展示清单 → 等用户说「开始」或「▶」
    ↓
【第三步】识别「跟我有关的」（结合文档上下文）
    ↓
【第四步】模糊的问我（y/n）
    ↓
【第五步】我的 todo → 能做就做，不能做就帮我记待办
```

---

## 第一步：文档上下文扫描（建立背景）

**目的：在拆 todo 之前，先了解用户在做什么项目/事情，帮助更准确地判断归属和优先级。**

### 何时触发扫描

**必须扫描：**
- 输入是会议纪要或逐字稿（需要理解会议背景）
- 输入涉及多个项目或模糊的业务方向
- 用户说「你先了解一下我在做什么」

**可以跳过：**
- 口述待办（主语明确是「我」）
- 单条简单任务
- 用户明确说「不用扫描，直接处理」

### 扫描流程

```
1. 调用 redoc-enhanced 或 hi-search 获取用户有权限的文档空间列表
   - 超时限制：10 秒；超时则跳过扫描，继续主流程（不阻塞）
   - 文档数量上限：取最近更新的 50 篇，超出部分忽略
2. 对文档按以下维度分类：
   - 项目：**动态归纳**，从文档标题/内容中提取高频关键词聚类，不预设项目名
     （如标题含「招呼词」「宠物」的聚成一组，含「数据看板」「DAU」的聚成另一组）
   - 类型：从文档标题/结构推断（PRD / 会议纪要 / 数据分析 / 方案 / 周报 / 其他）
   - 活跃度：按最后更新时间排序（最近 7 天 / 7-30 天 / 30 天以上）
3. 建立「当前用户上下文」：
   {
     "activeProjects": ["<从文档归纳出的项目名>", "<另一个>"],
     "recentDocs": [
       { "title": "<文档标题>", "inferredProject": "<归纳出的项目>", "inferredType": "<类型>" },
       { "title": "<文档标题>", "inferredProject": "<归纳出的项目>", "inferredType": "<类型>" }
     ],
     "inferredFocus": "<一句话总结用户当前主要在做什么，从文档内容归纳>"
   }
4. 将上下文注入后续步骤，辅助判断
5. 扫描结果缓存到当次会话（同一对话不重复扫描）
```

**降级处理：**
- 超时（> 10s）→ 提示「文档扫描超时，将跳过上下文建立」，继续执行
- 文档数 > 50 → 只取最近 50 篇，提示「仅分析最近 50 篇文档」
- 扫描报错 → 静默跳过，不影响主流程

### 上下文如何影响后续步骤

- **拆 todo 时**：参考活跃项目，识别模糊业务词（如「招呼词」「助手」）属于哪个项目
- **识别「我的」时**：如果 todo 涉及活跃项目，归属概率更高
- **路由时**：发现相关文档 → 直接关联（「这个 todo 可能对应你的招呼词PRD，要更新进去吗？」）
- **针对性建议**：结合文档内容，给出更具体的推进建议而非泛泛之词

> 扫描结果缓存到当次会话，同一次对话中不重复扫描。

---

## 第二步：拆出完整 Todo 列表

从输入中提取**所有**有行动意义的事项，输出格式：

```
【完整 Todo 列表】（{N} 条）

1. {事项描述} | 负责人：{姓名 / 未指定} | 截止：{时间 / 未提及}
2. {事项描述} | 负责人：{姓名 / 未指定} | 截止：{时间 / 未提及}
...
```

提取原则：
- 宁多勿少，不过滤
- 每条是一个独立、可执行的动作
- 保留原文中的负责人、截止时间信息

---

## 第二·五步：执行前展示清单（等用户确认）

拆完 todo 后，**先展示再执行**，格式参见 `TODO_LIST_FORMAT.md`。

核心要点：
- 按「可直接执行 / 需确认 / 需你推进」三类分组展示
- 末尾有「开始执行 ▶」提示，**等用户回复后才进入第三步**
- 用户可以在此删除/修改某条 todo，agent 按修改后的列表执行
- 如果用户回复「全部执行」/「go」/「开始」/「▶」等 → 进入第三步
- 如果用户只确认部分 → 只处理被确认的条目

> ⚠️ 不要在展示清单后立即执行，必须等用户回复。

---

## 第三步：识别「跟我有关的」

遍历完整 Todo 列表，判断每条是否跟 {redName}（运行时获取）有关：

**明确是我的（直接归入 A 类）：**
- 负责人字段包含上面获取到的任意一个身份字段（userName、redName、emailPrefix、完整 email）
- 原文中动作主语是我（「{redName} 你来…」「我来跟进…」「{userName} 负责…」）
- 口述输入中主语是「我」（「我要去做…」「我需要…」）

**不确定是不是我的（归入 B 类，待确认）：**
- 负责人未指定，但内容与我的工作方向相关
- 模糊表述（「产品侧跟进」「PM 确认」等）
- 涉及我所在团队但未点名
- **【新】** 文档上下文中涉及我的活跃项目，但未明确点名

**明确不是我的（归入 C 类）：**
- 负责人明确是其他人

---

## 第四步：询问 B 类（模糊的）

对每条 B 类 todo，简洁询问：

```
❓ 这几条可能跟你有关，确认一下：

1. 「产品侧跟进招呼词上线效果」→ 是你来做吗？
2. 「对齐下次汇报时间」→ 你来牵头吗？

直接回「1是2否」或全部是/否
```

收到确认后：
- 说「是」的 → 加入 A 类，进入第五步
- 说「否」的 → 加入 C 类，整理进分享清单

---

## 第五步：处理 A 类（我的 todo）

对每条确定是我的 todo，做两层判断：

### 5a. 能做 → 直接做（结合文档上下文给针对性帮助）

**执行前先检查文档上下文，让帮助更有针对性：**

```
如果 todo 涉及某个项目/功能，且文档上下文中有相关文档：
  → 主动关联：「找到了你的招呼词PRD，这个 todo 要更新进去吗？」
  → 用相关文档中的背景信息丰富执行内容
    （如写纪要时，自动带入 PRD 中的功能描述作为背景）
  → 给出更具体的推进建议
    （「这个功能的 Owner 是图图，建议先发给她确认」）

如果没有找到相关文档：
  → 按通用路由执行，不强行关联
```

按以下决策树路由：

> ⚡ **路由入口优先判断**：先检查 todo 是否属于 MCP 直接路由范围（见下方 MCP 路由表）。命中则走 MCP，不走后续路由；未命中再按下方顺序匹配。

**【MCP 直接路由】（最优先，命中即走，不继续往下）**

> 以下工具通过 `codewiz.jsonc` 中已配置的 MCP server 直接调用，无需走 hi-docs / project-management 中转。

| todo 类型 | MCP server | 说明 |
|-----------|-----------|------|
| PingCode 任务 / 工单 / 迭代 | `pingcode-toolkit` | 创建/查询任务、关联迭代、更新状态 |
| 工作流触发 / 流水线 | `workflow-toolkit` | 触发 CI/CD、查看流水线状态 |
| MR Review / 代码审查 | `mr-review` | 拉取 MR 列表、查看 diff、添加评论 |
| 代码库查询 / 搜索 | `codebase` | 搜索代码、查看文件、分析依赖 |

```
MCP 调用失败（报错 / 超时）→ 降级到下方对应路由
```

---

**约会议 / 找时间 → `hi-calendar` + `hi-search`**
```
解析参会人 → hi-search 查 contactId
→ calendar:get-user-schedules 查空闲
→ 列 2-3 个候选时间给我选
→ 我选择 → calendar:create 建日程
```

**出文档 / 写纪要 / 整理方案 → `hi-docs` / `redoc-enhanced`**
```
判断文档类型（纪要/PRD/数据需求说明）
→ 选模板（见 DOC_TEMPLATES.md）
→ hi-docs:create 生成 RedDoc
→ 返回文档链接
```
> 若输入是腾讯会议链接：用 project-management:get_transcript.py 先拿转写

**拉数据 / 看数据 → `bi-data-fetch` / `sql-development`**
```
有 RedBI 链接 → bi-data-fetch 直接取数，展示预览
有指标/表名 → sql-development 生成并执行 SQL
什么都没有 → 生成数据需求说明文档（RedDoc）
```

**发消息给某人 / 某个群 → `project-management` API**
```
展示消息内容预览 → 我确认 → 发送
个人私信末尾加签名：—— {redName}（用运行时获取的薯名）
```

**做 PPT / 可视化图 → `live-data-ppt` / `xhs-viz-page` / `pptx`**
```
有数据 → live-data-ppt
有 RedDoc 文稿 → redoc-ppt-studio
工作材料 → xhs-viz-page
通用 → pptx
```

**同步项目进展到群 → `project-management`**
```
解析项目名 → 查项目配置
→ 读最新周会文档 → 生成进展摘要
→ 展示预览 → 确认 → 发群
```

**找数据 / 不知道数据在哪 → `bi-assets-discovery` → `bi-data-fetch`**
```
自然语言描述需求 → bi-assets-discovery 定位看板/数据集
→ 用户确认 → bi-data-fetch 正式取数
→ 需要理解指标口径 → redbi-metrics 提取指标定义
```

**查行业数据 / 笔记洞察 / 竞品 → `idea-data-skill`**
```
行业趋势 / 搜索分析 / 人群资产 / 品牌情感 → 灵犀全量数据
人群定向投放 → linxi-crowd-finder
```

**读写 Excel / Word → `xlsx` / `docx`**
```
输入或输出是 .xlsx → xlsx
输入或输出是 .docx → docx
```

**绘制图表 / 架构图 → `ai-drawing-assistant` / `allin-design-image-generate`**
```
流程图/时序图/思维导图/架构图 → ai-drawing-assistant（Drawio）
宣传图/PPT配图/设计图 → allin-design-image-generate（ALLIN平台）
```

**会议室预订 / Focus Time / 节假日查询 → `calendar`**
```
与 hi-calendar 的区别：
- hi-calendar → hi 系统日历（约人、建日程、查个人空闲）
- calendar → 红薯日历（会议室预订、节假日、Focus Time）
```

**宠物行业专项 → `pet-newproduct-monitor` / `brand-review-petfood` / `xhs-xiaohongxing-review`**
```
宠物新品动态 → pet-newproduct-monitor
品牌营销复盘 → brand-review-petfood
广告投放复盘 → xhs-xiaohongxing-review
```

**数据完整链路（不知道数据在哪时）**
```
自然语言描述需求
  → bi-assets-discovery 找看板/数据集
  → 用户确认
  → bi-data-fetch 取数（产出 data_ref）
  → data-process 清洗/聚合/转换（可选）
  → report-generation 生成报告 / data-visualization 生成图表
需要了解指标口径 → redbi-metrics
需要直接找底表 → table-discovery → sql-development
需要日报/周报数据 → redbi-report-day-or-week
```

**实验分析完整链路**
```
查实验配置 → experiment-info-discovery
分析实验结果 → experiment_result_analysis
多方案赛马 → racing
判断显著性 → racing-significance
```

**用户行为分析**
```
找埋点 → event-point-discovery
分析行为路径 → event-behavior-analysis
追踪特定用户 → user-behavior-trace
生成用户画像 → xhs-one-persona-agent
```

**报告 & PPT 生产**
```
有数据 → data-visualization（图表）→ report-generation（报告）
数据→动态PPT → live-data-ppt
RedDoc文稿→PPT → redoc-ppt-studio / nanobanana-ppt-skills
已有PPT换模板 → pptx-reskin
```

**研发工具（代码/工程类 todo）**
```
搜代码 → code-search
看代码/函数 → code-view / code-context
MR Review → yunxiao-mr-review（云效）
流水线操作 → pipeline-manage
查/建 PingCode 工作项 → pingcode-query-workitem / pingcode-create-workitem
查线上事件/告警 → xhs-event-query
```

**文档搜索 & 在线表格**
```
找 RedDoc 文档 → redoc-search
读写在线表格 → redoc-sheet
```

**通用工具**
```
查天气 → weather
查 HR / 假期 / 工资 → employee-self-service
内容灵感 → xhs-content-inspiration
PRD → 原型图 → prd-to-prototype
摘要/新闻 → summarize / news-summary
周报自动生成 → weekly-report-writer
```

### 5b. 不能做 → 帮我记待办 + 标注来源

**什么是「不能做」：**
- 需要登录外部系统（微信、钉钉等）
- 需要物理操作
- 需要专业技能（如写代码、部署上线）
- 需要人工决策或审批

**处理方式：**
```
1. hi-todos:create 创建一条待办
   - 事项：{原始描述}
   - 来源：来自「{输入来源}」的 Just Do It 整理
   - 截止：{若有}
2. 告知用户：
   「「{事项}」我做不了，已帮你记了一个待办，
     会在截止前提醒你。
     推进建议：{一句话说下一步找谁 / 做什么}」
```

**待办创建完必须给我一个链接或确认，确保没有丢失。**

---

## 执行节奏

**立即执行（不问我）：**
- 给自己创建待办（包括「不能做」的记录）
- 生成文档草稿
- 拉取数据（只读）

**展示后等我确认：**
- 给他人发消息
- 发群消息
- 约会议（列候选时间让我选）

**并行执行：** A 类多条 todo 同时推进，不串行等待

---

## 最终输出

所有步骤完成后，输出结果卡：

```
✅ Just Did It · {来源} · {日期}

━━━━━━━━━━━━━━━━━━━━━━━━━━━
🙋 你的事 · 已做完（{N} 项）

  🗓️ 和图图的对齐会
     → 候选时间 1. 4/8(周二)10:00  2. 4/9(周三)14:00
     → 请选 1 或 2

  📄 会议纪要文档
     → https://docs.xiaohongshu.com/doc/xxx

━━━━━━━━━━━━━━━━━━━━━━━━━━━
📝 你的事 · 已记待办，等你来做（{N} 项）

  · 「招呼词功能上线」
    → 待办已创建 ✓，推进建议：在群里 @ 图图确认排期

━━━━━━━━━━━━━━━━━━━━━━━━━━━
📋 其他人的 Todo（{N} 条，整理好方便分享）

  图图：招呼词优化上线（截止4/10）、整理A/B测试数据
  小明：后端接口联调（本周内）
  阿珍：效果文档（截止4/13）

  要把某份直接发给对应同学吗？

━━━━━━━━━━━━━━━━━━━━━━━━━━━
没有遗漏的事情了 🏁
```

---

## 模块五：Memory 系统（用户行为学习）

### 设计原则
每次执行结束时，把本次的关键信息写进记忆文件，**下次执行前先读记忆**，用历史偏好指导当前决策，越用越懂你。

### 记忆文件位置
安装后首次运行时自动创建：
`~/.openclaw/workspace/skills/just-do-it/memory/preferences.json`
`~/.openclaw/workspace/skills/just-do-it/memory/skill_versions.json`

### 记忆结构

```json
{
  "lastUpdated": "2026-04-04T15:30:00+08:00",
  "identity": {
    "email": "xxx@xiaohongshu.com",
    "emailPrefix": "xxx",
    "userName": "<真实姓名>",
    "redName": "<花名>",
    "fetchedAt": "2026-04-06T10:00:00+08:00"
  },
  "preferences": {
    "meetingDuration": 60,
    "meetingTimeRange": { "start": "10:00", "end": "18:00" },
    "preferredMeetingSlots": ["上午10-12点", "下午16-18点"],
    "avoidSlots": ["午饭前后11:30-13:30"],
    "defaultNotifyAssignee": true,
    "preferDocType": {
      "会议纪要": "hi-docs",
      "方案": "hi-docs",
      "数据需求": "hi-docs"
    }
  },
  "corrections": [],
  "frequentContacts": [],
  "todoPatterns": [],
  "executionHistory": []
}
```

> `identity.fetchedAt` 超过 24 小时则重新查询（走优先级 3 CLI）。写入时统一放 `identity` 对象内，不要放顶层。

### 执行时如何使用记忆

**执行前读取记忆：**
1. 读取 `preferences.json`
2. 用 `preferences` 调整决策：
   - 约会议 → 优先推荐 `preferredMeetingSlots` 中的时段
   - 时长 → 用 `meetingDuration` 作为默认值
   - 联系人 → `frequentContacts` 中的人，直接用记忆里的邮箱，不重新搜索
3. 用 `corrections` 调整行为：如上次用户把14点改成10点，这次直接推上午

**执行后写入记忆：**
```
1. 更新 executionHistory（本次来源、todo数量、执行结果）
2. 更新 frequentContacts（出现过的联系人自动加入）
3. 如果用户修改了我的选择 → 写入 corrections + 提炼 learning
4. 如果用户明确说「以后都这样」→ 写入 preferences
```

### 用户修改如何触发记忆更新

当用户：
- 改了我推荐的时间 → 记录「用户偏好 XX 时段」
- 说「不用通知他」→ 记录「该联系人不需要通知」
- 改了文档标题/格式 → 记录偏好模板风格
- 说「以后都 xxx」→ 直接写入 `preferences`

### 查看/重置记忆

用户说「你记住了什么」→ 输出 `preferences.json` 的人类可读摘要  
用户说「忘掉 xxx」→ 删除对应记忆条目  
用户说「重置记忆」→ 清空 `preferences.json`

---

## 模块六：依赖 Skill 自动进化

### 设计原则
依赖的 skill 会持续更新迭代，just-do-it 需要：
1. **自动检测**依赖 skill 有无新版本
2. **自动更新**（用户确认后）
3. **分析新能力**，把有用的新能力补进自己的路由表
4. 让自己越来越强

### 检测文件
`~/.openclaw/workspace/skills/just-do-it/memory/skill_versions.json`

```json
{
  "lastChecked": "2026-04-04T00:00:00+08:00",
  "deps": [
    { "slug": "hi-calendar", "installedVersion": "1.0.8", "latestVersion": "1.0.8", "hasUpdate": false },
    { "slug": "hi-todos", "installedVersion": "1.0.5", "latestVersion": "1.0.6", "hasUpdate": true, "newFeatures": ["支持设置提醒时间"] },
    { "slug": "project-management", "installedVersion": "1.1.0", "latestVersion": "1.1.0", "hasUpdate": false }
  ]
}
```

### 每日自动检测流程（在心跳或用户触发时执行）

```
1. 读取 skill_versions.json 中的 installedVersion
2. 对每个依赖 skill，运行：
   clawhub info <slug> --json → 获取 latestVersion
3. 对比版本号，找出有更新的 skill
4. 更新 skill_versions.json
5. 如果有更新：
   → 向用户推送：「{N} 个依赖 skill 有新版本，要更新吗？」
   → 列出：skill名 / 当前版本 / 最新版本 / 新功能描述
```

### 用户确认后的更新流程

```
1. clawhub install <slug> 逐个更新
2. 读取更新后的 SKILL.md，提取新增能力描述
3. 分析新能力是否可以扩展 just-do-it 的路由：
   → 新增了某个 API → 是否可以用在新的 todo 类型？
   → 修复了某个 bug → 是否影响现有路由的稳定性？
4. 把有用的新能力写入：
   ROUTE_EXTENSIONS.md（待用新能力记录）
5. 如果新能力明显有用（如 hi-todos 新增了「设置提醒」），
   直接更新 SKILL.md 中对应路由的执行步骤
6. 向用户汇报：「已更新 {N} 个 skill，just-do-it 新增了 {M} 项能力」
```

### 触发方式

**自动触发（心跳检测时）：**
- 每天检测一次，有更新时推送通知

**手动触发：**
- 用户说「检查一下依赖更新」
- 用户说「更新一下你用到的 skill」

### 新能力分析规则

读取更新后的 SKILL.md，提取以下信息：
```
- description 字段的变化（新增了哪些功能点）
- 有没有新的命令/API（新的 CLI 子命令）
- 有没有新的输出字段（可用于新的路由判断）
```

判断是否需要更新 just-do-it 的路由表：
```
如果新能力属于以下类型 → 更新路由：
  - 新增了我还没覆盖的 todo 类型
  - 现有路由可以用新能力做得更好
  - 解决了现有路由的某个已知问题

否则 → 记录在 ROUTE_EXTENSIONS.md 备用
```

---

## 原则

1. **先判意图** —— 输入是在「用」还是在「谈」这个 skill？搞错了一切都白做
2. **五步不乱** —— 必须按顺序：元意图 → 扫描上下文 → 拆全 → 识别 → 确认 → 做
3. **以我为中心** —— 我的事最优先，其他人的整理好即可
4. **不能做的不丢失** —— 一定要帮我记待办，带来源标注
5. **不猜负责人** —— 不确定是不是我的，问清楚再执行
6. **有上下文才有针对性** —— 扫描文档是为了帮忙更准，不是走形式；没有上下文时用通用路由
7. **越用越懂你** —— 每次执行都在学习，记忆驱动下次决策
8. **主动进化** —— 依赖更新时自动分析新能力，扩展自己的路由

---

## 依赖 Skills（38个，按类别）

### 🗓️ 协作 & 沟通（5个）
| skill | 用途 | 触发场景 |
|-------|------|---------|
| `hi-search` | 查员工信息（邮箱/花名/部门） | 需要找某人的联系方式 |
| `hi-calendar` | 查空闲、建日程（hi 日历） | 约会议、找时间 |
| `hi-todos` | 创建/管理待办 | 记录任务、设置提醒 |
| `hi-im` | hi IM 消息/文件上传 | 发文件到 hi 群 |
| `calendar` | 红薯日历（会议室/节假日/Focus Time） | 订会议室、查节假日、设 Focus Time |

### 📄 文档 & 内容（10个）
| skill | 用途 | 触发场景 |
|-------|------|---------|
| `hi-docs` | 创建 RedDoc 文档 | 写纪要、方案、PRD |
| `redoc-enhanced` | 读写 RedDoc 富文本（加粗/颜色/表格/图片） | 编辑已有文档、复杂格式 |
| `redoc-search` | 搜索 RedDoc 文档库 | 找已有文档、检索资料 |
| `redoc-sheet` | RedDoc 表格读写 | 读写在线表格 |
| `summarize` | 内容摘要（URL/文件/视频转写） | 总结文章、会议录音 |
| `news-summary` | 新闻/资讯摘要 | 快速了解行业动态 |
| `weekly-report-writer` | 自动生成周报 | 写周报 |
| `xlsx` | 读写 Excel 文件 | 处理表格数据 |
| `pptx` | 读写 PPT 文件 | 制作/编辑幻灯片 |
| `docx` | 读写 Word 文件 | 制作/编辑 Word 文档 |

### 📊 数据 & 分析（9个）
| skill | 用途 | 触发场景 |
|-------|------|---------|
| `bi-assets-discovery` | RedBI 找数（定位看板/数据集/指标） | 不知道数据在哪 |
| `bi-data-fetch` | RedBI 取数（执行提数） | 已知看板链接/数据集，拉数据 |
| `redbi-metrics` | 提取 RedBI 看板指标定义/口径 | 理解指标口径、NL-to-SQL |
| `redbi-report-day-or-week` | 自动生成日报/周报数据 | 定期数据报告 |
| `table-discovery` | 数据仓库表结构发现 | 找数据表、理解字段含义 |
| `data-process` | 数据清洗/转换/聚合 | 拿到 data_ref 后的二次处理 |
| `sql-development` | 写/跑 SQL | 直接查数据库 |
| `idea-data-skill` | 灵犀全量数据（行业趋势/笔记洞察/人群） | 查行业数据、竞品分析 |
| `linxi-crowd-finder` | 灵犀精准人群寻找 | 营销投放人群定向 |

### 🧪 实验平台（4个）
| skill | 用途 | 触发场景 |
|-------|------|---------|
| `experiment-info-discovery` | 查找实验信息（实验组/对照组/指标） | 查某个实验的配置 |
| `experiment_result_analysis` | 实验结果分析（核心指标对比） | 分析 A/B 实验结论 |
| `racing` | 实验赛马（多版本快速对比） | 多方案赛马评估 |
| `racing-significance` | 实验显著性计算 | 判断实验结果是否显著 |

### 👤 用户行为（4个）
| skill | 用途 | 触发场景 |
|-------|------|---------|
| `event-point-discovery` | 埋点发现（查看有哪些埋点事件） | 找某功能的埋点 |
| `event-behavior-analysis` | 用户行为事件分析 | 分析用户操作路径 |
| `user-behavior-trace` | 用户行为追踪（单用户/群体） | 追踪特定用户行为 |
| `xhs-one-persona-agent` | 用户画像 Agent | 生成用户 persona |

### 📈 报告生产（5个）
| skill | 用途 | 触发场景 |
|-------|------|---------|
| `report-generation` | 自动生成数据报告 | 数据→报告全流程 |
| `data-visualization` | 数据可视化图表 | 数据转图表 |
| `nanobanana-ppt-skills` | nanobanana 模型 PPT 生成 | AI 生成 PPT |
| `pptx-reskin` | PPT 模板换肤 | 替换已有 PPT 底板风格 |
| `live-data-ppt` | 数据→动态交互 PPT | 数据汇报幻灯片 |

### 🛠️ 研发工具（8个）
| skill | 用途 | 触发场景 |
|-------|------|---------|
| `code-search` | 代码搜索 | 在代码库中搜索 |
| `code-view` | 代码查看 | 查看具体文件/函数 |
| `code-context` | 代码上下文理解 | 理解代码依赖关系 |
| `yunxiao-mr-review` | 云效 MR Review | 查看/审查 MR |
| `pipeline-manage` | 流水线管理（触发/查看） | CI/CD 操作 |
| `pingcode-query-workitem` | PingCode 工作项查询 | 查任务/工单状态 |
| `pingcode-create-workitem` | PingCode 创建工作项 | 新建任务/缺陷 |
| `xhs-event-query` | 小红书事件查询 | 查线上事件/告警 |

### 🎨 设计 & 可视化（3个）
| skill | 用途 | 触发场景 |
|-------|------|---------|
| `xhs-viz-page` | 工作材料→可视化长图 | 数据报告配图 |
| `allin-design-image-generate` | ALLIN 平台图片生成（架构图/宣传图） | 生成设计图 |
| `ai-drawing-assistant` | 绘制架构图/流程图/时序图（Drawio） | 技术方案配图 |

### 🔍 业务专项 & 通用（5个）
| skill | 用途 | 触发场景 |
|-------|------|---------|
| `project-management` | 发消息/获取会议转写/项目进展同步 | 发群消息、会议纪要 |
| `xhs-content-inspiration` | 小红书内容灵感 | 内容创作/选题 |
| `prd-to-prototype` | PRD 转原型图 | 需求文档转 UI 原型 |
| `employee-self-service` | HR 自助（查假期/工资/福利） | 查人事信息 |
| `weather` | 天气查询 | 出行/活动天气参考 |

## MCP Servers（codewiz.jsonc 中已配置，直接调用）

| server | 用途 |
|--------|------|
| `pingcode-toolkit` | PingCode 任务 / 工单 / 迭代管理（含查询和创建） |
| `workflow-toolkit` | CI/CD 工作流触发与状态查询 |
| `mr-review` | MR 列表、diff 查看、评论 |
| `codebase` | 代码库搜索、文件查看、依赖分析 |

安装：`bash scripts/install_deps.sh`

---

## 参考文档

- `TODO_LIST_FORMAT.md` · Todo 列表与结果卡格式
- `SHARE_LIST_FORMAT.md` · 其他人 todo 的分享格式
- `DOC_TEMPLATES.md` · 文档模板
- `EXAMPLES.md` · 完整示例
- `FALLBACKS.md` · 失败处理
- `ROUTE_EXTENSIONS.md` · 待用新能力记录（自动维护）
- `memory/preferences.json` · 用户行为记忆（运行时自动创建并维护）
- `memory/skill_versions.json` · 依赖 skill 版本记录（运行时自动创建并维护）
