# Vben Admin Menu Reorganization

**Date:** 2026-01-28
**Round:** MENU-01
**Status:** Ready for local testing

## Changes Summary

| Old Category | Items | New Categories |
|--------------|-------|----------------|
| 系统管理 (28 items) | SPLIT | 账号权限 (6) + 成长体系 (7) + 治理中心 (7) + 系统监控 (6) |
| 运维中心 (28 items) | SPLIT | 推广管理 (4) + 客服中心 (3) + 商城中心 (6) + 财务中心 (5) + 媒体管理 (3) + 运营监控 (8) |
| 审计中心 (1 item) | EXPANDED | 审计中心 (2) - added 白名单审计 |

**New URL paths:**
- `/admin/accounts/*` - 账号权限 (Users & Permissions)
- `/admin/growth/*` - 成长体系 (Growth System)
- `/admin/governance/*` - 治理中心 (Governance)
- `/admin/affiliate/*` - 推广管理 (Affiliate)
- `/admin/service/*` - 客服中心 (Customer Service)
- `/admin/shop/*` - 商城中心 (Shop Center)
- `/admin/finance/*` - 财务中心 (Finance)
- `/admin/media/*` - 媒体管理 (Media)
- `/admin/ops/*` - 运营监控 (Operations) - path unchanged, content reduced
- `/admin/system/*` - 系统监控 (System Monitor) - path unchanged, content reduced
- `/admin/audit/*` - 审计中心 (Audit) - path unchanged, content expanded

---

## Current Structure (Before)

Total visible menu items: ~66

### 1. 概览 (Dashboard) - 2 items
| # | Name | Title |
|---|------|-------|
| 1 | Analytics | 数据分析 |
| 2 | Workbench | 工作台 |

### 2. 用户中心 (User Center) - 15 items
| # | Name | Title |
|---|------|-------|
| 1 | UserProfile | 个人信息 |
| 2 | UserWallet | 我的钱包 |
| 3 | InviteCodes | 邀请码管理 |
| 4 | UserAffiliate | 推广中心 |
| 5 | UserOrders | 我的订单 |
| 6 | UserTickets | 我的工单 |
| 7 | UserAfterSales | 我的售后 |
| 8 | UserMediaAccounts | 媒体账号 |
| 9 | UserInviteTree | 邀请树 |
| 10 | UserCheckin | 每日签到 |
| 11 | UserSeats | 我的席位 |
| 12 | UserRenewalCards | 续期卡 |
| 13 | UserShop | 商城 |
| 14 | UserCDKeys | CDKEY管理 |
| 15 | UserWhitelist | 白名单 |

### 3. 设置中心 (Settings Center) - 7 items
| # | Name | Title |
|---|------|-------|
| 1 | SettingsOverview | 设置总览 |
| 2 | RegistrationSettings | 注册与限制 |
| 3 | DomainSettings | 域名与CORS |
| 4 | PaymentSettings | 支付设置 |
| 5 | MediaSettings | 媒体服务 |
| 6 | TelegramBotsSettings | TG Bots |
| 7 | LegacySettings | 旧版设置 |

### 4. 系统管理 (System Management) - 28 items
| # | Name | Title |
|---|------|-------|
| 1 | UserManagement | 用户管理 |
| 2 | RoleManagement | 角色管理 |
| 3 | RoleBind | 角色匹配绑定 |
| 4 | RolePermissions | 角色权限设置 |
| 5 | PermissionTemplates | 权限模板 |
| 6 | RoleTemplates | 角色模板 |
| 7 | SystemLogs | 系统日志 |
| 8 | SystemDiagnostics | 系统诊断 |
| 9 | RuntimeLogs | 运行日志 |
| 10 | SystemMetrics | 系统指标 |
| 11 | AppTokenManagement | App Token |
| 12 | PolicyManagement | 策略管理 |
| 13 | PaymentProviders | 支付服务 |
| 14 | FxSettings | 汇率设置 |
| 15 | SlaveStatus | Slave 状态 |
| 16 | LevelRules | 等级规则 |
| 17 | GrowthRoles | 成长称号 |
| 18 | GrowthRecompute | 批量升级 |
| 19 | GrowthAnalytics | 成长分析 |
| 20 | DisplayExport | 展示导出 |
| 21 | BusinessTags | 商业标签 |
| 22 | BusinessPolicies | 权益策略 |
| 23 | WhitelistExemptions | Whitelist豁免 |
| 24 | Delegation | 角色委任 |
| 25 | Delegations | 委任事务 |
| 26 | DelegationAudits | 委任审计 |
| 27 | DelegationGraph | 委任图谱 |
| 28 | SysopPolicy | Sysop策略 |
| 29 | GovernanceAudits | 治理审计 |

