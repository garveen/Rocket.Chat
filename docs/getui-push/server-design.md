# 个推（GeTui）推送集成 — 服务端设计文档

## 1. 架构概述

### 1.1 系统架构图

```
┌─────────────────────────────────────────────────────────────────────┐
│                         Rocket.Chat 服务端                           │
│                                                                      │
│  ┌──────────────────┐    ┌─────────────────┐   ┌────────────────┐   │
│  │  消息处理层       │    │  推送服务层      │   │  管理配置层     │   │
│  │                  │    │                 │   │                │   │
│  │ afterSaveMessage │───→│ GetuiPushService│   │  Settings      │   │
│  │    Hook          │    │                 │   │  (Getui_*)     │   │
│  │                  │    │ ┌─────────────┐ │   │                │   │
│  └──────────────────┘    │ │ AuthManager │ │   └────────────────┘   │
│                          │ └─────────────┘ │                        │
│  ┌──────────────────┐    │ ┌─────────────┐ │   ┌────────────────┐   │
│  │  REST API 层      │    │ │ BatchPusher │ │   │  数据层         │   │
│  │                  │    │ └─────────────┘ │   │                │   │
│  │ POST getui.token │    │ ┌─────────────┐ │   │ GetuiPushToken │   │
│  │ DELETE getui.token│    │ │ ContentBuild│ │   │ (MongoDB)      │   │
│  │                  │    │ └─────────────┘ │   │                │   │
│  └──────────────────┘    └────────┬────────┘   └────────────────┘   │
│                                   │                                  │
└───────────────────────────────────┼──────────────────────────────────┘
                                    │ HTTPS
                                    ▼
                          ┌─────────────────┐
                          │   个推 REST API   │
                          │  restapi.getui.com│
                          └─────────────────┘
```

### 1.2 核心模块

| 模块 | 职责 | 位置 |
|------|------|------|
| GetuiPushService | 个推推送核心服务，管理鉴权、批量推送 | `apps/meteor/server/services/getui-push/` |
| GetuiTokenAPI | Token 注册/删除 REST API | `apps/meteor/app/api/server/v1/getui-push.ts` |
| GetuiPushToken Model | 数据库模型 | `packages/models/src/models/GetuiPushToken.ts` |
| Room 扩展 | 房间 getuiPushEnabled 字段 | 多处修改 |
| Settings 扩展 | 个推全局配置 | `apps/meteor/server/settings/push.ts` |
| Hook 扩展 | 消息推送触发 | `apps/meteor/app/lib/server/lib/sendNotificationsOnMessage.ts` |

### 1.3 技术选型

- **HTTP 客户端**: 使用 Rocket.Chat 内置的 `@rocket.chat/server-fetch`（基于 node-fetch，带 SSRF 防护）
- **日志**: 使用 `@rocket.chat/logger` 的 `Logger` 类
- **数据库**: MongoDB，遵循现有模型模式
- **配置管理**: 使用 Rocket.Chat `settingsRegistry` 标准设置机制

## 2. 详细设计

### 2.1 数据模型

#### 2.1.1 IGetuiPushToken 类型定义

**文件**: `packages/core-typings/src/IGetuiPushToken.ts`

```typescript
import type { IRocketChatRecord } from './IRocketChatRecord';

export interface IGetuiPushToken extends IRocketChatRecord {
  /** 个推客户端标识（CID） */
  token: string;

  /** 应用标识符 */
  appName: string;

  /** 关联的用户 ID */
  userId: string;

  /** 客户端平台 */
  platform: 'android' | 'harmony';

  /** 是否启用 */
  enabled: boolean;

  /** 创建时间 */
  createdAt: Date;
}
```

#### 2.1.2 IRoom 扩展

**文件**: `packages/core-typings/src/IRoom.ts`（新增字段）

```typescript
export interface IRoom extends IRocketChatRecord {
  // ... existing fields ...

  /** 是否启用个推推送 */
  getuiPushEnabled?: boolean;
}
```

#### 2.1.3 MongoDB 集合设计

**集合名**: `getui_push_tokens`

