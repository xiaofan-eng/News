# NewsHub 后台配置系统 — Claude Code 指引

## 项目概述

为 NewsHub（newshub-blush.vercel.app）新增后台管理系统，支持动态配置新闻分类和 RSS 来源，无需修改代码即可扩展内容。

详细需求见 `admin-PRD.md`，技术方案见 `admin-plan.md`。

**代码目录**：`/Users/fujiaxing/myproject/newshub/`

---

## 技术栈

- **框架**：Next.js 15 (App Router) + Tailwind CSS（复用现有项目）
- **数据库**：PostgreSQL (Neon) + Prisma ORM
- **认证**：Cookie + 签名 token（自行实现，不引入 NextAuth）
- **部署**：Vercel（新增 `ADMIN_PASSWORD` 环境变量）

---

## 新增目录结构

```
app/
  admin/
    layout.tsx            # 认证检查（读 Cookie）
    page.tsx              # 后台主页
    login/
      page.tsx            # 登录页
  api/
    admin/
      login/route.ts      # POST：密码验证，写 Cookie
      logout/route.ts     # POST：清除 Cookie
      categories/
        route.ts          # GET 列表 / POST 新增
        [id]/route.ts     # PUT 修改 / DELETE 删除
      sources/
        route.ts          # GET 列表 / POST 新增
        [id]/route.ts     # PUT 修改 / DELETE 删除
        test/route.ts     # POST：测试 RSS URL 可用性
    categories/route.ts   # 前端公开读取分类列表
components/
  admin/
    CategoryList.tsx      # 左侧分类列表
    SourceTable.tsx       # 右侧来源表格
    SourceForm.tsx        # 新增/编辑来源弹窗
```

---

## 数据库 Schema 变更

在现有 `prisma/schema.prisma` 中新增：

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
```

---

## API 路由

| 路由 | 方法 | 权限 | 说明 |
|------|------|------|------|
| `/admin` | GET | 管理员 | 后台主页 |
| `/admin/login` | GET | 公开 | 登录页 |
| `/api/admin/login` | POST | 公开 | 密码验证 |
| `/api/admin/logout` | POST | 管理员 | 退出登录 |
| `/api/admin/categories` | GET/POST | 管理员 | 分类列表/新增 |
| `/api/admin/categories/[id]` | PUT/DELETE | 管理员 | 修改/删除分类 |
| `/api/admin/sources` | GET/POST | 管理员 | 来源列表/新增 |
| `/api/admin/sources/[id]` | PUT/DELETE | 管理员 | 修改/删除来源 |
| `/api/admin/sources/test` | POST | 管理员 | 测试 RSS URL |
| `/api/categories` | GET | 公开 | 前端读取启用的分类 |

---

## 实现步骤

1. **数据库迁移** — 新增 Category、Source 表，执行 `prisma db push`，写一次性迁移脚本将现有 21 个硬编码来源导入数据库

2. **认证模块** — 实现 `/admin/login` 页面和 `app/admin/layout.tsx` 中间件，密码对比 `process.env.ADMIN_PASSWORD`，Cookie 有效期 7 天

3. **管理 API** — 实现所有 `/api/admin/*` 接口，test 接口用 `rss-parser` 验证 URL，超时 8s

4. **管理 UI** — 左右分栏布局，左侧分类列表，右侧来源表格（含状态开关、测试按钮、编辑/删除）

5. **改造 fetch 路由** — `lib/rss.ts` 的 `RSS_SOURCES` 改为从数据库查询（`enabled=true` 的 Source）

6. **改造 CategoryTabs** — 从硬编码数组改为调用 `/api/categories` 动态获取

---

## 环境变量

现有：
```
DATABASE_URL
DEEPSEEK_API_KEY
CRON_SECRET
```

新增：
```
ADMIN_PASSWORD=你设置的后台密码
ADMIN_TOKEN_SECRET=随机字符串（用于签名 Cookie）
```

---

## 编码规范

- 所有 `/api/admin/*` 接口在入口处验证 Cookie token，非法返回 401
- 认证 token = `sha256(ADMIN_TOKEN_SECRET + timestamp)`，有效期 7 天
- RSS test 接口设 8s 超时，失败返回 `{ ok: false, error: 'timeout' }`
- Category 删除前检查是否有关联 Source，有则拒绝并提示
- 前端 CategoryTabs 动态化后，分类顺序由 `Category.order` 字段控制
