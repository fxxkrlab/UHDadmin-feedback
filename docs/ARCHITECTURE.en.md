# UHDadmin Architecture Document

[中文文档](ARCHITECTURE.zh-CN.md)

This document describes UHDadmin's system architecture, data flow, deployment patterns, and core design decisions.

---

## Table of Contents

- [1. System Overview](#1-system-overview)
- [2. Architecture Diagrams (Mermaid)](#2-architecture-diagrams-mermaid)
- [3. Service Components](#3-service-components)
- [4. Data Flow](#4-data-flow)
- [5. Deployment Architecture](#5-deployment-architecture)
- [6. Blue/Green & Canary Releases](#6-bluegreen--canary-releases)
- [7. Security Architecture](#7-security-architecture)
- [8. Scalability Design](#8-scalability-design)

---

## 1. System Overview

UHDadmin is a full-stack user management and subscription service platform using a **microservice + monolithic backend** hybrid architecture:

| Feature | Description |
|---------|-------------|
| **Gateway Layer** | APISIX 3.8 provides L7 routing, blue/green releases, canary traffic splitting |
| **Backend** | Python FastAPI monolith with modular design |
| **Frontend** | Vue 3 (Vben Admin) + Nuxt 3 (Portal) dual frontend |
| **Data Layer** | PostgreSQL 15 + Redis 7 |
| **External Integration** | Telegram Bot API, Payment gateways |
| **Deployment** | Docker + Portainer Stack + GHCR |

---

## 2. Architecture Diagrams (Mermaid)

### 2.1 Overall System Architecture

```mermaid
graph TB
    subgraph Internet["Internet Entry"]
        USER[User Browser/App]
        TG_USER[Telegram User]
    end

    subgraph Tunnel["Tunnel Layer"]
        CF[Cloudflared Tunnel]
    end

    subgraph Gateway["Gateway Layer"]
        APISIX[APISIX L7 Gateway<br/>Traffic Split / Rate Limit / CORS]
    end

    subgraph Services["Service Layer"]
        subgraph Blue["Blue Slot (stable)"]
            API_B[API Blue<br/>:8000]
            VBEN_B[Vben Blue<br/>:80]
            PORTAL_B[Portal Blue<br/>:3000]
        end

        subgraph Green["Green Slot (canary)"]
            API_G[API Green<br/>:8000]
            VBEN_G[Vben Green<br/>:80]
            PORTAL_G[Portal Green<br/>:3000]
        end
    end

    subgraph Data["Data Layer"]
        PG[(PostgreSQL 15<br/>Primary Database)]
        REDIS[(Redis 7<br/>Cache/Session)]
        ETCD[(etcd<br/>APISIX Config)]
    end

    subgraph External["External Services"]
        TG_API[Telegram Bot API]
        PAY_GW[Payment Gateway<br/>Multi-channel]
    end

    USER --> CF
    TG_USER --> TG_API
    CF --> APISIX

    APISIX -->|"X-Slot: blue"| API_B
    APISIX -->|"X-Slot: green"| API_G
    APISIX --> VBEN_B & VBEN_G
    APISIX --> PORTAL_B & PORTAL_G

    API_B & API_G --> PG
    API_B & API_G --> REDIS
    API_B & API_G --> TG_API
    API_B & API_G --> PAY_GW

    APISIX --> ETCD
```

### 2.2 Backend Module Architecture

```mermaid
graph LR
    subgraph Routers["API Router Layer"]
        AUTH[auth.py<br/>Auth]
        USER[user.py<br/>User]
        SHOP[shop.py<br/>Shop]
        PAY[payment.py<br/>Payment]
        edgea[miniapp.py<br/>MiniApp]

        subgraph Admin["admin/"]
            A_USER[users.py]
            A_ORDER[orders.py]
            A_ROLE[roles.py]
            A_MEDIA[media_services.py]
            A_RELEASE[release.py]
        end
    end

    subgraph Services["Service Layer"]
        SVC_AUTH[Auth Service]
        SVC_ORDER[Order Service]
        SVC_MEDIA[Media Account Service]
        SVC_AFF[Affiliate Service]
        SVC_TG[Telegram Service]
    end

    subgraph Models["Data Model Layer"]
        M_USER[User]
        M_ORDER[Order]
        M_MEDIA[MediaAccount]
        M_ROLE[Role/Permission]
    end

    subgraph Core["Core Modules"]
        DB[database.py<br/>Connection Pool]
        CACHE[cache.py<br/>Redis]
        SEC[security.py<br/>JWT/Encryption]
    end

    AUTH --> SVC_AUTH
    USER --> SVC_AUTH
    SHOP --> SVC_ORDER
    PAY --> SVC_ORDER
    A_MEDIA --> SVC_MEDIA

    SVC_AUTH --> M_USER
    SVC_ORDER --> M_ORDER
    SVC_MEDIA --> M_MEDIA

    M_USER & M_ORDER & M_MEDIA --> DB
    SVC_AUTH --> CACHE
    SVC_AUTH --> SEC
```

### 2.3 Frontend Architecture

```mermaid
graph TB
    subgraph VbenAdmin["Vben Admin (Vue 3)"]
        V_VIEWS[views/<br/>Page Components]
        V_API[api/<br/>Request Wrappers]
        V_STORE[store/<br/>State Management]
        V_ROUTER[router/<br/>Routing]
    end

    subgraph NuxtPortal["Nuxt Portal"]
        N_PAGES[pages/<br/>Pages]
        N_COMP[components/<br/>Components]
        N_COMPOSE[composables/<br/>Composables]
        N_API[api/<br/>Requests]
    end

    subgraph MiniApp["Telegram MiniApp"]
        MINI_WEB[WebApp SDK]
        MINI_UI[Nuxt UI Components]
    end

    V_VIEWS --> V_API
    V_API -->|HTTP| APISIX[APISIX Gateway]

    N_PAGES --> N_API
    N_API -->|HTTP| APISIX

    MINI_WEB --> N_API
```

---

## 3. Service Components

### 3.1 Component List

| Component | Tech Stack | Port | Responsibility |
|-----------|------------|------|----------------|
| **API** | Python 3.11 + FastAPI | 8000 | Backend API service |
| **Vben Admin** | Vue 3 + Ant Design Vue | 80 (nginx) | Admin SPA |
| **Portal** | Nuxt 3 + Nuxt UI | 3000 | User Portal SSR |
| **APISIX** | Apache APISIX 3.8 | 9080/9443 | L7 Gateway |
| **PostgreSQL** | PostgreSQL 15 | 5432 | Primary database |
| **Redis** | Redis 7 | 6379 | Cache/Session |
| **etcd** | etcd 3.5 | 2379 | APISIX config store |

### 3.2 Container Naming Convention

```
uhdadmin-{service}-{slot}-{replica}
```

Examples:
- `uhdadmin-api-blue-1` - API blue slot replica 1
- `uhdadmin-vben-green-1` - Vben green slot replica 1
- `uhdadmin-postgres-1` - PostgreSQL (no slot)

---

## 4. Data Flow

### 4.1 User Purchase Flow

```mermaid
sequenceDiagram
    participant U as User
    participant P as Portal
    participant A as API
    participant DB as PostgreSQL
    participant R as Redis
    participant Pay as Payment Gateway

    U->>P: 1. Browse Products
    P->>A: GET /api/v1/shop/products
    A->>R: Query Cache
    alt Cache Hit
        R-->>A: Return Product List
    else Cache Miss
        A->>DB: SELECT * FROM products
        DB-->>A: Product Data
        A->>R: Write Cache
    end
    A-->>P: Product List JSON
    P-->>U: Render Product Page

    U->>P: 2. Select Plan & Order
    P->>A: POST /api/v1/orders
    A->>DB: INSERT INTO orders
    A->>Pay: Create Payment Order
    Pay-->>A: Payment URL
    A-->>P: Return Payment URL
    P->>U: Redirect to Payment Page

    Pay->>A: 3. Payment Callback
    A->>DB: UPDATE orders SET status='paid'
    A->>DB: Assign Media Account
    A->>R: Clear Related Cache
    A-->>Pay: Callback Confirmation
```

### 4.2 Telegram Login Flow

```mermaid
sequenceDiagram
    participant U as User
    participant TG as Telegram
    participant P as Portal/MiniApp
    participant A as API
    participant DB as PostgreSQL

    U->>TG: 1. Open Bot, Click Login
    TG->>P: With initData
    P->>A: POST /api/v1/auth/telegram
    Note over A: Verify initData Signature
    A->>DB: Query/Create User
    A-->>P: JWT Token
    P-->>U: Login Success, Redirect Home
```

### 4.3 RBAC Permission Check Flow

```mermaid
sequenceDiagram
    participant U as User
    participant A as API
    participant MW as Permission Middleware
    participant DB as PostgreSQL
    participant R as Redis

    U->>A: Request (Bearer Token)
    A->>MW: Permission Check
    MW->>R: Query Permission Cache
    alt Cache Hit
        R-->>MW: Permission List
    else Cache Miss
        MW->>DB: Query User Roles & Permissions
        DB-->>MW: Permission Data
        MW->>R: Write Cache (TTL 5min)
    end

    alt Has Permission
        MW-->>A: Allow
        A-->>U: Normal Response
    else No Permission
        MW-->>U: 403 Forbidden
    end
```

---

## 5. Deployment Architecture

### 5.1 Container Orchestration

```mermaid
graph TB
    subgraph DockerHost["Docker Host"]
        subgraph Network["uhdadmin-network (bridge)"]
            APISIX[apisix:9080]
            ETCD[etcd:2379]

            subgraph BlueStack["Blue Stack"]
                API_B[api-blue:8000]
                VBEN_B[vben-blue:80]
                PORTAL_B[portal-blue:3000]
            end

            subgraph GreenStack["Green Stack"]
                API_G[api-green:8000]
                VBEN_G[vben-green:80]
                PORTAL_G[portal-green:3000]
            end

            PG[postgres:5432]
            REDIS[redis:6379]
        end

        subgraph Volumes["Named Volumes"]
            V_PG[postgres_data]
            V_REDIS[redis_data]
            V_ETCD[etcd_data]
            V_BOOT[boot_config]
        end
    end

    API_B & API_G --> PG
    API_B & API_G --> REDIS
    APISIX --> ETCD

    PG --> V_PG
    REDIS --> V_REDIS
    ETCD --> V_ETCD
    API_B & API_G --> V_BOOT
```

### 5.2 Storage Mode Comparison

| Mode | Pros | Cons | Recommended For |
|------|------|------|-----------------|
| **Named Volumes** | Zero prep, Docker managed | Non-intuitive paths | First deployment, quick testing |
| **Bind Mounts** | Direct access, easy vim/rsync | Requires pre-creation | Production, direct management |

### 5.3 Port Mapping

| External Port | Internal Port | Service | Description |
|---------------|---------------|---------|-------------|
| 9080 | 9080 | APISIX | HTTP entry |
| 9443 | 9443 | APISIX | HTTPS entry |
| 9180 | 9180 | APISIX | Admin API |
| - | 8000 | API | Backend (via APISIX) |
| - | 5432 | PostgreSQL | Database (internal) |
| - | 6379 | Redis | Cache (internal) |

---

## 6. Blue/Green & Canary Releases

### 6.1 Release Modes

```mermaid
graph LR
    subgraph Modes["Release Modes"]
        BLUE[100% Blue<br/>Stable Version]
        CANARY[Blue + Green<br/>Canary Testing]
        GREEN[100% Green<br/>New Version]
    end

    BLUE -->|"traffic.sh canary 10"| CANARY
    CANARY -->|"traffic.sh canary 50"| CANARY
    CANARY -->|"traffic.sh green"| GREEN
    GREEN -->|"rollback_blue.sh"| BLUE
    CANARY -->|"rollback_blue.sh"| BLUE
```

### 6.2 Traffic Split Rules

```mermaid
flowchart TD
    REQ[Request Arrives] --> CHECK_WHITE{User in Allowlist?}
    CHECK_WHITE -->|Yes| GREEN[100% Green]
    CHECK_WHITE -->|No| CHECK_BLACK{User in Denylist?}
    CHECK_BLACK -->|Yes| BLUE[100% Blue]
    CHECK_BLACK -->|No| HASH{Hash by UID}
    HASH -->|"Hash % 100 < Canary %"| GREEN
    HASH -->|"Hash % 100 >= Canary %"| BLUE
```

### 6.3 APISIX Traffic Control

```bash
# Check current status
./deploy/apisix/traffic.sh status

# Switch to 100% blue (stable)
./deploy/apisix/traffic.sh blue

# Canary release 10% traffic to green
./deploy/apisix/traffic.sh canary 10

# Gradually increase canary percentage
./deploy/apisix/traffic.sh canary 25
./deploy/apisix/traffic.sh canary 50

# Full switch to green
./deploy/apisix/traffic.sh green

# Emergency rollback
./deploy/apisix/rollback_blue.sh
```

---

## 7. Security Architecture

### 7.1 Authentication Flow

```mermaid
sequenceDiagram
    participant U as User
    participant A as API
    participant DB as PostgreSQL
    participant R as Redis

    U->>A: POST /auth/login (username/password)
    A->>DB: Query User
    A->>A: Verify Password (bcrypt)
    A->>A: Generate JWT
    A->>R: Store Token Blacklist Check
    A-->>U: {access_token, refresh_token}

    Note over U,A: Subsequent Requests
    U->>A: Request (Authorization: Bearer xxx)
    A->>R: Check Blacklist
    A->>A: Verify JWT Signature
    A-->>U: Response
```

### 7.2 Security Measures

| Layer | Measure | Description |
|-------|---------|-------------|
| **Gateway** | CORS Whitelist | Prohibits `*` wildcard |
| **Gateway** | Rate Limiting | Limit requests by IP/User |
| **Auth** | JWT + bcrypt | Stateless tokens + secure hash |
| **Permissions** | RBAC | Role-Permission-Resource model |
| **Data** | Field Encryption | Sensitive data AES encrypted |
| **Transport** | HTTPS | Cloudflared tunnel TLS |

### 7.3 Sysop Protection

Sysop (System Operator) role has special protection:
- Cannot be deleted
- Cannot be demoted
- Forced operation logging
- Two-factor authentication (optional)

---

## 8. Scalability Design

### 8.1 Horizontal Scaling

```mermaid
graph TB
    subgraph LB["Load Balancer (APISIX)"]
        APISIX[APISIX]
    end

    subgraph APICluster["API Cluster"]
        API1[API-1]
        API2[API-2]
        API3[API-N]
    end

    subgraph DBCluster["Database Cluster"]
        PG_M[(Primary)]
        PG_R1[(Replica 1)]
        PG_R2[(Replica N)]
    end

    subgraph Cache["Cache Cluster"]
        REDIS_M[Redis Master]
        REDIS_R[Redis Replica]
    end

    APISIX --> API1 & API2 & API3
    API1 & API2 & API3 --> PG_M
    API1 & API2 & API3 -.->|Read| PG_R1 & PG_R2
    API1 & API2 & API3 --> REDIS_M
    REDIS_M --> REDIS_R
```

### 8.2 Extension Points

| Extension Point | Implementation | Description |
|-----------------|----------------|-------------|
| **Payment Channels** | `app/services/payment/` | Implement PaymentProvider interface |
| **Media Providers** | `app/services/media/` | Implement MediaProvider interface |
| **Notification Channels** | `app/services/notification/` | Implement NotificationChannel interface |
| **Third-party Login** | `app/routers/auth.py` | Add OAuth2 providers |

---

## Related Documentation

- [Installation Guide](INSTALL.md)
- [Deployment Runbook](DEPLOY_RUNBOOK.md)
- [Environment Variable Reference](ENV_REFERENCE.en.md)
- [Configuration Guide](BOOT_RUNTIME_CONFIG.md)
- [README](../README.en.md)
