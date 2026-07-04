# Database Schema Architecture

ArukinSec utilizes PostgreSQL, hosted on Supabase, as its primary datastore. The architecture relies heavily on PostgreSQL's relational integrity and Supabase's Row Level Security (RLS) to enforce strict multitenancy and access controls.

## Overview

The schema is designed to model the relationship between investigators (Managers) and their targets (Members), while strictly tracking system usage, access grants, and audit trails.

## Core Entities

### 1. Managers (`public.managers`)
*(Note: Historically referred to as `auditors` in earlier migrations, renamed to `managers` in `20260704000010_rename_auditor_to_manager.sql`)*
- **Purpose**: Represents the end-users of ArukinSec who conduct investigations.
- **Key Columns**:
  - `id` (UUID): Primary key.
  - `email` (VARCHAR): The manager's email address.
  - `tier` (VARCHAR): Current subscription tier (`FREE`, `TRIAL`, `PRO`).
  - `additional_slots` (INT): Extra capacity for connecting members.
- **Integration**: Managers are tightly coupled to Supabase Auth (`auth.users`).

### 2. Members (`public.members`)
- **Purpose**: Represents the targets (the connected Google accounts) being investigated.
- **Key Columns**:
  - `id` (UUID): Primary key.
  - `manager_id` (UUID): Foreign key linking the member exclusively to a specific manager.
  - `email`, `name`, `avatar_url`: Profile data fetched from Google.
  - `provider_id` (TEXT): The Google unique ID.
  - `access_token` (TEXT) & `google_refresh_token` (TEXT): The OAuth credentials required for the edge functions to proxy API requests.
  - `tier` (VARCHAR): The capability tier granted to this member, dynamically assigned based on the manager's tier via database triggers.

### 3. Execution & Logging
- **`public.scan_executions`**: Tracks the asynchronous status (e.g., `pending`, `running`, `completed`) of various intelligence gathering tasks.
- **`public.audit_logs` & `public.usage_logs`**: Immutable ledgers that record significant actions (e.g., viewing an email, running a scan) for compliance and quota enforcement.

## Security & Row Level Security (RLS)

ArukinSec employs a "Defense in Depth" strategy. We do not rely solely on application logic (API endpoints) to prevent unauthorized access. PostgreSQL RLS policies enforce boundaries at the lowest level.

**Key RLS Policies:**
- **Manager Isolation**: `Managers can manage their own profile`
  - Policy: `USING (LOWER(email) = LOWER(auth.jwt() ->> 'email'))`
  - A manager can only read/update their own row in the `managers` table.
- **Member Segregation**: `Managers can view and manage their connected members`
  - Policy: `USING (manager_id IN (SELECT id FROM managers WHERE LOWER(email) = LOWER(auth.jwt() ->> 'email')))`
  - A manager can only query members where the `manager_id` foreign key points to their own ID.

> [!WARNING]
> Because RLS explicitly denies access by default, Edge Functions that require cross-tenant access (like webhooks) or background processing (like token refreshing) must use the `service_role` key to bypass RLS. These functions *must* manually re-implement authorization checks (e.g., verifying `member.manager_id == request.user.id`).

## Triggers & Functions (RPCs)

To maintain data integrity without requiring multiple round-trips from the client, we utilize PostgreSQL triggers and functions:
- **`assign_member_tier_on_connect`**: A trigger that runs `BEFORE INSERT` on the `members` table to automatically inherit the tier capabilities of the parent `manager`.
- **`update_members_tier_on_manager_upgrade`**: A trigger that runs `AFTER UPDATE OF tier` on `managers`, cascading subscription upgrades (or downgrades) to all associated members.
- **`verify_manager_capacity`**: A callable RPC function used during the OAuth connection flow to atomically check if a manager has reached their allowed quota of connected members before granting access.

## Design Rationale (The "Why")
- **Why store Google tokens in PostgreSQL instead of a separate vault?** 
  Storing tokens on the `members` row allows us to use RLS to ensure a manager can never even read the token of a member they don't own. The Edge Functions retrieve the tokens server-side, preventing them from ever leaking to the client browser.
- **Why rename Auditors to Managers?** 
  The term "Auditor" implied a passive, read-only compliance role. As the application evolved into a proactive investigation and intelligence tool, "Manager" more accurately reflects the user's relationship with the connected accounts.
