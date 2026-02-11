# 更新日志

## [1.2.0] - 2026-02-10

### 国际化 (I18N)

> 1.2.0 版本实现全栈 4 语言 (en / zh-CN / zh-TW / ja) 国际化，
> 覆盖 Portal、MiniApp、Vben Admin、Backend API 全链路。

#### 架构关系图

```
┌─────────────────────────────────────────────────────────────────────┐
│                    I18N 架构总览 (v1.2.0)                            │
│                    语言: en / zh-CN / zh-TW / ja                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────────┐   ┌──────────────────┐   ┌─────────────────┐  │
│  │  Portal (Nuxt)  │   │ MiniApp (Nuxt)   │   │  Vben Admin     │  │
│  │  @nuxtjs/i18n   │   │ @nuxtjs/i18n     │   │  vue-i18n       │  │
│  ├─────────────────┤   ├──────────────────┤   ├─────────────────┤  │
│  │ I18N-01: 外壳层 │   │ I18N-01: 外壳层  │   │ I18N-02: 基建   │  │
│  │  - 顶栏/页脚    │   │  - 顶栏          │   │  - 4 语言类型   │  │
│  │  - 登录/注册    │   │  - 菜单侧边栏    │   │  - Antd/dayjs   │  │
│  │  - 设置/首页    │   │  - 仪表盘        │   │  - 路由标题     │  │
│  │  - 3 个布局     │   │  - 签到/错误     │   │  - 认证错误     │  │
│  │  - 语言切换器   │   │  - 认证表单      │   │  - 通知数据     │  │
│  │                 │   │  - 语言切换器    │   │                 │  │
│  ├─────────────────┤   ├──────────────────┤   │ 142 page.json   │  │
│  │ I18N-04: 全页面 │   │ I18N-04: 全页面  │   │ keys × 4 locale │  │
│  │  23 个 Portal 页│   │ 19 个业务区域    │   └─────────────────┘  │
│  │  1422 keys × 4  │   │ miniapp.vue 内   │                        │
│  └────────┬────────┘   └────────┬─────────┘                        │
│           │                     │                                   │
│           └──────────┬──────────┘                                   │
│                      │                                              │
│              ┌───────▼────────┐                                     │
│              │  Locale 文件   │                                     │
│              │  (Nuxt i18n)   │                                     │
│              ├────────────────┤                                     │
│              │ en.json        │  每份 1422 keys                     │
│              │ zh-CN.json     │  合计 5688 条                       │
│              │ zh-TW.json     │                                     │
│              │ ja.json        │                                     │
│              └───────┬────────┘                                     │
│                      │ ?lang= / localStorage / Accept-Language      │
│              ┌───────▼────────┐                                     │
│              │  后端 API      │                                     │
│              │  (FastAPI)     │                                     │
│              ├────────────────┤                                     │
│              │ I18N-03: 基建  │                                     │
│              │  - t() 函数   │  192 keys × 4 locales               │
│              │  - 中间件     │  28 个 message_key                   │
│              │  - menu.py    │  129 个 page.* keys                  │
│              │  - 异常体系   │  全部支持 message_key                │
│              └────────────────┘                                     │
│                                                                     │
│  语言优先级: ?lang= > localStorage > Accept-Language > zh-CN        │
│  localStorage key: uhd_lang                                         │
│  i18n 框架: @nuxtjs/i18n (Nuxt) / vue-i18n (Vben) / 自定义        │
│            Accept-Language (后端)                                    │
└─────────────────────────────────────────────────────────────────────┘
```

#### Round I18N-04: Portal + MiniApp 全量页面 key 化
- 23 个 Portal 页面 `.vue` 文件全量 key 化：所有硬编码中英文 UI 字符串替换为 `$t()` 调用
- miniapp.vue 业务页面区域 key 化（19 个区域）：profile, wallet, media-accounts, orders, tickets, after-sales, shop, checkin, benefits, display-prefs, affiliate, cdkeys, seats, renewal-cards, assets, invite-codes, invite-tree, subscriptions, watch-history
- 1422 个 key × 4 语言 = 5688 条翻译条目，全语言一致
- 新增 `portal.*` 命名空间（75+ 通用 key + 23 个页面专属区域）
- 新增 `miniapp.*` 业务区域（9 个新子区域 + 通用 key）
- `pnpm build` 全部通过