### 5. 运维中心 (Operations Center) - 28 items
| # | Name | Title |
|---|------|-------|
| 1 | InviteTree | 邀请树管理 |
| 2 | SystemInvites | 系统邀请码 |
| 3 | AffiliateManagement | 推广管理 |
| 4 | AffiliateSettings | 推广设置 |
| 5 | OrderManagement | 订单管理 |
| 6 | TicketManagement | 工单管理 |
| 7 | AfterSalesManagement | 售后管理 |
| 8 | MediaAccountManagement | 媒体账号 |
| 9 | MiniAppSettings | MiniApp 设置 |
| 10 | TelegramBots | Telegram Bots |
| 11 | OpsPanel | 运营面板 |
| 12 | OpsAnalytics | 运营分析 |
| 13 | PolicyEngineRules | 策略规则 |
| 14 | CheckinRiskControl | 签到风控 |
| 15 | CheckinSettings | 签到活动管理 |
| 16 | PermissionTemplates | 权限模板 (重复) |
| 17 | RoamHub | 漫游中台 |
| 18 | RenewalCards | 续期卡管理 |
| 19 | ShopProducts | 商城管理 |
| 20 | ShopOrders | 商店订单 |
| 21 | SystemDispatch | 系统资源派发 |
| 22 | AdminCDKeys | CDKEY管理 |
| 23 | PaymentReconcile | 支付对账 |
| 24 | RefundManagement | 退款管理 |
| 25 | PromoManagement | 促销/优惠码 |
| 26 | RiskControl | 风控管理 |
| 27 | SettlementReport | 结算报表 |
| 28 | WhitelistAudits | 白名单审计 |

### 6. CMS中心 (CMS Center) - 2 items
| # | Name | Title |
|---|------|-------|
| 1 | ContentManagement | 内容管理 |
| 2 | AnnouncementsManagement | 公告管理 |

### 7. 审计中心 (Audit Center) - 1 item
| # | Name | Title |
|---|------|-------|
| 1 | AuditEvents | 审计事件 |

---

## Issues with Current Structure

1. **系统管理** has 28+ items - too many, mixes unrelated functions
2. **运维中心** has 28 items - too many, mixes unrelated functions
3. **PermissionTemplates** appears in both 系统管理 and 运维中心 (duplicate)
4. **Telegram Bots** appears in both 设置中心 and 运维中心
5. Growth system items scattered in 系统管理
6. Finance items scattered across 系统管理 and 运维中心
7. Shop items scattered in 运维中心

---

## Proposed New Structure (After)

Total categories: 12 (from 7)
Each category: 2-8 items (instead of 28)

### 1. 概览 (Dashboard) - UNCHANGED
- Analytics (数据分析)
- Workbench (工作台)

### 2. 用户中心 (User Center) - UNCHANGED
- (15 items remain the same)

### 3. 设置中心 (Settings Center) - UNCHANGED
- (7 items remain the same)

### 4. 账号权限 (Users & Permissions) - NEW (from 系统管理)
- UserManagement (用户管理)
- RoleManagement (角色管理)
- RoleBind (角色匹配绑定)
- RolePermissions (角色权限设置)
- PermissionTemplates (权限模板)
- RoleTemplates (角色模板)

