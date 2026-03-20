# 个推（GeTui）推送集成 — RC App 插件技术设计文档

> **本文档为插件版设计文档**，对应嵌入式方案的 `server-design.md`。
> 可行性分析见 [`plugin-feasibility.md`](./plugin-feasibility.md)，需求见 [`plugin-requirements.md`](./plugin-requirements.md)。

## 1. 架构概述

### 1.1 系统架构图

```
┌──────────────────────────────────────────────────────────────────┐
│                      Rocket.Chat 服务端                           │
│                                                                   │
│  ┌─────────────────┐      ┌─────────────────────────────────┐    │
│  │  RC Core        │      │  GeTui Push RC App (插件)        │    │
│  │                 │      │                                  │    │
│  │ 消息发送         │─────→│  IPostMessageSent                │    │
│  │ 用户登出         │─────→│  IPostUserLoggedOut              │    │
│  │                 │      │                                  │    │
│  │ Apps-Engine API │←────→│  GetuiPushApp                    │    │
│  │ (IPersistence,  │      │  ├─ GetuiAuthService             │    │
│  │  IHttp, IRead,  │      │  ├─ GetuiPushService             │    │
│  │  IModify...)    │      │  ├─ TokenService                 │    │
│  │                 │      │  ├─ ContentBuilder               │    │
│  │ Apps API 路由    │←────→│  └─ ApiEndpoints                 │    │
│  │ /api/apps/...   │      │      ├─ POST getui-token         │    │
│  └─────────────────┘      │      ├─ DELETE getui-token       │    │
│                            │      └─ GET info                │    │
└────────────────────────────┼────────────────────────────────────┘
                             │ HTTPS
                             ▼
                   ┌─────────────────┐
                   │   个推 REST API  │
                   │ restapi.getui.com│
                   └─────────────────┘
```

### 1.2 核心模块

| 模块 | 职责 | 文件 |
|------|------|------|
| `GetuiPushApp` | 插件主入口，注册生命周期和配置 | `GetuiPushApp.ts` |
| `GetuiAuthService` | 个推鉴权 Token 获取与缓存 | `services/GetuiAuthService.ts` |
| `GetuiPushService` | 批量推送逻辑 | `services/GetuiPushService.ts` |
| `TokenService` | 个推 Token 的增删查 | `services/TokenService.ts` |
| `ContentBuilder` | 推送内容构建 | `services/ContentBuilder.ts` |
| `GetuiTokenEndpoint` (POST) | Token 注册端点 | `endpoints/GetuiTokenRegisterEndpoint.ts` |
| `GetuiTokenEndpoint` (DELETE) | Token 删除端点 | `endpoints/GetuiTokenDeleteEndpoint.ts` |
| `InfoEndpoint` | 插件信息/发现端点 | `endpoints/InfoEndpoint.ts` |
| `GetUiPushCommand` | `/getui-push` Slash 命令 | `commands/GetUiPushCommand.ts` |
| `PostMessageSentHandler` | 消息发送事件处理 | `handlers/PostMessageSentHandler.ts` |
| `PostUserLoggedOutHandler` | 用户登出事件处理 | `handlers/PostUserLoggedOutHandler.ts` |

### 1.3 技术选型

- **框架**: Rocket.Chat Apps-Engine（TypeScript）
- **外部 HTTP**: `IHttp`（Apps-Engine 内置，带沙箱限制）
- **存储**: `IPersistence`（Apps-Engine 内置键值存储）
- **日志**: `ILogger`（Apps-Engine 内置，日志显示在 Admin → Apps → Logs）
- **配置**: `ISettingsExtend`（`SettingType.PASSWORD` 用于 MasterSecret）

## 2. 项目结构

```
getui-push-app/                    # RC App 插件根目录
├── GetuiPushApp.ts                # 插件主入口
├── app.json                       # 插件元数据
├── endpoints/
│   ├── GetuiTokenRegisterEndpoint.ts  # POST getui-token
│   ├── GetuiTokenDeleteEndpoint.ts    # DELETE getui-token
│   └── InfoEndpoint.ts                # GET info
├── handlers/
│   ├── PostMessageSentHandler.ts      # IPostMessageSent 实现
│   └── PostUserLoggedOutHandler.ts    # IPostUserLoggedOut 实现
├── commands/
│   └── GetUiPushCommand.ts            # /getui-push slash 命令
├── services/
│   ├── GetuiAuthService.ts            # 个推鉴权 Token 管理
│   ├── GetuiPushService.ts            # 批量推送
│   ├── TokenService.ts                # Token CRUD
│   └── ContentBuilder.ts             # 推送内容构建
└── types/
    └── index.ts                       # 内部类型定义
```

