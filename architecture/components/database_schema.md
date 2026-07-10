# Database Schema Architecture

Arukin uses PostgreSQL on Supabase as its primary datastore. The architecture relies on relational integrity, Row-Level Security (RLS) for multitenancy, Column-Level Privileges (CLP) for sensitive columns, and database triggers for backend-only token handling.

## Current Tables

### `managers`
Represents the end-users of Arukin who conduct investigations.

| Column | Type | Notes |
|--------|------|-------|
| id | uuid | PK |
| email | text | Unique, linked to Google OAuth |
| role | text | Default 'manager' |
| tier | text | FREE, TRIAL, PRO, LOCKED |
| additional_slots | integer | Purchased overage slots |
| billing_cycle | text | yearly, weekly |
| pro_expires_at | timestamptz | PRO expiry |
| razorpay_customer_id | text | Razorpay reference |
| razorpay_subscription_id | text | Razorpay reference |
| current_active_connections | integer | Denormalized count |
| current_total_connections | integer | Denormalized count |

### `members`
Represents the Google accounts being monitored.

| Column | Type | Notes |
|--------|------|-------|
| id | uuid | PK |
| email | text | Member's email |
| name | text | Display name |
| avatar_url | text | Google avatar |
| provider_id | text | Google sub |
| manager_id | uuid | FK -> managers.id |
| access_token | text | Google OAuth access token — CLP-protected, service_role only |
| google_refresh_token | text | Google OAuth refresh token — CLP-protected, service_role only |
| tier | text | Capability tier, inherited from manager |
| slot_no | integer | Slot assignment |

### `scan_executions`
Tracks async scan task status.

| Column | Type |
|--------|------|
| id | uuid |
| manager_id | uuid |
| member_id | uuid |
| scan_category | text |
| scan_depth | text |
| status | text |

### `tiers`
Defines subscription tier capabilities.

| Column | Type |
|--------|------|
| id | uuid |
| name | text |
| base_max_active | integer |
| base_max_total | integer |
| slot_price_weekly | numeric |
| slot_price_yearly | numeric |

### `app_config`
Simple key-value configuration store.

| Column | Type |
|--------|------|
| key | text |
| value | text |

## Security

### Row-Level Security (RLS)
- **Manager isolation**: Each manager can only read/update their own row in `managers` (matched by `auth.jwt() ->> 'email'`).
- **Member segregation**: Managers can only query members where `manager_id` matches their own ID.

### Column-Level Privileges (CLP) — Token Protection
Migration `20260705000000_secure_tokens_clp.sql` revokes SELECT on all columns of `members` from `anon`/`authenticated`, then re-grants SELECT only on safe columns. **`access_token` and `google_refresh_token` are explicitly omitted** from the grant. This means:
- Client-side Supabase queries (`supabase.from('members').select(...)`) **cannot read token columns** even with a valid session.
- Only `service_role` (used by Edge Functions) can read tokens.
- This is enforced at the PostgreSQL column level, independent of RLS policies.

### Token Write Path — Database Trigger
Migration `20260705010000_secure_tokens_trigger.sql` creates a trigger `on_auth_identity_created_or_updated` on `auth.identities`. When a Google identity is created or updated, the trigger function `sync_identity_tokens_to_members()` extracts `provider_token` and `provider_refresh_token` from `identity_data` and upserts them into `public.members`. This runs as `SECURITY DEFINER`, bypassing RLS, so the frontend never needs to handle tokens.

### Edge Function Authorization
Edge Functions that need cross-tenant access (webhooks, token refresh) use the `service_role` key to bypass RLS. They manually verify authorization by checking `member.manager_id` against the authenticated manager's ID (`google-proxy/index.ts:53`).

## Triggers & Functions
- **`sync_identity_tokens_to_members`**: Trigger on `auth.identities` — syncs Google tokens to `members` table backend-side.
- **`assign_member_tier_on_connect`**: BEFORE INSERT trigger on `members` — inherits tier from parent manager.
- **`update_members_tier_on_manager_upgrade`**: AFTER UPDATE OF tier on `managers` — cascades tier changes to members.
- **`verify_manager_capacity`**: RPC function that checks manager slot limits during OAuth connection.

## Design Rationale
- **Tokens in PostgreSQL, not vault**: Storing tokens on the `members` row with CLP protection allows Edge Functions to read them via `service_role` while keeping them invisible to client queries. This avoids the complexity of a separate vault service.
- **Trigger-based token capture**: By syncing tokens through a DB trigger on `auth.identities`, the frontend is removed from the token write path entirely — tokens never transit the browser.
