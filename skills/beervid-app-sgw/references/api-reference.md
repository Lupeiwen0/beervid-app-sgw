# API 接口文档

## 目录

- [通用约定](#通用约定)
- [Admin API](#admin-api)
  - [创建应用](#post-admin-apiv1apps)
  - [配置 Webhook](#put-admin-apiv1appsidwebhook)
- [账号与授权](#账号与授权)
  - [获取账号列表](#get-open-apiv1accounts)
  - [TT 账号信息实时查询](#get-open-apiv1ttaccountsauthorizedaccountidinfo)
  - [TTS 账号信息实时查询](#get-open-apiv1ttsaccountsauthorizedaccountidinfo)
  - [TT OAuth 授权链接](#post-open-apiv1ttauthlink)
  - [TTS OAuth 授权链接](#post-open-apiv1ttsauthlink)
- [文件上传](#文件上传)
  - [生成 Upload Token](#post-open-apiv1upload-tokengenerate)
  - [TT 视频上传](#post-open-apiv1uploadtt-video)
  - [TTS 视频上传](#post-open-apiv1uploadtts-video)
- [商品查询](#商品查询)
  - [TTS 店铺商品](#get-open-apiv1ttsproductsshop)
  - [TTS 橱窗商品](#get-open-apiv1ttsproductsshowcase)
- [视频发布](#视频发布)
  - [TT 视频发布](#post-open-apiv1ttvideospublish)
  - [TTS 挂车视频发布](#post-open-apiv1ttsvideospublish)
  - [查询发布任务](#get-open-apiv1publish-taskspublishtaskid)
- [视频数据](#视频数据)
  - [TT 视频数据查询](#get-open-apiv1ttvideos)
  - [TTS 视频带货表现](#get-open-apiv1ttsvideosperformances)

## 通用约定

### 响应格式

```jsonc
// 成功
{ "code": 0, "message": "ok", "data": { ... } }

// 错误
{ "code": "ERROR_CODE", "message": "错误描述", "requestId": "uuid" }
```

成功时 `code` 为数字 `0`，错误时 `code` 为字符串。错误响应中的 `requestId` 对应响应头 `x-request-id`。

### 鉴权

| 路径前缀 | Header | 说明 |
|----------|--------|------|
| `/admin-api/*` | `Authorization: Bearer <token>` | Admin Token |
| `/open-api/*`（除 upload） | `X-Api-Key: <apiKey>` | 应用 API Key（仅 active 应用） |
| `/open-api/v1/upload/*` | `X-Upload-Token: <uploadToken>` | Upload Token，10 分钟有效 |

---

## Admin API

### POST /admin-api/v1/apps

创建应用，返回一次性 API Key。

**Request Body**

```json
{
  "name": "我的应用",
  "description": "应用描述（可选）"
}
```

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `name` | string | 是 | 1-128 字符 |
| `description` | string | 否 | 最大 1000 字符 |

**Response `201`**

```json
{
  "code": 0,
  "data": {
    "app": {
      "id": "uuid",
      "name": "string",
      "description": "string | null",
      "status": "active",
      "createdAt": "ISO8601",
      "updatedAt": "ISO8601"
    },
    "apiKey": "sk-..."
  }
}
```

`apiKey` 仅创建时返回一次，后续无法再获取。

---

### PUT /admin-api/v1/apps/:id/webhook

创建或更新应用的下游 webhook 配置。

**Path Params**

| 字段 | 说明 |
|------|------|
| `id` | 应用 ID |

**Request Body**

```json
{
  "webhookUrl": "https://your-app.com/webhooks",
  "webhookSecret": "your-secret",
  "enabled": true,
  "subscribedEvents": ["publish.published", "publish.failed"]
}
```

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `webhookUrl` | string (URL) | 是 | 回调地址 |
| `webhookSecret` | string | 是 | 1-255 字符，用于签名 |
| `enabled` | boolean | 是 | 是否启用 |
| `subscribedEvents` | string[] | 否 | 订阅事件列表，最多 100 条 |

**Response `200`**

```json
{
  "code": 0,
  "data": {
    "webhook": {
      "id": "uuid",
      "appId": "uuid",
      "webhookUrl": "string",
      "enabled": true,
      "subscribedEvents": ["..."],
      "timeoutMs": 5000,
      "createdAt": "ISO8601",
      "updatedAt": "ISO8601"
    }
  }
}
```

---

## 账号与授权

### GET /open-api/v1/accounts

获取当前应用下所有已授权账号（TT + TTS）。

**Response `200`**

```json
{
  "code": 0,
  "data": {
    "appId": "uuid",
    "accounts": [
      {
        "id": "uuid",
        "openId": "string",
        "username": "string",
        "displayName": "string | null",
        "avatarUrl": "string | null",
        "authStatus": "active | expired | revoked | refresh_failed",
        "grantedScopes": ["user.info.basic", "..."]
      }
    ]
  }
}
```

| authStatus | 说明 | 应该做 |
|---|---|---|
| `active` | 正常 | 正常使用 |
| `expired` | 授权过期 | 重新授权 |
| `revoked` | 用户撤销 | 重新授权 |
| `refresh_failed` | token 刷新失败 | 重试或重新授权 |

---

### GET /open-api/v1/tt/accounts/:authorizedAccountId/info

实时查询 TikTok 账号信息（从 TikTok 拉取最新资料并更新）。

**Path Params**

| 字段 | 说明 |
|------|------|
| `authorizedAccountId` | TT 授权账号 ID |

**Response `200`**

```json
{
  "code": 0,
  "data": {
    "id": "uuid",
    "openId": "string",
    "platform": "tt",
    "username": "string",
    "displayName": "string | null",
    "avatarUrl": "string | null",
    "authStatus": "active | expired | revoked | refresh_failed",
    "grantedScopes": ["user.info.basic", "..."],
    "lastAuthorizedAt": "ISO8601",
    "updatedAt": "ISO8601"
  }
}
```

**错误**

| HTTP | Code | 说明 |
|------|------|------|
| 404 | `ACCOUNT_NOT_FOUND` | 账号不存在 |
| 400 | `PLATFORM_MISMATCH` | 账号不是 TT 平台 |
| 401 | `TT_REAUTH_REQUIRED` | 授权已过期 |

---

### GET /open-api/v1/tts/accounts/:authorizedAccountId/info

实时查询 TikTok Shop Creator 账号信息（从 TTS API 拉取最新资料并更新本地）。

**Path Params**

| 字段 | 说明 |
|------|------|
| `authorizedAccountId` | TTS 授权账号 ID |

**Response `200`**

```json
{
  "code": 0,
  "data": {
    "id": "uuid",
    "openId": "string",
    "platform": "tts",
    "username": "string",
    "displayName": null,
    "avatarUrl": "string | null",
    "authStatus": "active | expired | revoked | refresh_failed",
    "grantedScopes": ["video.publish", "..."],
    "lastAuthorizedAt": "ISO8601",
    "updatedAt": "ISO8601",
    "userType": "AFFILIATE_CREATOR",
    "sellerType": "LOCAL",
    "registerRegion": "US",
    "selectionRegion": "US"
  }
}
```

**错误**

| HTTP | Code | 说明 |
|------|------|------|
| 400 | `PLATFORM_MISMATCH` | 账号不是 TTS 平台 |
| 404 | `ACCOUNT_NOT_FOUND` | 账号不存在 |
| 409 | `ACCOUNT_PROFILE_MISMATCH` | 拉取到的账号与请求账号不匹配 |
| 503 | `TTS_NOT_CONFIGURED` | TTS 集成未配置 |

---

### POST /open-api/v1/tt/auth/link

生成 TikTok OAuth 授权链接。

**Request Body**

```json
{
  "redirectUri": "https://your-app.com/callback"
}
```

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `redirectUri` | string (URL) | 是 | 授权完成后重定向地址 |

**Response `200`**

```json
{
  "code": 0,
  "data": {
    "state": "uuid",
    "authorizeUrl": "https://www.tiktok.com/v2/auth/authorize?..."
  }
}
```

将用户浏览器跳转到 `authorizeUrl`，授权完成后 302 到 `redirectUri`。回调 URL 的 `state` 参数格式：

```jsonc
// 解码后（无业务 state 时）
{ "id": "account_uuid", "openId": "ou_xxx", "username": "user", "platform": "tt" }

// 解码后（有业务 state 时，合并返回）
{ "yourField": "value", "id": "account_uuid", "openId": "ou_xxx", "username": "user", "platform": "tt" }
```

> 透传业务 state 的详细说明见 [OAuth 授权](oauth-authorization.md#透传业务-state)

---

### POST /open-api/v1/tts/auth/link

生成 TikTok Shop OAuth 授权链接。参数和返回格式与 TT 一致。

**Request Body**

```json
{
  "redirectUri": "https://your-app.com/callback"
}
```

**Response `200`**

```json
{
  "code": 0,
  "data": {
    "state": "uuid",
    "authorizeUrl": "https://shop.tiktok.com/alliance/creator/auth?..."
  }
}
```

回调 `state` 解码后 `platform` 为 `"tts"`。

---

## 文件上传

上传文件前需要先生成 Upload Token，上传时使用 `X-Upload-Token` 鉴权（非 `X-Api-Key`）。

### POST /open-api/v1/upload-token/generate

生成临时 Upload Token（10 分钟有效）。

**Response `200`**

```json
{
  "code": 0,
  "data": {
    "uploadToken": "base64_encoded_string",
    "expiresIn": 600
  }
}
```

**注意**：Upload Token 有效期仅 10 分钟，生成后立即使用。

---

### POST /open-api/v1/upload/tt-video

上传 TikTok 视频。

**Headers**: `X-Upload-Token`（必填）

**Request**: `multipart/form-data`

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `file` | binary | 是 | 视频文件 |

**Response `200`**

```json
{
  "code": 0,
  "data": {
    "ossUrl": "oss://bucket/tt/uploads/appId/uuid.mp4",
    "fileSize": 10485760,
    "contentType": "video/mp4"
  }
}
```

`ossUrl` 用于后续 `POST /open-api/v1/tt/videos/publish`。

---

### POST /open-api/v1/upload/tts-video

上传 TikTok Shop 视频。

**Headers**: `X-Upload-Token`（必填）

**Request**: `multipart/form-data`

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `file` | binary | 是 | 视频文件 |
| `authorizedAccountId` | string (UUID) | 是 | TTS 授权账号 ID |

**Response `200`**

```json
{
  "code": 0,
  "data": {
    "videoFileId": "v12d00gd0024d3t2197og65lmt35avog",
    "uploadType": "SMALL_FILE | LARGE_FILE_CHUNKED"
  }
}
```

`videoFileId` 用于后续 `POST /open-api/v1/tts/videos/publish`。

**错误**

| HTTP | Code | 说明 |
|------|------|------|
| 400 | `INVALID_ACCOUNT` | 账号不属于当前应用或不是 TTS 账号 |
| 503 | `TTS_NOT_CONFIGURED` | TTS 集成未配置 |

---

## 商品查询

### GET /open-api/v1/tts/products/shop

查询 TikTok Shop 店铺商品列表。

**Query Params**

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `authorizedAccountId` | UUID | 是 | TTS 授权账号 ID |
| `pageSize` | number | 是 | 每页数量，1-100 |
| `pageToken` | string | 否 | 翻页 token（从响应获取） |

**Response `200`**

```json
{
  "code": 0,
  "data": {
    "products": [
      {
        "id": "string",
        "title": "string",
        "price": { "currency": "USD", "amount": "19.99" },
        "addedStatus": "ADDED",
        "brandName": "string",
        "images": ["https://..."],
        "salesCount": 100
      }
    ],
    "totalCount": 50,
    "nextPageToken": "cursor_string | null"
  }
}
```

---

### GET /open-api/v1/tts/products/showcase

查询 TikTok Shop 橱窗商品列表。

**Query Params**

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `authorizedAccountId` | UUID | 是 | TTS 授权账号 ID |
| `pageSize` | number | 是 | 每页数量，1-100 |
| `pageToken` | string | 否 | 翻页 token |

**Response `200`**

```json
{
  "code": 0,
  "data": {
    "products": [
      {
        "id": "string",
        "title": "string",
        "mainImages": [{ "url": "https://..." }],
        "price": { "original_price": {}, "platform_discount_price": {} },
        "shop": { "name": "string" },
        "source": "AFFILIATE",
        "status": {
          "addedStatus": "APPROVED",
          "inventoryStatus": "IN_STOCK",
          "isHidden": false,
          "reviewStatus": "APPROVED"
        }
      }
    ],
    "totalCount": 30,
    "nextPageToken": "cursor_string | null"
  }
}
```

---

## 视频发布

### POST /open-api/v1/tt/videos/publish

创建 TikTok 视频发布任务（异步）。

**Headers**

| Header | 必填 | 说明 |
|--------|------|------|
| `X-Api-Key` | 是 | 应用 API Key |
| `Idempotency-Key` | 是 | 幂等键，推荐 UUID |

**Request Body**

```json
{
  "authorizedAccountId": "uuid",
  "ossUrl": "oss://bucket/tt/uploads/appId/uuid.mp4",
  "caption": "视频标题"
}
```

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `authorizedAccountId` | UUID | 是 | TT 授权账号 ID |
| `ossUrl` | string | 是 | 上传接口返回的 ossUrl |
| `caption` | string | 是 | 视频标题，至少 1 字符 |

**Response `202`**

```json
{
  "code": 0,
  "data": {
    "id": "uuid",
    "appId": "uuid",
    "authorizedAccountId": "uuid",
    "ossUrl": "oss://...",
    "platform": "tt",
    "caption": "string",
    "publishMode": "tt_direct",
    "status": "created",
    "upstreamPublishId": null,
    "upstreamVideoId": null,
    "failureCode": null,
    "failureReason": null,
    "productId": null,
    "videoFileId": null,
    "productTitle": null,
    "createdAt": "ISO8601",
    "updatedAt": "ISO8601",
    "publishedAt": null
  }
}
```

任务异步执行，通过 `GET /open-api/v1/publish-tasks/:id` 轮询状态或配置 webhook 接收通知。

**错误**

| HTTP | Code | 说明 |
|------|------|------|
| 400 | `MISSING_IDEMPOTENCY_KEY` | 缺少 Idempotency-Key header |
| 400 | `PLATFORM_MISMATCH` | 账号不是 TT 平台 |
| 404 | `ACCOUNT_NOT_FOUND` | 账号不存在 |
| 404 | `MEDIA_NOT_FOUND` | ossUrl 对应的文件不存在 |

---

### POST /open-api/v1/tts/videos/publish

创建 TikTok Shop 挂车视频发布任务。Headers 与 TT 一致。

**Request Body**

```json
{
  "authorizedAccountId": "uuid",
  "videoFileId": "v12d00gd0024d3t2197og65lmt35avog",
  "caption": "挂车视频标题 #标签",
  "productId": "product_id",
  "productTitle": "点击购买"
}
```

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `authorizedAccountId` | UUID | 是 | TTS 授权账号 ID |
| `videoFileId` | string | 是 | 上传接口返回的视频文件 ID |
| `caption` | string | 是 | 视频标题，支持 `#hashtag` `@mention` |
| `productId` | string | 是 | 商品查询接口返回的商品 ID |
| `productTitle` | string | 是 | 商品锚点文案，最多 30 字符 |

**Response `202`** — 结构同 TT 发布，但 `platform: "tts"`、`publishMode: "tts_direct"`、`ossUrl: null`，`videoFileId`/`productId`/`productTitle` 有值。

TTS 发布通常直接到达 `published` 状态。

**错误**

| HTTP | Code | 说明 |
|------|------|------|
| 400 | `MISSING_IDEMPOTENCY_KEY` | 缺少 Idempotency-Key header |
| 400 | `PLATFORM_MISMATCH` | 账号不是 TTS 平台 |
| 404 | `ACCOUNT_NOT_FOUND` | 账号不存在 |

---

### GET /open-api/v1/publish-tasks/:publishTaskId

查询发布任务详情。

**Path Params**

| 字段 | 说明 |
|------|------|
| `publishTaskId` | 发布任务 ID（创建接口返回的 `id`） |

**Response `200`** — 结构同创建接口返回。

**错误**

| HTTP | Code | 说明 |
|------|------|------|
| 404 | `PUBLISH_TASK_NOT_FOUND` | 任务不存在或不属于当前应用 |

---

## 视频数据

### GET /open-api/v1/tt/videos

查询 TikTok 视频数据。仅限 TT 授权账号。

**Query Params**

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `authorizedAccountId` | UUID | 是 | TT 授权账号 ID |
| `videoIds` | string | 否 | 逗号分隔的视频 ID；不传则不加该过滤 |

**示例**: `GET /open-api/v1/tt/videos?authorizedAccountId=xxx&videoIds=vid1,vid2`

**Response `200`**

```json
{
  "code": 0,
  "data": {
    "items": [
      {
        "videoId": "string",
        "raw": {
          "item_id": "string",
          "create_time": 0,
          "thumbnail_url": "https://...",
          "share_url": "https://..."
        }
      }
    ],
    "raw": {
      "list": []
    }
  }
}
```

**错误**

| HTTP | Code | 说明 |
|------|------|------|
| 404 | `ACCOUNT_NOT_FOUND` | 账号不存在 |
| 400 | `PLATFORM_MISMATCH` | 账号不是 TT 平台 |
| 401 | `TT_REAUTH_REQUIRED` | 授权已过期，需重新授权 |

---

### GET /open-api/v1/tts/videos/performances

查询 TTS Creator 视频带货表现数据（GMV、订单数、CTR 等）。仅限 TTS 授权账号，且仅 US Creator。

**Query Params**

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `authorizedAccountId` | UUID | 是 | TTS 授权账号 ID |
| `videoIds` | string | 是 | 逗号分隔的视频 ID，最多 100 个 |
| `startTimeGe` | number | 是 | 起始日期 Unix 时间戳 |
| `endTimeLe` | number | 是 | 结束日期 Unix 时间戳 |

**限制**：
- 仅 US Creator
- 数据延迟 1 天（T-1），无法查询当天
- `startTimeGe` 必须在最近 180 天内
- 数据按天聚合

**Response `200`**

```json
{
  "code": 0,
  "data": {
    "videos": [
      {
        "id": "7271486684427046149",
        "performances": [
          {
            "timeRange": {
              "startTime": 1704067200,
              "endTime": 1704067200
            },
            "metrics": {
              "anchorDisplayRate": "0.64",
              "clickThroughRate": "0.08",
              "orderCount": 3,
              "itemSoldCount": 3,
              "gmv": {
                "amount": "27.85",
                "currency": "USD"
              }
            }
          }
        ]
      }
    ]
  }
}
```

**错误**

| HTTP | Code | 说明 |
|------|------|------|
| 400 | `INVALID_ACCOUNT` | 账号不存在或不是 TTS 账号 |
| 503 | `TTS_NOT_CONFIGURED` | TTS 集成未配置 |
