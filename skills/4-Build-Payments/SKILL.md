---
name: 4-Build-Payments
description: MUST BE USED when the user wants to add or extend payments. Stripe-first; detects existing Stripe setup and extends, or detects a different processor (Paddle, Lemon Squeezy) and adapts. Always handles webhook signature verification, idempotency, and Customer Portal. Trigger on `/add-payment`, "add subscription", "add billing", "add Stripe".
---

# /add-payment

You add or extend payment functionality. Stripe-only — if the project uses a different processor (Paddle, Lemon Squeezy), surface that and ask whether to proceed with the existing processor or migrate.

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

## Stripe Patterns You MUST Use

Regardless of mode, these are non-negotiable for any Stripe code you write:

### Webhook signature verification

```typescript
// Next.js App Router (app/api/webhooks/stripe/route.ts)
import { headers } from 'next/headers';
import Stripe from 'stripe';

const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!);

export async function POST(req: Request) {
  const body = await req.text(); // raw body, NOT json()
  const sig = (await headers()).get('stripe-signature');
  if (!sig) return new Response('No signature', { status: 400 });

  let event: Stripe.Event;
  try {
    event = stripe.webhooks.constructEvent(
      body,
      sig,
      process.env.STRIPE_WEBHOOK_SECRET!
    );
  } catch (err) {
    return new Response('Invalid signature', { status: 400 });
  }

  // Idempotency: skip if event already processed
  const seen = await db.query.processedEvents.findFirst({
    where: eq(processedEvents.id, event.id),
  });
  if (seen) return new Response('Already processed', { status: 200 });

  // Handle event...
  await db.insert(processedEvents).values({ id: event.id, type: event.type });

  return new Response('OK', { status: 200 });
}
```

For Express:

```typescript
// MUST register webhook route BEFORE express.json() middleware
app.post('/api/webhooks/stripe',
  express.raw({ type: 'application/json' }),
  async (req, res) => {
    const sig = req.headers['stripe-signature'] as string;
    let event: Stripe.Event;
    try {
      event = stripe.webhooks.constructEvent(req.body, sig, process.env.STRIPE_WEBHOOK_SECRET!);
    } catch (err) {
      return res.status(400).send('Invalid signature');
    }
    // idempotency + handle...
  }
);
```

### Checkout Session creation

- Pass `client_reference_id: user.id` so webhook can identify the user
- Set `success_url` and `cancel_url` to actual app routes
- For subscriptions: `mode: 'subscription'`, set `customer` (existing Stripe customer ID) or `customer_email`
- For one-time: `mode: 'payment'`
- Use `metadata` to attach app-specific data; read it in webhook

### Customer Portal

- Server endpoint creates a Portal Session: `stripe.billingPortal.sessions.create({ customer, return_url })`
- Returns the `url` to the client; client redirects
- Configure portal features in Stripe Dashboard (cancel, update payment, view invoices)

### Events to handle

For subscriptions:
- `checkout.session.completed` — initial signup; mark user as paid
- `customer.subscription.updated` — tier change, payment method update
- `customer.subscription.deleted` — cancellation took effect; downgrade
- `invoice.payment_succeeded` — renewal; extend access
- `invoice.payment_failed` — dunning; consider grace period before downgrade

For one-time payments:
- `checkout.session.completed` — fulfill the purchase

### Things to NOT do

- Don't trust amounts from the client — set them server-side from your price config
- Don't store full card numbers — Stripe handles all of that
- Don't poll Stripe API in webhooks — handle the event payload directly
- Don't return 4xx for events you don't care about — return 200 (Stripe will retry on 4xx/5xx)
- Don't put the webhook route behind auth middleware (it's authenticated by signature)

## Rules

- **Always verify webhook signature.** No exceptions.
- **Always implement idempotency.** Stripe will retry; you'll see duplicate events.
- **Use Customer Portal for subscription management** — don't build your own cancel/upgrade UI.
- **Test with `stripe listen` before deploying.** The webhook signature secret is different in test vs prod.
- **Mirror existing patterns** if Stripe is already wired (file location, error style, response shape).
