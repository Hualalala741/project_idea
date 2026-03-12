# Echo Day 产品需求文档 (PRD)

## 1. 文档信息

| 项目 | 内容 |
| :--- | :--- |
| **产品名称** | Echo Day |
| **产品定位** | 以情绪与记忆为核心的 AI 日记应用 |
| **目标平台** | Web (PWA)，移动端优先 |
| **技术栈** | React 18 + TypeScript + Next.js 14 (App Router) + Tailwind CSS + Vercel |
| **AI 模型** | Anthropic Claude API + OpenAI Realtime API |
| **音乐** | Spotify Web API + Web Playback SDK |

---

## 2. 产品概述

### 2.1 产品定位与目标用户

**Echo Day** 是一款以**情绪和记忆**为核心的 AI 日记应用。名字取自两层含义：音乐的回响（Echo）和每日的记录（Day）。

区别于传统文字日记（需要主动写作）和相册（只有照片、缺乏情感叙述），Echo Day 通过**照片 + AI 语音对话**的组合方式，让用户只需要自然地说话，AI 就会自动整理、总结，生成一篇充满情感的日记，并为这一天配上最合适的音乐。

**目标用户：** 有记录生活意愿但懒得动笔，或希望在日后重温某天情绪氛围的年轻用户。

### 2.2 核心价值主张

1. **零门槛记录：** 只需拍一张照片，说几句话，AI 自动生成完整日记。
2. **情绪驱动的音乐匹配：** 音乐不是随机的，是根据日记情绪专门挑选的，强化当天的感受。
3. **沉浸式回忆：** 重新打开某天的记录，照片 + 文字 + 音乐同时呈现，在视觉、文字、听觉三个维度重回那一天。
4. **情绪可视化时间轴：** 每天的情绪颜色构成一条色彩时间轴，让用户看见自己情绪的流动。

---

## 3. 核心功能模块

### 3.1 每日录制向导（5步流程）

每日记录是应用的核心交互，采用线性向导形式，分为5个步骤。

#### 步骤 1 — 照片

| 功能点 | 描述 | 技术要点 |
| :--- | :--- | :--- |
| **拍摄/上传** | 支持调用相机直接拍摄或从相册选取 | `input[capture]` / File API |
| **即时预览** | 上传前可预览，支持重新选取 | Canvas 本地预览 |
| **云端存储** | 上传到 Supabase Storage，获取持久化 URL | `POST /api/photos/upload` |
| **草稿创建** | 照片上传完成后立即在数据库创建 DRAFT 记录 | `POST /api/entries`，支持中途恢复 |

#### 步骤 2 — 语音对话

| 功能点 | 描述 | 技术要点 |
| :--- | :--- | :--- |
| **实时双向语音** | 用户说话，AI 实时回应，像和朋友聊天一样记录今天 | OpenAI Realtime API（WebRTC） |
| **临时 token** | 服务端生成 WebRTC 临时凭证，API Key 不暴露到客户端 | `POST /api/realtime/token` |
| **实时转录字幕** | 对话过程中同步显示文字字幕 | WebRTC DataChannel 事件流 |
| **波形可视化** | 说话时显示音频波形动画，提升体验感 | Web Audio API AnalyserNode |
| **结束对话** | 用户主动结束或沉默超时，触发后续步骤 | 前端状态机控制 |

**AI 对话策略：** AI 以温和的好奇心引导用户叙述，聚焦"今天发生了什么"和"你当时的感受"，不做评判，不提建议，只是倾听和追问。

#### 步骤 3 — 日记预览与编辑

| 功能点 | 描述 | 技术要点 |
| :--- | :--- | :--- |
| **AI 生成日记** | 基于对话转录 + 照片内容，Claude 生成约 200 字的第一人称日记 | `POST /api/ai/diary`，Claude vision + 对话摘要 |
| **流式输出** | 日记文字逐字流式显示，减少等待感 | SSE 流式响应 |
| **简单编辑** | 用户可直接修改 AI 生成的文字 | 富文本 textarea |

**Claude Prompt 设计原则：** 第一人称、口语化、情感真实、约 200 字、结尾留有余韵。不使用"今天"开头，不堆砌形容词。

#### 步骤 4 — 情绪颜色

