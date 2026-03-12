# Echo Day 产品需求文档 (PRD)

## 1. 文档信息

| 项目 | 内容 |
| :--- | :--- |
| **产品名称** | Echo Day |
| **产品定位** | 以情绪与记忆为核心的 AI 日记应用 |
| **目标平台** | Web |
| **技术栈** | React 18 + TypeScript + Next.js 14 (App Router) + Tailwind CSS + Vercel |
| **AI 模型** | Anthropic Claude API + OpenAI Realtime API |
| **音乐** | Spotify Web API + Web Playback SDK |

---

## 2. 产品概述

### 2.1 产品定位与目标用户

**Echo Day** 是一款以**情绪和记忆**为核心的 AI 日记应用。名字取自两层含义：音乐的回响（Echo）和每日的记录（Day）。本产品旨在通过**照片 + AI 实时语音对话**的组合方式，解决用户"想记录但不愿动笔"的核心痛点，并借助情绪分析与音乐匹配技术，实现视觉、文字、听觉三维度的沉浸式记忆回溯体验。

区别于传统文字日记（需要主动写作）和相册（只有照片、缺乏情感叙述），Echo Day 让用户只需自然地说话，AI 就会自动整理、总结，生成一篇充满情感的日记，并为这一天配上最合适的音乐。

**目标用户：** 有记录生活意愿但懒得动笔，或希望日后重温某天情绪氛围的年轻用户。

### 2.2 核心价值主张

本应用的核心价值在于构建一个**从记录到回忆的情感闭环**：

1. **零门槛记录：** 通过实时双向语音对话替代手动写作，用户只需讲述当天经历，AI 自动生成完整日记，将录入摩擦降至最低。
2. **情绪驱动的音乐匹配：** 通过对日记内容进行情绪向量分析，将情绪参数映射为 Spotify 音频特征，召回与当日情感最匹配的曲目，使音乐成为情绪的延伸而非随机背景。
3. **沉浸式记忆回溯：** 重新打开某天记录时，照片、日记文字、音乐同时呈现，在视觉、文字、听觉三个维度同步重现当日情绪氛围。
4. **情绪可视化时间轴：** 每天的情绪颜色构成一条色彩轴线，让用户直观感受自身情绪随时间的流动与变化。

---

## 3. 核心功能模块

### 3.1 每日录制向导

每日记录是应用的核心交互入口，采用线性向导形式分为 5 个步骤，引导用户完成从照片到完整日记的全流程。

#### 3.1.1 照片采集与草稿创建

| 功能点 | 描述 | 交互细节与逻辑 |
| :--- | :--- | :--- |
| **拍摄/上传** | 支持调用相机直接拍摄或从相册选取 | `input[capture]` / File API，上传前本地 Canvas 预览，支持重选 |
| **云端存储** | 上传到 Supabase Storage，获取持久化路径 | `POST /api/photos/upload`，返回 `{ photoKey, photoUrl }` |
| **草稿即时创建** | 照片上传完成后立即在数据库创建 DRAFT 记录 | 用户中途退出后，下次进入向导时检测当日 DRAFT 并从断点续接 |

#### 3.1.2 AI 实时语音对话

该步骤通过 OpenAI Realtime API 建立 WebRTC 双向语音连接，用户自然讲述，AI 适时追问，模拟与朋友倾诉的体验。

| 功能点 | 描述 | 交互细节与逻辑 |
| :--- | :--- | :--- |
| **实时双向语音** | 用户说话，AI 实时语音回应 | OpenAI Realtime API（WebRTC），服务端生成临时 token，API Key 不暴露客户端 |
| **实时字幕显示** | 对话过程中同步渲染转录文字 | WebRTC DataChannel 接收 `response.audio_transcript.delta` 事件流 |
| **波形可视化** | 说话时显示实时音频波形动画 | Web Audio API `AnalyserNode`，`next/dynamic` 懒加载，仅在录制页按需初始化 |
| **结束对话** | 用户主动结束，触发后续步骤 | 前端 `wizardSlice` 状态机推进，完整转录文本 dispatch 至 Redux |

