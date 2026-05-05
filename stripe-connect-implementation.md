Stripe Connect — KuotFlow Implementation Pack
This document contains Replit prompts and validation checklists used to correctly implement Stripe Connect (Standard accounts) for KuotFlow, a quotation‑to‑payment SaaS platform.

Key principle
KuotFlow orchestrates payments, but never touches funds.
Users are the merchants. Stripe handles compliance, KYC, payouts.


1️⃣ Replit Prompt — Stripe Connect Webhooks (Dedicated)
Use this prompt after OAuth + basic Connect integration is implemented.

📌 Prompt: Stripe Connect Webhook Implementation (KuotFlow)

Plain TextYou are implementing STRIPE WEBHOOKS for the SaaS platform **KuotFlow**.KuotFlow uses **Stripe Connect (STANDARD accounts)**.KuotFlow users connect their own Stripe accounts to receive paymentsfrom their customers. KuotFlow does NOT hold funds.This prompt is ONLY about WEBHOOK HANDLING.Do NOT re‑implement checkout, OAuth, or subscriptions.---✅ CORE REQUIREMENTS1. The webhook endpoint MUST support **Stripe Connect events**2. It MUST correctly identify:   - Platform events (KuotFlow SaaS subscriptions)   - Connected account events (user payments)3. It MUST be idempotent and safe for retries4. It MUST NOT assume events always arrive in order---✅ WEBHOOK EVENTS TO HANDLE (MINIMUM)CONNECTED ACCOUNT EVENTS (have `event.account`):- checkout.session.completed- payment_intent.succeeded- payment_intent.payment_failed- account.updatedPLATFORM EVENTS (NO connected account):- customer.subscription.updated- customer.subscription.deleted- invoice.payment_failed---✅ IDENTIFICATION LOGIC (MANDATORY)If `event.account` exists:→ This event belongs to a CONNECTED STRIPE ACCOUNT→ Use `event.account` to locate the KuotFlow userIf `event.account` is NULL:→ This event belongs to the PLATFORM Stripe account→ Used only for KuotFlow SaaS subscription logic---✅ DATABASE UPDATE RULESWhen payment succeeds:- Locate quote by Stripe reference- Mark quote as PAID- Create payment record- Store:  - stripe_payment_intent_id  - stripe_account_id  - statusWhen payment fails:- Do NOT delete quote- Mark quote/payment as FAILED- Allow retry if applicableWhen account.updated:- Update user.stripe_account_status- If restricted → disable paid quotations---✅ SECURITY REQUIREMENTS- Verify webhook signature using STRIPE_WEBHOOK_SECRET- Reject unsigned or invalid events- Never trust frontend payment confirmation- Webhook logic must be server‑only---✅ FINAL VALIDATIONImplementation is correct ONLY IF:- No customer payment is processed without a webhook- Events from multiple users’ Stripe accounts are isolated- Platform subscriptions and Connect payments never mix- Idempotent handling prevents duplicate recordsProceed step by step and explain assumptions.``Show more lines

2️⃣ Replit Prompt — Stripe Connect Failure & Recovery UX
This prompt ensures robust user experience when Stripe is disconnected, restricted, or incomplete.

📌 Prompt: Stripe Connect Failure‑Recovery UX (KuotFlow)

Plain TextYou are designing FAILURE‑RECOVERY UX for **Stripe Connect (Standard)** in KuotFlow.Stripe Connect status directly affects whether users can receive payments.Your task is to ensure the UI and backend respond correctly to ALL failure states.---✅ STATES TO HANDLE (NON‑NEGOTIABLE)1. Stripe NOT CONNECTED2. Stripe CONNECTED but details INCOMPLETE3. Stripe ACCOUNT RESTRICTED4. Stripe ACCOUNT DISCONNECTED / DEAUTHORIZED5. Stripe TEMPORARILY UNAVAILABLE (API error)---✅ UX RULES- Never silently fail- Always explain what action the user must take- Always provide a “Fix / Reconnect Stripe” CTA when possible- Paid quotation actions must be BLOCKED when Stripe is unusable---✅ REQUIRED UI STATESIF Stripe NOT CONNECTED:- Show CTA: “Connect your Stripe account to receive payments”- Disable ‘Send Paid Quotation’ buttonIF Stripe INCOMPLETE or RESTRICTED:- Show warning banner- Explain missing steps (verification, details, etc.)- Provide “Continue on Stripe” linkIF Stripe DISCONNECTED:- Force reconnection- Explain that KuotFlow cannot receive money on user’s behalfIF Stripe API ERROR:- Show graceful fallback message- Allow retry- Log error for support---✅ BACKEND ENFORCEMENT (CRITICAL)Frontend checks are NOT sufficient.Backend must:- Reject paid quotation creation if stripe_connected = false- Reject checkout creation if account_status != “enabled”- Log all Stripe connection failures---✅ SUCCESS CRITERIA- Users always understand why payment is blocked- No payment attempts occur on restricted accounts- Stripe issues do not corrupt business data- UX feels professional, not brokenDo NOT implement styling.Focus on logic, states, and messages.Show more lines

3️⃣ Stripe‑Safe Test Checklist (Pre‑Production & Production)
Use this checklist before enabling live mode.

✅ A. Stripe Connect Onboarding Tests

 User can connect an existing Stripe account
 User can create a new Stripe account via OAuth
 stripe_account_id stored correctly
 stripe_connected = true only after success
 Account status updates correctly from webhooks


✅ B. Restriction & Failure Tests

 Incomplete Stripe account blocks paid quotations
 Restricted account triggers warning UX
 Deauthorized account disables checkout immediately
 Stripe reconnect flow works
 No stale Stripe IDs remain active


✅ C. Payment Flow Tests (Connect)

 Payment created on connected account
 Payment appears in USER’s Stripe dashboard
 Platform Stripe account receives $0
 Quote marked PAID via webhook (not frontend)
 Duplicate webhooks do NOT duplicate records


✅ D. Separation of Concerns Tests

 KuotFlow SaaS subscription works independently
 Subscription failure blocks app access
 Subscription logic never uses Connect account
 Connect payment never affects KuotFlow billing


✅ E. Environment Safety

 Test keys used only in test mode
 Live keys used only in production
 Webhook secrets match environment
 No mixed test/live Stripe IDs


✅ F. Kill‑Switch Tests

 Disable Stripe Connect and confirm payments stop
 Webhook endpoint failure does not crash app
 Manual Stripe dashboard actions sync correctly


✅ Final Notes

Stripe Connect Standard keeps KuotFlow legally safe
No platform fees = simplest, lowest‑risk launch
This setup is future‑proof for bundles, fees, and marketplaces
Webhooks are the single source of truth


✅ Recommended File Placement
/docs
  └── stripe-connect-implementation.md

