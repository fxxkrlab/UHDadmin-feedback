# I18N 多语言国际化 — 总路线图

> 目标语言：en / zh-CN / zh-TW / ja
> 语言优先级：`?lang=` > localStorage (`uhd_lang`) > Accept-Language > default `zh-CN`
> 唯一 i18n 框架：`@nuxtjs/i18n`（Nuxt）/ Vue I18n（Vben）/ 后端 Accept-Language 分发

---

## 总览

| Round | 优先级 | 范围 | 状态 |
|-------|--------|------|------|
| I18N-01 | P0 | Portal + MiniApp：i18n 基建 + 外壳层 key 化 | **DONE** |
| I18N-02 | P0 | Vben：i18n 基建 + 菜单/布局/通用组件 key 化 | **DONE** |
| I18N-03 | P0 | Backend：API message_key + Accept-Language + 动态内容多语言 | **DONE** |
| I18N-04 | P1 | Portal + MiniApp 全量替换（23 页面 + 19 miniapp 区域） | **DONE** |
| I18N-05 | P1 | Vben：页面级全量替换 | TODO |
| I18N-06 | P0 | CI 闸门：硬编码检测 + Playwright 冒烟 + 覆盖率报表 | TODO |
| I18N-07 | P1 | 后端字段级 detail 本地化 + 可配置文案模板 | TODO |
| I18N-08 | P0 | 统一翻译工作流：key 变更检测 + 缺翻译阻断 + 发布验收 | TODO |

---

## Round I18N-01 — DONE (2026-02-10)

### 范围
- `@nuxtjs/i18n` 安装与 Nuxt 配置（`strategy: 'no_prefix'`, `lazy: true`）
- 4 份 locale JSON（268 keys，前缀：common/nav/auth/setup/landing/footer/miniapp/lang）
- Portal 外壳层 key 化：AppHeader、AppFooter、login、register、setup、index、3 个 layouts
- MiniApp 外壳层 key 化：topbar、menu、dashboard 卡片、签到、错误页、登录注册区域
- 语言切换 UI（AppHeader + MiniApp topbar）+ localStorage 持久化
- Client plugin `i18n-init.client.ts`（语言优先级）

### 关键文件
```
nuxt-portal/
├── i18n/locales/{en,zh-CN,zh-TW,ja}.json  (268 keys each)
├── i18n/KEYS.md
├── plugins/i18n-init.client.ts
├── nuxt.config.ts (i18n module)
├── components/AppHeader.vue (+ lang switcher)
├── components/AppFooter.vue
├── pages/{login,register,setup,index,miniapp}.vue
└── layouts/{default,portal,setup}.vue
```

### 证据
- `logs/ac_i18n01_*` (8 files), smoke 13/13 PASS, build SUCCESS

### 踩坑记录
- `langDir` 应写 `'locales'` 而非 `'i18n/locales'`（v10 已有 `i18n/` 基目录）
- 并行 agent key 化后需统一校验 key 名一致性（4 处 mismatch 手工修正）
- CSR 页面不可 curl 验证文案替换

---

## Round I18N-02 — DONE (2026-02-10)

### 范围
Vben (vue-vben-admin) 菜单/布局/通用组件 key 化。扩展 4 语言支持 (en-US/zh-CN/zh-TW/ja)。

### 完成的任务
1. **扩展 SupportedLanguagesType**：添加 `'ja' | 'zh-TW'` 到类型定义 + SUPPORT_LANGUAGES 数组
2. **创建 12 份新 locale 文件**：packages 级 4 文件 × 2 语言 + app 级 2 文件 × 2 语言
3. **Antd/Dayjs locale 集成**：index.ts 添加 ja_JP/zh_TW 的 antd locale + dayjs locale 动态导入
4. **admin.example.com 路由 key 化**：88 个 title 全部替换为 `$t('page.admin.*')`，新增 81 个 admin key
5. **user-center.ts 路由 key 化**：7 个硬编码中文 title 替换为 `$t('page.userCenter.*')`
6. **basic.vue 通知数据 key 化**：4 条示例通知的中文文案替换为 `$t('common.notification.*')`
7. **store/auth.ts 错误消息 key 化**：访问受限提示替换为 `$t('common.*')`
8. **change-password 页面**："正在跳转..." 替换为 `$t('common.redirecting')`
9. **构建验证**：`pnpm build` 14 tasks 全部成功

### 关键文件
```
vue-vben-admin/
├── packages/locales/src/typing.ts              (SupportedLanguagesType 扩展)
├── packages/constants/src/core.ts              (SUPPORT_LANGUAGES 扩展)
├── packages/locales/src/langs/zh-TW/           (4 files: common/auth/preferences/ui)
├── packages/locales/src/langs/ja/              (4 files: common/auth/preferences/ui)
├── apps/web-antd/src/locales/
│   ├── index.ts                                (antd/dayjs locale loader)
│   └── langs/
│       ├── en-US/page.json                     (142 keys)
│       ├── zh-CN/page.json                     (142 keys)
│       ├── zh-TW/page.json + demos.json        (142 keys + demos)
│       └── ja/page.json + demos.json           (142 keys + demos)
├── apps/web-antd/src/router/routes/modules/
│   ├── admin.example.com                                (88 titles key-ized)
│   └── user-center.ts                          (7 titles key-ized)
├── apps/web-antd/src/layouts/basic.vue          (notifications key-ized)
├── apps/web-antd/src/store/auth.ts              (error messages key-ized)
└── apps/web-antd/src/views/user/change-password/index.vue (redirect text key-ized)
```