**AI 对话策略：** AI 以温和的好奇心引导叙述，聚焦"今天发生了什么"和"你当时的感受"，不做评判、不提建议，只倾听和追问。

#### 3.1.3 AI 日记生成与编辑

| 功能点 | 描述 | 交互细节与逻辑 |
| :--- | :--- | :--- |
| **AI 生成日记** | 基于对话转录 + 照片内容，Claude 生成约 200 字第一人称日记 | `POST /api/ai/diary`，Claude vision 解析照片 + 对话摘要，流式 SSE 输出 |
| **可编辑预览** | 用户可直接修改 AI 生成的文字 | 富文本 textarea，修改内容实时更新至 `wizardSlice` |

**Claude Prompt 设计原则：** 第一人称、口语化、情感真实、约 200 字、结尾留有余韵；避免"今天"开头，避免堆砌形容词。

#### 3.1.4 情绪颜色选择

| 功能点 | 描述 | 交互细节与逻辑 |
| :--- | :--- | :--- |
| **AI 情绪分析** | 基于日记文本输出结构化情绪向量 | `POST /api/ai/emotion`，返回 `{ emotionLabel, emotionVector: {valence, energy, complexity}, suggestedColorHex }` |
| **推荐色高亮** | 从预设 20 色情绪色盘中高亮推荐色 | 情绪向量 → 最近邻颜色映射，推荐色以脉冲动画标记 |
| **手动选色** | 用户可从色盘中自行选择，AI 推荐仅作参考 | `ColorPalette` 组件（与设置页共用，`readOnly` prop 控制），选择结果记录 `colorSource: 'AI' \| 'USER_MODIFIED'` |

**情绪色盘（20色）：** 按情绪象限分组，覆盖平静/满足（冷色调）、轻盈/愉悦（暖色调）、低沉/疲惫（灰暗色调）、激动/复杂（深饱和色调）、温柔/怀念（低饱和暖色调）五个情绪区域。

#### 3.1.5 Spotify 音乐匹配

| 功能点 | 描述 | 交互细节与逻辑 |
| :--- | :--- | :--- |
| **情绪 → 音频特征** | 将情绪向量映射为 Spotify 音频特征参数 | `lib/spotify/emotion-map.ts`：valence/energy/acousticness 等参数转换 |
| **Spotify 推荐** | 调用 Spotify Recommendations API，返回 3 首候选曲 | `POST /api/spotify/search`，同时获取每首曲目的 `previewUrl`（30s 片段） |
| **候选曲预听** | 悬停或点击可预听 30s 片段 | `HTMLAudioElement` 直接播放 `previewUrl`，无需 SDK |
| **用户选定** | 从 3 首中选择最合适的一首 | 选定结果（含 `spotifyPreviewUrl`）存入 `wizardSlice` 并持久化到数据库 |

---

### 3.2 主页 — 日历 + 时间轴双视图

主页是历史记录的浏览入口，支持两种视图通过顶部 `ViewToggle` 组件自由切换，切换状态存入 Redux `uiSlice`。

#### 3.2.1 日历视图

| 功能点 | 描述 | 交互细节与逻辑 |
| :--- | :--- | :--- |
| **月份网格** | 标准月历格式，有记录的日期显示情绪色点 | 按月请求最小投影数据（date, moodColorHex, photoUrl），RTK Query 缓存 |
| **照片悬浮预览** | 鼠标悬停日期格时弹出照片缩略图 + 2 行日记摘要 | `PhotoHoverCard` 组件，Supabase Storage `?width=200` 裁剪 URL |
| **月份导航** | 前后翻月，已加载月份从缓存读取，无重复请求 | RTK Query `providesTags` 月份级别缓存策略 |

#### 3.2.2 时间轴视图

| 功能点 | 描述 | 交互细节与逻辑 |
| :--- | :--- | :--- |
| **竖向卡片流** | 按日期倒序排列，每条记录是一张卡片 | `TimelineCard` 组件，复用 `MoodDot`、`DiaryPreview` 共享组件 |
| **虚拟滚动** | 大量记录时只渲染可见区域 | `@tanstack/react-virtual`，支持 700+ 条记录流畅滚动 |
| **懒加载分页** | 滚动到底部自动加载更多 | `useIntersection` Hook + RTK Query cursor 分页（20条/页） |