#### Round I18N-03: Backend API 多语言基础设施
- 新建 `app/i18n/` 模块：`t()` 翻译函数 + 4 语言 JSON locale (192 keys each)
- Accept-Language 中间件：解析请求头注入 locale，支持 `?lang=` 覆盖
- `success_response()` / `error_response()` 扩展 `message_key` 可选字段
- 异常体系 (`exceptions.py`) 全部支持 `message_key` 参数
- `menu.py` 全量 key 化：129 个 `page.*` keys，菜单标题由 Vben 前端 `$t()` 解析
- 7 个 router 文件 28 个 message_key 替换（auth/orders/cdkeys/sessions/roles/users/checkin）
- Vben page.json 同步新增至 183 keys × 4 locales
- Python 语法 + JSON 校验 + 导入测试全部通过

#### Round I18N-02: Vben Admin 多语言基础设施
- 扩展 `SupportedLanguagesType` 支持 4 语言 (en-US/zh-CN/zh-TW/ja)
- 创建 12 份新 locale 文件 (packages 级 + app 级各 2 语言)
- 集成 Ant Design Vue / dayjs 的 zh-TW 和 ja locale
- admin.example.com 路由：88 个 title 全部 key 化 (新增 81 个 admin key)
- user-center.ts 路由：7 个硬编码中文 title key 化
- basic.vue 通知数据：4 条示例通知中文文案 key 化
- store/auth.ts：访问受限错误提示 key 化
- change-password 页面："正在跳转..." key 化
- 142 个 page.json key × 4 locales = 568 翻译条目，全语言一致
- `pnpm build` 14 tasks 全部通过

#### Round I18N-01: Portal + MiniApp 多语言基础设施
- 安装 `@nuxtjs/i18n`，配置 `strategy: 'no_prefix'`、懒加载、4 语言
- 创建 4 份 locale JSON（268 keys each）：前缀 common/nav/auth/setup/landing/footer/miniapp/lang
- Portal 外壳层 key 化：AppHeader、AppFooter、login、register、setup、index、3 个 layouts
- MiniApp 外壳层 key 化：topbar、菜单侧边栏、仪表盘卡片、签到区域、错误页、认证表单
- 语言切换 UI（AppHeader + MiniApp topbar）+ localStorage 持久化
- 客户端插件 `i18n-init.client.ts`：语言优先级 `?lang=` > localStorage > Accept-Language > zh-CN
- 268 keys × 4 locales = 1072 条翻译条目，全语言一致
- Smoke 测试 13/13 PASS，`pnpm build` 通过

#### I18N 统计汇总

| Round | 范围 | Keys × Locales | 翻译条目 |
|-------|------|----------------|----------|
| I18N-01 | Portal + MiniApp 外壳层 | 268 × 4 | 1,072 |
| I18N-02 | Vben Admin 基建 | 142 × 4 | 568 |
| I18N-03 | 后端 API + 菜单 | 192 × 4 (后端) + 183 × 4 (前端) | 1,500 |
| I18N-04 | Portal + MiniApp 全页面 | 1,422 × 4 | 5,688 |
| **合计** | **全栈** | | **8,828** |

---

## [1.1.22] - 2026-02-05

### 新功能

#### 密码显示/隐藏功能 (Issue #5)
- 登录和注册页面的密码输入框增加眼睛图标切换按钮
- 点击可切换密码明文/密文显示
- 影响页面：`/login`、`/register`

#### L2级菜单限制 (Issue #18)
- 将「MiniApp 菜单权限控制」重命名为「L2级菜单限制」
- 明确权限双重闸门逻辑：菜单可见 = L1角色权限 AND L2菜单限制
- 更新 Vben Admin 页面、后端菜单结构、设置中心

### 界面优化

#### H5 移动端菜单按钮 (Issue #8)
- 增大按钮尺寸 (36px → 56px)
- 添加 ring 外框和脉冲动画
- 增强阴影和交互反馈

#### H5 侧边栏动画 (Issue #10)
- 遮罩层淡入淡出 (0.3s)
- 侧边栏左滑进入/退出
- 使用 cubic-bezier 缓动曲线

#### MiniApp 首页 (Issue #16)
- 登录按钮文案改为「登录/注册」
- 服务器状态功能完成

### Bug 修复

#### 席位卡功能 (Issue #1, #2, #3)
- 席位配置默认值生成
- 用户已有席位显示
- CDKEY 兑换功能

---

## [1.1.21] - 2026-02-05

### 新功能

#### 注册表单 Placeholder 后台配置
- 用户名和密码输入框的提示文字可以在后台配置
- 新增设置项：`username_policy_placeholder`、`password_policy_placeholder`
- 影响：MiniApp 认领流程、Portal 注册页面

### 界面优化

#### MiniApp 主界面布局重构 (Issue #9)
用固定网格布局替代可拖拽的 Splitpanes，解决以下问题：
- Splitpanes 在手机上容易误触导致面板拖乱
- 滚动冲突，用户体验差
- 分区比例不好控制

