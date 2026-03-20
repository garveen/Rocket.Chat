# 个推（GeTui）推送集成 — 服务端需求文档

## 1. 项目背景

### 1.1 项目概述

Rocket.Chat 是一个开源的团队通信平台，当前支持通过 FCM（Firebase Cloud Messaging）和 APNs（Apple Push Notification Service）进行移动端推送通知。由于 FCM 在中国大陆网络环境下无法正常访问，需要引入国内推送服务——**个推（GeTui）**，以保障国内用户能够正常接收推送通知。

本项目将开发一个基于 uni-app 框架的新手机应用（以下简称"个推App"），支持 Android 和鸿蒙 Next（HarmonyOS Next）平台，并在 Rocket.Chat 服务端集成个推 SDK 以实现消息推送。

### 1.2 现有架构

当前 Rocket.Chat 推送通知架构：

```
消息发送 → afterSaveMessage Hook → sendMessageNotifications()
→ 查询订阅 & 用户偏好设置 → shouldNotifyMobile()
→ PushNotification.send() → Push.send()
→ FCM/APNs（直接发送）或 Rocket.Chat Cloud Gateway（网关转发）
```

推送 Token 管理：
- REST API: `POST /api/v1/push.token`（注册）、`DELETE /api/v1/push.token`（删除）
- 存储集合: `_raix_push_app_tokens`
- Token 类型: `apn`（iOS）、`gcm`（Android/FCM）

### 1.3 名词定义

| 术语 | 说明 |
|------|------|
| 个推（GeTui） | 国内主流推送服务提供商，提供 Android 和鸿蒙推送通道 |
| CID | 个推客户端标识（Client ID），个推 SDK 为每个设备生成的唯一标识 |
| 批量推送 | 个推的推送方式之一，通过指定 CID 列表进行批量消息推送 |
| 推送频道 | Rocket.Chat 中启用了个推推送功能的频道/房间 |
| 个推App | 本项目新开发的 uni-app 手机应用 |
| 官方App | Rocket.Chat 官方 React Native 应用 |

## 2. 需求总览

### 2.1 功能需求列表

| 编号 | 功能 | 优先级 | 说明 |
|------|------|--------|------|
| SR-001 | 集成个推服务端 SDK | P0 | 使用个推 REST API 进行批量推送 |
| SR-002 | 频道推送设置 | P0 | 为频道添加"启用个推推送"开关 |
| SR-003 | 个推 Token API | P0 | 接收和管理手机App的推送 Token (CID) |
| SR-004 | 消息推送触发 | P0 | 对启用推送的频道触发个推批量推送 |
| SR-005 | 个推全局配置 | P0 | 管理员配置个推 AppID/AppKey/MasterSecret |
| SR-006 | 推送内容构建 | P1 | 构建适配个推格式的推送内容 |
| SR-007 | 推送日志 | P1 | 记录推送日志以便故障排查 |
| SR-008 | Token 上限管理 | P1 | 限制每个用户的 Token 数量（默认上限1） |

### 2.2 非功能需求

| 编号 | 需求 | 说明 |
|------|------|------|
| NR-001 | 性能 | 单次批量推送应在 5 秒内完成（个推 API 限制单次最多 200 个 CID） |
| NR-002 | 可靠性 | 推送失败时应进行重试，并记录日志 |
| NR-003 | 安全性 | 个推 MasterSecret 等密钥应加密存储，API 需认证 |
| NR-004 | 兼容性 | 不影响现有 FCM/APNs 推送功能 |
| NR-005 | 可配置性 | 个推功能可通过管理后台开关控制 |

## 3. 详细需求

### 3.1 SR-001: 集成个推服务端 SDK

#### 3.1.1 描述

在 Rocket.Chat 服务端集成个推推送能力，使用个推 REST API v2 进行消息推送。采用**批量推送**方式（`/push/list/message`），因其额度限制最为宽松。

#### 3.1.2 个推 REST API 调用流程

```
1. 获取鉴权 Token
   POST https://restapi.getui.com/v2/{appId}/auth
   Body: { sign, timestamp, appkey }
   → 返回 token（有效期最长 24 小时）

2. 创建消息体
   POST https://restapi.getui.com/v2/{appId}/push/list/message
   Body: { ...notification }
   → 返回 taskId

3. 批量推送
   POST https://restapi.getui.com/v2/{appId}/push/list/cid
   Body: { audience: { cid: [...] }, taskid, is_async: true }
   → 返回推送结果
```