---

### 3.3 日记详情页

进入某天的详情页，是 Echo Day 最核心的情感体验。

| 功能点 | 描述 | 交互细节与逻辑 |
| :--- | :--- | :--- |
| **沉浸式入口遮罩** | 进入页面时显示全屏情绪色背景 + "进入这一天"按钮 | Framer Motion 全屏覆盖，引导用户做好情感准备，同时作为 autoplay 用户手势触发点 |
| **音乐自动播放** | 用户点击入口按钮，音乐立即响起，无需额外操作 | 用户手势合法解除浏览器 autoplay 限制（详见 4.2 技术难点） |
| **全屏照片背景** | 当天照片铺满背景，叠加情绪色渐变蒙层 | `DayHero` 组件，CSS backdrop-filter 处理对比度 |
| **日记文字呈现** | 照片之上展示 AI 生成的日记全文 | `DiaryText` 组件，Server Component 直接读取数据库，减少客户端 JS |
| **页面进入动画** | 日历情绪色点 → 详情页全屏背景的连续动画 | Framer Motion `layoutId` layout animation，保持视觉连续性 |

**音乐播放策略：**

| 用户类型 | 播放方式 |
| :--- | :--- |
| Spotify Premium 用户 | Spotify Web Playback SDK，全曲播放，品质最高 |
| Spotify 普通用户 | `HTMLAudioElement` 播放存储的 30s `previewUrl` |
| 未连接 Spotify | 日记与照片正常展示，音乐功能隐藏，提示连接 Spotify |

---

### 3.4 用户与权限系统

| 模块 | 关键功能 |
| :--- | :--- |
| **注册/登录** | Google OAuth；JWT 鉴权（NextAuth.js） |
| **Spotify 连接** | 独立 OAuth 流程，可随时连接/断开；登录时检测 `product` 字段记录 Premium 状态 |
| **用户档案** | 头像、昵称、Spotify 连接状态 |
| **记录管理** | 查看历史记录、删除某天记录 |
| **路由权限** | 双层守卫：Edge Middleware 请求级拦截 + Layout Server Component 细粒度权限（详见 4.3.4） |

---

## 4. 技术架构与实现

### 4.1 技术栈推荐

| 模块 | 技术栈 | 作用与优势 |
| :--- | :--- | :--- |
| **前端框架** | React 18 + TypeScript + Next.js 14 (App Router) | Server Components 减少客户端 JS；App Router 支持路由级 Layout 权限守卫 |
| **样式** | Tailwind CSS + shadcn/ui | 原子化 CSS 与高质量组件库，快速构建情绪主题动态配色 |
| **状态管理** | Redux Toolkit + RTK Query | `createSlice` 管理向导状态机；RTK Query 处理服务端数据缓存与标签失效 |
| **动画** | Framer Motion | `layoutId` 实现日历色点 → 全屏背景的物理连续过渡动画 |
| **实时语音** | OpenAI Realtime API（WebRTC） | 低延迟双向语音，DataChannel 流式传输转录文本 |
| **AI 日记/情绪** | Anthropic Claude API | Claude vision 解析照片内容；结构化 JSON 输出情绪向量 |
| **音乐** | Spotify Web API + Web Playback SDK | Recommendations API 情绪召回；SDK 实现浏览器内 Premium 全曲播放 |
| **数据库** | PostgreSQL（Supabase） | 原生 Date 类型支持 `@@unique([userId, date])` 约束；连接池分离 migrations |
| **ORM** | Prisma | 类型安全 DB 操作，自动生成 TypeScript 类型 |
| **认证** | NextAuth.js | 统一处理 Google OAuth + Spotify OAuth，JWT session |
| **文件存储** | Supabase Storage | 照片存储 + CDN 分发；URL 参数支持服务端图片裁剪 |
| **部署** | Vercel | Edge Network + 自动 CI/CD，Next.js 原生优化 |

