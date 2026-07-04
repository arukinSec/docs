# Razorpay Webhook & Idempotency

## Overview

The `razorpay-webhook` Supabase Edge Function is responsible for asynchronously fulfilling purchases made via Razorpay. It listens for events such as `order.paid` or `payment.captured` and updates the database accordingly (upgrading the manager's tier or adding slots).

## Payload Verification & Processing

1. **Signature Verification:** 
   The function extracts the `x-razorpay-signature` header and compares it against an HMAC SHA256 hash generated using the raw request body and the `RAZORPAY_WEBHOOK_SECRET`.
   
2. **Entity Extraction:**
   It parses the payload to find either an `order` or `payment` entity.
   It extracts the `orderId` and, critically, the metadata from the `notes` object (which was injected during the `create-subscription` step):
   - `notes.manager_id`
   - `notes.action`
   - `notes.email`

3. **Fulfillment Logic:**
   - **Add-on Slots (`action === 'add-slot'`):** Calls the Supabase RPC function `increment_manager_slots({ manager_uuid: managerId })`.
   - **Pro Upgrades:** Calculates a new `pro_expires_at` date (7 days for weekly, 365 days for yearly) and updates the `managers` table, setting `tier = 'PRO'` and storing the `razorpay_subscription_id`.

## Critical Issue: Lack of Idempotency

> [!CAUTION]
> The current implementation of the `razorpay-webhook` **lacks proper idempotency logic**. Razorpay (and most payment gateways) explicitly state that webhooks may be delivered more than once for the same event due to network retries or distributed system behavior.

### The Risks

1. **Double Slot Allocation:**
   If the webhook for an `add-slot` purchase is delivered twice, the script simply calls `increment_manager_slots` twice. This will result in the manager receiving 2 extra slots for the price of 1.
   
2. **Redundant Upgrade Processing:**
   If an upgrade webhook is delivered twice, it executes the `UPDATE` query on the `managers` table twice. While this is less critical than double slot allocation (as it simply overwrites the same tier and expiration date), it is still inefficient and bad practice.

### Recommended Fix

To achieve true idempotency, the system must track which orders have already been processed.

**Solution 1: Order Processing Table (Recommended)**
1. Create a table `processed_orders` with columns `order_id` (Primary Key, String) and `processed_at` (Timestamp).
2. At the start of the webhook function, attempt to `INSERT INTO processed_orders (order_id)`. 
3. If the insert fails due to a unique constraint violation, it means the order was already processed. The webhook should return a `200 OK` immediately without taking further action.

**Solution 2: RPC Modification**
Modify the `increment_manager_slots` RPC to accept the `order_id` and handle the idempotency internally within a Postgres transaction, checking a log table before incrementing the integer.
