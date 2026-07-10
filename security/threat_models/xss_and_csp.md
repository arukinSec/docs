# Cross-Site Scripting (XSS) and Content Security Policy (CSP)

Arukin handles sensitive Google account data. Preventing XSS and data exfiltration is critical. The primary defense is a Content Security Policy, but there are **two separate CSPs** — one for development and one for production — and they diverge.

## Development CSP (`frontend/index.html`)

Defined via `<meta http-equiv="Content-Security-Policy">`:

```
default-src 'self';
script-src 'self' blob: https://static.cloudflareinsights.com https://*.razorpay.com;
worker-src 'self' blob:;
style-src 'self' 'unsafe-inline';
img-src 'self' data: blob: https://cdn.simpleicons.org https://*.googleusercontent.com https://yt3.ggpht.com;
frame-src 'self' https://*.razorpay.com;
connect-src 'self' https://*.supabase.co https://oauth2.googleapis.com https://cloudflareinsights.com https://*.razorpay.com;
```

No `'unsafe-inline'` or `'unsafe-eval'` in `script-src` — inline scripts are blocked, which is the most important XSS mitigation.

## Production CSP (`frontend/public/_headers`)

Applied by Firebase Hosting via the `_headers` file:

```
default-src 'self';
script-src 'self' https://*.razorpay.com;
style-src 'self' 'unsafe-inline';
img-src 'self' data: https:;
connect-src 'self' https://*.supabase.co https://*.supabase.in https://*.googleapis.com wss://*.supabase.co https://*.razorpay.com;
frame-src 'self' https://*.razorpay.com;
font-src 'self' data:;
```

## Key Differences Between Environments

| Directive | `index.html` (dev) | `_headers` (production) | Impact |
|-----------|---------------------|-------------------------|--------|
| `script-src` | `blob:` + Cloudflare + Razorpay | Razorpay only | Production missing Cloudflare analytics and `blob:` — analytics won't fire, `blob:` scripts blocked |
| `worker-src` | `self` `blob:` | Not set | Production may block service workers |
| `img-src` | Allowlisted domains (SimpleIcons, Googleusercontent, yt3) | `data: https:` (broad) | Production allows any HTTPS image (tracking pixels, exfiltration) |
| `connect-src` | `oauth2.googleapis.com` (specific) | `*.googleapis.com` (broad) | Production allows all Google APIs; dev restricts to OAuth only |
| `font-src` | Not set | `self` `data:` | Production allows data: fonts |

## Threat Mitigations

- **Inline scripts blocked**: Both CSPs omit `'unsafe-inline'` in `script-src`, preventing injected `<script>` tags from executing.
- **Data exfiltration limited**: `connect-src` restricts `fetch()` destinations to allowed origins.
- **Clickjacking prevented**: `frame-src` restricts iframes to Razorpay only.

The CSP provides reasonable protection but would be stronger with a single source of truth — either generate both from a Vite plugin or always deploy via `_headers` and remove the `index.html` meta tag.