| 字段 | 类型 | 索引 | 说明 |
|------|------|------|------|
| `_id` | ObjectId | 主键 | 文档 ID |
| `token` | string | 唯一索引 | 个推 CID |
| `appName` | string | - | 应用标识 |
| `userId` | string | 普通索引 | 用户 ID |
| `platform` | string | - | android/harmony |
| `enabled` | boolean | - | 是否启用 |
| `createdAt` | Date | - | 创建时间 |
| `_updatedAt` | Date | - | 更新时间 |

索引定义：
```javascript
db.getui_push_tokens.createIndex({ token: 1 }, { unique: true });
db.getui_push_tokens.createIndex({ userId: 1 });
db.getui_push_tokens.createIndex({ userId: 1, createdAt: 1 });
```

### 2.2 GetuiPushToken 数据库模型

**文件**: `packages/models/src/models/GetuiPushToken.ts`

```typescript
import type { IGetuiPushToken } from '@rocket.chat/core-typings';
import type { IGetuiPushTokenModel } from '@rocket.chat/model-typings';
import type { Db, IndexDescription, InsertOneResult, DeleteResult, UpdateResult } from 'mongodb';

import { BaseRaw } from './BaseRaw';

export class GetuiPushTokenRaw extends BaseRaw<IGetuiPushToken> implements IGetuiPushTokenModel {
  constructor(db: Db) {
    super(db, 'getui_push_tokens');
  }

  protected modelIndexes(): IndexDescription[] {
    return [
      { key: { token: 1 }, unique: true },
      { key: { userId: 1 } },
      { key: { userId: 1, createdAt: 1 } },
    ];
  }

  async findByUserId(userId: string): Promise<IGetuiPushToken[]> {
    return this.find({ userId, enabled: true }).toArray();
  }

  async findByToken(token: string): Promise<IGetuiPushToken | null> {
    return this.findOne({ token });
  }

  async findByUserIds(userIds: string[]): Promise<IGetuiPushToken[]> {
    return this.find({
      userId: { $in: userIds },
      enabled: true,
    }).toArray();
  }

  async upsertToken(data: {
    token: string;
    userId: string;
    appName: string;
    platform: 'android' | 'harmony';
  }): Promise<UpdateResult> {
    const now = new Date();
    return this.updateOne(
      { token: data.token },
      {
        $set: {
          userId: data.userId,
          appName: data.appName,
          platform: data.platform,
          enabled: true,
          _updatedAt: now,
        },
        $setOnInsert: {
          createdAt: now,
        },
      },
      { upsert: true },
    );
  }

  async removeByToken(token: string, userId: string): Promise<DeleteResult> {
    return this.deleteOne({ token, userId });
  }

  async removeByUserId(userId: string): Promise<DeleteResult> {
    return this.deleteMany({ userId });
  }

  async removeOldestByUserId(userId: string, keepCount: number): Promise<void> {
    const tokens = await this.find(
      { userId },
      { sort: { createdAt: -1 }, skip: keepCount, projection: { _id: 1 } },
    ).toArray();

    if (tokens.length > 0) {
      await this.deleteMany({ _id: { $in: tokens.map((t) => t._id) } });
    }
  }

  async removeByTokens(tokens: string[]): Promise<DeleteResult> {
    return this.deleteMany({ token: { $in: tokens } });
  }
}
```

### 2.3 GetuiPushService 核心服务

**文件**: `apps/meteor/server/services/getui-push/service.ts`

#### 2.3.1 类结构

