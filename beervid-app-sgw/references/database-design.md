# 数据库设计

## 目录

- [设计原则](#设计原则)
- [ER 关系图](#er-关系图)
- [核心表结构](#核心表结构)
- [索引策略](#索引策略)
- [扩展建议](#扩展建议)

## 设计原则

系统遵循以下数据库设计原则：

1. **无外键约束**：所有关系一致性由应用层维护，不使用数据库外键、触发器、存储过程、函数
2. **唯一索引保证幂等**：通过 `uniqueIndex` 防止重复数据（如 `idempotency_key`、`event_key`）
3. **组合索引优化查询**：多列查询优先组合索引，关注列顺序（区分度高的列在前）
4. **枚举类型约束**：使用 `pgEnum` 限制状态、平台等字段取值
5. **审计字段标准化**：所有表包含 `createdAt` + `updatedAt`（自动 `$onUpdate`）

## ER 关系图

```
┌─────────────────┐       ┌──────────────────────┐
│     apps        │       │    app_webhooks       │
├─────────────────┤       ├──────────────────────┤
│ id (PK)         │──1:*──│ appId                │
│ name            │       │ webhookUrl           │
│ apiKeyHash (UQ) │       │ webhookSecret        │
│ status          │       │ enabled              │
│ description     │       │ subscribedEvents     │
│ createdAt       │       │ timeoutMs            │
│ updatedAt       │       └──────────────────────┘
└────────┬────────┘
         │
         │ 1:*
         v
┌──────────────────────┐       ┌──────────────────────┐
│ app_account_bindings │       │ authorized_accounts  │
├──────────────────────┤       ├──────────────────────┤
│ id (PK)              │──:*:1─│ id (PK)              │
│ appId                │       │ platform (enum)      │
│ authorizedAccountId  │       │ openId               │
│ bindingStatus        │       │ username             │
│ boundAt              │       │ displayName          │
└──────────────────────┘       │ avatarUrl           │
                               │ sellerName (TTS)     │
                               │ userType (TTS)       │
                               │ sellerType (TTS)     │
                               │ registerRegion (TTS) │
                               │ selectionRegion (TTS)│
                               │ authStatus (enum)    │
                               │ grantedScopes (jsonb)│
                               └──────────┬───────────┘
                                          │
                                          │ 1:1
                                          v
                               ┌──────────────────────┐
                               │   account_tokens     │
                               ├──────────────────────┤
                               │ id (PK)              │
                               │ authorizedAccountId  │
                               │ platform             │
                               │ accessTokenEnc       │
                               │ refreshTokenEnc      │
                               │ accessTokenExpiresAt │
                               │ refreshTokenExpiresAt│
                               │ tokenVersion         │
                               │ lastRefreshError     │
                               └──────────────────────┘

┌──────────────────────┐       ┌──────────────────────────┐
│   publish_tasks      │       │  publish_status_events    │
├──────────────────────┤       ├──────────────────────────┤
│ id (PK)              │──1:*──│ id (PK)                  │
│ appId                │       │ publishTaskId            │
│ authorizedAccountId  │       │ platform                 │
│ ossUrl               │       │ eventType                │
│ platform             │       │ eventKey (UQ)            │
│ caption              │       │ status                   │
│ publishMode          │       │ rawPayload (jsonb)       │
│ productId            │       │ occurredAt               │
│ videoFileId          │       └──────────────────────────┘
│ productTitle         │
│ idempotencyKey (UQ)  │       ┌──────────────────────────┐
│ upstreamPublishId(UQ)│       │  webhook_deliveries      │
│ upstreamVideoId      │       ├──────────────────────────┤
│ status (enum)        │──1:*──│ id (PK)                  │
│ failureCode          │       │ appId                    │
│ failureReason        │       │ publishTaskId            │
│ rawRequest (jsonb)   │       │ eventId                  │
│ rawResponse (jsonb)  │       │ targetUrl                │
│ publishedAt          │       │ requestBody (jsonb)      │
└──────────────────────┘       │ responseStatus          │
                               │ responseBody            │
┌──────────────────────┐       │ attemptCount            │
│    media_assets      │       │ deliveryStatus          │
├──────────────────────┤       │ nextRetryAt             │
│ id (PK)              │       │ lastAttemptAt           │
│ appId                │       └──────────────────────────┘
│ authorizedAccountId  │
│ sourceType           │
│ ossBucket            │
│ ossKey               │
│ fileSize             │
│ platform             │
└──────────────────────┘
```

## 核心表结构

### apps — 应用

```typescript
// 枚举
appStatusEnum: 'active' | 'disabled'

apps {
  id: uuid PK
  name: varchar(128) NOT NULL
  apiKeyHash: varchar(128) NOT NULL UNIQUE  // SHA256 hash，原文仅创建时返回
  status: appStatusEnum NOT NULL
  description: text
  createdAt: timestamp NOT NULL
  updatedAt: timestamp NOT NULL
}
```

### app_webhooks — Webhook 配置

```typescript
appWebhooks {
  id: uuid PK
  appId: uuid NOT NULL
  webhookUrl: text NOT NULL
  webhookSecret: varchar(255) NOT NULL      // 用于签名下游投递
  enabled: boolean NOT NULL
  subscribedEvents: text[] NOT NULL         // 订阅的事件类型列表
  timeoutMs: integer NOT NULL               // 下游超时时间
  createdAt: timestamp NOT NULL
  updatedAt: timestamp NOT NULL
}
// 索引: (appId, enabled, webhookUrl)
```

### authorized_accounts — 授权账号

```typescript
// 枚举
platformEnum: 'tt' | 'tts'
authStatusEnum: 'active' | 'expired' | 'revoked' | 'refresh_failed'

authorizedAccounts {
  id: uuid PK
  platform: platformEnum NOT NULL
  externalAccountId: varchar(128)           // TTS 的 creatorUserOpenId
  openId: varchar(128) NOT NULL             // 平台 OpenID
  username: varchar(128) NOT NULL
  displayName: varchar(128)
  avatarUrl: text
  // TTS 专属字段
  sellerName: varchar(128)
  userType: varchar(64)
  sellerType: varchar(64)
  registerRegion: varchar(32)
  selectionRegion: varchar(32)
  // 状态
  authStatus: authStatusEnum NOT NULL
  grantedScopes: jsonb NOT NULL             // string[]
  lastAuthorizedAt: timestamp NOT NULL
  lastRefreshedAt: timestamp NOT NULL
  createdAt: timestamp NOT NULL
  updatedAt: timestamp NOT NULL
}
// 唯一索引: (platform, openId) — 同平台同用户只保留一条记录
```

### account_tokens — 账号 Token

```typescript
accountTokens {
  id: uuid PK
  authorizedAccountId: uuid NOT NULL
  platform: platformEnum NOT NULL
  accessTokenEnc: text NOT NULL             // AES 加密
  refreshTokenEnc: text NOT NULL            // AES 加密
  accessTokenExpiresAt: timestamp NOT NULL
  refreshTokenExpiresAt: timestamp NOT NULL
  tokenVersion: integer NOT NULL            // 乐观锁，刷新时自增
  lastRefreshError: text
  createdAt: timestamp NOT NULL
  updatedAt: timestamp NOT NULL
}
// 唯一索引: (authorizedAccountId) — 一个账号只有一条 token 记录
```

### app_account_bindings — 应用-账号绑定

```typescript
appAccountBindings {
  id: uuid PK
  appId: uuid NOT NULL
  authorizedAccountId: uuid NOT NULL
  bindingStatus: varchar(32) NOT NULL
  boundAt: timestamp NOT NULL
  createdAt: timestamp NOT NULL
  updatedAt: timestamp NOT NULL
}
// 唯一索引: (appId, authorizedAccountId) — 一个应用与一个账号只绑定一次
// 索引: (appId, bindingStatus, authorizedAccountId) — 按应用查询活跃绑定
```

### publish_tasks — 发布任务

```typescript
// 枚举
publishModeEnum: 'tt_direct' | 'tts_direct' | 'tts_product'
publishTaskStatusEnum: 'created' | 'submitted' | 'published' | 'failed'

publishTasks {
  id: uuid PK
  appId: uuid NOT NULL
  authorizedAccountId: uuid NOT NULL
  ossUrl: text                              // TT: 有值; TTS: null
  platform: platformEnum NOT NULL
  caption: text NOT NULL
  publishMode: publishModeEnum NOT NULL
  // TTS 专属
  productId: varchar(128)
  videoFileId: text
  productTitle: varchar(128)
  // 幂等与上游
  idempotencyKey: varchar(255) NOT NULL
  upstreamPublishId: varchar(128)
  upstreamVideoId: varchar(128)
  // 状态
  status: publishTaskStatusEnum NOT NULL
  failureCode: varchar(128)
  failureReason: text
  // 原始数据
  rawRequest: jsonb
  rawResponse: jsonb
  publishedAt: timestamp
  createdAt: timestamp NOT NULL
  updatedAt: timestamp NOT NULL
}
// 唯一索引: (appId, idempotencyKey) — 应用级幂等
// 唯一索引: (upstreamPublishId) — 上游发布 ID 去重
```

### publish_status_events — 状态事件日志

```typescript
publishStatusEvents {
  id: uuid PK
  publishTaskId: uuid NOT NULL
  platform: platformEnum NOT NULL
  eventType: varchar(64) NOT NULL           // task.created, task.published, task.failed, etc.
  eventKey: varchar(255) NOT NULL           // 去重键
  status: publishTaskStatusEnum NOT NULL
  rawPayload: jsonb
  occurredAt: timestamp NOT NULL
  createdAt: timestamp NOT NULL
}
// 唯一索引: (eventKey) — 事件去重
```

### webhook_deliveries — Webhook 投递记录

```typescript
webhookDeliveries {
  id: uuid PK
  appId: uuid NOT NULL
  publishTaskId: uuid NOT NULL
  eventId: uuid NOT NULL
  targetUrl: text NOT NULL
  requestBody: jsonb NOT NULL
  responseStatus: integer
  responseBody: text
  attemptCount: integer NOT NULL            // 当前重试次数
  deliveryStatus: varchar(32) NOT NULL      // pending, delivered, failed
  nextRetryAt: timestamp                    // 指数退避下次重试时间
  lastAttemptAt: timestamp
  createdAt: timestamp NOT NULL
  updatedAt: timestamp NOT NULL
}
// 唯一索引: (eventId, targetUrl) — 同一事件不重复投递到同一 URL
// 索引: (deliveryStatus, nextRetryAt) — 查询待重试的投递
```

## 索引策略

### 已有索引总览

| 表 | 索引类型 | 列 | 用途 |
|---|---------|---|------|
| apps | UNIQUE | apiKeyHash | API Key 快速查找 |
| authorized_accounts | UNIQUE | (platform, openId) | 同平台同用户去重（upsert 依赖） |
| account_tokens | UNIQUE | (authorizedAccountId) | 按 accountId 查 token |
| app_account_bindings | UNIQUE | (appId, authorizedAccountId) | 绑定去重 |
| app_account_bindings | INDEX | (appId, bindingStatus, authorizedAccountId) | 按应用查活跃绑定 |
| app_webhooks | INDEX | (appId, enabled, webhookUrl) | 查询应用的活跃 webhook |
| publish_tasks | UNIQUE | (appId, idempotencyKey) | 幂等去重 |
| publish_tasks | UNIQUE | (upstreamPublishId) | 上游 ID 去重 |
| publish_status_events | UNIQUE | (eventKey) | 事件去重（webhook 回调场景） |
| webhook_deliveries | UNIQUE | (eventId, targetUrl) | 投递去重 |
| webhook_deliveries | INDEX | (deliveryStatus, nextRetryAt) | 待重试投递查询 |

### 索引设计原则

1. **唯一索引 = 幂等保障**：`idempotencyKey`、`eventKey` 等通过唯一索引防止重复
2. **组合索引关注列顺序**：区分度高的列在前（如 `appId` 在 `idempotencyKey` 前面）
3. **覆盖常见查询路径**：
   - `listByAppId(appId)` → 按 `appId` 过滤
   - `listStaleTasks({ statuses, updatedBefore })` → 按 `status + updatedAt`
   - `listExpiringAccessTokenRecords(expiresBefore)` → 按 `accessTokenExpiresAt`

## 扩展建议

### 1. 多应用隔离查询优化

如果需要频繁查询某个应用的所有发布任务，可添加：

```sql
CREATE INDEX idx_publish_tasks_app_status ON publish_tasks (appId, status);
```

### 2. 账号 Token 刷新扫描优化

定时刷新 Job 查询即将过期的 token：

```sql
-- 已有的唯一索引 (authorizedAccountId) 可覆盖单账号查询
-- 批量扫描建议添加：
CREATE INDEX idx_account_tokens_access_expires ON account_tokens (accessTokenExpiresAt)
WHERE accessTokenExpiresAt > now();
```

### 3. 历史数据归档

`publish_status_events` 和 `webhook_deliveries` 是高频写入表，建议：

- 设置数据保留策略（如保留 90 天）
- 定期归档到冷存储或分析库
- 考虑分区表（按月/季度）

### 4. 扩展新平台

如果需要支持新平台（如 YouTube、Instagram），扩展 `platformEnum`：

```sql
ALTER TYPE platform ADD VALUE 'youtube' BEFORE 'tiktok';
```

`authorized_accounts` 表的 TTS 专属字段使用 nullable 设计，新平台可复用现有字段或新增 nullable 列。

### 5. Webhook 投递监控

建议添加投递统计视图：

```sql
CREATE VIEW webhook_delivery_stats AS
SELECT
  appId,
  targetUrl,
  COUNT(*) AS total_deliveries,
  SUM(CASE WHEN deliveryStatus = 'delivered' THEN 1 ELSE 0 END) AS success_count,
  SUM(CASE WHEN deliveryStatus = 'failed' THEN 1 ELSE 0 END) AS failed_count,
  AVG(attemptCount) AS avg_attempts
FROM webhook_deliveries
GROUP BY appId, targetUrl;
```

### 6. Redis 数据结构补充

OAuth state 使用 Redis 存储（非数据库表）：

```
Key: oauth:state:{stateUuid}
Value: JSON { appId, redirectUri, platform }
TTL: 600 seconds

Key: tt:refresh_lock:{authorizedAccountId}
Value: lockOwnerToken
TTL: refreshLockTtlSeconds (configurable)
```
