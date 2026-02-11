# Round 1.2.0: 配置方案系统 (Config Profile System)

> 创建时间: 2026-02-04
> 状态: 开发中

---

## 问题分析

### 当前架构缺陷

```
基础数据（白名单、URI规则、限流…） ← 全局共享，按 service_type 区分
        │
        │ 所有同类型 slave 读同一份规则
        ▼
edgea (公共免费服)  ─┐
edgeb (VIP付费服)   ─┤──→ 完全相同的规则 ← BUG: 无法差异化
micro (测试服)     ─┘
```

**具体问题:**

1. **规则无法差异化**: `_build_lua_config()` 按 `service_type=emby` 查询，3台 Emby 服拿到完全相同的白名单、URI 规则、限流配置
2. **配置向导无实际意义**: 向导生成的快照（已保存配置）只是存档给人看的，Slave 不使用它
3. **限流无法按服分配**: 限流与并发配置也是全局的，无法给不同服设置不同限流
4. **多服务器 lua 无法区分**: 一个 Slave Docker 跑 3 个 server block，共用一个 access.lua，但每个服需要不同规则

### 期望效果

- node-a.example.com: 严格封禁 + 重度限流（公共免费服）
- node-b.example.com: 宽松规则 + 轻度限流（VIP 付费服）
- node-c.example.com: 无限制（测试服）

---

## 架构设计

### 整体流程

```
基础数据 = 规则池（所有可用的规则在这里创建/管理）
                │
                │ 配置向导从中挑选
                ▼
┌──────────────────────────────────────────────┐
│          配置方案 (Config Profile)             │
│                                              │
│  "公共服严格方案"                              │
│    ├─ 白名单: [Emby Theater, Infuse]          │
│    ├─ URI封禁: [CameraUploads, SyncPlay, ...] │
│    ├─ URI跳过: [Videos/stream, Audio/stream]  │
│    ├─ 限流: 100 req/min, 50GB/月              │
│    └─ 伪造人数: 888                           │
│                                              │
│  "VIP服宽松方案"                              │
│    ├─ 白名单: [全部客户端]                     │
│    ├─ URI封禁: [CameraUploads]                │
│    ├─ 限流: 1000 req/min, 无配额              │
│    └─ 伪造人数: 关闭                          │
└──────────┬───────────────────────────────────┘
           │ 分配给 Slave
           ▼
┌─ 主机配置 (203.0.113.10) ──────────────────────┐
│                                                │
│  edgea  → config_profile: "公共服严格方案"       │
│  edgeb  → config_profile: "VIP服宽松方案"        │
│  micro → config_profile: "测试服无限制"         │
│                                                │
│  [也支持一键全部用同一方案]                      │
└────────────────────┬───────────────────────────┘
                     │
                     ▼ Slave 每30s拉取
┌────────────────────────────────────────────────┐
│  GET /config 返回:                              │
│                                                │
│  rendered_nginx:                               │
│    server node-a.example.com {                   │
│      set $config_profile_id "1";               │
│      access_by_lua_file access.lua;            │
│    }                                           │
│    server node-b.example.com {                   │
│      set $config_profile_id "2";               │
│    }                                           │
│                                                │
│  lua_configs: {                                │
│    "1": { block_list:[...], rate:100/min },    │
│    "2": { block_list:[...], rate:1000/min },   │
│    "3": { block_list:[], rate:null }           │
│  }                                             │
└────────────────────────────────────────────────┘
```

### 一个 access.lua 多份配置数据

**核心原理**: lua 代码是逻辑，配置数据是参数。逻辑不变，参数按 profile 切换。

```
请求 → node-a.example.com
         │
         ▼
┌─ server block (edgea) ─────────────┐
│  set $config_profile_id "1";      │  ← nginx 变量标识方案
│  access_by_lua_file access.lua;   │
└────────────────┬──────────────────┘
                 │
                 ▼
┌─ access.lua (同一个文件) ─────────────────────────┐
│                                                   │
│  local pid = ngx.var.config_profile_id  -- "1"    │
│  local cfg = shared_dict:get("cfg:" .. pid)       │
│                                                   │
│  -- 8 步检查链代码完全不变                          │
│  -- 但 cfg.block_list / cfg.allowed_clients 等     │
│  -- 是 profile "1" 专属的数据                       │
└───────────────────────────────────────────────────┘
```

---

## 数据模型

### 新增: MediaConfigProfile

