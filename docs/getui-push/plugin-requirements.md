# 个推（GeTui）推送集成 — RC App 插件需求文档

> **本文档为插件版需求文档**，对应嵌入式方案的 `server-requirements.md`。
> 可行性分析见 [`plugin-feasibility.md`](./plugin-feasibility.md)。

## 1. 项目背景

### 1.1 项目概述

本项目将以 **Rocket.Chat Apps-Engine 插件**形式开发个推推送集成功能，不修改任何 Rocket.Chat 核心服务端代码。插件可在 RC 管理后台直接上传安装，升级和卸载均无需重启服务端。

### 1.2 现有架构

Rocket.Chat Apps-Engine 提供以下机制供插件使用：

```
插件事件：IPostMessageSent（消息发送后）、IPostUserLoggedOut（用户登出后）

插件 API：
  GET/POST/DELETE /api/apps/public/{appId}/{path}
  → 支持 authRequired: true（用户鉴权）

插件存储：IPersistence（键值+关联记录存储）

外部 HTTP：IHttp（调用个推 REST API）

插件设置：ISettingsExtend（Admin → Apps → GeTui Push）
```

### 1.3 与嵌入式方案的主要差异

| 事项 | 嵌入式方案 | 插件方案 |
|------|-----------|----------|
| Token API 路径 | `/api/v1/getui.token` | `/api/apps/public/{appId}/getui-token` |
| 房间推送开关 | Admin UI 原生设置 | Slash 命令 `/getui-push` |
| 用户通知偏好过滤 | ✅ 支持 | ⚠️ 不支持（已知限制） |
| 数据存储 | 独立 MongoDB 集合 | Apps-Engine IPersistence |
| 服务端改动 | 8+ 个核心文件 | **零改动** |

## 2. 需求总览

### 2.1 功能需求列表

| 编号 | 功能 | 优先级 | 说明 |
|------|------|--------|------|
| PR-001 | 集成个推 REST API | P0 | 使用个推 REST API v2 批量推送 |
| PR-002 | 频道推送开关 | P0 | 通过 Slash 命令切换频道推送状态 |
| PR-003 | 个推 Token API | P0 | 接收和管理手机 App 的推送 Token (CID) |
| PR-004 | 消息推送触发 | P0 | 消息发送后通过个推推送给房间成员 |
| PR-005 | 插件全局配置 | P0 | 在 Apps 设置页面配置个推 AppID/AppKey/MasterSecret |
| PR-006 | 推送内容构建 | P1 | 构建适配个推格式的推送内容 |
| PR-007 | 推送日志 | P1 | 记录推送日志以便故障排查 |
| PR-008 | Token 上限管理 | P1 | 限制每个用户的 Token 数量（默认上限 1） |
| PR-009 | 用户登出清理 | P1 | 用户登出时自动清理该用户的个推 Token |

### 2.2 非功能需求

| 编号 | 需求 | 说明 |
|------|------|------|
| NR-001 | 性能 | 单次批量推送应在 5 秒内完成（个推 API 限制单次最多 200 个 CID） |
| NR-002 | 可靠性 | 推送失败时进行重试，并记录日志 |
| NR-003 | 安全性 | MasterSecret 以密码类型字段存储；Token API 必须鉴权 |
| NR-004 | 兼容性 | 不影响 RC 现有 FCM/APNs 推送功能 |
| NR-005 | 可维护性 | 插件可在不重启服务端的情况下独立更新和卸载 |

## 3. 详细需求

### 3.1 PR-001: 集成个推 REST API

#### 3.1.1 描述

使用 Apps-Engine 的 `IHttp` 接口调用个推 REST API v2，实现消息推送。

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

- **FR-001-1**: 使用 `IHttp.post()` 实现个推鉴权 Token 获取
- **FR-001-2**: 将鉴权 Token 缓存到 `IPersistence`，过期前自动刷新（安全边际 5 分钟）
- **FR-001-3**: 实现批量推送接口调用，支持单次最多 200 个 CID
- **FR-001-4**: 当 CID 数量超过 200 时，自动分批发送
- **FR-001-5**: 实现推送失败重试机制（最多 3 次，指数退避）
- **FR-001-6**: 处理个推返回的无效 CID，自动清理 `IPersistence` 中的对应 Token

#### 3.1.4 验收标准

- 能成功获取个推鉴权 Token 并缓存
- 能向指定 CID 列表批量推送消息
- CID 超过 200 时能自动分批
- 无效 CID 能被自动清理
- 个推 API 返回错误时有重试机制

---

### 3.2 PR-002: 频道推送开关

#### 3.2.1 描述

通过 Slash 命令管理员可以为当前频道启用或禁用个推推送。

> **说明**：由于 Apps-Engine 不能修改 RC 核心 Admin UI 中的房间设置，此功能通过 Slash 命令实现，房间开关状态存储在 `room.customFields.getuiPushEnabled` 中。

#### 3.2.2 Slash 命令规范

```
/getui-push enable      — 为当前频道启用个推推送
/getui-push disable     — 为当前频道禁用个推推送
/getui-push status      — 查看当前频道的推送状态
```

#### 3.2.3 功能要求

