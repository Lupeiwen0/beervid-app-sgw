# 发布工作流

## 目录

- [端到端流程总览](#端到端流程总览)
- [TT 视频发布](#tt-视频发布)
- [TTS 挂车视频发布](#tts-挂车视频发布)
- [状态查询与轮询](#状态查询与轮询)
- [发布成功后查询视频数据](#发布成功后查询视频数据)
- [推荐：队列定时刷新数据](#推荐队列定时刷新数据)
- [幂等策略](#幂等策略)
- [平台约束速查](#平台约束速查)

## 端到端流程总览

无论 TT 还是 TTS，流程都是：**授权 → 上传 → 发布 → 查状态 → 查视频数据**。

```
  Step 1            Step 2             Step 3              Step 4              Step 5
  授权账号          上传视频            创建发布任务          查询状态            查询视频数据
  ────────         ────────           ──────────          ────────           ──────────
  POST              POST               POST                GET                GET
  /auth/link  →    /upload-token/   → /videos/publish  → /publish-tasks/:id → /tt/videos
  跳转授权         generate                                                  或
  回调拿 accountId  POST             返回 taskId          created → submitted /tts/videos/performances
                    /upload/*        → published/failed  → 用 upstreamVideoId
                    上传文件                              查询视频信息
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
| 视频数据查询 | `GET /open-api/v1/tt/videos` | `GET /open-api/v1/tts/videos/performances` |

## TT 视频发布

### Step 1：上传视频

两步完成：先生成 Upload Token，再上传文件。

```javascript
// 1. 生成 Upload Token（10 分钟有效）
const tokenResp = await fetch('/open-api/v1/upload-token/generate', {
  method: 'POST',
  headers: { 'X-Api-Key': apiKey }
});
const tokenData = await tokenResp.json();

// 2. 上传视频文件
const formData = new FormData();
formData.append('file', videoFile);

const uploadResp = await fetch('/open-api/v1/upload/tt-video', {
  method: 'POST',
  headers: { 'X-Upload-Token': tokenData.data.uploadToken },
  body: formData
});
const uploadData = await uploadResp.json();

// 返回 ossUrl，后续发布接口使用
const { ossUrl, fileSize, contentType } = uploadData.data;
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
const publishData = await publishResp.json();

// 返回 202，异步执行
const { id: publishTaskId, status } = publishData.data;
```

### Step 3：查询发布状态

```javascript
const taskResp = await fetch(`/open-api/v1/publish-tasks/${publishTaskId}`, {
  headers: { 'X-Api-Key': apiKey }
});
const task = await taskResp.json();

// task.data.status: 'created' | 'submitted' | 'published' | 'failed'
```

### Step 4：查询视频数据（发布成功后）

当 `status === 'published'` 时，使用返回的 `upstreamVideoId` 查询视频详情：

```javascript
if (task.data.status === 'published' && task.data.upstreamVideoId) {
  const videoResp = await fetch(
    `/open-api/v1/tt/videos?authorizedAccountId=${ttAccountId}` +
    `&videoIds=${task.data.upstreamVideoId}`,
    { headers: { 'X-Api-Key': apiKey } }
  );
  const videoData = await videoResp.json();
  // videoData.data.items[0] 包含 thumbnail_url, share_url 等
}
```

> 推荐使用队列定时刷新，详见 [推荐：队列定时刷新数据](#推荐队列定时刷新数据)

## TTS 挂车视频发布

### Step 1：上传视频

```javascript
// 1. 生成 Upload Token
const tokenResp = await fetch('/open-api/v1/upload-token/generate', {
  method: 'POST',
  headers: { 'X-Api-Key': apiKey }
});
const tokenData = await tokenResp.json();

// 2. 上传视频文件（需携带 authorizedAccountId）
const formData = new FormData();
formData.append('file', videoFile);
formData.append('authorizedAccountId', ttsAccountId);  // TTS 授权账号 ID

const uploadResp = await fetch('/open-api/v1/upload/tts-video', {
  method: 'POST',
  headers: { 'X-Upload-Token': tokenData.data.uploadToken },
  body: formData
});
const uploadData = await uploadResp.json();

// 返回 videoFileId，后续发布接口使用
const { videoFileId } = uploadData.data;
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
const publishData = await publishResp.json();
const { id: publishTaskId, status } = publishData.data;

// TTS 创建接口仍返回 202 + created，但通常很快进入终态
```

### Step 4：查询视频带货表现（发布成功后）

TTS 发布成功后，`upstreamVideoId` 可用于查询带货表现数据：

```javascript
const taskResp = await fetch(`/open-api/v1/publish-tasks/${publishTaskId}`, {
  headers: { 'X-Api-Key': apiKey }
});
const task = await taskResp.json();

if (task.data.status === 'published' && task.data.upstreamVideoId) {
  // T-1 延迟，查询昨天数据
  const yesterday = Math.floor(Date.now() / 1000) - 86400;
  const perfResp = await fetch(
    `/open-api/v1/tts/videos/performances` +
    `?authorizedAccountId=${ttsAccountId}` +
    `&videoIds=${task.data.upstreamVideoId}` +
    `&startTimeGe=${yesterday}` +
    `&endTimeLe=${yesterday}`,
    { headers: { 'X-Api-Key': apiKey } }
  );
  const perfData = await perfResp.json();
}
```

> TTS 数据有 T-1 延迟，推荐使用队列每天定时刷新，详见 [推荐：队列定时刷新数据](#推荐队列定时刷新数据)

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
    const data = await resp.json();

    const { status, failureReason } = data.data;

    if (status === 'published') return data.data;
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
    subscribedEvents: ['video.status.changed']
  })
});
```

Webhook 投递签名通过 `x-ttm-signature` header 传递。验证方式：用你配置的 `webhookSecret` 对请求 body（原始 JSON 字符串）做 HMAC-SHA256，比对 `x-ttm-signature` 值。

#### Webhook 请求体

```json
{
  "event": "video.status.changed",
  "platform": "tt",
  "app_id": "uuid",
  "authorized_account_id": "uuid",
  "open_id": "string",
  "publish_task_id": "uuid",
  "upstream_publish_id": "string | null",
  "upstream_video_id": "string | null",
  "status": "success | failed | submitted",
  "reason": "string",
  "occurred_at": "ISO8601"
}
```

| 字段 | 说明 |
|------|------|
| `status` | `success`（发布成功）、`failed`（发布失败）、`submitted`（已提交到平台） |
| `publish_task_id` | 对应创建发布接口返回的 `id`，用于关联你的发布任务记录 |
| `upstream_video_id` | 仅 `status: success` 时有值，可用于查询视频数据 |
| `reason` | 状态变更原因，失败时包含错误描述 |

## 发布成功后查询视频数据

发布成功（`status: 'published'`）后，发布任务返回中会包含 `upstreamVideoId`，这是 TikTok/TTS 平台的视频 ID，可用于查询视频的详细信息与数据表现。

### 获取 upstreamVideoId

发布任务状态为 `published` 时，`upstreamVideoId` 字段有值：

```javascript
const taskResp = await fetch(`/open-api/v1/publish-tasks/${publishTaskId}`, {
  headers: { 'X-Api-Key': apiKey }
});
const task = await taskResp.json();

if (task.data.status === 'published') {
  const videoId = task.data.upstreamVideoId;
  // 用 videoId 查询视频数据
}
```

### TT 视频数据查询

查询 TT 平台已发布视频的基本信息（缩略图、分享链接等）：

```javascript
const videosResp = await fetch(
  `/open-api/v1/tt/videos?authorizedAccountId=${ttAccountId}&videoIds=${upstreamVideoId}`,
  { headers: { 'X-Api-Key': apiKey } }
);
const videos = await videosResp.json();

// videos.data.items[0] 包含：
// - videoId: 视频ID
// - raw.item_id: TikTok 视频 item ID
// - raw.create_time: 创建时间戳
// - raw.thumbnail_url: 缩略图 URL
// - raw.share_url: 分享链接
```

也可以不传 `videoIds`，查询该账号下所有视频列表。

### TTS 视频带货表现查询

查询 TTS Creator 视频的带货表现数据（GMV、订单数、CTR 等）：

```javascript
const perfResp = await fetch(
  `/open-api/v1/tts/videos/performances` +
  `?authorizedAccountId=${ttsAccountId}` +
  `&videoIds=${upstreamVideoId}` +
  `&startTimeGe=${startTimestamp}` +
  `&endTimeLe=${endTimestamp}`,
  { headers: { 'X-Api-Key': apiKey } }
);
const performances = await perfResp.json();

// performances.data.videos[0].performances[0].metrics 包含：
// - anchorDisplayRate: 商品锚点展示率
// - clickThroughRate: 点击率
// - orderCount: 订单数
// - itemSoldCount: 售出商品数
// - gmv: { amount, currency } GMV 金额
```

**TTS 注意事项**：
- 仅 US Creator 可用
- 数据延迟 1 天（T-1），无法查询当天数据
- `startTimeGe` 必须在最近 180 天内
- 数据按天聚合，建议按天查询

## 推荐：队列定时刷新数据

视频发布后，发布状态和视频数据会随时间变化。**不建议仅在发布后做一次性查询**，推荐使用队列（或定时任务）定时消费并刷新数据，确保本地数据与平台保持同步。

### 发布状态同步

TT 发布是异步的，状态会从 `submitted` 变为 `published` 或 `failed`。系统会在后台自动同步状态，但客户端也应设计定期刷新机制：

```javascript
// 推荐：使用队列定时刷新发布状态
// 伪代码示例
const publishRefreshQueue = createQueue('publish-status-refresh');

// 每次发布后，入队一个延迟任务
publishRefreshQueue.add('refresh-status', {
  publishTaskId,
  authorizedAccountId,
  appId
}, {
  delay: 30_000,   // 30 秒后首次检查
  attempts: 10,     // 最多重试 10 次
  backoff: {        // 指数退避
    type: 'exponential',
    delay: 15_000
  }
});

// Worker: 检查状态，未完成则重新入队
publishRefreshQueue.process(async (job) => {
  const task = await fetchPublishTask(job.data.publishTaskId);

  if (task.status === 'published' || task.status === 'failed') {
    // 状态已终态，更新本地数据，开始查询视频数据
    if (task.status === 'published') {
      await scheduleVideoDataRefresh(task);
    }
    return;
  }

  // 未终态，抛出错误触发重试（按 backoff 策略延迟）
  throw new Error('Status not final yet');
});
```

### 视频数据定时刷新

视频发布成功后，视频的播放量、互动数据、带货表现会持续变化。推荐定时拉取最新数据：

```javascript
// 推荐：定时刷新视频数据
const videoDataRefreshQueue = createQueue('video-data-refresh');

// 发布成功后，安排数据刷新任务
async function scheduleVideoDataRefresh(publishTask) {
  const videoId = publishTask.upstreamVideoId;

  if (publishTask.platform === 'tt') {
    videoDataRefreshQueue.add('refresh-tt-video', {
      authorizedAccountId: publishTask.authorizedAccountId,
      videoId
    }, {
      repeat: { cron: '0 */4 * * *' },  // 每 4 小时刷新一次
      jobId: `tt-video-${videoId}`
    });
  }

  if (publishTask.platform === 'tts') {
    videoDataRefreshQueue.add('refresh-tts-performance', {
      authorizedAccountId: publishTask.authorizedAccountId,
      videoId
    }, {
      repeat: { cron: '0 6 * * *' },  // 每天 6 点刷新（T-1 数据）
      jobId: `tts-perf-${videoId}`
    });
  }
}

// TT Worker: 刷新视频基础数据
videoDataRefreshQueue.process('refresh-tt-video', async (job) => {
  const { authorizedAccountId, videoId } = job.data;
  const data = await fetchTTVideos(authorizedAccountId, [videoId]);
  // 更新本地数据库
  await updateLocalVideoData(videoId, data);
});

// TTS Worker: 刷新带货表现
videoDataRefreshQueue.process('refresh-tts-performance', async (job) => {
  const { authorizedAccountId, videoId } = job.data;
  const yesterday = getStartOfYesterday();
  const data = await fetchTTSPerformances(authorizedAccountId, [videoId], yesterday, yesterday);
  // 更新本地数据库
  await updateLocalPerformanceData(videoId, data);
});
```

### 刷新策略建议

| 数据类型 | 刷新频率 | 说明 |
|---------|---------|------|
| TT 发布状态 | 30s 起步，指数退避，最多 10 次 | 直到终态（published/failed） |
| TT 视频数据 | 每 4-6 小时 | 播放量、互动数据持续更新 |
| TTS 带货表现 | 每天 1 次（T-1） | 数据延迟 1 天，无需更高频率 |
| TTS 发布状态 | 2-3s 起步，最多 3 次 | TTS 通常直接成功，少量重试即可 |

**核心原则**：不要依赖一次性查询，用队列 + 定时任务持续同步数据，保证本地数据的时效性。

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
