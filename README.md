# Foodie

Foodie keeps the original React + Vite + TypeScript customer experience and adds a Supabase-backed marketplace foundation. The current milestone supports restaurant/menu discovery, email/password accounts, saved addresses, customer order history, cash-on-delivery test orders, and a database-protected admin dashboard.

It is not a production-ready delivery network. COD orders are real database test records, but there is no restaurant acceptance workflow, dispatch, online payment, live tracking, or notification delivery yet. Seed restaurants and menu rows are examples marked `is_demo`; no sample orders or revenue are inserted.

## Run Locally

Prerequisites: Node.js 20+ and Docker Desktop running.

1. Install JavaScript dependencies once:

   ```sh
   npm install
   ```

2. Start the local Supabase services and reset the database from version-controlled migrations and the demo catalog seed:

   ```sh
   npx supabase start
   npx supabase db reset
   npx supabase status
   ```

3. Copy `.env.example` to `.env.local`. Use the local API URL and anon/publishable key shown by `supabase status`:

   ```dotenv
   VITE_SUPABASE_URL=http://127.0.0.1:54321
   VITE_SUPABASE_PUBLISHABLE_KEY=your-local-anon-or-publishable-key
   ```

4. Start Vite:

   ```sh
   npm run dev
   ```

5. Open the URL Vite prints, usually `http://localhost:5173/`. Local email confirmation messages are available in Supabase Inbucket at `http://127.0.0.1:54324`.

Without `.env.local`, the app shows a setup notice and does not substitute mock database responses.

## Use A Hosted Supabase Project

1. Create a Supabase project and choose a strong database password.
2. In Project Settings → API Keys, copy the Project URL and **publishable** key into `.env.local` using the variable names above. Never put a secret/service-role key in a `VITE_` variable.
3. Configure Authentication → URL Configuration with your Vite site URL and redirect URLs, including `http://localhost:5173` and `http://127.0.0.1:5173`. Keep email confirmation enabled and configure production URLs before deployment.
4. Apply migrations through the Supabase CLI:

   ```sh
   npx supabase login
   npx supabase link --project-ref YOUR_PROJECT_REF
   npx supabase db push
   ```

5. If you want the explicitly fictional restaurant/menu examples, run `supabase/seed.sql` once in the hosted project's SQL Editor. The seed contains no orders, customers, or financial metrics. Do not use those listings as real businesses.
6. Sign up through Foodie and confirm the email. To grant the first administrator, find that account's UUID in Authentication → Users, then run this trusted SQL in the Supabase SQL Editor:

   ```sql
   insert into public.admin_users (user_id)
   values ('PASTE_AUTH_USER_UUID_HERE')
   on conflict do nothing;
   ```

   Admin membership is stored outside editable user metadata. Browser clients have no read/write policy on `admin_users`; never grant admin by changing frontend state or user metadata.

## Database And Security

- `supabase/migrations/202610120001_initial_marketplace.sql` creates profiles, admin membership, restaurants, menu categories/items, addresses, orders, order items, and status history with indexes, constraints, RLS, and realtime publication for order status changes.
- `place_cod_order` accepts only restaurant/address/item IDs and quantities, checks ownership and availability, serializes duplicate submissions with an idempotency key, reads current prices in a transaction, and calculates tax/fees on the database. Client-submitted totals are never accepted.
- COD orders start `unpaid`. No client can update payment status or totals. Paid revenue excludes unpaid and cancelled orders.
- `transition_order_status` verifies admin membership and allowed state changes; `cancel_my_order` allows a customer to cancel only an order still in `placed` status.
- RLS restricts customer profiles, addresses, orders, and order items to their owner. Public catalog policies expose only published restaurants and available menu/category rows.
- Restaurant/menu create, update, and delete operations are limited to provisioned database admins. These are super-admin controls for this milestone, not a restaurant-partner portal.

## Penalty And Dispute Review Migration

