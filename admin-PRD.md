# NewsHub 后台配置系统 PRD

## 1. 产品概述

**目标**：为 NewsHub 新增一个可视化后台管理界面，支持动态配置新闻分类和 RSS 来源，无需修改代码即可扩展内容覆盖范围。

**访问地址**：`/admin`，密码保护，仅管理员可用。

---

## 2. 功能需求

### 2.1 认证

| 功能 | 描述 |
|------|------|
| 密码登录 | 输入 `ADMIN_PASSWORD` 环境变量中的密码，验证后写入 Cookie，有效期 7 天 |
| 退出登录 | 清除 Cookie |

### 2.2 分类管理

| 功能 | 描述 |
|------|------|
| 查看分类列表 | 显示所有分类，含名称、来源数量、排序、启用状态 |
| 新增分类 | 输入分类名，设置排序 |
| 编辑分类 | 修改名称、排序 |
| 启用/禁用分类 | 禁用后前端 Tab 不展示该分类 |
| 删除分类 | 仅允许删除无来源的分类 |

### 2.3 来源管理

| 功能 | 描述 |
|------|------|
| 查看来源列表 | 按分类分组展示，含来源名、RSS URL、是否中文、启用状态、最近抓取文章数 |
| 新增来源 | 填写来源名、RSS URL、所属分类、是否中文 |
| 编辑来源 | 修改以上字段 |
| 启用/禁用来源 | 禁用后下次 Cron 不抓取该来源 |
| 删除来源 | 软删除（不影响已抓取的历史文章） |
| 测试 RSS | 点击「测试」按钮，实时请求该 URL，返回是否可访问及最新文章数 |

---

## 3. 页面设计

### 3.1 布局

```
/admin
├── 登录页（未认证时显示）
└── 管理后台（认证后）
    ├── 顶部导航：Logo + 退出登录
    ├── 左侧：分类列表（可点击切换）
    └── 右侧：当前分类下的来源列表
```

### 3.2 来源列表行

```
[来源名]  [RSS URL（截断）]  [中文/外文]  [文章数]  [状态]  [测试] [编辑] [删除]
```

---

## 4. 数据库变更

```prisma
model Category {
  id        String   @id @default(cuid())
  name      String   @unique
  order     Int      @default(0)
  enabled   Boolean  @default(true)
  sources   Source[]
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

## 5. API Routes

| 路由 | 方法 | 说明 |
|------|------|------|
| `/api/admin/login` | POST | 密码验证，返回 Cookie |
| `/api/admin/categories` | GET/POST | 查询/新增分类 |
| `/api/admin/categories/[id]` | PUT/DELETE | 修改/删除分类 |
| `/api/admin/sources` | GET/POST | 查询/新增来源 |
| `/api/admin/sources/[id]` | PUT/DELETE | 修改/删除来源 |
| `/api/admin/sources/test` | POST | 测试 RSS URL 可用性 |

---

## 6. 联动改造

| 模块 | 改造内容 |
|------|---------|
| `lib/rss.ts` | `RSS_SOURCES` 改为从数据库 `Source` 表读取（仅 enabled=true） |
| `CategoryTabs` | 分类列表改为从 `/api/categories` 动态获取 |
| 数据迁移 | 首次部署时将现有硬编码来源写入数据库 |

---

## 7. 安全

- 所有 `/api/admin/*` 接口校验 Cookie 中的 token，未认证返回 401
- `ADMIN_PASSWORD` 通过环境变量配置，不存储明文
- RSS URL 测试接口做超时限制（8s），防止 SSRF 滥用