- **FR-002-1**: 注册 `/getui-push` Slash 命令，支持 `enable` / `disable` / `status` 子命令
- **FR-002-2**: 执行 `enable`/`disable` 时，通过 `IRoomBuilder.setCustomFields({ getuiPushEnabled: true/false })` 写入房间
- **FR-002-3**: 仅具有 `edit-room` 或更高权限的用户可执行 `enable`/`disable`
- **FR-002-4**: 执行后向当前频道回复确认消息
- **FR-002-5**: 默认状态为未启用（`customFields.getuiPushEnabled` 不存在或为 `false`）

#### 3.2.4 验收标准

- 管理员可用 Slash 命令启用/禁用频道推送
- 权限不足的用户收到权限错误提示
- `status` 命令正确显示当前状态
- 状态变更后对应频道的推送行为立即生效

---

### 3.3 PR-003: 个推 Token API

#### 3.3.1 描述

提供 App 公开端点，允许手机 App 注册和管理个推推送 Token（CID）。

#### 3.3.2 API 定义

##### 注册/更新 Token

```
POST /api/apps/public/{appId}/getui-token

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
  "token": "<个推CID>",
  "userId": "<用户ID>"
}
```

##### 删除 Token

```
DELETE /api/apps/public/{appId}/getui-token

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

##### 获取插件信息（供手机 App 发现端点路径）

```
GET /api/apps/public/{appId}/info

Response (200):
{
  "appId": "<appId>",
  "version": "<version>",
  "tokenEndpoint": "/api/apps/public/{appId}/getui-token"
}
```

#### 3.3.3 功能要求

- **FR-003-1**: 端点设置 `authRequired: true`，`request.user` 即为当前登录用户
- **FR-003-2**: 同一用户的 Token 数量默认上限为 1，超出时替换最早的 Token
- **FR-003-3**: Token 上限可通过插件设置 `Getui_Max_Tokens_Per_User` 调整
- **FR-003-4**: 注册时如果相同 Token 已存在（属于同一用户），则更新而非新建
- **FR-003-5**: 注册时如果相同 Token 属于其他用户，应先从其他用户移除再注册给当前用户
- **FR-003-6**: 删除 Token 时仅允许删除属于当前用户的 Token
- **FR-003-7**: 用户登出时（`IPostUserLoggedOut`）自动清理该用户的所有 Token

#### 3.3.4 数据存储（IPersistence）

使用 Apps-Engine `IPersistence` 存储个推 Token，数据结构如下：

```typescript
interface GetuiTokenRecord {
  token: string;              // 个推 CID
  appName: string;            // 应用标识
  userId: string;             // 用户 ID
  platform: 'android' | 'harmony';  // 平台类型
  createdAt: string;          // ISO 日期字符串
  updatedAt: string;          // ISO 日期字符串
}
```

关联策略：
- 按用户查询：`RocketChatAssociationModel.USER` + `userId`
- 按 Token 查询：`RocketChatAssociationModel.MISC` + `token:${cid}`
- 每条记录同时建立两个关联

#### 3.3.5 验收标准

- 已登录用户可注册个推 Token
- 同一用户超过 Token 上限时自动淘汰旧 Token
- 相同 Token 不会重复注册
- Token 可被正确删除
- 用户登出后 Token 被自动清理

---

### 3.4 PR-004: 消息推送触发

#### 3.4.1 描述

通过实现 `IPostMessageSent` 接口，在消息发送后触发个推推送。

#### 3.4.2 推送流程

```
IPostMessageSent.executePostMessageSent()
  → 检查 message.room.customFields?.getuiPushEnabled 是否为 true
  → 检查插件全局设置 Getui_Enabled 是否为 true
  → 系统消息（message.type 不为 undefined）跳过
  → 获取房间成员列表（read.getRoomReader().getMembers(roomId)）
  → 排除消息发送者（message.sender.id）
  → 从 IPersistence 查询这些用户的个推 Token（CID）
  → 构建推送内容
  → 调用个推批量推送 API（IHttp）