`supabase/migrations/202610120003_fair_penalty_disputes.sql` is a forward-only migration prepared for review. It has **not** been applied to the configured Supabase project. It adds policy-version/disclosure records, order-linked cases, party response notes, admin-reviewed recovery decisions, appeals, and append-only-style audit events with RLS and server-checked RPCs. No policy rules or case data are seeded.

Important limits before applying it:

- The configured app currently supports unpaid Cash on Delivery only. Cashfree is not configured, and there is no verified payment/refund/settlement ledger. The migration does not charge a payment method or alter an order/payment/settlement balance.
- A complaint, cancellation, stock issue, or reported-loss amount never creates a recovery automatically. Admin recovery proposals require an enabled, disclosed rule, response/evidence notes, a reviewer-entered verified-loss value, and refunds/prior recoveries to offset it. A second human decision is required to record an approved amount. Approval is a case record only; collection is not implemented.
- Platform/payment-gateway failures are recorded as platform cases and cannot be mapped to a customer/restaurant recovery rule.
- File uploads and transactional notifications are not configured; the current UI stores text responses/evidence labels only.
- No response deadline, appeal deadline, cancellation fee, blanket no-refund term, penalty percentage, or rule version has been invented. Response/appeal deadlines need an approved policy configuration before they can be shown. Indian consumer-law/contract terms require qualified legal review and explicit publication/acceptance before any policy rule is enabled.

Review the complete SQL file before applying it. To test locally without touching a hosted database, start a disposable Supabase instance and reset it from the reviewed migrations; this resets that local instance only:

```sh
npx supabase start
npx supabase db reset
psql "postgresql://postgres:postgres@127.0.0.1:54322/postgres" -v ON_ERROR_STOP=1 -f supabase/tests/penalty_disputes_security.sql
```

For a hosted project, first back up the project, review the migration and legal policy text, then apply it manually in Supabase SQL Editor or through an explicitly approved migration workflow. Do not run a destructive reset against a hosted project. The regression script is rollback-only and exercises the authorization, recovery-offset, duplicate, and appeal paths without leaving test rows.

## Tests And Build

```sh
npm test
npm run build
```

The SQL security test is rollback-only and expects the migration and seed to be applied. With local Supabase running and `psql` available:

```sh
psql "postgresql://postgres:postgres@127.0.0.1:54322/postgres" -v ON_ERROR_STOP=1 -f supabase/tests/marketplace_security.sql
psql "postgresql://postgres:postgres@127.0.0.1:54322/postgres" -v ON_ERROR_STOP=1 -f supabase/tests/penalty_disputes_security.sql
```

The marketplace test checks public listing filters, cross-customer isolation, admin-role escalation denial, current-price COD totals, unpaid status, idempotency, and order status transitions. The dispute test checks stage-aware cancellation, complaint-without-penalty behavior, platform outage/stock cases, customer and restaurant isolation, recovery offsets and duplicate prevention, admin review, appeals, and that recovery decisions do not mutate payment state. These tests use disposable fixtures and roll back. They do not replace testing against a hosted project or legal review.

## Current Scope And Next Milestones

Implemented here: responsive web customer flow, database-backed catalog, email/password signup/login/logout, email confirmation and password reset flows, account/profile/address views, persistent browser cart, server-priced COD order creation, order history/confirmation/cancellation, RLS-protected admin dashboard, restaurant/menu CRUD, and admin order filtering/status updates.

Not implemented: Razorpay or refunds, restaurant partner accounts/onboarding/settlements, delivery partner accounts/dispatch/GPS tracking, Google Maps/geocoding, push/email/SMS notifications, coupons/reviews/loyalty, support/audit operations, and the dedicated Expo Android application. These require separate milestones and provider/configuration decisions; the UI does not claim them as connected.

For deployments, use HTTPS, configure real Auth redirect URLs and SMTP, apply migrations before releasing the client, provision admin users through a trusted process, add monitoring/backups/rate limits, and complete security/privacy review. Do not enable online payments until test transactions and signed webhook verification are implemented and verified.