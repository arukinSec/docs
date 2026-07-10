# Architecture Decision Record: Token Security — CLP + Database Trigger

## 1. Title
Securing Google OAuth Tokens via Column-Level Privileges and a Database Trigger

## 2. Status
Accepted (implemented in migrations `20260705000000_secure_tokens_clp.sql` and `20260705010000_secure_tokens_trigger.sql`)

## 3. Context

Google OAuth tokens (`access_token`, `google_refresh_token`) are the most sensitive data in the system. They grant full access to members' Gmail and Drive. Three threat vectors needed addressing:

**Vector A — Browser write path**: Initial implementation wrote tokens to the `members` table from the browser via `supabase.from('members').upsert({ access_token, google_refresh_token })`. This exposed tokens in the JS heap, network request bodies, and required RLS policies that allowed `anon` INSERT/UPDATE on token columns.

**Vector B — Browser read path**: RLS policies on `members` allowed authenticated managers to SELECT all columns, including token columns. Any manager with devtools could exfiltrate every connected member's tokens.

**Vector C — Plaintext storage**: Tokens are stored as plain `text` in the `members` table with no encryption at rest.

## 4. Decision

Apply two database-layer protections and remove the browser write path:

### 4.1 Database Trigger — Eliminate Browser Write Path (`20260705010000_secure_tokens_trigger.sql`)

A trigger `on_auth_identity_created_or_updated` on `auth.identities` runs as `SECURITY DEFINER` after INSERT or UPDATE of a Google identity. It extracts `provider_token` and `provider_refresh_token` from `identity_data` and upserts them into `public.members`.

**Why a trigger instead of an Edge Function?**
- Zero latency — runs inside PostgreSQL, no network hop.
- Atomic — token capture is transactional with the auth event.
- No additional infrastructure — no separate service to maintain.
- Supabase Auth already writes to `auth.identities` — we piggyback on that existing flow.

**Tradeoff:** `SECURITY DEFINER` functions execute with superuser privileges. A bug in the trigger could bypass RLS on `members`. The trigger logic is minimal (extract + upsert) to limit surface area.

### 4.2 Column-Level Privileges — Block Browser Read Path (`20260705000000_secure_tokens_clp.sql`)

Revoke SELECT on ALL columns of `public.members` from `anon`, `authenticated`, and `public`, then re-grant SELECT only on safe columns, **explicitly omitting** `access_token` and `google_refresh_token`.

**Why CLP instead of a separate tokens table?**
- A separate `member_tokens` table would require JOINs on every member query and a new set of RLS policies.
- CLP achieves the same result (tokens invisible to client queries) with a single migration and zero application changes.
- CLP is enforced at the PostgreSQL column level — no RLS policy can override it.

**Tradeoff:** `SELECT *` in Edge Functions (using `service_role`) still returns token columns. This is by design — Edge Functions need tokens. But it means any `SELECT *` in a `service_role` query includes tokens, so Edge Functions should be explicit about column selection where possible.

### 4.3 Frontend Write Path Removal

The frontend (`ClientGateway.jsx`) was updated to no longer extract or store `provider_token` / `provider_refresh_token` from the Supabase session. The token columns are absent from the upsert payload. This removes the browser entirely from the write path.

## 5. Consequences

### Positive
- **Tokens never written from browser**: The write path is exclusively server-side (DB trigger). Tokens never transit the network as request bodies from the browser.
- **Tokens never read by browser**: CLP prevents any client-side query from reading token columns, even with a valid session.
- **No application code changes needed for read protection**: CLP works at the database level — no frontend or Edge Function changes required.
- **Defense in depth**: Even if an RLS policy is misconfigured, CLP blocks column-level access.

### Negative
- **Plaintext in database**: Tokens remain unencrypted at rest. An attacker with direct database access (e.g., leaked `service_role` key) can read all tokens. Mitigation: `pgcrypto` + Vault for encryption at rest.
- **Trigger runs as SECURITY DEFINER**: A bug in the trigger function could bypass RLS. Mitigation: minimal trigger logic, code-reviewed.
- **Browser memory**: The `provider_token` still briefly exists in the JS heap after OAuth redirect (returned by `supabase.auth.getSession()`). This is inherent to Supabase's OAuth flow and cannot be eliminated without server-side-only auth handling.
- **Existing tokens**: Tokens stored in the DB before the CLP migration was applied remain readable by the `service_role` but are invisible to client queries.
