# 个推推送手机 App — 设计文档

## 1. 技术架构

### 1.1 整体架构图

```
┌──────────────────────────────────────────────────────────────────────┐
│                        个推App (uni-app)                              │
│                                                                       │
│  ┌─────────────┐  ┌──────────────┐  ┌──────────────┐  ┌───────────┐ │
│  │   视图层     │  │   业务逻辑层  │  │   数据层      │  │  原生插件  │ │
│  │             │  │              │  │              │  │           │ │
│  │ Pages:      │  │ Services:    │  │ Store:       │  │ GeTui SDK │ │
│  │ - Login     │  │ - AuthService│  │ - Vuex/Pinia │  │ (Android) │ │
│  │ - Home      │  │ - PushService│  │ - Storage    │  │           │ │
│  │ - Settings  │  │ - APIService │  │              │  │ GeTui SDK │ │
│  │ - WebView   │  │ - JumpService│  │              │  │ (HarmonyOS│ │
│  │             │  │              │  │              │  │  Next)    │ │
│  └──────┬──────┘  └──────┬───────┘  └──────┬───────┘  └─────┬─────┘ │
│         │                │                  │                │       │
└─────────┼────────────────┼──────────────────┼────────────────┼───────┘
          │                │                  │                │
          │                ▼                  │                ▼
          │    ┌────────────────────┐         │    ┌────────────────────┐
          │    │ Rocket.Chat Server │         │    │    个推服务器        │
          │    │  REST API          │         │    │  (推送通道)         │
          │    └────────────────────┘         │    └────────────────────┘
          │                                   │
          ▼                                   ▼
  ┌──────────────┐                   ┌──────────────┐
  │  UI 渲染      │                   │ 本地存储      │
  │ (Vue 3)      │                   │ (uni.storage) │
  └──────────────┘                   └──────────────┘
```

### 1.2 技术栈

| 层次 | 技术 | 说明 |
|------|------|------|
| 框架 | uni-app (Vue 3 + Composition API) | 跨平台应用框架 |
| 语言 | TypeScript | 类型安全 |
| 状态管理 | Pinia | Vue 3 官方推荐的状态管理 |
| HTTP 客户端 | uni.request（uni-app 内置） | 网络请求 |
| 本地存储 | uni.setStorageSync / uni.getStorageSync | 本地数据持久化 |
| 推送 SDK | 个推 uni-app 插件 | 推送能力 |
| UI 组件 | uni-ui | DCloud 官方 UI 组件库 |
| 构建工具 | HBuilderX / CLI | 编译和打包 |

### 1.3 项目结构

```
getui-push-app/
├── src/
│   ├── App.vue                     # 应用入口
│   ├── main.ts                     # 主入口文件
│   ├── manifest.json               # 应用配置
│   ├── pages.json                  # 页面路由配置
│   ├── uni.scss                    # 全局样式
│   │
│   ├── pages/                      # 页面
│   │   ├── login/
│   │   │   ├── server-config.vue   # 服务器配置页
│   │   │   └── webview-login.vue   # OAuth PKCE 登录页（唯一 WebView 页面）
│   │   ├── home/
│   │   │   └── index.vue           # 主页
│   │   ├── settings/
│   │   │   └── index.vue           # 设置页
│   │   └── notification/
│   │       └── detail.vue          # 原生通知详情页（替代 WebView 方案）
│   │
│   ├── services/                   # 业务服务
│   │   ├── auth.service.ts         # 认证服务
│   │   ├── api.service.ts          # API 调用服务
│   │   ├── push.service.ts         # 推送管理服务
│   │   ├── jump.service.ts         # 跳转服务
│   │   └── storage.service.ts      # 存储服务
│   │
│   ├── stores/                     # 状态管理
│   │   ├── auth.store.ts           # 认证状态
│   │   ├── push.store.ts           # 推送状态
│   │   └── server.store.ts         # 服务器配置状态
│   │
│   ├── types/                      # TypeScript 类型定义
│   │   ├── api.types.ts            # API 类型
│   │   ├── push.types.ts           # 推送类型
│   │   └── server.types.ts         # 服务器类型
│   │
│   ├── utils/                      # 工具函数
│   │   ├── constants.ts            # 常量
│   │   ├── logger.ts               # 日志工具
│   │   └── platform.ts             # 平台检测
│   │
│   └── static/                     # 静态资源
│       ├── images/
│       │   ├── logo.png            # 应用图标
│       │   └── notification.png    # 通知图标
│       └── fonts/
│
├── nativeplugins/                  # 原生插件（个推 SDK）
│   └── getui-push-plugin/
│
├── package.json
├── tsconfig.json
└── README.md
```