| 功能点 | 描述 | 技术要点 |
| :--- | :--- | :--- |
| **AI 情绪分析** | 基于日记文本分析情绪，输出结构化的情绪向量 | `POST /api/ai/emotion`，返回 `{ emotionLabel, emotionVector, suggestedColorHex }` |
| **AI 推荐颜色** | 从预设情绪色盘中推荐最匹配的颜色，脉冲高亮显示 | 情绪向量 → 颜色映射算法 |
| **手动选色** | 用户可从 20 色情绪色盘中自行选择，AI 推荐仅作参考 | `ColorPalette` 组件（可复用） |
| **记录来源** | 区分 AI 推荐还是用户修改，存入 `colorSource` 字段 | `'AI' \| 'USER_MODIFIED'` |

**情绪色盘设计（20色）：** 按情绪维度分组，覆盖从平静（冷色调）到激动（暖色调）、从轻盈（高亮度）到沉重（低亮度）的情绪象限。

#### 步骤 5 — 音乐匹配

| 功能点 | 描述 | 技术要点 |
| :--- | :--- | :--- |
| **情绪 → 音频特征** | 将情绪向量映射为 Spotify 音频特征参数（valence、energy、acousticness 等） | `lib/spotify/emotion-map.ts` |
| **Spotify 推荐** | 调用 Spotify Recommendations API，返回 3 首候选曲 | `POST /api/spotify/search` |
| **30s 预听** | 鼠标悬停或点击候选曲时可预听 30s 片段 | `HTMLAudioElement` + Spotify previewUrl |
| **用户选定** | 用户从 3 首中选择最合适的一首，或跳过 | 选择结果存入 wizardSlice |

#### 保存完成

所有步骤完成后，一键保存，将完整记录持久化到数据库（状态从 `DRAFT` 变为 `COMPLETE`），自动跳转到当天的详情页。

---

### 3.2 主页 — 日历 + 时间轴双视图

主页是记录的浏览入口，支持两种视图自由切换。

#### 3.2.1 日历视图

| 功能点 | 描述 | 技术要点 |
| :--- | :--- | :--- |
| **月份网格** | 标准月历格式，有记录的日期显示情绪色点 | 按月请求数据，最小投影（date, moodColorHex） |
| **照片悬浮预览** | 鼠标悬停日期格时，弹出当天照片缩略图 + 2 行日记摘要 | `PhotoHoverCard`，Supabase Storage 缩略图 URL |
| **月份导航** | 前后翻月，已加载的月份数据从缓存读取 | RTK Query `providesTags` 缓存策略 |
| **今日标记** | 今天的日期格有特殊样式，引导用户记录 | 日期比较逻辑 |

#### 3.2.2 时间轴视图

| 功能点 | 描述 | 技术要点 |
| :--- | :--- | :--- |
| **竖向卡片流** | 按日期倒序排列，每条记录是一张卡片 | `TimelineCard` 组件 |
| **卡片内容** | 照片缩略图 + 情绪标签 + 日记摘要 + 音乐曲名 | 复用 `MoodDot`、`DiaryPreview` 共享组件 |
| **虚拟滚动** | 大量记录时只渲染可见区域，保持流畅 | `@tanstack/react-virtual` |
| **懒加载分页** | 滚动到底部自动加载更多（20条/页） | `useIntersection` Hook + RTK Query cursor 分页 |

#### 3.2.3 视图切换

主页顶部的 `ViewToggle` 组件控制视图切换状态，存入 Redux `uiSlice`，刷新后恢复上次视图偏好。

---

### 3.3 日记详情页

进入某天的详情页，是 Echo Day 最核心的情感体验。

| 功能点 | 描述 | 技术要点 |
| :--- | :--- | :--- |
| **沉浸式入口** | 进入页面时显示全屏情绪色遮罩 + "进入这一天" 按钮，让用户做好情感准备 | Framer Motion 全屏覆盖层 |
| **自动播放音乐** | 用户点击"进入"按钮（用户手势触发），音乐立即响起，无需额外点击 | 用户手势解除浏览器 autoplay 限制 |
| **全屏照片背景** | 当天照片铺满背景，叠加情绪色渐变蒙层 | `DayHero`，CSS backdrop-filter |
| **日记文字呈现** | 在照片之上展示 AI 生成的日记全文 | `DiaryText` 组件 |
| **音乐播放栏** | 显示当前播放曲目信息（封面、曲名、歌手） | `SpotifyPlayer` 组件 |
| **情绪标签** | 展示 AI 分析的情绪标签 + 情绪色点 | `MoodDot`（`lg` 尺寸） |
| **页面进入动画** | 日历色点 → 详情页全屏背景的连续动画 | Framer Motion `layoutId` layout animation |

