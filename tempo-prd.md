# Tempo App 产品需求文档 (PRD)

## 1. 文档信息

| 项目 | 内容 |
| :--- | :--- |
| **产品名称** | Tempo |
| **产品定位** | AI 驱动的生活节奏助手 |
| **目标平台** | Web (PWA) |
| **技术栈** | React + TypeScript + Next.js 14 (App Router) + Tailwind CSS + Vercel |
| **AI 模型** | OpenAI GPT-4o / Anthropic Claude API |

---

## 2. 产品概述

### 2.1 产品定位与目标用户

**Tempo** 是一款帮助用户对抗拖延、建立生活节奏的 AI 陪伴型应用。核心交互方式是 **AI 对话**，AI 通过**情绪安抚 + 动作拆分 + 时间锚定**的话术策略，帮助用户在每个行动转换节点启动下一步。

区别于 TodoList（管理任务完成）和番茄钟（被动计时），Tempo 聚焦于帮助用户**"开始"行动**，并通过闹钟提醒、跨会话记忆、Routine 复盘持续跟进。

### 2.2 核心价值主张

1. **消灭启动摩擦：** AI 将模糊的行动拆解为最小物理步骤，降低启动心理门槛。
2. **精准时间锚定：** AI 获取实时时间，将时间焦虑转化为具体时间线——"现在 10:00，洗澡 20 分钟，10:20 你就自由了"。
3. **健康的休息替代：** 用 AI 对话替代刷短视频/社交媒体，AI 到时间主动把用户拉回正轨。
4. **Routine 固化与复盘：** AI 自动总结用户的日常模式，帮助建立稳定的生活节奏。

---

## 3. 核心功能模块

### 3.1 AI 对话引擎

#### 3.1.1 对话模式与交互设计

AI 对话是整个应用的第一交互入口，采用聊天气泡样式。

| 功能点 | 描述 | 技术要点 |
| :--- | :--- | :--- |
| **文字聊天** | 聊天气泡 UI，AI 流式输出 | SSE (Server-Sent Events) |
| **语音聊天** | 语音输入 + AI 语音回复 | MVP: Web Speech API；进阶: Whisper + TTS API |
| **实时时间感知** | AI 在每次请求中获取客户端当前时间，实现时间锚定 | 前端将 `new Date()` 注入每次 API 请求，拼接进 System Prompt |
| **任务上下文** | AI 自动读取用户的 TodoList，在对话中基于任务列表进行安排和引导 | 每次对话请求携带当日任务列表 JSON |
| **跨会话记忆** | 前一天晚上的对话和计划延续到次日 | 每日对话生成摘要存入 DB，次日加载到 System Prompt |
| **对话历史** | 按天分组查看历史记录 | 无限滚动 + 按日期筛选 |

#### 3.1.2 AI 人格策略

AI 定位为**温和但坚定的朋友**，核心话术三要素：

| 策略 | 描述 |
| :--- | :--- |
| **情绪安抚** | 先共情再引导，降低行动心理门槛 |
| **微动作拆分** | 将行动拆解为最小物理步骤（"先翻个身→坐起来→脚放地上"） |
| **时间锚定** | 每次引导附带基于实时时间的具体估算（"现在做的话 11:05 就能完成"） |
| **正向反馈** | 用户完成行动后立即肯定，强化"做到了"的体验 |
| **跨时段连续性** | 引用之前的对话和计划，保持记忆感 |
| **个性化话术** | 根据用户偏好切换温柔鼓励型 / 直接督促型 |

#### 3.1.3 对话中的闹钟设定

闹钟通过 AI 对话自然触发，而非用户手动设置时间。

**流程：** AI 在对话中提议 → 发送含确认按钮的气泡 → 用户点击确认 → 前端创建定时器。

通过 LLM **Function Calling** 从 AI 回复中提取闹钟指令：

```json
{
  "type": "alarm_request",
  "delay_minutes": 5,
  "message": "5 分钟到了！该回去学习了",
  "context": "用户正在休息"
}
```

### 3.2 任务列表（TodoList）

用于让 AI 知晓用户当天的任务，AI 在对话中基于任务列表进行安排和引导。

| 功能点 | 描述 | 技术要点 |
| :--- | :--- | :--- |
| **任务 CRUD** | 用户手动添加、编辑、删除、完成任务 | 标准列表操作 |
| **AI 读取** | AI 对话时自动获取当日任务列表作为上下文 | 任务列表 JSON 注入 System Prompt |
| **AI 创建任务** | 用户在对话中说"我明天要做 X"，AI 通过 Function Calling 自动创建任务 | 结构化输出解析 |
| **任务排序** | 支持手动拖拽排序 | Framer Motion drag |
| **日期分组** | 按日期查看任务，支持为明天预设任务 | 日期筛选 |

