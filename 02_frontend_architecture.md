# Frontend Architecture

The Arukin frontend is built on a modern, high-performance React stack. It is housed in the `arukin-frontend` repository.

## Tech Stack Overview

* **Frameworks & Build Tools**: React 19, React Router v7, and Vite 8 for optimized builds and fast Hot Module Replacement (HMR).
* **Styling**: Tailwind CSS v4.
* **Authentication & Database**: Supabase JS Client (`@supabase/supabase-js`) handles auth, DB queries, and Edge Function invocation.
* **Data Visualization**: Recharts (for charts) and Lucide React (for iconography).
* **Storage & Security**: `localforage` for robust IndexedDB interactions, and `DOMPurify` for HTML sanitization.
* **Linting**: Oxlint, a highly performant Rust-based linter.

## Architecture & State Management

The frontend architecture prioritizes security, secure API isolation, and rapid data access through local caching.

### State Flow & Context
* **Global Auth**: `App.jsx` listens to Supabase `onAuthStateChange`. It actively manages the manager's session, checking database tier statuses and persisting session metadata to `localStorage`.
* **Custom Hooks**: The `useManager.js` hook encapsulates data fetching logic for the manager's profile, providing a clean `{ manager, loading }` interface.
* **Toast System**: A lightweight global notification system is implemented using native DOM CustomEvents (`window.dispatchEvent(new CustomEvent('arukin-toast'))`).

### Routing
Routing is handled by `react-router-dom`:
* **Public Routes**: `/`, `/about`, `/pricing`, `/how-it-works`, `/use-cases`
* **Manager Entry**: `/manager` (`ManagerGateway.jsx`) serves as authentication and onboarding for account managers.
* **Client Portal**: `/client` (`ClientGateway.jsx`) is the portal where family members/clients explicitly grant Google account access.
* **Dashboards**: `/dashboard` (`MembersList.jsx`) displays all connected clients, and `/member/:id` (`MemberDashboard.jsx`) is the root container for a specific client's data.

## Core Components (Gateways)

Arukin modularizes its data viewers into comprehensive "Gateways":

* **`GmailUI.jsx`**: A fully functional email inbox viewer with pagination, search, and a **Footprint Scanner** to map connected Social Media and Financial accounts based on sender domains.
* **`DriveUI.jsx`**: A Google Drive explorer rendering files using exact Google Drive paradigms. Includes a `FileViewerModal` capable of previewing images, PDFs, and native Google Workspace documents.
* **`ContactsUI.jsx`**: Retrieves and lists Google Contacts, segmenting them into individuals and associated organizations.
* **`ProfileUI.jsx`**: The command center for an individual account, utilizing Recharts to display storage distribution and embedding modular scanners (`SocialScanner`, `FinancialScanner`).
