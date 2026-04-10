# 错误处理

## 目录

- [错误响应格式](#错误响应格式)
- [完整错误码表](#完整错误码表)
- [重试策略](#重试策略)
- [常见错误排查](#常见错误排查)
- [错误处理最佳实践](#错误处理最佳实践)

## 错误响应格式

所有业务错误使用统一响应格式：

```json
{
  "code": "ERROR_CODE",
  "message": "人类可读的错误描述",
  "requestId": "uuid（对应响应头 x-request-id）"
}
```

成功响应：
```json
{
  "code": 0,
  "message": "ok",
  "data": { ... }
}
```

区分成功/错误的关键：成功时 `code` 为数字 `0`，错误时 `code` 为字符串。

## 完整错误码表

### 认证错误（401）

| Code | 触发场景 | 处理方式 |
|------|---------|---------|
| `UNAUTHORIZED` | 缺少或无效的 API Key / Admin Token | 检查鉴权 header 是否正确 |
| `INVALID_UPLOAD_TOKEN` | Upload Token 格式错误或签名无效 | 重新生成 uploadToken |
| `UPLOAD_TOKEN_EXPIRED` | Upload Token 超过 10 分钟有效期 | 重新生成 uploadToken |
| `TT_REAUTH_REQUIRED` | TikTok refresh token 已过期 | 引导用户重新走 OAuth 授权流程 |
| `TT_WEBHOOK_SIGNATURE_INVALID` | TikTok webhook 签名缺失或无效 | 系统内部处理，客户端无需关注 |

### 参数错误（400）

| Code | 触发场景 | 处理方式 |
|------|---------|---------|
| `BAD_REQUEST` | 通用参数错误 | 检查请求参数 |
| `VALIDATION_ERROR` | Zod 校验失败（字段类型、长度、格式等） | 检查 body/query/path 参数格式 |
| `INVALID_STATE` | OAuth state 无效、过期或平台不匹配 | 重新发起 auth/link |
| `MISSING_FILE` | 上传请求未包含 file 字段 | 确认 multipart/form-data 包含 file |
| `MISSING_IDEMPOTENCY_KEY` | 发布接口缺少 Idempotency-Key header | 添加 `Idempotency-Key` header |
| `PLATFORM_MISMATCH` | 账号平台与请求平台不一致 | 检查 authorizedAccountId 对应的平台 |
| `INVALID_ACCOUNT` | 账号不属于当前应用或平台不匹配 | 确认 accountId 和 appId 归属 |
| `INVALID_TT_WEBHOOK_PAYLOAD` | TikTok webhook payload 非法 | 系统内部处理 |
| `TT_WEBHOOK_CLIENT_KEY_INVALID` | TikTok webhook client_key 非法 | 系统内部处理 |

### 资源不存在（404）

| Code | 触发场景 | 处理方式 |
|------|---------|---------|
| `APP_NOT_FOUND` | 应用不存在或 API Key 对应的应用无效 | 检查 appId 或重新创建应用 |
| `ACCOUNT_NOT_FOUND` | 授权账号不存在 | 确认 authorizedAccountId 正确 |
| `MEDIA_NOT_FOUND` | OSS 文件不存在 | 确认 ossUrl 对应的文件已上传 |
| `PUBLISH_TASK_NOT_FOUND` | 发布任务不存在或不在当前 app 下 | 确认 publishTaskId 正确 |

### 权限错误（403）

| Code | 触发场景 | 处理方式 |
|------|---------|---------|
| `FORBIDDEN` | OSS bucket 或前缀不匹配 | 不要伪造 ossUrl |

### 服务不可用（503）

| Code | 触发场景 | 处理方式 |
|------|---------|---------|
| `TTS_NOT_CONFIGURED` | TTS 集成未配置（服务端配置问题） | 联系管理员 |
| `TTS_AUTH_UNAVAILABLE` | TTS 认证服务不可用 | 稍后重试 |
| `TOKEN_REFRESH_IN_PROGRESS` | 同一账号的 token 正在刷新中 | 等待后重试 |
| `TOKEN_REFRESH_UNAVAILABLE` | 当前平台未配置 refresh 能力 | 联系管理员 |

### 服务端错误（500）

| Code | 触发场景 | 处理方式 |
|------|---------|---------|
| `INTERNAL_SERVER_ERROR` | 未捕获的服务端错误 | 带 `x-request-id` 联系管理员 |

### 回调内部错误

| Code | HTTP | 触发场景 |
|------|------|---------|
| `TOKEN_EXCHANGE_FAILED` | 503 | TikTok/TTS token 兑换失败 |

## 重试策略

### 可重试的错误

| 错误码 | 重试策略 |
|--------|---------|
| `TOKEN_REFRESH_IN_PROGRESS` | 固定间隔 2-3 秒，最多 3 次 |
| `TTS_AUTH_UNAVAILABLE` | 指数退避，初始 1 秒，最多 5 次 |
| `TOKEN_EXCHANGE_FAILED` | 指数退避，初始 2 秒，最多 3 次 |
| `INTERNAL_SERVER_ERROR` | 指数退避，初始 1 秒，最多 3 次 |
| 5xx 错误（网络/超时） | 指数退避，初始 1 秒，最多 5 次 |

### 不可重试的错误

| 错误码 | 原因 |
|--------|------|
| `VALIDATION_ERROR` | 参数格式错误，重试无意义，需修正参数 |
| `PLATFORM_MISMATCH` | 账号平台不对，需要换正确的 accountId |
| `TT_REAUTH_REQUIRED` | 需要用户重新授权 |
| `APP_NOT_FOUND` / `ACCOUNT_NOT_FOUND` | 资源不存在 |
| `MISSING_IDEMPOTENCY_KEY` | 缺少必要 header |
| `FORBIDDEN` | 权限不足 |

### 重试实现示例

```javascript
async function fetchWithRetry(url, options, maxRetries = 3) {
  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    const resp = await fetch(url, options);
    const data = await resp.json();

    // 成功
    if (data.code === 0) return data;

    // 不可重试的错误
    const nonRetryable = [
      'VALIDATION_ERROR', 'PLATFORM_MISMATCH', 'INVALID_ACCOUNT',
      'TT_REAUTH_REQUIRED', 'APP_NOT_FOUND', 'ACCOUNT_NOT_FOUND',
      'MISSING_IDEMPOTENCY_KEY', 'FORBIDDEN', 'MISSING_FILE'
    ];
    if (nonRetryable.includes(data.code)) throw data;

    // 可重试
    if (attempt < maxRetries) {
      const delay = Math.min(1000 * Math.pow(2, attempt), 10000); // 指数退避，最大 10 秒
      await new Promise(r => setTimeout(r, delay));
      continue;
    }

    throw data;
  }
}
```

## 常见错误排查

### PLATFORM_MISMATCH / INVALID_ACCOUNT

**根因**：使用了错误平台的 authorizedAccountId

```javascript
// 问题：用 TT 账号调 TTS 接口
const result = await publishTtsVideo({
  authorizedAccountId: 'tt-account-uuid', // ← TT 账号
  // ...
});
// 返回: { code: 'PLATFORM_MISMATCH', message: 'Account platform is not tts' }

// 解决：使用对应平台的账号
const accounts = await listAccounts();
const ttsAccount = accounts.find(a => /* TTS 账号 */);
```

### TT_REAUTH_REQUIRED

**根因**：TikTok refresh token 已过期，无法自动续期

```
正常流程：access token 过期 → 用 refresh token 自动刷新 → 继续
异常流程：refresh token 也过期 → 无法刷新 → TT_REAUTH_REQUIRED
```

处理方式：
1. 提示用户授权已过期
2. 重新调用 `POST /open-api/v1/tt/auth/link`
3. 完成授权后用新的 accountId

### TOKEN_REFRESH_IN_PROGRESS

**根因**：并发请求触发同一账号的 token 刷新，Redis 锁冲突

处理方式：等待 2-3 秒后重试，通常一次重试即可。

### VALIDATION_ERROR

**根因**：Zod 校验失败，通常是参数格式问题

```json
{
  "code": "VALIDATION_ERROR",
  "message": "Validation failed",
  "details": [
    { "field": "caption", "message": "String must contain at least 1 character(s)" }
  ]
}
```

检查 `message` 中的具体字段和规则。

### UPLOAD_TOKEN_EXPIRED

**根因**：Upload Token 超过 10 分钟有效期

Upload Token 结构为 `Base64(appId:expiration:HMAC-SHA256)`，生成后应立即使用。

## 错误处理最佳实践

### 1. 全局错误拦截

```javascript
class TtsManagerError extends Error {
  constructor(code, message, requestId) {
    super(message);
    this.code = code;
    this.requestId = requestId;
  }
}

async function callApi(url, options) {
  const resp = await fetch(url, options);
  const data = await resp.json();

  if (data.code === 0) return data.data;

  // 保存 requestId 便于问题追踪
  throw new TtsManagerError(data.code, data.message, data.requestId);
}
```

### 2. 区分用户错误和系统错误

```javascript
function handleError(error) {
  if (error.code === 'TT_REAUTH_REQUIRED') {
    // 用户需要操作：重新授权
    redirectToOAuth();
  } else if (error.code === 'VALIDATION_ERROR' || error.code === 'PLATFORM_MISMATCH') {
    // 开发者需要修正：检查参数
    showValidationError(error.message);
  } else if (error.code === 'TOKEN_REFRESH_IN_PROGRESS') {
    // 系统临时状态：重试
    retryAfter(3000);
  } else {
    // 未知错误：上报
    reportError(error);
  }
}
```

### 3. 保留 requestId

所有错误响应包含 `requestId`（也在响应头 `x-request-id` 中），在排查问题时需要提供此 ID。

### 4. 幂等接口的错误处理

发布接口使用 `Idempotency-Key`，网络超时后应使用**相同的 key** 重试：

```javascript
async function safePublish(publishParams) {
  const idempotencyKey = crypto.randomUUID();

  // 持久化 idempotencyKey（如 localStorage）
  saveIdempotencyKey(publishParams.referenceId, idempotencyKey);

  try {
    return await callApi('/open-api/v1/tt/videos/publish', {
      method: 'POST',
      headers: {
        'X-Api-Key': apiKey,
        'Idempotency-Key': idempotencyKey
      },
      body: JSON.stringify(publishParams)
    });
  } catch (error) {
    if (isNetworkError(error)) {
      // 网络超时：用同一个 key 重试
      // 服务端幂等保证不会创建重复任务
      return await retryWithSameKey(idempotencyKey, publishParams);
    }
    throw error;
  }
}
```
