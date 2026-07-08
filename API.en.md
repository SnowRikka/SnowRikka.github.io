> I built this because of something interesting. Many indie developers need to verify user emails, but free overseas services typically limit you to 100-200 sends per day — and require registration plus DNS records to go beyond that. I wondered: isn't there an API you can just use, right away? So I built this service.
>
> Free email sending at scale isn't realistic — the sender IP gets blacklisted by major spam filters fast. So I flipped the approach: **receive mail instead**. You don't need users to type in their email address — just have them send a token-bearing email from any mailbox to a given address. Once verified, you've got their email, and you can store it in their profile.
>
> Hope you find it useful 📬

# Email Verification Service API Docs

English | [中文](./README.md)

> Base URL: `https://api.mailflow.lat`
>
> All endpoints return JSON.

## Overview

This service verifies email ownership. Core flow:

1. Call `/create` to start a verification task and get a recipient address + token
2. Have the user send an email to that address with the token in the body
3. Poll `/result` until the status becomes `verified`
4. Optionally call `/task` to delete the task

```
User sends email → verify+<alias>@mailflow.lat
                         ↓
                 Service auto-verifies
                         ↓
               Status becomes verified → query via /result
```

## Rate Limits

| Tier | Limit | Notes |
|------|-------|-------|
| Daily cap | 1000 req/day/IP | Resets at 00:00 Beijing Time (UTC+8); returns 429 when exceeded |
| Burst | 10 req/s/IP | Short bursts allowed |

Rate-limited response:

```json
HTTP 429 Too Many Requests
Retry-After: <seconds>
{"error": "rate limit exceeded"}
```

---

## Endpoints

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | /health | None | Health check |
| POST | /create | None | Create verification task |
| POST | /result | Credentials | Query verification result |
| DELETE | /task | Credentials | Delete task |

---

## GET /health

Health check.

### Request

No parameters, no request body.

### Response

| Status | Meaning | Body |
|--------|---------|------|
| 200 | Service OK | `{"status": "ok"}` |
| 503 | Service unavailable | `{"status": "degraded"}` |

### Example

```bash
curl https://api.mailflow.lat/health
```

```json
{"status": "ok"}
```

---

## POST /create

Create an email verification task. Returns the recipient address and verification info. The caller should guide the user to send an email containing the token.

### Request

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| callback | string | No | HTTPS URL to notify on verification success. Leave empty to skip |

### Response

| Status | Meaning | Body |
|--------|---------|------|
| 200 | Created | See below |
| 400 | Bad payload | `{"error": "bad payload"}` |
| 500 | Internal error | `{"error": "internal error"}` |

#### 200 Response Fields

| Field | Type | Description |
|-------|------|-------------|
| task_id | string | Unique task ID |
| secret | string | Task secret, used for subsequent queries and deletion |
| signature | string | Credential signature for authentication |
| mail | string | Recipient address, format `verify+<alias>@mailflow.lat` |
| subject | string | Suggested email subject, format `VERIFY-<alias>` |
| token | string | Verification token, **must appear in the email body** |
| expire_at | integer | Expiration time (Unix timestamp), default 24 hours |

### Example

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

### Usage Notes

1. Give the `mail` address to the user as the recipient
2. Use `subject` as the email subject (or customize, but include the alias)
3. **Put the `token` in the email body** — this is the key to verification
4. Save `task_id`, `secret`, `signature` for subsequent `/result` and `/task` calls
5. Pass a `callback` URL if you want automatic notification on verification

---

## POST /result

Query verification result. Requires the credentials returned at task creation.

### Request

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| task_id | string | Yes | Task ID |
| secret | string | Yes | Task secret |
| signature | string | Yes | Credential signature |

### Response

| Status | Meaning | Body |
|--------|---------|------|
| 200 | Pending | `{"status": "pending"}` |
| 200 | Verified | `{"status": "verified", "email": "sender@example.com"}` |
| 400 | Bad payload | `{"error": "bad payload"}` |
| 403 | Invalid credentials | `{"error": "forbidden"}` |
| 404 | Task not found or expired | `{"error": "not found"}` |
| 500 | Internal error | `{"error": "internal error"}` |

#### 200 Response Fields

**Pending:**

| Field | Type | Description |
|-------|------|-------------|
| status | string | `"pending"` — waiting for the user to send the verification email |

**Verified:**

| Field | Type | Description |
|-------|------|-------------|
| status | string | `"verified"` — email ownership confirmed |
| email | string | Verified sender's email address |

### Example

```bash
curl -X POST https://api.mailflow.lat/result \
  -H "Content-Type: application/json" \
  -d '{
    "task_id": "019f4326-9e8a-79f2-9221-99d5bfe25d06",
    "secret": "WkveK_yM2FN0rxOkFQwLdqC93bCBCjV1VHoVOtZOAS0",
    "signature": "d4568c5b552c5c56868f786d42a125637bda035fcc2aa05179111e7457339deb"
  }'
```

While pending:

```json
{"status": "pending"}
```

After verification:

```json
{"status": "verified", "email": "user@example.com"}
```

---

## DELETE /task

Delete a verification task. Requires the credentials returned at task creation.

### Request

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| task_id | string | Yes | Task ID |
| secret | string | Yes | Task secret |
| signature | string | Yes | Credential signature |

### Response

