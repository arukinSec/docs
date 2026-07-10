# Razorpay Webhook & Idempotency

## Overview

The `razorpay-webhook` Edge Function (`supabase/functions/razorpay-webhook/index.ts`) fulfills purchases asynchronously. It listens for `order.paid` and `payment.captured` events, then upgrades the manager's tier or increments slots.

## Processing

1. **Signature verification**: HMAC-SHA256 of the raw body, compared via `crypto.timingSafeEqual` using the `RAZORPAY_WEBHOOK_SECRET`.
2. **Payload extraction**: Reads `orderId`, `manager_id`, `action`, and `email` from the `notes` object (injected during order creation).
3. **Fulfillment**:
   - `add-slot`: Calls `increment_manager_slots({ manager_uuid })` RPC.
   - Others: Sets `tier = 'PRO'`, calculates `pro_expires_at` (7 or 365 days), updates `managers` table.

## Current Gap: No Idempotency

Razorpay may deliver the same webhook event more than once due to network retries. The current implementation does not track processed events:

- **Double slot allocation**: Two deliveries of an `add-slot` webhook call `increment_manager_slots` twice, granting 2 slots for the price of 1.
- **Redundant upgrade**: Two deliveries of an upgrade event overwrite the same `pro_expires_at` — less critical but wasteful.

### Recommended Fix

Create a `processed_orders` table as a deduplication lock:

```sql
CREATE TABLE processed_orders (
  order_id TEXT PRIMARY KEY,
  processed_at TIMESTAMPTZ DEFAULT NOW()
);
```

At the start of the webhook handler, `INSERT INTO processed_orders (order_id)`. If the insert violates the primary key, return 200 OK immediately — the order was already processed.
