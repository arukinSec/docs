# Architecture Decision Record: Open Manager Self-Provisioning

## 1. Title
Open Self-Provisioning of Manager Accounts

## 2. Status
Accepted (by design)

## 3. Context

Arukin's access model: any Google-authenticated user who navigates to `/dashboard` with no existing manager record gets one auto-created with `tier: 'FREE'`. There is no invite gate, email whitelist, approval queue, or admin authorization.

This was flagged during security audits as a potential vulnerability (unauthorized account creation). However, it is an intentional product decision.

## 4. Decision

Keep open self-provisioning. Rationale:

### Product Rationale
- **Zero friction onboarding**: The product is a self-serve SaaS. Requiring an invite or admin approval before a user can even see the dashboard creates a barrier to conversion.
- **Google Auth as the gate**: Authentication via Google OAuth provides identity verification. The user's Google email is their identity — there is no anonymous access.
- **FREE tier is non-critical**: New accounts start on the FREE tier with limited capabilities (2 simple scans/month, no financial/drive intelligence, no file downloads). There's no data or functionality exposed that represents a security or business risk.
- **No sensitive data at registration**: Auto-provisioning creates a row in `managers` with only the user's email. No PII beyond what Google already provides, no billing info, no member connections.
- **Upgrade requires payment**: All meaningful functionality (PRO tier, additional slots) requires payment via Razorpay. Free accounts are intentionally constrained.

### Security Rationale
- **Supabase Auth handles authentication**: The app trusts Supabase's JWT verification. If a user has a valid Google session, they are authenticated. There is no anonymous or unauthenticated access to any endpoint.
- **All Edge Functions re-verify**: Every Edge Function independently verifies the caller's identity via `auth.getUser(jwt)` and checks `managers` table authorization server-side.
- **No privileged escalation path**: Auto-provisioning creates a FREE-tier account. There is no code path that allows a user to self-escalate to PRO or admin without payment.
- **Rate limiting is a separate concern**: If abuse becomes an issue (bulk account creation), rate limiting on the signup endpoint can be added independently without changing the provisioning model.

## 5. Consequences

### Positive
- **Low friction**: Users can try the product immediately after Google login.
- **Simple architecture**: No invite token generation, no email verification flow, no approval queue.
- **Auditable**: All auto-provisioning events are deterministic — if a user exists in `auth.users` but not in `managers`, they get one row. No ambiguity.

### Negative
- **No access control for registration**: Any Google user can create a Manager account. For enterprise deployments where only specific domains should have access, this model is insufficient.
- **Brute-force surface**: The 6-digit `auth_id` (used for member-to-manager linking) combined with open registration means any Manager could attempt to guess another Manager's linking code. Mitigation: the `auth_id` is a routing PIN, not an authentication credential.
- **Potential abuse**: Automated scripts could create many FREE accounts. Mitigation: Google OAuth requires a real Google account, and Supabase rate limits are available if needed.

## 6. Alternatives Considered

**Invite-only signup**: Rejected. Adds friction incompatible with a self-serve SaaS product. Could be added later as an optional enterprise feature.

**Email domain whitelist**: Rejected. The product isn't limited to a single organization. Domain whitelisting would exclude legitimate users.

**Admin approval queue**: Rejected. Introduces latency between signup and first use. Could be added for a future enterprise tier.