### 5. 成长体系 (Growth System) - NEW (from 系统管理)
- LevelRules (等级规则)
- GrowthRoles (成长称号)
- GrowthRecompute (批量升级)
- GrowthAnalytics (成长分析)
- BusinessTags (商业标签)
- BusinessPolicies (权益策略)
- DisplayExport (展示导出)

### 6. 治理中心 (Governance) - NEW (from 系统管理)
- WhitelistExemptions (Whitelist豁免)
- Delegation (角色委任)
- Delegations (委任事务)
- DelegationAudits (委任审计)
- DelegationGraph (委任图谱)
- SysopPolicy (Sysop策略)
- GovernanceAudits (治理审计)

### 7. 推广管理 (Affiliate) - NEW (from 运维中心)
- InviteTree (邀请树管理)
- SystemInvites (系统邀请码)
- AffiliateManagement (推广管理)
- AffiliateSettings (推广设置)

### 8. 客服中心 (Customer Service) - NEW (from 运维中心)
- OrderManagement (订单管理)
- TicketManagement (工单管理)
- AfterSalesManagement (售后管理)

### 9. 商城中心 (Shop Center) - NEW (from 运维中心)
- ShopProducts (商城管理)
- ShopOrders (商店订单)
- AdminCDKeys (CDKEY管理)
- RenewalCards (续期卡管理)
- SystemDispatch (系统资源派发)
- PromoManagement (促销/优惠码)

### 10. 财务中心 (Finance) - NEW (from 系统管理+运维中心)
- PaymentProviders (支付服务)
- FxSettings (汇率设置)
- PaymentReconcile (支付对账)
- RefundManagement (退款管理)
- SettlementReport (结算报表)

### 11. 媒体管理 (Media) - NEW (from 运维中心+系统管理)
- MediaAccountManagement (媒体账号)
- RoamHub (漫游中台)
- SlaveStatus (Slave 状态)

### 12. 运营监控 (Operations) - REFORMED (from 运维中心)
- OpsPanel (运营面板)
- OpsAnalytics (运营分析)
- MiniAppSettings (MiniApp 设置)
- TelegramBots (Telegram Bots)
- CheckinRiskControl (签到风控)
- CheckinSettings (签到活动管理)
- RiskControl (风控管理)
- PolicyEngineRules (策略规则)

### 13. 系统监控 (System Monitor) - NEW (from 系统管理)
- SystemLogs (系统日志)
- RuntimeLogs (运行日志)
- SystemDiagnostics (系统诊断)
- SystemMetrics (系统指标)
- AppTokenManagement (App Token)
- PolicyManagement (策略管理)

### 14. 审计中心 (Audit Center) - EXPANDED
- AuditEvents (审计事件)
- WhitelistAudits (白名单审计) ← moved from 运维中心

### 15. CMS中心 (CMS Center) - UNCHANGED
- ContentManagement (内容管理)
- AnnouncementsManagement (公告管理)

---

## Migration Comparison Table