## 2. 详细设计

### 2.1 页面路由设计

**文件**: `src/pages.json`

```json
{
  "pages": [
    {
      "path": "pages/home/index",
      "style": {
        "navigationBarTitleText": "RC Push"
      }
    },
    {
      "path": "pages/login/server-config",
      "style": {
        "navigationBarTitleText": "连接服务器"
      }
    },
    {
      "path": "pages/login/webview-login",
      "style": {
        "navigationBarTitleText": "登录"
      }
    },
    {
      "path": "pages/settings/index",
      "style": {
        "navigationBarTitleText": "设置"
      }
    },
    {
      "path": "pages/notification/detail",
      "style": {
        "navigationBarTitleText": "通知详情"
      }
    }
  ],
  "globalStyle": {
    "navigationBarTextStyle": "black",
    "navigationBarTitleText": "RC Push",
    "navigationBarBackgroundColor": "#F5F5F5",
    "backgroundColor": "#F5F5F5"
  }
}
```

### 2.2 服务层设计

#### 2.2.1 API Service

**文件**: `src/services/api.service.ts`

```typescript
import { useServerStore } from '@/stores/server.store';
import { useAuthStore } from '@/stores/auth.store';

/**
 * API 服务 - 封装所有与 Rocket.Chat 服务端的 HTTP 通信
 */
class APIService {
  /**
   * 通用请求方法
   * - 自动附加 X-Auth-Token 和 X-User-Id 头
   * - 自动处理 401 未授权错误（清除认证信息）
   */
  private async request<T>(
    method: 'GET' | 'POST' | 'DELETE',
    path: string,
    data?: Record<string, unknown>,
  ): Promise<T>;

  /**
   * 验证服务器有效性
   * GET /api/v1/info
   * - 检查返回的 success 字段
   * - 获取服务器版本信息
   */
  async validateServer(serverUrl: string): Promise<ServerInfo>;

  /**
   * 验证认证 Token
   * GET /api/v1/me
   * - 返回当前用户信息
   */
  async validateAuth(): Promise<UserInfo>;

  /**
   * 注册个推 Token
   * POST /api/v1/getui.token
   */
  async registerPushToken(params: {
    token: string;
    appName: string;
    platform: 'android' | 'harmony';
  }): Promise<GetuiTokenResponse>;

  /**
   * 删除个推 Token
   * DELETE /api/v1/getui.token
   */
  async removePushToken(token: string): Promise<void>;
}

export const apiService = new APIService();
```

#### 2.2.2 Auth Service

**文件**: `src/services/auth.service.ts`

