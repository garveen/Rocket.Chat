# 个推推送集成 — 插件可行性分析

## 1. 背景

Rocket.Chat 提供了一套 **Apps-Engine**（插件框架），允许开发者在不修改核心服务端代码的情况下扩展 Rocket.Chat 功能。本文分析个推服务端集成功能是否可以通过 Apps-Engine 插件形式实现，并与嵌入式方案进行对比。

## 2. Rocket.Chat Apps-Engine 能力概述

Apps-Engine 为插件提供以下核心能力：

| 能力 | 接口 | 说明 |
|------|------|------|
| 消息事件监听 | `IPostMessageSent` | 消息发送后触发回调 |
| 自定义 REST API 端点 | `IApiExtend` + `IApiEndpoint` | 注册公开或私有 API 端点，支持用户鉴权 |
| 插件内置存储 | `IPersistence` | 键值对+关联记录存储（替代 MongoDB 集合） |
| HTTP 外部请求 | `IHttp` | 调用外部服务（如个推 REST API） |
| 插件设置管理 | `ISettingsExtend` | 在管理后台 Apps 页面添加可配置项 |
| 读取房间成员 | `IRoomRead.getMembers()` | 获取房间所有成员列表 |
| 读取/写入房间自定义字段 | `IRoomBuilder.setCustomFields()` | 通过 `customFields` 存储房间级别配置 |
| 定时任务 | `ISchedulerExtend` / `ISchedulerModify` | 注册和调度周期性任务 |
| Slash 命令 | `ISlashCommandsExtend` | 注册斜线命令（用于管理员操作） |
| 用户信息读取 | `IUserRead` | 读取用户基本信息 |
| 鉴权 API 请求 | `IApiEndpoint.authRequired` | 端点支持 RC 标准 X-Auth-Token 鉴权 |

## 3. 各功能项可行性逐项分析

### 3.1 SR-001: 集成个推服务端 SDK（HTTP 调用）

| 评估项 | 结论 |
|--------|------|
| 获取个推鉴权 Token | ✅ 使用 `IHttp.post()` 调用 GeTui auth API |
| 缓存鉴权 Token | ✅ 使用 `IPersistence` 存储 token 和过期时间 |
| 批量推送（POST /push/list/cid） | ✅ 使用 `IHttp.post()` 调用 GeTui 推送 API |
| 分批处理（>200 CID） | ✅ 纯业务逻辑，无限制 |
| 重试机制 | ✅ 纯业务逻辑，无限制 |
| 无效 CID 清理 | ✅ 使用 `IPersistence` 删除记录 |

**结论：✅ 完全可行**

---

### 3.2 SR-002: 频道推送设置（房间级别开关）

| 评估项 | 结论 |
|--------|------|
| 向 IRoom 添加 `getuiPushEnabled` 字段 | ❌ Apps 不能修改 RC 核心类型定义 |
| 修改 `saveRoomSettings` 方法 | ❌ Apps 不能修改 RC 核心方法 |
| 在房间 admin UI 添加设置开关 | ❌ Apps 不能修改 RC admin 面板的房间设置项 |
| **替代方案：使用 `room.customFields`** | ✅ Apps-Engine `IRoom` 已有 `customFields: { [key: string]: any }` 字段；可通过 `IRoomBuilder.setCustomFields()` 写入 |
| **替代方案：Slash 命令管理开关** | ✅ 管理员可用 `/getui-push enable` / `/getui-push disable` 命令切换 |

**结论：⚠️ 可行（需替代方案）**：使用 `room.customFields.getuiPushEnabled` 替代新增字段；通过 Slash 命令替代 admin UI 开关。不需要修改任何 RC 核心代码。

---

### 3.3 SR-003: 个推 Token API（注册/删除）

| 评估项 | 结论 |
|--------|------|
| POST /api/v1/getui.token | ❌ Apps 不能在 `/api/v1/` 路径下注册端点 |
| **替代方案：App 公开端点** | ✅ 可注册 `POST /api/apps/public/{appId}/getui-token`，设置 `authRequired: true`，`request.user` 即为已登录用户 |
| DELETE /api/v1/getui.token | ❌ 同上 |
| **替代方案：** `DELETE /api/apps/public/{appId}/getui-token` | ✅ 同上 |
| Token 数据存储 | ✅ 使用 `IPersistence` 替代 `getui_push_tokens` MongoDB 集合，通过 `RocketChatAssociationModel.USER` 按用户关联 |
| Token 上限管理 | ✅ 查询用户的所有 Token，超限则删除最旧的 |
| Token 唯一性检查 | ✅ 使用 `MISC` 关联模型按 token 值检索 |
| 用户登出清理 Token | ✅ 监听 `IPostUserLoggedOut` 事件 |

**结论：✅ 完全可行**（API 路径格式略有不同）

---

### 3.4 SR-004: 消息推送触发

| 评估项 | 结论 |
|--------|------|
| afterSaveMessage Hook | ✅ 使用 `IPostMessageSent.executePostMessageSent()` |
| 检查 `getuiPushEnabled` | ✅ 从 `message.room.customFields?.getuiPushEnabled` 读取 |
| 获取房间成员 | ✅ `read.getRoomReader().getMembers(roomId)` |
| 排除消息发送者 | ✅ 纯业务逻辑 |
| 查询成员 Token | ✅ 从 `IPersistence` 读取 |
| 构建推送内容 | ✅ 纯业务逻辑 |
| 调用个推 API | ✅ 使用 `IHttp` |
| 异步执行（不阻塞消息保存） | ✅ `executePostMessageSent` 是在消息保存后调用，不阻塞原流程 |
| **检查用户移动通知偏好** | ⚠️ Apps-Engine `IUser.settings.preferences` 仅暴露 `language` 字段，不包含移动通知偏好 |

