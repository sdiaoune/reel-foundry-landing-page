# Overview
ReelFoundry is an AI ad-creation platform that turns a product URL into multiple video ad variants, optimized for rapid iteration by beauty/skincare brands. This plan uses Next.js (App Router, TypeScript), Supabase, and Stripe to deliver a secure, multi-tenant SaaS with subscription billing and webhook-driven entitlements.

# Goal, scope, non-goals
**Goal**
- Ship a production-ready SaaS foundation for generating and managing AI ad batches from product URLs, with reliable billing and entitlements.

**Scope**
- Authentication, multi-tenant workspaces, ad batch creation, asset storage, billing/entitlements, and ops tooling.

**Non-goals**
- The ML/video rendering pipeline internals, ad delivery orchestration to Meta/TikTok, or a mobile client.

# High-level architecture diagram
```
[Browser]
   |  (URL intake, dashboard, results)
   v
[Next.js App Router]
   |  (Server Actions, Route Handlers)
   v
[Supabase: Auth + Postgres + Storage]  <──>  [Stripe Billing]
   ^                                         ^
   |                                         |
   └────────────── Webhooks (Stripe -> Next.js)┘
```

# Architecture Decisions
- **Next.js App Router** for server components, route handlers, and server actions to keep secrets server-side and minimize client data access. (References)
- **Supabase** as the system of record for product URLs, ad batches, assets, and team membership with Row Level Security enforcing tenant isolation. (References)
- **Stripe** as the system of record for billing state; Supabase stores Stripe IDs and entitlement snapshots for fast authorization checks. (References)
- **Webhook-driven entitlements** so billing status changes immediately and deterministically based on Stripe events.

# Frontend vs backend responsibilities
**Frontend (Client Components)**
- URL intake, batch status UI, asset previews, export flows, and basic validation.
- Enforce a **hard paywall** immediately after authentication by routing unpaid users to a paywall screen.

**Backend (Server Components/Actions + Route Handlers)**
- Create ad batch records, call internal AI pipeline, create Stripe Checkout/Portal sessions, and handle webhooks.
- Gate dashboard routes with subscription checks (server-side) so only active subscribers can access paid features.

# Data ownership
- **Supabase owns:** users, organizations, members, product URLs, ad batches, creatives, exports, entitlements.
- **Stripe owns:** products, prices, subscriptions, invoices, payment methods, and portal sessions.

# Entitlements / access source of truth
- **Source of truth:** Stripe subscription status and plan metadata.
- **In Supabase:** store `stripe_customer_id`, `stripe_subscription_id`, `status`, `plan`, `current_period_end`, and **usage counters** (ads generated per cycle).
- **Access checks:** RLS + server-side checks for plan limits (e.g., number of ads/month).

# AI/UGC Generation Services (Implementation Notes)
ReelFoundry’s core promise is realistic, high-quality UGC ads. Centralize AI provider integrations in a server-only “creative pipeline” module and keep all API keys private. Recommended approach:
- Use **OpenAI** (concepts, scripts, hooks, and scene breakdowns).
- Use **Nano Banana** (image/visual refinement or style matching, if applicable).
- Use **Replicate** or **Runway** for video synthesis / enhancement.
- Use **ElevenLabs** for voiceovers and realistic narration.

Create a single orchestrator (server action or route handler) that takes a `product_url`, generates a structured creative brief, and fans out to render variants. Store raw prompts, model metadata, and generation settings for reproducibility and QA.

## AI pipeline data flow (high-level)
1. Parse product URL → extract product claims/benefits/social proof.
2. Generate 10 hooks/scripts/scene lists.
3. Render visual variants (UGC-style shots, overlays).
4. Add VO + captions.
5. Store assets + metadata and mark `ad_variants` ready.

# Repository Setup (Next.js)
## Recommended project structure
```
/app
  /(marketing)
  /(auth)
  /(dashboard)
  api/
    stripe/webhook/route.ts
  layout.tsx
  page.tsx
/components
/lib
  supabase/
    client.ts
    server.ts
  stripe/
    stripe.ts
    webhook.ts
/db
  migrations/
  seed.ts
/types
```

## Environment variables (.env.example)
```
# Supabase
NEXT_PUBLIC_SUPABASE_URL=
NEXT_PUBLIC_SUPABASE_ANON_KEY=
SUPABASE_SERVICE_ROLE_KEY=

# AI + UGC services
OPENAI_API_KEY=
NANO_BANANA_API_KEY=
REPLICATE_API_TOKEN=
ELEVENLABS_API_KEY=
RUNWAYML_API_KEY=

# Stripe
STRIPE_SECRET_KEY=
STRIPE_WEBHOOK_SECRET=
NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY=
STRIPE_PORTAL_CONFIGURATION_ID=

# App
NEXT_PUBLIC_APP_URL=http://localhost:3000
```