```typescript
import { useAuthStore } from '@/stores/auth.store';
import { apiService } from './api.service';
import { pushService } from './push.service';
import { storageService } from './storage.service';

/**
 * 认证服务 - 管理用户 OAuth PKCE 登录/登出流程
 *
 * 安全说明: 使用 PKCE (Proof Key for Code Exchange, RFC 7636)，
 * 无需在 App 中嵌入 client_secret，即使 APK 被反编译也不会泄露密钥。
 */
class AuthService {
  /**
   * 生成 PKCE 参数
   * - code_verifier: 随机 64 字节，Base64URL 编码
   * - code_challenge: BASE64URL(SHA256(code_verifier))
   */
  private generatePKCE(): { codeVerifier: string; codeChallenge: string };

  /**
   * 启动 OAuth PKCE 登录流程
   *
   * 1. 生成 code_verifier 和 code_challenge
   * 2. 打开 WebView 加载服务器 OAuth 授权页:
   *    /oauth/authorize?response_type=code&client_id={publicClientId}
   *      &redirect_uri=rcpush://oauth&code_challenge={challenge}
   *      &code_challenge_method=S256
   * 3. WebView 拦截 rcpush://oauth?code={authCode} 重定向
   * 4. 关闭 WebView，返回授权码
   */
  async startOAuthLogin(serverUrl: string): Promise<string>; // returns authCode

  /**
   * 使用授权码换取访问令牌（无需 client_secret）
   *
   * POST /oauth/token
   * { grant_type: 'authorization_code', code, redirect_uri,
   *   client_id, code_verifier }
   * → 返回 access_token（RC authToken）和 userId
   */
  async exchangeCodeForToken(params: {
    serverUrl: string;
    authCode: string;
    codeVerifier: string;
  }): Promise<AuthCredentials>;

  /**
   * 处理登录完成回调
   * - 保存 authToken 和 userId 到安全存储
   * - 更新 Auth Store
   * - 触发 Token 注册
   */
  async handleLoginComplete(credentials: AuthCredentials): Promise<void>;

  /**
   * 自动登录
   * - 从安全存储读取保存的认证信息
   * - 调用 /api/v1/me 验证是否仍然有效
   * - 有效则直接进入主页
   * - 无效则跳转到登录页
   */
  async autoLogin(): Promise<boolean>;

  /**
   * 登出
   * 1. 调用 DELETE /api/v1/getui.token 删除服务端 Token
   * 2. 清除本地认证信息
   * 3. 重置所有 Store
   * 4. 跳转到登录页
   */
  async logout(): Promise<void>;
}

export const authService = new AuthService();
```

#### 2.2.3 Push Service

**文件**: `src/services/push.service.ts`

```typescript
import { apiService } from './api.service';
import { storageService } from './storage.service';

/**
 * 推送管理服务 - 管理个推 SDK 和 Token 注册
 */
class PushService {
  private cid: string | null = null;

  /**
   * 初始化个推 SDK
   *
   * 1. 调用 uni.getPushClientId 获取 CID
   * 2. 注册 CID 变化监听
   * 3. 注册推送消息接收监听
   * 4. 注册通知点击监听
   *
   * uni-app 推送 API:
   * - uni.getPushClientId(): 获取客户端推送标识
   * - uni.onPushMessage(): 监听推送消息
   */
  async initialize(): Promise<void>;

  /**
   * 获取当前 CID
   */
  getCid(): string | null;

  /**
   * 注册 Token 到服务端
   *
   * 1. 检查是否已登录
   * 2. 检查 CID 是否已获取
   * 3. 检查是否已注册相同 CID（避免重复注册）
   * 4. 调用 POST /api/v1/getui.token
   * 5. 保存注册状态到本地
   *
   * 重试策略:
   * - 最多重试 3 次
   * - 指数退避: 1s, 2s, 4s
   */
  async registerToken(): Promise<void>;

  /**
   * 注销 Token
   * - 调用 DELETE /api/v1/getui.token
   * - 清除本地注册状态
   */
  async unregisterToken(): Promise<void>;

  /**
   * 处理接收到的推送消息
   *
   * 消息类型:
   * 1. 透传消息 (receive): 应用在前台时收到
   *    → 显示应用内通知栏
   * 2. 通知消息 (click): 用户点击系统通知
   *    → 触发跳转逻辑
   */
  handlePushMessage(message: PushMessage): void;

  /**
   * 处理通知点击事件
   *
   * 从通知负载中提取:
   * - host: 服务器地址
   * - rid: 房间 ID
   * - msgId: 消息 ID
   * - roomName: 房间名称
   *
   * 然后调用 JumpService 进行跳转
   */
  handleNotificationClick(payload: NotificationPayload): void;
}

export const pushService = new PushService();
```

#### 2.2.4 Jump Service

**文件**: `src/services/jump.service.ts`

