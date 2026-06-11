# NewsHub 技术实现方案

## 技术选型

- **Next.js 15** (App Router) — 前后端一体
- **Tailwind CSS** — 样式
- **PostgreSQL on Neon** — 数据库，免费套餐，支持 serverless 连接
- **Prisma** — ORM，类型安全，迁移方便
- **rss-parser** — 解析 RSS XML
- **DeepL Free API** — 翻译（免费额度每月 50 万字符）
- **Claude API** (`claude-sonnet-4-6`) — 每日精选评分 + 摘要生成
- **Vercel** — 部署 + Cron Jobs

---

## 架构

```
[Vercel Cron] 每天触发两次
     │
     ├─ 00:00 → /api/cron/fetch   抓取所有 RSS 源 → 翻译 → 存库
     └─ 02:00 → /api/cron/pick    从今日文章中让 LLM 选 3 篇 → 生成摘要 → 存库

[用户访问页面]
     └─ Next.js SSR 直接查数据库渲染 → 返回 HTML
```

---

## 项目目录结构

```
newshub/
├── app/
│   ├── page.tsx                  # 主页（精选 Banner + 文章列表）
│   ├── layout.tsx
│   └── api/
│       ├── cron/
│       │   ├── fetch/route.ts    # Cron: 抓取 RSS + 翻译
│       │   └── pick/route.ts     # Cron: 每日精选 + AI 摘要
│       └── articles/route.ts     # 前端分页拉文章
├── components/
│   ├── DailyPicks.tsx
│   ├── CategoryTabs.tsx
│   ├── ArticleList.tsx
│   └── Header.tsx
├── lib/
│   ├── db.ts                     # Prisma client 单例
│   ├── rss.ts                    # RSS 抓取逻辑
│   ├── translate.ts              # DeepL 翻译
│   └── ai.ts                     # Claude 精选 + 摘要
├── prisma/
│   └── schema.prisma
├── .env.local
└── vercel.json                   # Cron 配置
```

---

## 数据库 Schema

```prisma
model Article {
  id          String   @id @default(cuid())
  url         String   @unique          // 去重键
  title       String                    // 已翻译为中文
  description String                    // 已翻译为中文
  source      String                    // "Reuters World"
  category    String                    // "世界"
  publishedAt DateTime
  fetchedAt   DateTime @default(now())  // 入库时间，用于精选范围筛选
  isPickedAt  DateTime?                 // 非空表示哪天被选为精选
  aiSummary   String?                   // 仅精选文章有值
}
```

清理逻辑：每次抓取时 `DELETE WHERE fetchedAt < NOW() - 7 days`

---

## 路由设计

| 路由 | 方法 | 说明 |
|------|------|------|
| `/` | GET | 主页，SSR，展示精选 + 第一页文章 |
| `/api/articles` | GET | 分页查询，参数：`category`, `page`, `pageSize=20` |
| `/api/cron/fetch` | GET | Cron：抓取 + 翻译 + 入库 |
| `/api/cron/pick` | GET | Cron：精选评分 + 摘要生成 |

---

## 实现步骤

1. **搭骨架** — 初始化 Next.js，接 Neon，跑通 Prisma migration
2. **RSS 抓取** — `/api/cron/fetch` 拉 13 个源，URL 去重，非中文来源调 DeepL 翻译，批量写库
3. **每日精选** — `/api/cron/pick` 查今日文章，发给 Claude 选 3 篇，再对 3 篇生成中文摘要，更新 `isPickedAt` + `aiSummary`
4. **页面渲染** — 主页 SSR 查精选 + 第一页，切 Tab 后客户端请求 `/api/articles`
5. **数据清理** — 在 fetch Cron 开头删除 7 天前数据

---

## 部署方案

```
1. 代码推 GitHub
2. Vercel 连接仓库，自动部署
3. 环境变量（Vercel Dashboard）：
   DATABASE_URL      = neon 连接串
   DEEPL_API_KEY     = xxx
   ANTHROPIC_API_KEY = xxx
   CRON_SECRET       = 随机字符串（防止 Cron 接口被外部调用）
4. vercel.json 配置 Cron：
   00:00 UTC → /api/cron/fetch
   18:00 UTC → /api/cron/pick   （对应北京时间 02:00）
```

**MVP 阶段成本：零**
- Vercel 免费套餐：支持 2 个 Cron Jobs
- Neon 免费套餐：0.5GB（7天数据约 4MB）
- DeepL 免费：50 万字符/月
- Claude API：每日仅调用一次精选，成本可忽略不计