### 3.3 智能闹钟系统

| 功能点 | 描述 | 技术实现 |
| :--- | :--- | :--- |
| **AI 对话触发** | 闹钟由 AI 对话中的 Function Calling 指令创建 | 结构化输出提取参数 |
| **定时响铃** | 到时间播放提示音 + 全屏覆盖 AI 消息 | Web Worker 定时 + Web Audio API |
| **手动关闭** | 必须点击"我知道了"才能关闭 | 全屏 Modal 覆盖层 |
| **跨天闹钟** | 支持晚上设定次日闹钟（如起床闹钟） | 闹钟持久化到 DB，Service Worker 触发 |
| **渐进音量** | 未关闭时音量逐渐增大 | Web Audio API GainNode |
| **PWA 推送** | 后台/锁屏时推送通知 | Service Worker + Notification API |
| **自定义提示音** | 可选不同提示音 | 预置音频文件 |

### 3.4 个性化引擎

#### 3.4.1 显式设置（Onboarding 问卷）

| 维度 | 数据用途 |
| :--- | :--- |
| 作息习惯（起床/睡觉时间） | AI 引导的时间基准 |
| 拖延触发场景 | AI 在对应节点加强引导 |
| 上瘾源（短视频/社交媒体/游戏等） | AI 休息结束时针对性引导 |
| 激励偏好（温柔/直接/数据驱动） | 调整 AI System Prompt |
| 日常事项时长（洗澡/做饭等） | 时间估算初始参数 |

#### 3.4.2 隐式学习

| 数据维度 | 学习目标 |
| :--- | :--- |
| 实际用时 vs 预估用时 | 校准用户真实时长 |
| 启动延迟 | 识别启动困难最严重的事项 |
| 休息超时频率 | 调整提醒策略力度 |
| 话术有效性 | 学习最有效的引导方式 |

#### 3.4.3 Prompt 动态注入

每次 AI 调用前，从数据库读取**用户档案 + 当日任务列表 + 昨日对话摘要 + 行为数据 + 当前时间**，动态拼接 System Prompt。

### 3.5 Routine 自动总结与复盘

| 功能点 | 描述 | 触发方式 |
| :--- | :--- | :--- |
| **每日总结** | AI 自动生成当天行为总结（做得好/可改进/趋势） | Cron Job，睡前 30 分钟生成 |
| **周报** | 一周数据汇总 + Routine 模式识别 | Cron Job，每周日生成 |
| **Routine 固化** | AI 发现连续 3+ 天有效模式时，提议存为模板 | AI 在对话中主动提出 |
| **模板管理** | 查看/编辑/删除 routine 模板 | 设置页面 |
| **模板自动加载** | 对应场景时 AI 加载模板辅助引导 | AI 在对话中应用 |

### 3.6 用户与权限系统

| 模块 | 关键功能 |
| :--- | :--- |
| **注册/登录** | 邮箱 + Google/GitHub OAuth；JWT 鉴权（NextAuth.js） |
| **用户档案** | Onboarding 结果 + 偏好设置 + 行为数据 |
| **RBAC 权限** | Free（每日 20 轮对话、无语音）/ Pro（无限对话、语音、数据导出） |
| **设置** | AI 偏好、通知开关、Routine 管理、提示音选择 |

---

## 4. 技术架构与实现

### 4.1 技术栈

| 模块 | 技术栈 | 作用 |
| :--- | :--- | :--- |
| **前端框架** | React 18 + TypeScript + Next.js 14 (App Router) | SSR/SSG、Server Components |
| **样式** | Tailwind CSS + shadcn/ui | 组件化设计系统 |
| **状态管理** | Redux Toolkit + RTK Query | 全局状态 + API 缓存同步 |
| **实时通信** | SSE / WebSocket | AI 流式响应 / 语音双工通信 |
| **音频** | Web Audio API + Web Speech API | 闹钟播放 + 语音识别合成 |
| **PWA** | next-pwa + Service Worker | 离线缓存、推送通知 |
| **动画** | Framer Motion | 拖拽排序、页面过渡 |
| **后端** | Next.js Route Handlers | RESTful API |
| **数据库** | PostgreSQL (Supabase) | 关系型数据存储 |
| **ORM** | Prisma | 类型安全 DB 操作 |
| **认证** | NextAuth.js | OAuth + JWT |
| **AI** | OpenAI GPT-4o / Claude API | 对话引擎 + Function Calling |
| **部署** | Vercel | CI/CD + Cron Jobs |
| **支付** | Stripe | 订阅付费 |

