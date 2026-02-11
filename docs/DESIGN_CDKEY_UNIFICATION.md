# CDKEY 系统统一化开发文档

> Round-CDKEY-UNIFY | 版本 1.0 | 2026-02-07

---

## 目录

1. [现状分析](#1-现状分析)
2. [目标架构](#2-目标架构)
3. [Phase 1: Admin 页面合并](#3-phase-1-admin-页面合并)
4. [Phase 2: Portal 页面合并](#4-phase-2-portal-页面合并)
5. [Phase 3: Portal UI 增强](#5-phase-3-portal-ui-增强)
6. [文件变更总清单](#6-文件变更总清单)
7. [向后兼容与风险](#7-向后兼容与风险)

---

## 1. 现状分析

### 1.1 数据模型现状

系统中存在 **3 套独立的数据模型** 来处理不同类型的用户资产：

| 模型 | 表名 | 主要用途 | 状态枚举 |
|------|------|----------|----------|
| `CDKey` | `cdkeys` | 统一 CDKEY（9 种类型） | available/owned/used/expired/revoked |
| `InvitationDB` | `invitationDB` | 邀请码（用户创建/购买） | pending/used/expired/disabled |
| `RenewalCard` | `renewal_cards` | 续期卡（余额购买/应用） | unused/used/expired |

**CDKey 类型枚举 (`CDKeyType`)**:
```
seat / renewal / invite / bundle / pretty_id / pretty_renewal / username_style / style_renewal
```

**CDKey 发行来源 (`CDKeyIssuedBy`)**:
```
system / admin / purchase / dispatch
```

### 1.2 Admin 页面现状

运维中心 (`/admin/devops`) 下有 3 个独立页面：

| 页面 | 路由 | 权限 | 功能 |
|------|------|------|------|
| 系统级CDKEY创建 | `system-cdkey-create` | `system_cdkey_create` | 3步向导：选类型→配置→数量，生成 AVAILABLE 状态的 CDKEY |
| 系统资源派发 | `system-dispatch` | `system_cdkey_dispatch` | 3步向导：选类型→配置→派发方式（用户/角色/等级），生成 OWNED 状态 |
| 系统级CDKEY追踪 | `system-cdkey-tracking` | `system_cdkey_create` | 查询/统计/撤销已创建的 CDKEY |

**问题**：创建和派发的 Step 1（类型选择）和 Step 2（配置）完全相同，代码重复。且创建后的 CDKEY 无法直接从创建页面派发给用户。

### 1.3 Portal 页面现状

用户中心有 3 个独立页面：

| 页面 | 路由 | 菜单名 | 数据源 | API 前缀 |
|------|------|--------|--------|----------|
| 邀请码管理 | `/portal/invite-codes` | InviteCodes | `InvitationDB` | `/user/me/invite-codes` |
| 续期卡 | `/portal/renewal-cards` | UserRenewalCards | `RenewalCard` | `/user/credits/renewal-cards` |
| CDKEY 管理 | `/portal/cdkeys` | UserCDKeys | `CDKey` | `/user/cdkeys/*` |

**问题**：用户需要在 3 个不同页面管理本质相似的资产。CDKey 模型已能覆盖 invite 和 renewal 类型，但旧模型仍在使用。

### 1.4 Miniapp 现状

| 页面 | 状态 | 数据源 |
|------|------|--------|
| cdkeys | 已实现（显示 owned CDKEYs，支持兑换） | `CDKey` |
| invite | 已实现（显示邀请码列表） | `InvitationDB` |
| renewal-cards | 占位页（"请前往网页版"） | - |

### 1.5 权限现状

**ALL_PERMISSIONS 中 CDKEY 相关权限** (`app/routers/menu.py:37-147`):

| 权限 key | 用途 | 位置 |
|----------|------|------|
| `manage_cdkeys` | 通用 CDKEY 管理 | ALL_PERMISSIONS |
| `view_cdkeys` | 查看 CDKEY 库存 | ALL_PERMISSIONS |
| `redeem_cdkey` | 兑换 CDKEY | ALL_PERMISSIONS |
| `manage_system_dispatch` | 系统资源派发 | ALL_PERMISSIONS |
| `system_cdkey_create` | 系统级 CDKEY 创建 | roles.py (非 ALL_PERMISSIONS) |
| `system_cdkey_dispatch` | 系统资源派发 | roles.py (非 ALL_PERMISSIONS) |
| `manage_renewal_cards` | 续期卡管理 | ALL_PERMISSIONS |
| `create_invite_codes` | 创建邀请码 | ALL_PERMISSIONS |
| `view_invite_codes` | 查看邀请码 | ALL_PERMISSIONS |
| `system_invite_manage` | 系统邀请管理 | ALL_PERMISSIONS |

**注意**: `system_cdkey_create` 和 `system_cdkey_dispatch` 在 `roles.py` 的 ADMIN_PERMISSIONS 中定义，但**不在** `menu.py` 的 ALL_PERMISSIONS 列表中。

### 1.6 API 端点现状

**Admin 端点** (`/api/v1/admin/cdkeys`):

| 端点 | 方法 | 权限 | 说明 |
|------|------|------|------|
| `/stats` | GET | - | CDKEY 统计 |
| `/list` | GET | - | CDKEY 列表（分页/筛选） |
| `/create` | POST | `system_cdkey_create` | 创建 CDKEY（≥10需确认令牌） |
| `/revoke/{id}` | POST | - | 撤销 CDKEY |
| `/{id}` | GET | - | CDKEY 详情 |
| `/dispatch/users` | POST | `system_cdkey_dispatch` | 按用户派发 |
| `/dispatch/role` | POST | `system_cdkey_dispatch` | 按角色派发（需确认令牌） |
| `/dispatch/level` | POST | `system_cdkey_dispatch` | 按等级派发（需确认令牌） |

**User 端点** (`/api/v1/user-cdkeys`, `/api/v1/user`):

| 端点 | 方法 | 说明 |
|------|------|------|
| `/user/cdkeys/list` | GET | 用户 CDKEY 列表 |
| `/user/cdkeys/stats` | GET | 用户 CDKEY 统计 |
| `/user/cdkeys/redeem` | POST | 兑换已拥有的 CDKEY |
| `/user/cdkeys/redeem-code` | POST | 通过码直接兑换 |
| `/user/cdkeys/{id}` | GET | CDKEY 详情 |
| `/user/cdkeys/entitlements/list` | GET | 权益列表 |
| `/user/cdkeys/entitlements/apply` | POST | 应用权益 |
| `/user/me/invite-codes` | GET | 邀请码列表 |
| `/user/me/invite-code` | POST | 创建邀请码 |
| `/user/invite-settings` | GET | 邀请码设置 |
| `/user/credits/renewal-cards` | GET | 续期卡列表 |
| `/user/credits/store-info` | GET | 商店信息 |
| `/user/credits/purchase-renewal-card` | POST | 购买续期卡 |
| `/user/credits/apply-renewal-card` | POST | 应用续期卡 |

---

## 2. 目标架构

### 2.1 核心思路

**不做数据模型迁移**（风险太大），而是在**展示层**做统一：

1. **Admin**: 将"系统级CDKEY创建"和"系统资源派发"合并为一个页面，支持"创建池"和"派发"两种模式
2. **Portal**: 将 3 个页面合并为一个统一的"我的资产"页面，聚合显示所有类型
3. **Miniapp**: 对应更新聚合显示

### 2.2 实施阶段

| Phase | 内容 | 复杂度 | 影响范围 |
|-------|------|--------|----------|
| **Phase 1** | Admin 创建+派发页面合并 | 中 | Admin Vue (2→1 页面) + 新 API |
| **Phase 2** | Portal 3 页面合并为统一视图 | 中 | Portal Vue (3→1 页面) + 聚合 API |
| **Phase 3** | Portal/Miniapp UI 增强（卡片视图+筛选） | 低 | Portal/Miniapp Vue |

---

## 3. Phase 1: Admin 页面合并

### 3.1 目标

将 `system-cdkey-create` 和 `system-dispatch` 合并为 `system-cdkey-manage`，包含：
- **Tab 1: 创建 & 派发**（合并后的3步向导）
- **Tab 2: CDKEY 池**（查看 AVAILABLE 状态的 CDKEY，支持后续派发）
- **Tab 3: 追踪**（从 `system-cdkey-tracking` 迁移）

### 3.2 数据库变更

**无新表、无新字段**。现有 `cdkeys` 表已完全满足需求。

### 3.3 新增 API 端点

#### `POST /api/v1/admin/cdkeys/dispatch/from-pool`

从已创建的 AVAILABLE 状态 CDKEY 池中派发给用户。

```python
# app/routers/admin/cdkeys.py

class DispatchFromPoolRequest(BaseModel):
    cdkey_ids: list[int]           # 要派发的 CDKEY ID 列表
    user_ids: list[int]            # 目标用户 ID 列表
    # 分配策略：
    # - "one_each": 每个用户分配一个 CDKEY（要求 len(cdkey_ids) >= len(user_ids)）
    # - "all_to_all": 每个用户都收到所有 CDKEY（复制模式，为每个用户创建新 CDKEY）
    strategy: str = "one_each"

# Response:
{
    "code": 0,
    "data": {
        "dispatched": 5,
        "failed": 0,
        "details": [
            {"user_id": 123, "cdkey_id": 456, "success": true},
            ...
        ]
    }
}
```

**权限**: `system_cdkey_dispatch`
**确认令牌**: 当 `len(cdkey_ids) >= 10` 或 `len(user_ids) >= 10` 时需要

#### 后端实现

```python
# app/services/cdkey_service.py — 新增方法

@staticmethod
async def dispatch_from_pool(
    cdkey_ids: list[int],
    user_ids: list[int],
    strategy: str,
    dispatched_by_user_id: int,
) -> dict:
    """从 CDKEY 池中派发已创建的 CDKEY 给用户"""
    results = []

    if strategy == "one_each":
        # 1:1 分配
        cdkeys = await CDKey.filter(
            id__in=cdkey_ids, status=CDKeyStatus.AVAILABLE
        ).all()

        if len(cdkeys) < len(user_ids):
            raise ValueError(
                f"Not enough available CDKEYs: {len(cdkeys)} available, "
                f"{len(user_ids)} users"
            )

        for i, user_id in enumerate(user_ids):
            cdkey = cdkeys[i]
            cdkey.owner_id = user_id
            cdkey.status = CDKeyStatus.OWNED
            cdkey.owned_at = datetime.now(timezone.utc)
            cdkey.issued_by = CDKeyIssuedBy.DISPATCH
            cdkey.issued_by_user_id = dispatched_by_user_id
            await cdkey.save()
            results.append({
                "user_id": user_id,
                "cdkey_id": cdkey.id,
                "code": cdkey.code,
                "success": True,
            })
    # strategy == "all_to_all" 暂不实现

    return {
        "dispatched": sum(1 for r in results if r["success"]),
        "failed": sum(1 for r in results if not r["success"]),
        "details": results,
    }
```

### 3.4 权限变更

#### 合并权限

新增一个统一权限 `manage_system_cdkeys`，替代 `system_cdkey_create` + `system_cdkey_dispatch`：

**但为了向后兼容，保留旧权限**，新页面检查 `system_cdkey_create OR system_cdkey_dispatch`。

无需改动 `ALL_PERMISSIONS`。

### 3.5 菜单变更

#### `app/routers/menu.py` 修改

```python
# 替换 DevOps Center children 中的 3 个条目为 1 个：
{
    "path": "system-cdkey-manage",
    "name": "SystemCDKeyManage",
    "component": "/admin/devops/system-cdkey-manage/index",
    "meta": {
        "icon": "lucide:key-round",
        "title": "CDKEY 管理中心",
    },
    "required_permission": "system_cdkey_create",  # 有创建或派发权限即可见
},
```

**保留旧路由重定向**（向后兼容书签）：
```python
# 旧路由 system-cdkey-create → 重定向到 system-cdkey-manage
# 旧路由 system-dispatch → 重定向到 system-cdkey-manage
# 旧路由 system-cdkey-tracking → 重定向到 system-cdkey-manage?tab=tracking
```

### 3.6 前端路由变更

#### `vue-vben-admin/.../admin.example.com` 修改

```typescript
// DevOps Center children:
{
  path: 'system-cdkey-manage',
  name: 'SystemCDKeyManage',
  component: () => import('#/views/admin/devops/system-cdkey-manage/index.vue'),
  meta: { icon: 'lucide:key-round', title: 'CDKEY 管理中心' },
},
// 旧路由重定向
{
  path: 'system-cdkey-create',
  redirect: '/admin/devops/system-cdkey-manage',
},
{
  path: 'system-dispatch',
  redirect: '/admin/devops/system-cdkey-manage',
},
{
  path: 'system-cdkey-tracking',
  redirect: '/admin/devops/system-cdkey-manage?tab=tracking',
},
```

### 3.7 前端组件设计

#### 新建 `system-cdkey-manage/index.vue`

```
┌─────────────────────────────────────────────────────────┐
│ CDKEY 管理中心                                          │
├─────────────────────────────────────────────────────────┤
│ [创建 & 派发]  [CDKEY 池]  [追踪]                       │
├─────────────────────────────────────────────────────────┤
│                                                         │
│ Tab 1: 创建 & 派发                                      │
│ ┌───────────────────────────────────────────────────┐   │
│ │ Step 1: 选择类型 (9种类型卡片网格)                 │   │
│ │ Step 2: 配置参数 (根据类型动态表单)                │   │
│ │ Step 3: 操作选择                                   │   │
│ │   ┌──────────────┐  ┌──────────────┐              │   │
│ │   │ 🔑 创建到池  │  │ 📤 直接派发  │              │   │
│ │   │ (AVAILABLE)  │  │ (OWNED)      │              │   │
│ │   └──────────────┘  └──────────────┘              │   │
│ │                                                    │   │
│ │   [创建到池] 模式:                                 │   │
│ │   - 数量输入 (1-1000)                             │   │
│ │   - CDKEY 有效期（可选）                           │   │
│ │   - 创建按钮                                       │   │
│ │   - 结果: 显示生成的 code 列表，支持复制/CSV导出   │   │
│ │                                                    │   │
│ │   [直接派发] 模式:                                 │   │
│ │   - 派发方式: 按用户/按角色/按等级                 │   │
│ │   - CDKEY 有效期（可选）                           │   │
│ │   - 执行派发按钮                                   │   │
│ │   - 结果: 成功/失败统计 + 详情表                   │   │
│ └───────────────────────────────────────────────────┘   │
│                                                         │
│ Tab 2: CDKEY 池                                         │
│ ┌───────────────────────────────────────────────────┐   │
│ │ 筛选: [类型▼] [状态▼] [搜索code...]               │   │
│ │                                                    │   │
│ │ ☐ | ID | Code | 类型 | 状态 | 创建时间 | 操作     │   │
│ │ ☐ | 1  | CK.. | 席位 | 可用 | 2026-... | [详情]  │   │
│ │ ☐ | 2  | CK.. | 续期 | 可用 | 2026-... | [详情]  │   │
│ │                                                    │   │
│ │ 已选 N 个  [批量派发给用户▼] [批量撤销] [导出CSV]  │   │
│ └───────────────────────────────────────────────────┘   │
│                                                         │
│ Tab 3: 追踪 (从 system-cdkey-tracking 迁移)             │
│ ┌───────────────────────────────────────────────────┐   │
│ │ 统计卡片 (6x): 总创建/未用/已分配/已兑换/过期/撤销│   │
│ │ 筛选: [类型▼] [状态▼] [发行来源▼] [搜索...]       │   │
│ │ 表格: ID|Code|类型|状态|来源|拥有者|创建|使用|操作 │   │
│ │ CDKEY 详情抽屉                                     │   │
│ └───────────────────────────────────────────────────┘   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

#### 实现要点

1. **Step 1 & Step 2**: 从 `system-cdkey-create/index.vue` 和 `system-dispatch/index.vue` 提取公共逻辑（类型列表、配置表单），两者 Step 1/2 代码完全相同
2. **Step 3**: 新增操作选择面板，分"创建到池"和"直接派发"两个选项卡
3. **CDKEY 池 Tab**: 新增，复用 `/admin/cdkeys/list?status=available` 接口 + 新增批量派发功能
4. **追踪 Tab**: 迁移 `system-cdkey-tracking/index.vue` 代码

### 3.8 Phase 1 文件变更清单

| 操作 | 文件 | 说明 |
|------|------|------|
| 新建 | `vue-vben-admin/.../views/admin/devops/system-cdkey-manage/index.vue` | 合并后的页面 |
| 改造 | `vue-vben-admin/.../router/routes/modules/admin.example.com` | 路由：新增 + 旧重定向 |
| 改造 | `app/routers/menu.py` | 菜单：合并3→1条目 |
| 改造 | `app/routers/admin/cdkeys.py` | 新增 `dispatch/from-pool` 端点 |
| 改造 | `app/services/cdkey_service.py` | 新增 `dispatch_from_pool()` 方法 |
| 保留 | `vue-vben-admin/.../views/admin/devops/system-cdkey-create/index.vue` | 暂保留，待新页面稳定后删除 |
| 保留 | `vue-vben-admin/.../views/admin/devops/system-dispatch/index.vue` | 暂保留，待新页面稳定后删除 |
| 保留 | `vue-vben-admin/.../views/admin/devops/system-cdkey-tracking/index.vue` | 暂保留 |

---

## 4. Phase 2: Portal 页面合并

### 4.1 目标

将"邀请码管理"、"续期卡"、"CDKEY管理"3个页面合并为统一的"我的资产"页面，聚合显示所有类型的用户资产。

### 4.2 新增聚合 API

#### `GET /api/v1/user/assets/summary`

聚合 3 个数据源的统计信息。

```python
# app/routers/user_assets.py (新建)

@router.get("/assets/summary")
async def get_assets_summary(user: Users = Depends(get_current_user)):
    """聚合用户所有资产统计"""
    # 1. CDKey 统计
    cdkey_stats = await CDKey.filter(owner_id=user.id).annotate(
        count=Count("id")
    ).group_by("status").values("status", "count")

    # 2. InvitationDB 统计
    invite_stats = await InvitationDB.filter(inviter_id=user.id).annotate(
        count=Count("id")
    ).group_by("status").values("status", "count")

    # 3. RenewalCard 统计
    renewal_stats = await RenewalCard.filter(user_id=user.id).annotate(
        count=Count("id")
    ).group_by("status").values("status", "count")

    return success_response({
        "cdkeys": {
            "owned": ..., "used": ..., "total": ...,
            "by_type": { "seat": ..., "renewal": ..., ... }
        },
        "invite_codes": {
            "active": ..., "used": ..., "expired": ..., "total": ...
        },
        "renewal_cards": {
            "unused": ..., "used": ..., "expired": ..., "total": ...
        },
        "total_assets": ...  # 所有可用资产数
    })
```

#### `GET /api/v1/user/assets/list`

统一列表接口，支持跨模型分页。

```python
@router.get("/assets/list")
async def list_user_assets(
    asset_type: Optional[str] = None,  # "cdkey" | "invite" | "renewal" | None(全部)
    status: Optional[str] = None,      # 按状态筛选
    skip: int = 0,
    limit: int = 20,
    user: Users = Depends(get_current_user),
):
    """统一资产列表"""
    items = []

    if asset_type in (None, "cdkey"):
        cdkeys = await CDKey.filter(owner_id=user.id, ...)
        items.extend([{
            "source": "cdkey",
            "id": ck.id,
            "type": ck.cdkey_type,
            "code": ck.code,
            "status": ck.status,
            "config": ck.config,
            "created_at": ck.created_at,
            "expires_at": ck.expires_at,
            "is_actionable": ck.is_redeemable,
            "action_label": "兑换",
            "action_endpoint": "/user/cdkeys/redeem",
        } for ck in cdkeys])

    if asset_type in (None, "invite"):
        invites = await InvitationDB.filter(inviter_id=user.id, ...)
        items.extend([{
            "source": "invite",
            "id": inv.id,
            "type": "invite",
            "code": inv.invite_token,
            "status": inv.status,
            "config": {"seat_grant_qty": inv.seat_grant_qty},
            "created_at": inv.created_at,
            "expires_at": inv.expires_at,
            "is_actionable": False,  # 邀请码无兑换操作
            "action_label": None,
            "action_endpoint": None,
        } for inv in invites])

    if asset_type in (None, "renewal"):
        cards = await RenewalCard.filter(user_id=user.id, ...)
        items.extend([{
            "source": "renewal",
            "id": card.id,
            "type": "renewal",
            "code": card.cdkey or f"RC-{card.id}",
            "status": card.status,
            "config": {"days": card.days, "scope": card.scope},
            "created_at": card.created_at,
            "expires_at": card.expires_at,
            "is_actionable": card.status == "unused",
            "action_label": "应用",
            "action_endpoint": "/user/credits/apply-renewal-card",
        } for card in cards])

    # 按 created_at 倒序排列
    items.sort(key=lambda x: x["created_at"], reverse=True)

    return success_response({
        "items": items[skip:skip+limit],
        "total": len(items),
    })
```

### 4.3 菜单变更

#### `app/routers/menu.py` — Portal 菜单

现有 UserCenter children 中包含 InviteCodes、UserRenewalCards、UserCDKeys 三个菜单项。

**方案**: 保留 3 个菜单项指向同一页面的不同 tab（通过 query 参数区分），或合并为 1 个菜单项：

```python
# 方案 A: 合并为单一菜单（推荐）
{
    "name": "UserAssets",
    "path": "/portal/assets",
    "meta": {
        "icon": "lucide:package",
        "title": "我的资产",
    },
    "required_permission": "view_cdkeys",
},

# 方案 B: 保留3个菜单但指向同一页面不同tab
# InviteCodes → /portal/assets?tab=invite
# UserRenewalCards → /portal/assets?tab=renewal
# UserCDKeys → /portal/assets?tab=cdkey
```

**推荐方案 A**，更简洁。旧路由保留重定向。

#### `nuxt-portal/composables/usePortalMenu.ts` 修改

```typescript
// 新增
UserAssets: { label: '我的资产', to: '/portal/assets', icon: 'i-heroicons-cube' },

// 旧映射保留重定向
InviteCodes: { label: '邀请码', to: '/portal/assets?tab=invite', icon: 'i-heroicons-ticket' },
UserRenewalCards: { label: '续期卡', to: '/portal/assets?tab=renewal', icon: 'i-heroicons-credit-card' },
UserCDKeys: { label: 'CDKeys', to: '/portal/assets?tab=cdkey', icon: 'i-heroicons-key' },
```

### 4.4 前端组件设计

#### 新建 `nuxt-portal/pages/portal/assets.vue`

```
┌─────────────────────────────────────────────────────────┐
│ 我的资产                              [兑换码输入] [🔍]  │
├─────────────────────────────────────────────────────────┤
│                                                         │
│ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐                    │
│ │ 待用 │ │ 已用 │ │ 席位 │ │ 续期 │  ← 统计卡片        │
│ │  12  │ │  8   │ │  5   │ │  3   │                    │
│ └──────┘ └──────┘ └──────┘ └──────┘                    │
│                                                         │
│ ┌─────────────────────────────────────────────────────┐ │
│ │ [全部] [CDKEY] [邀请码] [续期卡]   [状态▼] [排序▼]  │ │
│ ├─────────────────────────────────────────────────────┤ │
│ │                                                     │ │
│ │ ┌───────────────────┐ ┌───────────────────┐         │ │
│ │ │ 🔑 席位 CDKEY     │ │ 📨 邀请码         │         │ │
│ │ │ CK-A1B2C3...      │ │ abc123def...      │         │ │
│ │ │ 状态: 待兑换      │ │ 状态: 可用        │         │ │
│ │ │ 配置: 30天        │ │ 赠送席位: 1       │         │ │
│ │ │ [兑换]            │ │ [复制链接]        │         │ │
│ │ └───────────────────┘ └───────────────────┘         │ │
│ │                                                     │ │
│ │ ┌───────────────────┐ ┌───────────────────┐         │ │
│ │ │ 🔄 续期卡         │ │ 🎁 套餐 CDKEY     │         │ │
│ │ │ RC-42             │ │ CK-X9Y8Z7...      │         │ │
│ │ │ 状态: 未使用      │ │ 状态: 待兑换      │         │ │
│ │ │ 续期: 30天        │ │ 内含: 席位x1...   │         │ │
│ │ │ [应用到账号]      │ │ [兑换]            │         │ │
│ │ └───────────────────┘ └───────────────────┘         │ │
│ │                                                     │ │
│ └─────────────────────────────────────────────────────┘ │
│                                                         │
│ ← 1 / 3 →                                              │
└─────────────────────────────────────────────────────────┘
```

### 4.5 Miniapp 更新

#### miniapp.vue 修改

将现有的 `cdkeys` 页面扩展为聚合显示：

```typescript
// MiniAppPage 类型新增 'assets'
// 或复用 'cdkeys' 页面名，在内部聚合显示

// 数据加载：
async function loadUserAssets() {
  // 并行加载 3 个数据源
  const [cdkeys, invites, renewals] = await Promise.all([
    fetchWithTimeout(`${getApiBase()}/user/cdkeys/list?status=owned`, ...),
    fetchWithTimeout(`${getApiBase()}/user/me/invite-codes`, ...),
    // renewal-cards 目前是 stub，可先跳过
  ])
  // 合并显示
}
```

**Miniapp 变更较小**：主要是在 `cdkeys` 页面底部增加"邀请码"区块。`renewal-cards` 页面从 stub 改为显示实际数据或指向聚合页。

### 4.6 Phase 2 文件变更清单

| 操作 | 文件 | 说明 |
|------|------|------|
| 新建 | `app/routers/user_assets.py` | 聚合 API 路由 |
| 新建 | `nuxt-portal/pages/portal/assets.vue` | 统一资产页面 |
| 改造 | `app/routers/menu.py` | Portal 菜单合并 |
| 改造 | `nuxt-portal/composables/usePortalMenu.ts` | 路由映射更新 |
| 改造 | `nuxt-portal/pages/miniapp.vue` | Miniapp cdkeys 页面聚合 |
| 改造 | `app/main.py` | 注册新路由 |
| 保留 | `nuxt-portal/pages/portal/cdkeys.vue` | 保留旧页面，添加重定向 |
| 保留 | `nuxt-portal/pages/portal/invite-codes.vue` | 保留旧页面，添加重定向 |
| 保留 | `nuxt-portal/pages/portal/renewal-cards.vue` | 保留旧页面，添加重定向 |

---

## 5. Phase 3: Portal UI 增强

### 5.1 目标

在 Phase 2 的统一页面基础上增加：
- **Filter Chips**: 类型筛选改为可视化标签切换
- **Card Grid View**: 3:4 网格卡片视图（可切换列表/网格）
- **移动端优化**: 响应式卡片布局

### 5.2 Filter Chips 设计

替换下拉筛选为水平标签组：

```html
<div class="flex flex-wrap gap-2">
  <button
    v-for="chip in typeChips"
    :key="chip.key"
    :class="[
      'px-3 py-1.5 rounded-full text-sm font-medium transition-colors',
      activeType === chip.key
        ? 'bg-blue-500 text-white'
        : 'bg-gray-100 text-gray-600 hover:bg-gray-200'
    ]"
    @click="activeType = chip.key"
  >
    {{ chip.icon }} {{ chip.label }}
    <span v-if="chip.count" class="ml-1 opacity-70">({{ chip.count }})</span>
  </button>
</div>
```

**Chips 列表**:
```typescript
const typeChips = [
  { key: 'all', label: '全部', icon: '📦', count: totalCount },
  { key: 'cdkey', label: 'CDKEY', icon: '🔑', count: cdkeyCount },
  { key: 'invite', label: '邀请码', icon: '📨', count: inviteCount },
  { key: 'renewal', label: '续期卡', icon: '🔄', count: renewalCount },
]
```

### 5.3 Card Grid View 设计

双模式切换（列表 / 网格）：

```html
<!-- 视图切换按钮 -->
<div class="flex gap-1">
  <button @click="viewMode = 'list'" :class="viewMode === 'list' ? 'active' : ''">
    ☰ 列表
  </button>
  <button @click="viewMode = 'grid'" :class="viewMode === 'grid' ? 'active' : ''">
    ⊞ 网格
  </button>
</div>

<!-- 网格视图 -->
<div v-if="viewMode === 'grid'" class="grid grid-cols-2 sm:grid-cols-3 lg:grid-cols-4 gap-4">
  <div
    v-for="item in filteredItems"
    :key="`${item.source}-${item.id}`"
    class="aspect-[3/4] rounded-xl border p-4 flex flex-col justify-between"
  >
    <!-- 顶部：类型图标 + 标签 -->
    <div class="text-center">
      <div class="text-3xl mb-2">{{ typeIcon(item.type) }}</div>
      <div class="text-sm font-medium">{{ typeLabel(item.type) }}</div>
    </div>

    <!-- 中部：码 + 配置 -->
    <div class="text-center">
      <div class="font-mono text-xs text-gray-500 truncate">{{ item.code }}</div>
      <div v-if="item.config.days" class="text-sm text-gray-600 mt-1">
        {{ item.config.days }}天
      </div>
    </div>

    <!-- 底部：状态 + 操作 -->
    <div class="text-center">
      <span :class="statusBadgeClass(item.status)" class="text-xs px-2 py-0.5 rounded-full">
        {{ statusLabel(item.status) }}
      </span>
      <button
        v-if="item.is_actionable"
        class="mt-2 w-full bg-blue-500 text-white text-sm py-1.5 rounded-lg"
        @click="handleAction(item)"
      >
        {{ item.action_label }}
      </button>
    </div>
  </div>
</div>
```

### 5.4 Phase 3 文件变更清单

| 操作 | 文件 | 说明 |
|------|------|------|
| 改造 | `nuxt-portal/pages/portal/assets.vue` | 添加 filter chips + grid view |
| 改造 | `nuxt-portal/pages/miniapp.vue` | Miniapp 卡片网格适配 |

---

## 6. 文件变更总清单

### Phase 1 (Admin 页面合并)

| # | 操作 | 文件 |
|---|------|------|
| 1 | 新建 | `vue-vben-admin/apps/web-antd/src/views/admin/devops/system-cdkey-manage/index.vue` |
| 2 | 改造 | `vue-vben-admin/apps/web-antd/src/router/routes/modules/admin.example.com` |
| 3 | 改造 | `app/routers/menu.py` (DevOps 菜单) |
| 4 | 改造 | `app/routers/admin/cdkeys.py` (新端点) |
| 5 | 改造 | `app/services/cdkey_service.py` (新方法) |

### Phase 2 (Portal 页面合并)

| # | 操作 | 文件 |
|---|------|------|
| 6 | 新建 | `app/routers/user_assets.py` |
| 7 | 新建 | `nuxt-portal/pages/portal/assets.vue` |
| 8 | 改造 | `app/routers/menu.py` (Portal 菜单) |
| 9 | 改造 | `nuxt-portal/composables/usePortalMenu.ts` |
| 10 | 改造 | `nuxt-portal/pages/miniapp.vue` |
| 11 | 改造 | `app/main.py` (注册路由) |

### Phase 3 (UI 增强)

| # | 操作 | 文件 |
|---|------|------|
| 12 | 改造 | `nuxt-portal/pages/portal/assets.vue` (filter chips + grid) |
| 13 | 改造 | `nuxt-portal/pages/miniapp.vue` (卡片网格) |

### 总计: 5 新建 + 8 改造 = 13 文件

---

## 7. 向后兼容与风险

### 7.1 数据兼容

- **不迁移数据**：InvitationDB 和 RenewalCard 表保留，旧 API 继续工作
- **聚合查询**：新 API 同时查 3 个表，合并结果
- **旧 API 保留**：所有现有端点不删除，确保旧客户端兼容

### 7.2 路由兼容

- **Admin 旧路由**: `system-cdkey-create`、`system-dispatch`、`system-cdkey-tracking` 通过 redirect 指向新页面
- **Portal 旧路由**: `invite-codes`、`renewal-cards`、`cdkeys` 保留页面文件，添加跳转提示或直接重定向
- **书签兼容**: 所有旧 URL 均可正常访问

### 7.3 权限兼容

- **不新增权限**: 新页面复用现有权限 key
- **Admin**: `system_cdkey_create` OR `system_cdkey_dispatch` 可访问合并页
- **Portal**: `view_cdkeys` 可访问统一资产页

### 7.4 风险点

| 风险 | 概率 | 影响 | 缓解 |
|------|------|------|------|
| 聚合查询性能 | 中 | 低 | 3 表并行查询，加缓存 |
| 旧页面用户找不到入口 | 低 | 中 | 旧路由重定向 + 过渡期保留旧菜单 |
| Miniapp 兼容 | 低 | 低 | Miniapp 改动最小，仅增加聚合区 |
| 新 dispatch-from-pool API 安全 | 低 | 高 | 复用现有权限 + 确认令牌机制 |

### 7.5 验证清单

**Phase 1 验证**:
- [ ] Admin 新页面加载正常，3 个 Tab 均可用
- [ ] 创建 CDKEY 到池 → 在池 Tab 可见
- [ ] 从池批量派发给用户 → 用户收到 OWNED CDKEY
- [ ] 直接派发（用户/角色/等级）功能正常
- [ ] 旧路由重定向正常
- [ ] 权限控制正确（无权限用户看不到菜单）

**Phase 2 验证**:
- [ ] 统一资产页加载正常，聚合 3 类数据
- [ ] Tab 切换和筛选正常
- [ ] CDKEY 兑换、续期卡应用、邀请码复制均正常
- [ ] 旧页面路由重定向正常
- [ ] Miniapp 聚合显示正常

**Phase 3 验证**:
- [ ] Filter chips 切换正常
- [ ] 列表/网格视图切换正常
- [ ] 网格卡片 3:4 比例显示正常
- [ ] 移动端响应式布局正常