新布局结构：
```
┌─────────────────────────────────┐
│  📢 公告横幅 (最多3条)           │
├─────────────────────────────────┤
│ [MY INFO]  │ 用户名/ID/等级/角色 │
├─────────────────────────────────┤
│ 席位/账户  │ [MEDIA ACCOUNTS]    │
├─────────────────────────────────┤
│ [签到日历] │ 日历+签到/补签功能   │
├─────────────────────────────────┤
│         🏠 返回首页              │
└─────────────────────────────────┘
```

设计特点：
- 标签块使用蓝紫渐变
- 标签左右交替排列
- 日历改为中文 UI

### Bug 修复

#### Refresh 按钮图标重复 (Issue #14)
- **问题**：席位页面刷新按钮显示重复图标
- **修复**：改用 UButton 的 icon prop 代替手动 UIcon

#### 认领流程验证同步
- **问题**：前后端验证不一致
- **修复**：同步验证规则，必须同时提供用户名和密码

#### 签到日期时区一致性 (Issue #12)
- **问题**：MiniApp 和 Portal 签到显示不同日期
- **修复**：统一使用服务器时区 (Asia/Shanghai)

---

## [1.1.20] - 2026-02-05

### 新功能

#### MiniApp 服务器状态页面
用户可在 MiniApp 首页查看所有媒体服务器的实时状态：
- 服务器在线/离线状态（10次重试确认离线）
- 服务器运行时间
- 内容统计（电影/剧集/集数）
- 实时播放人数
- 每小时自动刷新 + 管理员手动刷新

#### 平台身份与角色权限增强
- 角色模板编辑页改用 checkbox 分组选择器
- 角色管理新增 `platform_role_hint` 字段
- 用户被赋予角色时自动升级 `which_role`
- 支持重置老用户认领状态

### Bug 修复

#### #6 媒体账号修改密码失败
- **修复**：添加空值检查，改用中文错误提示

#### #3 积分兑换 400 错误无提示
- **修复**：加强错误处理，支持多种响应结构

#### #13 账户有效期日期格式歧义
- **修复**：Portal 使用 "5 Feb 2026" 格式，MiniApp 使用 "2026/02/05" 格式

---

## [1.1.19] - 2026-02-05

### Bug 修复

#### 积分兑换功能修复 (Critical)
- **问题**：用户积分兑换经验值时返回「积分不足」，即使用户有足够积分
- **根因**：签到系统将积分存储在 `Users.point` 字段，但兑换服务检查的是 `UserCredits.balance`（两个完全不同的表）
- **修复**：修改 `ExpExchangeService.exchange_points_to_exp()` 直接使用 `Users.point` 字段
- **影响范围**：所有用户的积分兑换功能

#### Portal 成长中心菜单缺失
- **问题**：Portal 左侧菜单没有「成长中心」入口
- **修复**：
  - 后端 `menu.py` UserCenter 添加 Growth 菜单项（权限：`exchange_points_to_exp`）
  - 前端 `portal.example.com` 降级菜单添加 Growth Center

#### JWT 过期处理优化
- **问题**：JWT 过期后，用户刷新页面会看到完整的英文菜单（降级菜单），而非跳转登录
- **修复**：`portal.example.com` 捕获 401 错误时自动清除 token 并跳转到登录页

### 文件变更

| 文件 | 变更类型 | 说明 |
|------|----------|------|
| `app/services/exp_exchange_service.py` | 修改 | 积分检查改用 `Users.point`，新增 `InsufficientPointsError` |
| `app/routers/user_level.py` | 修改 | 异常处理改用 `InsufficientPointsError` |
| `app/routers/menu.py` | 修改 | UserCenter 添加成长中心菜单项 |
| `nuxt-portal/layouts/portal.example.com` | 修改 | 401 跳转登录 + 降级菜单添加 Growth |

---

## [1.1.18] - 2026-02-05

### Bug 修复

#### 平台账号席位统计修复
- **问题**：平台账号席位显示 25 个用户，实际只有 14 个活跃用户
- **根因**：`/public/quota-stats` API 统计时包含了 `is_active=false` 的孤儿账号
- **修复**：用户计数查询条件从 `is_deleted=False` 改为 `is_deleted=False, is_active=True`

#### 签到功能 500 错误修复
- **问题**：用户签到时返回 500 Internal Server Error
- **根因**：`checkin_risk_engine.py` 中访问了不存在的字段 `risk_metadata`
- **修复**：将 `risk_metadata` 改为正确的字段名 `request_metadata`

#### APISIX 路由配置修复
- **问题**：外部 API 请求返回 404（内部直连正常）
- **根因**：`api-blue` 路由的 `regex_uri` 配置错误地移除了 `/api` 前缀
- **修复**：移除 `api-blue` 路由中的 `regex_uri` 配置