**结论：✅ 可行（用户通知偏好检查为已知限制，见第 4 节）**

---

### 3.5 SR-005: 个推全局配置

| 评估项 | 结论 |
|--------|------|
| 添加到 Admin → Push 分组 | ❌ Apps 的设置在 Admin → Apps → {App 名称} 下 |
| `Getui_Enabled` 开关 | ✅ 通过 `ISettingsExtend.provideSetting()` 添加，`SettingType.BOOLEAN` |
| `Getui_AppId` | ✅ `SettingType.STRING` |
| `Getui_AppKey` | ✅ `SettingType.STRING` |
| `Getui_MasterSecret` | ✅ `SettingType.PASSWORD`（显示为密码输入框） |
| `Getui_Api_Url` | ✅ `SettingType.STRING` |
| `Getui_Max_Tokens_Per_User` | ✅ `SettingType.NUMBER` |

**结论：✅ 完全可行**（设置位置在 Apps 页面而非 Push 页面，UX 有差异但功能完整）

---

### 3.6 SR-006: 推送内容构建

**结论：✅ 完全可行**（纯业务逻辑，无限制）

---

### 3.7 SR-007: 推送日志

| 评估项 | 结论 |
|--------|------|
| 日志记录 | ✅ 使用 `ILogger`（`this.getLogger()`） |
| 日志级别 | ✅ 支持 `debug/info/warn/error` |

**结论：✅ 完全可行**

---

### 3.8 SR-008: Token 上限管理

**结论：✅ 完全可行**（纯业务逻辑，在 IPersistence 基础上实现）

---

### 3.9 用户登出自动清理 Token

| 评估项 | 结论 |
|--------|------|
| 用户登出时清理 Token | ✅ 实现 `IPostUserLoggedOut` 接口，在回调中清理该用户的所有 Token |

**结论：✅ 完全可行**

## 4. 已知限制与缓解措施

### 4.1 用户移动通知偏好检查

**问题**：Apps-Engine 未暴露用户的移动通知偏好设置（如"关闭移动端推送"），因此无法实现与嵌入式方案等同的 `shouldNotifyMobile()` 检查。

**缓解措施**：
- 个推 App 推送给房间内所有拥有有效 Token 的成员（不过滤通知偏好）
- 若用户不希望接收个推推送，应在个推 App 内登出（删除 Token），而不是依赖 RC 内部的通知设置
- 在 App 文档中明确说明此限制

### 4.2 API 端点路径变更

**问题**：Token API 路径从 `/api/v1/getui.token` 变为 `/api/apps/public/{appId}/getui-token`。

**缓解措施**：
- 手机 App 在首次连接时调用服务器发现接口（如 `/api/v1/apps/getui-push/info`，或通过约定固定 appId）获取实际端点路径
- 或在手机 App 中将 RC App 的 appId 作为配置项

### 4.3 房间开关无原生 Admin UI

**问题**：无法在 RC Admin 后台的房间设置页面添加原生"启用个推推送"开关。

**缓解措施**：
- 使用 Slash 命令（`/getui-push enable` / `/getui-push disable`）切换房间开关
- 管理员可通过命令快速配置，无需访问 admin 面板

### 4.4 设置位置变更

**问题**：插件设置在 Admin → Apps → GeTui Push 下，而非 Admin → Push 设置分组下。

**缓解措施**：此差异不影响功能，只是 UX 位置不同，管理员可接受。

## 5. 方案对比

| 维度 | 嵌入式方案 | 插件方案 |
|------|-----------|----------|
| 修改核心代码量 | 大（8+ 个文件） | **零**（完全独立） |
| 可安装/卸载 | ❌ 需重新部署 | ✅ 在线安装/卸载 |
| 升级隔离性 | ❌ RC 升级可能冲突 | ✅ 插件独立版本管理 |
| 功能完整性 | 完整（含用户通知偏好） | 接近完整（略有限制） |
| API 端点路径 | `/api/v1/getui.token` | `/api/apps/public/{id}/getui-token` |
| 房间设置入口 | Admin UI 原生开关 | Slash 命令 |
| 配置位置 | Admin → Push 分组 | Admin → Apps → GeTui Push |
| 部署方式 | 修改服务端并重启 | 上传 `.zip` 插件包 |
| 维护成本 | 高（与 RC 版本耦合） | 低（独立维护） |
| 数据存储 | 独立 MongoDB 集合 | Apps-Engine 内置存储 |

## 6. 结论

**个推推送集成完全可以通过 Rocket.Chat Apps-Engine 插件形式实现**，核心功能（个推 API 调用、Token 管理、消息触发推送、插件设置）均可覆盖，仅有以下可接受的限制：

1. 用户通知偏好检查不可用（缓解：用户通过登出管理）
2. API 路径格式略有不同（缓解：手机 App 可动态发现）
3. 房间开关通过 Slash 命令而非 Admin UI 配置（可接受）

**推荐采用插件方案**，理由：
- 零核心代码修改，降低维护风险和版本耦合
- 可打包发布、在线安装，部署更灵活
- 独立的生命周期管理（安装、更新、卸载）

新的需求文档和设计文档见：
- [`plugin-requirements.md`](./plugin-requirements.md) — 插件版需求文档
- [`plugin-design.md`](./plugin-design.md) — 插件版技术设计文档