### Key 统计
- page.json: 142 keys × 4 locales = 568 翻译条目
- packages locale: 268 keys × 4 locales = 1072 翻译条目
- 总计: 1640 翻译条目，4 语言一致

### 约束
- 不动 Portal/MiniApp（已在 I18N-01 完成）
- 不动后端
- 沿用 Vben 已有的 vue-i18n 机制

---

## Round I18N-03 — DONE (2026-02-10)

### 范围
后端 API 响应消息本地化 + Accept-Language 中间件 + 菜单/权限 key 化。

### 完成的任务
1. **Backend i18n 基础设施** (`app/i18n/__init__.py`)
   - `t(key, locale, **kwargs)` 翻译函数，支持模板变量 (`{field}`, `{permission}`)
   - `contextvars` locale 传播，4 语言 JSON locale 文件 (192 keys each)
   - `normalize_locale()` 处理各种变体 (zh-Hant-TW → zh-TW, en-US → en)
   - `parse_accept_language()` 带 quality 值解析
2. **Accept-Language 中间件** (`app/middleware/locale.py`)
   - Starlette `BaseHTTPMiddleware`，注入 `request.state.locale`
   - 优先级：`?lang=` > Accept-Language header > default zh-CN
   - 注册于 `app/main.py`，位于 CORS 之后
3. **Response envelope 扩展** (`app/utils.py`)
   - `success_response()` / `error_response()` 新增 `message_key` 参数
   - 有 `message_key` 时自动通过 `t()` 翻译 `message`，并在响应中包含 `message_key`
4. **Exception hierarchy** (`app/exceptions.py`)
   - `UHDAdminException` 及所有子类新增 `message_key` 参数
   - 4 个异常处理器均传递 `message_key` 到 `error_response()`
5. **Menu labels key 化** (`app/routers/menu.py`)
   - FULL_MENU_STRUCTURE: 所有 `meta.title` 替换为 `page.admin.*` keys (122 keys)
   - PARTNER_MENU_STRUCTURE: 6 个 title 替换为 `page.partner.*` keys
   - DEFAULT_PERMISSION_GROUPS: 7 个 label 替换为 `page.permissionGroup.*` keys
   - Vben 前端 page.json 同步添加 183 keys × 4 locales
6. **API response messages key 化** (7 router files, 28 message_keys)
   - 17 个中文硬编码消息 → `message_key` (auth.py, user_orders.py, user_cdkeys.py, sessions.py, media_access_control.py, media_account_dashboard.py, menu.py)
   - 11 个英文 success_response → `message_key` (roles.py, users.py, checkin_settings.py)

### 关键文件
```
app/
├── i18n/
│   ├── __init__.py              (i18n 核心：t(), set_locale(), normalize_locale())
│   └── locales/
│       ├── en.json              (192 keys)
│       ├── zh-CN.json           (192 keys)
│       ├── zh-TW.json           (192 keys)
│       └── ja.json              (192 keys)
├── middleware/locale.py          (Accept-Language 中间件)
├── utils.py                     (message_key 扩展)
├── exceptions.py                (message_key 扩展)
├── main.py                      (中间件注册)
└── routers/
    ├── menu.py                  (129 page.* keys)
    ├── auth.py                  (4 message_keys)
    ├── user_orders.py           (1 message_key)
    ├── user_cdkeys.py           (3 message_keys)
    └── admin/
        ├── sessions.py          (4 message_keys)
        ├── media_access_control.py (3 message_keys)
        ├── media_account_dashboard.py (1 message_key)
        ├── roles.py             (4 message_keys)
        ├── users.py             (4 message_keys)
        └── checkin_settings.py  (3 message_keys)
```

### Key 统计
- Backend locale: 192 keys × 4 locales = 768 翻译条目
- Frontend page.json: 183 keys × 4 locales = 732 翻译条目
- Router message_keys: 28 unique message_key values

### 验证
- Python syntax: 15 modified files ALL OK
- JSON validation: 8 locale files ALL OK
- Import test: `t()`, `normalize_locale()`, template substitution ALL PASS
- Key consistency: 4 backend + 4 frontend locale files all match

### 约束
- 不破坏现有 API 信封格式 `{code, message, data}`
- `message_key` 为新增可选字段，`message` 保持向后兼容
- 无 Accept-Language 时默认 zh-CN
- 本轮聚焦菜单 + 高频错误/成功消息，剩余 ~435 个 HTTPException 留待后续

---

## Round I18N-04 — DONE (2026-02-10)