```typescript
import { ServiceClassInternal } from '@rocket.chat/core-services';
import { Logger } from '@rocket.chat/logger';

const logger = new Logger('GetuiPush');

export class GetuiPushService extends ServiceClassInternal {
  private authToken: string | null = null;
  private authTokenExpiry: number = 0;

  // === 鉴权管理 ===

  /**
   * 获取个推鉴权 Token
   * - 如果已缓存且未过期，直接返回
   * - 否则调用个推 auth API 获取新 Token
   * - Token 缓存时间为返回的 expire_time 减去 5 分钟安全边际
   */
  private async getAuthToken(): Promise<string>;

  /**
   * 生成鉴权签名
   * sign = SHA256(appKey + timestamp + masterSecret)
   */
  private generateSign(timestamp: string): string;

  // === 推送发送 ===

  /**
   * 发送批量推送
   * @param cids - 目标 CID 列表
   * @param notification - 通知内容
   *
   * 流程:
   * 1. 获取鉴权 Token
   * 2. 创建消息体 (POST /push/list/message)
   * 3. 按 200 个一批分割 CID
   * 4. 对每批调用推送接口 (POST /push/list/cid)
   * 5. 处理结果，清理无效 CID
   */
  async sendBatchPush(
    cids: string[],
    notification: GetuiNotification,
  ): Promise<GetuiPushResult>;

  /**
   * 创建消息体
   * POST /v2/{appId}/push/list/message
   */
  private async createMessage(
    notification: GetuiNotification,
  ): Promise<string>; // returns taskId

  /**
   * 执行批量推送
   * POST /v2/{appId}/push/list/cid
   */
  private async pushToCids(
    taskId: string,
    cids: string[],
    isAsync: boolean,
  ): Promise<GetuiPushBatchResult>;

  // === Token 管理 ===

  /**
   * 注册个推 Token
   * - 检查 Token 是否已存在
   * - 如已存在且属于其他用户，先移除再注册
   * - 检查用户 Token 数量，超限则移除最老的
   * - 执行 upsert 操作
   */
  async registerToken(params: {
    token: string;
    userId: string;
    appName: string;
    platform: 'android' | 'harmony';
  }): Promise<IGetuiPushToken>;

  /**
   * 删除个推 Token
   */
  async removeToken(token: string, userId: string): Promise<void>;

  /**
   * 清除用户的所有个推 Token
   */
  async clearUserTokens(userId: string): Promise<void>;

  // === 消息通知 ===

  /**
   * 处理新消息的个推推送
   * 由 afterSaveMessage Hook 调用
   *
   * @param message - 消息对象
   * @param room - 房间对象
   *
   * 流程:
   * 1. 检查房间 getuiPushEnabled
   * 2. 检查全局 Getui_Enabled
   * 3. 获取房间成员列表（排除发送者）
   * 4. 查询成员的个推 Token
   * 5. 构建推送内容
   * 6. 调用 sendBatchPush
   */
  async handleNewMessage(message: IMessage, room: IRoom): Promise<void>;

  /**
   * 构建推送通知内容
   */
  private buildNotification(
    message: IMessage,
    room: IRoom,
  ): GetuiNotification;
}
```

#### 2.3.2 鉴权流程

```
┌──────────────────┐
│ getAuthToken()   │
└────────┬─────────┘
         │
         ▼
  ┌──────────────┐   是    ┌─────────────┐
  │ Token 已缓存 │────────→│ 返回缓存Token│
  │ 且未过期？    │         └─────────────┘
  └──────┬───────┘
         │ 否
         ▼
  ┌──────────────────────────────┐
  │ POST /v2/{appId}/auth        │
  │ Body: {                      │
  │   sign: SHA256(appKey +      │
  │         timestamp +          │
  │         masterSecret),       │
  │   timestamp: "ms时间戳",      │
  │   appkey: appKey             │
  │ }                            │
  └──────────────┬───────────────┘
                 │
                 ▼
  ┌──────────────────────────────┐
  │ 缓存 Token                   │
  │ 设置过期时间 =                │
  │   expire_time - 5min         │
  └──────────────────────────────┘
```

#### 2.3.3 批量推送流程

