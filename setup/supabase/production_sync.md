# Supabase Production Sync

All schema changes must go through SQL migrations. Do not edit the production database via Supabase Studio.

## Link

```bash
supabase link --project-ref <PROJECT_ID>
```

You'll be prompted for the database password.

## Migrations

**Create a new migration:**
```bash
supabase migration new my_change
```

**Apply locally:**
```bash
supabase db reset
```

**Push to production:**
```bash
supabase db push
```

## Edge Functions

```bash
# Deploy all functions
supabase functions deploy

# Deploy a specific function
supabase functions deploy google-proxy
```

## Secrets

```bash
# Set from .env file
supabase secrets set --env-file .env

# Set a single secret
supabase secrets set RAZORPAY_WEBHOOK_SECRET=value

# List current secrets
supabase secrets list
```

## Deployment Flow

1. Develop locally: `supabase start` + `supabase functions serve`
2. Create migration: `supabase migration new <name>`
3. Link to production: `supabase link --project-ref <ID>`
4. Deploy schema: `supabase db push`
5. Set secrets: `supabase secrets set --env-file .env`
6. Deploy functions: `supabase functions deploy`
