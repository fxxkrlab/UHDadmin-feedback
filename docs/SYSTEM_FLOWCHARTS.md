# UHDadmin 系统流程图 / System Flowcharts

> v1.1.35 | 本文档包含所有核心功能的 Mermaid 流程图，方便编写文档时引用。

[English Version](#english-version)

---

## 目录

1. [系统架构总览](#1-系统架构总览)
2. [用户认证流程](#2-用户认证流程)
3. [Telegram MiniApp 启动流程](#3-telegram-miniapp-启动流程)
4. [签到系统](#4-签到系统)
5. [商城购买流程](#5-商城购买流程)
6. [媒体账号管理](#6-媒体账号管理)
7. [席位管理](#7-席位管理)
8. [邀请与推广返利](#8-邀请与推广返利)
9. [成长等级系统](#9-成长等级系统)
10. [工单系统](#10-工单系统)
11. [CDKey 兑换流程](#11-cdkey-兑换流程)
12. [菜单权限控制 (L2)](#12-菜单权限控制-l2)
13. [Docker 构建与发布](#13-docker-构建与发布)
14. [部署流程](#14-部署流程)

---

## 1. 系统架构总览

```mermaid
graph TB
    subgraph 客户端
        TG[Telegram Bot/MiniApp]
        WEB[Web Portal<br/>Nuxt 3]
        ADMIN[Admin 后台<br/>Vue Vben Admin]
    end

    subgraph 网关层
        APISIX[Apache APISIX<br/>API Gateway]
    end

    subgraph 服务层
        API[FastAPI Backend<br/>Python 3.11]
        PORTAL[Nuxt Portal<br/>SSR/SPA]
        VBEN[Vben Admin<br/>Static SPA]
    end

    subgraph 数据层
        PG[(PostgreSQL 15)]
        REDIS[(Redis 7)]
    end

    subgraph 外部服务
        EMBY[Emby Server]
        JF[Jellyfin Server]
        STRIPE[Stripe / 支付]
    end

    TG --> APISIX
    WEB --> APISIX
    ADMIN --> APISIX

    APISIX -->|/api/v1/*| API
    APISIX -->|/portal/*| PORTAL
    APISIX -->|/admin/*| VBEN

    API --> PG
    API --> REDIS
    API --> EMBY
    API --> JF
    API --> STRIPE
```

## 2. 用户认证流程

### 2.1 Web 登录

```mermaid
sequenceDiagram
    participant U as 用户浏览器
    participant P as Portal (Nuxt)
    participant A as API (FastAPI)
    participant DB as PostgreSQL

    U->>P: 访问 /login
    P->>U: 渲染登录页面
    U->>A: POST /api/v1/auth/login<br/>{username, password}
    A->>DB: 查询用户 + 验证密码
    alt 认证成功
        A->>A: 生成 JWT (access_token + refresh_token)
        A->>U: {access_token, refresh_token, user}
        U->>U: localStorage 存储 token
        U->>P: 跳转 /portal
    else 认证失败
        A->>U: 401 Invalid credentials
    end
```

### 2.2 Telegram MiniApp 认证

```mermaid
sequenceDiagram
    participant TG as Telegram Client
    participant MA as MiniApp (Vue SPA)
    participant A as API (FastAPI)
    participant DB as PostgreSQL

    TG->>MA: 打开 WebApp (initData)
    MA->>MA: 解析 initData (tg_uid, hash)
    MA->>A: POST /api/v1/auth/telegram-login<br/>{initData}
    A->>A: 验证 HMAC 签名
    A->>DB: 查找/创建用户 (by tg_uid)
    alt 用户存在
        A->>A: 生成 JWT
        A->>MA: {access_token, user}
    else 新用户
        A->>DB: 创建用户 (auto-register)
        A->>A: 生成 JWT
        A->>MA: {access_token, user}
    end
    MA->>MA: 存储 session, 加载主界面
```

## 3. Telegram MiniApp 启动流程

```mermaid
flowchart TD
    START([MiniApp 打开]) --> CHECK_TG{是否在 Telegram 中?}
    CHECK_TG -->|否| NOT_TG[显示 NOT_TELEGRAM 页面]
    CHECK_TG -->|是| PARSE[解析 initData]
    PARSE --> CHECK_SESSION{本地有 session?}
    CHECK_SESSION -->|是| VERIFY[验证 session 有效性]
    CHECK_SESSION -->|否| TG_LOGIN[Telegram 自动登录]
    VERIFY -->|有效| LOAD_HOME[加载首页数据]
    VERIFY -->|过期| REFRESH{有 refresh_token?}
    REFRESH -->|是| DO_REFRESH[刷新 token]
    REFRESH -->|否| TG_LOGIN
    DO_REFRESH -->|成功| LOAD_HOME
    DO_REFRESH -->|失败| TG_LOGIN
    TG_LOGIN --> API_LOGIN[POST /auth/telegram-login]
    API_LOGIN -->|成功| SAVE_SESSION[保存 session]
    API_LOGIN -->|失败| ERROR[显示错误页面]
    SAVE_SESSION --> LOAD_HOME
    LOAD_HOME --> PARALLEL[并行加载]
    PARALLEL --> A1[公告 /announcements]
    PARALLEL --> A2[配额 /user/quota]
    PARALLEL --> A3[版本 /public/version]
    PARALLEL --> A4[品牌 /public/branding]
    PARALLEL --> A5[日历 /user/checkin/calendar]
    PARALLEL --> A6[菜单 /miniapp/menu]
    A1 & A2 & A3 & A4 & A5 & A6 --> RENDER([渲染主界面])
```

## 4. 签到系统

### 4.1 每日签到

```mermaid
flowchart TD
    START([用户点击签到]) --> CHECK_AUTH{已登录?}
    CHECK_AUTH -->|否| REJECT[拒绝]
    CHECK_AUTH -->|是| API[POST /user/checkin]
    API --> RISK{风控检查}
    RISK -->|拦截| BLOCK[返回风控错误]
    RISK -->|通过| DUP{今日已签?}
    DUP -->|是| DUP_ERR[CHECKIN_DUPLICATE]
    DUP -->|否| CALC[计算积分]
    CALC --> EVENT{活动期间?}
    EVENT -->|是| EVT_PTS[event_points_min ~ max]
    EVENT -->|否| NRM_PTS[points_min ~ max]
    EVT_PTS & NRM_PTS --> AWARD[random.randint 积分]
    AWARD --> STREAK[更新连签天数]
    STREAK --> LOG[写入 CheckinLog]
    LOG --> ADD_PTS[用户积分 += awarded]
    ADD_PTS --> RESP([返回 streak + points_awarded])
```

### 4.2 补签流程

```mermaid
flowchart TD
    START([用户选择过去日期]) --> CHECK{canMakeup?}
    CHECK -->|否| IGNORE[不可操作]
    CHECK -->|是| MODAL[显示补签确认弹窗]
    MODAL --> CONFIRM[确认补签]
    CONFIRM --> API[POST /user/checkin/makeup]
    API --> RISK{风控检查}
    RISK -->|拦截| BLOCK[风控错误]
    RISK -->|通过| LIMIT{次数检查}
    LIMIT -->|超限| LIMIT_ERR[日/周/月限额]
    LIMIT -->|可用| POINTS{积分足够?}
    POINTS -->|不足| INSUFF[积分不足]
    POINTS -->|足够| DEDUCT[扣除补签积分]
    DEDUCT --> AWARD[发放补签奖励积分<br/>min ~ max 随机]
    AWARD --> LOG[写入 CheckinLog<br/>ip_address=user_makeup]
    LOG --> RESP([返回 cost + awarded])
```

## 5. 商城购买流程

```mermaid
flowchart TD
    START([用户选择商品]) --> TYPE{支付方式}
    TYPE -->|积分| PTS_CHECK{积分足够?}
    TYPE -->|余额 Credits| CRD_CHECK{余额足够?}
    TYPE -->|加密货币| CRYPTO[创建支付链接]

    PTS_CHECK -->|否| INSUFF_P[积分不足]
    PTS_CHECK -->|是| PTS_PAY[扣除积分]
    CRD_CHECK -->|否| INSUFF_C[余额不足]
    CRD_CHECK -->|是| CRD_PAY[扣除余额]

    PTS_PAY & CRD_PAY --> ORDER[创建订单 Order]
    CRYPTO --> WAIT[等待支付确认]
    WAIT -->|超时| CANCEL[取消订单]
    WAIT -->|确认| ORDER

    ORDER --> PRODUCT{商品类型}
    PRODUCT -->|席位卡| SEAT[创建/续期 Seat]
    PRODUCT -->|积分包| ADD_PTS[增加用户积分]
    PRODUCT -->|余额包| ADD_CRD[增加用户余额]
    PRODUCT -->|CDKey| GEN_KEY[生成 CDKey]

    SEAT & ADD_PTS & ADD_CRD & GEN_KEY --> AFF{有推广人?}
    AFF -->|是| COMMISSION[计算推广佣金<br/>仅 Credits/Crypto]
    AFF -->|否| DONE
    COMMISSION --> DONE([订单完成])
```

## 6. 媒体账号管理

```mermaid
flowchart TD
    START([媒体账号页面]) --> LIST[GET /user/media-accounts]
    LIST --> DISPLAY[显示账号列表]
    DISPLAY --> ACTION{用户操作}

    ACTION -->|创建账号| CREATE_MODAL[创建账号弹窗]
    CREATE_MODAL --> SELECT_SEAT[选择席位]
    SELECT_SEAT --> SELECT_SVC[选择媒体服务]
    SELECT_SVC --> SET_USER[设置用户名]
    SET_USER --> POST_CREATE[POST /user/media-accounts]
    POST_CREATE --> PROVIDER{服务商类型}
    PROVIDER -->|Emby| EMBY_API[Emby API 创建用户]
    PROVIDER -->|Jellyfin| JF_API[Jellyfin API 创建用户]
    EMBY_API & JF_API --> SAVE[保存到数据库]
    SAVE --> TEMPLATE[应用权限模板]
    TEMPLATE --> REFRESH[刷新列表]

    ACTION -->|修改密码| PWD_MODAL[修改密码弹窗]
    PWD_MODAL --> PWD_API[POST .../change-password]
    PWD_API --> PROVIDER2{服务商}
    PROVIDER2 --> SYNC[同步到 Emby/Jellyfin]

    ACTION -->|CDKey 续期| CDK_MODAL[CDKey 续期弹窗]
    CDK_MODAL --> REDEEM[POST /shop/cdkey/redeem]
    REDEEM --> EXTEND[延长账号到期时间]
```

## 7. 席位管理

```mermaid
flowchart TD
    SEAT_PAGE([席位管理页面]) --> FETCH[GET /shop/seats]
    FETCH --> DISPLAY[显示席位列表]

    subgraph 席位信息
        NAME[席位名称]
        QUOTA[配额: 已用/总量]
        EXPIRY[到期时间]
        STATUS[状态: active/expired]
    end

    DISPLAY --> REDEEM[CDKey 兑换]
    REDEEM --> INPUT[输入 CDKey]
    INPUT --> API[POST /shop/cdkey/redeem]
    API --> VALID{CDKey 有效?}
    VALID -->|否| ERR[错误提示]
    VALID -->|是| APPLY{CDKey 类型}
    APPLY -->|新席位| NEW[创建新 Seat]
    APPLY -->|续期| EXTEND[延长到期时间]
    APPLY -->|配额| ADD_Q[增加配额]
    NEW & EXTEND & ADD_Q --> DONE([刷新列表])
```

## 8. 邀请与推广返利

```mermaid
flowchart TD
    subgraph 邀请码系统
        GEN[生成邀请码] --> CODE[唯一邀请码]
        CODE --> SHARE[分享给朋友]
        SHARE --> REGISTER[新用户注册时填写]
        REGISTER --> BIND[绑定邀请关系]
    end

    subgraph 推广返利
        BIND --> TREE[邀请树 InviteTree]
        TREE --> PURCHASE[被邀请人购买]
        PURCHASE --> CHECK{付费方式}
        CHECK -->|积分| NO_COMM[不产生佣金]
        CHECK -->|Credits/Crypto| CALC[计算佣金比例]
        CALC --> L1[一级返利 %]
        CALC --> L2[二级返利 %]
        L1 & L2 --> CREDIT[佣金入账<br/>推广人余额]
    end
```

## 9. 成长等级系统

```mermaid
flowchart TD
    subgraph 经验获取
        CHECKIN[签到获取积分]
        PURCHASE[购买获取积分]
        INVITE[邀请获取积分]
    end

    CHECKIN & PURCHASE & INVITE --> POINTS[用户积分池]
    POINTS --> EXCHANGE[积分兑换经验]
    EXCHANGE --> INPUT[输入兑换数量]
    INPUT --> CONVERT[points -= amount<br/>exp += amount * rate]
    CONVERT --> CHECK{经验达到升级线?}
    CHECK -->|是| LEVEL_UP[升级!]
    CHECK -->|否| KEEP[保持当前等级]

    LEVEL_UP --> ROLE{匹配成长角色}
    ROLE --> GR[Growth Role 匹配<br/>level_min ≤ level ≤ level_max]
    GR --> PERMS[解锁对应权益]
    PERMS --> MENU[菜单权限更新]
    PERMS --> QUOTA[配额提升]
    PERMS --> BADGE[称号变更]
```

## 10. 工单系统

```mermaid
flowchart TD
    subgraph 用户端
        CREATE[创建工单] --> FILL[填写主题+内容]
        FILL --> SUBMIT[POST /user/tickets]
        SUBMIT --> OPEN[状态: open]
        OPEN --> REPLY_U[用户回复<br/>POST .../reply]
    end

    subgraph 管理端
        OPEN --> ADMIN_VIEW[管理员查看]
        ADMIN_VIEW --> REPLY_A[管理员回复]
        REPLY_A --> CLOSE_A[关闭工单]
    end

    subgraph MiniApp 工单
        VIEW_LIST[工单列表] --> CLICK[点击查看详情]
        CLICK --> DETAIL[GET /user/tickets/:id]
        DETAIL --> MSGS[显示消息记录]
        MSGS --> CAN_REPLY{工单未关闭?}
        CAN_REPLY -->|是| INPUT_REPLY[输入回复]
        CAN_REPLY -->|否| CLOSED_NOTICE[已关闭提示]
        INPUT_REPLY --> SEND[POST .../reply]
    end
```

## 11. CDKey 兑换流程

```mermaid
flowchart TD
    START([输入 CDKey]) --> API[POST /shop/cdkey/redeem]
    API --> FIND{查找 CDKey}
    FIND -->|不存在| ERR1[无效 CDKey]
    FIND -->|已使用| ERR2[已被使用]
    FIND -->|已过期| ERR3[已过期]
    FIND -->|有效| TYPE{CDKey 类型}

    TYPE -->|seat_new| CREATE_SEAT[创建新席位]
    TYPE -->|seat_extend| EXTEND_SEAT[续期席位]
    TYPE -->|seat_quota| ADD_QUOTA[增加配额]
    TYPE -->|points| ADD_POINTS[增加积分]
    TYPE -->|credits| ADD_CREDITS[增加余额]
    TYPE -->|media_extend| EXTEND_MEDIA[延长媒体账号]

    CREATE_SEAT & EXTEND_SEAT & ADD_QUOTA & ADD_POINTS & ADD_CREDITS & EXTEND_MEDIA --> MARK[标记 CDKey 已使用]
    MARK --> LOG[记录兑换日志]
    LOG --> DONE([返回兑换结果])
```

## 12. 菜单权限控制 (L2)

```mermaid
flowchart TD
    subgraph 后端配置
        MENU_SYNC[menu_sync.py<br/>PORTAL_MENU_DEFINITIONS] --> DB[(miniapp_menu_settings)]
        DB --> ENABLED{menu.enabled?}
    end

    subgraph 请求处理
        REQ[GET /miniapp/menu] --> AUTH[验证用户身份]
        AUTH --> LEVEL[获取用户等级/角色]
        LEVEL --> FILTER[过滤可见菜单]
        FILTER --> ENABLED
        ENABLED -->|启用| CHECK_ROLE{角色权限匹配?}
        ENABLED -->|禁用| HIDE[隐藏菜单项]
        CHECK_ROLE -->|是| SHOW[返回菜单项]
        CHECK_ROLE -->|否| HIDE
    end

    subgraph 前端渲染
        SHOW --> RENDER[渲染侧栏菜单]
        RENDER --> CLICK[用户点击]
        CLICK --> GUARD{路由守卫}
        GUARD -->|有权限| PAGE[加载页面]
        GUARD -->|无权限| DENIED[访问被拒绝]
    end
```

## 13. Docker 构建与发布

```mermaid
flowchart TD
    subgraph 触发
        TAG[推送 Git Tag v1.x.x]
        MANUAL[手动 workflow_dispatch]
    end

    TAG & MANUAL --> PREPARE[确定版本号]

    PREPARE --> BUILD_API[构建 API 镜像]
    PREPARE --> BUILD_PORTAL[构建 Portal 镜像]
    PREPARE --> BUILD_VBEN[构建 Vben 镜像]

    subgraph API 构建
        BUILD_API --> RUST[编译 Rust 扩展]
        RUST --> PIP[安装 Python 依赖]
        PIP --> COMPILE[编译 .py → .pyc]
        COMPILE --> STRIP[删除 .py 源码]
    end

    subgraph Portal 构建
        BUILD_PORTAL --> PNPM_P[pnpm install]
        PNPM_P --> NUXT[nuxt build]
        NUXT --> ONLY_OUTPUT[仅保留 .output/]
    end

    subgraph Vben 构建
        BUILD_VBEN --> PNPM_V[pnpm install]
        PNPM_V --> VITE[vite build]
        VITE --> ONLY_DIST[仅保留 dist/]
    end

    STRIP & ONLY_OUTPUT & ONLY_DIST --> PUSH[推送到 GHCR]
    PUSH --> TAG_IMG[标签: v1.1.35 / stable / latest]
    TAG_IMG --> SANITY[镜像安全验证<br/>无源码检查]
    SANITY --> SCAN[Trivy 漏洞扫描]
    SCAN --> SBOM[生成 SBOM]
    SBOM --> DONE([发布完成])
```

## 14. 部署流程

```mermaid
flowchart TD
    subgraph CI/CD
        RELEASE[GitHub Release] --> GHCR[GHCR 镜像就绪]
    end

    subgraph 生产服务器
        GHCR --> PULL[docker pull 新镜像]
        PULL --> COMPOSE[docker compose up -d]
        COMPOSE --> HEALTH[健康检查]
        HEALTH -->|通过| SMOKE[冒烟测试]
        HEALTH -->|失败| ROLLBACK[回滚到 last_good_tag]
        SMOKE -->|通过| UPDATE_TAG[更新 last_good_tag]
        SMOKE -->|失败| ROLLBACK
        UPDATE_TAG --> DONE([部署完成])
        ROLLBACK --> NOTIFY[通知管理员]
    end
```

---

## 附录：组件关系图

```mermaid
graph LR
    subgraph 前端
        PORTAL[Portal<br/>Nuxt 3 + Nuxt UI]
        MINIAPP[MiniApp<br/>Vue SPA in Portal]
        VBEN[Admin<br/>Vue Vben Admin]
    end

    subgraph 后端 API
        AUTH[auth 模块]
        USER[user 模块]
        SHOP[shop 模块]
        MEDIA[media 模块]
        CHECKIN[checkin 模块]
        TICKET[ticket 模块]
        ADMIN_API[admin 模块]
    end

    subgraph 数据模型
        USERS[(Users)]
        SEATS[(Seats)]
        ORDERS[(Orders)]
        MEDIA_ACC[(MediaAccounts)]
        CHECKIN_LOG[(CheckinLog)]
        CDKEYS[(CDKeys)]
        TICKETS[(Tickets)]
        INVITE_CODES[(InviteCodes)]
        GROWTH_ROLES[(GrowthRoles)]
    end

    PORTAL --> AUTH & USER & SHOP & CHECKIN & TICKET
    MINIAPP --> AUTH & USER & SHOP & CHECKIN & MEDIA
    VBEN --> ADMIN_API

    AUTH --> USERS
    USER --> USERS & GROWTH_ROLES
    SHOP --> ORDERS & CDKEYS & SEATS
    MEDIA --> MEDIA_ACC & SEATS
    CHECKIN --> CHECKIN_LOG & USERS
    TICKET --> TICKETS
    ADMIN_API --> USERS & SEATS & ORDERS & MEDIA_ACC
```

---

<a id="english-version"></a>
## English Version

> This document contains Mermaid flowcharts for all core UHDadmin features. See the Chinese sections above for diagrams — Mermaid syntax is language-agnostic and can be rendered in any Markdown viewer that supports it.
>
> For English documentation, see:
> - [ARCHITECTURE.en.md](./ARCHITECTURE.en.md) — System architecture
> - [CHANGELOG.en.md](./CHANGELOG.en.md) — Version history
> - [README.en.md](../README.en.md) — Product overview
