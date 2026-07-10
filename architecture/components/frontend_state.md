# Frontend State Management & Caching

The Arukin frontend manages authentication state, member investigation data, and scan results using a hybrid of React component state, React Context, and an encrypted IndexedDB caching layer.

## Overview

State is split across three concerns:
- **Authentication**: Supabase session via `@supabase/supabase-js`.
- **Member context**: Current investigation target, loaded via React Context.
- **Scan results**: Large JSON payloads (social footprint, financial scan, Gmail contents) cached locally in encrypted IndexedDB.

## Encrypted Caching Layer (`cache.js`)

The file at `src/utils/cache.js` provides `getEncryptedItem`, `setEncryptedItem`, and `removeEncryptedItem` wrappers around `localforage` (backed by IndexedDB).

**Write flow:**
1. Derives an AES-GCM key by hashing the current Supabase session `access_token` with SHA-256.
2. Serializes the data to JSON.
3. Generates a random 12-byte IV via `crypto.getRandomValues`.
4. Encrypts with `crypto.subtle.encrypt({ name: 'AES-GCM', iv }, key, data)`.
5. Prepends the IV to the ciphertext and stores in `localforage`.

**Read flow:**
1. Retrieves the stored IV+ciphertext from `localforage`.
2. Derives the key from the current session token.
3. Slices the first 12 bytes as IV, decrypts with `crypto.subtle.decrypt`.
4. If the token has changed (new session, logged out), decryption fails and returns `null`.

The encryption binds cached data to an active session. Logging out or token expiry renders the cache unreadable.

## State Handling in Components

Components like `GmailUI.jsx`:
- On mount, check the encrypted cache for existing scan data (`footprints_${member.id}`, `fin_scan_${member.id}`).
- If cached data decrypts successfully, load it into React `useState` for instant display.
- Pagination and transient UI state (current page, list vs. detail view) stay in `useState` / `useRef`.

## API Proxy Pattern

The frontend never holds Google API tokens. UI components call `googleProxyFetch`, which invokes the `google-proxy` Edge Function. The function attaches the stored token server-side and proxies the response. No Google API keys exist in the browser context.