```
┌─────────────────────────┐
│ sendBatchPush(cids, msg)│
└───────────┬─────────────┘
            │
            ▼
  ┌─────────────────────┐
  │ 1. getAuthToken()   │
  └─────────┬───────────┘
            │
            ▼
  ┌─────────────────────────────────┐
  │ 2. createMessage(notification)  │
  │    POST /v2/{appId}/push/list/  │
  │         message                 │
  │    → 返回 taskId               │
  └─────────┬───────────────────────┘
            │
            ▼
  ┌──────────────────────────────────┐
  │ 3. 将 CID 分批 (每批 ≤ 200)     │
  │    [batch1, batch2, ...]        │
  └─────────┬────────────────────────┘
            │
            ▼
  ┌──────────────────────────────────┐
  │ 4. 对每批执行:                   │
  │    pushToCids(taskId, batchCids) │
  │    POST /v2/{appId}/push/list/  │
  │         cid                     │
  │    Body: {                      │
  │      audience: { cid: [...] },  │
  │      taskid: taskId,            │
  │      is_async: true             │
  │    }                            │
  └─────────┬────────────────────────┘
            │
            ▼
  ┌──────────────────────────────────┐
  │ 5. 处理结果                      │
  │    - 收集成功/失败 CID          │
  │    - 清理无效 CID               │
  │    - 记录日志                    │
  └──────────────────────────────────┘
```

#### 2.3.4 错误处理与重试策略

```typescript
const RETRY_CONFIG = {
  maxRetries: 3,
  baseDelay: 1000,  // 1 秒
  maxDelay: 10000,  // 10 秒
};

// 重试逻辑
async function withRetry<T>(
  fn: () => Promise<T>,
  context: string,
): Promise<T> {
  for (let attempt = 0; attempt <= RETRY_CONFIG.maxRetries; attempt++) {
    try {
      return await fn();
    } catch (error) {
      if (attempt === RETRY_CONFIG.maxRetries) throw error;

      const delay = Math.min(
        RETRY_CONFIG.baseDelay * Math.pow(2, attempt),
        RETRY_CONFIG.maxDelay,
      );

      logger.warn(
        `${context} failed (attempt ${attempt + 1}/${RETRY_CONFIG.maxRetries}), ` +
        `retrying in ${delay}ms: ${error.message}`
      );

      await sleep(delay);
    }
  }
}
```

错误码处理策略:

| 个推错误码 | 含义 | 处理 |
|-----------|------|------|
| 10001 | Token 无效/过期 | 清除缓存，重新鉴权 |
| 10002 | CID 不存在 | 从数据库移除该 CID |
| 其他 | API 错误 | 重试或记录错误 |

### 2.4 REST API 实现

**文件**: `apps/meteor/app/api/server/v1/getui-push.ts`

#### 2.4.1 Token 注册端点

```typescript
import { API } from '../api';

API.v1.addRoute(
  'getui.token',
  { authRequired: true },
  {
    async post() {
      const { token, appName, platform } = this.bodyParams;

      // 参数校验
      if (!token || typeof token !== 'string') {
        return API.v1.failure('Token is required');
      }
      if (!appName || typeof appName !== 'string') {
        return API.v1.failure('appName is required');
      }
      if (!['android', 'harmony'].includes(platform)) {
        return API.v1.failure('platform must be "android" or "harmony"');
      }

      // 检查个推是否启用
      if (!settings.get('Getui_Enabled')) {
        return API.v1.failure('GeTui push is not enabled');
      }

      const result = await getuiPushService.registerToken({
        token,
        userId: this.userId,
        appName,
        platform,
      });

      return API.v1.success({ result });
    },

    async delete() {
      const { token } = this.bodyParams;

      if (!token || typeof token !== 'string') {
        return API.v1.failure('Token is required');
      }

      await getuiPushService.removeToken(token, this.userId);

      return API.v1.success();
    },
  },
);
```

#### 2.4.2 REST API 类型定义

**文件**: `packages/rest-typings/src/v1/getui-push.ts`

```typescript
export type GetuiPushTokenEndpoint = {
  '/v1/getui.token': {
    POST: (params: {
      token: string;
      appName: string;
      platform: 'android' | 'harmony';
    }) => {
      result: IGetuiPushToken;
    };
    DELETE: (params: {
      token: string;
    }) => void;
  };
};
```

### 2.5 房间设置扩展

#### 2.5.1 IRoom 类型修改

**文件**: `packages/core-typings/src/IRoom.ts`