---

## [1.1.6] - 2026-01-30

### 新功能

#### Round-MINIAPP-02: MiniApp 首页（公开信息展示）
- **首页模式**：MiniApp 打开后先显示公开首页，用户点击"登录用户中心"后才进入登录流程
- **公告区块**：
  - 默认显示 1 条公告标题（自动截取）
  - 点击展开显示 5 条公告
  - 点击某条公告查看详情
  - "查看更多"跳转公告列表页
  - 仅显示 `audience=global` 的公开公告
- **席位状态区块**：
  - 显示 "席位状态 X / Y" 格式，加粗进度条
  - "已占用 X%" 白色加粗文字显示在进度条内部
  - 进度条颜色：<80% 蓝色，80-90% 绿色，90-100% 黄色，≥100% 红色
- **蓝紫渐变主题**：全局应用蓝紫渐变配色方案
- **新增公开 API**：`GET /api/v1/public/quota-stats` 返回席位配额统计

#### Round-MINIAPP-01c: MiniApp 注册表单增强
- **确认密码字段**：MiniApp 注册表单新增确认密码输入框，防止密码输入错误
- **邮箱字段**：支持后台设置 `registration_email_required` 控制是否必填邮箱
- **MiniApp 注册开关**：检查 `telegram_restricted_signup_enabled`，若为 false 则阻止 MiniApp 注册

#### Round-MINIAPP-01c: 后台注册设置扩展
- **邮箱必填开关**：新增 `registration_email_required` 设置项
- **定时开放时间**：新增 `registration_schedule_time` 设置项（schedule 模式）
- **倒计时天数**：新增 `registration_countdown_days` 设置项（countdown 模式）
- Vben 注册设置页面支持以上新字段的配置

#### Round-DEPLOY-05: Schema 完整性检查
- 启动时自动检查数据库 schema 完整性，缺失关键表时阻止启动

### Bug修复

#### Round-MINIAPP-02: MiniApp 菜单/导航/UI 修复
- **侧边栏菜单修复**：商城/席位路由映射错误修正（`ShopProducts` → `UserShop`）
- **HTTPS 拦截器修复**：dev 模式下通过外部域名访问时不再将 http 强制转 https，避免 SSL 连接失败
- **"返回首页" 按钮修复**：点击后正确返回 MiniApp 公开首页（清除登录状态，重新加载首页数据）
- **外部域名 API 路由**：dev 模式下通过外部域名访问时使用相对路径，让 Vite proxy 处理请求

#### Round-MINIAPP-01c: 统一邀请码错误提示（中文化）
- `Invitation code is required` → `请输入邀请码`
- `Invalid or already used invitation code` → `邀请码无效或已使用`
- `Invitation code has expired` → `邀请码已过期`
- `This invitation code is not applicable for your registration level` → `此邀请码不适用于您的注册等级`
- 所有邀请码相关错误响应添加 `error_code` 字段便于前端处理

### API 变更

- `GET /auth/register-settings` 返回新增 `email_required` 字段
- `PUT /admin/register-settings` 支持新字段：
  - `registration_email_required`
  - `registration_schedule_time`
  - `registration_countdown_days`
- MiniApp 注册 API 支持 `email` 参数
- `GET /api/v1/public/quota-stats` 新增公开 API

### 数据库迁移

运行以下迁移（如果尚未运行）：
```bash
psql -d your_database -f migrations/364_add_shop_seats_menu.sql
```

---

## [1.1.5] - 2025-01-29

### 新功能

#### Round-MENU-01e: 超级创建用户功能
- 新增权限 `super_create_user` - 超级创建用户（绕过邀请码/激活码）
- 权限限制：只有 sysop（系统第一个管理员）可以授予他人此权限
- 新增API端点：`POST /admin/users/super-create`
  - 需要 `super_create_user` 权限
  - 创建的用户自动激活（is_active=True）
  - source 标记为 `super_create` 以区分
- 前端交互：
  - 隐藏触发区域：用户管理页面右上角20x20像素区域
  - 三次点击触发红色警告确认框
  - 无权限用户看不到触发区域

#### Round-MENU-01f: 系统时区设置
- 新增系统设置项 `system_timezone`（默认：Asia/Shanghai）
- 支持时区：Asia/Shanghai, Asia/Tokyo, Asia/Singapore, Asia/Hong_Kong, UTC, America/New_York, America/Los_Angeles, Europe/London, Europe/Paris
- 新增时区工具函数（`app/utils.py`）：
  - `get_system_timezone()` - 异步获取系统时区（带5分钟缓存）
  - `get_system_timezone_sync()` - 同步获取时区
  - `format_datetime_tz()` - 格式化为指定时区时间
  - `now_in_system_tz()` - 获取系统时区当前时间
  - `utc_to_system_tz()` - UTC转系统时区
