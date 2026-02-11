# 05_CONTRACTS — 接口/契约索引
> 原始契约文档保存在 `docs/90_ARCHIVE/2025-12/CONTRACTS.md`。本页为索引，详细接口与约束请查阅归档。

## 通用约定
| 约定 | 内容 |
|------|------|
| Base URL | `/api/v1` |
| 认证 | `Authorization: Bearer <JWT>` |
| 管理员判定 | `level >= 91`，Sysop=100 禁止修改/删除 |
| 分页 | `?skip=<int>&limit=<int>` |
| 追踪 | `X-Request-ID` 透传，返回 `trace_id` |

## 响应 Envelope
```json
{"code": 0, "message": "ok", "data": {...}, "trace_id": "uuid"}
```

## 错误码
| Code | 含义 |
|------|------|
| 0 | 成功 |
| 400 | 参数/业务校验失败 |
| 401 | 未登录或 token 失效 |
| 403 | 权限不足 |
| 404 | 资源不存在 |
| 409 | 冲突（重复） |
| 500 | 服务器错误 |
| 501 | 功能未启用（可选组件） |

## 核心接口
| 模块 | 端点 | 说明 |
|------|------|------|
| Auth | `POST /auth/login` | 登录，返回 access_token |
| Auth | `POST /auth/register` | 注册（受开关控制） |
| Auth | `GET /auth/register-settings` | 公开的注册配置 |
| User | `GET /user/me` | 当前用户信息 |
| User | `POST /user/me/reset-password` | 修改密码 |
| User | `POST /user/me/invite-code` | 创建邀请码 |
| Admin | `GET /admin/users` | 用户列表 |
| Admin | `POST /admin/users` | 创建用户 |
| Admin | `GET /admin/roles` | 角色列表 |
| Admin | `POST /admin/roles/assign-role` | 分配角色 |
| Admin | `GET /admin/system-settings` | 系统设置 |
| Menu | `GET /menu/all` | 动态菜单（基于权限） |

## 参考
- 归档契约：`90_ARCHIVE/2025-12/CONTRACTS.md`
- 架构概要：`04_ARCH.md`
