> 做这个项目是因为一件挺有意思的事。很多独立开发者需要验证用户邮箱，海外的免费服务一般一天只让发 100-200 封，想多用花Money。我就想，有没有那种拿来就能用的接口？于是就有了这个服务。
>
> 不过免费帮大家发邮件这事不太现实——发多了 IP 很快就被各大邮箱的反垃圾系统拉黑。所以换了个思路：**收邮件**。你不需要让用户填邮箱地址，只需要让用户用任意邮箱往指定地址发一封包含验证码的邮件就行，验证通过后你自然就拿到了用户的邮箱，再存到用户信息里就好了。
>
> 希望对你有用 📬

# 邮箱验证服务 API 文档

> Base URL: `https://api.mailflow.lat`
>
> 所有接口返回 JSON 格式数据。

## 概述

本服务提供邮箱所有权验证能力。核心流程：

1. 调用 `/create` 创建验证任务，获得收件地址和验证 token
2. 引导用户向指定地址发送邮件，邮件正文需包含 token
3. 轮询 `/result` 查询验证状态，直到返回 `verified`
4. 验证完成后可调用 `/task` 删除任务

```
用户发送邮件 → verify+<alias>@mailflow.lat
                        ↓
                服务自动完成验证
                        ↓
              状态变为 verified → 可通过 /result 查询
```

## 限流

| 层级 | 限制 | 说明 |
|------|------|------|
| 日级上限 | 1000 req/日/IP | 北京时间 00:00 重置，超限返回 429 |
| 秒级突发 | 10 req/s/IP | 允许短时突发 |

超限响应：

```json
HTTP 429 Too Many Requests
Retry-After: <秒数>
{"error": "rate limit exceeded"}
```

---

## 接口列表

| 方法 | 路径 | 鉴权 | 说明 |
|------|------|------|------|
| GET | /health | 无 | 健康检查 |
| POST | /create | 无 | 创建验证任务 |
| POST | /result | 凭证 | 查询验证结果 |
| DELETE | /task | 凭证 | 删除任务 |

---

## GET /health

健康检查。

### 请求

无参数，无请求体。

### 响应

| 状态码 | 含义 | 响应体 |
|--------|------|--------|
| 200 | 服务正常 | `{"status": "ok"}` |
| 503 | 服务不可用 | `{"status": "degraded"}` |

### 示例

```bash
curl https://api.mailflow.lat/health
```

```json
{"status": "ok"}
```

---

## POST /create

创建邮箱验证任务。返回收件地址和验证信息，调用方需引导用户发送包含 token 的邮件。

### 请求

| 字段 | 类型 | 必需 | 说明 |
|------|------|------|------|
| callback | string | 否 | 验证成功后的回调通知 URL，必须 HTTPS。留空则不回调 |

### 响应

| 状态码 | 含义 | 响应体 |
|--------|------|--------|
| 200 | 创建成功 | 见下方 |
| 400 | 请求体格式错误 | `{"error": "bad payload"}` |
| 500 | 内部错误 | `{"error": "internal error"}` |

#### 200 响应字段

| 字段 | 类型 | 说明 |
|------|------|------|
| task_id | string | 任务唯一 ID |
| secret | string | 任务密钥，用于后续查询和删除 |
| signature | string | 凭证签名，用于身份验证 |
| mail | string | 收件地址，格式 `verify+<alias>@mailflow.lat` |
| subject | string | 建议邮件主题，格式 `VERIFY-<alias>` |
| token | string | 验证令牌，**必须出现在邮件正文中** |
| expire_at | integer | 过期时间（Unix 时间戳），默认 24 小时 |

### 示例

```bash
curl -X POST https://api.mailflow.lat/create \
  -H "Content-Type: application/json" \
  -d '{"callback": ""}'
```

```json
{
  "task_id": "019f4326-9e8a-79f2-9221-99d5bfe25d06",
  "secret": "WkveK_yM2FN0rxOkFQwLdqC93bCBCjV1VHoVOtZOAS0",
  "signature": "d4568c5b552c5c56868f786d42a125637bda035fcc2aa05179111e7457339deb",
  "mail": "verify+bjox5y9h@mailflow.lat",
  "subject": "VERIFY-bjox5y9h",
  "token": "YENsg_CZjz_23sEmrAko4JofsCDidM6nzS_L1yMfLZ8",
  "expire_at": 1783624432
}
```

### 使用说明

1. 将 `mail` 地址告知用户，作为收件人
2. 将 `subject` 作为邮件主题（或自定义，但建议包含 alias）
3. **将 `token` 放入邮件正文中** — 这是验证的关键
4. 保存 `task_id`、`secret`、`signature`，用于后续调用 `/result` 和 `/task`
5. 如需验证完成后自动接收通知，传入 `callback` URL

