# Architecture Decision Record: Client-Side Caching with IndexedDB and Encryption

## 1. Title
Client-Side Caching with IndexedDB and Encryption

## 2. Status
Accepted

## 3. Context
The ArukinSec frontend (e.g., `GmailUI.jsx`) processes and displays sensitive intelligence scans, such as a member's social footprint or financial integrations.
- **Performance**: Re-fetching or re-computing these scans on every page reload or navigation is computationally expensive and slow, resulting in a poor user experience.
- **Storage Limits**: `localStorage` is synchronous, blocks the main thread, and is strictly limited to ~5MB, which is insufficient for large scan payloads.
- **Security**: Storing PII (Personally Identifiable Information) like email addresses, social accounts, and banking footprints in plaintext within the browser's persistent storage (like IndexedDB) is a significant security risk. If the user's device is compromised or accessed while logged out, the data remains readable.

## 4. Decision
We have implemented a secure client-side caching layer using `localforage` (backed by IndexedDB) combined with the Web Crypto API.

**Implementation Details:**
1. **Storage Engine**: We use `localforage` to asynchronously store large data blobs in IndexedDB without blocking the UI thread.
2. **Encryption Mechanism**: Before any scan data (e.g., `footprints_${member.id}`, `fin_scan_${member.id}`) is written to disk, it is serialized to JSON and encrypted using `AES-GCM` via the browser's native Web Crypto API.
3. **Key Derivation**: The AES encryption key is dynamically derived from the authenticated manager's Supabase session `access_token` using a `SHA-256` digest (`cache.js`).
4. **Initialization Vector (IV)**: A randomized 12-byte IV is generated for every write operation, ensuring cryptographic security. The IV is prepended to the encrypted payload before storage.
5. **Decryption on Read**: When reading, the system extracts the IV, derives the key from the current session token, and decrypts the payload. If the decryption fails (e.g., wrong token, logged out), it gracefully falls back to a cache miss (`null`).

## 5. Consequences

### Positive
- **Security at Rest**: Cached intelligence data is unreadable unless the manager has an active, valid Supabase session. If they log out, the cache becomes cryptographically inaccessible.
- **Performance**: Significant reduction in API calls and perceived latency upon page navigation and reloads.
- **Storage Capacity**: Utilizing IndexedDB allows us to comfortably store large JSON structures that would otherwise exceed `localStorage` limits.
- **Native APIs**: Relying on the Web Crypto API avoids the need for heavy third-party cryptographic libraries, reducing the bundle size.

### Negative
- **CPU Overhead**: AES-GCM encryption and decryption add slight computational overhead during read/write operations, though modern browsers handle Web Crypto exceptionally well.
- **Cache Invalidation**: Because the encryption key is tied to the JWT `access_token`, refreshing the token or logging out inherently orphans or invalidates previously cached data. While good for security, it forces a fresh fetch upon re-login.
