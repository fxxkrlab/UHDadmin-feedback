# Round-MENU-01e/f/g 更新日志

## 版本: Round-MENU-01e/f/g
## 日期: 2025-01-29

---

## Round-MENU-01g: 媒体服务器管理优化

### 1. 媒体服务器管理页面重构

**问题:** 旧版是卡片式管理，新版变成了表单设置页面

**修复:**
- 将 `vue-vben-admin/apps/web-antd/src/views/admin/settings-groups/providers.vue` 重构为卡片式管理页面
- 支持响应式布局（xl:4列, lg:3列, md:2列, sm/xs:1列）
- 每个服务器显示为独立卡片，包含健康状态、延迟、版本信息
- 自动检测所有启用服务的健康状态

### 2. 服务类型切换

**问题:** 添加媒体服务器时没有 Emby/Jellyfin 切换

**修复:**
- 在添加/编辑模态框中添加服务类型下拉选择
- 支持 Emby 和 Jellyfin 两种类型
- 图标和颜色区分（Emby=绿色, Jellyfin=紫色）

### 3. 服务器编号 (code) 字段

**问题:** 需要编号字段用于 legacy 账号映射

**实现:**
- 新增数据库字段: `media_services.code` (INTEGER, UNIQUE)
- 添加迁移文件: `migrations/334_add_media_services_code.sql`
- 支持手动指定编号（纯数字，如 1, 2, 999）
- 编号不允许重复
- 服务列表按编号排序

**后端修改:**
- `app/services/media_services.py`:
  - 所有查询添加 `code` 字段
  - 新增 `get_service_by_code()` 方法
  - 新增 `is_code_unique()` 验证方法
  - 新增 `get_code_to_service_mapping()` 映射方法
- `app/routers/admin/media_services.py`:
  - `MediaServiceCreate` 和 `MediaServiceUpdate` 添加 `code` 字段
  - 创建和更新时验证编号唯一性

**前端修改:**
- `vue-vben-admin/apps/web-antd/src/api/core/media-services.ts`:
  - 类型定义添加 `code` 字段
