# Cross-Site Scripting (XSS) and Content Security Policy (CSP)

## Overview
As ArukinSec deals with highly sensitive oversight capabilities for Google accounts, preventing malicious scripts from executing in the context of the user's browser is paramount. The primary defense against Cross-Site Scripting (XSS) and data exfiltration is a strict Content Security Policy (CSP).

## Implemented Content Security Policy

The CSP is defined in `frontend/index.html` via an `<meta http-equiv="Content-Security-Policy">` tag.

```csp
default-src 'self';
script-src 'self' blob: https://static.cloudflareinsights.com https://*.razorpay.com;
worker-src 'self' blob:;
style-src 'self' 'unsafe-inline';
img-src 'self' data: blob: https://cdn.simpleicons.org https://*.googleusercontent.com https://yt3.ggpht.com;
frame-src 'self' https://*.razorpay.com;
connect-src 'self' https://*.supabase.co https://oauth2.googleapis.com https://cloudflareinsights.com https://*.razorpay.com;
```

## Threat Mitigations

### 1. Prevention of Unauthorized Script Execution
- **Directive**: `script-src 'self' blob: https://static.cloudflareinsights.com https://*.razorpay.com;`
- **Protection**: The `'unsafe-inline'` keyword is intentionally omitted. This prevents any inline `<script>` tags injected by an attacker (e.g., via a stored XSS payload in a username or profile field) from executing. 
- **Allowed Sources**: Only scripts originating from the same origin, Cloudflare Analytics, and Razorpay (for billing) are permitted to run.

### 2. Prevention of Data Exfiltration
- **Directive**: `connect-src 'self' https://*.supabase.co https://oauth2.googleapis.com https://cloudflareinsights.com https://*.razorpay.com;`
- **Protection**: Even if an attacker finds a way to execute JavaScript (e.g., via a compromised dependency), they cannot easily exfiltrate data to an attacker-controlled server. 
- **Allowed Sources**: `fetch()` or `XMLHttpRequest` calls are strictly limited to the ArukinSec backend (Supabase), Google OAuth endpoints, and specific integrated services.

### 3. Clickjacking and Malicious Iframing
- **Directive**: `frame-src 'self' https://*.razorpay.com;`
- **Protection**: Prevents ArukinSec from loading untrusted external iframes that could be used for phishing or malicious interactions.

### 4. Untrusted Asset Loading
- **Directive**: `img-src 'self' data: blob: https://cdn.simpleicons.org https://*.googleusercontent.com https://yt3.ggpht.com;`
- **Protection**: Restricts image loading to trusted sources (such as Google profile pictures and UI icons). This prevents attackers from loading tracking pixels or exploiting image parsing vulnerabilities.

## Development Considerations
- If new third-party integrations (like a new analytics provider or support widget) are added, the CSP in `index.html` must be explicitly updated to allow them.
- Avoid using inline styles where possible, although `style-src 'self' 'unsafe-inline'` is currently permitted for UI framework compatibility (e.g., React/Tailwind dynamic styles). Future iterations should aim to move towards strict CSS hashing if viable.
