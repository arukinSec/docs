# Architecture Decision Record: Proxying Google APIs

## 1. Title
Proxying Google APIs via Edge Functions

## 2. Status
Accepted

## 3. Context
The ArukinSec application requires access to members' Google accounts (e.g., Gmail, Google Drive) on behalf of a "Manager" (the investigator/user of the app). 
Initially, one might consider passing the Google OAuth access token directly to the frontend client, allowing the browser to make direct HTTP calls to Google APIs. However, this approach presents significant security and architectural challenges:
- **Security Risks**: Exposing member access tokens and refresh tokens to the manager's browser is a high security risk.
- **Token Expiry**: Google access tokens expire frequently (typically 1 hour). Handling the refresh flow on the client side securely without exposing the `client_secret` is impossible.
- **Authorization**: We must guarantee that a manager can only access the Google data of members assigned directly to them. Client-side requests cannot be securely validated against our database schema.
- **CORS Issues**: Some Google APIs have strict CORS policies that make direct browser invocation complex.

## 4. Decision
We have decided to route all Google API requests through a dedicated Supabase Edge Function named `google-proxy`. 

**Implementation Details:**
1. The frontend invokes the `google-proxy` function, passing the target `url`, `method`, `body`, and the `memberId`.
2. The Edge Function verifies the Manager's identity using the Authorization header (Supabase JWT).
3. The function uses the `service_role` key to securely fetch the `access_token` and `google_refresh_token` for the specified `memberId`.
4. It strictly enforces authorization by verifying that the `member.manager_id` matches the authenticated Manager's `id`.
5. The function proactively checks the token's validity using `tokeninfo`. If expired, it invokes a shared utility (`refreshGoogleToken.ts`) to obtain a new access token using the refresh token, and updates the database.
6. The function makes the actual HTTP request to the Google API and proxies the response back to the client, wrapping it in a uniform JSON structure.

## 5. Consequences

### Positive
- **High Security**: Google access and refresh tokens never touch the manager's browser. They remain securely in the database and server-side memory.
- **Centralized Authorization**: Access control is rigidly enforced in the edge function, guaranteeing managers can only access their authorized members.
- **Simplified Client Logic**: The frontend doesn't need to handle token expiration, refresh cycles, or complex OAuth flows during scans.
- **CORS Mitigation**: Server-to-server calls bypass browser CORS restrictions.
- **Auditability**: It provides a centralized choke-point where we can easily implement request logging and rate limiting in the future.

### Negative
- **Latency**: Introduces an extra network hop (Client -> Supabase Edge -> Google API -> Supabase Edge -> Client), increasing overall latency for scan operations.
- **Payload Limits**: Supabase Edge Functions have memory and execution time limits. Proxying large binary payloads (e.g., massive email attachments or large Drive files) could hit timeouts or memory limits.
- **Cost**: High volume of API requests increases Edge Function invocations, which may impact hosting costs at scale.
