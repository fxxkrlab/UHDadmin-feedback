# API Permission Matrix

> BUGFIX-099: Comprehensive API authentication and authorization audit

## Authentication Types

| Type | Header Format | Description |
|------|---------------|-------------|
| **admin** | `Authorization: Bearer <jwt>` | Admin JWT (which_role != 'user') |
| **user** | `Authorization: Bearer <jwt>` | User JWT (any level) |
| **app** | `Authorization: App <token>` | App API token |
| **public** | None | No authentication required |

## Route Coverage by Router

### Auth Routes (`/api/v1/auth`)
| Method | Path | Auth | Notes |
|--------|------|------|-------|
| POST | /login | public | Login endpoint |
| POST | /register | public | Registration |
| POST | /logout | user | Logout current user |
| POST | /telegram/miniapp/login | public | MiniApp password login + TG auto-bind (Round-8) |
| POST | /telegram/miniapp/restricted-signup | public | MiniApp restricted signup seat=0 (Round-8) |

### User Routes (`/api/v1/user`)
| Method | Path | Auth | Notes |
|--------|------|------|-------|
| GET | /me | user | Current user info |
| POST | /me/invite-code | user | Create invite code |
| GET | /me/invite-codes | user | List user's invite codes |

### User Bind Routes (`/api/v1/user/bind`) - ROUNDREQ P0-4
| Method | Path | Auth | Notes |
|--------|------|------|-------|
| GET | /status | user | Get bind status (telegram/email) |
| GET | /telegram-config | public | Telegram Login Widget config |
| POST | /telegram | user | Bind Telegram via Login Widget |
| POST | /email | user | Bind email address |

### Admin User Routes (`/api/v1/admin/users`)
| Method | Path | Auth | Notes |
|--------|------|------|-------|
| GET | / | admin | List users |
| GET | /{id} | admin | Get user detail |
| GET | /{id}/full | admin | Get user full detail with relations |
| POST | / | admin | Create user |
| PUT | /{id} | admin | Update user |
| DELETE | /{id} | admin | Delete user |
| POST | /{id}/credits/deposit | admin | Add credits to user |
| POST | /{id}/unbind/telegram | admin | Unbind user's Telegram |
| POST | /{id}/unbind/email | admin | Unbind user's email |
| POST | /{id}/rebind/telegram | admin | Rebind user's Telegram (P0-3: numeric only) |
| POST | /{id}/rebind/email | admin | Rebind user's email |

### Admin Roles Routes (`/api/v1/admin/roles`)
| Method | Path | Auth | Notes |
|--------|------|------|-------|
| GET | / | admin | List roles |
| POST | / | admin | Create role |
| PUT | /{id} | admin | Update role |
| DELETE | /{id} | admin | Delete role |
| GET | /permissions/available | admin | Available permissions |
| PUT | /{id}/permissions | admin | Update role permissions |
| POST | /assign-role | admin | Assign role to user |
| POST | /remove-role | admin | Remove role from user |

### Admin System Settings (`/api/v1/admin/system-settings`)
| Method | Path | Auth | Notes |
|--------|------|------|-------|
| GET | / | admin | List settings |
| GET | /schema | admin | Settings schema |
| PUT | /{key} | admin | Update setting |
| GET | /registration-settings | admin | Registration settings |
| POST | /cors-cache/refresh | admin | Refresh CORS cache (HOTFIX-111B) |
| GET | /cors-cache/status | admin | CORS cache status (HOTFIX-111B) |

### Admin Domain Routes (`/api/v1/admin/domain-routes`) - HOTFIX-111B
| Method | Path | Auth | Notes |
|--------|------|------|-------|
| GET | / | admin | List domain routes |
| GET | /{id} | admin | Get single route |
| POST | / | admin | Create route |
| PUT | /{id} | admin | Update route |
| DELETE | /{id} | admin | Delete route |
| PUT | / | admin | Bulk update routes |
| GET | /export/cloudflared | admin | Export Cloudflared config |
| GET | /export/nginx | admin | Export Nginx config |