在 `IRoom` 接口中新增：

```typescript
/** 是否启用个推推送 - 当为 true 时该频道的消息会通过个推推送 */
getuiPushEnabled?: boolean;
```

#### 2.5.2 房间设置保存

**文件**: `apps/meteor/app/channel-settings/server/methods/saveRoomSettings.ts`

在 `RoomSettings` 类型中新增 `getuiPushEnabled`:

```typescript
type RoomSettings = {
  // ... existing fields ...
  getuiPushEnabled: boolean;
};
```

新增保存函数：

```typescript
async function saveGetuiPushEnabled(rid: string, value: boolean): Promise<void> {
  if (value !== true && value !== false) {
    throw new Meteor.Error('error-invalid-value', 'Invalid value for getuiPushEnabled');
  }
  await Rooms.setGetuiPushEnabledById(rid, value);
}
```

在设置项映射中添加:

```typescript
const settingHandlers: Record<string, (rid: string, value: any) => Promise<void>> = {
  // ... existing handlers ...
  getuiPushEnabled: saveGetuiPushEnabled,
};
```

#### 2.5.3 Room 模型扩展

**文件**: `packages/models/src/models/Rooms.ts`

```typescript
async setGetuiPushEnabledById(rid: string, enabled: boolean): Promise<UpdateResult> {
  return this.updateOne(
    { _id: rid },
    { $set: { getuiPushEnabled: enabled } },
  );
}
```

### 2.6 消息推送触发 Hook

**文件**: `apps/meteor/app/lib/server/lib/sendNotificationsOnMessage.ts`

在 `sendAllNotifications` 函数的末尾（或通过单独的 Hook）添加个推推送触发逻辑:

```typescript
// 在 sendAllNotifications 函数或 afterSaveMessage 回调中添加:

async function triggerGetuiPush(message: IMessage, room: IRoom): Promise<void> {
  // 前置条件检查
  if (!settings.get<boolean>('Getui_Enabled')) {
    return;
  }

  if (!room.getuiPushEnabled) {
    return;
  }

  // 系统消息不推送
  if (message.t) {
    return;
  }

  // 异步执行推送，不阻塞消息发送
  setImmediate(async () => {
    try {
      await getuiPushService.handleNewMessage(message, room);
    } catch (error) {
      logger.error('Failed to trigger GeTui push:', error);
    }
  });
}
```

### 2.7 全局配置

**文件**: `apps/meteor/server/settings/push.ts`

在现有 Push 设置组中添加 GeTui 子分组:

```typescript
await settingsRegistry.addGroup('Push', async function () {
  // ... 现有 Push 设置 ...

  await this.section('GeTui', async function () {
    await this.add('Getui_Enabled', false, {
      type: 'boolean',
      public: true,
      i18nLabel: 'GeTui_Enabled',
      i18nDescription: 'GeTui_Enabled_Description',
    });

    await this.add('Getui_AppId', '', {
      type: 'string',
      public: false,
      i18nLabel: 'GeTui_AppId',
      enableQuery: { _id: 'Getui_Enabled', value: true },
    });

    await this.add('Getui_AppKey', '', {
      type: 'string',
      public: false,
      i18nLabel: 'GeTui_AppKey',
      enableQuery: { _id: 'Getui_Enabled', value: true },
    });

    await this.add('Getui_MasterSecret', '', {
      type: 'password',
      public: false,
      secret: true,
      i18nLabel: 'GeTui_MasterSecret',
      enableQuery: { _id: 'Getui_Enabled', value: true },
    });

    await this.add('Getui_Api_Url', 'https://restapi.getui.com/v2', {
      type: 'string',
      public: false,
      i18nLabel: 'GeTui_Api_Url',
      enableQuery: { _id: 'Getui_Enabled', value: true },
    });

    await this.add('Getui_Max_Tokens_Per_User', 1, {
      type: 'int',
      public: false,
      i18nLabel: 'GeTui_Max_Tokens_Per_User',
      i18nDescription: 'GeTui_Max_Tokens_Per_User_Description',
      enableQuery: { _id: 'Getui_Enabled', value: true },
    });
  });
});
```