#### 3.1.3 功能要求

- **FR-001-1**: 实现个推鉴权 Token 的获取和缓存，Token 过期前自动刷新
- **FR-001-2**: 实现批量推送接口调用，支持单次最多 200 个 CID
- **FR-001-3**: 当 CID 数量超过 200 时，自动分批发送
- **FR-001-4**: 实现推送失败重试机制（最多 3 次，指数退避）
- **FR-001-5**: 处理个推返回的无效 CID，自动清理数据库中的无效 Token

#### 3.1.4 验收标准

- 能成功获取个推鉴权 Token 并缓存
- 能向指定 CID 列表批量推送消息
- CID 超过 200 时能自动分批
- 无效 CID 能被自动清理
- 个推 API 返回错误时有重试机制

---

### 3.2 SR-002: 频道推送设置

#### 3.2.1 描述

为 Rocket.Chat 的频道/房间（Room）添加一个新的设置项，控制该频道是否启用个推推送。只有启用了此设置的频道中的消息才会通过个推进行推送。

#### 3.2.2 功能要求

- **FR-002-1**: 在 IRoom 接口中新增 `getuiPushEnabled` 布尔字段
- **FR-002-2**: 在房间设置保存方法中支持 `getuiPushEnabled` 字段的读写
- **FR-002-3**: 在管理后台的房间设置 UI 中展示该开关（适配现有 `saveRoomSettings` 流程）
- **FR-002-4**: 仅具有 `edit-room` 权限的用户（管理员/房间所有者/版主）可修改此设置
- **FR-002-5**: 默认值为 `false`（不启用推送）

#### 3.2.3 影响范围

- 类型定义: `packages/core-typings/src/IRoom.ts`
- 房间设置: `apps/meteor/app/channel-settings/server/methods/saveRoomSettings.ts`
- REST API: `apps/meteor/app/api/server/v1/channels.ts` 或 `rooms.ts`
- 房间模型: `packages/models/src/models/Rooms.ts`

#### 3.2.4 验收标准

- 房间设置中可见"启用个推推送"开关
- 仅有 `edit-room` 权限的用户可修改
- 设置变更后正确保存至数据库
- 新建频道默认不启用个推推送

---

### 3.3 SR-003: 个推 Token API

#### 3.3.1 描述

提供 REST API 端点，允许手机 App 注册和管理个推推送 Token（CID）。

#### 3.3.2 API 定义

##### 注册/更新 Token

```
POST /api/v1/getui.token

Headers:
  X-Auth-Token: <登录Token>
  X-User-Id: <用户ID>

Body:
{
  "token": "<个推CID>",
  "appName": "<应用标识>",
  "platform": "android" | "harmony"
}

Response (200):
{
  "success": true,
  "result": {
    "_id": "<文档ID>",
    "token": "<个推CID>",
    "appName": "<应用标识>",
    "userId": "<用户ID>",
    "platform": "android" | "harmony",
    "enabled": true,
    "createdAt": "<ISO日期>",
    "_updatedAt": "<ISO日期>"
  }
}
```

##### 删除 Token

```
DELETE /api/v1/getui.token

Headers:
  X-Auth-Token: <登录Token>
  X-User-Id: <用户ID>

Body:
{
  "token": "<个推CID>"
}

Response (200):
{
  "success": true
}
```

#### 3.3.3 功能要求

- **FR-003-1**: 注册 Token 时需验证用户身份（需要登录态）
- **FR-003-2**: 同一用户的 Token 数量默认上限为 1，超出时替换最早的 Token
- **FR-003-3**: Token 上限可通过全局设置 `Getui_Max_Tokens_Per_User` 进行调整
- **FR-003-4**: 注册时如果相同 Token 已存在（属于同一用户），则更新而非新建
- **FR-003-5**: 注册时如果相同 Token 属于其他用户，应先从其他用户移除后再注册给当前用户
- **FR-003-6**: 删除 Token 时仅允许删除属于当前用户的 Token
- **FR-003-7**: 用户登出时自动清理该用户的个推 Token

