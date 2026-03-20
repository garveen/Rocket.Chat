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

### 3.4 SR-004: 消息推送触发与通知偏好过滤

| 评估项 | 结论 |
|--------|------|
| afterSaveMessage Hook | ✅ 使用 `IPostMessageSent.executePostMessageSent()` |
| 检查 `getuiPushEnabled` | ✅ 从 `message.room.customFields?.getuiPushEnabled` 读取 |
| 获取房间成员 | ✅ `read.getRoomReader().getMembers(roomId)` |
| 排除消息发送者 | ✅ 纯业务逻辑 |
| 查询成员 Token | ✅ 从 `IPersistence` 读取 |
| 构建推送内容 | ✅ 纯业务逻辑 |
| 调用个推 API | ✅ 使用 `IHttp` |
| 异步执行（不阻塞消息保存） | ✅ 在消息保存后触发，不阻塞原流程 |
| **判断 @提及（hasMentionToAll/Here/User）** | ✅ 通过 `message._unmappedProperties_.mentions` 实时获取 |
| **服务器级默认推送设置** | ✅ 通过 `IServerSettingRead` 读取（公开设置，非 secret） |
| **频道级推送偏好（mobilePushNotifications）** | ✅ 注册 Token 时缓存到 IPersistence（详见下文） |
| **频道通知完全关闭（disableNotifications）** | ✅ 注册 Token 时缓存 |
| **屏蔽群组提及（muteGroupMentions）** | ✅ 注册 Token 时缓存 |
| **关键词高亮（userHighlights）** | ✅ 注册 Token 时缓存 |
| **线程回复关注者（hasReplyToThread）** | ⚠️ 无法在插件层获取（见限制 4.1） |

#### 深度调查结论

通过逐一审查 RC Apps-Engine 的全部 Bridge 实现（`RoomBridge`、`UserBridge`、`MessageBridge`、`InternalBridge`、`ExperimentalBridge`）以及消息转换器（`messages.js`）、Deno 运行时序列化链路（`codec.ts` msgpack + `hydrateMessageObjects`），确认以下两类数据可供插件使用：

**【A】消息层数据 — 在 `IPostMessageSent` 回调中实时可用**

消息转换器使用 `transformMappedData()` 将 RC 消息映射到 `IMessage`，未映射字段**完整保留在 `_unmappedProperties_` 属性中**。`mentions` 字段未在映射表内，因此在 `_unmappedProperties_` 中存在。Deno 运行时通过 msgpack 序列化全量传输对象，`hydrateMessageObjects()` 只处理 `room` 字段，不删除任何其他属性。

因此，可在 `executePostMessageSent()` 中访问：

```typescript
const mentions = (message as any)._unmappedProperties_?.mentions as Array<{
    _id: string;   // 'all' | 'here' | userId
    username?: string;
    type?: 'user' | 'team';
}> | undefined;

const hasMentionToAll  = mentions?.some(m => m._id === 'all') ?? false;
const hasMentionToHere = mentions?.some(m => m._id === 'here') ?? false;
const mentionedUserIds = mentions
    ?.filter(m => !['all','here'].includes(m._id) && (!m.type || m.type === 'user'))
    .map(m => m._id) ?? [];
```

此外，`message.type`（`_unmappedProperties_.t`）、`message.threadId`（`tmid`）均已映射，可直接用于判断系统消息和线程消息。

**【B】频道订阅偏好 — 注册时获取，缓存至 `IPersistence`**

Apps-Engine 所有 Bridge 均**不暴露** `ISubscription` 的通知偏好字段（`mobilePushNotifications`、`disableNotifications`、`muteGroupMentions`、`userHighlights`）。`subscriptions.getOne` RC REST API 端点也只返回**调用者自己**的订阅数据，无管理员旁路可查询他人数据（已验证 `apps/meteor/app/api/server/v1/subscriptions.ts`）。

但 `IApiRequest.headers` 包含原始 HTTP 请求头（`api.ts::_appApiExecutor` 直接透传 `req.headers`），含 `x-auth-token` 和 `x-user-id`。插件在 Token 注册端点中可：

1. 从 `request.headers['x-auth-token']` 提取用户凭证
2. 使用 `IHttp` 调用 `GET {siteUrl}/api/v1/subscriptions.get`
3. 提取每个频道的 `{ rid, mobilePushNotifications, disableNotifications, muteGroupMentions, userHighlights }` 并缓存至 `IPersistence`
4. 在 `executePostMessageSent()` 时从缓存读取，执行与原生 `shouldNotifyMobile()` 等效的过滤逻辑

**结论：✅ 基本完全可行**（mentions 和服务器默认设置实时可用；订阅偏好通过注册时缓存提供，仅 `hasReplyToThread` 无法实现，见第 4 节）

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

### 4.1 线程回复关注者（hasReplyToThread）

**问题**：RC 原生系统会将"关注了某个线程的用户"纳入推送接收范围（即使这些用户没有被 @提及）。插件无法获取线程关注者列表，因为 Apps-Engine 未暴露任何接口查询 `thread.followers` 或线程订阅关系。

**影响**：仅影响线程回复消息（`message.threadId` 不为空）。用户关注线程后，若本插件未给其发送个推，用户需通过 RC 原生通知或手动查看。

**缓解措施**：对线程消息，可以放宽过滤策略（例如对 `mobilePushNotifications === 'all'` 或 `undefined` 的用户仍推送），以减少漏推概率。如需精确实现，须在 RC 核心增加接口。

---

### 4.2 订阅偏好缓存时效性