```typescript
/**
 * 跳转服务 - 管理从通知到目标应用/页面的跳转
 *
 * 策略: 优先 Deep Link 跳转官方 App；失败时跳转到原生通知详情页。
 * 禁止使用 WebView 打开消息页面。
 */
class JumpService {
  /**
   * Rocket.Chat 官方 App 的 URL Scheme
   */
  private readonly RC_SCHEME = 'rocketchat://';

  /**
   * 执行跳转
   *
   * 策略:
   * 1. 尝试 Deep Link 跳转到官方 App
   * 2. 如果失败，跳转到应用内原生通知详情页（不使用 WebView）
   *
   * @param params 跳转参数
   */
  async navigateToMessage(params: {
    host: string;
    rid: string;
    msgId: string;
    roomName?: string;
    senderName?: string;
    title?: string;
    body?: string;
  }): Promise<void>;

  /**
   * 检查官方 App 是否已安装
   *
   * Android: 使用 plus.runtime.isApplicationExist() 检查包名
   *   包名: chat.rocket.android (官方App的包名)
   *
   * 鸿蒙 Next: 使用对应的 API 检查应用是否安装
   */
  async isOfficialAppInstalled(): Promise<boolean>;

  /**
   * 通过 Deep Link 跳转到官方 App
   *
   * Deep Link 格式:
   *   rocketchat://room/{rid}?host={serverUrl}&messageId={msgId}
   *
   * 实现方式:
   * - Android: plus.runtime.openURL() 或 plus.runtime.launchApplication()
   * - 鸿蒙 Next: 使用对应的系统 API
   *
   * 超时检测:
   * - 设置 2 秒超时
   * - 如果 2 秒后应用仍在前台，说明跳转失败
   * - 回退到原生通知详情页
   */
  async openWithDeepLink(params: {
    host: string;
    rid: string;
    msgId: string;
  }): Promise<boolean>;

  /**
   * 跳转到原生通知详情页（不使用 WebView）
   *
   * 展示通知摘要：标题、正文、发送者、频道名；
   * 提供"打开官方 App"或"前往市场下载"按钮。
   */
  openNotificationDetail(params: {
    host: string;
    rid: string;
    msgId: string;
    roomName?: string;
    senderName?: string;
    title?: string;
    body?: string;
  }): void;
}

export const jumpService = new JumpService();
```

#### 2.2.5 Storage Service

**文件**: `src/services/storage.service.ts`

```typescript
/**
 * 安全存储服务 - 管理本地数据的安全存储
 */
class StorageService {
  /** 存储键定义 */
  private readonly KEYS = {
    SERVER_URL: 'rc_server_url',
    AUTH_TOKEN: 'rc_auth_token',
    USER_ID: 'rc_user_id',
    USER_NAME: 'rc_user_name',
    PUSH_CID: 'rc_push_cid',
    PUSH_REGISTERED: 'rc_push_registered',
    RECENT_NOTIFICATIONS: 'rc_recent_notifications',
  };

  /**
   * 保存服务器地址
   */
  saveServerUrl(url: string): void;

  /**
   * 获取服务器地址
   */
  getServerUrl(): string | null;

  /**
   * 保存认证信息
   * 优先使用平台安全存储（Android KeyStore / 鸿蒙 HUKS）
   * 降级方案：uni.setStorageSync（仅限开发/测试环境）
   */
  saveAuth(auth: { token: string; userId: string; userName: string }): void;

  /**
   * 获取认证信息
   */
  getAuth(): { token: string; userId: string; userName: string } | null;

  /**
   * 清除认证信息
   */
  clearAuth(): void;

  /**
   * 保存推送注册状态
   */
  savePushRegistered(cid: string): void;

  /**
   * 获取已注册的 CID
   */
  getRegisteredCid(): string | null;

  /**
   * 保存最近通知列表（最多 50 条）
   */
  saveRecentNotification(notification: NotificationRecord): void;

  /**
   * 获取最近通知列表
   */
  getRecentNotifications(): NotificationRecord[];

  /**
   * 清除所有数据
   */
  clearAll(): void;
}

export const storageService = new StorageService();
```

### 2.3 状态管理设计

#### 2.3.1 Auth Store

**文件**: `src/stores/auth.store.ts`

```typescript
import { defineStore } from 'pinia';

interface AuthState {
  /** 是否已登录 */
  isLoggedIn: boolean;
  /** 认证 Token */
  authToken: string | null;
  /** 用户 ID */
  userId: string | null;
  /** 用户名 */
  userName: string | null;
  /** 登录中 */
  isLoggingIn: boolean;
}

export const useAuthStore = defineStore('auth', {
  state: (): AuthState => ({
    isLoggedIn: false,
    authToken: null,
    userId: null,
    userName: null,
    isLoggingIn: false,
  }),

  actions: {
    setAuth(auth: { token: string; userId: string; userName: string }): void;
    clearAuth(): void;
    setLoggingIn(value: boolean): void;
  },
});
```

