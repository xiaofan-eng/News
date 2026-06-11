# PRD：全球新闻聚合器 NewsHub

## 1. 产品概述

**项目名称**：NewsHub

**目标**：构建一个基于 RSS 的多领域新闻聚合 Web 应用，自动抓取、分类、展示来自全球可靠来源的最新资讯。

**核心价值**：
- 一站式覆盖 AI、科技、经济、政治、社会五大领域
- 中英双语来源并重，外文内容自动翻译为中文
- 每日 AI 筛选 3 篇最重要文章，并生成中文摘要
- 无需注册，开箱即用，信息密度高、噪音低

---

## 2. 目标用户

| 用户类型 | 描述 |
|----------|------|
| 科技从业者 | 需要跟踪 AI 动态、行业资讯、开发者社区热点 |
| 投资者 / 分析师 | 关注经济、金融、政策变化 |
| 学生 / 研究者 | 需要获取国际政治、社会议题的权威来源 |
| 信息密集型用户 | 厌倦算法推荐，希望自主掌控信息源 |

---

## 3. 功能需求

### 3.1 RSS 数据源

MVP 内置以下 12 个来源，按分类管理：

```json
[
  { "name": "Reuters World",   "url": "https://feeds.reuters.com/reuters/worldNews",               "category": "世界" },
  { "name": "BBC World",       "url": "https://feeds.bbci.co.uk/news/world/rss.xml",               "category": "世界" },
  { "name": "AP News",         "url": "https://rsshub.app/apnews/topics/world-news",               "category": "世界" },
  { "name": "Financial Times", "url": "https://www.ft.com/rss/home",                               "category": "财经" },
  { "name": "财新网",           "url": "https://www.caixin.com/rss/home.xml",                       "category": "财经" },
  { "name": "OpenAI Blog",     "url": "https://openai.com/blog/rss/",                              "category": "AI"  },
  { "name": "Anthropic Blog",  "url": "https://www.anthropic.com/rss.xml",                         "category": "AI"  },
  { "name": "Hugging Face",    "url": "https://huggingface.co/blog/feed.xml",                      "category": "AI"  },
  { "name": "The Batch",       "url": "https://www.deeplearning.ai/the-batch/feed/",               "category": "AI"  },
  { "name": "TechCrunch",      "url": "https://techcrunch.com/feed/",                              "category": "科技" },
  { "name": "Hacker News",     "url": "https://news.ycombinator.com/rss",                          "category": "科技" },
  { "name": "NYT",             "url": "https://rss.nytimes.com/services/xml/rss/nyt/HomePage.xml", "category": "社会" },
  { "name": "澎湃新闻",         "url": "https://www.thepaper.cn/rss_1621",                          "category": "社会" }
]
```

### 3.2 功能描述

| 功能 | 描述 | 优先级 |
|------|------|--------|
| RSS 抓取 | 每 24 小时增量拉取所有源，按 URL 去重后追加存入数据库，保留 7 天历史文章，自动清理过期数据 | P0 |
| 分类展示 | 按 世界 / AI / 科技 / 财经 / 社会 分 Tab 展示文章列表，按时间倒序 | P0 |
| 文章卡片 | 显示标题、来源、发布时间、中文摘要；点击跳转原文（新标签页） | P0 |
| 去重 | 按 URL 去重，同一文章不重复入库 | P0 |
| 全部 Tab | 默认 Tab，跨分类按时间倒序展示所有文章 | P0 |
| 外文自动翻译 | 调用翻译 API 将非中文来源的 title 和 description 翻译为中文；中文来源（财新、澎湃）跳过翻译 | P0 |
| 每日精选 | 每天凌晨 2:00 从当天日期入库的全量文章中不限分类筛选 3 篇最重要文章，横向并排展示于顶部 Banner；当日精选全天固定不变 | P0 |
| 精选 AI 摘要 | 仅对每日精选 3 篇调用 LLM 生成高质量中文摘要 | P0 |
| 搜索 | 标题关键词搜索 | P1 |
| 来源筛选 | 按具体来源过滤文章 | P1 |
| 健康检查 | 监控各 RSS 源可用性，失败时标记并告警 | P1 |