### App Token Routes (`/api/v1/app`)
| Method | Path | Auth | Notes |
|--------|------|------|-------|
| POST | /clients | admin | Create app client |
| GET | /clients | admin | List app clients |
| DELETE | /clients/{id} | admin | Delete client |
| POST | /tokens | admin | Create token |
| GET | /tokens | admin | List tokens |
| POST | /tokens/{id}/revoke | admin | Revoke token |
| POST | /tokens/{id}/rotate | admin | Rotate token |
| GET | /ping | app(ping) | Demo endpoint |

### Content Routes (`/api/v1/content`)
| Method | Path | Auth | Notes |
|--------|------|------|-------|
| GET | /blog | public | Public blog list |
| GET | /blog/{slug} | public | Public blog post |
| GET | /docs | public | Public docs list |
| GET | /docs/{slug} | public | Public doc page |
| GET | /wiki | public | Public wiki list |
| GET | /wiki/{slug} | public | Public wiki page |
| GET | /changelog | public | Public changelog |

### Admin Content Routes (`/api/v1/admin/content`)
| Method | Path | Auth | Notes |
|--------|------|------|-------|
| GET | / | user* | List content (permission check) |
| POST | / | user* | Create content |
| PUT | /{id} | user* | Update content |
| DELETE | /{id} | user* | Delete content |

### Announcements (`/api/v1/announcements`)
| Method | Path | Auth | Notes |
|--------|------|------|-------|
| GET | /active | public | Active announcements |
| GET | /{id} | public | Single announcement |
| POST | /{id}/read | user | Mark as read |

### Admin Announcements (`/api/v1/admin/announcements`)
| Method | Path | Auth | Notes |
|--------|------|------|-------|
| GET | / | user* | List all (permission check) |
| POST | / | user* | Create |
| PUT | /{id} | user* | Update |
| DELETE | /{id} | user* | Delete |

### User Check-in Routes (`/api/v1/user/checkin`) - ROUNDREQ R3
| Method | Path | Auth | Notes |
|--------|------|------|-------|
| GET | /status | user | Get check-in status (streak, today status) |
| POST | / | user | Perform check-in |
| GET | /history | user | Check-in history |
| GET | /settings | admin | Check-in settings (admin only) |

### Admin Check-in Risk Routes (`/api/v1/admin/checkin-risk`) - ROUNDREQ R3
| Method | Path | Auth | Notes |
|--------|------|------|-------|
| GET | / | admin | List users with risk status |
| PUT | /{id}/risk | admin | Update user risk status |

### Admin Checkin Settings Routes (`/api/v1/admin/checkin`) - Round-9
| Method | Path | Auth | Notes |
|--------|------|------|-------|
| GET | /basic | admin | Get basic checkin settings |
| PUT | /basic | admin | Update basic checkin settings |
| GET | /rules | admin | List event rules |
| POST | /rules | admin | Create event rule |
| GET | /rules/{id} | admin | Get single rule |
| PUT | /rules/{id} | admin | Update event rule |
| DELETE | /rules/{id} | admin | Delete event rule |
| POST | /rules/{id}/toggle | admin | Toggle rule enabled status |
| GET | /summary | admin | Get checkin settings summary |

### Admin Telegram Bots (`/api/v1/admin/telegram-bots`) - ROUNDREQ
| Method | Path | Auth | Notes |
|--------|------|------|-------|
| GET | / | admin | List Telegram bots |
| POST | / | admin | Create bot config |
| PUT | /{id} | admin | Update bot config |
| DELETE | /{id} | admin | Delete bot config |
| GET | /config/web-login | public | Get web-login bot config |

### Public Routes (`/api/v1/public`) - HOTFIX-411
| Method | Path | Auth | Notes |
|--------|------|------|-------|
| GET | /runtime-config | public | Frontend runtime config |

### edgea App Routes (`/miniapp` + `/api/v1/auth/telegram/webapp`) - ADD-413~417 Round-7
| Method | Path | Auth | Notes |
|--------|------|------|-------|
| GET | /miniapp/ | public | edgea App HTML page (splitpanes 5-section layout) |
| GET | /miniapp/?test_tg_uid=xxx | public | Test mode: skip Telegram validation |
| POST | /api/v1/auth/telegram/webapp/verify | public* | WebApp initData HMAC-SHA256 verify, returns JWT |