#### 2.3.2 Push Store

**文件**: `src/stores/push.store.ts`

```typescript
import { defineStore } from 'pinia';

interface PushState {
  /** 个推 CID */
  cid: string | null;
  /** SDK 初始化状态 */
  sdkInitialized: boolean;
  /** Token 是否已注册到服务端 */
  tokenRegistered: boolean;
  /** 最近推送通知列表 */
  recentNotifications: NotificationRecord[];
}

export const usePushStore = defineStore('push', {
  state: (): PushState => ({
    cid: null,
    sdkInitialized: false,
    tokenRegistered: false,
    recentNotifications: [],
  }),

  actions: {
    setCid(cid: string): void;
    setSdkInitialized(value: boolean): void;
    setTokenRegistered(value: boolean): void;
    addNotification(notification: NotificationRecord): void;
  },
});
```

### 2.4 类型定义

**文件**: `src/types/push.types.ts`

```typescript
/** 推送消息类型 */
export interface PushMessage {
  /** 消息类型: receive=透传 click=通知点击 */
  type: 'receive' | 'click';
  /** 消息数据 */
  data: {
    title?: string;
    content?: string;
    payload?: string;     // JSON 字符串
  };
}

/** 通知负载 */
export interface NotificationPayload {
  host: string;           // 服务器地址
  rid: string;            // 房间 ID
  msgId: string;          // 消息 ID
  roomName?: string;      // 房间名称
  senderName?: string;    // 发送者名称
}

/** 通知记录（本地存储） */
export interface NotificationRecord {
  id: string;             // 通知 ID
  title: string;          // 通知标题
  body: string;           // 通知正文
  payload: NotificationPayload;
  receivedAt: number;     // 接收时间戳
  read: boolean;          // 是否已读
}
```

**文件**: `src/types/api.types.ts`

```typescript
/** 服务器信息 */
export interface ServerInfo {
  version: string;
  success: boolean;
}

/** 用户信息 */
export interface UserInfo {
  _id: string;
  username: string;
  name: string;
  emails: Array<{ address: string; verified: boolean }>;
}

/** 认证凭证 */
export interface AuthCredentials {
  authToken: string;
  userId: string;
}

/** 个推 Token 注册响应 */
export interface GetuiTokenResponse {
  success: boolean;
  result: {
    _id: string;
    token: string;
    appName: string;
    userId: string;
    platform: string;
    enabled: boolean;
    createdAt: string;
    _updatedAt: string;
  };
}
```

### 2.5 核心页面设计

#### 2.5.1 App.vue — 应用入口

```typescript
// 生命周期处理

export default {
  onLaunch() {
    // 1. 初始化推送 SDK
    pushService.initialize();

    // 2. 尝试自动登录
    authService.autoLogin().then((success) => {
      if (success) {
        // 已登录，跳转主页
        uni.switchTab({ url: '/pages/home/index' });
        // 注册推送 Token
        pushService.registerToken();
      } else {
        // 未登录，跳转到服务器配置页
        uni.redirectTo({ url: '/pages/login/server-config' });
      }
    });
  },

  onShow() {
    // 应用从后台回到前台时检查登录状态
  },
};
```

#### 2.5.2 OAuth 登录页设计

**文件**: `src/pages/login/webview-login.vue`

```
核心逻辑（OAuth PKCE 流程）:
1. 从 authService 接收 code_challenge 和 redirect_uri
2. 构建授权 URL:
   {serverUrl}/oauth/authorize
     ?response_type=code
     &client_id={publicClientId}
     &redirect_uri=rcpush://oauth
     &code_challenge={code_challenge}
     &code_challenge_method=S256
3. 加载该 URL 到 WebView
4. 监听 WebView URL 变化事件（@load / onUrlChange）
5. 检测重定向到 rcpush://oauth?code=... 时：
   - 提取 code 参数
   - 关闭 WebView
   - 通知 authService 完成授权码交换
6. WebView 不注入任何脚本来读取 localStorage 或 Cookie

WebView 安全配置:
- 仅允许加载服务器域名（白名单）
- 禁止访问本地文件
- 禁止 JavaScript 读取设备存储
- 仅用于此 OAuth 登录流程，其他页面均使用原生界面
```