### 2.8 i18n 国际化

**文件**: `apps/meteor/packages/rocketchat-i18n/i18n/en.i18n.json`（英文）及 `zh-CN.i18n.json`（中文）

```json
// en.i18n.json
{
  "GeTui_Enabled": "Enable GeTui Push",
  "GeTui_Enabled_Description": "Enable GeTui push notification service for Chinese domestic push",
  "GeTui_AppId": "GeTui App ID",
  "GeTui_AppKey": "GeTui App Key",
  "GeTui_MasterSecret": "GeTui Master Secret",
  "GeTui_Api_Url": "GeTui API URL",
  "GeTui_Max_Tokens_Per_User": "Max tokens per user",
  "GeTui_Max_Tokens_Per_User_Description": "Maximum number of GeTui push tokens per user",
  "GeTui_Push_Enabled_For_Channel": "Enable GeTui Push for this channel",
  "GeTui_Push_Not_Enabled": "GeTui push is not enabled"
}

// zh-CN.i18n.json
{
  "GeTui_Enabled": "启用个推推送",
  "GeTui_Enabled_Description": "启用个推推送服务，用于国内推送通道",
  "GeTui_AppId": "个推 App ID",
  "GeTui_AppKey": "个推 App Key",
  "GeTui_MasterSecret": "个推 Master Secret",
  "GeTui_Api_Url": "个推 API 地址",
  "GeTui_Max_Tokens_Per_User": "每用户最大 Token 数",
  "GeTui_Max_Tokens_Per_User_Description": "每个用户可注册的个推推送 Token 最大数量",
  "GeTui_Push_Enabled_For_Channel": "为此频道启用个推推送",
  "GeTui_Push_Not_Enabled": "个推推送未启用"
}
```

## 3. 文件变更清单

### 3.1 新增文件

| 文件路径 | 说明 |
|----------|------|
| `packages/core-typings/src/IGetuiPushToken.ts` | 个推 Token 类型定义 |
| `packages/model-typings/src/models/IGetuiPushTokenModel.ts` | Token 模型接口定义 |
| `packages/models/src/models/GetuiPushToken.ts` | Token 数据库模型 |
| `apps/meteor/server/services/getui-push/service.ts` | 个推推送核心服务 |
| `apps/meteor/server/services/getui-push/logger.ts` | 推送服务日志 |
| `apps/meteor/server/services/getui-push/types.ts` | 服务内部类型定义 |
| `apps/meteor/app/api/server/v1/getui-push.ts` | REST API 端点 |
| `packages/rest-typings/src/v1/getui-push.ts` | REST API 类型定义 |

### 3.2 修改文件

| 文件路径 | 修改内容 |
|----------|----------|
| `packages/core-typings/src/IRoom.ts` | 新增 `getuiPushEnabled` 字段 |
| `packages/core-typings/src/index.ts` | 导出 `IGetuiPushToken` |
| `packages/model-typings/src/models/index.ts` | 导出 `IGetuiPushTokenModel` |
| `packages/models/src/index.ts` | 导出 `GetuiPushToken` |
| `apps/meteor/server/models/startup.ts` | 注册 GetuiPushToken 模型 |
| `apps/meteor/app/channel-settings/server/methods/saveRoomSettings.ts` | 支持 `getuiPushEnabled` 设置 |
| `packages/models/src/models/Rooms.ts` | 新增 `setGetuiPushEnabledById` 方法 |
| `apps/meteor/server/settings/push.ts` | 添加 GeTui 配置项 |
| `apps/meteor/app/lib/server/lib/sendNotificationsOnMessage.ts` | 添加个推推送触发逻辑 |
| `apps/meteor/packages/rocketchat-i18n/i18n/en.i18n.json` | 英文翻译 |
| `apps/meteor/packages/rocketchat-i18n/i18n/zh-CN.i18n.json` | 中文翻译 |

## 4. 推送时序图

### 4.1 Token 注册时序