*Note: POST /verify accepts initData from Telegram WebApp SDK; validates HMAC-SHA256 hash with bot_token

### User by TG UID (`/api/v1/user/by-tg-uid`) - ROUNDREQ R3
| Method | Path | Auth | Notes |
|--------|------|------|-------|
| GET | /{tg_uid} | public | Get user info by Telegram UID (for Bot) |

### After-Sales Routes (`/api/v1/after-sales`) - ROUNDREQ R3
| Method | Path | Auth | Notes |
|--------|------|------|-------|
| GET | / | user | List user's after-sales cases |
| POST | / | user | Create after-sales case |
| GET | /{id} | user | Get case detail |
| POST | /{id}/messages | user | Add message |

### Admin After-Sales (`/api/v1/admin/after-sales`) - ROUNDREQ R3
| Method | Path | Auth | Notes |
|--------|------|------|-------|
| GET | / | user* | List all after-sales cases |
| GET | /{id} | user* | Get case detail |
| PUT | /{id}/status | user* | Update case status |
| POST | /{id}/messages | user* | Reply to case |

### Menu Routes (`/api/v1/menu`)
| Method | Path | Auth | Notes |
|--------|------|------|-------|
| GET | /all | user | All menus for current user |

### Affiliate Routes (`/api/v1/aff`)
| Method | Path | Auth | Notes |
|--------|------|------|-------|
| GET | /link | user | Get affiliate link |
| GET | /stats | user | Affiliate stats |
| GET | /commissions | user | Commission history |

### Payment Routes (`/api/v1/payment`)
| Method | Path | Auth | Notes |
|--------|------|------|-------|
| POST | /create-order | user | Create payment order |
| GET | /order/{id} | user | Get order status |
| POST | /tron/callback | public | TRON payment callback |

### Tickets (`/api/v1/tickets`)
| Method | Path | Auth | Notes |
|--------|------|------|-------|
| GET | / | user | List user tickets |
| POST | / | user | Create ticket |
| GET | /{id} | user | Get ticket |
| POST | /{id}/messages | user | Add message |
| POST | /{id}/close | user | Close ticket |

### Admin Tickets (`/api/v1/admin/tickets`)
| Method | Path | Auth | Notes |
|--------|------|------|-------|
| GET | / | user* | List all tickets |
| GET | /{id} | user* | Get ticket |
| PUT | /{id}/assign | user* | Assign ticket |
| POST | /{id}/messages | user* | Reply |
| PUT | /{id}/status | user* | Change status |

## Permission Key System

Routes marked with `user*` use the role-based permission system:
- User must be authenticated
- User's role permissions are checked against required permission key
- Permission keys are defined in `rolesDB.permissions` JSON array

### Available Permission Keys

| Key | Description |
|-----|-------------|
| view_users | View user list |
| edit_users | Create/update users |
| delete_users | Delete users |
| view_roles | View roles |
| edit_roles | Create/update roles |
| view_logs | View system logs |
| edit_settings | Modify system settings |
| view_content | View content list |
| edit_content | Create/update content |
| view_tickets | View all tickets |
| manage_tickets | Manage ticket assignments |
| view_announcements | View announcements |
| edit_announcements | Create/update announcements |
| manage_app_tokens | Create/revoke app tokens |
| manage_checkin_risk | Manage check-in risk control (Round-6) |
| manage_policies | Manage policies - slave policies, etc (Round-7) |
| manage_domain_settings | Manage CORS/domain routes (HOTFIX-111B) |

## Security Notes

1. **Public Endpoints**: Content and announcements have public read endpoints by design
2. **Admin Guard**: All `/api/v1/admin/*` routes require either:
   - `get_current_admin` (which_role != 'user'), or
   - `get_current_user` with permission key validation
3. **App Token Isolation**: App tokens can ONLY access `/api/v1/app/*` routes
4. **JWT Validation**: All user/admin routes validate JWT signature and expiration

### Round-DEPLOY-01: which_role Entry Control (2026-01-18)

The `which_role` field replaces the legacy `level >= 91` checks for admin access control.

#### which_role Values

