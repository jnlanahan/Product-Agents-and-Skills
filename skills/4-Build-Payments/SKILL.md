---
name: add-payment
description: MUST BE USED when the user wants to add or extend payments. Stripe-first; detects existing Stripe setup and extends, or detects a different processor (Paddle, Lemon Squeezy) and adapts. Always handles webhook signature verification, idempotency, and Customer Portal. Trigger on `/add-payment`, "add subscription", "add billing", "add Stripe".
---

# /add-payment

You add or extend payment functionality. Stripe-only — if the project uses a different processor (Paddle, Lemon Squeezy), surface that and ask whether to proceed with the existing processor or migrate.

## Pre-flight

- Read `.claude/progress.md` (last 5 entries) and `.claude/context.md` if present
- Call `project-state-detector`; if mode is off-pattern for this skill, surface a one-line warning (do NOT block)

## Post-flight

- Append to `.claude/progress.md`: timestamp, skill name, output path, key decisions, suggested next step
- If `.claude/progress.md` is missing, create it with a header first

## Critical

- Always start with Stripe test-mode keys (`sk_test_...`). Never wire live keys (`sk_live_...`) until the full flow is verified in test mode.
- Webhook signature verification is non-negotiable — never skip it, even for a quick demo.
- Do not handle raw card data in application code — all card collection must go through Stripe Elements or Stripe Checkout.

## Procedure

### Step 1: Detect

Run in parallel:
- `stack-detector` — confirm framework, see if Stripe is already in package.json
- `codebase-classifier` — know whether to expect clean patterns or improvise
- `pattern-finder` — "Find existing API routes that handle external service callbacks (e.g., webhooks). Also find route auth pattern."

Read `_stack-preferences.md` and `_adaptation-playbook.md`.

### Step 2: Determine mode

Three cases:

**Case A: No payments yet (greenfield-ish)**
- Install `stripe`, `@stripe/stripe-js` (and `@stripe/react-stripe-js` if Vite/CRA project; not needed for Next.js)
- Wire from scratch using preferences
- Proceed to Step 3

**Case B: Stripe already wired**
- Detect existing Checkout/webhook/Customer Portal patterns
- User wants to *extend* (new tier, new event handler, one-time purchase, usage-based pricing)
- Mirror existing patterns exactly
- Proceed to Step 3

**Case C: Different payment processor detected (Paddle, Lemon Squeezy, etc.)**
- Stop and ask:
  > This project uses <processor>. I'm best at Stripe. Two options:
  > 1. Add the new feature using <processor> (I can guide but I'm less confident)
  > 2. Add Stripe alongside (not recommended — two processors are a maintenance burden)
  > Which?
- If user wants Stripe alongside, treat as Case A but flag the dual-processor concern in the final report.

### Step 3: Ask what to add

Use `AskUserQuestion` if available, or prompt:

> What are you adding?
> 1. **New subscription tier** (additional plan price)
> 2. **One-time purchase** (single charge, not recurring)
> 3. **Usage-based pricing** (charge per API call / per generation / etc.)
> 4. **Customer Portal** (let users manage their own subscription)
> 5. **New webhook event handler** (e.g., `invoice.payment_failed` for dunning)

Then ask for the specific details (price, currency, product description, billing interval).

### Step 4: Write the plan

Show concrete changes. Example for "add new $99/yr tier":

```
PLAN
====
Stripe dashboard setup (you do this manually before I run):
  - Create Product: "Pro Annual"
  - Create Price: $99/year recurring
  - Note the price ID (starts with "price_")

Files to modify:
  - .env.example: add STRIPE_PRICE_PRO_ANNUAL
  - .env.local: add STRIPE_PRICE_PRO_ANNUAL=price_xxx (you provide)
  - <pricing page>: add new tier card
  - <checkout endpoint>: support new price ID
  - <user schema>: extend tier enum if applicable
  - <webhook handler>: no change needed (existing checkout.session.completed works)

Migration needed: <yes/no, depends on schema>
```

### Step 5: Execute, then verify

After execution:

1. Run `stripe listen --forward-to <local-webhook-url>` and trigger a test checkout
2. Use test card `4242 4242 4242 4242`, any future expiry, any CVC
3. Verify webhook signature passes (no 400 from `stripe.webhooks.constructEvent`)
4. Verify DB row updated correctly
5. Verify Customer Portal link works (if relevant)

→ See [stripe-patterns.md](references/stripe-patterns.md) for implementation patterns (webhook signature verification, Checkout Session creation, Customer Portal, events to handle, and anti-patterns).

## Rules

- **Always verify webhook signature.** No exceptions.
- **Always implement idempotency.** Stripe will retry; you'll see duplicate events.
- **Use Customer Portal for subscription management** — don't build your own cancel/upgrade UI.
- **Test with `stripe listen` before deploying.** The webhook signature secret is different in test vs prod.
- **Mirror existing patterns** if Stripe is already wired (file location, error style, response shape).

## If Something Goes Wrong

- **Webhook not received** — confirm the Stripe CLI is running in test mode (`stripe listen --forward-to localhost:3000/api/webhooks/stripe`) and the endpoint is publicly reachable in production.
- **Webhook signature verification fails** — confirm `STRIPE_WEBHOOK_SECRET` is the signing secret for this specific endpoint (each endpoint has its own secret) and that the raw request body is passed to `constructEvent`, not a parsed JSON body.
- **Payment fails in test mode** — use Stripe's test card numbers (`4242 4242 4242 4242`); confirm the test API key starts with `sk_test_`.
- **Customer Portal not loading** — confirm the Portal is configured in the Stripe dashboard (Settings > Billing > Customer Portal) with at least one product and return URL.