## 3. 详细设计

### 3.1 插件主入口 (`GetuiPushApp.ts`)

```typescript
import { App } from '@rocket.chat/apps-engine/definition/App';
import { IAppInfo } from '@rocket.chat/apps-engine/definition/metadata';
import {
  IConfigurationExtend,
  IEnvironmentRead,
  ILogger,
  IAppAccessors,
} from '@rocket.chat/apps-engine/definition/accessors';
import { SettingType } from '@rocket.chat/apps-engine/definition/settings';
import { IPostMessageSent } from '@rocket.chat/apps-engine/definition/messages';
import { IPostUserLoggedOut } from '@rocket.chat/apps-engine/definition/users';
import { ApiVisibility, ApiSecurity } from '@rocket.chat/apps-engine/definition/api';

import { PostMessageSentHandler } from './handlers/PostMessageSentHandler';
import { PostUserLoggedOutHandler } from './handlers/PostUserLoggedOutHandler';
import { GetuiTokenRegisterEndpoint } from './endpoints/GetuiTokenRegisterEndpoint';
import { GetuiTokenDeleteEndpoint } from './endpoints/GetuiTokenDeleteEndpoint';
import { InfoEndpoint } from './endpoints/InfoEndpoint';
import { GetUiPushCommand } from './commands/GetUiPushCommand';

export class GetuiPushApp extends App
  implements IPostMessageSent, IPostUserLoggedOut {

  constructor(info: IAppInfo, logger: ILogger, accessors?: IAppAccessors) {
    super(info, logger, accessors);
  }

  // 注册设置、API 端点、Slash 命令
  protected async extendConfiguration(
    configuration: IConfigurationExtend,
    _environmentRead: IEnvironmentRead,
  ): Promise<void> {
    // === 设置 ===
    await configuration.settings.provideSetting({
      id: 'Getui_Enabled',
      type: SettingType.BOOLEAN,
      packageValue: false,
      required: false,
      public: false,
      i18nLabel: 'GeTui_Enabled',
      i18nDescription: 'GeTui_Enabled_Description',
    });
    await configuration.settings.provideSetting({
      id: 'Getui_AppId',
      type: SettingType.STRING,
      packageValue: '',
      required: false,
      public: false,
      i18nLabel: 'GeTui_AppId',
    });
    await configuration.settings.provideSetting({
      id: 'Getui_AppKey',
      type: SettingType.STRING,
      packageValue: '',
      required: false,
      public: false,
      i18nLabel: 'GeTui_AppKey',
    });
    await configuration.settings.provideSetting({
      id: 'Getui_MasterSecret',
      type: SettingType.PASSWORD,
      packageValue: '',
      required: false,
      public: false,
      i18nLabel: 'GeTui_MasterSecret',
    });
    await configuration.settings.provideSetting({
      id: 'Getui_Api_Url',
      type: SettingType.STRING,
      packageValue: 'https://restapi.getui.com/v2',
      required: false,
      public: false,
      i18nLabel: 'GeTui_Api_Url',
    });
    await configuration.settings.provideSetting({
      id: 'Getui_Max_Tokens_Per_User',
      type: SettingType.NUMBER,
      packageValue: 1,
      required: false,
      public: false,
      i18nLabel: 'GeTui_Max_Tokens_Per_User',
    });

    // === API 端点 ===
    await configuration.api.provideApi({
      visibility: ApiVisibility.PUBLIC,
      security: ApiSecurity.UNSECURE,  // 鉴权由端点的 authRequired 处理
      endpoints: [
        new GetuiTokenRegisterEndpoint(this),
        new GetuiTokenDeleteEndpoint(this),
        new InfoEndpoint(this),
      ],
    });

    // === Slash 命令 ===
    await configuration.slashCommands.provideSlashCommand(
      new GetUiPushCommand(this),
    );
  }

  // IPostMessageSent 委托给 Handler
  public async checkPostMessageSent(message, read, http): Promise<boolean> {
    return PostMessageSentHandler.check(message, read);
  }

  public async executePostMessageSent(message, read, http, persistence, modify): Promise<void> {
    return PostMessageSentHandler.execute(message, read, http, persistence, modify, this.getLogger());
  }

  // IPostUserLoggedOut 委托给 Handler
  public async executePostUserLoggedOut(user, read, http, persistence, modify): Promise<void> {
    return PostUserLoggedOutHandler.execute(user, persistence, this.getLogger());
  }
}
```

