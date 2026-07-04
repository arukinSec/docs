# Frontend State Management & Caching

This document outlines how ArukinSec's React frontend manages state, particularly regarding sensitive member intelligence data, UI caching, and interactions with `localforage`.

## Overview

The ArukinSec frontend needs to maintain a complex state machine that includes:
- Global authentication state (Manager session).
- Contextual member data (which Member is currently being investigated).
- Large, deeply nested intelligence scan results (Social Footprints, Financial Footprints, Gmail contents).

To balance performance with strict security requirements, the application employs a hybrid approach using React Component State, React Context, and a cryptographically secure IndexedDB caching layer.

## Key Components

### 1. The Secure Caching Layer (`cache.js`)
At the core of the data persistence strategy is `src/utils/cache.js`. This utility provides an asynchronous, encrypted wrapper around `localforage` (which defaults to IndexedDB).

**Flow:**
- **Writing (`setEncryptedItem`)**: 
  - Retrieves the current Supabase session `access_token`.
  - Hashes the token using `SHA-256` to derive an `AES-GCM` key.
  - Serializes the state object to JSON.
  - Encrypts the data using a randomly generated 12-byte IV.
  - Stores the combined IV + ciphertext in `localforage`.
- **Reading (`getEncryptedItem`)**:
  - Retrieves the ciphertext from `localforage`.
  - Derives the key from the current session token.
  - Slices out the IV and decrypts the payload. If the token has changed or expired, decryption fails and the cache returns `null`.

> [!IMPORTANT]
> The encryption guarantees that sensitive footprint data is strictly tied to an active, valid session. Logging out or token expiration immediately renders the cached data unreadable.

### 2. UI Components & Local State (`GmailUI.jsx`)
Components like `GmailUI.jsx` represent the interactive views for investigation. 

**State Handling:**
- **Initialization**: Upon mounting (via `useEffect`), the component immediately checks the encrypted cache (`getEncryptedItem`) for pre-existing scans (e.g., `footprints_${member.id}` or `fin_scan_${member.id}`).
- **In-Memory State**: If cached data is successfully decrypted, it is loaded into React's local state (`useState`) to drive the UI instantly.
- **Pagination & Viewing**: Transient UI states, such as the current page of emails (`currentPage`), page tokens (`listNextPageToken`), and viewer modes (list vs. detail view), are maintained purely in memory via React state (`useState` and `useRef`). This ensures that navigating back and forth within a session feels instantaneous.

### 3. API Interactions & Proxies (`googleProxyFetch`)
When the UI needs fresh data (e.g., fetching a specific email thread or executing a new scan), it does not store Google API tokens locally. 
Instead, components invoke `googleProxyFetch`, a frontend utility that sends requests to the backend `google-proxy` Edge Function. The response is then parsed, displayed using local React state, and potentially cached to disk if it represents an expensive scan result.

## Design Rationale (The "Why")
- **Why not Redux/Zustand?** While a global state store could hold this data, refreshing the browser would clear it. We needed persistent storage.
- **Why not `localStorage`?** `localStorage` is synchronous (blocking the UI), has a ~5MB limit (insufficient for large footprint JSONs), and stores data in plain text (violating security requirements for PII).
- **Why encrypt locally?** Defense in depth. Even if an attacker gains physical access to the device or browser developer tools, they cannot read the cached scan results without the user's active, memory-resident Supabase authentication token.