## Linting/formatting
- ESLint (Next.js defaults) + Prettier.
- TypeScript `strict: true`.

## Server/client boundaries & security notes
- Stripe + Supabase service role keys **must stay server-side** (route handlers/server actions only).
- The client uses the Supabase **anon** key and session tokens.
- Use server components or middleware to enforce paywall redirects before rendering paid pages.

# Hard Paywall After Authentication
All authenticated users must be paid subscribers to access the app beyond a dedicated paywall screen.

## Paywall routing rules (App Router)
1. Auth success → user lands on `/app` (dashboard root).
2. If no active subscription, redirect to `/paywall`.
3. `/paywall` only allows: `Start trial`, `Upgrade`, or `Manage billing` actions.
4. `/app/*` routes hard-block if subscription is not `active` or `trialing`.

## Server-side enforcement (recommended)
- Use `middleware.ts` to read the user session, fetch subscription status from Supabase (server-side), and redirect to `/paywall` if not active.
- Additionally, in server components for dashboard routes, **double-check** entitlement status to prevent bypass.

### Example middleware logic (TypeScript)
```
// middleware.ts
import { NextResponse } from "next/server";
import type { NextRequest } from "next/server";

export async function middleware(req: NextRequest) {
  const url = req.nextUrl.clone();
  if (!url.pathname.startsWith("/app")) return NextResponse.next();

  // TODO: read session cookie and fetch subscription status via server-only Supabase client
  const isActiveSubscriber = false;

  if (!isActiveSubscriber) {
    url.pathname = "/paywall";
    return NextResponse.redirect(url);
  }

  return NextResponse.next();
}
```

# Brand Kit (from landing page)
Apply the landing page brand system to the app UI for visual consistency.

## Colors
- Background: `#0B0C10`
- Panel: `#11131A`
- Ink: `#F7F3EA`
- Muted: `#7D8790`
- Accent: `#FF4F00`
- Acid: `#B8FF2C`

## Typography
- Headlines: **Gloock**
- Body: **Recursive**
- Mono/utility: **Spline Sans Mono**

## UI tokens (example CSS)
```
:root {
  --bg: #0B0C10;
  --panel: #11131A;
  --ink: #F7F3EA;
  --muted: #7D8790;
  --accent: #FF4F00;
  --acid: #B8FF2C;
  --radius: 16px;
  --radius-sm: 10px;
}
```

## Components to theme
- Paywall screen, checkout buttons, and pricing cards.
- Dashboard shells (nav, cards, tables) should use `--panel` background with `--ink` text and `--muted` subtext.

# Supabase Setup
## Project initialization steps
**Local dev**
1. Create a Supabase project via MCP tool.
2. Copy project URL, anon key, and service role key into `.env.local`.

**Production**
1. Create a separate Supabase project.
2. Configure production auth + redirect URLs.

### MCP: Supabase project creation (example)
```
# Tool: supabase.createProject
{
  "name": "reel-foundry-dev",
  "region": "us-east-1"
}
```

## Auth configuration
- Enable email/password + magic link.
- Redirect URLs: `http://localhost:3000/auth/callback` and `https://<your-domain>/auth/callback`.

### MCP: Supabase auth config (example)
```
# Tool: supabase.updateAuthConfig
{
  "project_id": "<project_id>",
  "site_url": "http://localhost:3000",
  "redirect_urls": [
    "http://localhost:3000/auth/callback",
    "https://reelfoundry.com/auth/callback"
  ],
  "enable_magic_link": true,
  "enable_email_password": true
}
```

## Database schema
### Tables (core)
- `organizations` (id, name, owner_id)
- `organization_members` (org_id, user_id, role)
- `products` (id, org_id, product_url, brand_name, source)
- `ad_batches` (id, org_id, product_id, status, requested_ads, created_by)
- `ad_variants` (id, ad_batch_id, hook, script, aspect_ratio, status)
- `ad_assets` (id, ad_variant_id, storage_path, type)
- `subscriptions` (org_id, stripe_customer_id, stripe_subscription_id, status, plan, current_period_end, ads_generated)

