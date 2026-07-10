# Supabase Local Environment Setup

## Prerequisites
- Docker Desktop (or OrbStack, Colima)
- Supabase CLI

## Setup

```bash
cd supabase
supabase start
```

This starts PostgreSQL, GoTrue (Auth), Realtime, Storage, and Studio in Docker.

### Local Endpoints
- API: `http://127.0.0.1:54321`
- Studio: `http://localhost:54323`
- DB: `localhost:54322`

### Edge Function Secrets

```bash
cp .env.example .env
```

Populate `.env`:
```env
GOOGLE_CLIENT_ID=your_client_id.apps.googleusercontent.com
GOOGLE_CLIENT_SECRET=your_client_secret
GOOGLE_REDIRECT_URI=http://127.0.0.1:54321/auth/v1/callback
RAZORPAY_WEBHOOK_SECRET=your_test_webhook_secret
```

### Serve Edge Functions

```bash
supabase functions serve --env-file .env
```

## Design Rationale

Using `supabase start` ensures parity between development and production. It applies all SQL migrations from `supabase/migrations/` and respects `config.toml` settings locally before they're deployed.
