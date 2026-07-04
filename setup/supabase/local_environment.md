# Supabase Local Environment Setup

Welcome to the **ArukinSec Backend** local development guide. Our backend is entirely driven by **Supabase**, utilizing PostgreSQL for secure data storage with Row-Level Security (RLS) and Deno Edge Functions for proxying sensitive Google OAuth API requests.

This guide explains how to spin up the entire backend stack locally using the Supabase CLI.

> [!IMPORTANT]
> ArukinSec relies heavily on database triggers, strict RLS policies, and secure edge functions. Running a local Supabase instance is the safest and most effective way to develop and test these mechanisms without impacting production data.

## Prerequisites

- **Docker Desktop** (or equivalent container runtime like OrbStack or Colima) must be installed and running.
- **Supabase CLI**: Follow the [official installation guide](https://supabase.com/docs/guides/cli/getting-started).

## Step-by-Step Setup

### 1. Initialize and Start the Local Instance

Navigate to the `supabase` directory. If the project isn't initialized, the CLI configuration already exists in `supabase/config.toml` (Project ID: `arukin`).

Start the Docker containers:

```bash
cd supabase
supabase start
```

This command pulls down the necessary Docker images (PostgreSQL, GoTrue for Auth, Realtime, Storage, and Studio) and starts them. The process may take a few minutes on the first run.

### 2. Accessing Local Services

Once started, the CLI will output several local endpoints. The most important ones are:
- **API URL**: `http://127.0.0.1:54321` (Use this for your frontend `VITE_SUPABASE_URL`)
- **Studio URL**: `http://localhost:54323` (Local web interface for managing the database, identical to the cloud dashboard)
- **Local DB Port**: `54322`

> [!TIP]
> Use the local Supabase Studio (`http://localhost:54323`) to easily browse tables, manage users, and inspect logs while developing.

### 3. Configuring Edge Function Secrets

ArukinSec heavily relies on Edge Functions (like the `audit-gateway`) to securely proxy requests to Google APIs. These functions require secrets that must not be committed to version control.

Copy the `.env.example` file to create your local `.env`:

```bash
cp .env.example .env
```

Populate `.env` with your development credentials:

```env
GOOGLE_CLIENT_ID=your_google_client_id.apps.googleusercontent.com
GOOGLE_CLIENT_SECRET=your_google_client_secret
GOOGLE_REDIRECT_URI=http://127.0.0.1:54321/auth/v1/callback # Note the local URL

RAZORPAY_WEBHOOK_SECRET=your_razorpay_test_webhook_secret
```

### 4. Running Edge Functions Locally

To test the edge functions locally, serve them using the Supabase CLI and pass your `.env` file so the Deno runtime can access the secrets:

```bash
supabase functions serve --env-file .env
```

With this running, the frontend can securely call your local edge functions, simulating the exact behavior of the production environment.

## Design Rationale: Why Local Supabase?

By utilizing `supabase start`, we ensure absolute parity between development and production. 
1. **Migrations & Seed Data**: The CLI automatically applies all SQL migrations in `supabase/migrations/` and injects mock data from `supabase/seed.sql`.
2. **Auth Webhooks**: We can test complex auth flows and JWT generation locally.
3. **Configuration as Code**: Settings in `supabase/config.toml` (such as Auth providers, rate limits, and custom JWT expiration) are instantly applied locally and can be systematically deployed.