#### 3.3.4 数据存储

使用独立集合 `getui_push_tokens` 存储个推 Token：

```typescript
interface IGetuiPushToken {
  _id: string;
  token: string;              // 个推 CID
  appName: string;            // 应用标识
  userId: string;             // 用户 ID
  platform: 'android' | 'harmony';  // 平台类型
  enabled: boolean;           // 是否启用
  createdAt: Date;
  _updatedAt: Date;
}
```

索引：
- `{ userId: 1 }` — 按用户查询
- `{ token: 1 }` — 按 Token 查询，唯一索引

#### 3.3.5 验收标准

- 已登录用户可注册个推 Token
- 同一用户超过 Token 上限时自动淘汰旧 Token
- 相同 Token 不会重复注册
- Token 可被正确删除
- 用户登出后 Token 被自动清理

---

### 3.4 SR-004: 消息推送触发

#### 3.4.1 描述

当启用了个推推送的频道收到新消息时，系统应将该消息通过个推推送给所有对该频道具有阅读权限的用户。

#### 3.4.2 推送流程

```
新消息 → afterSaveMessage Hook
  → 检查房间 getuiPushEnabled 是否为 true
  → 如为 true：
    → 查询该房间的所有成员（具有阅读权限的用户）
    → 排除消息发送者
    → 查询这些用户的个推 Token（CID）
    → 构建推送内容（标题、正文、负载）
    → 调用个推批量推送 API
```

#### 3.4.3 功能要求

- **FR-004-1**: 在消息保存后的 Hook 中检查房间是否启用个推推送
- **FR-004-2**: 查询房间所有成员（通过订阅关系），排除消息发送者
- **FR-004-3**: 批量查询成员的个推 Token
- **FR-004-4**: 仅向拥有有效 Token 的用户发送推送
- **FR-004-5**: 推送操作应异步执行，不阻塞消息保存流程
- **FR-004-6**: 需要尊重用户级别的通知偏好设置（如用户关闭了移动推送则不推送）
- **FR-004-7**: 推送所有消息，不仅限于 @提及

#### 3.4.4 推送内容格式

```json
{
  "notification": {
    "title": "频道名称 - 发送者",
    "body": "消息内容（截取前 200 字符）",
    "click_type": "intent",
    "intent": "rocket-getui-app://message?host=<serverUrl>&rid=<roomId>&msgId=<messageId>"
  },
  "transmission": {
    "host": "<服务器地址>",
    "rid": "<房间ID>",
    "msgId": "<消息ID>",
    "roomName": "<房间名称>",
    "senderName": "<发送者名称>"
  }
}
```

#### 3.4.5 验收标准

- 启用推送的频道发送消息后，所有成员收到个推推送
- 消息发送者不收到自己消息的推送
- 已关闭移动推送偏好的用户不收到推送
- 推送操作不影响消息发送速度
- 推送内容正确包含频道名、发送者、消息预览

---

### 3.5 SR-005: 个推全局配置

#### 3.5.1 描述

在 Rocket.Chat 管理后台（Administration → Settings）中添加个推相关配置项。

#### 3.5.2 配置项列表

| 设置项 ID | 类型 | 默认值 | 说明 |
|-----------|------|--------|------|
| `Getui_Enabled` | boolean | false | 启用/禁用个推推送服务 |
| `Getui_AppId` | string | '' | 个推应用 AppID |
| `Getui_AppKey` | string | '' | 个推应用 AppKey |
| `Getui_MasterSecret` | string (secret) | '' | 个推应用 MasterSecret |
| `Getui_Api_Url` | string | 'https://restapi.getui.com/v2' | 个推 REST API 基础 URL |
| `Getui_Max_Tokens_Per_User` | int | 1 | 每用户最大 Token 数量 |

#### 3.5.3 功能要求

- **FR-005-1**: 所有配置项在管理后台的 "Push" 分组下展示，使用 "GeTui / 个推" 子分组
- **FR-005-2**: `Getui_MasterSecret` 应为密码类型字段（`secret: true`）
- **FR-005-3**: 非 `Getui_Enabled` 的设置项仅在启用个推时显示（`enableQuery`）
- **FR-005-4**: 配置变更后应即时生效，无需重启服务
- **FR-005-5**: 配置项需要 `admin` 权限才能修改