### Migration example (SQL)
```
-- 001_init.sql
create table organizations (
  id uuid primary key default gen_random_uuid(),
  owner_id uuid references auth.users not null,
  name text not null,
  created_at timestamptz default now()
);

create table organization_members (
  org_id uuid references organizations on delete cascade,
  user_id uuid references auth.users on delete cascade,
  role text not null check (role in ('owner','admin','member')),
  created_at timestamptz default now(),
  primary key (org_id, user_id)
);

create table products (
  id uuid primary key default gen_random_uuid(),
  org_id uuid references organizations on delete cascade,
  product_url text not null,
  brand_name text,
  source text default 'url',
  created_at timestamptz default now()
);

create table ad_batches (
  id uuid primary key default gen_random_uuid(),
  org_id uuid references organizations on delete cascade,
  product_id uuid references products on delete cascade,
  status text not null default 'queued',
  requested_ads int not null default 10,
  created_by uuid references auth.users,
  created_at timestamptz default now()
);

create table ad_variants (
  id uuid primary key default gen_random_uuid(),
  ad_batch_id uuid references ad_batches on delete cascade,
  hook text,
  script text,
  aspect_ratio text default '9:16',
  status text not null default 'draft',
  created_at timestamptz default now()
);

create table ad_assets (
  id uuid primary key default gen_random_uuid(),
  ad_variant_id uuid references ad_variants on delete cascade,
  storage_path text not null,
  type text not null,
  created_at timestamptz default now()
);

create table subscriptions (
  org_id uuid primary key references organizations on delete cascade,
  stripe_customer_id text,
  stripe_subscription_id text,
  status text,
  plan text,
  current_period_end timestamptz,
  ads_generated int not null default 0
);
```

## Row Level Security (RLS)
Enable RLS on all tables and add explicit policies.

### RLS policies (example)
```
alter table organizations enable row level security;
alter table organization_members enable row level security;
alter table products enable row level security;
alter table ad_batches enable row level security;
alter table ad_variants enable row level security;
alter table ad_assets enable row level security;
alter table subscriptions enable row level security;

create policy "orgs:read" on organizations
for select using (
  exists (
    select 1 from organization_members m
    where m.org_id = organizations.id
      and m.user_id = auth.uid()
  )
);

create policy "products:read" on products
for select using (
  exists (
    select 1 from organization_members m
    where m.org_id = products.org_id
      and m.user_id = auth.uid()
  )
);

create policy "batches:insert" on ad_batches
for insert with check (
  exists (
    select 1 from organization_members m
    where m.org_id = ad_batches.org_id
      and m.user_id = auth.uid()
  )
);
```

## Storage buckets (optional)
- Create `ad-assets` bucket for generated videos and thumbnails.

### MCP: Create bucket (example)
```
# Tool: supabase.createStorageBucket
{
  "project_id": "<project_id>",
  "name": "ad-assets",
  "public": false
}
```

## Seeds/dev data strategy
- Create a seed script that inserts a sample org and a demo product URL for UI demos.

# Stripe Setup
## Products/prices strategy
- Tiers reflect ad volume caps: **Prototype**, **Operator**, **Foundry**.
- Create two prices per tier: monthly + annual.

### MCP: Create Stripe products/prices (example)
```
# Tool: stripe.createProduct
{
  "name": "ReelFoundry Operator",
  "metadata": {"plan": "operator", "ads_per_month": "150"}
}

# Tool: stripe.createPrice
{
  "product": "<product_id>",
  "unit_amount": 69900,
  "currency": "usd",
  "recurring": {"interval": "month"}
}
```

## Checkout/Billing flow design
- Use **Stripe Checkout** for initial purchase.
- Use **Customer Portal** for plan changes and cancellations.

## Webhook endpoint
- `POST /api/stripe/webhook` route handler.
- Verify signatures using `STRIPE_WEBHOOK_SECRET`.

### MCP: Create webhook endpoint (example)
```
# Tool: stripe.createWebhookEndpoint
{
  "url": "https://reelfoundry.com/api/stripe/webhook",
  "enabled_events": [
    "checkout.session.completed",
    "customer.subscription.created",
    "customer.subscription.updated",
    "customer.subscription.deleted",
    "invoice.payment_failed"
  ]
}
```

## Customer Portal configuration
- Configure portal settings and store `STRIPE_PORTAL_CONFIGURATION_ID`.

### MCP: Create portal configuration (example)
```
# Tool: stripe.createBillingPortalConfiguration
{
  "business_profile": {"headline": "Manage your ReelFoundry subscription"},
  "features": {
    "customer_update": {"allowed_updates": ["email", "address"]},
    "payment_method_update": {"enabled": true},
    "subscription_cancel": {"enabled": true},
    "subscription_update": {"enabled": true}
  }
}
```

## Tax/VAT considerations
- Use Stripe Tax if selling internationally; otherwise document tax assumptions.