```
个推App                   Rocket.Chat Server              MongoDB
  │                            │                            │
  │  POST /api/v1/getui.token  │                            │
  │  {token, appName, platform}│                            │
  │───────────────────────────→│                            │
  │                            │                            │
  │                            │  查找现有 Token             │
  │                            │──────────────────────────→ │
  │                            │  ← 结果                    │
  │                            │                            │
  │                            │  [如存在且属于其他用户]       │
  │                            │  移除旧记录                 │
  │                            │──────────────────────────→ │
  │                            │                            │
  │                            │  [检查 Token 数量上限]      │
  │                            │  查询用户 Token 数          │
  │                            │──────────────────────────→ │
  │                            │  ← 数量                    │
  │                            │                            │
  │                            │  [如超限] 移除最老 Token    │
  │                            │──────────────────────────→ │
  │                            │                            │
  │                            │  Upsert Token              │
  │                            │──────────────────────────→ │
  │                            │  ← 结果                    │
  │                            │                            │
  │  ← 200 { result: token }  │                            │
  │←───────────────────────────│                            │
```

### 4.2 消息推送时序

```
发送者       Rocket.Chat Server                个推 API            接收者设备
  │               │                              │                  │
  │ 发送消息       │                              │                  │
  │──────────────→│                              │                  │
  │               │                              │                  │
  │               │ afterSaveMessage             │                  │
  │               │ → triggerGetuiPush()         │                  │
  │               │                              │                  │
  │               │ ① 检查 room.getuiPushEnabled │                  │
  │               │ ② 查询房间成员               │                  │
  │               │ ③ 查询成员的个推 Token        │                  │
  │               │ ④ 构建推送内容               │                  │
  │               │                              │                  │
  │               │ ⑤ POST /auth                 │                  │
  │               │─────────────────────────────→│                  │
  │               │ ← authToken                  │                  │
  │               │                              │                  │
  │               │ ⑥ POST /push/list/message    │                  │
  │               │─────────────────────────────→│                  │
  │               │ ← taskId                     │                  │
  │               │                              │                  │
  │               │ ⑦ POST /push/list/cid        │                  │
  │               │   (每批 ≤ 200 CID)           │                  │
  │               │─────────────────────────────→│                  │
  │               │ ← 推送结果                   │                  │
  │               │                              │                  │
  │               │                              │ 推送通知          │
  │               │                              │────────────────→ │
  │               │                              │                  │
  │ ← 消息确认    │                              │                  │
  │←──────────────│                              │                  │
```

## 5. 安全考虑

### 5.1 密钥管理

- `Getui_MasterSecret` 使用 `password` 类型存储，配置 `secret: true`
- 日志中不输出 MasterSecret 和鉴权 Token
- 鉴权签名使用 SHA256 哈希生成

### 5.2 API 安全

- Token 注册/删除 API 需要用户登录认证（`authRequired: true`）
- 删除 Token 时校验 Token 归属（只能删除自己的 Token）
- Token 数据不对外暴露其他用户的信息

### 5.3 推送安全

- 推送内容尊重服务器隐私设置
- 仅对有订阅（阅读权限）的用户推送
- 系统消息（类型消息）不触发推送

## 6. 性能考虑

### 6.1 异步推送

- 推送操作通过 `setImmediate` 异步执行，不阻塞消息处理
- 批量推送使用 `is_async: true` 参数，个推异步处理

### 6.2 数据库查询优化

- CID 查询使用 `userId` 索引
- 批量查询使用 `$in` 操作符
- Token 查询使用唯一索引

### 6.3 Auth Token 缓存

- 鉴权 Token 缓存在内存中，减少 API 调用
- 提前 5 分钟刷新，避免过期导致推送失败

## 7. 监控与告警

### 7.1 日志输出

- **INFO**: 推送触发、推送成功、Token 注册/删除
- **WARN**: 推送重试、Token 上限淘汰
- **ERROR**: 推送失败、鉴权失败、API 错误

### 7.2 可观测指标（未来可扩展）

- 推送触发次数
- 推送成功/失败率
- 推送延迟
- Token 注册/清理数量