```python
class MediaConfigProfile(Model):
    """配置方案 — 从基础数据规则池中选择一组规则"""

    class Meta:
        table = "media_config_profiles"

    id = fields.IntField(pk=True)
    name = fields.CharField(max_length=100)              # 方案名称
    service_type = fields.CharField(max_length=20)       # emby / jellyfin
    description = fields.TextField(default="")           # 描述

    # 选中的规则 ID 列表 (引用基础数据中的规则)
    whitelist_ids = fields.JSONField(default=[])         # → MediaPlayerWhitelist.id[]
    uri_rule_ids = fields.JSONField(default=[])          # → MediaUriRuleAssignment.id[]
    location_rule_ids = fields.JSONField(default=[])     # → MediaLocationRule.id[]
    nginx_map_ids = fields.JSONField(default=[])         # → MediaNginxMap.id[]
    custom_header_ids = fields.JSONField(default=[])     # → MediaCustomHeader.id[]

    # 限流配置 (内嵌，不引用外部表)
    rate_limit_config = fields.JSONField(default={})
    # 结构: {
    #   "req_per_second": 10,
    #   "req_per_minute": 100,
    #   "burst": 20,
    #   "quota_requests": 100000,   # 月请求配额
    #   "quota_bandwidth_gb": 50,   # 月带宽配额 GB
    #   "max_concurrent_streams": 3,
    # }

    # 杂项设置
    misc_settings = fields.JSONField(default={})
    # 结构: {
    #   "fake_counts_enabled": false,
    #   "fake_counts_value": 888,
    #   "deny_message": "...",
    # }

    # 元信息
    enabled = fields.BooleanField(default=True)
    created_by_id = fields.IntField(null=True)
    created_at = fields.DatetimeField(auto_now_add=True)
    updated_at = fields.DatetimeField(auto_now=True)
```

### 修改: MediaServiceSlave

```python
# 新增字段
config_profile_id = fields.IntField(null=True)  # FK → MediaConfigProfile.id
# null = 使用全局默认（向后兼容）
```

### 关系图

```
MediaPlayerWhitelist ──┐
MediaUriRuleAssignment ┤
MediaLocationRule ──────┤──→ MediaConfigProfile ──→ MediaServiceSlave
MediaNginxMap ──────────┤         (选择规则子集)        (每个服引用一个方案)
MediaCustomHeader ──────┘
```

---

## 改动清单

### Phase 1: 数据模型 + 迁移

| # | 文件 | 操作 | 说明 |
|---|------|------|------|
| 1.1 | `app/models/media_access.py` | EDIT | 新增 `MediaConfigProfile` 模型 |
| 1.2 | `app/models/media_access.py` | EDIT | `MediaServiceSlave` 增加 `config_profile_id` 字段 |
| 1.3 | `migrations/` | NEW | 新增迁移: 建表 + 加字段 |

### Phase 2: 后端 API

| # | 文件 | 操作 | 说明 |
|---|------|------|------|
| 2.1 | `app/routers/admin/media_access_control.py` | EDIT | CRUD 端点: 配置方案列表/创建/更新/删除 |
| 2.2 | `app/routers/admin/media_access_control.py` | EDIT | 主机配置: Slave 绑定 profile |
| 2.3 | `app/routers/media_slave_api.py` | EDIT | `GET /config`: 按 profile 构建 per-slave lua_config |
| 2.4 | `app/services/media_access_config_generator.py` | EDIT | server block 生成 `set $config_profile_id` |

### Phase 3: 前端

| # | 文件 | 操作 | 说明 |
|---|------|------|------|
| 3.1 | `vue-vben-admin/.../ConfigWizard.vue` (或类似) | REWRITE | 改为创建/编辑配置方案（非快照）|
| 3.2 | `vue-vben-admin/.../SavedConfigsPanel.vue` | REWRITE | 改为「配置方案」列表管理 |
| 3.3 | `vue-vben-admin/.../HostConfigPanel.vue` | EDIT | 每个 Slave 行增加「配置方案」下拉选择器 |
| 3.4 | `vue-vben-admin/.../api/media-access.ts` | EDIT | 新增 API 调用 |

### Phase 4: Slave 端 (Lua)

| # | 文件 | 操作 | 说明 |
|---|------|------|------|
| 4.1 | `UHDadmin-media-slave/lua/access.lua` | EDIT | 读 `ngx.var.config_profile_id`，从 shared_dict 加载对应配置 |
| 4.2 | `UHDadmin-media-slave/lua/init_worker.lua` | EDIT | 存储多份 lua_config 到 shared_dict (按 profile_id 分 key) |

---

## 详细设计

### Slave API 返回结构变化

**改前 (当前):**
```json
{
  "mode": "multi_server",
  "lua_config": { "allowed_clients": [...], ... },
  "rendered_nginx": "...",
  "rendered_lua": "..."
}
```
lua_config 是全局共享的一份，3 台服用同一份。

**改后:**
```json
{
  "mode": "multi_server",
  "lua_configs": {
    "1": {
      "profile_name": "公共服严格方案",
      "allowed_clients": ["Emby Theater", "Infuse"],
      "min_versions": {"Infuse": "7.0"},
      "block_list": [{"pattern": "/CameraUploads", "match_type": "prefix"}, ...],
      "skip_list": [{"pattern": "/Videos/.*/stream", "match_type": "regex"}, ...],
      "allow_list": [],
      "fake_counts_enabled": true,
      "fake_counts_value": 888,
      "deny_message": "请使用允许的客户端",
      "rate_limit": {
        "req_per_second": 10,
        "req_per_minute": 100,
        "quota_requests": 100000,
        "quota_bandwidth_gb": 50,
        "max_concurrent_streams": 3
      }
    },
    "2": {
      "profile_name": "VIP服宽松方案",
      "allowed_clients": ["*"],
      "block_list": [{"pattern": "/CameraUploads", "match_type": "prefix"}],
      "rate_limit": {
        "req_per_minute": 1000,
        "max_concurrent_streams": 10
      },
      ...
    }
  },
  "rendered_nginx": "...(含 set $config_profile_id)...",
  "rendered_lua": "...",
  "sibling_count": 3
}
```