**音乐播放策略：**

| 用户类型 | 播放方式 |
| :--- | :--- |
| Spotify Premium 用户 | Spotify Web Playback SDK，全曲播放，品质最高 |
| Spotify 普通用户 | HTMLAudioElement 播放 30s previewUrl（创建记录时存储） |
| 未连接 Spotify | 页面正常展示，音乐功能隐藏，提示连接 Spotify |

---

### 3.4 用户与权限系统

| 模块 | 关键功能 |
| :--- | :--- |
| **注册/登录** | Google OAuth；JWT 鉴权（NextAuth.js） |
| **Spotify 连接** | 独立的 Spotify OAuth 流程，可随时连接/断开 |
| **用户档案** | 头像、昵称、Spotify 连接状态、Premium 状态 |
| **记录管理** | 查看历史记录、删除某天的记录 |
| **设置** | 通知开关、Spotify 连接管理 |

---

### 3.5 路由结构

| 路由 | 说明 | 权限 |
| :--- | :--- | :--- |
| `/login` | 登录页（Google OAuth） | 公开 |
| `/` | 主页：日历 + 时间轴 | 需要登录 |
| `/record` | 每日录制向导（5步） | 需要登录 |
| `/day/[date]` | 某天的日记详情页 | 需要登录，仅本人可访问 |
| `/settings` | 用户设置 | 需要登录 |

---

## 4. 技术架构

### 4.1 技术栈

| 模块 | 技术 | 作用 |
| :--- | :--- | :--- |
| **前端框架** | React 18 + TypeScript + Next.js 14 (App Router) | SSR/SSG、Server Components |
| **样式** | Tailwind CSS + shadcn/ui | 组件化设计系统 |
| **状态管理** | Redux Toolkit + RTK Query | 全局状态 + 服务端数据缓存 |
| **动画** | Framer Motion | 页面过渡、layout animation |
| **实时语音** | OpenAI Realtime API（WebRTC） | 双向语音对话 |
| **AI 日记/情绪** | Anthropic Claude API | 日记生成 + 情绪分析 |
| **音乐** | Spotify Web API + Web Playback SDK | 曲目推荐 + 浏览器内播放 |
| **数据库** | PostgreSQL（Supabase） | 关系型数据存储 |
| **ORM** | Prisma | 类型安全 DB 操作 |
| **认证** | NextAuth.js | Google OAuth + Spotify OAuth + JWT |
| **文件存储** | Supabase Storage | 照片存储 + CDN 分发 |
| **部署** | Vercel | CI/CD + Edge Network |

### 4.2 六大技术考察点实现

#### a. Hook 原理与边界

自定义 Hooks 封装副作用逻辑，与 UI 组件严格分离，每个 Hook 只做一件事：

| Hook | 职责 | 关键技术点 |
| :--- | :--- | :--- |
| `useRealtime` | WebRTC 会话全生命周期管理 | `useRef` 持有 RTCPeerConnection 避免 stale closure；`useEffect` cleanup 关闭连接 |
| `useSpotifyPlayer` | Spotify SDK 单例管理 | `useRef` 持有 Player 实例；仅 Premium 用户初始化 SDK |
| `useColorExtract` | 照片主色提取 | Canvas getImageData 客户端分析，无需服务端 |
| `useIntersection` | 时间轴底部到达检测 | IntersectionObserver，触发翻页加载 |

**边界原则：** Hooks 不直接操作 Redux store；数据流向为 Hook → 组件 → dispatch，不反向依赖。

#### b. 组件复用

| 共享组件 | 复用场景 | 变体控制 |
| :--- | :--- | :--- |
| `MoodDot` | 日历格子 / 时间轴卡片 / 详情页 | `size: 'sm' \| 'md' \| 'lg'`，`pulse` 动画 |
| `DiaryPreview` | 时间轴卡片（4行）/ 悬浮卡片（2行） | `maxLines` prop |
| `ColorPalette` | 录制向导选色步骤 / 设置页预览 | `readOnly` prop |

#### c. 状态管理分层

