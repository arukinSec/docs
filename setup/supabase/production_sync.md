# Supabase Production Synchronization Guide

Managing the ArukinSec backend across multiple environments (Local, Staging, Production) requires strict adherence to schema versioning and secure secret management. This guide details how to sync your local database changes, edge functions, and secrets to the production Supabase project.

> [!WARNING]
> Never manually edit the production database schema via the Supabase Studio UI. All schema changes must be driven through SQL migrations stored in version control to prevent drift and ensure reproducibility.

## Linking the Local Project to Production

Before you can push changes, you must link your local Supabase CLI to your remote cloud project.

1. Obtain your remote **Project Reference ID**. You can find this in your Supabase Dashboard URL (e.g., `https://supabase.com/dashboard/project/<PROJECT_ID>`) or in the provided `secrets_supabase` file.
2. Run the link command and authenticate:

```bash
supabase link --project-ref <PROJECT_ID>
```

You will be prompted to enter your database password.

## Database Migrations (Schema Sync)

ArukinSec uses a migration-based workflow.

### Pulling Remote Changes

If changes were applied remotely (which should be avoided) or if you are onboarding and need to capture the current production state as a migration:

```bash
supabase db pull
```

### Creating a New Migration

When you make structural changes to your local database (e.g., adding a table for new billing logic):

1. Generate a new migration file:
   ```bash
   supabase migration new add_billing_table
   ```
2. Write your standard PostgreSQL commands and RLS policies in the generated `./supabase/migrations/<timestamp>_add_billing_table.sql` file.
3. Apply it locally to test: `supabase db reset`

### Pushing to Production

Once your local migrations are tested and committed to Git, deploy the schema changes to production:

```bash
supabase db push
```

## Deploying Edge Functions

Edge Functions (like `audit-gateway`) contain critical security logic for proxying Google API tokens. When you update the TypeScript/Deno code, you must deploy the functions to the edge network.

Deploy all functions:
```bash
supabase functions deploy
```

Deploy a specific function:
```bash
supabase functions deploy audit-gateway
```

> [!TIP]
> The edge function configuration (such as the Deno entrypoint and import maps) is defined in `supabase/config.toml` under the `[functions.audit-gateway]` block.

## Managing Production Secrets

Edge functions require sensitive environment variables (Google Client Secrets, Razorpay Webhook Secrets) that cannot be hardcoded or pushed via migrations.

To inject secrets from your local `.env` file into the production edge function runtime:

```bash
supabase secrets set --env-file .env
```

Alternatively, to set a single secret:
```bash
supabase secrets set MY_SECRET_KEY=value
```

To verify which secrets are currently active in production:
```bash
supabase secrets list
```

### Summary of Deployment Flow
1. **Develop Local**: `supabase start` & `supabase functions serve`
2. **Create Migration**: `supabase migration new ...`
3. **Link**: `supabase link --project-ref <ID>`
4. **Deploy Schema**: `supabase db push`
5. **Set Secrets**: `supabase secrets set --env-file .env`
6. **Deploy Functions**: `supabase functions deploy`