```

#### 3.4.3 功能要求

- **FR-004-1**: 实现 `IPostMessageSent` 接口，在 `executePostMessageSent` 中处理推送逻辑
- **FR-004-2**: 在 `checkPostMessageSent` 中进行快速前置检查（enabled 和 getuiPushEnabled），避免不必要的执行
- **FR-004-3**: 获取房间所有成员，排除消息发送者
- **FR-004-4**: 批量查询成员的个推 Token，仅向有有效 Token 的用户发送推送
- **FR-004-5**: 推送对所有用户消息都触发（不仅限于 @提及）
- **FR-004-6**: [已知限制] 不检查用户的移动通知偏好设置（Apps-Engine 未暴露此字段）

#### 3.4.4 推送内容格式

```json
{
  "notification": {
    "title": "[频道名称] 发送者名称",
    "body": "消息内容（截取前 200 字符）",
    "click_type": "payload"
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
- 系统消息不触发推送
- 推送内容正确包含频道名、发送者、消息预览

---

### 3.5 PR-005: 插件全局配置

#### 3.5.1 描述

在 Admin → Apps → GeTui Push 页面添加个推相关配置项。

#### 3.5.2 配置项列表

| 设置项 ID | 类型 | 默认值 | 说明 |
|-----------|------|--------|------|
| `Getui_Enabled` | boolean | false | 启用/禁用个推推送服务 |
| `Getui_AppId` | string | '' | 个推应用 AppID |
| `Getui_AppKey` | string | '' | 个推应用 AppKey |
| `Getui_MasterSecret` | password | '' | 个推应用 MasterSecret（密码类型，显示为星号） |
| `Getui_Api_Url` | string | 'https://restapi.getui.com/v2' | 个推 REST API 基础 URL |
| `Getui_Max_Tokens_Per_User` | int | 1 | 每用户最大 Token 数量 |

#### 3.5.3 功能要求

- **FR-005-1**: 在插件 `extendConfiguration` 中通过 `ISettingsExtend.provideSetting()` 注册以上配置项
- **FR-005-2**: `Getui_MasterSecret` 使用 `SettingType.PASSWORD` 类型
- **FR-005-3**: 配置项变更后应即时生效（通过 `onSettingUpdated` 回调处理）

#### 3.5.4 验收标准

- 在 Admin → Apps → GeTui Push 页面可见所有配置项
- MasterSecret 字段以密码形式显示
- 配置变更无需重启即可生效

---

### 3.6 PR-006: 推送内容构建

#### 3.6.1 描述

将 Rocket.Chat 消息转换为个推推送所需的内容格式。

#### 3.6.2 功能要求

- **FR-006-1**: 通知标题格式：`[频道名称] 发送者名称`
- **FR-006-2**: 通知正文：消息文本前 200 个字符，超出部分用 `...` 截断
- **FR-006-3**: 附件消息显示为 `[文件]`、`[图片]`、`[视频]` 等占位文本
- **FR-006-4**: `transmission`（自定义数据）中包含 `host`、`rid`、`msgId`、`roomName`、`senderName`
- **FR-006-5**: 服务器地址（`host`）通过读取服务端公开设置 `Site_Url` 获取（`read.getEnvironmentReader().getServerSettings().getValueById('Site_Url')`）

#### 3.6.3 验收标准

- 推送标题、正文格式正确
- 长消息正确截断
- 附件消息有合适的占位文本

---

### 3.7 PR-007: 推送日志

#### 3.7.1 描述

使用 Apps-Engine 提供的 `ILogger` 记录推送操作日志。

#### 3.7.2 功能要求

- **FR-007-1**: 使用 `this.getLogger()` 记录日志
- **FR-007-2**: 记录推送触发事件：房间 ID、消息 ID、目标用户数、CID 数
- **FR-007-3**: 记录 API 调用结果：鉴权、创建消息体、批量推送
- **FR-007-4**: 记录错误详情：API 错误码、错误描述、重试次数
- **FR-007-5**: 日志不包含敏感信息（如 MasterSecret）

---

### 3.8 PR-008: Token 上限管理

#### 3.8.1 描述

限制每个用户可注册的个推 Token 数量。

#### 3.8.2 功能要求

- **FR-008-1**: 默认每用户最多 1 个 Token
- **FR-008-2**: 超出上限时，移除最早创建的 Token
- **FR-008-3**: 上限可通过 `Getui_Max_Tokens_Per_User` 全局设置调整

---

### 3.9 PR-009: 用户登出清理 Token

#### 3.9.1 描述

监听用户登出事件，自动清理该用户的所有个推 Token。

#### 3.9.2 功能要求

- **FR-009-1**: 实现 `IPostUserLoggedOut` 接口
- **FR-009-2**: 在 `executePostUserLoggedOut` 中删除该用户的所有 Token
- **FR-009-3**: 清理失败（如存储异常）时记录错误日志，不影响登出流程

## 4. 约束与假设

### 4.1 约束

1. **无核心代码修改**: 插件不修改任何 RC 核心文件
2. **Apps-Engine 版本**: 目标插件 API 兼容当前 Rocket.Chat 版本的 Apps-Engine
3. **批量推送限制**: 单次请求最多 200 个 CID，需分批处理
4. **用户账号限制**: 服务端不允许直接注册用户；所有用户必须来自已配置的 OAuth 提供商
5. **用户通知偏好**: 插件不检查用户的移动通知偏好（已知限制），依赖用户主动删除 Token 退出推送

### 4.2 假设

1. 服务器可正常访问个推 API（`restapi.getui.com`）
2. 个推应用已在个推开发者平台创建并获取 AppID/AppKey/MasterSecret
3. 频道启用个推推送后，该频道的所有用户消息都会被推送
4. 个推 Token（CID）由客户端 SDK 生成并上报

## 5. 依赖关系

| 依赖项 | 说明 |
|--------|------|
| Rocket.Chat Apps-Engine | 插件运行环境（无需额外安装） |
| 个推开发者账号 | 需在 https://dev.getui.com 注册并创建应用 |
| 个推 REST API v2 | https://docs.getui.com/getui/server/rest_v2/push/ |
| 个推 App（客户端） | 需要 uni-app 客户端集成个推 SDK 生成 CID |
