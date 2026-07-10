# Google Workspace API Quotas

All Google API requests are proxied through the `google-proxy` Edge Function using individual user access tokens. Quotas are bound to per-user limits in addition to global project limits.

## Proxy Architecture

`Frontend` → `google-proxy` (Edge Function) → `Google APIs`

1. Frontend calls `google-proxy` with `memberId` and target `url`.
2. The function fetches the member's `access_token` from the database via `service_role`.
3. It validates the token via `tokeninfo`, refreshing if needed.
4. It forwards the request with `Authorization: Bearer {token}` and relays the response.

## Quota Limits (Standard Google Cloud Project)

### Gmail API
- Per-user rate limit: 250 quota units/sec
- Global limit: 2,500,000,000 quota units/day
- Different operations consume different units (read: ~5, list: ~1).

### Drive API
- Per-user rate limit: ~20,000 requests/100 sec
- Global limit: 1,000,000,000 requests/day

### People API (Contacts)
- Per-user rate limit: ~30 requests/sec

## Rate Limiting

The `google-proxy` function does **not** implement retry or exponential backoff on HTTP 429 responses. Rate limit errors are forwarded directly to the frontend.

### Recommendations
1. Implement exponential backoff in `google-proxy`.
2. Batch multiple operations into single HTTP requests where possible.
3. Cache scan results aggressively (as done with the encrypted IndexedDB layer) to reduce redundant API calls.
