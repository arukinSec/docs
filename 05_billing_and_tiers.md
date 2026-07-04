# Billing & Tiers

Arukin utilizes a tiered subscription system to manage API costs, enforced through database triggers and backend RPC functions. Billing operations are securely processed via Razorpay integrations.

## Subscription Tiers

* **FREE Tier**:
 * Limit: 1 Member connection per Manager.
 * Quotas: 1 Insight scan and 2 Footprint scans per month.
 * Restrictions: Direct Google Drive file downloads are blocked. Sensitive keywords in Gmail are redacted.
* **TRIAL Tier**:
 * Similar to FREE tier but acts as an introductory state for self-audits or specific promotions.
* **PRO Tier**:
 * Limit: Up to 4 Member connections natively.
 * Quotas: 5 Insight scans and 10 Footprint scans per month.
 * Features: Unlocked `deepScan` capabilities for deep forensic extraction (target aliases, devices, billing history, etc.), unredacted emails, and direct file downloads.

## Add-Ons (Additional Slots)

Managers on the PRO plan can bypass the 4-member limit by purchasing additional slots (`additional_slots` column). When a Razorpay add-on payment succeeds, a webhook triggers the `increment_manager_slots` RPC function via the service role, permanently adding +1 capacity.

## Billing Automation (Webhooks & Cron)

* **`create-subscription` Edge Function**: Generates Razorpay checkout sessions for standard upgrades, weekly licenses, or add-ons. Applies Early Bird Promo pricing dynamically for the first 10 PRO users.
* **`razorpay-webhook` Edge Function**: Receives `order.paid` webhooks, verifies HMAC signatures, and officially upgrades a manager's database `tier` and `pro_expires_at` timestamp.
* **Database Triggers**: When a manager's tier is upgraded, a PostgreSQL trigger automatically cascades the `PRO` tier down to all their connected members instantly.
* **`expire-pro` Cron**: A continuous background cron job automatically monitors `pro_expires_at`. Once expired, it revokes PRO status, downgrading the manager and all associated members back to the FREE tier to enforce limits.