> **注意**: 此 WebView 页面是应用中**唯一**的 WebView 页面，仅用于 OAuth 登录授权。

WebView 登录成功回调处理:

```typescript
// 监听 WebView URL 变化
onUrlChange(event: { url: string }) {
  if (event.url.startsWith('rcpush://oauth')) {
    const url = new URL(event.url);
    const code = url.searchParams.get('code');
    if (code) {
      // 关闭 WebView，使用 code + code_verifier 换取 token
      authService.exchangeCodeForToken({
        serverUrl: this.serverUrl,
        authCode: code,
        codeVerifier: this.codeVerifier,
      }).then(() => {
        uni.redirectTo({ url: '/pages/home/index' });
      });
    }
  }
}
```

#### 2.5.3 主页面设计

**文件**: `src/pages/home/index.vue`

```
页面结构:
┌─────────────────────────┐
│ Header: "RC Push" + 设置图标
├─────────────────────────┤
│ 连接状态卡片:
│   - 服务器地址
│   - 用户名
│   - 推送状态（已连接/未连接）
│   - CID (缩略显示)
├─────────────────────────┤
│ 最近通知列表:
│   - 时间排序
│   - 点击可跳转
│   - 下拉刷新
├─────────────────────────┤
│ 底部操作:
│   - [打开 Rocket.Chat] 按钮
└─────────────────────────┘

数据流:
- 从 PushStore 读取通知列表
- 从 AuthStore 读取用户信息
- 从 ServerStore 读取服务器信息
- 从 PushStore 读取连接状态
```

### 2.6 推送处理流程

#### 2.6.1 前台推送处理

```
个推 SDK 收到消息
  → uni.onPushMessage 触发
  → pushService.handlePushMessage(message)
  → 判断 type === 'receive'（透传消息）
  → 解析 payload
  → 创建应用内通知（uni.showNotificationBar 或自定义组件）
  → 保存到 recentNotifications
```

#### 2.6.2 后台推送处理

```
系统收到个推通知
  → 系统通知栏显示
  → 用户点击通知
  → App 被唤醒
  → uni.onPushMessage 触发 (type: 'click')
  → pushService.handleNotificationClick(payload)
  → jumpService.navigateToMessage(params)
```

#### 2.6.3 跳转流程详细设计

```typescript
// jumpService.navigateToMessage 实现逻辑

async navigateToMessage(params) {
  const { host, rid, msgId, roomName, senderName, title, body } = params;

  // 1. 检查官方 App 是否安装
  const isInstalled = await this.isOfficialAppInstalled();

  if (isInstalled) {
    // 2. 尝试 Deep Link
    const success = await this.openWithDeepLink({ host, rid, msgId });
    if (success) return; // 跳转成功
  }

  // 3. 回退到原生通知详情页（不使用 WebView）
  this.openNotificationDetail({ host, rid, msgId, roomName, senderName, title, body });
}
```

### 2.7 平台适配设计

#### 2.7.1 Android 适配

```
1. 个推 SDK 集成:
   - 使用 uni-push 2.0 插件（DCloud 官方集成个推）
   - 配置厂商通道（小米、华为、OPPO、vivo）

2. 后台保活:
   - 利用个推 SDK 的保活机制
   - 申请 FOREGROUND_SERVICE 权限（如需要）

3. Deep Link:
   - 使用 plus.runtime.openURL(deepLinkUrl)
   - 检测包名: 'chat.rocket.android'

4. 通知管理:
   - Android 8.0+ 使用通知渠道（Notification Channel）
   - 渠道 ID: 'rc_push_channel'
   - 渠道名: 'Rocket.Chat 消息'
```

#### 2.7.2 鸿蒙 Next 适配

```
1. 个推 SDK:
   - 使用个推鸿蒙 Next SDK（如 uni-app 插件支持）
   - 如不支持，使用原生插件桥接

2. 推送通道:
   - 使用鸿蒙推送服务（Push Kit）
   - 个推 SDK 自动适配鸿蒙推送通道

3. Deep Link:
   - 使用鸿蒙的 Want 机制进行应用间跳转
   - 需确认官方 App 是否有鸿蒙版本

4. 安全存储:
   - 使用鸿蒙安全存储 API
   - @ohos.security.huks（密钥管理）
```