**问题**：`mobilePushNotifications`、`disableNotifications`、`muteGroupMentions`、`userHighlights` 等字段均为缓存数据，在用户注册/刷新 Token 时更新，不能实时反映用户更改。

**缓解措施**：
- 手机 App 每次启动/前台激活时重新调用 Token 注册端点，触发偏好数据刷新
- 提供 `POST /api/apps/public/{appId}/sync-prefs` 端点，供用户主动刷新
- 缓存 TTL 默认 24 小时，过期后下次消息推送时触发异步刷新（本次 fail-safe 推送，下次生效）

---

### 4.3 API 端点路径变更

**问题**：Token API 路径从嵌入式方案的 `/api/v1/getui.token` 变为 `/api/apps/public/{appId}/getui-token`。

**缓解措施**：手机 App 通过插件 `info` 端点动态发现端点路径，或将 appId 作为 App 配置项。

---

### 4.4 房间开关无原生 Admin UI

**问题**：无法在 RC Admin 后台的房间设置页面添加原生"启用个推推送"开关。

**缓解措施**：使用 Slash 命令 `/getui-push enable` / `/getui-push disable` 切换，管理员可快速配置。

---

### 4.5 设置位置变更

**问题**：插件设置在 Admin → Apps → GeTui Push 下，而非 Admin → Push 设置分组下。

**缓解措施**：此差异不影响功能，只是 UX 位置不同，可接受。

---

### 4.6 用户级别全局推送偏好覆盖（user.settings.preferences.pushNotifications）

**问题**：RC 原生系统会读取 `user.settings.preferences.pushNotifications`（用户在个人偏好中设置的全局推送级别，可覆盖服务器默认值）。Apps-Engine 的用户转换器（`users.js::convertToApp`）只提取 `language` 字段，不提供此字段。

**影响范围**：只影响那些在 RC 个人偏好中**自定义了全局推送级别**（与服务器默认值不同）的用户。如果用户未自定义（使用服务器默认），则无影响。

**缓解措施**：在 `GET /api/v1/subscriptions.get` 返回的订阅数据中，若用户设置了个人全局覆盖，该覆盖值已被写入 `subscription.mobilePushNotifications`（通过 `updateNotificationPreferences` 写入）；因此，**实际上缓存的 `subscription.mobilePushNotifications` 已经包含了用户级别的覆盖**，此限制在多数情况下不成立。

## 5. 方案对比

| 维度 | 嵌入式方案 | 插件方案 |
|------|-----------|----------|
| 修改核心代码量 | 大（8+ 个文件） | **零**（完全独立） |
| 可安装/卸载 | ❌ 需重新部署 | ✅ 在线安装/卸载 |
| 升级隔离性 | ❌ RC 升级可能冲突 | ✅ 插件独立版本管理 |
| @提及检测（hasMentionToAll/Here/User） | ✅ 实时（原生） | ✅ 实时（`_unmappedProperties_.mentions`） |
| 频道级推送设置（mobilePushNotifications） | ✅ 实时查询 MongoDB | ✅ 缓存（注册时获取，有轻微延迟） |
| 频道完全静音（disableNotifications） | ✅ 实时 | ✅ 缓存 |
| 屏蔽群组提及（muteGroupMentions） | ✅ 实时 | ✅ 缓存 |
| 关键词高亮（userHighlights） | ✅ 实时 | ✅ 缓存 |
| 线程关注者（hasReplyToThread） | ✅ 实时 | ⚠️ 不支持（Apps-Engine 无接口） |
| 服务器级默认推送设置 | ✅ 实时 | ✅ 实时（`IServerSettingRead`） |
| 用户级全局推送覆盖（settings.preferences.pushNotifications） | ✅ 实时 | ✅ 已合并到订阅数据（RC 写入机制保证） |
| API 端点路径 | `/api/v1/getui.token` | `/api/apps/public/{id}/getui-token` |
| 房间开关入口 | Admin UI 原生 | Slash 命令 |
| 配置位置 | Admin → Push 分组 | Admin → Apps → GeTui Push |
| 部署方式 | 修改服务端并重启 | 上传 `.zip` 插件包 |
| 维护成本 | 高（与 RC 版本耦合） | 低（独立维护） |

## 6. 结论

**个推推送集成完全可以通过 Rocket.Chat Apps-Engine 插件形式实现**，核心通知过滤逻辑与原生 RC 方案的相似度达到 ~95%：

| 功能 | 状态 |
|------|------|
| 个推 API 调用 | ✅ 完整 |
| Token 管理 | ✅ 完整 |
| 频道推送开关 | ✅ 完整 |
| 消息触发推送 | ✅ 完整 |
| 插件全局设置 | ✅ 完整 |
| @提及检测（实时） | ✅ 完整（`_unmappedProperties_.mentions`） |
| 服务器默认推送设置（实时） | ✅ 完整（`IServerSettingRead`） |
| 频道级通知偏好（缓存） | ✅ 基本完整（注册时缓存，24h TTL） |
| 线程关注者推送 | ⚠️ 缺失（仅此一项，缓解策略见第 4.1 节） |

**推荐采用插件方案**，理由：
- 零核心代码修改，大幅降低维护风险和版本耦合
- 可打包发布、在线安装/卸载，部署更灵活
- 通知过滤逻辑与原生方案几乎等同（唯一限制为线程关注者，对大多数场景无影响）
- 频道级通知偏好通过缓存实现，手机 App 每次启动时刷新，时效性可接受

新的需求文档和设计文档见：
- [`plugin-requirements.md`](./plugin-requirements.md) — 插件版需求文档
- [`plugin-design.md`](./plugin-design.md) — 插件版技术设计文档