| 状态类型 | 存放位置 | 理由 |
| :--- | :--- | :--- |
| 用户信息、Spotify 状态 | `authSlice` (Redux) | 全局读取，登录态变化时全局响应 |
| 录制向导进度 + 数据 | `wizardSlice` (Redux) | 跨5个步骤组件传递，需要在步骤间持久 |
| 主页视图偏好 | `uiSlice` (Redux) | ViewToggle 写，Calendar/Timeline 读 |
| Spotify 播放状态 | `playerSlice` (Redux) | SpotifyPlayer 写，顶部迷你播放条读 |
| 服务端数据 | RTK Query | 自动缓存、标签失效、重验证 |
| WebRTC 连接 | Hook `useRef` | 副作用，不序列化，不放 Redux |
| 组件内动画状态 | `useState` | 不跨组件，无需全局 |

#### d. Redux Toolkit 核心实现

**`wizardSlice`：** 录制向导状态机，7个步骤状态（`PHOTO → VOICE → DIARY → COLOR → MUSIC → SAVING → COMPLETE`），`extraReducers` 监听 RTK Query mutation 结果自动推进状态。

**RTK Query `entriesApi`：**
- `getMonthEntries`：`providesTags` 实现月份级别缓存，切月无重复请求
- `saveEntry` mutation：`invalidatesTags` 保存后自动刷新相关缓存

#### e. 路由与权限

**双层守卫机制：**

1. **Edge Middleware（`middleware.ts`）：** 请求到达前验证 JWT token，未登录直接重定向 `/login`，无需到达 React 层。匹配规则：`/(app)/:path*`。
2. **Layout Server Component（`app/(app)/layout.tsx`）：** 读取完整 session，判断 Spotify 连接状态，为子页面注入 Provider tree（ReduxProvider → SpotifySDKProvider）。

`(auth)` 和 `(app)` 路由组通过 Next.js App Router 的 Group Layout 特性共享不同的 Provider tree，避免重复渲染。

#### f. 性能优化

| 优化场景 | 手段 | 收益 |
| :--- | :--- | :--- |
| 日历月份数据 | RTK Query 缓存，切月读缓存 | 减少重复网络请求 |
| 时间轴长列表 | `@tanstack/react-virtual` 虚拟滚动 | 700条记录仍保持 60fps |
| 照片加载 | Supabase Storage 变换 URL（`?width=200`）+ Next.js `<Image>` WebP | 缩略图体积降低 80% |
| Spotify SDK 按需加载 | `next/dynamic` + `{ ssr: false }` | 不影响首屏加载，非 Premium 用户零开销 |
| VoiceVisualizer | `next/dynamic` 懒加载 | Web Audio API 仅在 record 页加载 |
| 详情页数据 | Server Component 直接读数据库 | 减少客户端 JS，首屏更快 |
| 页面过渡 | Framer Motion `layoutId` layout animation | 色点 → 全屏背景的视觉连续性，无闪白 |

### 4.3 数据库设计

#### 核心数据表

**`users` 表**

| 字段 | 类型 | 说明 |
| :--- | :--- | :--- |
| `id` | String (cuid) | 主键 |
| `email` | String | 唯一，Google 登录用 |
| `googleId` | String? | Google OAuth ID |
| `spotifyId` | String? | Spotify OAuth ID |
| `spotifyAccessToken` | String? | AES-256 加密存储 |
| `spotifyRefreshToken` | String? | AES-256 加密存储 |
| `spotifyTokenExpiry` | DateTime? | token 过期时间 |
| `spotifyPremium` | Boolean | 是否 Premium，影响播放策略 |

**`diary_entries` 表**

| 字段 | 类型 | 说明 |
| :--- | :--- | :--- |
| `id` | String (cuid) | 主键 |
| `userId` | String (FK) | 关联用户 |
| `date` | Date | 日记日期，`@@unique([userId, date])` |
| `photoUrl` | String | Supabase Storage 路径 |
| `transcript` | Text? | 语音对话完整转录文本 |
| `diaryText` | Text? | Claude 生成的日记正文 |
| `emotionLabel` | String? | 情绪标签，e.g. "melancholic joy" |
| `emotionVector` | JSON? | `{ valence, energy, complexity }` |
| `moodColorHex` | String? | 情绪颜色，e.g. "#7EC8C8" |
| `colorSource` | Enum | `AI \| USER_MODIFIED` |
| `spotifyTrackId` | String? | Spotify 曲目 ID |
| `spotifyPreviewUrl` | String? | 30s 预览 URL（Free 用户 fallback） |
| `status` | Enum | `DRAFT \| COMPLETE`，支持中途恢复 |

