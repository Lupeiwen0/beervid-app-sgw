# OAuth 授权

## 目录

- [概览](#概览)
- [发起授权](#发起授权)
- [接收回调](#接收回调)
- [透传业务 State](#透传业务-state)
- [账号状态与重新授权](#账号状态与重新授权)
- [正确/错误示例](#正确错误示例)

## 概览

支持两个平台的 OAuth 授权，接入方式完全一致：

| | TikTok (TT) | TikTok Shop (TTS) |
|---|---|---|
| 接口 | `POST /open-api/v1/tt/auth/link` | `POST /open-api/v1/tts/auth/link` |
| 鉴权 | `X-Api-Key` | `X-Api-Key` |

下游只需做三件事：

1. **调用接口**，传入 `redirectUri`
2. **跳转用户浏览器**到返回的 `authorizeUrl`
3. **接收回调**——用户授权后，系统 302 重定向到你的 `redirectUri`，URL 的 `state` 参数携带账号信息

```
你的前端                     你的服务端                      本服务
  │                            │                            │
  │  POST /auth/link           │                            │
  │  { redirectUri }           │                            │
  │───────────────────────────>│───────────────────────────>│
  │                            │  { authorizeUrl, state }   │
  │  返回 authorizeUrl         │<───────────────────────────│
  │<───────────────────────────│                            │
  │                            │                            │
  │  window.location.href = authorizeUrl                    │
  │──────────────────────────────用户在 TikTok/TTS 授权页──>│
  │                            │                            │
  │                            │       (内部处理授权回调)     │
  │                            │                            │
  │  302 → redirectUri?state=<encoded JSON>                 │
  │<───────────────────────────│<───────────────────────────│
  │                            │                            │
  │  解析 state → 拿到 authorizedAccountId                  │
  │                            │                            │
```

## 发起授权

### Request

```
POST /open-api/v1/{tt|tts}/auth/link
X-Api-Key: <apiKey>
Content-Type: application/json
```

```json
{
  "redirectUri": "https://your-app.com/oauth/callback"
}
```

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `redirectUri` | string (URL) | 是 | 授权完成后重定向地址 |

### Response `200`

```json
{
  "code": 0,
  "data": {
    "state": "uuid",
    "authorizeUrl": "https://www.tiktok.com/v2/auth/authorize?..."
  }
}
```

| 字段 | 说明 |
|------|------|
| `state` | 服务端生成的 OAuth state（无需关注） |
| `authorizeUrl` | 将用户浏览器跳转到此地址 |

## 接收回调

用户在 TikTok/TTS 完成授权后，系统会 **302 重定向**到你传入的 `redirectUri`。

回调 URL 格式：

```
https://your-app.com/oauth/callback?state=<encodeURIComponent(JSON)>
```

`state` 解码后是一个 JSON 对象，始终包含以下字段：

```json
{
  "id": "authorizedAccountId",   // 后续接口使用的账号 ID
  "openId": "ou_xxx",           // 平台 OpenID
  "username": "john_doe",       // 用户名
  "platform": "tt"              // "tt" 或 "tts"
}
```

### 解析示例

```javascript
// 回调到达你的前端页面
const urlParams = new URLSearchParams(window.location.search);
const rawState = urlParams.get('state');

// URL decode + JSON parse
const state = JSON.parse(decodeURIComponent(rawState));

console.log(state.id);       // authorizedAccountId
console.log(state.platform); // "tt" 或 "tts"

// 保存 authorizedAccountId，后续上传/发布/查询都需要
saveAuthorizedAccountId(state.id);
```

## 透传业务 State

如果你需要在授权回调时拿到自己的业务上下文（如 sessionId、returnUrl），可以通过 `redirectUri` 的 `state` query 参数传入。

### 传入方式

```javascript
const businessState = { sessionId: 'sess_abc123', returnUrl: '/dashboard' };
const redirectUri = `https://your-app.com/callback?state=${
  encodeURIComponent(JSON.stringify(businessState))
}`;

const resp = await fetch('/open-api/v1/tt/auth/link', {
  method: 'POST',
  headers: { 'X-Api-Key': apiKey, 'Content-Type': 'application/json' },
  body: JSON.stringify({ redirectUri })
});
```

### 回调结果

回调时，你的业务 state 会与账号信息**合并**返回：

```javascript
// 回调 state 解码后：
{
  sessionId: 'sess_abc123',     // 你的业务字段
  returnUrl: '/dashboard',      // 你的业务字段
  id: 'account-uuid',          // 系统追加
  openId: 'ou_xxx',            // 系统追加
  username: 'john_doe',        // 系统追加
  platform: 'tt'               // 系统追加
}
```

### 业务 state 不是合法 JSON

如果你的 `state` 参数不是 JSON 字符串，原始值会被保留到 `_original` 字段：

```javascript
// redirectUri = "https://your-app.com/callback?state=my-token-123"
// 回调解码后：
{
  _original: 'my-token-123',   // 原始值被保留
  id: 'account-uuid',
  openId: 'ou_xxx',
  username: 'john_doe',
  platform: 'tt'
}
```

## TT 与 TTS 同用户账号关联

同一用户可能同时授权 TT 和 TTS 两个平台。由于 TikTok 和 TikTok Shop 之间没有统一的 openId 体系，两个平台返回的 `openId` 不同，无法直接关联。

### 推荐方式：通过 username 关联

回调 state 中的 `username` 是当前最可靠的关联字段——同一用户在 TT 和 TTS 上的 username 通常一致。

```
TT 回调:  { id: "acc-tt-1",   username: "john_doe", platform: "tt" }
TTS 回调: { id: "acc-tts-1",  username: "john_doe", platform: "tts" }
                                        ↑ 相同，可关联
```

### 关联实现建议

```javascript
// 你的用户表中存储 username → 账号映射
const userBindings = {
  'john_doe': {
    ttAccountId: 'acc-tt-1',     // TT 回调时写入
    ttsAccountId: null           // TTS 回调时补写
  }
};

function handleOAuthCallback() {
  const state = JSON.parse(decodeURIComponent(new URLSearchParams(location.search).get('state')));

  if (state.platform === 'tt') {
    userBindings[state.username].ttAccountId = state.id;
  } else {
    userBindings[state.username].ttsAccountId = state.id;
  }
}
```

### 注意事项

- `username` 理论上可能被用户修改（TT/TTS 侧），建议在每次授权回调时更新映射
- 如果业务场景对关联准确性要求极高，可让用户在前端手动确认两个平台账号是否为同一人
- 不要使用 `openId` 关联——TT 和 TTS 的 openId 是完全不同的值

## 账号状态与重新授权

授权完成后，可通过 `GET /open-api/v1/accounts` 查看所有已授权账号：

```json
{
  "code": 0,
  "data": {
    "accounts": [
      {
        "id": "authorizedAccountId",
        "authStatus": "active",
        "username": "john_doe"
      }
    ]
  }
}
```

`authStatus` 决定账号是否可用：

| authStatus | 含义 | 你应该做 |
|---|---|---|
| `active` | 正常可用 | 直接使用 |
| `expired` | 授权已过期 | 重新发起授权流程 |
| `revoked` | 用户撤销授权 | 重新发起授权流程 |
| `refresh_failed` | token 刷新失败 | 可重试接口调用；持续失败则重新授权 |

**Token 刷新是自动的**——你不需要关心 token 过期、刷新等细节，系统会在调用各接口时自动处理。只有 `expired` 和 `revoked` 状态需要引导用户重新授权。

### 重新授权

重新授权的流程与首次授权完全一致，再次调用 `POST /auth/link` 即可。系统会 upsert 账号记录，`authorizedAccountId` 不变（同平台同 openId）。

## 正确/错误示例

### ✅ 正确：带业务 state 发起授权

```javascript
const baseUrl = 'https://myapp.com/oauth/callback';
const bizState = { sessionId: 'sess_abc123', returnUrl: '/dashboard' };
const redirectUri = `${baseUrl}?state=${encodeURIComponent(JSON.stringify(bizState))}`;

const response = await fetch('/open-api/v1/tt/auth/link', {
  method: 'POST',
  headers: { 'X-Api-Key': apiKey, 'Content-Type': 'application/json' },
  body: JSON.stringify({ redirectUri })
});
const data = await response.json();

const { authorizeUrl } = data.data;
window.location.href = authorizeUrl;
```

### ✅ 正确：解析回调 state

```javascript
function handleOAuthCallback() {
  const params = new URLSearchParams(window.location.search);
  const state = JSON.parse(decodeURIComponent(params.get('state')));

  // state.id = authorizedAccountId，保存下来后续使用
  localStorage.setItem('authorizedAccountId', state.id);
  localStorage.setItem('platform', state.platform);

  if (state.returnUrl) {
    window.location.href = state.returnUrl;
  }
}
```

### ✅ 正确：同一用户授权 TT + TTS

```javascript
// TT 授权
const ttResp = await fetch('/open-api/v1/tt/auth/link', {
  method: 'POST',
  headers: { 'X-Api-Key': apiKey, 'Content-Type': 'application/json' },
  body: JSON.stringify({ redirectUri: `${baseUrl}/tt-callback` })
}).then(r => r.json());

// TTS 授权（同一用户可以同时授权两个平台）
const ttsResp = await fetch('/open-api/v1/tts/auth/link', {
  method: 'POST',
  headers: { 'X-Api-Key': apiKey, 'Content-Type': 'application/json' },
  body: JSON.stringify({ redirectUri: `${baseUrl}/tts-callback` })
}).then(r => r.json());

// 两次授权分别返回各自的 authorizedAccountId
// TT 的 accountId 只能用于 TT 功能，TTS 的只能用于 TTS 功能
```

### ❌ 错误：混用平台账号

```javascript
// 用 TT 授权返回的 accountId 调用 TTS 接口
await fetch('/open-api/v1/tts/videos/publish', {
  method: 'POST',
  headers: { 'X-Api-Key': apiKey, 'Idempotency-Key': crypto.randomUUID() },
  body: JSON.stringify({
    authorizedAccountId: ttAccountId,  // ❌ 这是 TT 的 accountId
    // ...
  })
});
// → 400 PLATFORM_MISMATCH
```

### ❌ 错误：忽略 authStatus 直接使用

```javascript
// 不检查 authStatus，在账号过期时直接调用接口
await fetch(`/open-api/v1/tt/videos?authorizedAccountId=${id}`, {
  headers: { 'X-Api-Key': apiKey }
});
// 如果账号 authStatus = 'expired'，接口可能返回 401 TT_REAUTH_REQUIRED
```

### ❌ 错误：redirectUri 的 state 不做 JSON 编码

```javascript
const redirectUri = `https://myapp.com/callback?state=some-random-string`;
// 不会报错，但回调时解析结果是：
// { _original: 'some-random-string', id: '...', ... }
// 需要额外处理 _original 字段，不如直接传 JSON
```