- 运行时日志时间戳使用系统时区记录

#### Round-MENU-01g: 媒体服务器管理优化
- **卡片式管理页面**：重构媒体服务页面为卡片式布局
  - 响应式布局（xl:4列, lg:3列, md:2列, sm/xs:1列）
  - 每个服务器显示为独立卡片
  - 自动检测所有启用服务的健康状态
- **服务类型切换**：添加/编辑时可选择 Emby 或 Jellyfin
- **服务器编号 (code)**：
  - 新增 `code` 字段（INTEGER, UNIQUE）
  - 迁移文件：`migrations/334_add_media_services_code.sql`
  - 用于 legacy 账号映射
  - 卡片显示编号标签 (#1, #2)
- **Legacy 账号认领优化**：
  - 动态 server_code 映射（替代硬编码）
  - 认领日期正确设置为 `created_at`、`updated_at`、`activation_date`

### Bug修复

#### Round-MINIAPP-01: MiniApp 邀请码注册支持
- **问题**：后台设置需要邀请码注册，但 MiniApp 没有显示邀请码输入框
- **修复**：
  - 后端 `/miniapp/register-initdata` API 添加 `invite_code` 参数
  - 后端 `/auth/telegram/miniapp/restricted-signup` API 添加 `invite_code` 参数
  - 注册前检查 `RegistrationPolicy`，验证邀请码有效性
  - 注册成功后自动消耗邀请码
  - 前端从 `/auth/register-settings` 获取 `invite_required` 设置
  - 当 `invite_required=true` 时显示邀请码输入框
  - 按钮在邀请码未填写时禁用

#### Round-MINIAPP-01b: 开发环境 API 域名映射（仅限 dev）
- **问题**：通过 cloudflared tunnel 访问 `portal.example.com` 时，API 请求发送到错误域名
- **修复**：
  - `miniapp.vue` 的 `getApiBase()` 添加 `portal.example.com → api.example.com` 映射
  - `useRuntimeConfig.ts` 的 `getInitialApiBase()` 添加相同映射
  - 仅影响开发环境，不影响生产环境（生产使用 APISIX 代理）

### 部署改进

#### Round-DEPLOY-05: Schema 完整性检查
- **问题背景**：发现生产环境 `media_services` 表缺失，但 `_migrations` 表记录显示已应用
- **新增检查**：在 migration 执行后验证关键表是否存在
- **检查的关键表**：
  - `usersDB` - 用户表
  - `rolesDB` - 角色表
  - `media_services` - 媒体服务表
  - `media_accounts` - 媒体账号表
  - `system_settings` - 系统设置表
  - `invitationDB` - 邀请码表
- **错误提示**：如果检测到缺失，输出详细错误信息并阻止启动
- **可检测的问题**：
  - 数据库备份不完整恢复
  - 手动删除表
  - Migration 跟踪不一致

### Bug修复

- **Whitelist API 500错误**：
  - 修复 `whitelist_exemption.py` 字段名错误
  - `ma.name` → `ma.external_username`
  - `ma.provider_type` → `ma.account_type`

- **ESLint 修复**（`users/index.vue`）：
  - 修复 union type 排序
  - 修复 `no-use-before-define` 错误
  - 修复 self-closing div 问题

### 数据库迁移

运行以下迁移（如果尚未运行）：
```bash
psql -d your_database -f migrations/334_add_media_services_code.sql
```

### 文件变更

**后端：**
- `app/routers/admin/roles.py` - 新增 super_create_user 权限和 sysop 限制
- `app/routers/admin/users.py` - 新增超级创建用户 API
- `app/routers/admin/system_settings.py` - 新增系统时区设置
- `app/routers/admin/whitelist_exemption.py` - 修复字段名
- `app/routers/admin/media_services.py` - 添加 code 字段支持
- `app/routers/auth.py` - 动态 server_code 映射、日期处理、邀请码支持（Round-MINIAPP-01）
- `app/routers/miniapp_api.py` - 注册 API 添加邀请码支持（Round-MINIAPP-01）
- `app/routers/menu.py` - 添加 super_create_user 权限
- `app/services/media_services.py` - 添加 code 相关方法
- `app/utils.py` - 添加时区工具函数

**部署：**
- `deploy/entrypoint.sh` - 新增 schema 完整性检查（Round-DEPLOY-05）

**前端：**
- `vue-vben-admin/.../users/index.vue` - 隐藏触发区域
- `vue-vben-admin/.../api/core/user.ts` - superCreateUser API
- `vue-vben-admin/.../api/core/media-services.ts` - code 字段类型
- `vue-vben-admin/.../settings-groups/providers.vue` - 卡片式管理页面
- `nuxt-portal/pages/miniapp.vue` - 添加邀请码输入框支持（Round-MINIAPP-01）

---

## [未发布] - Round-MENU-01

### 功能改进

- **Round-MENU-01**: 前端显示ID优化 - MiniApp和Portal隐藏数据库自增ID，改为显示display_id
  - MiniApp主页和个人资料页使用 `frontProfile.public_id ?? userData.display_id ?? userData.id` 优先级链
  - Portal个人资料页已正确使用 `frontProfile.public_id || userInfo.display_id` 显示
  - 所有认证成功路径增加 `fetchFrontProfile()` 调用，确保public_id在首页可用
  - public_id优先级: pretty_id(靓号) > display_id(普通9位ID) > id(数据库自增ID)

- **Round-MENU-01d**: Vben后台用户管理筛选功能实装
  - 实装搜索功能（之前仅有UI无实际功能）：
    - 用户名 - 模糊搜索
    - 显示ID - 模糊搜索
    - TG UID - 精确匹配
    - 媒体账号用户名 - 模糊搜索（支持跨所有媒体后端：Emby/Jellyfin/自研等）
  - UI优化：
    - 将 "Emby用户名" 改为 "媒体账号" - 因为1个用户可对应多个媒体账号，且支持多种媒体后端
    - 状态筛选使用 Switch 组件，带"是/否"文字，宽度控制在50px
    - 输入框宽度控制：用户名140px，显示ID/TG UID/媒体账号120px
  - 后端API增强 (`GET /admin/users`)：
    - 新增 `username` 参数 - 按用户名模糊搜索
    - 新增 `display_id` 参数 - 按显示ID模糊搜索
    - 新增 `tg_uid` 参数 - 按TG UID精确搜索
    - 新增 `media_username` 参数 - 按媒体账号用户名模糊搜索（跨表查询 `media_accounts.external_username`）

- **Round-MENU-01e**: 超级创建用户功能（权限受控+隐藏触发）
  - 新增权限 `super_create_user` - 超级创建用户（绕过邀请码/激活码）
  - 权限限制：只有 sysop（系统第一个管理员）可以授予他人此权限
  - 新增API端点：`POST /admin/users/super-create`
    - 需要 `super_create_user` 权限
    - 创建的用户自动激活（is_active=True）
    - source 标记为 `super_create` 以区分
    - 记录审计日志 `[SUPER CREATE]` 标识
  - 前端交互：
    - 移除可见的"新增用户"按钮
    - 有权限的用户可在用户管理页面右上角隐藏区域点击3次触发
    - 弹出红色警告框确认："是否使用超级创建用户功能？"
    - 注意事项明确："此功能将绕过邀请码/激活码限制，直接创建已激活的用户账号"
    - 创建成功后显示提示并自动关闭窗口
    - 无权限用户看不到触发区域，API也返回403

### Bug修复

- **Round-MENU-01**: 修复display_id前导零问题
  - 原问题: display_id生成算法会产生前导零（如 `040400138`）
  - 解决方案: 从 `100000000` 开始递增搜索，确保9位数无前导零
  - 新增API端点: `POST /admin/users/display-id/regenerate` - 重新生成所有用户的display_id

- **Round-MENU-01b**: 完善靓号规则
  - 新增规则：
    - 连续4个或以上的"0"算靓号（3个0允许，4个0不允许）
    - 同一数字在号码中出现超过4次算靓号（4次允许，5次不允许）
      - 例: `110403311` (4个1) ✓ 允许
      - 例: `111443311` (5个1) ✗ 靓号
    - 当9位数用完后自动扩展为10位数
  - 完整靓号规则：
    - 3个或以上连续相同的非零数字（如 `111`, `888`）
    - 4个或以上连续的"0"（如 `0000`，但 `000` 允许）
    - 3位顺增序列（如 `123`, `456`）
    - 3位顺降序列（如 `321`, `987`）
    - 同一数字出现超过4次
  - 已执行重新生成，第一个有效ID从 `100010144` 开始（避开了含有6个连续0的 `100000044`）

- **Round-MENU-01d**: 修复媒体账号搜索500错误
  - 问题描述: 按媒体账号用户名搜索时返回500错误
  - 根本原因: 使用了错误的字段名 `remote_username`，实际应为 `external_username`
  - 修复: `app/routers/admin/users.py` 第394行字段名更正

- **Round-MENU-01c**: 修复API响应缺少display_id字段的问题
  - 问题描述: MiniApp登录和认证相关API返回的user对象中缺少display_id字段
  - 根本原因: 多个API端点只返回了id,username,level等基础字段，未包含display_id
  - 修复文件：
    - `app/routers/auth.py`: 5处API响应添加display_id字段
      - `/login` - 普通登录
      - `/telegram/miniapp/login` - MiniApp密码登录
      - `/telegram/miniapp/restricted-signup` - 受限注册
      - `/telegram/miniapp/claim` - 遗留账号认领
      - `/telegram/miniapp/register` - 新账号注册
      - `/telegram/miniapp/login-session` - 会话登录
    - `app/routers/miniapp_api.py`: 4处API响应添加display_id字段
      - initData验证成功响应
      - 遗留账号认领成功响应
      - 注册成功响应
      - `/miniapp/me` 用户信息接口

### 文件变更

- `nuxt-portal/pages/miniapp.vue`:
  - 修改主页User ID显示为 `frontProfile?.public_id ?? userData?.display_id ?? userData?.id`
  - 修改个人资料页User ID显示逻辑
  - 11处认证成功路径增加 `fetchFrontProfile()` 加载调用
  - 更新 `userData` TypeScript 类型定义，添加 `display_id?: string` 字段

- `app/services/display_id_service.py`:
  - 新增常量 `DISPLAY_ID_START = 100000000`，`DISPLAY_ID_MAX_10_DIGIT = 9999999999`
  - 新增 `has_consecutive_zeros()` 函数 - 检测4个或以上连续的0
  - 新增 `has_digit_frequency_exceeded()` 函数 - 检测同一数字出现超过4次
  - 修改 `has_consecutive_identical()` 函数 - 只检测非零数字的连续
  - 修改 `is_lucky_number_pattern()` 函数 - 整合所有靓号规则
  - 修改 `find_next_valid_display_id()` 函数 - 支持9位用完后扩展到10位
  - 新增 `is_valid_display_id_candidate()` 函数
  - 新增 `get_last_display_id_number()` 函数
  - 重构 `generate_unique_display_id()` 函数 - 使用新的搜索算法
  - 新增 `regenerate_all_display_ids()` 函数 - 重新生成所有用户的display_id

- `app/routers/admin/users.py`:
  - 新增端点 `POST /display-id/regenerate` - 重新生成所有display_id

- `app/routers/auth.py`:
  - 5处API响应的user对象添加 `display_id` 字段

- `app/routers/miniapp_api.py`:
  - 4处API响应的user对象添加 `display_id` 字段

- `app/routers/admin/users.py`:
  - `get_users()` 函数新增搜索参数: `username`, `display_id`, `tg_uid`, `media_username`
  - 支持按媒体账号用户名跨表搜索（查询 `media_accounts.external_username` 字段）
  - 修复字段名错误: `remote_username` → `external_username`

- `vue-vben-admin/apps/web-antd/src/views/admin/users/index.vue`:
  - 实装筛选功能，将表单值传递给后端API
  - "Emby用户名" 改为 "媒体账号"
  - 状态筛选使用 Switch 组件（带"是/否"文字，宽度50px）
  - 输入框宽度控制：用户名140px，其他字段120px
  - **Round-MENU-01e**: 移除可见的"新增用户"按钮
  - **Round-MENU-01e**: 添加隐藏的右上角触发区域（3次点击触发）
  - **Round-MENU-01e**: 添加权限检查 `super_create_user`
  - **Round-MENU-01e**: 添加红色警告确认弹窗

- `vue-vben-admin/apps/web-antd/src/api/core/types.ts`:
  - `UserQueryParams` 接口新增搜索参数类型定义

- `vue-vben-admin/apps/web-antd/src/api/core/user.ts`:
  - **Round-MENU-01e**: 新增 `superCreateUser()` API函数

- `app/routers/admin/roles.py`:
  - **Round-MENU-01e**: 新增权限 `super_create_user` 到 AVAILABLE_PERMISSIONS
  - **Round-MENU-01e**: 新增权限组 `sysop_restricted`（系统管理员专属）
  - **Round-MENU-01e**: 添加权限授予限制 - 只有sysop能授予 `super_create_user`

- `app/routers/admin/users.py`:
  - **Round-MENU-01e**: 新增端点 `POST /admin/users/super-create` - 超级创建用户
  - 需要 `super_create_user` 权限
  - source 标记为 `super_create`
  - 用户创建后自动激活

---

## [1.1.0] - 2026-01-25

### 新功能

- **LEVEL-26,27,28,30C**: Vanity/Color Shop + Heuristic Risk Rules (a83ffc46)
- **LEVEL-29,31,32**: Unified Risk Policy Center + Audit Center v3 (75696e76)
- **LEVEL-30B-01**: enhance challenge config API with standardized error codes (4eeba048)
- **LEVEL-30A**: Challenge infrastructure and gate integration (d27385e7)
- **LEVEL-25**: enhanced API smoke test with time_total and menu consistency (17b5548d)
- **LEVEL-23**: audit enhancement with policy snapshot and client type tracking (fae65eaa)
- **LEVEL-22**: policy versioning + menu policy enforcement (23955885)
- **LEVEL-20/21**: ops analytics formula simulator + policy engine rules (6a0355b9)
- **LEVEL-18**: role permission templates and partner console API (4f6521fe)
- **LEVEL-17**: unified audit center with event logging and Vben UI (22a4436c)
- **LEVEL-16**: unified display profile system with badge management (ffd1a552)
- **LEVEL-15**: transaction-based delegation with preview/apply/rollback (37a5be2b)
- **LEVEL-14C**: Portal & MiniApp menu/route strictification (fc0abffc)
- **LEVEL-14B**: Vben menu/route strictification with partner role check (f83f7545)
- **LEVEL-12**: Whitelist audit CSV export endpoint (f5e05996)
- **LEVEL-11**: Platform Role Isolation & Partner Panel MVP (4bc5e9b6)
- **LEVEL-10**: Display Layer Unification - unified public_id, badges, and version (c53fc189)
- **LEVEL-09**: Governance Delegation system with hierarchy-based role management (91f2ec7f)
- **LEVEL-08**: Business Policies configuration system (d94a3a89)
- **LEVEL-07**: Business Tags & Whitelist Account management (625c8d23)
- **LEVEL-06**: Platform Identity three-entry isolation + RBAC + Partner Console MVP (5c92daec)
- **LEVEL05-05**: add delegation governance permissions (a17f622d)
- **LEVEL05-04**: add Vben delegation center UI (b3d9c0bd)
- **LEVEL05-03**: add delegation v2 API endpoints (dd9ba7fe)
- **LEVEL05-02**: add delegation policy service (1a6053f8)
- **LEVEL05-01**: add delegation governance migration and model update (1cd33185)
- **level04**: frontend pages for whitelist exemption (54d38eda)
- **level04**: media account expiration enforcer + whitelist integration (5a49e755)
- **level03**: platform role + delegation + sysop policy + default roles (b1458328)
- **level02-04**: add permissions and menu entries for business tags and whitelist exemption (9359d888)
- **LEVEL02-03**: Whitelist exemption for one media account per user (8633af7d)
- **LEVEL02-02**: Business identity tags (whitelist/vip/svip/partner) (b7cf19b7)
- **LEVEL02-01**: GrowthRole auto-match with fallback + recompute tool (bcba1386)
- **level-system**: implement Round-LEVEL-01 growth/level system (1aaf0339)

### Bug 修复

- **LEVEL-24**: resolve API smoke test failures and add evidence (eb98cb85)
- **lint**: remove unused imports (6d3d2e4c)

### 文档

- **LEVEL-33**: complete Gap Audit & Freeze round (bffd9f6a)
- update STATUS.md with CI infrastructure completion (85211e90)
- update STATUS.md with LEVEL-24 progress (b6586984)
- **LEVEL-15**: add planning and requirements documentation (449698d5)
- **LEVEL-14D**: verification PASS - all evidence collected (ab6d81ec)
- **LEVEL-14**: add UI screenshots for menu/route verification (e92cc5ff)
- **LEVEL-14**: mark LEVEL-14 as completed (756acfda)
- **LEVEL-14D**: add CI green evidence file (311202ae)
- **LEVEL-14**: update documentation and remove unused import (3adf3690)
- **LEVEL-14A**: add backend entry isolation evidence files (ecf37870)
- **LEVEL-13**: add evidence files for governance delegation verification (6e82154a)
- **LEVEL-08**: update documentation for Business Policies (0847d681)
- **LEVEL-07**: update documentation for Business Tags & Whitelist (8872f6a2)
- **LEVEL-06**: update ROUNDREQ and execution docs for Platform Identity (6291784f)
- **LEVEL05**: add evidence logs and update STATUS.md (9ab33987)
- update STATUS.md - Round-LEVEL-04 frontend complete (da34c7dc)
- update STATUS.md - Round-LEVEL-03 complete (36d04b07)
- update STATUS.md - Round-LEVEL-02 complete (44a2c7c7)

### 测试

- **LEVEL-24B**: add Playwright E2E test for Vben audit center (796efbd8)
- **LEVEL-24A**: add API smoke test script for Round-23 audit features (b2f0e8bb)

### CI/CD

- **LEVEL-24**: add E2E compose stack and seed script (8ab47643)
- **LEVEL-24C**: add level24-audit-gate job for API smoke and Playwright tests (854313d0)