| Status | Meaning | Body |
|--------|---------|------|
| 200 | Deleted | `{"ok": true}` |
| 400 | Bad payload | `{"error": "bad payload"}` |
| 403 | Invalid credentials | `{"error": "forbidden"}` |
| 404 | Task not found or expired | `{"error": "not found"}` |
| 500 | Internal error | `{"error": "internal error"}` |

### Example

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

## Callback Notification

If a `callback` URL was provided at task creation, the service will send a POST request to that URL upon verification:

### Callback Request

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

### Callback Fields

| Field | Type | Description |
|-------|------|-------------|
| task_id | string | Task ID |
| status | string | `"verified"` |
| email | string | Verified sender's email |
| verified_at | integer | Verification time (Unix timestamp) |

### Callback Rules

- **HTTPS only** — HTTP URLs will be rejected
- **5-second request timeout** — if no response within 5 seconds, the callback is dropped
- **Callback failure does not affect verification** — the task is already verified; the callback is just a notification
- **No retries** — failed callbacks are not re-sent
- **Max URL length: 2048 characters**

---

## Error Codes

| HTTP Status | error | Trigger |
|-------------|-------|---------|
| 400 | `bad payload` | Malformed JSON or missing required fields |
| 403 | `forbidden` | Credential mismatch (task_id/secret/signature) |
| 404 | `not found` | Task does not exist or has expired |
| 429 | `rate limit exceeded` | Rate limit triggered (daily or burst) |
| 500 | `internal error` | Server error |
| 503 | `degraded` | Service unavailable (/health only) |

---

## Full Flow Example

```bash
# 1. Create verification task
RESPONSE=$(curl -s -X POST https://api.mailflow.lat/create \
  -H "Content-Type: application/json" \
  -d '{"callback": ""}')

# Parse response
TASK_ID=$(echo $RESPONSE | jq -r '.task_id')
SECRET=$(echo $RESPONSE | jq -r '.secret')
SIGNATURE=$(echo $RESPONSE | jq -r '.signature')
MAIL=$(echo $RESPONSE | jq -r '.mail')
TOKEN=$(echo $RESPONSE | jq -r '.token')

echo "Send email to: $MAIL"
echo "Body must include: $TOKEN"

# 2. Poll for result (after user sends email)
while true; do
  RESULT=$(curl -s -X POST https://api.mailflow.lat/result \
    -H "Content-Type: application/json" \
    -d "{\"task_id\":\"$TASK_ID\",\"secret\":\"$SECRET\",\"signature\":\"$SIGNATURE\"}")
  
  STATUS=$(echo $RESULT | jq -r '.status')
  
  if [ "$STATUS" = "verified" ]; then
    echo "Verified! Sender: $(echo $RESULT | jq -r '.email')"
    break
  elif [ "$STATUS" = "pending" ]; then
    echo "Waiting..."
    sleep 5
  else
    echo "Error: $RESULT"
    break
  fi
done

# 3. Delete task
curl -s -X DELETE https://api.mailflow.lat/task \
  -H "Content-Type: application/json" \
  -d "{\"task_id\":\"$TASK_ID\",\"secret\":\"$SECRET\",\"signature\":\"$SIGNATURE\"}"
```

---

## Notes

- **Token must appear in the email body** — this is the sole verification criterion; tokens in the subject line are ignored
- **Tasks expire after 24 hours** — expired tasks return 404
- **Each task can only be verified once** — even if multiple emails arrive, status only transitions from pending to verified once
- **Verified tasks can be queried multiple times** — /result always returns verified status until the task expires or is deleted
- **Keep all three credentials (task_id + secret + signature)** — missing any one prevents queries and deletion
- **Use the returned subject field** — not required, but helps users identify the email
- **Callback URLs must use HTTPS** — HTTP URLs will be rejected
- **Without a callback, start polling 3-10 seconds after the user sends the email** — email delivery typically takes several seconds; polling too early only returns pending

---

## Code Examples

### curl

```bash
# Create task
curl -s -X POST https://api.mailflow.lat/create \
  -H "Content-Type: application/json" \
  -d '{"callback": ""}'

# Query result
curl -s -X POST https://api.mailflow.lat/result \
  -H "Content-Type: application/json" \
  -d '{"task_id":"<task_id>","secret":"<secret>","signature":"<signature>"}'

# Delete task
curl -s -X DELETE https://api.mailflow.lat/task \
  -H "Content-Type: application/json" \
  -d '{"task_id":"<task_id>","secret":"<secret>","signature":"<signature>"}'
```

### Python

```python
import requests

BASE = "https://api.mailflow.lat"

# Create verification task
resp = requests.post(f"{BASE}/create", json={"callback": ""})
task = resp.json()
print(f"Recipient: {task['mail']}")
print(f"Body must include: {task['token']}")

# Query verification result
resp = requests.post(f"{BASE}/result", json={
    "task_id": task["task_id"],
    "secret": task["secret"],
    "signature": task["signature"],
})
result = resp.json()
print(f"Status: {result['status']}")
# When verified: result["email"] is the sender's address

# Delete task
requests.delete(f"{BASE}/task", json={
    "task_id": task["task_id"],
    "secret": task["secret"],
    "signature": task["signature"],
})
```

### Node.js

```javascript
const BASE = "https://api.mailflow.lat";

// Create verification task
const createResp = await fetch(`${BASE}/create`, {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({ callback: "" }),
});
const task = await createResp.json();
console.log(`Recipient: ${task.mail}`);
console.log(`Body must include: ${task.token}`);

// Query verification result
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
console.log(`Status: ${result.status}`);
// When verified: result.email is the sender's address

// Delete task
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
