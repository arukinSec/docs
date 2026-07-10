# Google Workspace OAuth Scopes

## Requested Scopes

The OAuth flow (both manager self-audit and client gateway connection) requests:

```
openid https://www.googleapis.com/auth/userinfo.profile https://www.googleapis.com/auth/userinfo.email https://www.googleapis.com/auth/contacts https://www.googleapis.com/auth/drive.readonly https://mail.google.com/
```

| Scope | Access | Usage |
|-------|--------|-------|
| `openid` | Read | OpenID Connect identity |
| `userinfo.profile` | Read | Name, avatar |
| `userinfo.email` | Read | Email (used as identifier) |
| `contacts` | Read/Write | Social footprint analysis |
| `drive.readonly` | Read-only | Drive forensics (public files, sensitive docs) |
| `mail.google.com` | Full Read/Write | Email analysis, financial scanners, remediation |

The `mail.google.com` scope is highly sensitive under Google's OAuth verification policies. Public distribution will require a CASA tier 2/3 security assessment to avoid the "Unverified App" screen.

## Token Flow

1. **Consent**: User authorizes via Supabase Auth with `access_type=offline prompt=consent` to guarantee a `refresh_token`.
2. **Capture**: Supabase Auth stores tokens in `auth.identities`. A database trigger (`sync_identity_tokens_to_members`) copies them to `public.members`.
3. **Storage**: Tokens are stored in `members.access_token` and `members.google_refresh_token` — CLP-protected, readable only by `service_role`.
4. **Usage**: The frontend never receives tokens. All Google API calls go through `google-proxy` Edge Function, which reads tokens server-side.
