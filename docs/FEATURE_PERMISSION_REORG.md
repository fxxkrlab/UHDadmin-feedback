# 权限系统重组 / Permission System Reorganization

[English Version](#english-version)

## 1. 概述

v1.1.22 对权限系统进行重组，将权限按**受众**分为两大类：
- **用户权限 (User Permissions)** - Portal/MiniApp 用户可用的功能
- **管理员权限 (Admin Permissions)** - Vben 后台管理员专用功能

## 2. 设计原则

1. **每个页面至少一个权限**：没有对应权限的用户看不到该页面/菜单
2. **按受众分类**：用户权限在上，管理员权限在下
3. **功能性子分类**：每个大类下按功能进一步细分
4. **向后兼容**：现有权限键不变，只调整分组结构

## 3. 权限覆盖分析

### 3.1 Portal 页面权限对照

| Portal 页面 | 对应权限 | 状态 |
|------------|---------|------|
| profile.vue | `view_user_center` | ✓ 已有 |
| invite-codes.vue | `view_invite_codes` | ✓ 已有 |
| tickets.vue | `view_tickets` | ✓ 已有 |
| orders.vue | `view_orders` | ✓ 已有 |
| after-sales.vue | `view_after_sales` | ✓ 已有 |
| wallet.vue | `view_wallet` | ✓ 已有 |
| media-accounts.vue | `view_media_accounts` | ✓ 已有 |
| growth.vue | `view_growth` | ✓ 已有 |
| shop.vue | `shop:buy_vanity`, `shop:buy_color` | ✓ 已有 |
| affiliate.vue | `aff_enabled` | ✓ 已有 |
| display-preferences.vue | `view_display_profiles` | ✓ 已有 |
| whitelist.vue | `set_whitelist_exemption_self` | ✓ 已有 |
| invite-tree.vue | `view_invite_tree` | ⚠️ 新增 |
| renewal-cards.vue | `view_renewal_cards` | ⚠️ 新增 |
| watch-history.vue | `view_watch_history` | ⚠️ 新增 |
| benefits.vue | `view_benefits` | ⚠️ 新增 |
| cdkeys.vue | `view_cdkeys` | ⚠️ 新增 |
| checkin.vue | `view_checkin` | ⚠️ 新增 |
| seats.vue | `view_seats` | ⚠️ 新增 |

### 3.2 MiniApp 页面权限对照

| MiniApp 页面 | 对应权限 | 状态 |
|-------------|---------|------|
| server-status.vue | `view_server_status` | ⚠️ 新增 |

## 4. 新增权限列表

| 权限键 | 中文描述 | 对应页面 |
|--------|---------|---------|
| `view_seats` | 查看席位 | seats.vue |
| `view_checkin` | 签到日历 | checkin.vue |
| `view_benefits` | 会员权益 | benefits.vue |
| `view_cdkeys` | 我的CDKEY | cdkeys.vue |
| `view_renewal_cards` | 续期卡 | renewal-cards.vue |
| `view_watch_history` | 观看历史 | watch-history.vue |
| `view_invite_tree` | 邀请树 | invite-tree.vue |
| `view_server_status` | 服务器状态 | miniapp/server-status.vue |

## 5. 权限分组结构

### 5.1 用户权限 (user_permissions)

```
用户权限 (Portal/MiniApp)
├── user_basic - 基础功能
│   ├── view_user_center (访问用户中心)
│   ├── change_password (修改密码)
│   └── view_user_display_id (查看显示ID)
│
├── user_media - 媒体账号
│   ├── view_media_accounts (查看媒体账号)
│   ├── view_seats (查看席位)
│   ├── view_server_status (查看服务器状态)
│   └── view_watch_history (查看观看历史)
│
├── user_invite - 邀请与推广
│   ├── view_invite_codes (查看邀请码)
│   ├── create_invite_codes (创建邀请码)
│   ├── invite_purchase (购买邀请码)
│   ├── aff_enabled (推广功能)
│   └── view_invite_tree (查看邀请树)
│
├── user_service - 工单与售后
│   ├── view_tickets (工单)
│   ├── view_orders (订单)
│   └── view_after_sales (售后)
│
├── user_wallet - 钱包与积分
│   ├── view_wallet (钱包)
│   ├── credits_exchange (Credits兑换)
│   └── credits_topup (Credits充值)
│
├── user_growth - 成长与签到
│   ├── view_growth (成长信息)
│   ├── view_exp_ledger (经验记录)
│   ├── exchange_points_to_exp (积分换经验)
│   ├── view_checkin (签到日历)
│   └── view_benefits (会员权益)
│
├── user_shop - 商城与兑换
│   ├── shop:buy_vanity (购买靓号)
│   ├── shop:buy_color (购买颜色)
│   ├── redeem_pretty_id (兑换靓号)
│   ├── redeem_username_style (兑换用户名样式)
│   ├── view_cdkeys (查看我的CDKEY)
│   └── view_renewal_cards (查看续期卡)
│
├── user_bindind - 账号绑定
│   ├── bind_telegram (绑定Telegram)
│   └── bind_email (绑定Email)
│
└── user_display - 展示与白名单
    ├── view_display_profiles (展示偏好)
    └── set_whitelist_exemption_self (设置白名单豁免)
```

### 5.2 管理员权限 (admin_permissions)

```
管理员权限 (Vben后台)
├── admin_users - 用户管理
│   ├── manage_users
│   ├── manage_user_lifecycle
│   ├── manage_media_accounts
│   ├── admin_manage_identity
│   ├── view_user_dates
│   ├── view_user_logs
│   └── manage_legacy_users
│
├── admin_system - 系统管理
│   ├── access_system_management
│   ├── manage_system_settings
│   ├── manage_media_services
│   ├── view_system_logs
│   ├── manage_app_tokens
│   ├── manage_policies
│   ├── view_policies
│   └── manage_media_access
│
├── admin_roles - 角色与权限
│   ├── manage_roles
│   ├── manage_roles_nickname
│   ├── manage_role_permissions
│   └── manage_permission_templates
│
├── admin_ops - 运营管理
│   ├── manage_ops_panel
│   ├── system_invite_manage
│   ├── invite_expiry_bulk_edit
│   ├── promote_restricted_user
│   ├── grant_seat_admin
│   ├── aff_manage
│   ├── view_ops_analytics
│   └── manage_ops_analytics
│
├── admin_growth - 成长体系管理
│   ├── manage_level_rules
│   ├── manage_growth_roles
│   ├── manage_growth_rules
│   ├── manage_growth_recompute
│   ├── view_growth_analytics
│   ├── admin_exp_adjust
│   ├── manage_exchange_settings
│   ├── manage_business_tags
│   ├── manage_business_policies
│   ├── view_business_policies
│   ├── view_whitelist_audits
│   └── manage_whitelist_exemption_admin
│
├── admin_shop - 商城管理
│   ├── admin:manage_vanity_products
│   ├── admin:manage_color_products
│   ├── admin:issue_cdkeys
│   ├── admin:export_cdkeys
│   ├── manage_pretty_id_assign
│   ├── manage_username_style_assign
│   └── manage_id_style_settings
│
├── admin_governance - 治理委任
│   ├── manage_platform_role
│   ├── manage_delegation
│   ├── manage_delegations
│   ├── manage_admin_roles
│   ├── manage_governance
│   ├── view_delegation_audits
│   └── view_governance_audits
│
├── admin_audit - 审计中心
│   ├── view_audit_basic
│   ├── view_audit_sensitive
│   ├── view_audit_center
│   ├── export_audit
│   ├── export_audit_data
│   ├── audit_export
│   ├── manage_audit_settings
│   └── view_policy_snapshots
│
├── admin_risk - 风控管理
│   ├── manage_checkin_risk
│   ├── view_checkin_risk
│   ├── risk:view
│   ├── risk:manage_rules
│   └── risk:manage_actions
│
├── admin_display - 展示管理
│   ├── manage_display_profiles
│   ├── manage_display_policy
│   ├── manage_display_settings
│   ├── view_user_display_profile
│   └── export_users_display
│
└── admin_sysop - Sysop专属
    ├── manage_sysop_policy
    ├── super_create_user
    ├── manage_id_system_settings
    ├── maintenance_cleanup_vanity_color
    ├── manage_entitlements
    ├── system_cdkey_create
    └── system_cdkey_dispatch
```

## 6. 文件变更清单

| 文件 | 类型 | 说明 |
|------|------|------|
| `app/routers/admin/roles.py` | 修改 | 新增权限 + 重组 permission_groups |
| `nuxt-portal/pages/portal/*.vue` | 修改 | 各页面添加权限检查（如需要） |

## 7. 默认角色权限

新增的用户权限应自动授予所有用户角色（user 及以上）：
- `view_seats`
- `view_checkin`
- `view_benefits`
- `view_cdkeys`
- `view_renewal_cards`
- `view_watch_history`
- `view_invite_tree`
- `view_server_status`

## 8. 前端实现

权限编辑界面需要按新结构渲染：

```vue
<template>
  <div class="permission-editor">
    <!-- 用户权限分组 -->
    <div class="permission-category">
      <h3>🧑 用户权限 (Portal/MiniApp)</h3>
      <div v-for="group in userPermissionGroups" :key="group.key">
        <h4>{{ group.label }}</h4>
        <div class="permission-list">
          <Checkbox v-for="perm in group.permissions" :key="perm" />
        </div>
      </div>
    </div>

    <!-- 管理员权限分组 -->
    <div class="permission-category">
      <h3>🛡️ 管理员权限 (Vben后台)</h3>
      <div v-for="group in adminPermissionGroups" :key="group.key">
        <h4>{{ group.label }}</h4>
        <div class="permission-list">
          <Checkbox v-for="perm in group.permissions" :key="perm" />
        </div>
      </div>
    </div>
  </div>
</template>
```

---

# English Version

## Overview

v1.1.22 reorganizes the permission system by **audience**:
- **User Permissions** - Features available to Portal/MiniApp users
- **Admin Permissions** - Features for Vben admin panel (staff/admin/sysop only)

## Design Principles

1. **Every page needs at least one permission** - Users without permission cannot see the page/menu
2. **Categorize by audience** - User permissions on top, admin permissions below
3. **Functional subcategories** - Further organize by function within each category
4. **Backward compatible** - Existing permission keys unchanged, only grouping restructured

## New Permissions

| Permission Key | Description | Page |
|---------------|-------------|------|
| `view_seats` | View seats | seats.vue |
| `view_checkin` | Check-in calendar | checkin.vue |
| `view_benefits` | Member benefits | benefits.vue |
| `view_cdkeys` | My CDKEYs | cdkeys.vue |
| `view_renewal_cards` | Renewal cards | renewal-cards.vue |
| `view_watch_history` | Watch history | watch-history.vue |
| `view_invite_tree` | Invite tree | invite-tree.vue |
| `view_server_status` | Server status | miniapp/server-status.vue |
