# Google Workspace API Quotas

## Overview

ArukinSec relies heavily on Google Workspace APIs (Gmail, Drive, Contacts, and People API) to provide its security monitoring and forensics capabilities. Because all requests are proxied through the `google-proxy` Supabase Edge Function using individual user access tokens, the API quotas are generally bound to the **per-user** limits rather than exclusively to the global project limits, although both apply.

## Proxy Architecture

All API requests flow through:
`Frontend` ➡️ `google-proxy` (Edge Function) ➡️ `Google APIs`

The `google-proxy` handles:
- Fetching the correct `access_token` for the requested `memberId`.
- Testing token validity via the tokeninfo endpoint.
- Automatically refreshing the token using the `google_refresh_token` if expired.
- Forwarding the request and relaying the response.

## Expected Quota Limits (Standard Google Cloud Project)

While quotas can vary based on project verification status and billing, the standard limits to be aware of include:

### Gmail API
- **Per User Rate Limit:** 250 quota units per user per second.
- **Global Rate Limit:** 2,500,000,000 quota units per day.
- **Note:** Different operations consume different amounts of quota (e.g., reading a message might be 5 units, listing messages might be 1 unit, sending is 100 units).

### Drive API
- **Per User Rate Limit:** ~20,000 requests per 100 seconds per user.
- **Global Rate Limit:** 1,000,000,000 requests per day.

### People API (Contacts)
- **Per User Rate Limit:** ~30 requests per user per second.

## Handling Rate Limits (HTTP 429)

Currently, the `google-proxy` is a direct passthrough. It does not implement any intelligent queuing, backoff, or retry logic. 

> [!WARNING]
> If a rate limit is exceeded, Google APIs will return an `HTTP 429 Too Many Requests` status code. The `google-proxy` will forward this status directly to the frontend.

### Recommendations for Improvement

1. **Implement Exponential Backoff in Edge Function:** If the proxy receives a 429, it should wait and retry the request before failing back to the client.
2. **Batching:** Where possible, use Google API batching endpoints to group multiple operations (like fetching multiple emails or Drive file metadata) into a single HTTP request to reduce overhead and quota unit consumption.
3. **Caching:** Ensure that the frontend or an intermediary layer aggressively caches results (as seen with the local OSINT cache) to prevent redundant API calls for the same data.
