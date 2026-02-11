# Changelog

## [1.2.0] - 2026-02-10

### Internationalization (I18N)

> Version 1.2.0 delivers full 4-language (en / zh-CN / zh-TW / ja) internationalization
> across the entire stack: Portal, MiniApp, Vben Admin, and Backend API.

#### Architecture Relationship Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                    I18N Architecture (v1.2.0)                       │
│                    Languages: en / zh-CN / zh-TW / ja               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────────┐   ┌──────────────────┐   ┌─────────────────┐  │
│  │  Portal (Nuxt)  │   │ MiniApp (Nuxt)   │   │  Vben Admin     │  │
│  │  @nuxtjs/i18n   │   │ @nuxtjs/i18n     │   │  vue-i18n       │  │
│  ├─────────────────┤   ├──────────────────┤   ├─────────────────┤  │
│  │ I18N-01: Shell  │   │ I18N-01: Shell   │   │ I18N-02: Infra  │  │
│  │  - AppHeader    │   │  - Topbar        │   │  - 4 lang types │  │
│  │  - AppFooter    │   │  - Menu sidebar  │   │  - Antd/dayjs   │  │
│  │  - Login/Reg    │   │  - Dashboard     │   │  - Route titles │  │
│  │  - Setup/Index  │   │  - Checkin/Error  │   │  - Auth errors  │  │
│  │  - 3 Layouts    │   │  - Auth forms    │   │  - Notifications│  │
│  │  - Lang Switch  │   │  - Lang Switch   │   │                 │  │
│  ├─────────────────┤   ├──────────────────┤   │ 142 page.json   │  │
│  │ I18N-04: Pages  │   │ I18N-04: Pages   │   │ keys × 4 locale │  │
│  │  23 portal pages│   │ 19 biz sections  │   └─────────────────┘  │
│  │  1422 keys × 4  │   │ in miniapp.vue   │                        │
│  └────────┬────────┘   └────────┬─────────┘                        │
│           │                     │                                   │
│           └──────────┬──────────┘                                   │
│                      │                                              │
│              ┌───────▼────────┐                                     │
│              │  Locale Files  │                                     │
│              │  (Nuxt i18n)   │                                     │
│              ├────────────────┤                                     │
│              │ en.json        │  1422 keys each                     │
│              │ zh-CN.json     │  = 5688 entries total               │
│              │ zh-TW.json     │                                     │
│              │ ja.json        │                                     │
│              └───────┬────────┘                                     │
│                      │ ?lang= / localStorage / Accept-Language      │
│              ┌───────▼────────┐                                     │
│              │  Backend API   │                                     │
│              │  (FastAPI)     │                                     │
│              ├────────────────┤                                     │
│              │ I18N-03: Infra │                                     │
│              │  - t() func   │  192 keys × 4 locales               │
│              │  - middleware  │  28 message_keys                    │
│              │  - menu.py    │  129 page.* keys                    │
│              │  - exceptions │  message_key on all exceptions      │
│              └────────────────┘                                     │
│                                                                     │
│  Language Priority: ?lang= > localStorage > Accept-Language > zh-CN │
│  localStorage key: uhd_lang                                         │
│  i18n framework: @nuxtjs/i18n (Nuxt) / vue-i18n (Vben) / custom   │
│                  Accept-Language (Backend)                           │
└─────────────────────────────────────────────────────────────────────┘
```

#### Round I18N-04: Portal + MiniApp Full Page i18n Key-ization
- Key-ized all 23 portal page `.vue` files: replaced hardcoded English/Chinese UI strings with `$t()` i18n calls
- Key-ized miniapp.vue business page sections (19 sections): profile, wallet, media-accounts, orders, tickets, after-sales, shop, checkin, benefits, display-prefs, affiliate, cdkeys, seats, renewal-cards, assets, invite-codes, invite-tree, subscriptions, watch-history
- 1422 total flat keys × 4 locales = 5688 translation entries, fully consistent
- Added `portal.*` namespace (75+ common keys + 23 page-specific sections)
- Added `miniapp.*` business sections (9 new subsections + common keys)
- `pnpm build` passes with all tasks successful

#### Round I18N-03: Backend API i18n Infrastructure
- Created `app/i18n/` module: `t()` translation function + 4-language JSON locales (192 keys each)
- Accept-Language middleware: parses request header, injects locale, supports `?lang=` override
- Extended `success_response()` / `error_response()` with optional `message_key` field
- Exception hierarchy (`exceptions.py`) supports `message_key` parameter on all exception classes
- Full menu.py key-ization: 129 `page.*` keys, Vben frontend resolves via `$t()`
- Key-ized 28 message_keys across 7 router files (auth/orders/cdkeys/sessions/roles/users/checkin)
- Synced Vben page.json to 183 keys × 4 locales
- Python syntax + JSON validation + import tests all pass

#### Round I18N-02: Vben Admin i18n Infrastructure
- Extended `SupportedLanguagesType` to support 4 languages (en-US/zh-CN/zh-TW/ja)
- Created 12 new locale files (package-level + app-level for zh-TW and ja)
- Integrated Ant Design Vue / dayjs locales for zh-TW and ja
- Key-ized 88 route titles in admin.example.com (added 81 new admin keys)
- Key-ized 7 hardcoded Chinese titles in user-center.ts
- Key-ized notification demo data in basic.vue
- Key-ized access-denied error messages in store/auth.ts
- Key-ized redirect text in change-password page
- 142 page.json keys × 4 locales = 568 translation entries, fully consistent
- `pnpm build` passes with all 14 tasks successful

#### Round I18N-01: Portal + MiniApp i18n Infrastructure
- Installed `@nuxtjs/i18n` with `strategy: 'no_prefix'`, lazy loading, 4 locales
- Created 4 locale JSON files (268 keys each): prefixes common/nav/auth/setup/landing/footer/miniapp/lang
- Portal shell key-ization: AppHeader, AppFooter, login, register, setup, index, 3 layouts
- MiniApp shell key-ization: topbar, menu sidebar, dashboard cards, checkin area, error pages, auth forms
- Language switching UI in AppHeader + MiniApp topbar with localStorage persistence
- Client plugin `i18n-init.client.ts` for language priority: `?lang=` > localStorage > Accept-Language > zh-CN
- 268 keys × 4 locales = 1072 translation entries, fully consistent
- Smoke test 13/13 PASS, `pnpm build` SUCCESS

#### I18N Statistics Summary

| Round | Scope | Keys × Locales | Total Entries |
|-------|-------|----------------|---------------|
| I18N-01 | Portal + MiniApp shell | 268 × 4 | 1,072 |
| I18N-02 | Vben Admin infra | 142 × 4 | 568 |
| I18N-03 | Backend API + menu | 192 × 4 (backend) + 183 × 4 (frontend) | 1,500 |
| I18N-04 | Portal + MiniApp pages | 1,422 × 4 | 5,688 |
| **Total** | **Full stack** | | **8,828** |

---

## [1.1.0] - 2026-01-25

### Features

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

### Bug Fixes

- **LEVEL-24**: resolve API smoke test failures and add evidence (eb98cb85)
- **lint**: remove unused imports (6d3d2e4c)

### Documentation

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

### Tests

- **LEVEL-24B**: add Playwright E2E test for Vben audit center (796efbd8)
- **LEVEL-24A**: add API smoke test script for Round-23 audit features (b2f0e8bb)

### CI/CD

- **LEVEL-24**: add E2E compose stack and seed script (8ab47643)
- **LEVEL-24C**: add level24-audit-gate job for API smoke and Playwright tests (854313d0)