---

### 3.2 数据存储设计 (`TokenService.ts`)

#### 3.2.1 存储数据结构

```typescript
// types/index.ts
export interface GetuiTokenRecord {
  token: string;              // 个推 CID
  appName: string;            // 应用标识
  userId: string;             // 用户 ID
  platform: 'android' | 'harmony';
  createdAt: string;          // ISO 日期字符串
  updatedAt: string;
}

export interface GetuiAuthCache {
  token: string;              // 个推鉴权 Token
  expiresAt: number;          // Unix 时间戳（毫秒）
}
```

#### 3.2.2 IPersistence 存储策略

每个 Token 记录创建两个关联：

```typescript
import {
  RocketChatAssociationModel,
  RocketChatAssociationRecord,
} from '@rocket.chat/apps-engine/definition/metadata';

// 按用户查询（获取用户所有 Token）
const byUser = new RocketChatAssociationRecord(
  RocketChatAssociationModel.USER,
  userId,
);

// 按 Token 值唯一查询（检查 Token 是否已存在）
const byToken = new RocketChatAssociationRecord(
  RocketChatAssociationModel.MISC,
  `getui:token:${cid}`,
);
```

#### 3.2.3 TokenService 接口

```typescript
import { IPersistence, IPersistenceRead } from '@rocket.chat/apps-engine/definition/accessors';
import { RocketChatAssociationModel, RocketChatAssociationRecord } from '@rocket.chat/apps-engine/definition/metadata';
import { GetuiTokenRecord } from '../types';

export class TokenService {
  /**
   * 注册 Token（upsert 语义）：
   * 1. 检查 Token 是否已存在（按 MISC 关联查询）
   * 2. 如已存在且属于其他用户，先从其他用户移除
   * 3. 检查用户 Token 数量，超限则移除最旧的
   * 4. 创建/更新记录，建立两个关联
   */
  static async upsert(
    persistence: IPersistence,
    persis: IPersistenceRead,
    data: Omit<GetuiTokenRecord, 'createdAt' | 'updatedAt'>,
    maxPerUser: number,
  ): Promise<GetuiTokenRecord>;

  /** 按用户 ID 查询所有 Token */
  static async findByUserId(
    persis: IPersistenceRead,
    userId: string,
  ): Promise<GetuiTokenRecord[]>;

  /** 按 Token 值查询（返回 null 如不存在） */
  static async findByToken(
    persis: IPersistenceRead,
    token: string,
  ): Promise<GetuiTokenRecord | null>;

  /** 按用户 ID 查询多个用户的所有 Token */
  static async findByUserIds(
    persis: IPersistenceRead,
    userIds: string[],
  ): Promise<GetuiTokenRecord[]>;

  /** 删除指定用户的指定 Token */
  static async removeToken(
    persistence: IPersistence,
    persis: IPersistenceRead,
    token: string,
    userId: string,
  ): Promise<void>;

  /** 删除指定用户的所有 Token（登出时使用） */
  static async removeAllByUserId(
    persistence: IPersistence,
    persis: IPersistenceRead,
    userId: string,
  ): Promise<void>;

  /** 删除指定 Token 集合（无效 CID 清理时使用） */
  static async removeTokens(
    persistence: IPersistence,
    persis: IPersistenceRead,
    tokens: string[],
  ): Promise<void>;
}
```

---

### 3.3 个推鉴权服务 (`GetuiAuthService.ts`)

