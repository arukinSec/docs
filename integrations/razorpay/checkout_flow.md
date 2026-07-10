# Razorpay Checkout Flow

Managers upgrade their accounts and purchase additional slots via Razorpay. The flow spans the frontend, the `create-subscription` Edge Function, and the Razorpay Orders API.

## Sequence

1. Frontend sends `{ manager_id, action }` to `create-subscription` Edge Function.
2. Edge Function verifies the JWT, fetches the manager record by email, and validates `manager.id === manager_id` (server-side check).
3. It creates a Razorpay order via `POST /v1/orders`, embedding `manager_id`, `action`, and `email` in the `notes` object.
4. The order ID is returned to the frontend.
5. Frontend loads Razorpay Checkout JS and opens the payment modal.
6. On success, the frontend shows a toast and reloads — the webhook handles backend fulfillment asynchronously.

## Price Resolution

| Action | Amount | Prerequisite |
|--------|--------|-------------|
| `add-slot` | ₹1,200 (120000 paise) | Manager must be PRO with yearly billing |
| `weekly-license` | From `tiers.slot_price_weekly * 100` | — |
| `upgrade` | From `tiers.slot_price_yearly * 100` | — |

## Authorization

The `create-subscription` function (`supabase/functions/create-subscription/index.ts:49-52`) validates:
```
if (manager.id !== manager_id) throw new Error("Unauthorized to create subscription for this manager ID.")
```
Managers can only create subscriptions for themselves.