---

## POST /result

查询验证结果。需要创建任务时返回的凭证信息。

### 请求

| 字段 | 类型 | 必需 | 说明 |
|------|------|------|------|
| task_id | string | 是 | 任务 ID |
| secret | string | 是 | 任务密钥 |
| signature | string | 是 | 凭证签名 |

### 响应

| 状态码 | 含义 | 响应体 |
|--------|------|--------|
| 200 | 等待验证 | `{"status": "pending"}` |
| 200 | 验证通过 | `{"status": "verified", "email": "sender@example.com"}` |
| 400 | 请求体格式错误 | `{"error": "bad payload"}` |
| 403 | 凭证无效 | `{"error": "forbidden"}` |
| 404 | 任务不存在或已过期 | `{"error": "not found"}` |
| 500 | 内部错误 | `{"error": "internal error"}` |

#### 200 响应字段

**pending 状态：**

| 字段 | 类型 | 说明 |
|------|------|------|
| status | string | `"pending"` — 等待用户发送验证邮件 |

**verified 状态：**

| 字段 | 类型 | 说明 |
|------|------|------|
| status | string | `"verified"` — 邮箱验证通过 |
| email | string | 验证通过的发件人邮箱地址 |

### 示例

```bash
curl -X POST https://api.mailflow.lat/result \
  -H "Content-Type: application/json" \
  -d '{
    "task_id": "019f4326-9e8a-79f2-9221-99d5bfe25d06",
    "secret": "WkveK_yM2FN0rxOkFQwLdqC93bCBCjV1VHoVOtZOAS0",
    "signature": "d4568c5b552c5c56868f786d42a125637bda035fcc2aa05179111e7457339deb"
  }'
```

等待验证时：

```json
{"status": "pending"}
```

验证通过后：

```json
{"status": "verified", "email": "user@example.com"}
```

---

## DELETE /task

删除验证任务。需要创建任务时返回的凭证信息。

### 请求

| 字段 | 类型 | 必需 | 说明 |
|------|------|------|------|
| task_id | string | 是 | 任务 ID |
| secret | string | 是 | 任务密钥 |
| signature | string | 是 | 凭证签名 |

### 响应

| 状态码 | 含义 | 响应体 |
|--------|------|--------|
| 200 | 删除成功 | `{"ok": true}` |
| 400 | 请求体格式错误 | `{"error": "bad payload"}` |
| 403 | 凭证无效 | `{"error": "forbidden"}` |
| 404 | 任务不存在或已过期 | `{"error": "not found"}` |
| 500 | 内部错误 | `{"error": "internal error"}` |

### 示例 |

```bash
curl -X DELETE https://api.mailflow.lat/task \
  -H "Content-Type: application/json" \
  -d '{
    "task_id": "019f4326-9e8a-79f2-9221-99d5bfe25d06",
    "secret": "WkveK_yM2FN0rxOkFQwLdqC93bCBCjV1VHoVOtZOAS0",
    "signature": "d4568c5b552c5c56868f786d42a125637bda035fcc2aa05179111e7457339deb"
  }'
```

```json
{"ok": true}
```

---

## 回调通知

如果在创建任务时提供了 `callback` URL，验证通过后服务会向该 URL 发送 POST 请求：

### 回调请求

```
POST <callback_url>
Content-Type: application/json
```

```json
{
  "task_id": "019f4326-9e8a-79f2-9221-99d5bfe25d06",
  "status": "verified",
  "email": "user@example.com",
  "verified_at": 1783624500
}
```

### 回调字段

| 字段 | 类型 | 说明 |
|------|------|------|
| task_id | string | 任务 ID |
| status | string | `"verified"` |
| email | string | 验证通过的发件人邮箱 |
| verified_at | integer | 验证时间（Unix 时间戳） |

### 回调规则

- **仅 HTTPS** — HTTP 地址会被拒绝
- **请求超时 5 秒** — 服务发送 POST 后，如果 5 秒内未收到响应则放弃本次回调
- **回调失败不影响验证结果** — 验证状态已变为 verified，回调仅是通知
- **不重试** — 回调失败后不会再次发送
- **URL 最大长度 2048 字符**

---

## 错误码汇总

| HTTP 状态码 | error 值 | 触发条件 |
|-------------|----------|---------|
| 400 | `bad payload` | 请求体 JSON 格式错误或缺少必需字段 |
| 403 | `forbidden` | 凭证验证失败（task_id/secret/signature 不匹配） |
| 404 | `not found` | 任务不存在或已过期 |
| 429 | `rate limit exceeded` | 触发限流（日级或秒级） |
| 500 | `internal error` | 服务内部错误 |
| 503 | `degraded` | 服务不可用（仅 /health） |