```typescript
import { IHttp, IPersistence, IPersistenceRead, ILogger } from '@rocket.chat/apps-engine/definition/accessors';
import { RocketChatAssociationModel, RocketChatAssociationRecord } from '@rocket.chat/apps-engine/definition/metadata';
import { createHash } from 'crypto';  // 在 Apps-Engine 沙箱中可用

export class GetuiAuthService {
  // 鉴权 Token 缓存存储键
  private static readonly AUTH_CACHE_KEY = 'getui:auth:cache';
  private static readonly AUTH_ASSOC = new RocketChatAssociationRecord(
    RocketChatAssociationModel.MISC,
    GetuiAuthService.AUTH_CACHE_KEY,
  );

  /**
   * 获取鉴权 Token
   * - 先从 IPersistence 读取缓存，有效期内直接返回
   * - 否则调用个推 auth API 获取新 Token，更新缓存
   *
   * sign = SHA256(appKey + timestamp + masterSecret)
   */
  static async getToken(
    http: IHttp,
    persistence: IPersistence,
    persis: IPersistenceRead,
    config: {
      apiUrl: string;
      appId: string;
      appKey: string;
      masterSecret: string;
    },
    logger: ILogger,
  ): Promise<string>;

  /** 生成个推鉴权签名 */
  private static generateSign(
    appKey: string,
    timestamp: string,
    masterSecret: string,
  ): string {
    return createHash('sha256')
      .update(`${appKey}${timestamp}${masterSecret}`)
      .digest('hex');
  }
}
```

---

### 3.4 批量推送服务 (`GetuiPushService.ts`)

```typescript
export class GetuiPushService {
  private static readonly BATCH_SIZE = 200;

  /**
   * 发送批量推送
   *
   * 流程:
   * 1. 获取鉴权 Token（GetuiAuthService.getToken）
   * 2. 创建消息体（POST /push/list/message）→ taskId
   * 3. 将 CID 分批（每批 ≤ 200）
   * 4. 对每批调用推送接口（POST /push/list/cid）
   * 5. 处理结果：收集无效 CID，触发清理
   */
  static async sendBatch(
    http: IHttp,
    persistence: IPersistence,
    persis: IPersistenceRead,
    config: GetuiConfig,
    cids: string[],
    notification: GetuiNotification,
    logger: ILogger,
  ): Promise<void>;

  /** 创建消息体，返回 taskId */
  private static async createMessage(
    http: IHttp,
    config: GetuiConfig,
    authToken: string,
    notification: GetuiNotification,
  ): Promise<string>;

  /** 推送到一批 CID */
  private static async pushToCids(
    http: IHttp,
    config: GetuiConfig,
    authToken: string,
    taskId: string,
    cids: string[],
  ): Promise<{ successCids: string[]; invalidCids: string[] }>;

  /** 带重试的执行函数（最多 3 次，指数退避） */
  private static async withRetry<T>(
    fn: () => Promise<T>,
    context: string,
    logger: ILogger,
  ): Promise<T>;
}
```

**内部类型定义**:

```typescript
// types/index.ts（续）

export interface GetuiConfig {
  apiUrl: string;
  appId: string;
  appKey: string;
  masterSecret: string;
}

export interface GetuiNotification {
  title: string;
  body: string;
  clickType: 'payload';
  transmission: {
    host: string;
    rid: string;
    msgId: string;
    roomName: string;
    senderName: string;
  };
}
```

---

### 3.5 消息事件处理 (`PostMessageSentHandler.ts`)

```typescript
import { IHttp, IModify, IPersistence, IRead } from '@rocket.chat/apps-engine/definition/accessors';
import { IMessage } from '@rocket.chat/apps-engine/definition/messages';
import { ILogger } from '@rocket.chat/apps-engine/definition/accessors';

export class PostMessageSentHandler {
  /**
   * 快速前置检查（checkPostMessageSent）：
   * - 插件 Getui_Enabled 设置是否为 true
   * - message.room.customFields?.getuiPushEnabled 是否为 true
   * - message.type 为 undefined（非系统消息）
   */
  static async check(
    message: IMessage,
    read: IRead,
  ): Promise<boolean>;

  /**
   * 主处理逻辑（executePostMessageSent）：
   * 1. 读取插件配置（AppId, AppKey, MasterSecret, ApiUrl）
   * 2. 获取房间成员列表（read.getRoomReader().getMembers(roomId)）
   * 3. 排除消息发送者
   * 4. 从 IPersistence 查询成员的 Token
   * 5. 无 Token 则跳过
   * 6. 构建推送通知内容（ContentBuilder）
   * 7. 调用 GetuiPushService.sendBatch
   */
  static async execute(
    message: IMessage,
    read: IRead,
    http: IHttp,
    persistence: IPersistence,
    modify: IModify,
    logger: ILogger,
  ): Promise<void>;
}
```