#### 核心 API 接口

| 模块 | 接口 | 方法 | 描述 |
| :--- | :--- | :--- | :--- |
| 认证 | `/api/auth/[...nextauth]` | — | NextAuth 统一处理 |
| 日记 | `/api/entries` | GET | 获取月份记录列表（最小投影） |
| 日记 | `/api/entries` | POST | 创建 DRAFT 记录 |
| 日记 | `/api/entries/[id]` | GET/PATCH/DELETE | 详情/更新/删除 |
| 语音 | `/api/realtime/token` | POST | 生成 OpenAI WebRTC 临时 token |
| AI | `/api/ai/diary` | POST | transcript → Claude 日记生成 |
| AI | `/api/ai/emotion` | POST | 日记文本 → 情绪分析 + 颜色推荐 |
| 音乐 | `/api/spotify/search` | POST | 情绪向量 → Spotify 推荐曲目 |
| 音乐 | `/api/spotify/token` | POST | 刷新 Spotify access token |
| 存储 | `/api/photos/upload` | POST | 照片上传到 Supabase Storage |

### 4.4 关键技术难点

| 难点 | 解决方案 |
| :--- | :--- |
| **OpenAI Realtime API WebRTC 接入** | 服务端生成临时 token，前端创建 RTCPeerConnection，通过 DataChannel 接收 transcript delta 事件流 |
| **浏览器 Autoplay 策略** | 详情页入口遮罩的"进入这一天"点击本身是用户手势，在该 handler 内调用 `player.activateElement()` 或 `audio.play()`，合法绕过限制 |
| **Spotify Premium/Free 分支** | 登录时检测 `/v1/me` 的 `product` 字段，存入数据库。Premium 走 Web Playback SDK 全曲；Free 走存储的 `previewUrl` 30s 片段 |
| **Spotify Token 安全存储** | 使用 AES-256 对 access/refresh token 加密后入库，服务端解密使用，不落明文 |
| **照片主色提取** | 服务端用 `sharp` 提取主色调并映射到情绪色盘最近色，同时推荐给 AI 情绪分析步骤参考 |
| **向导中途恢复** | 照片上传完成即创建 DRAFT 记录；向导 mount 时检查当天是否有 DRAFT，有则从断点续接 |

---

## 5. 设计风格

### 5.1 视觉原则

- **情绪驱动配色：** 界面主色随当天情绪颜色动态变化，没有固定的品牌色，每一天有自己的颜色
- **照片优先：** 照片是最重要的内容元素，排版以照片为中心展开
- **克制的文字排版：** 日记文字使用舒适的行高和字间距，突出阅读体验而非界面设计
- **移动端优先：** 所有交互针对单手触控优化

### 5.2 交互原则

- **流畅的过渡动画：** 日历色点 → 详情页全屏背景，使用 Framer Motion layoutId 保持视觉连续性
- **最少操作步骤：** 录制向导每步只做一件事，减少认知负担
- **沉浸优先：** 详情页去掉所有导航栏，只有内容和退出按钮

### 5.3 情绪色盘（20色）

按情绪象限分组，供用户在步骤 4 中选择：

| 情绪区域 | 代表色（示例） |
| :--- | :--- |
| 平静 / 满足 | 薄荷绿、天空蓝、米白 |
| 轻盈 / 愉悦 | 柠檬黄、珊瑚粉、暖橙 |
| 低沉 / 疲惫 | 雾灰、烟蓝、深墨绿 |
| 激动 / 复杂 | 酒红、深紫、琥珀橙 |
| 温柔 / 怀念 | 藕粉、银灰紫、奶茶棕 |

---

## 6. 环境变量