# Payment + Entitlement Flow (Step-by-step)
## Client flow (UI)
1. User clicks “Upgrade.”
2. Client calls a server action to create a Checkout session.
3. User completes payment in Stripe Checkout.
4. User returns to `/billing/success` and sees updated plan after webhook sync.

## Server flow (route handlers/server actions)
1. Server action creates Stripe Checkout session with `customer`, `price`, and `metadata` (org_id).
2. Return `checkout_url` to the client.

## Webhook flow (signature verification, idempotency, retries)
1. Stripe posts to `/api/stripe/webhook`.
2. Verify signature with raw body.
3. Upsert `subscriptions` by `stripe_subscription_id`.
4. Reset `ads_generated` on billing period start.

## Data model for billing
- `subscriptions.org_id`
- `stripe_customer_id`
- `stripe_subscription_id`
- `status` (active, trialing, past_due, canceled)
- `plan` (prototype, operator, foundry)
- `current_period_end`
- `ads_generated` (usage counter)

## Handling upgrades/downgrades/cancellations/refunds
- Customer Portal for self-serve changes.
- Webhooks update `plan` and `status` in Supabase.
- On cancellation, enforce limits but keep access until period end.

# Implementation Plan (Milestones)
## Milestone 1: Scaffolding
- **Goal:** Next.js App Router setup with Supabase clients.
- **Files:** `app/`, `lib/supabase/*`, `.env.example`.
- **Acceptance:** Auth pages render; Supabase session works.

## Milestone 2: URL intake + ad batch model
- **Goal:** Create product URL intake and ad batch creation.
- **Files:** `app/(dashboard)/*`, `db/migrations/*`.
- **Acceptance:** User submits URL and sees queued batch.

## Milestone 3: Billing & entitlements
- **Goal:** Checkout + webhook updates to `subscriptions`.
- **Files:** `app/api/stripe/webhook/route.ts`, `lib/stripe/*`.
- **Acceptance:** Subscription state mirrors Stripe events.

## Milestone 4: Asset delivery
- **Goal:** Store ad assets in Supabase Storage and serve previews.
- **Files:** `app/(dashboard)/*`, `lib/supabase/*`.
- **Acceptance:** Members view and download ad assets.

# Security & Best Practices
- Never expose `SUPABASE_SERVICE_ROLE_KEY` or `STRIPE_SECRET_KEY`.
- Use raw body verification for Stripe webhooks. (References)
- Apply RLS to every table and test with `anon` + `authenticated` roles. (References)
- Use idempotent webhooks and store Stripe event IDs to prevent reprocessing.
- Store secrets in Vercel project settings; rotate on exposure.

# Testing Strategy
## Unit/integration
- Mock Stripe API for server action tests.
- Integration tests for checkout flow in Stripe test mode.

## Local webhook testing
```
stripe listen --forward-to localhost:3000/api/stripe/webhook
```

## RLS tests
- Automated SQL tests that run under `authenticated` and `anon` roles.

# Deployment Checklist
- Supabase migrations applied.
- Stripe webhook endpoint configured in production.
- Env vars set in Vercel.
- Smoke tests: signup, checkout, portal, URL intake.

# Common Failure Modes / Debug Playbook
## “Webhook not firing”
- Verify webhook URL and enabled events in Stripe.
- Confirm `STRIPE_WEBHOOK_SECRET` matches environment.
- Check Stripe dashboard event logs.

## “Subscription created but user not upgraded”
- Confirm webhook handler upserts subscription using `stripe_subscription_id`.
- Ensure metadata includes `org_id`.

## “RLS blocking inserts”
- Check `with check` clauses for `ad_batches` and `products`.
- Verify member exists in `organization_members`.

## “Duplicate webhook events”
- Store Stripe event ID and skip if processed.
- Use upserts with unique keys.

## “Auth redirect mismatch”
- Confirm Supabase redirect URLs include production domain.
- Ensure `NEXT_PUBLIC_APP_URL` matches deployment URL.

# References
- Next.js App Router: Route Handlers & Server Actions — https://nextjs.org/docs/app/building-your-application/routing/route-handlers
- Supabase Auth — https://supabase.com/docs/guides/auth
- Supabase RLS — https://supabase.com/docs/guides/auth/row-level-security
- Stripe Checkout — https://stripe.com/docs/checkout
- Stripe Billing — https://stripe.com/docs/billing
- Stripe Webhooks — https://stripe.com/docs/webhooks
- Stripe Customer Portal — https://stripe.com/docs/billing/subscriptions/customer-portal