### 范围
Portal 23 个页面 + MiniApp 19 个业务区域全量 key 化（原计划分 04a/b/c 三批，实际一轮完成）。

### 完成的页面

**Portal 页面 (23 个)**:
index, profile, orders, shop, media-accounts, seats, checkin, wallet, growth, benefits,
tickets, after-sales, affiliate, assets, cdkeys, whitelist, messages, display-preferences,
invite-codes, invite-tree, renewal-cards, subscriptions, watch-history

**MiniApp 业务区域 (19 个)**:
profile, wallet, media-accounts, cdkey-redeem, invite-codes, affiliate, orders, tickets,
after-sales, shop, cdkeys, seats, checkin-details, benefits, display-prefs, renewal-cards,
coming-soon, assets, modals

### Key 统计
- Portal: `portal.example.com.*` (75+ keys) + 23 个页面专属区域
- MiniApp: `miniapp.common.*` + 9 个新业务子区域
- **总计: 1422 keys × 4 locales = 5688 翻译条目**

### 关键文件
```
nuxt-portal/
├── i18n/locales/{en,zh-CN,zh-TW,ja}.json  (1422 keys each)
├── pages/portal/*.vue                      (23 files key-ized)
└── pages/miniapp.vue                       (19 business sections key-ized)
```

### 踩坑记录
- 并行 agent 在不同 locale 文件中创建了不同 key 名 → 以 en.json 为 source of truth 统一
- `unplugin-vue-i18n` 构建插件拒绝 locale 消息中的 HTML 标签 (`<strong>`) → 移除 HTML，改用纯文本
- miniapp.vue (13639 行) 必须分 3 批顺序处理，避免并行 agent 编辑冲突
- `v-html="$t()"` 不兼容严格 HTML 检查 → 改为 `{{ $t() }}` 文本插值

---

## Round I18N-05 — Vben 页面级全量

### 范围
Vben 管理后台所有业务页面 key 化。

### 预期页面
- 用户管理（列表/详情/编辑）
- 角色/权限管理
- 委派链管理
- 风控/审计
- 商品/订单管理
- 发票/财务
- 系统设置
- TG Bot 管理
- 签到设置
- 公告管理

### 约束
- 沿用 I18N-02 建立的 Vben i18n 基础设施
- Admin 页面优先级低于用户页面，可分批进行

---

## Round I18N-06 — CI 闸门

### 范围
自动化保障翻译质量，防止硬编码字符串回归。

### 预期任务
1. **硬编码检测脚本**：`scripts/i18n_lint.sh`
   - rg 扫描 `.vue`/`.ts` 文件中的中文字符、常见英文 UI 词（"Submit", "Cancel" 等）
   - 白名单机制（品牌名、技术术语等）
   - CI job `i18n-lint` 阻断
2. **Key 一致性检查**：4 份 locale key 集合必须相同
3. **Playwright 语言切换冒烟**：
   - 访问 Portal 4 语言，断言关键文案变化
   - 访问 MiniApp 4 语言，断言 dashboard 文案变化
4. **翻译覆盖率报表**：统计已翻译/未翻译的 key 比例

---

## Round I18N-07 — 后端字段级 detail 本地化（可选）

### 范围
- policy_code 解释文案本地化（如 `CHECKIN_DUPLICATE` → "该日期已签到" / "Already checked in"）
- 风控原因模板本地化
- 管理端可配置文案模板（Vben 界面编辑翻译 key）

---

## Round I18N-08 — 统一翻译工作流（可选）

### 范围
- Key 变更检测（新增/删除/修改 key 时 CI 提醒）
- 缺翻译阻断（任一语言缺失 key 则 CI 失败）
- 翻译覆盖率报表（按模块/页面统计）
- 发布前验收脚本（一键检查所有 locale 完整性）

---

## 补充项（GPT 原计划未覆盖）

### 日期/数字/货币格式化
- 统一使用 `@nuxtjs/i18n` 的 `$d()` / `$n()` 替代手写格式化
- 并入 I18N-04a/b/c 各页面替换时一并处理

### TG Bot 消息本地化
- Bot 发给用户的消息（签到提醒、验证码等）需根据用户语言偏好发送
- 依赖 I18N-03 的用户语言偏好存储
- 可并入 I18N-07 或单独处理

### 不需要处理的
- RTL 布局：4 种目标语言均不需要
- SEO meta：Portal/MiniApp 均为 CSR (`ssr:false`)，SEO 意义不大
- Email 模板：当前无邮件发送逻辑

---

## 执行顺序

```
I18N-01 ✅ → I18N-02 ✅ → I18N-03 ✅ → I18N-04 ✅ → I18N-05 → I18N-06 → I18N-07 → I18N-08
                │                         │
                └── 不依赖后端 ─────────────┘ 依赖 I18N-03 的 message_key
```

- I18N-01~04 已全部完成 ✅
- I18N-05 下一步：Vben 管理后台页面级全量 key 化
- I18N-06 应在 I18N-05 完成后建立（CI 闸门）
- I18N-07/08 为锦上添花，最后做