---

### 3.6 API 端点设计

#### 3.6.1 Token 注册端点 (`GetuiTokenRegisterEndpoint.ts`)

```typescript
import { IApiEndpoint } from '@rocket.chat/apps-engine/definition/api';

export class GetuiTokenRegisterEndpoint implements IApiEndpoint {
  path = 'getui-token';
  authRequired = true;

  async post(request, endpoint, read, modify, http, persis): Promise<IApiResponse> {
    const user = request.user;  // 已通过鉴权，user 不为 null
    const { token, appName, platform } = request.content;

    // 参数校验
    if (!token || typeof token !== 'string') {
      return { status: HttpStatusCode.BAD_REQUEST, content: { success: false, error: 'token is required' } };
    }
    if (!['android', 'harmony'].includes(platform)) {
      return { status: HttpStatusCode.BAD_REQUEST, content: { success: false, error: 'invalid platform' } };
    }

    // 检查个推是否启用
    const enabled = await read.getEnvironmentReader().getSettings().getValueById('Getui_Enabled');
    if (!enabled) {
      return { status: HttpStatusCode.BAD_REQUEST, content: { success: false, error: 'GeTui push is not enabled' } };
    }

    const maxPerUser = await read.getEnvironmentReader().getSettings().getValueById('Getui_Max_Tokens_Per_User');

    const persistence = ... // 从 modify 获取 persistence
    const record = await TokenService.upsert(persistence, persis, {
      token,
      appName,
      userId: user.id,
      platform,
    }, maxPerUser);

    return { status: HttpStatusCode.OK, content: { success: true, token: record.token, userId: record.userId } };
  }
}
```

#### 3.6.2 Token 删除端点 (`GetuiTokenDeleteEndpoint.ts`)

```typescript
export class GetuiTokenDeleteEndpoint implements IApiEndpoint {
  path = 'getui-token';
  authRequired = true;

  async delete(request, endpoint, read, modify, http, persis): Promise<IApiResponse> {
    const user = request.user;
    const { token } = request.content;

    if (!token) {
      return { status: HttpStatusCode.BAD_REQUEST, content: { success: false, error: 'token is required' } };
    }

    const persistence = ...;
    await TokenService.removeToken(persistence, persis, token, user.id);
    return { status: HttpStatusCode.OK, content: { success: true } };
  }
}
```

#### 3.6.3 信息发现端点 (`InfoEndpoint.ts`)

```typescript
export class InfoEndpoint implements IApiEndpoint {
  path = 'info';
  authRequired = false;

  async get(request, endpoint, read, modify, http, persis): Promise<IApiResponse> {
    const appId = ... // 从 app 获取
    return {
      status: HttpStatusCode.OK,
      content: {
        appId,
        version: this.app.getVersion(),
        tokenEndpoint: `/api/apps/public/${appId}/getui-token`,
      },
    };
  }
}
```

---

### 3.7 Slash 命令设计 (`GetUiPushCommand.ts`)