```bash
# 数据库
DATABASE_URL=          # Supabase PostgreSQL（连接池）
DIRECT_URL=            # Prisma migrations（非池）

# Supabase Storage
NEXT_PUBLIC_SUPABASE_URL=
SUPABASE_SERVICE_ROLE_KEY=   # 服务端专用，不暴露客户端

# NextAuth
NEXTAUTH_URL=
NEXTAUTH_SECRET=

# Google OAuth
GOOGLE_CLIENT_ID=
GOOGLE_CLIENT_SECRET=

# Spotify OAuth + API
# 所需 scopes: user-read-email user-read-private
#              streaming user-read-playback-state user-modify-playback-state
SPOTIFY_CLIENT_ID=
SPOTIFY_CLIENT_SECRET=
SPOTIFY_REDIRECT_URI=

# AI
OPENAI_API_KEY=        # OpenAI Realtime API，服务端专用
ANTHROPIC_API_KEY=     # Claude API，服务端专用

# 安全
TOKEN_ENCRYPTION_KEY=  # 32字节 AES-256 key，用于加密 Spotify token 入库
```

---

## 7. 项目文件结构

```
src/
├── app/
│   ├── (auth)/
│   │   └── login/page.tsx              # Google OAuth 登录页
│   ├── (app)/                          # 需要认证的路由组
│   │   ├── layout.tsx                  # 认证 Guard + Redux Provider + Spotify SDK Provider
│   │   ├── page.tsx                    # 主页：日历/时间轴双视图
│   │   ├── record/page.tsx             # 录制向导（5步）
│   │   ├── day/[date]/page.tsx         # 日记详情页（自动播放音乐）
│   │   └── settings/page.tsx           # 用户设置
│   └── api/
│       ├── auth/[...nextauth]/route.ts
│       ├── entries/
│       │   ├── route.ts                # GET list / POST create
│       │   └── [id]/route.ts           # GET / PATCH / DELETE
│       ├── realtime/token/route.ts
│       ├── ai/
│       │   ├── diary/route.ts
│       │   └── emotion/route.ts
│       ├── spotify/
│       │   ├── search/route.ts
│       │   └── token/route.ts
│       └── photos/upload/route.ts
│
├── components/
│   ├── ui/                             # shadcn/ui 基础组件
│   ├── record/
│   │   ├── RecordWizard.tsx            # 向导容器（状态机）
│   │   ├── steps/
│   │   │   ├── PhotoStep.tsx
│   │   │   ├── VoiceStep.tsx
│   │   │   ├── DiaryStep.tsx
│   │   │   ├── ColorStep.tsx
│   │   │   └── MusicStep.tsx
│   │   ├── VoiceVisualizer.tsx         # 音频波形（懒加载）
│   │   └── ColorPalette.tsx            # 情绪色盘（可复用）
│   ├── main/
│   │   ├── ViewToggle.tsx
│   │   ├── calendar/
│   │   │   ├── CalendarView.tsx
│   │   │   ├── CalendarCell.tsx
│   │   │   └── PhotoHoverCard.tsx
│   │   └── timeline/
│   │       ├── TimelineView.tsx        # 虚拟滚动容器
│   │       └── TimelineCard.tsx
│   ├── day/
│   │   ├── DayHero.tsx                 # 全屏照片 + 情绪色背景
│   │   ├── DiaryText.tsx
│   │   └── SpotifyPlayer.tsx           # Premium/Free 播放策略
│   └── shared/
│       ├── MoodDot.tsx                 # 情绪色点（跨页复用）
│       ├── DiaryPreview.tsx            # 日记摘要（跨页复用）
│       ├── PageTransition.tsx
│       └── ErrorBoundary.tsx
│
├── store/
│   ├── index.ts                        # configureStore
│   ├── hooks.ts                        # useAppDispatch / useAppSelector
│   ├── slices/
│   │   ├── authSlice.ts
│   │   ├── wizardSlice.ts              # 录制向导状态机
│   │   ├── uiSlice.ts                  # 视图切换等 UI 状态
│   │   └── playerSlice.ts              # Spotify 播放状态
│   └── api/
│       ├── entriesApi.ts               # RTK Query
│       └── spotifyApi.ts
│
├── lib/
│   ├── hooks/
│   │   ├── useRealtime.ts
│   │   ├── useSpotifyPlayer.ts
│   │   ├── useColorExtract.ts
│   │   └── useIntersection.ts
│   ├── ai/
│   │   ├── claude.ts
│   │   └── prompts.ts
│   ├── spotify/
│   │   ├── client.ts
│   │   ├── playback.ts
│   │   └── emotion-map.ts
│   └── utils/
│       ├── date.ts
│       └── cn.ts
│
├── types/
│   ├── diary.ts
│   ├── spotify.ts
│   └── store.ts
│
└── middleware.ts                       # Edge Middleware 路由守卫
```
