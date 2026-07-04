# Security Model

Security is paramount in Arukin, particularly because it handles highly sensitive email and drive data for at-risk individuals. The application implements a defense-in-depth approach.

## 1. Token Isolation (API Proxying)

The frontend **never** directly communicates with Google APIs. Instead, it utilizes a wrapper that calls Supabase Edge Functions (e.g., `google-proxy`). These edge functions inject the appropriate OAuth access tokens on the server side. Sensitive Google OAuth tokens are completely hidden from the browser context, rendering XSS-based token theft impossible.

## 2. Row-Level Security (RLS)

The PostgreSQL database relies on strict RLS policies to enforce multi-tenant isolation:

* **Managers** can only read and update their own profile data, securely verified via `auth.jwt() ->> 'email'`.
* **Members** and their sensitive tokens are strictly locked down. Managers can only view and interact with members that share their `manager_id`.
* **Audit and Usage Logs** are similarly siloed, ensuring a manager can only ever view their own activity history.

## 3. Client-Side Encryption (IndexedDB)

To prevent hitting Google API rate limits, heavy payloads like Google Profiles and Footprints are cached locally in the browser's IndexedDB using `localforage`.
All cached data is encrypted at rest using **AES-GCM** via the Web Crypto API. The encryption key is derived dynamically via SHA-256 from the active Supabase session's `access_token`. If a local device is compromised, the data remains unreadable without an actively authenticated Arukin session.

## 4. DOM Sanitization & CSP

* **Strict Content-Security-Policy (CSP)**: The application strictly whitelists permitted domains for scripts, connections, and images (Supabase, Cloudflare, Google APIs, Razorpay). It intentionally blocks raw Google API domains to enforce the proxy route.
* **HTML Sanitization**: In `GmailUI.jsx`, email bodies are inherently untrusted. The application utilizes `DOMPurify` to strictly sanitize the HTML payloads before injecting them into the DOM, neutralizing malicious tracking pixels and XSS vectors within emails.