```typescript
import { ISlashCommand, SlashCommandContext } from '@rocket.chat/apps-engine/definition/slashcommands';

export class GetUiPushCommand implements ISlashCommand {
  command = 'getui-push';
  i18nParamsExample = 'enable | disable | status';
  i18nDescription = 'Manage GeTui push notification for current room';

  async executor(
    context: SlashCommandContext,
    read: IRead,
    modify: IModify,
    http: IHttp,
    persistence: IPersistence,
  ): Promise<void> {
    const [subCommand] = context.getArguments();
    const room = context.getRoom();
    const sender = context.getSender();

    // 权限检查：仅 edit-room 权限用户可操作
    // （通过检查 sender 是否为管理员/房主来实现）

    switch (subCommand) {
      case 'enable':
        await this.setRoomPushEnabled(room, sender, modify, true);
        await this.notifyRoom(room, sender, modify, '✅ 个推推送已为本频道**启用**');
        break;
      case 'disable':
        await this.setRoomPushEnabled(room, sender, modify, false);
        await this.notifyRoom(room, sender, modify, '⛔ 个推推送已为本频道**禁用**');
        break;
      case 'status':
        const enabled = room.customFields?.getuiPushEnabled === true;
        await this.notifyRoom(room, sender, modify,
          `当前频道个推推送状态：${enabled ? '✅ 已启用' : '⛔ 已禁用'}`);
        break;
      default:
        await this.notifyRoom(room, sender, modify,
          '用法: `/getui-push enable|disable|status`');
    }
  }

  /** 使用 IRoomBuilder.setCustomFields 写入房间自定义字段 */
  private async setRoomPushEnabled(
    room: IRoom,
    updater: IUser,
    modify: IModify,
    value: boolean,
  ): Promise<void> {
    const builder = await modify.getUpdater().room(room.id, updater);
    const currentFields = room.customFields || {};
    builder.setCustomFields({ ...currentFields, getuiPushEnabled: value });
    await modify.getUpdater().finish(builder);
  }

  /** 向频道发送提示消息 */
  private async notifyRoom(
    room: IRoom,
    sender: IUser,
    modify: IModify,
    text: string,
  ): Promise<void> {
    const msg = modify.getCreator().startMessage()
      .setRoom(room)
      .setSender(sender)
      .setText(text);
    await modify.getCreator().finish(msg);
  }
}
```

---

### 3.8 推送内容构建 (`ContentBuilder.ts`)

```typescript
import { IMessage } from '@rocket.chat/apps-engine/definition/messages';
import { GetuiNotification } from '../types';

export class ContentBuilder {
  private static readonly MAX_BODY_LENGTH = 200;

  /**
   * 从消息构建推送通知内容
   *
   * @param message  RC 消息对象
   * @param serverUrl 服务器公开地址（从 Site_Url 设置读取）
   */
  static build(message: IMessage, serverUrl: string): GetuiNotification {
    const roomName = message.room.displayName || message.room.slugifiedName;
    const senderName = message.sender.name || message.sender.username;

    // 构建通知正文
    let body = '';
    if (message.text) {
      body = message.text.length > ContentBuilder.MAX_BODY_LENGTH
        ? `${message.text.substring(0, ContentBuilder.MAX_BODY_LENGTH)}...`
        : message.text;
    } else if (message.files?.length) {
      const fileType = message.files[0].type || '';
      if (fileType.startsWith('image/')) body = '[图片]';
      else if (fileType.startsWith('video/')) body = '[视频]';
      else body = '[文件]';
    } else if (message.attachments?.length) {
      body = '[附件]';
    }

    return {
      title: `[${roomName}] ${senderName}`,
      body,
      clickType: 'payload',
      transmission: {
        host: serverUrl,
        rid: message.room.id,
        msgId: message.id || '',
        roomName,
        senderName,
      },
    };
  }
}
```

---

### 3.9 错误处理与重试策略

```typescript
// 与嵌入式方案相同
const RETRY_CONFIG = {
  maxRetries: 3,
  baseDelay: 1000,  // 1 秒
  maxDelay: 10000,  // 10 秒
};

async function withRetry<T>(
  fn: () => Promise<T>,
  context: string,
  logger: ILogger,
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
      logger.warn(`${context} failed (attempt ${attempt + 1}/${RETRY_CONFIG.maxRetries}), retrying in ${delay}ms`);
      await new Promise(resolve => setTimeout(resolve, delay));
    }
  }
  throw new Error(`${context} failed after ${RETRY_CONFIG.maxRetries} retries`);
}
```

**错误码处理策略**:

| 个推错误码 | 含义 | 处理 |
|-----------|------|------|
| 10001 | Token 无效/过期 | 清除 IPersistence 中的鉴权缓存，重新鉴权 |
| 10002 | CID 不存在 | 从 IPersistence 移除该 CID 对应的 Token 记录 |
| 其他 | API 错误 | 重试或记录错误 |

---

## 4. 推送时序图

### 4.1 Token 注册时序

