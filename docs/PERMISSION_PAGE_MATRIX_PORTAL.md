# Permission & Page Coverage Matrix — Nuxt Portal

> 本文档记录 Nuxt Portal (nuxt-portal) 用户端页面与后端 API 的权限映射关系。
>
> 创建日期：2026-01-05
> ROUNDREQ P0-4

## 1. Portal 页面清单

| 页面路径 | 组件文件 | 布局 | 认证 | 说明 |
|----------|----------|------|------|------|
| `/portal` | `pages/portal/index.vue` | portal | 是 | Dashboard 首页 |
| `/portal/profile` | `pages/portal/profile.vue` | portal | 是 | 用户资料 + 身份绑定 |
| `/portal/invite-codes` | `pages/portal/invite-codes.vue` | portal | 是 | 邀请码管理 |
| `/portal/affiliate` | `pages/portal/affiliate.vue` | portal | 是 | 推广分佣 |
| `/portal/orders` | `pages/portal/orders.vue` | portal | 是 | 订单列表 |
| `/portal/tickets` | `pages/portal/tickets.vue` | portal | 是 | 工单列表 |
| `/portal/after-sales` | `pages/portal/after-sales.vue` | portal | 是 | 售后管理 |

## 2. 页面 → API 映射

### 2.1 Dashboard (`/portal`)

| API Endpoint | 方法 | 认证要求 | 权限要求 | 说明 |
|--------------|------|----------|----------|------|
| `/api/v1/user/me` | GET | Bearer JWT | 无 | 获取用户基本信息 |
| `/api/v1/menu/all` | GET | Bearer JWT | 无 | 获取菜单和权限列表 |

### 2.2 Profile (`/portal/profile`)

| API Endpoint | 方法 | 认证要求 | 权限要求 | 说明 |
|--------------|------|----------|----------|------|
| `/api/v1/user/me` | GET | Bearer JWT | 无 | 获取用户详情 |
| `/api/v1/user/bind/status` | GET | Bearer JWT | 无 | 获取绑定状态 |
| `/api/v1/user/bind/telegram-config` | GET | 无 | 无 | 获取 Telegram Widget 配置 |
| `/api/v1/user/bind/telegram` | POST | Bearer JWT | 无 | 绑定 Telegram (Widget) |
| `/api/v1/user/bind/email` | POST | Bearer JWT | 无 | 绑定 Email |

### 2.3 Invite Codes (`/portal/invite-codes`)

| API Endpoint | 方法 | 认证要求 | 权限要求 | 说明 |
|--------------|------|----------|----------|------|
| `/api/v1/user/me/invite-codes` | GET | Bearer JWT | 无 | 获取邀请码列表 |
| `/api/v1/user/invite-settings` | GET | Bearer JWT | 无 | 获取邀请码设置 |
| `/api/v1/user/me/invite-code` | POST | Bearer JWT | 无 | 创建邀请码 (扣除 Credits) |

### 2.4 Affiliate (`/portal/affiliate`)

| API Endpoint | 方法 | 认证要求 | 权限要求 | 说明 |
|--------------|------|----------|----------|------|
| `/api/v1/affiliate/my-link` | GET | Bearer JWT | 无 | 获取推广链接 |
| `/api/v1/affiliate/my-stats` | GET | Bearer JWT | 无 | 获取推广统计 |
| `/api/v1/affiliate/my-commissions` | GET | Bearer JWT | 无 | 获取佣金列表 |

### 2.5 Orders (`/portal/orders`)

| API Endpoint | 方法 | 认证要求 | 权限要求 | 说明 |
|--------------|------|----------|----------|------|
| `/api/v1/user/credits/orders` | GET | Bearer JWT | 无 | 获取订单列表 |

### 2.6 Tickets (`/portal/tickets`)

| API Endpoint | 方法 | 认证要求 | 权限要求 | 说明 |
|--------------|------|----------|----------|------|
| `/api/v1/tickets/` | GET | Bearer JWT | 无 | 获取工单列表 |
| `/api/v1/tickets/` | POST | Bearer JWT | 无 | 创建工单 |
| `/api/v1/tickets/{id}` | GET | Bearer JWT | 无 | 获取工单详情 |
| `/api/v1/tickets/{id}/reply` | POST | Bearer JWT | 无 | 回复工单 |

### 2.7 After Sales (`/portal/after-sales`)

| API Endpoint | 方法 | 认证要求 | 权限要求 | 说明 |
|--------------|------|----------|----------|------|
| `/api/v1/after-sales/` | GET | Bearer JWT | 无 | 获取售后列表 |
| `/api/v1/after-sales/` | POST | Bearer JWT | 无 | 创建售后申请 |
| `/api/v1/after-sales/{id}` | GET | Bearer JWT | 无 | 获取售后详情 |

## 3. 权限模型说明

### 3.1 用户端权限

Portal 用户端页面采用 **身份认证** 模式：
- 所有页面需要有效的 JWT Token
- 不需要额外的权限键 (Permission Key)
- 用户只能访问自己的数据（通过 `get_current_user` 依赖注入）

### 3.2 403 错误处理

每个页面在 API 调用时检查 403 状态码：
- 若返回 403，显示 "Permission denied (403)" 错误提示
- 不自动跳转，保持在当前页面

### 3.3 菜单权限

Portal 侧边栏菜单基于后端 `/menu/all` 返回的 `menus` 数组动态渲染：
- 只显示用户有权访问的菜单项
- `UserCenter` 子菜单映射到 `/portal/*` 路由

## 4. 验证清单

| 验证项 | 状态 | 证据文件 |
|--------|------|----------|
| Dashboard 加载成功 | PASS | logs/ac_portal_dashboard.png |
| Profile 绑定状态显示 | PASS | logs/ac_round3_email_binding_ui.png |
| Invite Codes 列表加载 | PASS | logs/ac_portal_invite_codes.png |
| Affiliate 统计显示 | PASS | logs/ac_portal_affiliate.png (empty state - expected) |
| Orders 列表加载 | PASS | logs/ac_portal_orders.png |
| Tickets 列表加载 | PASS | logs/ac_portal_tickets.png (empty state - expected) |
| After Sales 列表加载 | PASS | logs/ac_portal_after_sales.png |
| 403 错误正确显示 | PASS | 各页面已实现 403 错误处理 |

## 5. 后续改进

1. **权限键绑定**：若需要更细粒度的权限控制，可为每个 API 添加 `require_permission("xxx")` 装饰器
2. **菜单隐藏**：无权限页面应从侧边栏隐藏（当前已通过 `/menu/all` 实现）
3. **批量操作权限**：如批量删除、批量导出等操作可添加独立权限检查
