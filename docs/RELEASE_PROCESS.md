# RELEASE_PROCESS — 版本发布流程

> 本文档定义 UHDadmin 的版本规则、发布流程和 Tag 命名规范。

---

## 1. 版本规则 (Semantic Versioning)

UHDadmin 遵循 [语义化版本 2.0.0](https://semver.org/lang/zh-CN/) 规范：

```
MAJOR.MINOR.PATCH
  │     │     └── 修订号：向后兼容的 Bug 修复
  │     └──────── 次版本号：向后兼容的功能新增
  └────────────── 主版本号：不兼容的 API 变更
```

### 1.1 版本号递增规则

| 变更类型 | 版本递增 | 示例 |
|----------|----------|------|
| **Breaking Change** | MAJOR +1 | API 端点移除/重命名、数据库 Schema 不兼容 |
| **New Feature** | MINOR +1 | 新增 API 端点、新功能模块 |
| **Bug Fix** | PATCH +1 | 修复 Bug、安全补丁、文档更新 |
| **Pre-release** | 后缀 | `v2.0.0-alpha.1`, `v2.0.0-beta.2`, `v2.0.0-rc.1` |

### 1.2 版本号示例

| 版本号 | 说明 |
|--------|------|
| `v1.0.0` | 首个稳定版本 |
| `v1.1.0` | 新增功能（向后兼容） |
| `v1.1.1` | Bug 修复 |
| `v1.2.0` | 新功能 |
| `v2.0.0` | 重大变更（不兼容） |
| `v2.0.0-alpha.1` | Alpha 预发布 |
| `v2.0.0-rc.1` | 候选发布版本 |

---

## 2. Tag 命名规范

### 2.1 版本 Tag

| Tag 格式 | 用途 | 示例 |
|----------|------|------|
| `vMAJOR.MINOR.PATCH` | 正式版本 | `v1.6.0` |
| `vMAJOR.MINOR.PATCH-alpha.N` | Alpha 测试版 | `v2.0.0-alpha.1` |
| `vMAJOR.MINOR.PATCH-beta.N` | Beta 测试版 | `v2.0.0-beta.2` |
| `vMAJOR.MINOR.PATCH-rc.N` | 候选发布版 | `v2.0.0-rc.1` |

### 2.2 部署 Tag

| Tag 格式 | 用途 | 示例 |
|----------|------|------|
| `deploy-N-YYYYMMDD` | 部署标记 | `deploy-11-20260118` |
| `sha-XXXXXXXX` | Git 短 SHA | `sha-5e745d28` |

### 2.3 里程碑 Tag（已废弃）

| Tag 格式 | 用途 | 状态 |
|----------|------|------|
| `round-N-complete` | 轮次完成标记 | **已废弃** |
| `shop-N-complete` | 商城功能完成标记 | **已废弃** |

> **注意**：里程碑 Tag 已不再使用，请使用正式版本 Tag。

---

## 3. 发布流程

### 3.1 发布前检查

```bash
# 1. 确保 main 分支最新
git checkout main
git pull origin main

# 2. 本地 CI 全绿
bash scripts/ci_backend.sh
bash scripts/ci_frontend.sh
bash scripts/ci_smoke.sh

# 3. 检查 GitHub Actions
gh run list --workflow=ci.yml --limit=5
```

### 3.2 创建版本 Tag

```bash
# 1. 确定版本号
VERSION="v1.7.0"

# 2. 创建注释 Tag
git tag -a $VERSION -m "Release $VERSION

Changes:
- Feature A
- Bug fix B
- Documentation updates"

# 3. 推送 Tag
git push origin $VERSION
```

### 3.3 发布到生产

```bash
# 1. GitHub Actions 自动触发 deploy.yml
# 2. 构建镜像并推送到 GHCR
# 3. 部署到绿色槽

# 4. 金丝雀发布
./deploy/apisix/traffic.sh canary 10

# 5. 监控无异常后全量切换
./deploy/apisix/traffic.sh green

# 6. 确认稳定后打部署 Tag
git tag -a deploy-12-$(date +%Y%m%d) -m "Production deployment"
git push origin deploy-12-$(date +%Y%m%d)
```

### 3.4 回滚流程

```bash
# 1. 立即回滚流量
./deploy/apisix/rollback_blue.sh

# 2. 定位问题
docker logs uhdadmin-api-green

# 3. 修复后重新发布
# 递增 PATCH 版本号
```

---

## 4. 版本历史

| 版本 | 日期 | 主要变更 |
|------|------|----------|
| `v1.1.35` | 2026-02 | 文档版本引用更新至 1.1.35 |
| `v1.1.34` | 2026-02 | Portal "我的资产" 菜单修复 (L2 menu sync + 图标映射 + 启动自动同步) |
| `v1.1.32` | 2026-02 | 错误提示美化 (useErrorHandler + Portal 18页面 + Miniapp 全量适配) + Portal 菜单清理 |
| `v1.1.31` | 2026-02 | CDKEY 系统统一化 (Admin 3合1 + Portal 资产聚合 + UI 增强) + Session Auth 增强 |
| `v1.1.30` | 2026-02 | 商城 UI 3 修复: 定价模式翻译 + 日历选择器 + 支付方式过滤 |
| `v1.1.29` | 2026-02 | 6 Bug 修复: 席位卡服务器解析 + Portal 页面修复 (#23/#28/#33/#20) + Miniapp 售后路径 (#22/#85) |
| `v1.1.28` | 2026-02 | 3 个 500 错误修复 + 席位卡下拉显示优化 |
| `v1.1.27` | 2026-02 | A+B Session Auth (Phase 1-5) + 4 Bug 修复 |
| `v1.1.26` | 2026-02 | DevOps 增强 + 性能优化 + 25 Bug 修复 |
| `v1.1.25` | 2026-02 | 积分/余额命名统一 + Portal 菜单修复 |
| `v1.1.15` | 2026-02 | 注册全流程修复 + 密码策略 + CDKEY追踪 + 等级规则编辑器 |
| `v1.6.0` | 2026-01 | 当前稳定版本 |
| `v1.5.0` | 2025-12 | 商城模块完善 |
| `v1.4.0` | 2025-12 | 支付系统集成 |
| `v1.3.1` | 2025-12 | Bug 修复 |
| `v1.3.0` | 2025-12 | Telegram Bot 集成 |
| `v1.2.0` | 2025-11 | 签到系统 |
| `v1.1.0` | 2025-11 | 邀请码返佣 |
| `v1.0.0` | 2025-10 | 首个稳定版本 |

---

## 5. GHCR 镜像 Tag

### 5.1 自动 Tag

| 触发条件 | 镜像 Tag |
|----------|----------|
| push to main | `latest`, `sha-XXXXXXXX` |
| push version tag | `vX.Y.Z`, `latest` |
| deploy workflow | `deploy-N-YYYYMMDD` |

### 5.2 拉取示例

```bash
# 最新版本
docker pull ghcr.io/fxxkrlab/uhdadmin-api:latest

# 特定版本
docker pull ghcr.io/fxxkrlab/uhdadmin-api:v1.6.0

# 特定 commit
docker pull ghcr.io/fxxkrlab/uhdadmin-api:sha-5e745d28

# 部署版本
docker pull ghcr.io/fxxkrlab/uhdadmin-api:deploy-11-20260118
```

---

## 6. 版本 API 端点

### GET /api/v1/public/version

返回当前运行版本信息：

```json
{
  "version": "1.6.0",
  "api_version": "v1",
  "git_sha": "5e745d28",
  "deploy_tag": "deploy-11-20260118",
  "environment": "production",
  "deployed_at": "2026-01-18T12:00:00Z"
}
```

---

## 7. 最佳实践

1. **始终使用注释 Tag**：`git tag -a` 而非 `git tag`
2. **Tag 前确保 CI 全绿**：不要在失败的 commit 上打 Tag
3. **版本号只增不减**：发布后发现问题，递增 PATCH 版本
4. **保留历史 Tag**：不删除已发布的版本 Tag
5. **文档同步更新**：每次发布更新 CHANGELOG
