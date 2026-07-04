# Token Lifecycle Threat Model

## Overview
ArukinSec heavily relies on Google OAuth tokens to manage and oversee user accounts. Protecting the lifecycle of these tokens—from issuance to storage to usage—is one of the most critical security requirements of the platform.

This threat model outlines the architecture implemented to securely capture, store, and utilize Google OAuth tokens without exposing them to the frontend client.

## Threat: Token Leakage via Client-Side Exposure
If access tokens or refresh tokens are sent to the client (browser), they are vulnerable to interception via Cross-Site Scripting (XSS), malicious browser extensions, or network sniffing (if TLS is stripped).

### Mitigation: Backend-Only Token Capture
ArukinSec ensures that the frontend application **never** receives, processes, or transmits Google OAuth tokens.

1. **Authentication Flow**: The frontend initiates the OAuth flow via Supabase Auth (`@supabase/supabase-js`).
2. **Provider Callback**: Google redirects back to the Supabase Auth server, providing the tokens directly to the backend.
3. **Database Storage**: Supabase Auth stores these tokens inside the strictly internal `auth.identities` table in the `identity_data` JSONB column.

## Threat: Database Exposure (RLS Bypass)
If a vulnerability in Row-Level Security (RLS) policies occurs, malicious users might query the `public.members` table to extract active access tokens of other users.

### Mitigation 1: The `sync_identity_tokens_to_members` Trigger
Instead of the frontend sending tokens to the `members` table via a PostgREST API call, a PostgreSQL trigger automatically syncs tokens directly within the database.

- **Migration**: `20260705010000_secure_tokens_trigger.sql`
- **Execution**: The trigger runs as `SECURITY DEFINER` (superuser).
- **Mechanism**: On `INSERT` or `UPDATE` to `auth.identities` where `provider = 'google'`, the trigger extracts `provider_token` and `provider_refresh_token` and securely upserts them into `public.members`.
- **Why?**: This completely removes the frontend from the token write path.

### Mitigation 2: Column-Level Privileges (CLP)
To ensure that tokens cannot be read via the Supabase API even if RLS is somehow bypassed or misconfigured, Column-Level Privileges are enforced.

- **Migration**: `20260705000000_secure_tokens_clp.sql`
- **Mechanism**: 
  - Complete `SELECT` access on `public.members` is **REVOKED** from the `anon`, `authenticated`, and `public` roles.
  - Granular `SELECT` access is **GRANTED** only for safe columns (e.g., `id`, `email`, `name`, `manager_id`).
  - The `access_token` and `google_refresh_token` columns are explicitly omitted from the `GRANT` statement.
- **Why?**: Even a perfectly crafted malicious GraphQL or PostgREST query authenticated as a valid user cannot retrieve the token columns, because PostgreSQL itself denies the read operation at the column level.

## Conclusion
By isolating the token write path to an internal database trigger and restricting the read path using native PostgreSQL Column-Level Privileges, ArukinSec achieves a robust defense-in-depth posture against token leakage.
