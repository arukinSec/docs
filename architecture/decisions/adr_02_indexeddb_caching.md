# Architecture Decision Record: Client-Side Caching with IndexedDB and Encryption

## 1. Title
Client-Side Caching with IndexedDB and Encryption

## 2. Status
Accepted

## 3. Context
Scan results (footprint analysis, financial scans, email threads) are expensive to compute and large in size. Re-fetching them on every navigation or page reload degrades UX. Browser storage options are limited:

- **`localStorage`**: ~5MB limit, synchronous (blocks UI), plaintext storage.
- **IndexedDB**: Large capacity, async, but data is stored in plaintext by default.
- **Session-only**: Data lost on tab close, forcing re-fetch.

## 4. Decision
Cache scan results in IndexedDB via `localforage`, encrypted with AES-GCM using a key derived from the Supabase session token.

**Implementation** (`src/utils/cache.js`):
1. Key derivation: `SHA-256(session.access_token)` → AES-GCM key.
2. Encryption: Random 12-byte IV + `crypto.subtle.encrypt({ name: 'AES-GCM', iv }, key, data)`.
3. Storage: IV prepended to ciphertext, stored in `localforage`.
4. Decryption: Extract IV, re-derive key from current session, decrypt. On failure (wrong token, logged out), return `null`.

## 5. Consequences

### Positive
- **Encryption at rest**: Cached PII is unreadable without an active session. Logging out orphans the cache.
- **Large capacity**: IndexedDB handles JSON payloads well beyond the 5MB `localStorage` limit.
- **No extra dependencies**: Uses native Web Crypto API — no cryptographic library needed.

### Negative
- **CPU overhead**: AES-GCM encrypt/decrypt on every cache read/write. Acceptable for modern browsers.
- **Cache invalidation on token refresh**: When the JWT changes (token refresh), the derived key changes, and previously cached data becomes unreadable. This forces a re-fetch after re-login.
