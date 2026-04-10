# 发布工作流

## 目录

- [端到端流程总览](#端到端流程总览)
- [TT 视频发布](#tt-视频发布)
- [TTS 挂车视频发布](#tts-挂车视频发布)
- [状态查询与轮询](#状态查询与轮询)
- [幂等策略](#幂等策略)
- [平台约束速查](#平台约束速查)

## 端到端流程总览

无论 TT 还是 TTS，流程都是：**授权 → 上传 → 发布 → 查状态**。

```
  Step 1            Step 2             Step 3              Step 4
  授权账号          上传视频            创建发布任务          查询状态
  ────────         ────────           ──────────          ────────
  POST              POST               POST                GET
  /auth/link  →    /upload-token/   → /videos/publish  → /publish-tasks/:id
  跳转授权         generate
  回调拿 accountId  POST             返回 taskId          created → submitted
                    /upload/*        → published/failed
                    上传文件
```

### TT vs TTS 差异

| 步骤 | TT | TTS |
|------|----|-----|
| 上传接口 | `POST /open-api/v1/upload/tt-video` | `POST /open-api/v1/upload/tts-video` |
| 上传返回值 | `ossUrl` | `videoFileId` |
| 发布接口 | `POST /open-api/v1/tt/videos/publish` | `POST /open-api/v1/tts/videos/publish` |
| 发布参数 | `ossUrl` + `caption` | `videoFileId` + `caption` + `productId` + `productTitle` |
| 发布模式 | 异步（需轮询或 webhook） | 通常直接成功 |
| 商品查询 | 无 | `GET /tts/products/shop` 或 `/showcase` |

## TT 视频发布

### Step 1：上传视频

两步完成：先生成 Upload Token，再上传文件。

```javascript
// 1. 生成 Upload Token（10 分钟有效）
const tokenResp = await fetch('/open-api/v1/upload-token/generate', {
  method: 'POST',
  headers: { 'X-Api-Key': apiKey }
});

// 2. 上传视频文件
const formData = new FormData();
formData.append('file', videoFile);

const uploadResp = await fetch('/open-api/v1/upload/tt-video', {
  method: 'POST',
  headers: { 'X-Upload-Token': tokenResp.data.data.uploadToken },
  body: formData
});

// 返回 ossUrl，后续发布接口使用
const { ossUrl, fileSize, contentType } = uploadResp.data.data;
```

### Step 2：创建发布任务

```javascript
const publishResp = await fetch('/open-api/v1/tt/videos/publish', {
  method: 'POST',
  headers: {
    'X-Api-Key': apiKey,
    'Idempotency-Key': crypto.randomUUID(),  // 必填
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({
    authorizedAccountId: ttAccountId,  // TT 授权账号 ID
    ossUrl: ossUrl,                    // 上传接口返回的 ossUrl
    caption: '视频标题 #标签'
  })
});

// 返回 202，异步执行
const { id: publishTaskId, status } = publishResp.data.data;
```

### Step 3：查询发布状态

```javascript
const task = await fetch(`/open-api/v1/publish-tasks/${publishTaskId}`, {
  headers: { 'X-Api-Key': apiKey }
});

// task.data.data.status: 'created' | 'submitted' | 'published' | 'failed'
```

## TTS 挂车视频发布

### Step 1：上传视频

```javascript
// 1. 生成 Upload Token
const tokenResp = await fetch('/open-api/v1/upload-token/generate', {
  method: 'POST',
  headers: { 'X-Api-Key': apiKey }
});

// 2. 上传视频文件（需携带 authorizedAccountId）
const formData = new FormData();
formData.append('file', videoFile);
formData.append('authorizedAccountId', ttsAccountId);  // TTS 授权账号 ID

const uploadResp = await fetch('/open-api/v1/upload/tts-video', {
  method: 'POST',
  headers: { 'X-Upload-Token': tokenResp.data.data.uploadToken },
  body: formData
});

// 返回 videoFileId，后续发布接口使用
const { videoFileId } = uploadResp.data.data;
```

### Step 2：查询商品（可选）

发布挂车视频需要关联一个商品，可从店铺或橱窗获取：

```javascript
// 店铺商品
const shopProducts = await fetch(
  `/open-api/v1/tts/products/shop?authorizedAccountId=${ttsAccountId}&pageSize=20`,
  { headers: { 'X-Api-Key': apiKey } }
);

// 橱窗商品
const showcaseProducts = await fetch(
  `/open-api/v1/tts/products/showcase?authorizedAccountId=${ttsAccountId}&pageSize=20`,
  { headers: { 'X-Api-Key': apiKey } }
);
```

### Step 3：创建发布任务

```javascript
const publishResp = await fetch('/open-api/v1/tts/videos/publish', {
  method: 'POST',
  headers: {
    'X-Api-Key': apiKey,
    'Idempotency-Key': crypto.randomUUID(),  // 必填
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({
    authorizedAccountId: ttsAccountId,  // TTS 授权账号 ID
    videoFileId: videoFileId,           // 上传接口返回的 videoFileId
    caption: '挂车视频标题',
    productId: product.id,              // 商品查询接口返回的商品 ID
    productTitle: '点击购买'             // 商品锚点文案，最多 30 字符
  })
});

// TTS 发布通常直接到 published 状态
```

## 状态查询与轮询

### 发布状态流转

```
created → submitted → published
                    ↘ failed
   ↘ published（TTS 常见：直接成功）
   ↘ failed（直接失败）
```

### 轮询示例

```javascript
async function waitForPublishComplete(publishTaskId, options = {}) {
  const { maxAttempts = 30, intervalMs = 5000 } = options;

  for (let i = 0; i < maxAttempts; i++) {
    const resp = await fetch(`/open-api/v1/publish-tasks/${publishTaskId}`, {
      headers: { 'X-Api-Key': apiKey }
    });

    const { status, failureReason } = resp.data.data;

    if (status === 'published') return resp.data.data;
    if (status === 'failed') throw new Error(`Publish failed: ${failureReason}`);

    await new Promise(r => setTimeout(r, intervalMs));
  }

  throw new Error('Publish timeout');
}
```

**轮询间隔建议**：
- TT：5-10 秒（异步处理）
- TTS：2-3 秒（通常直接成功，轮询作为兜底即可）

### Webhook 替代轮询

配置 webhook 后，系统会在状态变更时主动推送，无需轮询：

```javascript
await fetch(`/admin-api/v1/apps/${appId}/webhook`, {
  method: 'PUT',
  headers: { 'Authorization': `Bearer ${adminToken}` },
  body: JSON.stringify({
    webhookUrl: 'https://your-app.com/webhooks/tt-manager',
    webhookSecret: 'your-signing-secret',
    enabled: true,
    subscribedEvents: ['publish.created', 'publish.submitted', 'publish.published', 'publish.failed']
  })
});
```

Webhook 投递签名通过 `x-ttm-signature` header 传递。

## 幂等策略

发布接口的 `Idempotency-Key` header 是**必填**的。

- **相同 key** → 返回已有任务（不重复创建）
- **已有任务状态为 `created`** → 自动重新入队执行
- **网络超时后** → 用同一个 key 重试，不会创建重复任务

### ✅ 正确：持久化 key，重试时复用

```javascript
const idempotencyKey = crypto.randomUUID();
// 持久化到本地（localStorage / 数据库）
await savePendingPublish({ userId, idempotencyKey, params });

const result = await fetch('/open-api/v1/tt/videos/publish', {
  method: 'POST',
  headers: {
    'X-Api-Key': apiKey,
    'Idempotency-Key': idempotencyKey
  },
  body: JSON.stringify(params)
});
```

### ❌ 错误：每次重试生成新 key

```javascript
// 这会创建多个重复任务
for (let i = 0; i < 3; i++) {
  await fetch('/open-api/v1/tt/videos/publish', {
    headers: { 'Idempotency-Key': crypto.randomUUID() },  // ❌ 每次新 key
    // ...
  });
}
```

## 平台约束速查

| 接口 | 需要的平台 | 说明 |
|------|-----------|------|
| `POST /upload/tt-video` | TT | 上传视频，返回 `ossUrl` |
| `POST /upload/tts-video` | TTS | 上传视频，返回 `videoFileId` |
| `POST /tt/videos/publish` | TT | 异步发布 |
| `POST /tts/videos/publish` | TTS | 通常同步发布 |
| `GET /tt/videos` | TT | 视频数据（insight） |
| `GET /tt/accounts/:id/info` | TT | 实时账号信息 |
| `GET /tts/products/shop` | TTS | 店铺商品 |
| `GET /tts/products/showcase` | TTS | 橱窗商品 |
| `GET /tts/videos/performances` | TTS | 视频带货表现（仅 US Creator，T-1 延迟） |

**核心规则**：TT 账号只能调 TT 接口，TTS 账号只能调 TTS 接口。混用会返回 `PLATFORM_MISMATCH` 或 `INVALID_ACCOUNT`。
