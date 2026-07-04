# Frontend Local Environment Setup

Welcome to the **ArukinSec Frontend** local development guide. The frontend is an advanced, highly secure React 19 + Vite 8 Single Page Application (SPA) providing a "Trusted Guardian" dashboard.

This guide will walk you through setting up the local environment, explaining both the **How** and the **Why** behind our architecture.

> [!IMPORTANT]
> The ArukinSec frontend is designed with extreme security in mind. It **never** connects directly to Google APIs. All requests are routed through the backend Supabase Edge Functions proxy to prevent token leakage.

## Prerequisites

Ensure you have the following installed on your system:
- **Node.js** (v18 or higher recommended)
- **npm** (v9 or higher)

## Step-by-Step Setup

### 1. Install Dependencies

Navigate to the `frontend` directory and install the necessary npm packages:

```bash
cd frontend
npm install
```

This will install all required dependencies listed in `package.json`, including React, Vite, Tailwind CSS v4, Lucide React, and critical security libraries like `DOMPurify` (for sanitizing inputs against XSS) and `localforage`/`idb-keyval` (used alongside the Web Crypto API for secure AES-GCM client caching).

### 2. Configure Environment Variables

The frontend relies on specific environment variables that Vite injects into the build. We use a `.env.local` file for local development.

Copy the provided example file:

```bash
cp .env.example .env.local
```

Open `.env.local` and populate the following values:

```env
# Supabase Configuration
# Points to your local or remote Supabase instance
VITE_SUPABASE_URL=http://127.0.0.1:54321 # Use remote URL if not running Supabase locally
VITE_SUPABASE_ANON_KEY=your_supabase_anon_key

# Razorpay Configuration (Billing)
VITE_RAZORPAY_KEY_ID=rzp_test_your_razorpay_test_key # Use the test key for local dev
VITE_RAZORPAY_PLAN_ID=plan_your_razorpay_plan_id
```

> [!CAUTION]
> **Only** prefix variables with `VITE_` if they are safe to be exposed to the public dashboard. Never put secret keys (like Supabase Service Role keys or Google Client Secrets) in the frontend environment.

### 3. Start the Development Server

Start the Vite development server by running:

```bash
npm run dev
```

By default, the application will be served at `http://localhost:5173`.

### 4. Connect to the Backend Proxy

For the dashboard to function correctly (especially authentication and Google API data fetching), the Supabase Edge Functions must be accessible. 

If you are developing full-stack locally, ensure you have run `supabase start` and `supabase functions serve` in the `supabase` directory. Otherwise, ensure your `VITE_SUPABASE_URL` points to a live staging environment.

## Design Rationale: Why Vite & React 19?

- **Vite 8**: Provides ultra-fast Hot Module Replacement (HMR) and optimized build times, significantly improving developer experience over legacy bundlers like Webpack.
- **React 19**: We utilize the latest React features for concurrent rendering, which is crucial for maintaining a responsive UI when heavily processing and decrypting client-side AES-GCM data from IndexedDB.