- `vue-vben-admin/apps/web-antd/src/views/admin/settings-groups/providers.vue`:
  - 表单添加编号输入框
  - 卡片显示编号标签 (#1, #2 等)

### 4. Legacy 账号认领映射优化

**问题:** 老用户认领账号时使用硬编码的 server_code 映射

**修复:**
- `app/routers/auth.py`:
  - `get_server_code_mapping()` 改为动态从数据库查询
  - 使用 `MediaServiceManager.get_code_to_service_mapping()`
  - 移除硬编码的 `{1: 1767356127189, 2: 1768235573713}` 映射

### 5. Legacy 账号认领日期处理

**问题:** 认领时媒体账号日期设置不正确

**修复:**
- `app/routers/auth.py` - 认领流程:
  - `created_at` = 认领日期
  - `updated_at` = 认领日期
  - `activation_date` = 认领日期

---

## 新功能

### 1. 超级创建用户功能 (super_create_user)

**后端修改:**

- **`app/routers/admin/roles.py`**
  - 新增 `super_create_user` 权限到 `AVAILABLE_PERMISSIONS`
  - 新增 `sysop_restricted` 权限组
  - 添加限制：只有 sysop 账号可以授予 `super_create_user` 权限给其他用户
  - 新增 `SYSOP_ONLY_PERMISSIONS` 列表用于控制特殊权限

- **`app/routers/admin/users.py`**
  - 新增 `POST /admin/users/super-create` API 端点
  - 需要 `super_create_user` 权限
  - 创建用户时设置 `is_active=True`, `source="super_create"`

- **`app/routers/menu.py`**
  - 将 `super_create_user` 添加到 `ALL_PERMISSIONS` 列表

**前端修改:**

- **`vue-vben-admin/apps/web-antd/src/views/admin/users/index.vue`**
  - 移除原有的"新增用户"按钮
  - 添加隐藏触发区域（20x20像素，位于右上角）
  - 实现三次点击触发机制
  - 使用 `menuStore.permissions` 进行权限检查
  - 点击触发后显示红色警告确认弹窗

- **`vue-vben-admin/apps/web-antd/src/api/core/user.ts`**
  - 新增 `superCreateUser()` API 函数

---

### 2. 系统时区设置 (Round-MENU-01f)

**后端修改:**

- **`app/routers/admin/system_settings.py`**
  - 新增 `system_timezone` 设置项
    - 类别: `system`
    - 类型: `string`
    - 默认值: `"Asia/Shanghai"`
    - 可选项: Asia/Shanghai, Asia/Tokyo, Asia/Singapore, Asia/Hong_Kong, UTC, America/New_York, America/Los_Angeles, Europe/London, Europe/Paris
  - 新增 `system` 设置类别（系统设置）
  - 更新设置时自动清除时区缓存

- **`app/utils.py`**
  - 新增 `zoneinfo` 模块导入
  - 新增以下时区工具函数:
    - `get_system_timezone()` - 异步获取系统配置的时区（带缓存）
    - `get_system_timezone_sync()` - 同步获取时区（用于日志等非异步场景）
    - `format_datetime_tz(dt, tz, fmt)` - 将 datetime 格式化为指定时区的字符串
    - `now_in_system_tz()` - 获取系统时区的当前时间
    - `utc_to_system_tz(dt)` - 将 UTC datetime 转换为系统时区
    - `clear_timezone_cache()` - 清除时区缓存
  - 时区缓存 TTL: 5分钟
  - 更新 `log_runtime_action()` 使用系统时区记录时间戳

---

## Bug 修复

### 1. Whitelist API 500 错误修复

**问题:** 用户详情页面加载 Whitelist 豁免信息时返回 500 错误
- 错误信息: `AttributeError: 'MediaAccount' object has no attribute 'name'`

**原因:** `MediaAccount` 模型字段名称不正确
- `name` → 应为 `external_username`
- `provider_type` → 应为 `account_type`

**修复文件:** `app/routers/admin/whitelist_exemption.py`

```python
# 修复前
"name": ma.name,
"provider_type": ma.provider_type,

# 修复后
"name": ma.external_username or f"Account #{ma.id}",
"provider_type": ma.account_type,
```

---

## 文件清单

### 后端文件修改:
1. `app/routers/admin/roles.py` - 权限定义和 sysop 限制
2. `app/routers/admin/users.py` - 超级创建用户 API
3. `app/routers/menu.py` - 权限列表更新
4. `app/routers/admin/system_settings.py` - 系统时区设置
5. `app/routers/admin/whitelist_exemption.py` - 字段名修复
6. `app/utils.py` - 时区工具函数

### 前端文件修改:
1. `vue-vben-admin/apps/web-antd/src/views/admin/users/index.vue` - 隐藏触发区域
2. `vue-vben-admin/apps/web-antd/src/api/core/user.ts` - superCreateUser API

### Round-MENU-01g 修改文件:

**数据库迁移:**
- `migrations/334_add_media_services_code.sql` - 新增 code 字段

**后端:**
- `app/services/media_services.py` - 添加 code 字段支持和新方法
- `app/routers/admin/media_services.py` - API 添加 code 字段和唯一性验证
- `app/routers/auth.py` - 动态 server_code 映射和日期处理

**前端:**
- `vue-vben-admin/apps/web-antd/src/api/core/media-services.ts` - 类型添加 code 字段
- `vue-vben-admin/apps/web-antd/src/views/admin/settings-groups/providers.vue` - 卡片式管理页面重构

### ESLint 修复:
- 修复 union type 排序: `null | ReturnType<typeof setTimeout>`
- 修复 `no-use-before-define`: 将 `showSuperCreateWarning` 移至 `getModalContainer` 和 `showCreateModal` 定义之后
- 修复 `vue/html-self-closing`: 将 `<div ... />` 改为 `<div ...></div>`

---

## 验证清单

### Round-MENU-01e/f
- [ ] 超级创建用户: 只有拥有 `super_create_user` 权限的用户能看到隐藏触发区域
- [ ] 超级创建用户: 三次点击触发红色警告弹窗
- [ ] 超级创建用户: 只有 sysop 能授予 `super_create_user` 权限
- [ ] 系统时区: 设置中心可以配置 `system_timezone`
- [ ] 系统时区: 日志时间戳使用配置的时区显示
- [ ] Whitelist API: 用户详情页面可以正常加载 Whitelist 豁免信息

### Round-MENU-01g 媒体服务器管理
- [ ] 媒体服务页面显示为卡片式布局
- [ ] 添加服务时可以选择 Emby 或 Jellyfin 类型
- [ ] 添加服务时可以设置编号 (code)
- [ ] 编号不允许重复（创建/更新时校验）
- [ ] 服务列表按编号排序显示
- [ ] 编辑服务时可以修改编号
- [ ] 健康检测正常工作（延迟显示、服务器版本显示）
- [ ] 运行迁移: `psql -f migrations/334_add_media_services_code.sql`
- [ ] Legacy 账号认领: server_code 能正确映射到 media_service_id

---

## 注意事项

1. **超级创建用户权限**是高危权限，仅限 sysop 账号授予
2. 时区设置修改后会自动清除缓存，立即生效
3. 时区缓存 TTL 为 5 分钟，频繁读取时不会重复查询数据库