| Value | Description | Entry Points |
|-------|-------------|--------------|
| **sysop** | System operator (superadmin) | Vben Admin |
| **admin** | Administrator | Vben Admin |
| **partner** | Partner | Vben Admin |
| **staff** | Staff member | Vben Admin |
| **user** | Regular user (default) | Portal / MiniApp only |

#### Entry Point Blocking

| Entry Point | which_role=user | which_role!=user |
|-------------|-----------------|------------------|
| Portal | ✅ Allowed | ✅ Allowed |
| MiniApp | ✅ Allowed | ✅ Allowed |
| Vben Admin | ❌ Blocked (403) | ✅ Allowed |

#### Gate Conditions (MUST FAIL)

| Scenario | Expected Result |
|----------|-----------------|
| which_role=user attempts Vben login | Login blocked, error notification |
| which_role=user with valid JWT accesses /admin/* | 403 Forbidden |
| which_role=user SSO token for Vben | Token rejected, redirect to login |

#### Migration Strategy

- Existing users default to `which_role='user'` (no admin access)
- Initial admin created via env variable gets `which_role='sysop'`
- Admins can update user's which_role via Vben User Detail modal

## Audit Results

| Check | Status |
|-------|--------|
| Admin routes protected | ✅ PASS |
| User routes protected | ✅ PASS |
| Public routes intentional | ✅ PASS |
| App token isolation | ✅ PASS |
| No unprotected admin endpoints | ✅ PASS |

### ROUNDREQ P0 Gate Verification (2026-01-04)

| Route | Auth Level | Expected | Actual | Status |
|-------|------------|----------|--------|--------|
| GET /user/bind/telegram-config | public | 200 | 200 | ✅ PASS |
| GET /user/bind/status | user | 401/200 | 401/200 | ✅ PASS |
| POST /user/bind/telegram | user | 401 | 401 | ✅ PASS |
| POST /user/bind/email | user | 401 | 401 | ✅ PASS |
| GET /admin/system-settings | admin | 401/200 | 401/200 | ✅ PASS |
| PUT /admin/system-settings/{key} | admin | 401 | 401 | ✅ PASS |
| GET /admin/system-settings/schema | admin | 401/200 | 401/200 | ✅ PASS |

**Domain Validation**: POST /user/bind/telegram validates `telegram_login_allowed_domains` and returns 403 if domain not allowed.

Evidence: `logs/ac_permission_matrix_roundreq.json`, `logs/ac_portal_tg_domain_denied.json`

### ROUNDREQ P0-2 Dynamic CORS (2026-01-05)

The DynamicCORSMiddleware reads allowed origins from `api_cors_origins` system setting.

| Test | Expected | Actual | Status |
|------|----------|--------|--------|
| OPTIONS with allowed origin | 204 | 204 | ✅ PASS |
| OPTIONS with denied origin | 403 | 403 | ✅ PASS |
| GET with allowed origin (CORS headers) | 200 + headers | 200 + headers | ✅ PASS |

Evidence:
- `logs/ac_cors_options_allowed.txt`
- `logs/ac_cors_options_denied.txt`
- `logs/ac_cors_get_allowed.json`

**Configuration**: Admin can edit `api_cors_origins` in System Settings → Domain Settings tab.
Screenshot: `logs/ac_ui_vben_domain_settings.png`

### HOTFIX-111B Domain Routes Management (2026-01-05)

Domain routes are persisted in `domain_routes` table and can be managed via Vben Admin UI.

| Route | Auth Level | Expected | Status |
|-------|------------|----------|--------|
| GET /admin/domain-routes | admin | 401/200 | ✅ PASS |
| POST /admin/domain-routes | admin | 401/200 | ✅ PASS |
| PUT /admin/domain-routes/{id} | admin | 401/200 | ✅ PASS |
| DELETE /admin/domain-routes/{id} | admin | 401/200 | ✅ PASS |
| GET /admin/domain-routes/export/cloudflared | admin | 401/200 | ✅ PASS |
| GET /admin/domain-routes/export/nginx | admin | 401/200 | ✅ PASS |
| POST /admin/system-settings/cors-cache/refresh | admin | 401/200 | ✅ PASS |
| GET /admin/system-settings/cors-cache/status | admin | 401/200 | ✅ PASS |

Features:
- **Domain Routes CRUD**: Create/Read/Update/Delete domain route mappings
- **Export Configs**: Generate cloudflared ingress YAML or nginx server blocks
- **CORS Cache Refresh**: Admin can manually refresh CORS allowlist cache
- **Vben UI Tab**: System Settings → Domain Routes tab

Evidence:
- `logs/ac_domain_routes_crud.json`
- `logs/ac_export_cloudflared.yaml`
- `logs/ac_export_nginx.conf`
- `logs/ac_cors_cache_refresh.json`
- `logs/ac_ui_vben_domain_routes_tab.png`

### HOTFIX-411 Round-5: Comprehensive Permission Smoke Test (2026-01-06)

Full permission matrix smoke test covering 25 endpoints across all auth levels.

#### Core Routes Verification

| Route | Auth Level | No Auth | User Auth | Admin Auth | Status |
|-------|------------|---------|-----------|------------|--------|
| GET /health | public | 200 | - | - | ✅ PASS |
| GET /api/v1/public/runtime-config | public | 200 | - | - | ✅ PASS |
| GET /api/v1/user/me | user | 401 | 200 | 200 | ✅ PASS |
| GET /api/v1/admin/users | admin | 401 | 403 | 200 | ✅ PASS |
| GET /api/v1/admin/system-settings | admin | 401 | - | 200 | ✅ PASS |
| GET /api/v1/admin/roles/ | admin | 401 | - | 200 | ✅ PASS |
| GET /api/v1/menu/all | user | 401 | 200 | 200 | ✅ PASS |
| GET /api/v1/content/blog | public | 200 | - | - | ✅ PASS |
| GET /api/v1/user/bind/telegram-config | public | 200 | - | - | ✅ PASS |
| GET /api/v1/user/bind/status | user | 401 | 200 | 200 | ✅ PASS |
| GET /api/v1/admin/domain-routes | admin | 401 | - | 200 | ✅ PASS |
| GET /api/v1/admin/orders/ | admin | 401 | - | 200 | ✅ PASS |
| GET /api/v1/admin/tickets/ | admin | 401 | - | 200 | ✅ PASS |

#### Round-3/Round-4 New Endpoints

| Route | Auth Level | No Auth | User Auth | Admin Auth | Status |
|-------|------------|---------|-----------|------------|--------|
| GET /api/v1/user/checkin/status | user | 401 | 200 | 200 | ✅ PASS |
| GET /api/v1/user/checkin/history | user | 401 | 200 | 200 | ✅ PASS |
| GET /api/v1/user/checkin/settings | admin | 401 | - | 200 | ✅ PASS |
| GET /api/v1/admin/checkin-risk/ | admin | 401 | - | 200 | ✅ PASS |
| GET /api/v1/admin/telegram-bots | admin | 401 | - | 200 | ✅ PASS |
| GET /api/v1/app/clients | admin | 401 | - | 200 | ✅ PASS |
| GET /api/v1/miniapp/ | public | 200 | - | - | ✅ PASS |
| GET /api/v1/user/by-tg-uid/{tg_uid} | public | 200 | - | - | ✅ PASS |
| GET /api/v1/after-sales/ | user | 401 | 200 | 200 | ✅ PASS |

#### Summary

- **Total Endpoints Tested**: 25
- **All Pass**: ✅ YES
- **Auth Enforcement**: Correct (401 for unauthenticated, 403 for unauthorized, 200 for authorized)

Evidence:
- `logs/ac_permission_matrix_smoke.json` (comprehensive smoke test results)

### Round-8: Bot + edgea App Verification (2026-01-06)

HOTFIX-418, HOTFIX-419, ADD-420, HOTFIX-421

#### Telegram Bot Single Instance (HOTFIX-418)

| Check | Status |
|-------|--------|
| Lockfile created on startup | ✅ PASS |
| PID written to lockfile | ✅ PASS |
| Duplicate instance detection | ✅ PASS |
| Stale lockfile cleanup | ✅ PASS |
| atexit cleanup on exit | ✅ PASS |

Evidence: `logs/ac_bot_single_instance_check.txt`

#### Bot URL HTTPS (HOTFIX-419)

| Setting | Before | After | Status |
|---------|--------|-------|--------|
| API_BASE | http://localhost:8000/api/v1 | https://api.example.com/api/v1 | ✅ FIXED |
| PORTAL_URL | http://localhost:3001 | https://portal.example.com | ✅ FIXED |
| MINIAPP_URL | https://api.example.com/miniapp | https://app.example.com/miniapp | ✅ FIXED |

Evidence: `logs/ac_bot_button_url_https.txt`

#### Nuxt edgea App Page (ADD-420)

| Feature | Status |
|---------|--------|
| Page created at nuxt-portal/pages/miniapp.vue | ✅ PASS |
| Telegram WebApp SDK integration | ✅ PASS |
| initData verification flow | ✅ PASS |
| /user/me data display | ✅ PASS |
| Topbar shows TG UID (pure number) | ✅ PASS |
| Splitpanes-style grid layout | ✅ PASS |
| Credits/Points display | ✅ PASS |
| Mobile responsive | ✅ PASS |

Evidence:
- `logs/ac_webapp_verify_ok.json`
- `logs/ac_webapp_me_ok.json`

#### edgea App Routes Permission

| Route | Auth | Expected | Status |
|-------|------|----------|--------|
| GET /miniapp/ | public | 200 | ✅ PASS |
| GET /miniapp/?test_tg_uid=xxx | public | 200 | ✅ PASS |
| POST /auth/telegram/webapp/verify | public* | 200 | ✅ PASS |
| GET /user/by-tg-uid/{tg_uid} | public | 200/404 | ✅ PASS |

*verify endpoint validates initData HMAC-SHA256 hash

#### Nuxt Portal edgea App Route

| Route | Auth | Expected | Status |
|-------|------|----------|--------|
| GET /miniapp | public | 200 | ✅ PASS |

Evidence: `logs/ac_permission_matrix_round8.json`

### Round-4: Orders/Tickets/After-sales + Announcements/Invite Tree/AFF (2026-01-07)

#### New/Updated Routes

##### User Orders (`/api/v1/user/orders`)
| Method | Path | Auth | Notes |
|--------|------|------|-------|
| GET | / | user | List user's orders |
| GET | /{id} | user | Get order detail |

##### User Tickets (`/api/v1/user/tickets`)
| Method | Path | Auth | Notes |
|--------|------|------|-------|
| GET | / | user | List user's tickets |
| POST | / | user | Create ticket |
| GET | /{id} | user | Get ticket detail |
| POST | /{id}/reply | user | Reply to ticket |
| POST | /{id}/close | user | Close ticket |

##### User After-Sales (`/api/v1/user/after-sales`)
| Method | Path | Auth | Notes |
|--------|------|------|-------|
| GET | / | user | List user's after-sales cases |
| POST | / | user | Create case (order_ids[] + ticket_id required) |
| GET | /{id} | user | Get case detail |
| GET | /options/orders | user | Get orders for dropdown |
| GET | /options/tickets | user | Get tickets for dropdown |

##### User Invite Tree (`/api/v1/user/invite-tree`)
| Method | Path | Auth | Notes |
|--------|------|------|-------|
| GET | / | user | Get my invite tree (ancestors + descendants) |
| GET | /stats | user | Get my invite statistics |
| GET | /invitees | user | List my direct invitees |

##### Admin Orders (`/api/v1/admin/orders`)
| Method | Path | Auth | Notes |
|--------|------|------|-------|
| GET | / | admin | List all orders |
| GET | /{id} | admin | Get order detail |
| GET | /stats | admin | Order statistics |

##### Admin Tickets (`/api/v1/admin/tickets`)
| Method | Path | Auth | Notes |
|--------|------|------|-------|
| GET | / | admin | List all tickets |
| GET | /{id} | admin | Get ticket detail (includes internal notes) |
| POST | /{id}/reply | admin | Reply (can add internal note) |
| PUT | /{id} | admin | Update ticket (status/priority/category/assigned_to) |
| GET | /stats | admin | Ticket statistics |

##### Admin After-Sales (`/api/v1/admin/after-sales`)
| Method | Path | Auth | Notes |
|--------|------|------|-------|
| GET | / | admin | List all after-sales cases |
| GET | /{id} | admin | Get case detail |
| POST | / | admin | Create case |
| POST | /{id}/process | admin | Process case (approve/reject) |
| PUT | /{id}/status | admin | Update case status |
| GET | /stats | admin | After-sales statistics |

##### Announcements (`/api/v1/announcements`)
| Method | Path | Auth | Notes |
|--------|------|------|-------|
| GET | / | public | List published announcements (filter by audience) |
| GET | /latest | public | Get latest announcement |
| GET | /latest/global | public | Get latest global announcement |
| GET | /latest/user | public | Get latest user announcement |
| GET | /milestones | public | List milestone announcements |
| GET | /{id} | public | Get announcement detail |

##### Admin Announcements (`/api/v1/admin/announcements`)
| Method | Path | Auth | Notes |
|--------|------|------|-------|
| GET | / | user* | List all announcements |
| POST | / | user* | Create (supports audience + is_milestone) |
| GET | /{id} | user* | Get detail |
| PUT | /{id} | user* | Update |
| DELETE | /{id} | user* | Delete |
| POST | /{id}/publish | user* | Publish |
| POST | /{id}/unpublish | user* | Unpublish |

#### Round-4 Permission Keys

| Key | Description | Type |
|-----|-------------|------|
| view_orders | View orders list | user |
| view_tickets | View tickets list | user |
| create_ticket | Create new ticket | user |
| view_after_sales | View after-sales cases | user |
| create_after_sales | Create after-sales case | user |
| view_invite_tree | View invite tree | user |
| view_announcements | View announcements | user |
| manage_orders | Manage all orders | admin |
| manage_tickets | Manage all tickets | admin |
| manage_after_sales | Manage after-sales | admin |
| manage_announcements | Create/edit announcements | admin |
| aff_enabled | Access affiliate system | user |
| aff_manage | Manage affiliate system | admin |

#### After-Sales order_ids[] Requirement

Round-4 requires after-sales cases to bind multiple orders:
- `order_ids[]`: Required array of order IDs (min 1)
- `ticket_id`: Required ticket ID (must belong to user)

API Response includes `order_ids` array instead of single `order_id`.

#### Announcements audience + milestone

Announcements support:
- `audience`: "global" (visible before login) or "user" (visible after login)
- `is_milestone`: boolean for timeline display

### Round-6: Credits/Payments/Store/Risk Management (2026-01-07)

#### User Credits Routes (`/api/v1/user/credits`) - Extended
| Method | Path | Auth | Notes |
|--------|------|------|-------|
| GET | / | user | Get user credits balance |
| GET | /ledger | user | Get credits transaction history |
| GET | /topup-info | user | Get TRON USDT topup info (address, rate) |
| GET | /store-info | user | Get store info (seat/invite prices, enable flags) |
| POST | /purchase-seat | user | Purchase seat or invite code with credits |

#### Admin Payments Routes (`/api/v1/admin/payments`) - NEW
| Method | Path | Auth | Notes |
|--------|------|------|-------|
| GET | / | admin | List all payments |
| GET | /{payment_id} | admin | Get payment detail with linked topups |
| POST | /scan | admin | Manual TRON USDT payment scan |
| POST | /adjust-credits | admin | Admin adjust user credits (+ or -) |
| GET | /topups/ | admin | List all topup orders |
| GET | /seat-orders/ | admin | List all seat/invite purchase orders |

#### App Policies Routes (`/api/v1/app/policies`) - NEW
| Method | Path | Auth | Notes |
|--------|------|------|-------|
| GET | /export | app(policies:read) | Export policies with version hash |

#### Round-6 Permission Matrix Verification

| Route | Auth Level | No Auth | User Auth | Admin Auth | Status |
|-------|------------|---------|-----------|------------|--------|
| GET /api/v1/user/credits/topup-info | user | 401 | 200 | 200 | ✅ PASS |
| GET /api/v1/user/credits/store-info | user | 401 | 200 | 200 | ✅ PASS |
| POST /api/v1/user/credits/purchase-seat | user | 401 | 200 | 200 | ✅ PASS |
| GET /api/v1/admin/payments | admin | 401 | 403 | 200 | ✅ PASS |
| POST /api/v1/admin/payments/scan | admin | 401 | 403 | 200 | ✅ PASS |
| POST /api/v1/admin/payments/adjust-credits | admin | 401 | 403 | 200 | ✅ PASS |
| GET /api/v1/admin/payments/topups/ | admin | 401 | 403 | 200 | ✅ PASS |
| GET /api/v1/admin/payments/seat-orders/ | admin | 401 | 403 | 200 | ✅ PASS |

#### Round-6 Features

**Credits System**:
- Credits (credits) for purchases: seats, invite codes, subscriptions
- Points (points) for rewards: check-in only
- Complete separation: purchases ONLY deduct credits, check-in ONLY adds points

**TRON USDT Payment**:
- Idempotent by txid unique constraint
- Credits only after confirmations threshold reached
- Admin can trigger manual scan

**Risk Scoring Plugin**:
- IP/ASN frequency analysis (+20)
- Fixed-time check-in pattern (+30)
- Script UA detection (+40)
- Same IP multi-user (+25)
- VPN/Proxy ASN detection (+15)
- Threshold configurable via system settings

Evidence:
- `logs/ac_round6_credits_topup.json`
- `logs/ac_round6_store_purchase.json`
- `logs/ac_round6_admin_payments.json`

### Round-9: MiniApp UX + Ops Panel + Smoke Tests (2026-01-08)

#### MiniApp Check-in Calendar (`/api/v1/user/checkin`)
| Method | Path | Auth | Notes |
|--------|------|------|-------|
| GET | /calendar | user | Get month calendar data (checked_dates, month_stats) |

#### Admin Ops Panel (`/api/v1/admin/ops`) - NEW
| Method | Path | Auth | Notes |
|--------|------|------|-------|
| GET | /restricted-users | admin | List users with level <= 0 (restricted) |
| POST | /promote-user | admin | Promote restricted user (level, seats, role) |
| POST | /grant-seat | admin | Grant seats to user from admin pool |
| GET | /stats | admin | Ops statistics (total/restricted/active users) |
| GET | /available-roles | admin | List roles for assignment dropdown |

#### MiniApp Admin Settings (`/api/v1/admin/miniapp`) - Round-8 Extended
| Method | Path | Auth | Notes |
|--------|------|------|-------|
| GET | /menu-settings | admin | Get all MiniApp menu settings |
| PUT | /menu-settings | admin | Bulk update menu settings |
| PUT | /menu-settings/{id} | admin | Update single menu item |

#### MiniApp User Menu (`/api/v1/miniapp`)
| Method | Path | Auth | Notes |
|--------|------|------|-------|
| GET | /menu | user | Get user's available MiniApp menu items |
| GET | /menu/all | public | Get all enabled MiniApp menus (no permission check) |

#### Round-9 Permission Keys

| Key | Description | Type |
|-----|-------------|------|
| manage_ops_panel | Access ops panel for restricted users | admin |
| grant_seat_admin | Grant seats to users from admin pool | admin |
| promote_restricted_user | Promote restricted users | admin |

#### Round-9 MiniApp Features

**Check-in Calendar**:
- edgea calendar component with checked days highlighted
- Month navigation (prev/next)
- Streak progress bar (7-day visualization)
- Monthly stats (days checked, points earned)

**Announcements Filter**:
- Filter tabs: All / Global / User / Milestone
- Audience badge (G=global, U=user)
- Milestone badge (trophy icon)

**Media Accounts Entry**:
- Clickable pane navigates to Media page
- View All link for quick access

#### Smoke Test Scripts

| Script | Purpose |
|--------|---------|
| `scripts/smoke_portal_api.sh` | Portal login -> /me -> /menu/all |
| `scripts/smoke_miniapp_api.sh` | MiniApp login -> /me -> /miniapp/menu |
| `scripts/smoke_admin_api.sh` | Admin login -> admin endpoints |

Evidence:
- `logs/ac_smoke_round9_portal.txt`
- `logs/ac_smoke_round9_miniapp.txt`
- `logs/ac_smoke_round9_admin.txt`
- `docs/TELEGRAM_WEBVIEW_MATRIX.md`
