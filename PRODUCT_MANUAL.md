# AI NPC 聊天 · 产品文档

> 版本：v0.1（存档 schema `version: 11`）
> 生成日期：2026-09-21
> 适用代码：`ai-npc-chat/`（`src/` 下 42 个源文件）
> 文档性质：面向产品 / 设计 / 前后端协作的**完整功能与接口说明**，所有结论均以当前代码实现为准。

---

## 目录

1. [产品概述](#1-产品概述)
2. [信息架构：页面每一部分与上下级关系](#2-信息架构页面每一部分与上下级关系)
3. [产品需求与痛点](#3-产品需求与痛点)
4. [数据模型（核心实体）](#4-数据模型核心实体)
5. [接口与输入输出总览](#5-接口与输入输出总览)
6. [按板块的功能文档](#6-按板块的功能文档)
7. [核心流程时序](#7-核心流程时序)
8. [常量、阈值与默认值速查](#8-常量阈值与默认值速查)
9. [非功能需求、边界与已知问题](#9-非功能需求边界与已知问题)

---

## 1. 产品概述

### 1.1 一句话定位

开源、可自托管、**自带密钥（BYOK）** 的 AI 角色扮演与多智能体社交模拟平台：不是「你和一个角色聊天」，而是**一群 NPC 在你选定的世界观里活着，你走了进去**。

### 1.2 三大支柱

| 支柱 | 没有它会怎样 | 本项目怎么做 |
| --- | --- | --- |
| 人物卡 | NPC 是复读机 | 兼容社区 Character Card V2，并扩展社交六维参数、目标、日程、角色详情五段式 |
| 长记忆 | 每次都是初见 | 三层记忆：滚动摘要 + 结构化事实卡 + 遗忘衰减检索 |
| 社交模拟 | 多 NPC 只是排队发言 | 发言调度器决定「下一个谁说话」；关系图谱、好感、恋爱让世界自己演化 |

### 1.3 技术形态

| 项 | 值 |
| --- | --- |
| 类型 | 纯前端 SPA，**无后端、无服务端数据库** |
| 技术栈 | React 18 + TypeScript 5 + Zustand 4（`persist`）+ Tailwind 3 + Vite 5 |
| 运行时依赖 | 仅 `react` / `react-dom` / `zustand`（`src/core/` 零第三方依赖，只用 `fetch`、`Blob`、`FileReader`、canvas） |
| 数据落点 | 浏览器 `localStorage`（key：`ai-npc-chat`），导出为 JSON 文件 |
| 网络出口 | 浏览器直连玩家自己填写的模型服务 Base URL（OpenAI 兼容 / Anthropic / Ollama） |
| 启动 | `npm install && npm run dev` → http://localhost:5173 ；`npm run build` 产出纯静态 `dist/` |

### 1.4 内置内容

- 内置且仅内置一包官方世界观：**巫师 3 · 狂猎**（`id: 'witcher3'`，低魔奇幻 / 北方战争 / 道德灰区）。
- 内含：9 位可聊天角色（杰洛特 / 叶奈法 / 特莉丝 / 希里 / 维瑟米尔 / 丹德里恩 / 卓尔坦 / 血腥男爵 / 艾瑞汀）、12 股势力（各带自己的通行货币与物价）、设定集七件套、12 条世界书、4 个时段场景、10 个随机事件、开局钩子。
- 世界观只是内容，不是产品：可编辑、另存为副本（Fork）、导出、删除。

---

## 2. 信息架构：页面每一部分与上下级关系

### 2.1 顶层结构树

```
└─ 应用根 App.tsx  (flex h-full)
   │
   ├─ 顶部横幅（条件渲染，位于主工作区上方）
   │   ├─ 错误横幅   ⚠ {error} + 「知道了」          ← error !== null
   │   └─ 缺 Key 横幅 ⚠ 还没有配置 API Key + 「去设置」 ← 当前 Provider 无 apiKey
   │
   ├─ 左侧栏 Sidebar (w-60)                        【常驻，任何板块都在】
   │   ├─ ① 世界选择头
   │   │    ├─ 标题 「💬 AI NPC 聊天」
   │   │    ├─ 世界观下拉 <select>  （官方的带「（官方）」后缀）
   │   │    └─ 按钮 「＋ 创建新世界」
   │   │
   │   ├─ ② 板块列表 <nav>（可 ▲▼ 微调 / 长按 300ms 拖拽排序）
   │   │    ├─ 🌍 世界观      → View: world
   │   │    ├─ 🧭 旅人        → View: traveler
   │   │    ├─ 🎭 角色        → View: characters
   │   │    ├─ 🕸️ 关系        → View: relations
   │   │    ├─ 📖 日志        → View: journal
   │   │    └─ 📜 规则        → View: rules
   │   │
   │   ├─ ③ 会话区（flex-1，滚动）
   │   │    ├─ 「💬 聊天」入口  +  「＋」→ NewChatMenu
   │   │    │                              ├─ 🍺 群聊      → newSession()
   │   │    │                              ├─ 🤫 私聊      → 选 1 人 → startPrivate()
   │   │    │                              └─ 👥 自己建群   → 选 ≥2 人 → startGroup()
   │   │    └─ 会话条目[]（头像 + 标题 + 「私聊」小标 + hover ✕ 删除）
   │   │
   │   └─ ④ 固定入口（永远最底，不参与排序）
   │        └─ ⚙️ 设置 → View: settings
   │
   └─ 主工作区 <main>（按 activeView 二选一渲染）
       │
       ├─ chat  ── ChatView
       │    ├─ 左列（flex-1）
       │    │   ├─ 场景栏
       │    │   │   ├─ 会话名 + 「修改」（改名）
       │    │   │   ├─ 标签：🤫 私聊 / 🍺 酒馆
       │    │   │   ├─ 时段切换：清晨 / 正午 / 黄昏 / 深夜（仅群聊）
       │    │   │   ├─ 按钮 「🎲 随机事件」（仅群聊）
       │    │   │   ├─ 按钮 「▶ 推进剧情」
       │    │   │   └─ 在场人员条：点名@ / 💬私聊 / ✕离场 / 「＋ 叫人」
       │    │   ├─ 消息流
       │    │   │   ├─ 「剧情摘要」虚线框（有 summary 时）
       │    │   │   └─ 气泡[]：user（右） / assistant（左） / narration（居中斜体）
       │    │   │        └─ hover 操作：编辑 / 删除 / 重生成（末条）
       │    │   └─ 输入区
       │    │        ├─ 私聊信息条（头像 / 档位 / 好感 / 💞恋爱中 / ❤️表白·💔结束关系）
       │    │        ├─ textarea（Enter 发送，Shift+Enter 换行）
       │    │        └─ 「发送」or「■ 停止」 + 「⟳ 重生成」
       │    │
       │    └─ 右列 RightPanel (w-80)   【仅 chat 板块存在】
       │         ├─ Tab 记忆：长期记忆（事实卡列表）+ 剧情摘要
       │         └─ Tab 上下文：最近调度决策 / Token 用量 / 实际发送的系统提示
       │
       ├─ world ─── WorldView
       │    ├─ Header：世界名 + v{version}·{author} + 按钮组
       │    │    （导入世界观 / 导出 / 另存为副本 / 恢复默认 / 删除）
       │    ├─ Tab ✨ 快速生成（仅非官方世界）
       │    │    └─ 题材 / 基调 / 剧情方向 / 想包含的元素（多选 chips）
       │    │       + 「随手写两句」 + 要生成哪些分区
       │    │       + 「✨ 按这些生成」「🎲 离线随机毛坯」「清空重填」
       │    ├─ Tab 概览：世界观名称 / 标签 / 世界背景 / 开局钩子
       │    └─ Tab × 7（设定集七件套，每块都有：🎲随机 / ✨AI 生成 / 清空）
       │         ├─ 🌌 起源（含「力量体系」）
       │         ├─ 🗺️ 地理（名字 / 方位 / 主要势力 / 详情）
       │         ├─ 🧝 种族（种族名 / 一句话特征 / 详情）
       │         ├─ ⚔️ 势力阵营（名称 / 立场 / 说明 / 通行货币 / 物价参考）
       │         ├─ 🐺 生物（生物名 / 危险程度 / 出没地与习性）
       │         ├─ 📜 大事年表（年代 / 事件名 / 说明）
       │         └─ 👤 重要角色（名字 / 身份头衔 / 背景 + 关联角色卡）
       │
       ├─ traveler ─ TravelerView
       │    └─ 头像卡 + 性别（生理 / 认同）+ 长相 + 出身 + 性格 + 其他
       │       + 「这段会怎么告诉 NPC」预览 + 「清空重来」「去酒馆」
       │
       ├─ characters ─ CharactersView
       │    ├─ 顶部：导入角色卡 / ＋ 新建角色
       │    ├─ 筛选行：搜索框 + 全部 / 已置顶 / 官方 / 自建 + 取消全部置顶
       │    ├─ 分区 1：💞 亲密关系（恋爱中 / 挚交）
       │    ├─ 分区 2：其他 NPC（置顶优先，再按好感降序）
       │    └─ 角色详情面板（点击卡片飞入，Esc / 点背景关闭）
       │         ├─ 左栏：头像 / emoji / 主题色 / 名字 / 势力 / 关系状态 / 私聊 / 置顶
       │         └─ 8 个详情签：基本情况 / 形象 / 北极星 / 性格与背景 /
       │                        需求与创伤 / 说话方式 / 社交参数 / 补充
       │
       ├─ relations ─ RelationsView（力导图，纯展示）
       │    └─ 顶栏：世界观切换 / 🔍搜人物 / 分组筛选 / 清除筛选 / －百分比＋ / 图例
       │       画布：滚轮缩放（0.4~2.5）/ 拖空白平移 / 拖人物挪位 / 悬停提示 / 点选聚焦
       │
       ├─ journal ─ JournalView
       │    ├─ 「你是个什么样的人」：侧写 Textarea + 「✨ 现在记一笔」+ 自动记一笔开关
       │    ├─ 「自己记一条」：Input + 「记下」
       │    └─ 「记录（N）」：条目列表（时间 / 看出来的·自己写的 / 改 / 删）+ 「清空」
       │
       ├─ rules ─── RulesView
       │    ├─ 叙述视角 / 每段最多段落数 / 文风 / 硬性限制（每行一条）
       │    └─ 底层写作规则（Textarea + 「恢复默认」）
       │
       └─ settings ─ SettingsView（4 个签）
            ├─ 链接：模型服务（BYOK）+ 生成与模拟参数
            ├─ 操作指引：6 组图文说明
            ├─ 底层规则：全局写作规则（与「规则」板块同源）
            └─ 数据管理：导出单个世界 / 导出全部 / 导入备份 / 清空数据
```

### 2.2 上下级关系（数据所有权）

```
WorldPack（世界观包）                        ← 顶层容器，侧栏顶部的下拉决定「当前世界」
 ├─ codex     设定集七件套                   ← WorldView 编辑
 ├─ rules     规则（视角/段落/文风/限制）     ← RulesView 编辑
 ├─ factions  势力[]（每个势力带货币）        ← WorldView 编辑
 ├─ characters 角色卡[]（世界内置角色）       ← CharactersView 编辑
 ├─ initialRelationships 初始关系[]          ← 会话创建时复制进 ChatSession
 ├─ scenes    场景[]（按时段）+ seats        ← 只读（JSON 内可改）
 ├─ lorebook  世界书[]（关键词触发）         ← 只读（JSON 内可改）
 ├─ events    随机事件[]（权重 + 时段）      ← 只读（JSON 内可改）
 └─ openingHooks 开局钩子[]                  ← WorldView 概览签编辑

PlayerProfile（旅人，跨世界共享）            ← TravelerView
JournalEntry[] + journalPortrait（日志）      ← JournalView
CharacterCard[]（世界外的自定义角色）         ← CharactersView（导入 / 新建）
ChatSession[]（会话）                        ← 侧栏会话区 + ChatView
 ├─ messages[]  user / assistant / narration
 ├─ facts[]     事实卡（长期记忆）
 ├─ relationships[] 会话内关系
 └─ summary     滚动摘要
ProviderConfig[]（模型服务）                 ← SettingsView
GenerationSettings（生成参数）               ← SettingsView
writingRules（底层写作规则，全局）           ← SettingsView / RulesView
affinity: Record<角色id, -100~100>           ← 聊天自动累积
romances: RomanceLink[]                      ← 私聊表白产生
```

### 2.3 板块排序规则

- 可排序板块共 6 个（`NAV_ITEMS`），顺序存 `navOrder`。
- 「聊天」不在排序列表里——它常驻侧栏下半部分的会话区。
- 「设置」固定最底，不参与排序。
- 新增板块自动补到末尾（`normalizeNavOrder` 保证不丢、不重）。

---

## 3. 产品需求与痛点

### 3.1 用户角色

| 角色 | 典型诉求 |
| --- | --- |
| TRPG / 跑团玩家 | 想要一个「世界真的在转」的沉浸感，而不是问答机器人 |
| 同人 / 原创写作者 | 想让笔下角色自己互动，从中捞灵感与对白 |
| 角色卡玩家 | 已有社区 CCv2 卡，想直接导入并置入一个世界 |
| 独立开发者 / 自托管用户 | 不想把数据和 Key 交给第三方，要能本地跑、能改源码 |

### 3.2 痛点 → 解法 → 落点

| # | 痛点 | 解法 | 落在哪 |
| --- | --- | --- | --- |
| P1 | 多 NPC 轮流发言立刻出戏 | 发言调度器按 6 个分量打分，最高分低于阈值就**无人接话、转环境描写** | `core/social/orchestrator.ts` → ChatView |
| P2 | NPC 一边说台词一边写动作，格式糊成一团 | **台词 / 旁白彻底分离**：角色输出被 `stripActions` 剥掉动作，动作交给独立旁白轮 | `core/prompt/speech.ts` + `buildNarratorPrompt` |
| P3 | 聊十轮就忘了前面 | 三层记忆：滚动摘要（>阈值触发）+ 事实卡（LLM 抽取 + 半衰期遗忘 + 检索） | `core/memory/engine.ts` + RightPanel「记忆」 |
| P4 | 换个世界要重做所有设定 | World Pack 单文件封装：设定集 / 规则 / 势力 / 场景 / 世界书 / 事件 / 角色，可导入导出 Fork | `WorldPack` + WorldView / 设置数据管理 |
| P5 | 造世界门槛高 | 七件套每块可 🎲 离线随机（不联网不花钱）、✨ AI 生成、手写；还有「快速生成」按题材一键铺满 | `core/worldgen/*` + WorldView「✨ 快速生成」 |
| P6 | 世界越大越容易前后矛盾 | 设定集按分区结构化；来源标记（预制 / 随机 / AI / 手写）；生成时把已有设定喂给模型避免重复 | `CodexSource` + `worldBrief()` |
| P7 | 数据 / Key 不想上云 | 纯前端 + localStorage + BYOK；导出备份自动剔除 API Key | `persist` + `downloadJSON` |
| P8 | 提示词越长越贵、越容易被截断 | Token 预算：system 最多占 `contextBudget` 的 45%，超了按片段优先级从低到高裁剪，每段保底 60 token | `core/prompt/builder.ts` |
| P9 | 不知道模型到底收到了什么、为什么是 TA 说话 | 上下文检视器：展示实际 system 提示、各片段 token 占用、调度决策原因 | RightPanel「上下文」 |
| P10 | 关系进展看不见摸不着 | 五级好感（陌生→认识→熟悉→好友→挚交）+ 表白成功率 + 恋爱关系 + 力导图可视化 | `AFFINITY_STAGES` / RelationsView |
| P11 | AI 味重（破折号、排比、先否定后肯定） | 全局「底层写作规则」注入每一轮角色提示、旁白提示与所有世界生成请求 | `core/writingRules.ts` |
| P12 | 只想跟某个人说话 | 对话里说「私下聊聊」自动开私聊；角色卡、在场人物、气泡头像双击都能发起 | `PRIVATE_WORDS` + `startPrivate` |
| P13 | 侧栏板块不符合个人习惯 | hover ▲▼ 微调 + 长按拖拽排序，顺序存进存档 | `moveNav` / `reorderNav` |
| P14 | 玩家自己是谁，NPC 不知道 | 「旅人」档案（名字 / 性别 / 长相 / 出身 / 性格）+「日志」自动侧写，都进提示词 | TravelerView / JournalView |

### 3.3 功能优先级（P0–P3，对齐 docs/PRD.md）

| 级别 | 内容 | 状态 |
| --- | --- | --- |
| P0 | 模型接入（BYOK）、单/多角色对话、角色卡、会话存档、摘要记忆 | ✅ |
| P1 | 设定集七件套、世界生成（随机/AI）、世界书、规则、好感与恋爱、关系图 | ✅ |
| P2 | 社交模拟调度、旁白轮、事实卡、日志侧写、上下文检视器、导入导出 | ✅ |
| P3 | 情报传播与失真、目标驱动主动搭话、日程与自主离场、分支对话树、向量记忆、TTS、i18n | 🚧/📋 规划中 |

---

## 4. 数据模型（核心实体）

### 4.1 WorldPack（世界观包）

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `id` | string | 唯一 id，官方为 `witcher3` |
| `spec` | `'chat_world_v1'` | 格式标识 |
| `name` / `version` / `author` / `tags` | — | 元信息 |
| `derivedFrom` | `{id,version} \| null` | Fork 来源 |
| `overview` | string | 世界背景 |
| `codex` | `WorldCodex` | 设定集七件套 |
| `rules` | `WorldRules` | 视角 / 每段最多段落数 / 文风 / 硬性限制 |
| `scenes` | `Scene[]` | 时段场景（ambience + npcIds + seats） |
| `lorebook` | `LorebookEntry[]` | 世界书（keys / secondaryKeys / priority / probability / cooldown / position） |
| `factions` | `Faction[]` | 势力（name / description / alignment / **currency{unit,note}** / source） |
| `characters` | `CharacterCard[]` | 世界内置角色 |
| `initialRelationships` | `Relationship[]` | NPC↔NPC、NPC↔玩家的初始关系 |
| `events` | `WorldEvent[]` | 随机事件（name / weight / timeOfDay / prompt） |
| `openingHooks` | string[] | 开局钩子 |
| `builtin` | boolean? | 是否官方包（决定是否显示「恢复默认」） |

### 4.2 WorldCodex（设定集七件套）

| 分区 | 类型 | 形态 |
| --- | --- | --- |
| `origin` | `CodexSection {text, source, updatedAt}` | 成段 |
| `powerSystem` | string | 力量体系（挂在「起源」页签里） |
| `geography` | `CodexPlace[]`（名字/方位/主要势力/详情） | 一条一条 |
| `races` | `CodexEntry[]`（名字/短标签/详情） | 一条一条 |
| `creatures` | `CodexEntry[]` | 一条一条 |
| `timeline` | `TimelineEntry[]`（年代/标题/详情） | 一条一条 |
| `figures` | `CodexFigure[]`（名字/身份/详情/characterId） | 一条一条 |

> 势力阵营不在这里，用 `WorldPack.factions`（本身就带货币与来源）。

### 4.3 CharacterCard（人物卡）

| 分组 | 字段 |
| --- | --- |
| 基础 | `id`、`name`、`description`、`personality`、`scenario`、`tags[]` |
| 台词 | `firstMes`、`alternateGreetings[]`、`mesExample`、`postHistoryInstructions`、`systemPrompt` |
| 社交 | `social{extraversion, talkativeness, secrecy, gullibility, temper, goals[], schedule{4时段}}` |
| 记忆 | `memory{rememberPlayerFacts, halfLifeDays}` |
| 外观 | `presentation{emoji, color, avatar?, voice?}` |
| 元信息 | `creator`、`creatorNotes`、`builtin?`、`proactive?`、`faction?` |
| **详情五段式** | `profile{gender, build, race, appearance, northStar, bonds, clothing, backstory, surfaceNeed, coreNeed, trauma}` |

### 4.4 其他实体

| 实体 | 关键字段 |
| --- | --- |
| `PlayerProfile` | name / origin / personality / bioSex / gender / appearance / avatar / emoji / note |
| `JournalEntry` | id / at / text / source(`ai`\|`manual`) |
| `ChatSession` | id / title / worldId / sceneId / kind(`tavern`\|`private`) / privateWith / timeOfDay / characterIds / messages / summary / facts / relationships / createdAt / updatedAt |
| `ChatMessage` | id / role(`user`\|`assistant`\|`narration`\|`system`) / speakerId / speakerName / content / createdAt / tokens? / reason? |
| `MemoryFact` | id / subject / predicate / object / confidence / source / createdAt / score |
| `Relationship` | from / to / affinity(-100~100) / trust(0~100) / familiarity(0~100) / tags[] / kind? / aware? |
| `ProviderConfig` | id / name / type / baseUrl / apiKey / model / temperature / topP / maxTokens / frequencyPenalty / presencePenalty / priceIn? / priceOut? |
| `GenerationSettings` | contextBudget / recentTurns / summarizeThreshold / maxAutoTurns / socialSimulation / autoRelationship / autoFacts / autoJournal? / narration? / userName / userPersona |

---

## 5. 接口与输入输出总览

> 本项目没有自建后端，所谓「接口」分五类：**① 模型服务 HTTP 接口（唯一外网出口）**、**② 内部模块函数接口**、**③ 状态仓库（Store）action 接口**、**④ 持久化接口**、**⑤ 文件导入导出接口**。

### 5.1 ① 模型服务 HTTP 接口

#### 5.1.1 统一契约 `LLMClient`

```ts
interface LLMClient {
  readonly type: string;
  chat(req: LLMRequest): Promise<LLMResult>;                       // 非流式
  stream(req: LLMRequest, onDelta: (t: string) => void): Promise<LLMResult>; // 流式
  listModels(): Promise<string[]>;
  test(): Promise<{ ok: boolean; error?: string; latencyMs: number }>;
}
interface LLMRequest { messages: LLMMessage[]; model: string; temperature?; topP?; maxTokens?;
                       frequencyPenalty?; presencePenalty?; stop?: string[]; signal?: AbortSignal }
interface LLMResult  { content: string; usage?: { promptTokens?: number; completionTokens?: number } }
class LLMError extends Error { constructor(message: string, readonly status?: number) }
```

工厂：`createClient(cfg)` → `anthropic` 走 `AnthropicClient`，`ollama` 与 `openai-compatible` **共用** `OpenAICompatibleClient`。

#### 5.1.2 OpenAI 兼容（`openai-compatible` / `ollama`）

| 项 | 值 |
| --- | --- |
| 对话 | `POST {baseUrl}/chat/completions`（`baseUrl` 先去掉结尾所有 `/`） |
| 模型列表 | `GET {baseUrl}/models` |
| 请求头 | `Content-Type: application/json`；**仅当 apiKey 非空**时加 `Authorization: Bearer {apiKey}`（Ollama 可无 Key） |
| 请求体 | `{ model, messages, stream, temperature, top_p, max_tokens, frequency_penalty, presence_penalty, stop?, stream_options?{include_usage} }` |
| 非流式解析 | `json.choices[0].message.content`；usage ← `usage.prompt_tokens / completion_tokens` |
| 流式解析 | 手写 SSE：按 `\n` 切行、留半行 buffer、只认 `data:` 前缀、`[DONE]` 结束、取 `choices[0].delta.content`、末包 `usage` |
| 错误 | `!res.ok` → `throw new LLMError(readError(res), res.status)`；无 body → `LLMError('当前响应不支持流式输出')` |

`readError` 会把状态码翻成人话：401/403 → 「API Key 无效或没有权限」；404 → 「接口地址可能有误，常见写法是 `https://api.openai.com/v1`」；status 0 → 「请求被拦截：通常是跨域 CORS 或网络不通」。

#### 5.1.3 Anthropic（`anthropic`）

| 项 | 值 |
| --- | --- |
| 对话 | `POST {baseUrl}/messages` |
| 模型列表 | `GET {baseUrl}/models?limit=100`（失败返回 `[]`，不抛错） |
| 请求头 | `Content-Type`、`x-api-key: {apiKey}`、`anthropic-version: 2023-06-01` |
| 请求体 | `{ model, system（system 消息合并）, messages（role 仅 user/assistant）, max_tokens, temperature, top_p, stream }` |
| 解析 | 非流式 `content.find(c=>c.type==='text').text`；流式只处理 `content_block_delta.delta.text`（**流式不返回 usage**） |

#### 5.1.4 预设服务商 `PRESET_PROVIDERS`

| 名称 | 类型 | Base URL | 默认模型 | 单价（每百万 token） |
| --- | --- | --- | --- | --- |
| OpenAI | openai-compatible | `https://api.openai.com/v1` | `gpt-4o-mini` | in 0.15 / out 0.6 |
| DeepSeek | openai-compatible | `https://api.deepseek.com/v1` | `deepseek-chat` | in 0.27 / out 1.1 |
| Moonshot 月之暗面 | openai-compatible | `https://api.moonshot.cn/v1` | `moonshot-v1-8k` | — |
| Anthropic Claude | anthropic | `https://api.anthropic.com/v1` | `claude-3-5-sonnet-latest` | — |
| Ollama（本地） | ollama | `http://localhost:11434/v1` | `qwen2.5:7b` | — |

五条预设共用参数：`temperature 0.9 / topP 0.95 / maxTokens 1024 / frequencyPenalty 0 / presencePenalty 0`。
首次启动的默认 Provider：名称「我的模型」，type `openai-compatible`，**默认用 DeepSeek 的 URL 与模型**，`maxTokens: 700`，`apiKey: ''`。

#### 5.1.5 各类生成请求的参数对照（输入输出一览）

| 场景 | 构造提示词 | 模型参数 | 输入 | 输出 / 解析 |
| --- | --- | --- | --- | --- |
| 角色台词 | `buildPrompt` | 用 Provider 自身参数（默认 temp 0.9 / max 700） | 在场 NPC、世界、规则、世界书命中、事实卡、摘要、近期消息、场景、玩家 | 流式文本 → `stripActions` 剥动作 → 落 assistant 气泡 |
| 旁白 | `buildNarratorPrompt` | 同上 | 世界、场景、在场、近期 6 条 | 流式文本 → narration 气泡 |
| 滚动摘要 | `buildSummaryMessages` | maxTokens 500 / temp 0.3 | 旧摘要 + 待摘要消息 | 纯文本（≤300 字）→ `session.summary` |
| 事实抽取 | `buildFactMessages` | maxTokens 400 / temp 0.2 | 最近 8 条消息 | JSON 数组 → `parseFacts` → `mergeFacts` |
| 玩家侧写 | `buildJournalMessages` | maxTokens 400 / temp 0.4 | 最近 12 条消息 + 旧侧写 | `{notes[], portrait}` → 日志条目 + 侧写 |
| 世界核心 | `buildBriefMessages` | maxTokens 700 / temp 1 | `WorldBrief`（题材/基调/剧情/元素/随手写） | 四行文本 → `parseBrief` → `{name,tags,overview,openingHooks}` |
| 设定集分区 | `buildCodexMessages` | maxTokens 600 / temp 1 | 分区 kind + 已有设定摘要 + 写作规则 | 竖线分隔多行 → `parseCodex` → 各分区条目 |
| 连接测试 | `client.test()` | maxTokens 16 | 一条 `ping` | `{ok, latencyMs, error?}` |
| 模型列表 | `client.listModels()` | — | — | `string[]` |

> 所有 AI 生成请求的 system 提示末尾都会追加「生成的内容必须遵守以下写作原则：{底层写作规则}」，保证世界文本与对白同风格。

### 5.2 ② 内部模块接口

| 模块 | 导出 | 输入 → 输出 |
| --- | --- | --- |
| `prompt/builder.ts` | `buildPrompt(input)` | `{character, world, settings, player, lorebookHits, facts, summary, recent, present, scene, writingRules}` → `{messages, systemPrompt, parts}` |
| | `buildNarratorPrompt(input)` | `{world, settings, player, recent, scene, present, writingRules}` → `{messages, systemPrompt, parts:{}}` |
| | `buildSystemPrompt` / `describePlayer` / `toLLMMessages` | 内部片段装配 |
| `prompt/speech.ts` | `stripActions(text, speaker?)` | 去掉 `*…*`、≤60 字的括号动作、开头「某某：」 |
| `worldgen/ai.ts` | `buildCodexMessages(kind, world, count=4, writingRules='')` | → `LLMMessage[]` |
| | `parseCodex(kind, raw)` | 原始文本 → `string \| Faction[] \| CodexPlace[] \| CodexEntry[] \| TimelineEntry[] \| CodexFigure[]` |
| `worldgen/brief.ts` | `buildBriefMessages(b, writingRules='')` / `parseBrief(raw, fallback)` / `randomWorldCore(b)` | → `WorldCore{name,tags,overview,openingHooks}` |
| `worldgen/random.ts` | `randomCodex(kind)` | 无输入 → 该分区 3~4 条离线随机内容（source `random`） |
| `worldgen/apply.ts` | `applyCodex(world, kind, value, source)` | **追加**到对应分区（factions 写 `world.factions`）→ 新 WorldPack |
| | `clearCodex(world, kind)` | 清空该分区 → 新 WorldPack |
| `memory/engine.ts` | `retrieveFacts(facts, query, k=6)` | 半衰期衰减 + 关键词重合打分 → 前 k 条 |
| | `mergeFacts(existing, incoming)` | 以 `subject\|predicate` 去重 → 最多 120 条 |
| | `buildSummaryMessages` / `buildFactMessages` / `parseFacts` | 见 5.1.5 |
| `social/orchestrator.ts` | `selectNextSpeaker({candidates, messages, relationships, config?, forceSpeakerId?})` | → `{speakerId \| null, scores[], reason}` |
| | `applyRelationshipDelta(base, delta[])` | 关系增量 → 新关系表（自动 familiarity +3，clamp） |
| `social/relations.ts` | `buildGraph(world, {affinity, romanced, userName, playerAvatar?, playerEmoji?, extra?})` | → `{nodes, edges}` |
| | `relationGroups(world, edges)` | → 势力组 / 家庭组 |
| `social/forceGraph.ts` | `seedPosition(i,count,w,h)` / `relax(nodes,edges,w,h,focusIds)` | 力导图布局迭代 |
| `lorebook/matcher.ts` | `scanText(messages, scanDepth)` / `matchLorebook(entries, text, {cooldownState, maxEntries})` | → 命中条目[] |
| `journal.ts` | `buildJournalMessages(recent, prevPortrait)` / `parseJournal(raw)` | → `{notes[], portrait}` |
| `characterCard.ts` | `toCCv2(card)` / `fromCCv2(raw)` / `createEmptyCard()` / `downloadJSON(name, data)` | 见 5.5 |
| `avatar.ts` | `fileToAvatar(file, max=256)` | 图片 File → 压缩后的 dataURL（>20MB 或非图片抛错） |
| `llm/token.ts` | `estimateTokens` / `estimateMessagesTokens` / `estimateCost` / `truncateToTokens` | 文本 ↔ token 估算（CJK ×0.6，其余 /4） |
| `idleLines.ts` / `id.ts` / `writingRules.ts` | `pickIdleLine()` / `uid(prefix)` / `DEFAULT_WRITING_RULES` | 空态文案 / 唯一 id / 默认写作规则 |

### 5.3 ③ Store 接口（`useChatStore`，全部 action 的输入输出）

#### 查询（同步，无副作用）

| action | 输入 | 输出 |
| --- | --- | --- |
| `getWorld()` | — | 当前 `WorldPack \| undefined` |
| `allCharacters()` | — | 当前世界角色 + 世界外自定义角色 |
| `findCharacter(id)` | id | `CharacterCard \| undefined` |
| `getSession()` | — | 当前会话 |
| `presentCharacters()` | — | 当前会话在场角色[] |
| `getActiveProvider()` | — | 当前 Provider（找不到时退回 `providers[0]`） |
| `affinityOf(id)` | id | 好感值（缺省 0） |
| `stageOfCharacter(id)` | id | 档位对象 `{id,label,min,color,chance}` |
| `isRomancing(id)` | id | boolean |

#### 写操作

| action | 输入 | 输出 / 副作用 |
| --- | --- | --- |
| `setView(v)` | `View` | 切换主工作区板块 |
| `setActiveWorld(id)` | worldId | 切世界；合并好感表；当前会话不属于该世界时切换到该世界最近会话（没有就新开） |
| `updateWorld(id, patch)` | id + 部分字段 | 打补丁到该世界 |
| `resetWorld(id)` | id | 官方世界还原为随包版本 |
| `importWorld(raw)` | 任意 JSON | 校验 `name` → 追加世界并设为当前；返回新 id 或 `null` |
| `deleteWorld(id)` | id | 删除；删光了自动恢复官方包 |
| `createWorld(name?)` | 名称 | 建空白世界（空设定集、空规则、无角色）→ 返回 id |
| `generateCodex(worldId, kind\|'all', 'random'\|'ai')` | — | 异步：随机立即完成；AI 逐块请求并**追加**结果；期间 `codexBusy = kind` |
| `clearCodexSection(worldId, kind)` | — | 清空该分区 |
| `generateWorldFromBrief(worldId, brief, kinds[], mode)` | — | 异步：先清空所选分区 → 生成核心（名称/标签/简介/钩子）→ 逐块填分区 |
| `setWritingRules(v)` | 文本 | 覆盖全局底层写作规则 |
| `addCharacter()` | — | 建空卡（世界外），返回 id |
| `updateCharacter(id, patch)` | — | 同时更新世界内 / 世界外同名卡 |
| `deleteCharacter(id)` | — | 删除并移出置顶 |
| `importCharacter(raw)` | CCv2 JSON | → 新卡 id 或 `null` |
| `togglePinnedCharacter(id)` | — | true / false（满 6 个返回 false） |
| `clearPinnedCharacters()` | — | 清空置顶 |
| `updatePlayerProfile(patch)` | 部分档案 | 同步 `settings.userName` |
| `resetPlayerProfile()` | — | 回默认 |
| `refreshJournal()` | — | 异步：请求模型写侧写，追加 ≤3 条观察（总上限 200） |
| `addJournalEntry(text)` | 文本 | 手动记一条 |
| `setJournalPortrait(v)` / `updateJournalEntry(id,text)` / `deleteJournalEntry(id)` / `clearJournal()` | — | 日志 CRUD |
| `addProvider(preset?)` / `updateProvider(id,patch)` / `deleteProvider(id)` / `setActiveProvider(id)` | — | Provider CRUD（删光自动补默认） |
| `bumpAffinity(id, delta)` | — | 好感 ±，clamp(-100,100) |
| `proposeRomance(id)` | — | 按档位概率掷骰 → `{ok,label,chance}`；成功 +12 好感并置顶 |
| `endRomance(id)` | — | 解除恋爱 |
| `newSession()` | — | 建酒馆会话（黄昏场景 + 场景氛围旁白 + 主持人开场白）；主动型 NPC 会把玩家拉去私聊 |
| `startPrivate(id, {greeting?,opening?})` | — | 已有则切过去，否则建私聊（返回 sessionId 或 null） |
| `startGroup(ids[], title?)` | — | 建小群会话 |
| `deleteSession(id)` / `setActiveSession(id)` / `clearActiveSession()` | — | 会话管理 |
| `setTimeOfDay(tod)` | 时段 | 换场景（氛围旁白入流）+ 按日程重算在场人员 |
| `togglePresent(id)` | — | 叫人进场 / 请人离场 |
| `updateSession(patch)` | — | 改标题等 |
| `sendMessage(text)` | 玩家文本 | 异步：落 user 消息 → 识别「私下聊聊」→ `runTurns(maxAutoTurns)` |
| `advance()` | — | 异步：`runTurns(1)`，让 NPC 自己聊一轮 |
| `stop()` | — | `AbortController.abort()`，停止流式生成 |
| `regenerateLast()` | — | 删掉最后一条非 user 消息后重生成 |
| `editMessage(id, content)` / `deleteMessage(id)` | — | 消息编辑 / 删除 |
| `moveNav(id, delta)` / `reorderNav(id, toIndex)` / `resetNavOrder()` | — | 板块排序 |
| `updateSettings(patch)` | 部分设置 | 打补丁 |
| `clearError()` / `wipeAll()` | — | 清错误 / 清空全部数据 |

### 5.4 ④ 持久化接口

| 项 | 值 |
| --- | --- |
| 存储 | `localStorage`，key = `ai-npc-chat`（zustand `persist`） |
| 当前版本 | `version: 11` |
| 持久化字段（`partialize`） | `worlds, characters, pinnedCharacterIds, navOrder, playerProfile, playerJournal, journalPortrait, affinity, romances, activeWorldId, writingRules, providers, activeProviderId, sessions, activeSessionId, settings` |
| **不持久化** | `busy, error, debug, codexBusy, journalMark, journalBusy, activeView`（运行时态） |
| 世界书冷却态 | 内存 `Map<sessionId, Map<entryId, 剩余轮数>>`，刷新即重置 |

迁移历史（`migrate(persisted, version)`）：

| 版本 | 迁移内容 |
| --- | --- |
| v2 | 世界观补设定集七件套；预制世界补回官方设定 |
| v3 | 补 `pinnedCharacterIds` |
| v4 | 官方新增的世界包补进存档 |
| v5 | 好感 / 恋爱 / 私聊会话 / 板块顺序 |
| v6 | 官方世界只剩「巫师 3」，下架旧包（`broken-axe`、`rain-harbor`）并对齐默认世界与会话标题 |
| v7 | 补「关系」板块 |
| v8 | 地理 / 种族 / 生物由整段文字改为一条一条（老文本按行拆） |
| v9 | 补「旅人」板块，把设置里的自称 / 人设搬过去 |
| v10 | 补「日志」板块 |
| v11 | 规则瘦身（删 `tense/powerLevel/currency`）；力量体系进 `codex.powerSystem`；货币下放到每个 `Faction.currency`；角色补 `profile` 五段式；**预制世界直接用随包最新版覆盖**（玩家自建角色保留） |

### 5.5 ⑤ 文件 / 剪贴板接口

| 方向 | 格式 | 文件名规则 | 说明 |
| --- | --- | --- | --- |
| 导出世界观 | `WorldPack` JSON（`spec: chat_world_v1`） | `{world.name}.world.json` | WorldView「导出」/ 设置「只导出这一个」 |
| 导入世界观 | 同上，校验 `name` | — | 失败 `alert('无法识别的世界观文件')`；id 冲突时新建 id |
| 导出角色卡 | **Character Card V2**（`spec: chara_card_v2`，私有字段放 `data.extensions.chat`） | `{card.name}.json` | 兼容 SillyChat；`social / memory / presentation` 不丢 |
| 导入角色卡 | CCv2（宽容解析，支持 `char_name` 等别名） | — | 无 `name` 返回 null；id 始终新建 |
| 导出全量备份 | `{spec:'chat_backup_v1', exportedAt, worlds, characters, sessions, settings}` | `ai-npc-chat-backup.json` | **不含 API Key** |
| 导入备份 | 同上，校验 `data.worlds` | — | 直接整体覆盖（含 activeWorldId） |
| 头像上传 | `image/*` File | — | 压缩到最长边 256、JPEG 0.85，存 dataURL；>20MB 或非图片报错 |

> 已知坑：`fromCCv2` 内部 `num(v, fb) = clamp(Number(v),0,1)`，导入时 `memory.halfLifeDays`（默认 7）会被压到 0~1，需在导入后手动改回。

---

## 6. 按板块的功能文档

### 6.1 侧栏（常驻）

**职责**：切世界 / 切板块 / 管会话 / 进设置。

| 元素 | 行为 |
| --- | --- |
| 世界观下拉 | `setActiveWorld(id)`；官方包带「（官方）」后缀 |
| 「＋ 创建新世界」 | `createWorld()` 并跳到世界观板块 |
| 板块项 | 单击进入；hover 显示 ▲▼（`moveNav`）；**长按 300ms** 进入拖拽（移动 >6px 视为滚动并取消长按），松手落到目标位置（`reorderNav`） |
| 「💬 聊天」 | 回到当前会话 |
| 「＋」 | 打开 NewChatMenu：🍺 群聊 / 🤫 私聊（选 1 人）/ 👥 自己建群（选 ≥2 人） |
| 会话条目 | 点击进入；hover ✕ 直接删除（**无二次确认**） |
| 会话区空白处 | `clearActiveSession()`，退出会话选中 |
| ⚙️ 设置 | 进入设置板块（固定最底） |

### 6.2 聊天板块

| 能力 | 入口 | 规则 |
| --- | --- | --- |
| 发言 | 输入框 Enter 发送 | Shift+Enter 换行；`busy` 时禁止发送 |
| 停止 | 「■ 停止」 | `AbortController.abort()`，已生成部分保留 |
| 重生成 | 输入框旁「⟳」或末条气泡 hover「重生成」 | 删除最后一条非 user 消息后，强制同一发言者重来 |
| 编辑 / 删除消息 | 气泡 hover | 双击气泡也可编辑 |
| 点名 `@名字` | 点在场人员头像 / 名字，或手打 `@` | `forceSpeakerId` 直接指定，且私聊中点名 +2 好感 |
| 请人离场 / 叫人进场 | 在场条 ✕ / 「＋ 叫人」 | 改 `session.characterIds` |
| 切换时段 | 清晨/正午/黄昏/深夜（仅群聊） | 换场景 + 按 `social.schedule` 重算在场人员 |
| 随机事件 | 「🎲 随机事件」（仅群聊） | 按 `world.events` 中命中当前时段的条目加权随机，插入一条旁白后自动推进一轮 |
| 推进剧情 | 「▶ 推进剧情」 | `runTurns(1)`，让 NPC 自己聊 |
| 私聊 | 在场人员 💬 / 双击酒馆气泡头像 / 角色卡「🤫 私聊」 | 已有则切过去；对话中说「私下聊聊 / 借一步 / 只跟你」等会自动开 |
| 表白 / 结束 | 私聊信息条按钮 | 按档位概率掷骰；成功 +12 好感并自动置顶 |
| 上下文检视 | RightPanel「上下文」 | 调度原因 + 各片段 token + 实际 system 提示（只读） |
| 记忆检视 | RightPanel「记忆」 | 事实卡（主谓宾 + 置信度）+ 滚动摘要 |

**空态**：无世界 → 「请先在世界观栏里选一个世界」；无会话 → 「在左边挑一个会话接着聊，或者点「＋」开一段新的」+ 一句随机文学文案（20 条轮换不重复）。

### 6.3 世界观板块

| 能力 | 说明 |
| --- | --- |
| 概览 | 名称 / 标签（逗号分隔）/ 世界背景 / 开局钩子（每行一条） |
| 快速生成（仅自建世界） | 勾题材·基调·剧情·元素 + 随手写两句 → ✨AI（需 Key）或 🎲离线随机毛坯；可勾选只生成部分分区 |
| 七件套编辑 | 每块支持 🎲随机 / ✨AI 生成 / 清空 / 手写；生成是**追加**，不会覆盖已写内容 |
| 来源标记 | 每条显示「预制 / 随机生成 / AI 生成 / 手写」 |
| 力量体系 | 在「起源」页签内，用于防止 NPC 突然开挂 |
| 势力货币 | 每个势力两条输入：通行货币名 + 物价参考（NPC 报价按它来） |
| 分享与版本 | 导入 / 导出 `.world.json` / 另存为副本（写 `derivedFrom`）/ 恢复默认（仅官方）/ 删除 |

### 6.4 旅人板块

玩家自己的档案，**跨世界共享**，内容进「玩家」提示词片段。

字段：名字（与设置「玩家自称」同步）、头像（上传图片自动压缩 / 20 个 emoji 选一个）、生理性别、性别认同、长相、出身、性格、其他。
底部有「这段会怎么告诉 NPC」实时预览（`describePlayer` 输出），没写的字段整行省略。

### 6.5 角色板块

**列表**：搜索（名字 / 标签 / 简介）+ 筛选（全部 / 已置顶 / 官方 / 自建）；分「💞 亲密关系」与「其他 NPC」两区；置顶最多 6 个。
**卡片**：头像、名字、官方标、置顶序、档位、💞恋爱中、好感值、简介 3 行、标签；操作：★置顶、🤫 私聊、点击进入详情。

**详情八签**（点「保存」才写回）：

| 签 | 字段 |
| --- | --- |
| 基本情况 | 姓名 / 性别 / 体型 / 种族 / 势力 / 标签 / 一句话简介 / 当前情境 |
| 形象 | 角色形象（发色、肤色、五官、痕迹）/ 服装要求 |
| 北极星 | 一句话说明角色核心 |
| 性格与背景 | 角色性格 / **关系图表（人际关系）** / 角色故事背景 |
| 需求与创伤 | 表层需求 / 核心需求 / 创伤事件 |
| 说话方式 | 开场白 / 说话腔调 / 示例对话 |
| 社交参数 | 外向度 / 话痨度 / 守口如瓶 / 轻信度 / 情绪波动 / 目标动机 / 主动接近玩家 / 在场时段 |
| 补充 | 备用开场白 / 作者 / 作者备注 / 系统提示 |

底部：导出角色卡（CCv2）/ 删除 / 取消 / 保存。

### 6.6 关系板块

力导图（无表单）。节点 = 当前世界角色 + 玩家节点；边 = `initialRelationships` + 会话内关系 + 好感推导出的「对玩家」关系。

- 线色：家人（金）/ 对象（红）/ 挚友（蓝）/ 普通（灰）；**单向关系**（`aware === false` 或命中「暗恋 / 单恋 / 秘密」等词）带箭头。
- 交互：滚轮缩放（40%~250%）、拖空白平移、拖人物挪位并固定、悬停看关系条数、点选某人聚焦并高亮其关系线（其余降到 8% 透明度）、顶栏可切世界 / 搜人物 / 按势力或家庭筛选 / 清除筛选。

### 6.7 日志板块

- 「你是个什么样的人」：模型看完对话写出的玩家侧写（可手动改）；「✨ 现在记一笔」立即触发；「聊完自动记一笔」开关（默认开）。
- 「自己记一条」：手动记录（Enter 提交）。
- 「记录（N）」：按时间倒序，标「看出来的」（AI）或「自己写的」，可改可删，可一键清空。
- 自动触发：每积累 6 条新消息（`JOURNAL_EVERY`）触发一次，取最近 12 条消息，最多追加 3 条观察，总条数上限 200。

### 6.8 规则板块

作用于**当前选中的世界观**，对所有 NPC 立即生效。

| 字段 | 说明 |
| --- | --- |
| 叙述视角 | 进「规则」提示词片段 |
| 每段最多段落数 | 同上 |
| 文风 | 同上 |
| 硬性限制 | 每行一条，进输出要求 |
| 底层写作规则 | 全局（跨世界）去 AI 味规则副本，可在设置里改同一份 |

> 已下线字段：时态、力量体系、货币单位（前两者已迁移：力量体系 → 世界观-起源；货币 → 世界观-势力）。

### 6.9 设置板块

| 签 | 内容 |
| --- | --- |
| 链接 | **模型服务**：预设一键添加（5 家）；名称 / 类型 / 模型 / Base URL / API Key / Temperature / Top P / 最大输出 Token / 频率惩罚；「测试连接」「拉取模型列表」「删除」。**生成与模拟**：上下文预算 / 保留近期消息条数 / 接话轮数 / 触发摘要阈值 / 社交模拟开关 / 自动抽取长期记忆 / 对话间插入旁白 / 玩家自称 / 玩家人设 |
| 操作指引 | 6 组图文说明（聊天 / 角色 / 世界观 / 关系·日志·规则 / 设置 / 板块排序） |
| 底层规则 | 全局写作规则 Textarea + 恢复默认 |
| 数据管理 | 导出单个世界 / 导出全部备份 / 导入备份 / 清空数据 |

---

## 7. 核心流程时序

### 7.1 一次玩家发言（`sendMessage`）

```
玩家 Enter
  → 落 user 消息（speakerName = settings.userName）
  → 私聊中点名 → 好感 +2
  → 命中「私下聊聊」关键词 → startPrivate(目标) → 把这句带进私聊 → runTurns(1, 目标)
  → 否则 runTurns(settings.maxAutoTurns)
       每一轮：
         ① matchLorebook(world.lorebook, scanText(最近 6 条), {cooldown, maxEntries:6})
         ② retrieveFacts(session.facts, 最近 3 条拼接, 6)
         ③ selectNextSpeaker(在场, 全部消息, 关系, forceSpeakerId)
              → 有 speaker：buildPrompt(...) ；无 speaker（低于阈值）：buildNarratorPrompt(...)
         ④ 落占位消息 → client.stream（逐字写入气泡）
         ⑤ stripActions 剥动作：若一句台词都不剩 → 整条降级为旁白
         ⑥ 私聊且说出台词 → 好感 +2
         ⑦ 每 2 轮（i>0 且 (i+1)%2===0）插一段旁白
  → finally：busy=false → postProcess()（异步，不阻塞）
        · 消息数 > summarizeThreshold → 生成滚动摘要，并裁掉旧消息只留 recentTurns
        · autoFacts → 抽取事实卡 → mergeFacts
        · autoJournal ≠ false 且攒够 6 条 → refreshJournal()
```

### 7.2 世界生成

```
「✨ 按这些生成」
  → clearCodexSection(每个勾选分区)          // 先清，避免新旧叠加
  → client.chat(buildBriefMessages(brief, writingRules))  → parseBrief → {name, tags, overview, hooks}
  → updateWorld(核心字段)
  → 逐分区 generateCodex(worldId, kind, 'ai')
        client.chat(buildCodexMessages(kind, world, 4, writingRules)) → parseCodex → applyCodex(source:'ai')
「🎲 离线随机毛坯」
  → randomWorldCore(brief)（本地词库）→ 逐分区 randomCodex(kind) → applyCodex(source:'random')
```

AI 返回的文本格式（竖线分隔，一行一条）：

| 分区 | 字段 |
| --- | --- |
| 地理 | 地点名 \| 方位 \| 主要势力 \| 一句话 |
| 种族 | 种族名 \| 一句话特征 \| 关系与处境 |
| 生物 | 生物名 \| 危险程度 \| 出没地与习性 |
| 势力 | 名称 \| 立场 \| 说明 \| **通行货币名** \| **物价参考** |
| 年表 | 年代 \| 事件名 \| 说明 |
| 重要角色 | 名字 \| 身份头衔 \| 背景或秘密 |

解析容错：自动去 markdown 序号 / 代码块围栏；多余字段用「｜」并回详情；空结果时整段原文塞进第一条占位条目。

### 7.3 提示词装配（上下文预算）

system 提示按固定顺序拼 10 段，每段带 `keep` 权重；总预算 = `contextBudget × 45%`，超了按权重从低到高裁剪，每段保底 60 token，被裁量记在 `parts['裁剪']`。

| # | 片段 | keep |
| --- | --- | --- |
| 1 | 身份（角色 profile 五段式 → 人设 → 情境 → 目标） | 1.0 |
| 2 | 世界设定（概览 + 七件套 + 势力含货币） | 0.95 |
| 3 | 规则（视角 / 文风 / 段落 / 限制 + 力量体系） | 0.9 |
| 4 | 底层写作规则 | 0.88 |
| 5 | 场景（名称 / 氛围 / 在场名单） | 0.9 |
| 6 | 世界书命中 | 0.8 |
| 7 | 记忆（事实卡 ≤12 条） | 0.8 |
| 8 | 摘要 | 0.7 |
| 9 | 玩家 | 0.6 |
| 10 | 输出要求（角色只说台词 + postHistoryInstructions + systemPrompt） | 1.0 |

历史消息映射：`user` → `user`；`narration` → `assistant`（前缀 `*旁白*`）；`assistant` → 带「说话人：」前缀，**最后一条不加前缀**（让模型自然续写当前角色）；末尾补 `（现在轮到 X 发言）`。

### 7.4 发言调度器

```
score = 话题相关度 ×3.0 + 性格外向度 ×1.2 + 目标驱动 ×1.5
      + 冷却惩罚(负) ×1.6 + 关系驱动 ×0.8 + 随机扰动 ×0.8
```

| 分量 | 取值 |
| --- | --- |
| 相关度 | 被点名 1.0 / 命中目标前 4 字 0.7 / 命中标签 0.4 / 否则 0.15 |
| 性格 | `extraversion×0.8 + temper×0.2` |
| 目标 | 有目标时：相关度 ≥0.7 记 1，否则 0.35 |
| 冷却 | `-(窗口内发言次数/4) - (上一位是自己 ? 0.6 : 0)`（窗口 4 轮） |
| 关系 | 对上一位发言者 `|好感|/100`（喜欢和讨厌都会更想接话；无关系 0.1） |
| 噪声 | `random()` |

判定：`forceSpeakerId` 优先；最高分 < **1.1**（`silenceThreshold`）→ 本轮无人发言，转环境描写；否则最高分者发言。决策原因会写进消息 `reason` 并在 RightPanel 展示。

---

## 8. 常量、阈值与默认值速查

### 8.1 生成与模拟

| 常量 | 默认 | 说明 |
| --- | --- | --- |
| `contextBudget` | 6000 | 上下文 token 预算（可设 1000~32000） |
| `recentTurns` | 20 | 保留近期消息条数（4~60） |
| `summarizeThreshold` | 40 | 触发摘要的消息数（10~200） |
| `maxAutoTurns` | 2 | 玩家发言后 NPC 接话轮数（1~6） |
| `socialSimulation` | true | 关掉则退化为「永远第一个人说话」 |
| `autoFacts` | true | 自动抽取长期记忆 |
| `autoJournal` | true | 自动记玩家侧写 |
| `narration` | true | 对话间插入旁白 |
| `NARRATION_EVERY` | 2 | 每几轮插一段旁白 |
| `JOURNAL_EVERY` | 6 | 每积累几条消息记一次日志 |
| system 预算占比 | 45% | 超出按权重裁剪，每段保底 60 token |

### 8.2 好感与恋爱

| 档位 | 下限 | 表白成功率 |
| --- | --- | --- |
| 陌生 | -100 | 3% |
| 认识 | 10 | 10% |
| 熟悉 | 30 | 25% |
| 好友 | 55 | 60% |
| 挚交 | 80 | 85% |

好感变化：私聊中 NPC 说出台词 +2；玩家私聊中点名 +2；表白成功 +12；表白失败 −3；clamp(-100,100)。

### 8.3 记忆与世界书

| 项 | 值 |
| --- | --- |
| 事实卡检索 | `k=6`，打分 = 衰减分 ×(0.4+重合度)，过滤 `score>0.05` |
| 遗忘半衰期 | 7 天（`halfLifeDays`），`score × 0.5^(天数/半衰期)` |
| 事实卡上限 | 120 条（合并后按 score 截断） |
| 世界书扫描深度 | 最近 6 条消息 |
| 世界书最多命中 | 6 条 |
| 关键词计分 | `keys` 命中 +10，`secondaryKeys` +3；终分 = 分数 + `priority/100` + `random×0.5` |
| 日志上限 | 200 条；侧写 ≤500 字；每次最多 3 条观察 |

### 8.4 其它

| 项 | 值 |
| --- | --- |
| 置顶上限 | 6（`MAX_PINNED_CHARACTERS`） |
| 头像压缩 | 最长边 256，JPEG 0.85；原图 <60KB 且无需缩放时保留原图 |
| 用力导图 | 缩放 0.4~2.5，理想边长 130，阻尼 0.82 |
| 侧栏长按 | 300ms 触发拖拽，移动 >6px 取消 |
| 角色详情动画 | 380ms 飞入 |

---

## 9. 非功能需求、边界与已知问题

### 9.1 非功能需求

| 项 | 现状 |
| --- | --- |
| 隐私 | 纯前端，无后端；数据只在本机 localStorage；请求由浏览器直连玩家填写的 Base URL |
| 密钥 | 存浏览器本地；导出备份**不含** API Key；官方提示共享设备用完即删 |
| 可用性 | 无 Key 也能逛：世界观、角色卡编辑器、关系图、离线随机生成全部可用 |
| 离线 | 🎲 随机生成 / 离线随机毛坯完全不联网；Ollama 可全本地推理 |
| 成本 | Token 与费用为本地估算（`estimateTokens`：CJK×0.6、其余/4），**非精确计费**；Anthropic 流式不回传 usage |
| 浏览器 | 依赖 `fetch` 流式读取、`localStorage`、canvas（头像压缩）；浏览器直连可能受 CORS 限制 |
| 性能 | 关系图力导图每帧迭代；会话消息、事实卡、世界书均常驻内存（localStorage 约 5MB 上限，长会话与大世界需注意） |

### 9.2 边界与风险

| 风险 | 说明 / 缓解 |
| --- | --- |
| localStorage 容量 | 长会话 + 多世界可能超限；建议定期「导出全部数据」并清理旧会话 |
| 模型不守格式 | 世界书 / 事实卡 / 侧写 / 设定集的解析都有容错（去围栏、取首尾括号、失败静默），失败不影响主流程 |
| 台词被剥光 | 模型只输出动作时，整条会自动降级为旁白，不会留下空气泡 |
| 上下文膨胀 | 45% 预算 + 优先级裁剪 + 滚动摘要三重兜底 |
| 生成中途停止 | `AbortController` 中止，空占位消息自动清除 |
| 免费额度 / 限流 | 错误透传 HTTP 状态码与服务商信息，横幅提示 |

### 9.3 代码与文档/实现不一致（待修）

| # | 位置 | 问题 |
| --- | --- | --- |
| 1 | `SettingsView` 的 `GUIDE` 文案 | 仍写「气泡上的「说」= 让 TA 接着说一句」「角色详情四个签」，与实现（编辑/删除/重生成、八个签）不符 |
| 2 | `README.md` 的 World Pack 示例 | `rules.currency`、`codex.geography.text` 等为旧结构（现为 `factions[].currency`、数组条目） |
| 3 | `characterCard.ts` | 导出为 **CCv2**（非 CCv3）；`profile / faction / proactive` 不导出，导入他人卡会丢失这几项 |
| 4 | `fromCCv2` | `memory.halfLifeDays` 被 `clamp(0,1)` 压缩，导入后默认 7 天失效 |
| 5 | `ui.tsx` | `Modal` 组件已实现但无任何引用；`Avatar` 的 `color` 参数已声明但被忽略（头像不再上色） |
| 6 | `resetNavOrder` | 已定义但无 UI 入口 |
| 7 | `estimateCost` / `applyRelationshipDelta` | 已实现但未接入主流程（关系增量目前只由好感与恋爱驱动） |

---

## 附：源码索引

| 关注点 | 路径 |
| --- | --- |
| 应用外壳与板块路由 | `src/App.tsx` |
| 侧栏 / 会话区 / 新建会话 | `src/components/Sidebar.tsx`、`NewChatMenu.tsx` |
| 聊天 | `src/components/ChatView.tsx`、`RightPanel.tsx` |
| 世界观 | `src/components/WorldView.tsx` |
| 角色 | `src/components/CharactersView.tsx` |
| 关系图 | `src/components/RelationsView.tsx` |
| 旅人 / 日志 / 规则 / 设置 | `TravelerView.tsx` / `JournalView.tsx` / `RulesView.tsx` / `SettingsView.tsx` |
| 通用组件（14 个） | `src/components/ui.tsx` |
| 状态与全部 action | `src/store/useChatStore.ts` |
| 类型与常量 | `src/types/index.ts` |
| 模型接入 | `src/core/llm/{registry,openaiCompatible,anthropic,token,types}.ts` |
| 提示词 | `src/core/prompt/{builder,speech}.ts` |
| 世界生成 | `src/core/worldgen/{ai,brief,random,apply}.ts` |
| 记忆 / 社交 / 世界书 / 日志 | `src/core/memory/engine.ts`、`src/core/social/*`、`src/core/lorebook/matcher.ts`、`src/core/journal.ts` |
| 角色卡导入导出 / 头像 | `src/core/characterCard.ts`、`src/core/avatar.ts` |
| 官方世界 | `src/presets/worlds/witcherNorth.ts` |
| 现有文档 | `docs/PRD.md`、`docs/ARCHITECTURE.md`、`docs/WORLD_PACK.md`、`docs/CHARACTER_CARD.md` |