### 3.3 页面组件

#### 主页面布局

```
┌──────────────────────────────────────────────┐
│  Logo    [搜索框]                   [设置]    │  ← Header
├──────────────────────────────────────────────┤
│  今日精选（AI 推荐，不限分类）                │  ← DailyPicks Banner
│  ┌───────────┐  ┌───────────┐  ┌───────────┐ │
│  │[精选]标题  │  │[精选]标题  │  │[精选]标题  │ │  ← 横向并排 3 篇
│  │ 来源·时间  │  │ 来源·时间  │  │ 来源·时间  │ │
│  │ AI摘要... │  │ AI摘要... │  │ AI摘要... │ │
│  └───────────┘  └───────────┘  └───────────┘ │
├──────────────────────────────────────────────┤
│  全部 | 世界 | AI | 科技 | 财经 | 社会        │  ← Category Tabs
├──────────────────────────────────────────────┤
│  ┌────────────────────────────────────────┐   │
│  │ 标题(中文)            来源 · 时间      │   │  ← Article List
│  │ 中文摘要...                            │   │
│  ├────────────────────────────────────────┤   │
│  │ 标题(中文)            来源 · 时间      │   │
│  │ 中文摘要...                            │   │
│  ├────────────────────────────────────────┤   │
│  │ 标题(中文)            来源 · 时间      │   │
│  │ 中文摘要...                            │   │
│  └────────────────────────────────────────┘   │
│  [加载更多]                                   │
└──────────────────────────────────────────────┘
```

#### 组件清单

| 组件 | 说明 |
|------|------|
| `Header` | Logo + 搜索框 |
| `DailyPicks` | 置顶区域，展示当日 AI 筛选的 3 篇精选文章，带「精选」标签 |
| `CategoryTabs` | 分类切换 Tab，支持横向滚动（移动端） |
| `ArticleCard` | 标题（中文，点击跳转原文新标签页）、来源标签、时间、中文摘要 |
| `ArticleList` | 竖排列表，每行一篇：标题（左）+ 来源·时间（右）+ 摘要（次行），分隔线间隔 |
| `LoadMore` | 分页加载按钮（或无限滚动） |

---

## 4. 非功能性需求

### 响应式设计

| 断点 | 布局 |
|------|------|
| 所有断点 | 竖排列表，单列全宽 |

### 性能
- 首屏加载 < 2s（静态资源 CDN + 数据预渲染）
- RSS 抓取、翻译、AI 摘要生成均为异步后台执行，不阻塞前端请求
- 每日精选筛选任务在每天凌晨 2:00 执行，避开访问高峰

### AI 能力说明

| 能力 | 实现方式 | 触发时机 |
|------|----------|----------|
| 普通文章翻译 | 调用翻译 API（DeepL / Google Translate / 百度翻译）将 description 翻译为中文 | RSS 抓取后立即处理 |
| 每日精选 | 调用 LLM 对当日全量文章标题+摘要评分，取前 3 篇 | 每天凌晨 2:00 定时执行 |
| 精选 AI 摘要 | 调用 LLM 对精选 3 篇生成高质量中文摘要 | 每天凌晨 2:00 精选后执行 |

**每日精选评分维度（LLM Prompt 指引）**：
- 影响力：事件的全球/行业影响范围
- 时效性：是否为当日重大突发或首发
- 多领域关联度：跨经济、政治、科技等领域的综合重要性

### 可用性
- RSS 源失败不影响其他源正常展示
- 无文章时显示空状态提示

### 技术栈建议
- **前端**：Next.js + Tailwind CSS
- **后端**：Next.js API Routes
- **数据库**：PostgreSQL（线上部署，推荐 Supabase 或 Neon 免费套餐）
- **定时任务**：Vercel Cron Jobs（每 24h 抓取；每天 2:00 精选）
- **RSS 解析**：`rss-parser` (Node.js)
- **翻译**：DeepL API 或 Google Translate API
- **AI 摘要 / 精选**：Claude API (`claude-sonnet-4-6`) 或 OpenAI API
- **部署**：Vercel

---