### 4.2 关键技术难点总结

| 难点 | 方案描述 |
| :--- | :--- |
| **OpenAI Realtime API WebRTC 接入** | 服务端 `POST /api/realtime/token` 生成临时凭证（ephemeral token），前端创建 `RTCPeerConnection`，通过 DataChannel 接收 `response.audio_transcript.delta` 事件流拼接实时字幕，API Key 全程不暴露客户端 |
| **浏览器 Autoplay 策略** | 详情页入口遮罩的"进入这一天"点击本身是用户手势，在该 click handler 内调用 `player.activateElement()`（Premium SDK）或 `audio.play()`（Free 预览），合法触发音频播放，无需用户额外点击播放键 |
| **Spotify Premium / Free 播放分支** | 登录时调用 `/v1/me` 检测 `product === 'premium'` 并写入数据库；Premium 走 Web Playback SDK 全曲，Free 走创建记录时存储的 `previewUrl` 30s 片段；`next/dynamic + { ssr: false }` 懒加载 SDK，非 Premium 用户零开销 |
| **Spotify Token 安全存储** | access/refresh token 使用 AES-256 加密后入库，服务端 `/api/spotify/token` 路由解密使用，明文不落库、不暴露客户端 |
| **Hook 职责边界** | `useRealtime`、`useSpotifyPlayer` 等自定义 Hook 只封装副作用，不直接操作 Redux；转录结果、SDK 事件由组件层接收后 dispatch，保持 Hook 的单一职责和可测试性 |
| **向导中途恢复** | 照片上传完成即创建 `status: DRAFT` 记录；`RecordWizard` mount 时查询当日是否存在 DRAFT，存在则从断点续接，避免用户重新上传照片 |

### 4.3 后端架构与数据库设计

#### 4.3.1 后端系统架构概述

Echo Day 后端基于 **Next.js Route Handlers** 实现 RESTful API，通过以下核心服务协作完成主要业务逻辑：

1. **Auth Service（NextAuth.js）：** 统一处理 Google OAuth + Spotify OAuth，管理 JWT session 与用户信息写入。
2. **Realtime Token Service（`/api/realtime/token`）：** 服务端向 OpenAI 请求临时 WebRTC 凭证，验证用户身份并做每日使用量限流。
3. **AI Pipeline Service（`/api/ai/diary`、`/api/ai/emotion`）：** 串联 Claude vision 照片分析、对话转录摘要，输出日记文本与结构化情绪向量。
4. **Spotify Service（`/api/spotify/*`）：** 封装情绪向量 → 音频特征映射、Recommendations API 调用、access token 加密刷新。
5. **Storage Service（`/api/photos/upload`）：** 接收照片上传，写入 Supabase Storage，返回持久化路径。
6. **Entries CRUD（`/api/entries/*`）：** 日记记录的增删改查，向导各步通过 PATCH 渐进更新同一条记录。

#### 4.3.2 数据库设计（核心表）

| 表名 | 描述 | 关键字段 | 备注 |
| :--- | :--- | :--- | :--- |
| `users` | 用户基础信息与 Spotify 认证 | `id` (PK), `email`, `googleId`, `spotifyId`, `spotifyAccessToken`（加密）, `spotifyRefreshToken`（加密）, `spotifyTokenExpiry`, `spotifyPremium` | Spotify token AES-256 加密存储 |
| `diary_entries` | 日记核心记录 | `id` (PK), `userId` (FK), `date`（DB Date 类型）, `photoUrl`, `photoKey`, `transcript`, `diaryText`, `emotionLabel`, `emotionVector`（JSON）, `moodColorHex`, `colorSource`（Enum）, `spotifyTrackId`, `spotifyPreviewUrl`, `status`（Enum） | `@@unique([userId, date])` 保证一人一天唯一；`status: DRAFT \| COMPLETE` 支持向导中途恢复 |

