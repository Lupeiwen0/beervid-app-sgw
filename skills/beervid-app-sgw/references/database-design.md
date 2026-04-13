# 数据存储建议

## 目录

- [设计目标](#设计目标)
- [核心实体关系](#核心实体关系)
- [推荐表结构](#推荐表结构)
- [设计原则](#设计原则)

## 设计目标

本文档描述 **接入方** 在自己的系统中需要存储哪些数据。你不需要复制 tt-manager 的内部表结构——只需要存储从 API 响应中拿到的关键信息，以便后续调用其他接口。

核心思路：**按 API 流程逐步积累数据**。

```
OAuth 授权 → 拿到 authorizedAccountId → 存 authorized_accounts
上传视频   → 拿到 ossUrl / videoFileId → 存入发布参数
创建发布   → 拿到 publishTaskId        → 存 publish_tasks
轮询/Webhook → 状态变更               → 更新 publish_tasks
发布成功   → 拿到 upstreamVideoId      → 可查询视频数据
```

## 核心实体关系

```
┌──────────────────────┐
│  authorized_accounts │  ← OAuth 回调时写入
├──────────────────────┤
│ id (authorizedAccountId)
│ platform             │
│ openId               │
│ username             │
│ authStatus           │
│ ...                  │
└──────────┬───────────┘
           │
           │ 1:*
           v
┌──────────────────────┐
│   publish_tasks      │  ← 创建发布任务时写入
├──────────────────────┤
│ publishTaskId        │
│ authorizedAccountId  │
│ idempotencyKey       │
│ platform             │
│ status               │
│ upstreamVideoId      │  ← 发布成功后回填
│ ...                  │
└──────────────────────┘
```

## 推荐表结构

以下是接入方建议的表结构，可根据自身业务需求增减字段。

### authorized_accounts — 授权账号

来源：OAuth 回调的 `state` 参数 + `GET /open-api/v1/accounts` 接口。

```typescript
authorizedAccounts {
  // 来自 OAuth 回调 state
  id: uuid NOT NULL             // authorizedAccountId，后续所有接口都需要
  platform: enum('tt', 'tts') NOT NULL
  openId: varchar(128) NOT NULL
  username: varchar(128) NOT NULL

  // 来自 GET /accounts 或账号信息接口（可选）
  displayName: varchar(128)
  avatarUrl: text
  authStatus: enum('active', 'expired', 'revoked', 'refresh_failed') NOT NULL
  grantedScopes: json           // string[]

  // 你的业务字段
  // userId: uuid               // 关联你系统中的用户

  createdAt: timestamp NOT NULL
  updatedAt: timestamp NOT NULL
}
// 唯一约束: (platform, openId) — 同平台同用户只记录一次
// 索引: (authStatus) — 查询需要重新授权的账号
```

**何时写入**：
1. OAuth 回调到达时，解析 `state` 获取 `id`、`openId`、`username`、`platform`
2. 可通过 `GET /open-api/v1/accounts` 定期同步 `authStatus`

**注意**：同一用户可能同时有 TT 和 TTS 两个账号（不同的 `authorizedAccountId`），可通过 `username` 关联。

### publish_tasks — 发布任务

来源：`POST /open-api/v1/{tt|tts}/videos/publish` 响应 + 轮询/Webhook 状态更新。

```typescript
publishTasks {
  // 创建发布时写入
  publishTaskId: uuid NOT NULL    // 创建接口返回的 id
  appId: uuid NOT NULL            // 你的应用 ID，区分多应用场景
  authorizedAccountId: uuid NOT NULL
  platform: enum('tt', 'tts') NOT NULL
  idempotencyKey: varchar(255) NOT NULL  // 你生成的幂等键，必须持久化
  caption: text NOT NULL

  // TT 专属
  ossUrl: text                    // TT: 上传接口返回值; TTS: null

  // TTS 专属
  videoFileId: text               // TTS: 上传接口返回值; TT: null
  productId: varchar(128)
  productTitle: varchar(128)

  // 状态跟踪（轮询或 Webhook 更新）
  status: enum('created', 'submitted', 'published', 'failed') NOT NULL
  failureCode: varchar(128)
  failureReason: text
  upstreamVideoId: varchar(128)   // published 后有值，用于查询视频数据
  publishedAt: timestamp

  // 你的业务字段
  // userId: uuid
  // videoFilePath: text          // 原始视频文件路径等

  createdAt: timestamp NOT NULL
  updatedAt: timestamp NOT NULL
}
// 唯一约束: (appId, idempotencyKey) — 防止重复创建（服务端按 appId + key 查重）
// 索引: (authorizedAccountId, status) — 按账号查发布任务
// 索引: (status, updatedAt) — 查询需要刷新状态的任务
```

**何时写入**：
1. 调用发布接口后，将返回的 `id`（即 publishTaskId）、`status` 等存入
2. 轮询 `GET /open-api/v1/publish-tasks/:id` 或收到 Webhook 后更新 `status`、`upstreamVideoId` 等

**关于 idempotencyKey**：这个字段非常重要。生成后必须持久化，网络超时时用同一个 key 重试，避免创建重复任务。

### video_data — 视频数据缓存（可选）

来源：发布成功后通过视频数据查询接口获取。

```typescript
// TT 视频信息（来自 GET /open-api/v1/tt/videos）
ttVideos {
  id: uuid NOT NULL               // publishTaskId 或业务主键
  authorizedAccountId: uuid NOT NULL
  videoId: varchar(128) NOT NULL   // upstreamVideoId
  thumbnailUrl: text
  shareUrl: text
  raw: jsonb                      // 接口返回的完整数据
  fetchedAt: timestamp NOT NULL   // 最后一次拉取时间
}

// TTS 带货表现（来自 GET /open-api/v1/tts/videos/performances）
ttsPerformances {
  id: uuid NOT NULL
  authorizedAccountId: uuid NOT NULL
  videoId: varchar(128) NOT NULL   // upstreamVideoId
  date: date NOT NULL              // 表现数据所属日期（T-1 延迟）
  clickThroughRate: decimal
  orderCount: integer
  itemSoldCount: integer
  gmvAmount: decimal
  gmvCurrency: varchar(8)
  raw: jsonb
  fetchedAt: timestamp NOT NULL
}
// 唯一约束: (videoId, date) — 每天一条记录
```

这些表不是必须的——如果业务只需要展示最新数据，可以直接从 API 实时查询。但如果需要历史趋势分析或减少 API 调用，建议缓存。

## 设计原则

1. **只存 API 返回的数据**：不需要猜测 tt-manager 内部结构，按接口响应的字段存储即可
2. **idempotencyKey 必须持久化**：发布接口的幂等键在重试时必须复用，否则会创建重复任务
3. **authStatus 需要定期检查**：通过 `GET /accounts` 同步账号状态，`expired`/`revoked` 状态需引导用户重新授权
4. **upstreamVideoId 是后续查询的关键**：发布成功后才有值，用它来查询 TT 视频数据或 TTS 带货表现
5. **按你的业务补充字段**：上面的表结构是推荐的最小集，根据你的业务需要添加 `userId`、`tags`、`category` 等字段
