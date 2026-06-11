# NewsHub — Claude Code 指引

## 项目概述

基于 RSS 的全球新闻聚合器，自动抓取、翻译、分类展示新闻，每日 AI 精选 3 篇重要文章。

详细需求见 `PRD.md`，技术方案见 `plan.md`。

## 技术栈

- **框架**：Next.js 15 (App Router)
- **样式**：Tailwind CSS
- **数据库**：PostgreSQL (Neon) + Prisma ORM
- **部署**：Vercel + Vercel Cron Jobs
- **翻译**：DeepL API
- **AI**：Claude API (`claude-sonnet-4-6`)
- **RSS 解析**：rss-parser

## 项目结构

```
app/
  page.tsx                  # 主页，SSR
  layout.tsx
  api/
    cron/
      fetch/route.ts        # 每日抓取 RSS + 翻译
      pick/route.ts         # 每日精选 + AI 摘要
    articles/route.ts       # 分页查询接口
components/
  Header.tsx
  DailyPicks.tsx            # 精选 Banner，横排 3 篇
  CategoryTabs.tsx
  ArticleList.tsx           # 竖排文章列表
lib/
  db.ts                     # Prisma client 单例
  rss.ts                    # RSS 抓取
  translate.ts              # DeepL 翻译
  ai.ts                     # Claude 精选评分 + 摘要
prisma/
  schema.prisma
vercel.json                 # Cron 配置
```

## 数据库 Schema

```prisma
model Article {
  id          String    @id @default(cuid())
  url         String    @unique
  title       String
  description String
  source      String
  category    String
  publishedAt DateTime
  fetchedAt   DateTime  @default(now())
  isPickedAt  DateTime?
  aiSummary   String?
}
```

## RSS 数据源

| 来源 | 分类 | 是否需要翻译 |
|------|------|-------------|
| Reuters World | 世界 | 是 |
| BBC World | 世界 | 是 |
| AP News | 世界 | 是 |
| Financial Times | 财经 | 是 |
| 财新网 | 财经 | 否 |
| OpenAI Blog | AI | 是 |
| Anthropic Blog | AI | 是 |
| Hugging Face | AI | 是 |
| The Batch | AI | 是 |
| TechCrunch | 科技 | 是 |
| Hacker News | 科技 | 是 |
| NYT | 社会 | 是 |
| 澎湃新闻 | 社会 | 否 |

## 核心业务规则

- **抓取频率**：每 24 小时一次（Vercel Cron：`0 0 * * *` UTC）
- **去重**：按 `url` 字段唯一约束
- **翻译**：仅对非中文来源（财新、澎湃跳过）翻译 `title` + `description`
- **数据保留**：7 天，每次抓取时清理 `fetchedAt < NOW() - 7 days` 的数据
- **每日精选**：每天凌晨 2:00 北京时间（`0 18 * * *` UTC）执行，从当天 `fetchedAt` 的文章中不限分类选 3 篇，全天固定不变
- **精选评分维度**：影响力、时效性、多领域关联度
- **AI 摘要**：仅对精选 3 篇生成，普通文章直接展示翻译后的 description

## API 路由

| 路由 | 说明 |
|------|------|
| `GET /` | 主页，SSR，精选 + 第一页文章 |
| `GET /api/articles?category=&page=&pageSize=20` | 分页查询 |
| `GET /api/cron/fetch` | Cron 调用，需校验 `CRON_SECRET` |
| `GET /api/cron/pick` | Cron 调用，需校验 `CRON_SECRET` |

## 环境变量

```
DATABASE_URL        # Neon PostgreSQL 连接串
DEEPL_API_KEY       # DeepL Free API Key
ANTHROPIC_API_KEY   # Claude API Key
CRON_SECRET         # 随机字符串，保护 Cron 接口
```

## vercel.json

```json
{
  "crons": [
    { "path": "/api/cron/fetch", "schedule": "0 0 * * *" },
    { "path": "/api/cron/pick",  "schedule": "0 18 * * *" }
  ]
}
```

## 编码规范

- 所有组件使用 TypeScript
- 数据库操作统一通过 `lib/db.ts` 的 Prisma client
- Cron 接口校验 `Authorization: Bearer ${CRON_SECRET}` header，非法请求返回 401
- 翻译和 AI 调用失败不应阻断整体流程，捕获异常后跳过该条记录并记录日志
