# Frontend Local Environment Setup

## Prerequisites
- Node.js v18+
- npm v9+

## Setup

```bash
cd frontend
npm install
cp .env.example .env.local
```

Populate `.env.local`:
```env
VITE_SUPABASE_URL=http://127.0.0.1:54321
VITE_SUPABASE_ANON_KEY=your_supabase_anon_key
VITE_RAZORPAY_KEY_ID=rzp_test_your_test_key
VITE_RAZORPAY_PLAN_ID=plan_your_plan_id
```

Only prefix variables with `VITE_` if they're safe for public exposure. Never put secret keys in frontend environment variables.

## Run

```bash
npm run dev
```

Served at `http://localhost:5173`.

For full-stack development, ensure Supabase is running locally (`supabase start`, `supabase functions serve` in the `supabase/` directory) so Edge Functions are accessible.

## Tech Stack
- **React 19**: Concurrent rendering for responsive UI during crypto operations.
- **Vite 8**: Fast HMR and optimized builds.
- **Tailwind CSS v4**: Utility-first styling.
- **DOMPurify**: XSS sanitization for email HTML.
- **localforage + Web Crypto API**: Encrypted IndexedDB caching.
