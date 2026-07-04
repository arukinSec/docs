# Backend Architecture

The Arukin backend is built on Supabase, utilizing PostgreSQL, Row-Level Security (RLS), and Deno Edge Functions. It is housed in the `arukin-supabase` repository.

## Database Schema

The core relational model connects **Managers** with **Members**.

* **`managers`**: Stores profiles of admins/stewards logged into the dashboard. Contains fields like `tier` (FREE, PRO, TRIAL), `additional_slots`, and `pro_expires_at`.
* **`members`**: Stores the target users who have connected their Google accounts. Contains the sensitive `access_token` and `google_refresh_token` used to proxy Google API requests, along with a foreign key (`manager_id`) linking them to their manager. **Note:** Read access to these tokens is strictly denied to public roles via Column-Level Privileges (CLP).
* **`audit_logs`**: An immutable tracking table for security compliance and transparency. Tracks actions like `TRASH_EMAIL` or `DOWNLOAD_FILE`.
* **`usage_logs`**: Tracks API usage and scan frequencies for monthly rate limiting.
* **`app_config`**: Global configuration variables.

## RPC Functions

PostgreSQL Functions exposed via Supabase RPC for client-side operations:

* **`verify_manager_capacity`**: Validates if a manager can accept more members by checking `auth_id`. Limits FREE/TRIAL managers to 1 member, and PRO managers to `4 + additional_slots`.
* **`increment_manager_slots`**: Safely increments `additional_slots` by 1 via the service role upon successful Razorpay add-on payments.
* **`get_pro_manager_count`**: Returns the total number of PRO managers (used for billing logic).

## Triggers & Automation

Database triggers automate subscription tier cascading and security lifecycles:

* **`trg_extract_oauth_tokens`**: When a member signs in (AFTER INSERT/UPDATE on `auth.identities`), this securely extracts the raw Google OAuth tokens on the server-side and upserts them into the `members` table, ensuring the frontend never holds the tokens.
* **`trg_assign_member_tier`**: When a new member connects (BEFORE INSERT), it auto-sets their `tier` to match their manager's tier.
* **`trg_update_members_tier`**: When a manager upgrades or downgrades (AFTER UPDATE), the new tier is automatically cascaded down to all associated members.

## Edge Functions

Serverless Deno Edge Functions securely proxy API calls to hide tokens from the browser and handle webhooks:

1. **`audit-gateway`**: Proxies Gmail and Google Drive requests, enforcing strict plan restrictions (e.g., blocking file downloads on the FREE tier).
2. **`intel-gateway`**: Performs `insight_scan` or `social_scan` footprint intelligence, enforcing monthly usage limits.
3. **`google-proxy`**: A generic proxy that auto-refreshes OAuth tokens and securely forwards GET/POST requests to Google APIs.
4. **`refresh-google-token`**: Endpoint for managers to explicitly force-refresh a member's Google access token.
5. **`create-subscription`**: Generates Razorpay order IDs for upgrades, weekly licenses, or additional slots.
6. **`razorpay-webhook`**: Verifies HMAC signatures from Razorpay and updates the manager's tier to PRO or increments their slots.
7. **`expire-pro`**: A cron job that runs continuously, querying for expired `pro_expires_at` timestamps and resetting the manager and associated members back to the FREE tier.