#### 3.5.4 验收标准

- 管理后台能看到"GeTui / 个推"设置分组
- MasterSecret 字段以密码形式显示
- 关闭个推功能后其它设置项隐藏
- 配置变更无需重启即可生效

---

### 3.6 SR-006: 推送内容构建

#### 3.6.1 描述

将 Rocket.Chat 消息转换为个推推送所需的内容格式，包含通知标题、正文、自定义数据等。

#### 3.6.2 功能要求

- **FR-006-1**: 通知标题格式: `[频道名称] 发送者名称`
- **FR-006-2**: 通知正文: 消息文本前 200 个字符，超出部分用 `...` 截断
- **FR-006-3**: 附件消息显示为 `[文件]`、`[图片]`、`[视频]` 等占位文本
- **FR-006-4**: 自定义数据（transmission/payload）中包含 `host`、`rid`、`msgId`、`roomName`、`senderName`
- **FR-006-5**: 尊重服务器隐私设置（`Push_show_username_room`、`Push_show_message`）
- **FR-006-6**: 如果隐私设置不允许显示消息内容，标题显示"新消息"，正文显示"您有一条新消息"

#### 3.6.3 验收标准

- 推送标题、正文格式正确
- 长消息正确截断
- 附件消息有合适的占位文本
- 隐私设置正确生效

---

### 3.7 SR-007: 推送日志

#### 3.7.1 描述

记录个推推送操作的日志，包括推送请求、响应、错误等信息。

#### 3.7.2 功能要求

- **FR-007-1**: 使用 Rocket.Chat 现有日志框架（Logger）
- **FR-007-2**: 记录推送触发事件：房间ID、消息ID、目标用户数、CID数
- **FR-007-3**: 记录 API 调用结果：鉴权、创建消息体、批量推送
- **FR-007-4**: 记录错误详情：API 错误码、错误描述、重试次数
- **FR-007-5**: 日志级别: info（正常推送）、warn（重试）、error（失败）

#### 3.7.3 验收标准

- 每次推送在日志中有迹可查
- 错误信息有足够的诊断信息
- 日志不包含敏感信息（如 MasterSecret）

---

### 3.8 SR-008: Token 上限管理

#### 3.8.1 描述

限制每个用户可注册的个推 Token 数量，防止 Token 无限增长。

#### 3.8.2 功能要求

- **FR-008-1**: 默认每用户最多 1 个 Token
- **FR-008-2**: 超出上限时，移除最早创建的 Token
- **FR-008-3**: 上限可通过 `Getui_Max_Tokens_Per_User` 全局设置调整
- **FR-008-4**: 调整上限后，已注册的超额 Token 在下次注册时清理

#### 3.8.3 验收标准

- 用户注册第二个 Token 时，第一个 Token 被自动移除（默认上限1）
- 调整上限为 2 后，用户可以同时拥有 2 个有效 Token
- Token 超额清理策略正确（移除最老的）

## 4. 约束与假设

### 4.1 约束

1. **个推 SDK 版本**: 使用个推 REST API v2，不引入 Java/Native SDK
2. **批量推送限制**: 单次请求最多 200 个 CID，需分批处理
3. **推送频率限制**: 个推批量推送限制为每日 200 万次推送，需合理规划
4. **不修改现有推送**: 保留 FCM/APNs 现有推送流程不变
5. **兼容 Rocket.Chat 官方 App**: 官方 App 使用 FCM 推送，不做改动

### 4.2 假设

1. 服务器可正常访问个推 API（`restapi.getui.com`）
2. 个推应用已在个推开发者平台创建并获取 AppID/AppKey/MasterSecret
3. 频道启用个推推送后，该频道的所有消息都会被推送（不区分消息类型）
4. 个推 Token（CID）由客户端 SDK 生成并上报

## 5. 依赖关系

| 依赖项 | 说明 |
|--------|------|
| 个推开发者账号 | 需在 https://dev.getui.com 注册并创建应用 |
| 个推 REST API v2 | https://docs.getui.com/getui/server/rest_v2/push/ |
| Rocket.Chat 服务端 | 需要修改核心代码 |
| 个推 App（客户端） | 需要 uni-app 客户端集成个推 SDK 生成 CID |