### 2.8 网络请求封装

```typescript
// src/services/api.service.ts 详细实现

private async request<T>(
  method: 'GET' | 'POST' | 'DELETE',
  path: string,
  data?: Record<string, unknown>,
): Promise<T> {
  const serverUrl = storageService.getServerUrl();
  const auth = storageService.getAuth();

  const headers: Record<string, string> = {
    'Content-Type': 'application/json',
  };

  // 附加认证头
  if (auth) {
    headers['X-Auth-Token'] = auth.token;
    headers['X-User-Id'] = auth.userId;
  }

  return new Promise((resolve, reject) => {
    uni.request({
      url: `${serverUrl}${path}`,
      method,
      header: headers,
      data,
      timeout: 10000,  // 10 秒超时

      success: (res) => {
        if (res.statusCode === 200) {
          resolve(res.data as T);
        } else if (res.statusCode === 401) {
          // 认证失效，清除登录态
          authService.logout();
          reject(new Error('Unauthorized'));
        } else {
          reject(new Error(`HTTP ${res.statusCode}: ${JSON.stringify(res.data)}`));
        }
      },

      fail: (err) => {
        reject(new Error(`Network error: ${err.errMsg}`));
      },
    });
  });
}
```

## 3. 安全设计

### 3.1 认证信息安全

```
鉴权方案: OAuth 2.0 PKCE (RFC 7636)
- 不在 App 中嵌入任何 client_secret
- code_verifier 在运行时动态生成，不持久化存储
- 即使 APK 被反编译，也无法从中提取用于伪造请求的密钥

存储策略:
- authToken: 优先使用平台安全存储（Android KeyStore / 鸿蒙 HUKS），不可用时降级为 uni.setStorageSync
- 传输中: 始终使用 HTTPS（生产环境）
- 内存中: 使用 Pinia Store 管理，应用退出后清理

安全存储实现:
- Android: 通过 uni-app 原生插件调用 Android KeyStore API 加密 authToken
- 鸿蒙 Next: 使用 HUKS（HarmonyOS Universal KeyStore）存储敏感凭证
- 降级方案: 若原生插件不可用，使用 uni.setStorageSync（仅限开发/测试环境）
```

### 3.2 WebView 安全

```
WebView 仅用于 OAuth 登录授权流程，其他页面均使用原生界面。

限制措施:
- WebView 仅允许加载已配置的服务器域名（白名单）
- 禁止 WebView 访问本地文件系统
- 不向 WebView 注入读取 localStorage/Cookie 的脚本
- 通过 CSP（Content Security Policy）限制资源加载
- 登录完成后立即关闭 WebView
```

### 3.3 推送内容安全

```
- 推送内容由服务端控制，客户端不做敏感数据处理
- 通知展示遵循服务端隐私设置
- 本地通知记录定期清理（最多保存 50 条）
- 消息详情不通过 WebView 展示，避免 WebView 注入攻击
```

## 4. 构建与部署

### 4.1 构建配置

**文件**: `src/manifest.json`（关键配置）

```json
{
  "name": "RC Push",
  "appid": "__UNI__XXXXXX",
  "description": "Rocket.Chat 推送通知伴侣应用",
  "versionName": "1.0.0",
  "versionCode": "100",

  "app-plus": {
    "distribute": {
      "android": {
        "permissions": [
          "<uses-permission android:name=\"android.permission.INTERNET\"/>",
          "<uses-permission android:name=\"android.permission.ACCESS_NETWORK_STATE\"/>",
          "<uses-permission android:name=\"android.permission.VIBRATE\"/>",
          "<uses-permission android:name=\"android.permission.RECEIVE_BOOT_COMPLETED\"/>",
          "<uses-permission android:name=\"android.permission.POST_NOTIFICATIONS\"/>"
        ],
        "minSdkVersion": 26
      }
    },
    "modules": {
      "Push": {
        "unipush": {
          "version": "2",
          "offline": true
        }
      }
    }
  },

  "push": {
    "unipush": {
      "enable": true,
      "version": "2"
    }
  }
}
```