**关键约束设计：**
- `@@unique([userId, date])` + `@@index([userId, date])`：既保证业务唯一性，又支持日历月份查询的 O(1) 精确定位
- `emotionVector` 存为 JSON：情绪维度（valence/energy/complexity）可灵活扩展，不需要 schema migration
- `spotifyPreviewUrl` 在记录创建时存储：避免 Spotify URL 过期导致详情页播放失败，服务端可按需重新获取

#### 4.3.3 核心 API 接口定义

| 模块 | 接口/路径 | 方法 | 描述 |
| :--- | :--- | :--- | :--- |
| **认证** | `/api/auth/[...nextauth]` | — | NextAuth 统一处理 Google + Spotify OAuth |
| **日记** | `/api/entries` | GET | 获取月份记录列表（最小投影：date, moodColorHex, photoUrl, emotionLabel） |
| **日记** | `/api/entries` | POST | 创建 DRAFT 记录，返回 `entry_id` |
| **日记** | `/api/entries/[id]` | GET | 获取某天完整日记详情 |
| **日记** | `/api/entries/[id]` | PATCH | 向导各步渐进更新（diaryText / moodColor / spotifyTrack / status） |
| **日记** | `/api/entries/[id]` | DELETE | 删除记录（同时清理 Storage 照片） |
| **语音** | `/api/realtime/token` | POST | 生成 OpenAI WebRTC 临时 token，服务端验证身份 + 限流 |
| **AI** | `/api/ai/diary` | POST | `{ transcript, photoUrl }` → Claude 日记生成（SSE 流式响应） |
| **AI** | `/api/ai/emotion` | POST | `{ diaryText }` → `{ emotionLabel, emotionVector, suggestedColorHex }` |
| **音乐** | `/api/spotify/search` | POST | `{ emotionVector }` → Spotify Recommendations，返回 3 首候选（含 previewUrl） |
| **音乐** | `/api/spotify/token` | POST | 服务端解密 + 刷新 Spotify access token |
| **存储** | `/api/photos/upload` | POST | 照片上传到 Supabase Storage，返回 `{ photoKey, photoUrl }` |

#### 4.3.4 关键业务逻辑实现

**1. 路由权限双层守卫**

- **Edge Middleware（`middleware.ts`）：** 请求到达前验证 JWT token，未登录直接重定向 `/login`，无需进入 React 层，匹配规则 `/(app)/:path*`。
- **Layout Server Component（`app/(app)/layout.tsx`）：** 读取完整 session，注入 Provider tree（`ReduxProvider → SpotifySDKProvider`），`SpotifySDKProvider` 根据 `spotifyPremium` 字段决定是否加载 Spotify SDK。`(auth)` 与 `(app)` 路由组通过 Next.js App Router Group Layout 共享不同 Provider tree，避免重复渲染。

**2. 录制向导状态机（`wizardSlice`）**

向导状态机管理 7 个步骤状态：`PHOTO → VOICE → DIARY → COLOR → MUSIC → SAVING → COMPLETE`。每个步骤数据（photoUrl、transcript、diaryText、emotionVector、moodColorHex、spotifyTrack）存入 Redux，确保步骤间数据流畅传递。`extraReducers` 监听 RTK Query `saveEntry.matchFulfilled` 事件，自动推进状态至 `COMPLETE`。

**3. Redux 状态分层原则**

| 状态类型 | 存放位置 | 理由 |
| :--- | :--- | :--- |
| 用户信息、Spotify Premium 状态 | `authSlice` (Redux) | 全局读取，多组件依赖 |
| 录制向导进度 + 各步数据 | `wizardSlice` (Redux) | 跨 5 个步骤组件传递，需要在步骤间持久 |
| 主页视图切换偏好 | `uiSlice` (Redux) | ViewToggle 写，Calendar/Timeline 读 |
| Spotify 播放状态 | `playerSlice` (Redux) | SpotifyPlayer 写，顶部迷你播放条读 |
| 服务端数据（日记列表/详情） | RTK Query (`entriesApi`) | 自动缓存、标签失效、切月读缓存无重复请求 |
| WebRTC 连接、音频流对象 | `useRef`（Hook 内部） | 副作用不序列化，不放 Redux；`useEffect` cleanup 负责关闭连接 |
| 组件内动画临时状态 | `useState` | 不跨组件共享，无需全局 |