### 4.2 后端数据库设计

#### 4.2.1 核心数据表

| 表名 | 描述 | 关键字段 |
| :--- | :--- | :--- |
| `users` | 用户信息 | `id`, `email`, `name`, `avatar_url`, `plan` (Free/Pro), `created_at` |
| `user_profiles` | 个性化档案 | `user_id` (FK), `wake_time`, `sleep_time`, `procrastination_triggers` (JSON), `motivation_style`, `addiction_sources` (JSON), `task_durations` (JSON) |
| `todos` | 任务列表 | `id`, `user_id` (FK), `title`, `date`, `completed`, `order_index`, `created_at` |
| `chat_messages` | 对话消息 | `id`, `user_id` (FK), `role` (user/assistant/system), `content`, `message_type` (text/voice/alarm), `created_at` |
| `chat_summaries` | 每日对话摘要 | `id`, `user_id` (FK), `date`, `summary_text`, `plans_for_tomorrow` (JSON) |
| `alarms` | 闹钟 | `id`, `user_id` (FK), `trigger_at`, `message`, `context`, `status` (pending/fired/dismissed) |
| `daily_summaries` | AI 每日/周总结 | `id`, `user_id` (FK), `date`, `type` (daily/weekly), `summary_json` (JSON) |
| `routines` | Routine 模板 | `id`, `user_id` (FK), `name`, `trigger_scene`, `steps` (JSON), `avg_duration`, `created_at` |
| `achievements` | 成就记录 | `id`, `user_id` (FK), `achievement_type`, `unlocked_at` |

#### 4.2.2 核心 API 接口

| 模块 | 接口 | 方法 | 描述 |
| :--- | :--- | :--- | :--- |
| 认证 | `/api/auth/[...nextauth]` | — | NextAuth 统一处理 |
| 对话 | `/api/chat` | POST (SSE) | 发送消息，返回 AI 流式响应 |
| 对话历史 | `/api/chat/history` | GET | 分页 + 按日期筛选 |
| 任务 | `/api/todos` | GET / POST | 获取/创建任务 |
| 任务 | `/api/todos/[id]` | PATCH / DELETE | 更新/删除任务 |
| 任务 | `/api/todos/reorder` | PATCH | 拖拽排序 |
| 用户档案 | `/api/user/profile` | GET / PATCH | 获取/更新设置 |
| Routine | `/api/routines` | GET / POST / PATCH / DELETE | 模板 CRUD |
| 总结 | `/api/summary` | GET | 获取每日/周总结 |
| 成就 | `/api/achievements` | GET | 成就列表 |

### 4.3 关键技术难点

| 难点 | 解决方案 |
| :--- | :--- |
| **AI 实时时间感知** | 前端每次请求将 `new Date().toISOString()` 注入请求体，后端拼接到 System Prompt |
| **AI 结构化输出** | Function Calling 提取闹钟指令、任务创建指令等结构化数据 |
| **后台计时精度** | Web Worker 不受标签页可见性影响；Service Worker Push 保底 |
| **对话 Token 控制** | 每 N 轮生成摘要，后续只传摘要 + 最近 K 轮完整对话 |
| **跨会话记忆** | `chat_summaries` 表存每日摘要 + 次日计划，次日加载到 System Prompt |
| **离线闹钟** | Service Worker 缓存音频 + 本地 Notification |
| **Routine 模式识别** | Cron Job 分析连续多天任务完成数据，识别重复模式 |

---

## 5. 商业模式

### 5.1 分级付费

| 版本 | 权益 | 限制 |
| :--- | :--- | :--- |
| **Free** | 文字对话（每日 20 轮）；基础闹钟；每日总结 | 无语音；无数据导出 |
| **Pro ($4.99/月)** | 无限对话；语音模式；完整总结与复盘；数据导出 | — |

### 5.2 成本控制

| 策略 | 说明 |
| :--- | :--- |
| Token 配额 | Free 用户限制每日调用次数 |
| 对话摘要 | 长对话自动摘要减少 Token |
| 模型分级 | Free: GPT-4o-mini；Pro: GPT-4o |

---

## 6. 设计风格

- 温暖柔和配色，移动端优先
- 组件抽象化，封装性好
- 页面过渡平滑动效（Framer Motion）
- 底部固定导航栏（Chat / Todos / Settings）