### 4.2 环境配置

```
开发环境:
  - HBuilderX 或 CLI 开发
  - 使用 uni-app 模拟器或真机调试
  - 服务器可使用 HTTP

测试环境:
  - 使用测试个推 AppID
  - 测试服务器配置

生产环境:
  - 使用正式个推 AppID
  - 必须使用 HTTPS
  - 开启代码混淆
```

### 4.3 打包流程

```
Android:
  1. HBuilderX → 发行 → 原生App-云打包 → Android
  2. 配置签名证书
  3. 选择 CPU 架构 (arm64-v8a, armeabi-v7a)
  4. 生成 APK / AAB

鸿蒙 Next:
  1. HBuilderX → 发行 → 原生App-云打包 → HarmonyOS
  2. 配置鸿蒙签名
  3. 生成 HAP 包
  4. 上传到华为 AppGallery Connect
```

## 5. 测试策略

### 5.1 单元测试

| 模块 | 测试内容 |
|------|----------|
| api.service | HTTP 请求封装、错误处理、认证头附加 |
| push.service | Token 注册逻辑、重试机制 |
| jump.service | Deep Link 构建、跳转逻辑 |
| storage.service | 数据存取、清理逻辑 |

### 5.2 集成测试

| 场景 | 验证内容 |
|------|----------|
| 登录流程 | 服务器验证 → WebView 登录 → Token 注册 |
| 推送接收 | 发送消息 → 收到推送 → 通知展示 |
| 跳转流程 | 点击通知 → Deep Link → 官方App |
| 登出流程 | 登出 → Token 清理 → 停止推送 |

### 5.3 平台测试

| 平台 | 测试设备 |
|------|----------|
| Android | 小米/华为/OPPO/vivo 各一台，覆盖主流厂商 |
| 鸿蒙 Next | 华为 Mate 60 或 P60 系列 |

## 6. 时序图

### 6.1 完整登录并接收推送流程

```
用户          个推App         Rocket.Chat Server      个推 Server
 │              │                    │                    │
 │ 打开应用      │                    │                    │
 │─────────────→│                    │                    │
 │              │                    │                    │
 │              │ 初始化个推 SDK      │                    │
 │              │───────────────────────────────────────→ │
 │              │ ← CID              │                    │
 │              │←───────────────────────────────────────  │
 │              │                    │                    │
 │ 输入服务器    │                    │                    │
 │─────────────→│                    │                    │
 │              │ GET /api/v1/info   │                    │
 │              │──────────────────→ │                    │
 │              │ ← 服务器信息       │                    │
 │              │←──────────────────  │                    │
 │              │                    │                    │
 │              │ 打开 WebView       │                    │
 │              │──────────────────→ │                    │
 │ 在 WebView   │                    │                    │
 │ 中登录       │                    │                    │
 │─────────────→│                    │                    │
 │              │ 获取 authToken     │                    │
 │              │                    │                    │
 │              │ GET /api/v1/me     │                    │
 │              │──────────────────→ │                    │
 │              │ ← 用户信息        │                    │
 │              │←──────────────────  │                    │
 │              │                    │                    │
 │              │ POST /api/v1/      │                    │
 │              │  getui.token       │                    │
 │              │ {token:CID,...}    │                    │
 │              │──────────────────→ │                    │
 │              │ ← 注册成功        │                    │
 │              │←──────────────────  │                    │
 │              │                    │                    │
 │ ← 登录成功   │                    │                    │
 │←─────────────│                    │                    │
 │              │                    │                    │
 │              │    ... 某用户发送消息到推送频道 ...        │
 │              │                    │                    │
 │              │                    │ 批量推送            │
 │              │                    │ POST /push/list/cid│
 │              │                    │──────────────────→ │
 │              │                    │                    │
 │              │                    │                    │ 推送通知
 │              │←──────────────────────────────────────── │
 │              │ 收到推送           │                    │
 │              │ 显示系统通知       │                    │
 │              │                    │                    │
 │ 点击通知     │                    │                    │
 │─────────────→│                    │                    │
 │              │ 跳转到官方 App     │                    │
 │              │ 或 WebView 显示    │                    │
 │ ← 查看消息   │                    │                    │
 │←─────────────│                    │                    │
```