---

## 完整流程示例

```bash
# 1. 创建验证任务
RESPONSE=$(curl -s -X POST https://api.mailflow.lat/create \
  -H "Content-Type: application/json" \
  -d '{"callback": ""}')

# 解析返回值
TASK_ID=$(echo $RESPONSE | jq -r '.task_id')
SECRET=$(echo $RESPONSE | jq -r '.secret')
SIGNATURE=$(echo $RESPONSE | jq -r '.signature')
MAIL=$(echo $RESPONSE | jq -r '.mail')
TOKEN=$(echo $RESPONSE | jq -r '.token')

echo "请发送邮件到: $MAIL"
echo "邮件正文需包含: $TOKEN"

# 2. 轮询验证结果（用户发送邮件后）
while true; do
  RESULT=$(curl -s -X POST https://api.mailflow.lat/result \
    -H "Content-Type: application/json" \
    -d "{\"task_id\":\"$TASK_ID\",\"secret\":\"$SECRET\",\"signature\":\"$SIGNATURE\"}")
  
  STATUS=$(echo $RESULT | jq -r '.status')
  
  if [ "$STATUS" = "verified" ]; then
    echo "验证通过! 发件人: $(echo $RESULT | jq -r '.email')"
    break
  elif [ "$STATUS" = "pending" ]; then
    echo "等待验证..."
    sleep 5
  else
    echo "异常: $RESULT"
    break
  fi
done

# 3. 删除任务
curl -s -X DELETE https://api.mailflow.lat/task \
  -H "Content-Type: application/json" \
  -d "{\"task_id\":\"$TASK_ID\",\"secret\":\"$SECRET\",\"signature\":\"$SIGNATURE\"}"
```

---

## 注意事项

- **token 必须出现在邮件正文中** — 这是验证的唯一依据，主题中包含 token 无效
- **任务默认 24 小时过期** — 过期后查询返回 404
- **同一任务只能验证一次** — 即使收到多封邮件，状态也只会从 pending 变为 verified 一次
- **验证通过后可多次查询** — 在任务过期或被删除前，/result 可反复调用，始终返回 verified 状态
- **凭证三件套（task_id + secret + signature）必须完整保存** — 缺少任何一个都无法查询或删除
- **邮件主题建议使用返回的 subject 字段** — 虽然不强制，但有助于用户识别
- **callback URL 必须使用 HTTPS** — HTTP 地址会被拒绝
- **未设置回调时，建议在用户发送邮件后 3-10 秒开始查询** — 邮件投递链路通常需要数秒，过早查询只会得到 pending

---

## 示例代码

### curl

```bash
# 创建任务
curl -s -X POST https://api.mailflow.lat/create \
  -H "Content-Type: application/json" \
  -d '{"callback": ""}'

# 查询结果
curl -s -X POST https://api.mailflow.lat/result \
  -H "Content-Type: application/json" \
  -d '{"task_id":"<task_id>","secret":"<secret>","signature":"<signature>"}'

# 删除任务
curl -s -X DELETE https://api.mailflow.lat/task \
  -H "Content-Type: application/json" \
  -d '{"task_id":"<task_id>","secret":"<secret>","signature":"<signature>"}'
```

### Python

```python
import requests

BASE = "https://api.mailflow.lat"

# 创建验证任务
resp = requests.post(f"{BASE}/create", json={"callback": ""})
task = resp.json()
print(f"收件地址: {task['mail']}")
print(f"邮件正文需包含: {task['token']}")

# 查询验证结果
resp = requests.post(f"{BASE}/result", json={
    "task_id": task["task_id"],
    "secret": task["secret"],
    "signature": task["signature"],
})
result = resp.json()
print(f"状态: {result['status']}")
# verified 时: result["email"] 为发件人地址

# 删除任务
requests.delete(f"{BASE}/task", json={
    "task_id": task["task_id"],
    "secret": task["secret"],
    "signature": task["signature"],
})
```

### Node.js

```javascript
const BASE = "https://api.mailflow.lat";

// 创建验证任务
const createResp = await fetch(`${BASE}/create`, {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({ callback: "" }),
});
const task = await createResp.json();
console.log(`收件地址: ${task.mail}`);
console.log(`邮件正文需包含: ${task.token}`);

// 查询验证结果
const resultResp = await fetch(`${BASE}/result`, {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({
    task_id: task.task_id,
    secret: task.secret,
    signature: task.signature,
  }),
});
const result = await resultResp.json();
console.log(`状态: ${result.status}`);
// verified 时: result.email 为发件人地址

// 删除任务
await fetch(`${BASE}/task`, {
  method: "DELETE",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({
    task_id: task.task_id,
    secret: task.secret,
    signature: task.signature,
  }),
});
```
