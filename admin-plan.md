# NewsHub 后台配置系统 — 技术实现方案

## 技术选型

复用现有技术栈，无需新增依赖：
- **认证**：自己实现简单 Cookie 验证（不引入 NextAuth）
- **UI**：Tailwind CSS，沿用现有暖色调设计变量
- **数据库**：现有 PostgreSQL 新增两张表（Category、Source）

---

## 架构

```
用户访问 /admin
    ↓
中间件检查 Cookie 中的 admin_token
    ↓
有效 → 显示管理界面
无效 → 跳转登录页
    ↓
管理员操作（增删改来源/分类）
    → 写入 Category / Source 表
    ↓
次日 Cron 抓取时从数据库读取来源（而非硬编码）
    ↓
前端 CategoryTabs 从 /api/categories 动态渲染
```

---

## 项目目录结构

```
app/
  admin/
    page.tsx              # 后台主页（需认证）
    login/page.tsx        # 登录页
    layout.tsx            # 认证检查
  api/
    admin/
      login/route.ts      # POST 登录
      logout/route.ts     # POST 退出
      categories/
        route.ts          # GET/POST
        [id]/route.ts     # PUT/DELETE
      sources/
        route.ts          # GET/POST
        [id]/route.ts     # PUT/DELETE
        test/route.ts     # POST 测试 RSS
    categories/route.ts   # 前端读取分类（公开）
components/
  admin/
    CategoryList.tsx      # 左侧分类列表
    SourceTable.tsx       # 右侧来源表格
    SourceForm.tsx        # 新增/编辑来源弹窗
```

---

## 数据库 Schema

```prisma
model Category {
  id      String   @id @default(cuid())
  name    String   @unique
  order   Int      @default(0)
  enabled Boolean  @default(true)
  sources Source[]
}

model Source {
  id         String   @id @default(cuid())
  name       String
  url        String   @unique
  categoryId String
  category   Category @relation(fields: [categoryId], references: [id])
  isChinese  Boolean  @default(false)
  enabled    Boolean  @default(true)
  createdAt  DateTime @default(now())
}

// Article 表新增外键（可选，向后兼容）
// sourceId String?
```

---

## 路由设计

| 路由 | 权限 | 说明 |
|------|------|------|
| `/admin` | 管理员 | 后台主页 |
| `/admin/login` | 公开 | 登录页 |
| `/api/admin/*` | 管理员（Cookie） | 所有管理接口 |
| `/api/categories` | 公开 | 前端读取分类列表 |

---

## 实现步骤

1. **数据库迁移** — 新增 Category、Source 表，`prisma db push`，执行一次性脚本把现有 21 个来源写入数据库

2. **认证** — 登录页 + 中间件，密码对比 `ADMIN_PASSWORD` 环境变量，验证后写签名 Cookie

3. **管理 API** — categories 和 sources 的 CRUD 接口；test 接口用 `rss-parser` 实际请求 URL 验证可用性

4. **管理 UI** — 左侧分类列表 + 右侧来源表格，行内有启用开关、测试、编辑、删除；表单用弹窗实现

5. **改造 fetch 路由** — `lib/rss.ts` 的 `RSS_SOURCES` 改为从数据库查询 `Source`（`enabled=true`）

6. **改造前端 CategoryTabs** — 从硬编码数组改为调用 `/api/categories` 动态渲染

---

## 部署方案

新增环境变量：
```
ADMIN_PASSWORD=你设置的后台密码
```

推送 GitHub → Vercel 自动部署 → 运行 `prisma db push` 同步新表结构

**工作量估计**：约 300-400 行代码，1-2 天可完成。
