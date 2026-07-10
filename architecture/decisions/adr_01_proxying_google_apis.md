# Architecture Decision Record: Proxying Google APIs

## 1. Title
Proxying Google APIs via Edge Functions

## 2. Status
Accepted

## 3. Context
Arukin requires access to members' Google accounts (Gmail, Drive, Contacts). The straightforward approach — passing Google OAuth tokens to the browser and making direct Google API calls — has several problems:

- **Token exposure**: Access tokens and refresh tokens in the browser are vulnerable to XSS and extension interception.
- **Token refresh**: Handling refresh cycles client-side would require the `client_secret`, which cannot be stored in the browser.
- **Authorization**: Client-side requests cannot be securely validated against the database — nothing prevents a manipulated client from accessing another manager's member data.
- **CORS**: Some Google APIs have restrictive CORS policies.

## 4. Decision
Route all Google API requests through the `google-proxy` Supabase Edge Function.

**Flow:**
1. Frontend calls `google-proxy` with the target `url`, `method`, `body`, and `memberId`.
2. The function verifies the caller's identity via Supabase JWT (`auth.getUser(jwt)`).
3. It fetches the manager record by email (`supabaseAdmin.from('managers').select('id').eq('email', user.email)`).
4. It fetches the member's `access_token` and `google_refresh_token` using the `service_role` key.
5. It verifies `member.manager_id === manager.id` — managers can only access their own members.
6. It checks token validity via Google's `tokeninfo` endpoint. If expired, it calls `refreshGoogleToken()` to refresh using the stored refresh token.
7. It proxies the actual Google API request with the bearer token.
8. The response is returned to the frontend wrapped in a uniform JSON envelope.

**Token storage:** Tokens are never sent to the frontend. A database trigger on `auth.identities` (`sync_identity_tokens_to_members`) captures tokens backend-side. Column-Level Privileges (`20260705000000_secure_tokens_clp.sql`) prevent client-side reads of token columns.

## 5. Consequences

### Positive
- **Tokens never reach the browser**: Google tokens stay server-side, only accessed by Edge Functions via `service_role`.
- **Centralized authorization**: Access control is enforced in one place, guaranteeing managers can only access their own members.
- **Simplified frontend**: No token refresh, no OAuth complexity in browser code.
- **CORS eliminated**: Server-to-server calls have no CORS restrictions.

### Negative
- **Latency**: Extra network hop (Client → Edge → Google → Edge → Client).
- **Payload limits**: Edge Functions have memory/time limits — proxying large Drive files or attachments could hit them.
- **Cost**: Each scan request invokes an Edge Function, which may increase hosting costs at scale.