**4. 性能优化策略**

| 优化场景 | 手段 |
| :--- | :--- |
| 日历月份数据 | RTK Query 缓存，切月读缓存，`providesTags` 精确失效 |
| 时间轴长列表 | `@tanstack/react-virtual` 虚拟滚动，700+ 条记录保持 60fps |
| 照片加载 | Supabase Storage URL 追加 `?width=200&height=200`；Next.js `<Image>` 自动 WebP + lazy |
| Spotify SDK | `next/dynamic + { ssr: false }` 按需加载，非 Premium 用户零开销 |
| VoiceVisualizer | `next/dynamic` 懒加载，Web Audio API 仅在 record 页初始化 |
| 详情页数据 | Server Component 直接读数据库，减少客户端水合 JS |

---

## 5. 商业模式与合规

### 5.1 权限与付费模式（RBAC）

基于 RBAC 权限系统设计以下分级付费模式：

| 版本 | 目标用户 | 核心权益 | 限制与差异 |
| :--- | :--- | :--- | :--- |
| **免费版** | 体验用户、轻度记录者 | 基础录制向导；日历 + 时间轴浏览；30s 音乐预听 | 每日 AI 语音对话时长上限；无 Spotify Premium 全曲播放；无数据导出 |
| **Pro 版** | 重度记录、情感需求强的用户 | 无限制录制时长；Spotify Premium 全曲自动播放；历史记录数据导出 | 无次数限制，享有全部高级功能 |

### 5.2 数据合规与成本控制

1. **数据合规：** 用户语音对话转录文本及情绪数据属于敏感个人数据，隐私协议中须明确告知使用目的，提供 **数据用于模型优化的 Opt-in/Opt-out 选项**。Spotify access/refresh token 使用 AES-256 加密后入库，明文不落库。
2. **成本控制：** 服务端对 `/api/realtime/token` 接口实施每日每用户调用次数限流，防止 OpenAI Realtime API 滥用；免费用户限制对话时长而非次数，降低单次成本；图片上传服务端校验文件类型与大小上限（10MB），防止存储滥用。

---

## 6. 交互设计要点

1. **情绪色全局动态主题：** 界面无固定品牌色，每天的情绪色作为全局主题色动态注入，日历色点、详情页背景、音乐播放条配色均响应当天情绪，用户在不同天的记录中体验到截然不同的视觉氛围。
2. **物理连续的页面过渡：** 从日历/时间轴进入详情页时，情绪色点通过 Framer Motion `layoutId` 平滑扩展为全屏背景，避免路由切换导致的视觉割裂感；详情页内容在色块展开后顺序淡入，营造"走进那一天"的沉浸感。
3. **入口遮罩的双重作用：** 详情页的全屏情绪色入口遮罩既是情感准备的仪式感设计，也是解决浏览器 autoplay 策略的工程方案——"进入这一天"的点击行为即为音乐播放的合法用户手势。
4. **向导步骤的进度反馈：** 录制向导顶部显示步骤进度指示器，每步完成后以动画标记，让用户始终清楚当前位置和剩余步骤，降低长流程的焦虑感。
5. **音乐候选曲的即时预听：** MusicStep 中 3 首候选曲悬停即触发 30s 预听，无需跳转 Spotify，降低选择决策成本，同时通过封面图和歌手信息辅助情感判断。

---

## 7. 设计风格

1. 情绪驱动配色，界面无固定品牌色，每天有自己的颜色。
2. 照片优先，排版以照片为视觉中心展开，文字为照片的情感注脚。
3. 克制的文字排版，日记文字使用舒适行高与字间距，突出阅读体验。
4. 组件抽象化，`MoodDot`、`DiaryPreview`、`ColorPalette` 等核心组件跨页面复用，封装性好。
5. 页面过渡平滑，路由切换均有 Framer Motion 动画，杜绝白屏闪烁。
