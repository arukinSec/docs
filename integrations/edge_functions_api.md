# Edge Functions API Reference

All Edge Functions are Deno TypeScript running on Supabase's edge network. They follow a consistent pattern: verify the caller via Supabase JWT, use `service_role` for DB access, enforce authorization, and return JSON.

## Shared Utilities

### `_shared/google-token.ts`

**`refreshGoogleToken(supabaseAdmin, memberId, refreshToken?)`**
- Reads `google_refresh_token` from `members` table if not provided.
- Calls `https://oauth2.googleapis.com/token` with `client_id`, `client_secret`, `grant_type=refresh_token`.
- Writes the new `access_token` to `members` table.
- Returns the new access token string.
- Throws on failure.

**`fetchWithTokenRefresh(supabaseAdmin, memberId, url, token, options?)`**
- Wraps `fetch()` with automatic 401 retry logic.
- On 401, refreshes token via `refreshGoogleToken` and retries the request.
- Returns the final `Response`.

---

## `google-proxy`

Proxies arbitrary Google API requests with token injection and auto-refresh.

**Endpoint:** `supabase.functions.invoke('google-proxy', { body })`

**Auth:** Bearer JWT in `Authorization` header.

**Request:**
```json
{
  "url": "https://gmail.googleapis.com/gmail/v1/users/me/messages?maxResults=10",
  "memberId": "uuid",
  "method": "GET",
  "body": {}
}
```

**Authorization check:** Verifies `member.manager_id === manager.id`.

**Response:**
```json
{
  "__proxy": true,
  "status": 200,
  "body": "<raw response body>",
  "ok": true
}
```

Errors return `{ "__proxy": true, "status": 500, "body": null, "ok": false, "error": "..." }` with HTTP 200 (application error, not transport error).

---

## `audit-gateway`

Gmail and Drive operations — email listing, attachment handling, file downloads. Tier enforcement for trial users.

**Endpoint:** `supabase.functions.invoke('audit-gateway', { body })`

**Auth:** Bearer JWT in `Authorization` header.

### Action: `fetch-emails`

Lists and fetches email messages with pagination.

```json
{
  "action": "fetch-emails",
  "memberId": "uuid",
  "plan": "free|pro",
  "params": {
    "pageToken": "",
    "activeFolder": "INBOX",
    "query": "",
    "showAdvancedSearch": false,
    "searchFilters": {}
  }
}
```

- Fetches 24 messages per page from Gmail API.
- Returns message details (from, subject, date, body, label).
- Trial members get body redacted for security/alert emails.
- Supports folder labels: `INBOX`, `FACEBOOK`, `INSTAGRAM`.

**Response:** `{ "messages": [...], "nextPageToken": "..." }`

### Action: `download-file`

Downloads a Drive file or exports a Google Workspace file (Docs, Sheets, Slides).

```json
{
  "action": "download-file",
  "memberId": "uuid",
  "plan": "free|pro",
  "params": {
    "fileId": "drive_file_id",
    "fileName": "report.pdf",
    "mimeType": "application/pdf"
  }
}
```

- Google Workspace files (Docs, Sheets, Slides) are auto-exported to Office Open XML or PDF.
- Blocked for trial users (returns 403).

**Response:** Binary file with `Content-Disposition: attachment`.

---

## `intel-gateway`

Intelligence scanning — social footprint detection, financial footprint detection, and Drive forensics with rate limiting and tier enforcement.

**Endpoint:** `supabase.functions.invoke('intel-gateway', { body })`

**Auth:** Bearer JWT in `Authorization` header.

### Action: standard scan

```json
{
  "scanType": "social|financial|drive",
  "query": "facebook.com",
  "memberId": "uuid",
  "deepScan": false,
  "platformId": "facebook"
}
```

**Rate limiting** (per `scan_executions`, monthly):
| Tier | SIMPLE/month | DEEP/month |
|------|-------------|------------|
| FREE | 2 | 5 |
| PRO/TRIAL | 5 | 10 |

**Tier enforcement:**
- `financial` scans require member tier PRO or TRIAL.
- `drive` scans require manager tier PRO.
- Deep scan details require manager tier PRO.

**Response (simple):**
```json
{
  "detected": true,
  "volume": 42,
  "details": "Detected artifacts in communication history."
}
```

**Response (deep, PRO):**
```json
{
  "detected": true,
  "volume": 42,
  "locked": false,
  "latestSubject": "Your account statement",
  "lastActive": "Mon, 01 Jul 2026",
  "targetAlias": "contact@example.com",
  "securityFlags": ["Recent security or login alerts detected", "Billing or subscription records found"],
  "devices": ["iPhone", "Chrome OS"],
  "locations": ["Chicago, IL"],
  "details": "Deep scan complete."
}
```

### Action: `get_usage`

Returns monthly usage stats for a member.

```json
{
  "action": "get_usage",
  "memberId": "uuid"
}
```

**Response:** `{ "usage": [{ "scan_depth", "platform_id", "member_id", "scan_category" }] }`

---

## `create-subscription`

Creates a Razorpay order for PRO upgrades, weekly licenses, and addon slots.

**Endpoint:** `supabase.functions.invoke('create-subscription', { body })`

**Auth:** Bearer JWT in `Authorization` header.

```json
{
  "manager_id": "uuid",
  "plan_id": "optional_razorpay_plan_id",
  "action": "upgrade|add-slot|weekly-license"
}
```

**Authorization check:** `manager.id !== manager_id` → throws.

**Price resolution:**
- `add-slot`: ₹1,200. Requires PRO tier with yearly billing.
- `weekly-license`: From `tiers.slot_price_weekly`.
- `upgrade`: From `tiers.slot_price_yearly`.

**Response:** Razorpay order object `{ id, amount, currency, notes, ... }`.

---

## `razorpay-webhook`

Fulfills purchases asynchronously. Called by Razorpay on `order.paid` / `payment.captured`.

**Auth:** HMAC-SHA256 signature in `x-razorpay-signature` header, verified against `RAZORPAY_WEBHOOK_SECRET`.

**Payload:** Standard Razorpay webhook body. Reads `notes.manager_id`, `notes.action`, `notes.email` from order metadata.

**Fulfillment:**
- `add-slot`: Calls `increment_manager_slots({ manager_uuid })` RPC.
- Others: Sets `tier = 'PRO'`, `pro_expires_at`, `billing_cycle` on `managers` table.

**Idempotency:** Not implemented. Duplicate deliveries may double-count addon slots.

---

## `refresh-google-token`

Explicitly refreshes a member's Google OAuth token on demand.

**Endpoint:** `supabase.functions.invoke('refresh-google-token', { body })`

**Auth:** Bearer JWT in `Authorization` header.

```json
{
  "member_id": "uuid"
}
```

**Authorization check:** `member.manager_id !== manager.id` → throws.

**Response:** `{ "success": true, "message": "Token refreshed securely." }`

---

## `expire-pro`

Cron-style function that downgrades expired PRO subscriptions to FREE. Not on a scheduled trigger — must be called externally (e.g., Supabase cron or external scheduler).

**Auth:** No user auth — uses `service_role` directly. Should be called as a scheduled function or admin endpoint.

**Process:**
1. Queries `managers` where `tier = 'PRO' AND pro_expires_at < NOW()`.
2. Processes in chunks of 50 to avoid URI overflow.
3. Updates manager: `tier = 'FREE', pro_expires_at = null, billing_cycle = null`.
4. Cascades: Updates all members of those managers to `tier = 'FREE'`.

**Response:** `{ "expired": 5 }`