```
个推App                   RC App (Plugin)               IPersistence
  │                           │                              │
  │─ POST /api/apps/public/   │                              │
  │   {appId}/getui-token ───→│                              │
  │   Headers: X-Auth-Token   │                              │
  │   Body: { token, ... }    │                              │
  │                           │─ 检查 Getui_Enabled ────────→│
  │                           │─ findByToken(cid) ─────────→│
  │                           │                              │
  │                           │─ removeOldestIfOverLimit ──→│
  │                           │─ upsert(record, [assocs]) ─→│
  │                           │                              │
  │←── { success: true } ─────│                              │
```

### 4.2 消息推送时序

```
RC Core              PostMessageSentHandler     GetuiPushService        个推 API
   │                        │                        │                     │
   │─ IPostMessageSent ────→│                        │                     │
   │                        │─ check(enabled?) ─┐   │                     │
   │                        │←─ false: skip ────┘   │                     │
   │                        │                        │                     │
   │                        │─ getMembers(roomId) ──→│                     │
   │                        │─ findByUserIds([...]) →│                     │
   │                        │                        │                     │
   │                        │─ sendBatch(cids, notif)→│                    │
   │                        │                        │─ getAuthToken() ───→│
   │                        │                        │←── token ───────────│
   │                        │                        │─ createMessage() ──→│
   │                        │                        │←── taskId ──────────│
   │                        │                        │─ pushToCids() ─────→│
   │                        │                        │←── result ──────────│
   │                        │                        │─ cleanup invalid CID│
```

## 5. 插件元数据 (`app.json`)

```json
{
  "id": "getui-push-app",
  "name": "GeTui Push",
  "nameSlug": "getui-push",
  "version": "1.0.0",
  "requiredApiVersion": "^1.4.0",
  "description": "个推（GeTui）推送通知集成插件，为中国大陆用户提供可靠的移动推送服务",
  "author": {
    "name": "Your Name",
    "email": "your@email.com",
    "url": "https://example.com"
  },
  "classFile": "GetuiPushApp.js",
  "iconFile": "icon.png",
  "implements": [
    "IPostMessageSent",
    "IPostUserLoggedOut"
  ],
  "permissions": [
    { "name": "networking.interfaces", "domains": ["restapi.getui.com"] },
    { "name": "persistence" },
    { "name": "message.read" },
    { "name": "room.read" },
    { "name": "user.read" }
  ]
}
```

## 6. 文件变更清单

### 6.1 插件文件（独立仓库，无 RC 核心改动）

| 文件路径 | 说明 |
|----------|------|
| `GetuiPushApp.ts` | 插件主入口 |
| `app.json` | 插件元数据 |
| `types/index.ts` | 内部类型定义 |
| `services/GetuiAuthService.ts` | 个推鉴权 Token 管理 |
| `services/GetuiPushService.ts` | 批量推送服务 |
| `services/TokenService.ts` | Token CRUD（使用 IPersistence） |
| `services/ContentBuilder.ts` | 推送内容构建 |
| `handlers/PostMessageSentHandler.ts` | 消息发送后处理 |
| `handlers/PostUserLoggedOutHandler.ts` | 用户登出后清理 Token |
| `endpoints/GetuiTokenRegisterEndpoint.ts` | POST Token 端点 |
| `endpoints/GetuiTokenDeleteEndpoint.ts` | DELETE Token 端点 |
| `endpoints/InfoEndpoint.ts` | 插件发现端点 |
| `commands/GetUiPushCommand.ts` | `/getui-push` Slash 命令 |
| `i18n/en.json` | 英文 i18n（插件 i18n） |
| `i18n/zh.json` | 中文 i18n（插件 i18n） |

### 6.2 RC 核心文件改动

**无**。插件方案不需要修改任何 Rocket.Chat 核心代码。

## 7. 已知限制

| 限制 | 说明 | 缓解措施 |
|------|------|----------|
| 不检查用户移动通知偏好 | Apps-Engine 未暴露该字段 | 用户通过登出删除 Token 来退出推送 |
| API 路径与嵌入式方案不同 | `/api/apps/public/{appId}/getui-token` | 手机 App 通过 `/info` 端点动态发现路径 |
| 房间开关无原生 Admin UI | 无法在房间设置面板添加开关 | 管理员通过 `/getui-push enable/disable` 命令操作 |
| 设置在 Apps 页面而非 Push 页面 | UX 位置不同 | 可接受，功能完整 |