保留 `lua_config` 字段（值 = 第一个 profile 的配置）用于向后兼容旧版 Slave。

### Nginx 配置生成变化

每个 server block 头部增加:
```nginx
server {
    listen 443 ssl;
    server_name node-a.example.com;

    # Config Profile ID for Lua access check
    set $config_profile_id "1";

    # ... 其余不变 ...
    access_by_lua_file /opt/slave/lua/access.lua;
}
```

### access.lua 改动

```lua
-- ===== 改动点: 开头加载配置的方式 =====

-- 改前:
local config_json = shared_dict:get("lua_config")
local config = cjson.decode(config_json)

-- 改后:
local profile_id = ngx.var.config_profile_id or "default"
local config_json = shared_dict:get("cfg:" .. profile_id)
if not config_json then
    -- fallback: 尝试全局配置 (向后兼容无 profile 的旧模式)
    config_json = shared_dict:get("lua_config")
end
local config = config_json and cjson.decode(config_json) or {}

-- ===== 后续 8 步检查链代码完全不变 =====
```

### init_worker.lua 改动

```lua
-- pull_config() 中存储配置的部分

-- 改前:
shared_dict:set("lua_config", cjson.encode(cfg_data.lua_config))

-- 改后:
if cfg_data.lua_configs then
    -- 多 profile 模式
    for pid, pcfg in pairs(cfg_data.lua_configs) do
        shared_dict:set("cfg:" .. pid, cjson.encode(pcfg))
    end
    -- 向后兼容: 也存一份全局的 (取第一个)
    local first_key = next(cfg_data.lua_configs)
    if first_key then
        shared_dict:set("lua_config", cjson.encode(cfg_data.lua_configs[first_key]))
    end
else
    -- 旧模式 (单 profile)
    shared_dict:set("lua_config", cjson.encode(cfg_data.lua_config))
end
```

---

## 配置向导 → 配置方案 改造

### 改前流程
```
配置向导 → 勾选规则 → 生成快照 → 存到「已保存配置」(冻结的副本)
                                         ↓
                                     Slave 不用它
```

### 改后流程
```
配置向导 → 勾选规则 → 保存为「配置方案」(引用规则ID，活的)
                              ↓
                      在「主机配置」里分配给 Slave
                              ↓
                      Slave pull 时按方案构建配置
```

### 配置方案管理 UI

「已保存配置」Tab 改名为「配置方案」:
- 列表显示所有方案 (名称、服务类型、引用规则数、关联 Slave 数)
- 创建/编辑: 复用配置向导的步骤式表单
- 删除: 检查是否有 Slave 引用，提示确认
- 复制: 快速基于现有方案创建变体

### 主机配置 UI

每个 Slave 行增加:
```
EXAMPLE-1服 (edgea)
node-a.example.com  :39685
配置方案: [公共服严格方案 ▼]  编辑  删除
```

主机级快捷操作:
```
203.0.113.10  [3个服务]  [+ 添加]
[统一设置方案: 选择方案 ▼]  ← 一键给本主机所有服设同一方案
```

---

## 向后兼容

| 场景 | 处理方式 |
|------|----------|
| Slave 无 config_profile_id (null) | 使用全局默认规则（当前行为） |
| 旧版 Slave agent (不认识 lua_configs) | 保留 `lua_config` 字段 = 第一个 profile |
| 旧版 access.lua (不读 $config_profile_id) | shared_dict 仍存 `lua_config` 全局 key |
| 迁移: 已有 Slave 记录 | config_profile_id 默认 null，行为不变 |

---

## 限流与并发的整合

限流配置直接内嵌在 ConfigProfile 的 `rate_limit_config` JSON 字段中，不单独建表。

原因:
- 限流参数和访问规则是同一个"行为方案"的一部分
- 不同服的限流差异通常和访问规则差异一起变化
- 内嵌 JSON 更简单，不需要额外的关联关系

如果未来需要"限流模板"复用，可以在 UI 层做 preset 选择器，后端仍存到 profile 的 JSON 字段里。

---

## 验证步骤

1. 创建 3 个配置方案: 严格/宽松/无限制
2. 在主机配置中分别分配给 edgea/edgeb/micro
3. 用 edgea 的 slave token 调用 `GET /config`，确认返回:
   - `lua_configs` 包含 3 个 profile 的数据
   - `rendered_nginx` 包含 `set $config_profile_id "1"/"2"/"3"`
4. 在 Slave Docker 中验证:
   - init_worker 正确存储 3 份配置到 shared_dict
   - access.lua 根据请求域名应用不同规则
5. 修改基础数据中的一条 URI 规则 → 等 30s → Slave 拉取到更新后的规则（验证"活的"不是"冻结的"）
