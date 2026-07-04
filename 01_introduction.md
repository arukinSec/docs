# Arukin Documentation

Welcome to the Arukin documentation. Arukin is an advanced security monitoring and account management gateway designed for at-risk persons (such as vulnerable adults, the elderly, or targets of cyberstalking). It provides a "Trusted Guardian" console for managers to monitor Google Workspace environments securely.

## Core Philosophy

Arukin relies on a Zero-Install security model. Vulnerable users do not need to install mobile apps, browser extensions, or endpoint management software. Instead, it integrates directly with Google Cloud OAuth. Managers can remotely audit and steward Gmail, Google Drive, and Google Contacts data.

## Documentation Structure

* **[Frontend Architecture](02_frontend_architecture.md):** Details on the Vite/React stack, local state management, and the core Gateway components.
* **[Backend Architecture](03_backend_architecture.md):** Details on the Supabase database schema, Edge Functions, RPCs, and automated triggers.
* **[Security Model](04_security_model.md):** An overview of Arukin's strict defense-in-depth security approach, including IndexedDB AES-GCM caching, token isolation, and RLS.
* **[Billing & Tiers](05_billing_and_tiers.md):** Explanation of the FREE, TRIAL, and PRO subscription tiers and their enforced quotas.
