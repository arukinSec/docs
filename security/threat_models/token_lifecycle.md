# Token Lifecycle Threat Model

Google OAuth tokens are the most sensitive data in the system. They grant access to members' Gmail, Drive, and Contacts. The architecture secures them through three layers: a database trigger for capture, Column-Level Privileges for storage, and an Edge Function proxy for usage.

## Threat: Token Leakage via Client-Side Exposure

If tokens reach the browser, they're vulnerable to XSS, extensions, and devtools inspection.

### Mitigation 1: Trigger-Based Token Capture

When a user completes Google OAuth via Supabase Auth, the `provider_token` and `provider_refresh_token` are stored in `auth.identities`. A database trigger (`20260705010000_secure_tokens_trigger.sql`) on `auth.identities` runs as `SECURITY DEFINER` and upserts the tokens into `public.members`. The frontend never writes tokens to the database.

**Note on browser memory**: The `provider_token` is still returned to the browser in memory by `supabase.auth.getSession()` (inherent to the Supabase OAuth flow). This token lives in the JavaScript heap for the session duration. It is not persisted to the database by frontend code (the write path was removed from `ClientGateway.jsx`), but it is technically present in browser memory. Mitigations:
- The token is never stored in `localStorage`, IndexedDB, or any persistence layer.
- The encrypted IndexedDB cache uses a key derived from this token (SHA-256), but the cache stores scan results, not the token itself.
- CSP's `connect-src` restricts where the token could be exfiltrated.

### Mitigation 2: Column-Level Privileges

Migration `20260705000000_secure_tokens_clp.sql` revokes SELECT on `members` from `anon`/`authenticated` and re-grants it only on columns that exclude `access_token` and `google_refresh_token`. Client-side Supabase queries cannot read token columns.

### Mitigation 3: Service-Role Proxy for Usage

All Google API requests go through the `google-proxy` Edge Function. It reads tokens via `supabaseAdmin` (service_role), validates them, refreshes if needed, and proxies the request. The browser never receives tokens in API responses.

## Threat: RLS Bypass via Compromised Session

If a manager's session is compromised, an attacker could query the `members` table via the Supabase API.

CLP prevents token column reads even with a valid session. The attacker would only see safe columns (name, email, avatar, tier, etc.) — not the tokens themselves.

## Threat: Edge Function Compromise

All token access converges on the `service_role` key used by Edge Functions. A vulnerability in any one function (e.g., unsanitized input that leaks the database response in an error message) could expose all tokens.

**Mitigation**: Each function re-verifies authorization by checking `member.manager_id` against the authenticated manager's ID before using tokens. However, this relies on correct implementation in every function.

## Remaining Risks

- **Plaintext storage**: Tokens are not encrypted at rest in the database. `pgcrypto` encryption with a Vault-stored key would provide defense-in-depth.
- **Single point of failure**: The `service_role` key, if leaked, grants full token access.
- **Browser memory**: The `provider_token` exists in the JS heap briefly after OAuth. Supabase controls this — not application code.
