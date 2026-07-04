# Google Workspace OAuth Scopes

## Overview

ArukinSec integrates deeply with Google Workspace to perform security auditing, forensics, and threat monitoring. During the onboarding process (in both the standard Manager self-audit flow and the Client Gateway connection flow), the system requests a specific set of OAuth 2.0 scopes from the user.

These scopes are requested with `access_type=offline` and `prompt=consent` to ensure a `refresh_token` is generated, allowing the ArukinSec platform to maintain continuous monitoring without requiring the user to re-authenticate constantly.

## Requested Scopes

The exact string of scopes requested during the OAuth flow is:

```text
openid https://www.googleapis.com/auth/userinfo.profile https://www.googleapis.com/auth/userinfo.email https://www.googleapis.com/auth/contacts https://www.googleapis.com/auth/drive.readonly https://mail.google.com/
```

### Breakdown and Rationale

| Scope | Access Level | Rationale & Usage |
| :--- | :--- | :--- |
| `openid` | Read | Standard OpenID Connect scope for identity verification. |
| `.../auth/userinfo.profile` | Read | Retrieves the user's basic profile information (e.g., name, avatar) to populate the Manager console and UI elements. |
| `.../auth/userinfo.email` | Read | Retrieves the primary email address. Used as a unique identifier for matching users to Managers and for database record keeping. |
| `.../auth/contacts` | Read / Write | Required for Contact/Target Monitor features. Allows the system to map communication networks, identify unusual external contacts, and flag potential social engineering threats. |
| `.../auth/drive.readonly` | Read-only | Enables Drive Forensics. Allows the platform to scan Google Drive for publicly shared files, sensitive documents (e.g., files containing API keys or PII), and unauthorized access without risking accidental modification or deletion of the user's data. |
| `https://mail.google.com/` | Full Read / Write | The most permissive scope. Required for deep email analysis, including the Financial Scanners, phishing detection, and potentially taking active remediation steps (like isolating or trashing malicious emails). |

> [!WARNING]
> The `https://mail.google.com/` scope is a highly restricted, sensitive scope under Google's OAuth API verification policies. If this application is published to a wider audience, it will require an extensive third-party security assessment (e.g., CASA tier 2/3) to avoid the "Unverified App" screen.

## Dataflow and Storage

1. **Consent Granted:** Upon successful authorization via Supabase Auth, the tokens (`access_token` and `refresh_token`) are captured.
2. **Database Storage:** The tokens are stored in the `members` table.
3. **Usage via Proxy:** The frontend never directly handles the `access_token` to make Google API calls. Instead, it sends a payload to the `google-proxy` Edge Function containing the `memberId` and the target `url`.
4. **Proxy Injection:** The `google-proxy` function retrieves the token from the database, validates it, refreshes it if necessary (via `refreshGoogleToken`), and injects it into the Authorization header before forwarding the request to Google's APIs. This prevents token leakage on the client side.