| Menu Item | Before (旧分类) | After (新分类) |
|-----------|----------------|---------------|
| **用户管理** | 系统管理 | 账号权限 |
| **角色管理** | 系统管理 | 账号权限 |
| **角色匹配绑定** | 系统管理 | 账号权限 |
| **角色权限设置** | 系统管理 | 账号权限 |
| **权限模板** | 系统管理 + 运维中心(重复) | 账号权限 |
| **角色模板** | 系统管理 | 账号权限 |
| **等级规则** | 系统管理 | 成长体系 |
| **成长称号** | 系统管理 | 成长体系 |
| **批量升级** | 系统管理 | 成长体系 |
| **成长分析** | 系统管理 | 成长体系 |
| **商业标签** | 系统管理 | 成长体系 |
| **权益策略** | 系统管理 | 成长体系 |
| **展示导出** | 系统管理 | 成长体系 |
| **Whitelist豁免** | 系统管理 | 治理中心 |
| **角色委任** | 系统管理 | 治理中心 |
| **委任事务** | 系统管理 | 治理中心 |
| **委任审计** | 系统管理 | 治理中心 |
| **委任图谱** | 系统管理 | 治理中心 |
| **Sysop策略** | 系统管理 | 治理中心 |
| **治理审计** | 系统管理 | 治理中心 |
| **邀请树管理** | 运维中心 | 推广管理 |
| **系统邀请码** | 运维中心 | 推广管理 |
| **推广管理** | 运维中心 | 推广管理 |
| **推广设置** | 运维中心 | 推广管理 |
| **订单管理** | 运维中心 | 客服中心 |
| **工单管理** | 运维中心 | 客服中心 |
| **售后管理** | 运维中心 | 客服中心 |
| **商城管理** | 运维中心 | 商城中心 |
| **商店订单** | 运维中心 | 商城中心 |
| **CDKEY管理** | 运维中心 | 商城中心 |
| **续期卡管理** | 运维中心 | 商城中心 |
| **系统资源派发** | 运维中心 | 商城中心 |
| **促销/优惠码** | 运维中心 | 商城中心 |
| **支付服务** | 系统管理 | 财务中心 |
| **汇率设置** | 系统管理 | 财务中心 |
| **支付对账** | 运维中心 | 财务中心 |
| **退款管理** | 运维中心 | 财务中心 |
| **结算报表** | 运维中心 | 财务中心 |
| **媒体账号** | 运维中心 | 媒体管理 |
| **漫游中台** | 运维中心 | 媒体管理 |
| **Slave状态** | 系统管理 | 媒体管理 |
| **运营面板** | 运维中心 | 运营监控 |
| **运营分析** | 运维中心 | 运营监控 |
| **MiniApp设置** | 运维中心 | 运营监控 |
| **Telegram Bots** | 运维中心 | 运营监控 |
| **签到风控** | 运维中心 | 运营监控 |
| **签到活动管理** | 运维中心 | 运营监控 |
| **风控管理** | 运维中心 | 运营监控 |
| **策略规则** | 运维中心 | 运营监控 |
| **系统日志** | 系统管理 | 系统监控 |
| **运行日志** | 系统管理 | 系统监控 |
| **系统诊断** | 系统管理 | 系统监控 |
| **系统指标** | 系统管理 | 系统监控 |
| **App Token** | 系统管理 | 系统监控 |
| **策略管理** | 系统管理 | 系统监控 |
| **审计事件** | 审计中心 | 审计中心 (unchanged) |
| **白名单审计** | 运维中心 | 审计中心 |
| **内容管理** | CMS中心 | CMS中心 (unchanged) |
| **公告管理** | CMS中心 | CMS中心 (unchanged) |

---

## Summary of Changes

| Category | Before | After |
|----------|--------|-------|
| 概览 | 2 items | 2 items (unchanged) |
| 用户中心 | 15 items | 15 items (unchanged) |
| 设置中心 | 7 items | 7 items (unchanged) |
| 系统管理 | **28 items** | REMOVED (split into multiple) |
| 运维中心 | **28 items** | REMOVED (split into multiple) |
| CMS中心 | 2 items | 2 items (unchanged) |
| 审计中心 | 1 item | 2 items |
| 账号权限 | - | **6 items (NEW)** |
| 成长体系 | - | **7 items (NEW)** |
| 治理中心 | - | **7 items (NEW)** |
| 推广管理 | - | **4 items (NEW)** |
| 客服中心 | - | **3 items (NEW)** |
| 商城中心 | - | **6 items (NEW)** |
| 财务中心 | - | **5 items (NEW)** |
| 媒体管理 | - | **3 items (NEW)** |
| 运营监控 | - | **8 items (NEW)** |
| 系统监控 | - | **6 items (NEW)** |

**Key improvements:**
1. No more 28-item mega-categories
2. Each category now has 2-8 items
3. Items grouped by function, not arbitrary
4. Removed duplicate entries (权限模板)
5. Clear separation of concerns
