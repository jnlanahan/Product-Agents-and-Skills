---
title: "Stripe Patterns"
skill: "4-Build-Payments"
---

# Stripe Patterns You MUST Use

Regardless of mode, these are non-negotiable for any Stripe code you write.

## Webhook signature verification

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

For Hono:

```typescript
// Hono webhook route — register WITHOUT auth middleware.
// Stripe signature IS the auth — adding a Bearer/session check will reject all webhooks.
// c.req.text() gives the raw body string before any JSON parsing.
app.post('/api/webhooks/stripe', async (c) => {
  const body = await c.req.text(); // raw body — do NOT use c.req.json()
  const sig = c.req.header('stripe-signature');
  if (!sig) return c.text('No signature', 400);

  let event: Stripe.Event;
  try {
    event = stripe.webhooks.constructEvent(body, sig, process.env.STRIPE_WEBHOOK_SECRET!);
  } catch (err) {
    return c.text('Invalid signature', 400);
  }
  // idempotency + handle...
  return c.text('OK', 200);
});
```

> **Gotcha (Hono):** If the route sits behind an auth middleware that parses the body as JSON first, `c.req.text()` will return an empty string or the stringified JSON — both break signature verification. Register the webhook route on a sub-router that has no body-parsing middleware, or register it before attaching the auth middleware chain.

## Checkout button error handling

Always wrap the checkout redirect in try/catch with a user-visible error. A silent failure (button does nothing, no toast, no message) is worse than showing an error — the user cannot tell if they were charged.

```typescript
// React example — adapt toast to your UI library
async function handleBuyClick() {
  try {
    const res = await fetch('/api/checkout', { method: 'POST' });
    if (!res.ok) throw new Error(await res.text());
    const { url } = await res.json();
    window.location.href = url;
  } catch (err) {
    toast.error('Could not start checkout. Please try again.');
    console.error(err);
  }
}
```

Common causes of a silent 503/500 on the checkout endpoint:
- Stripe keys not loaded (server not restarted after adding to `.env`)
- `STRIPE_SECRET_KEY` is undefined at runtime — add a startup guard:

```typescript
if (!process.env.STRIPE_SECRET_KEY) {
  throw new Error('STRIPE_SECRET_KEY is not set — restart the server after adding it to .env');
}
```

## Checkout Session creation

- Pass `client_reference_id: user.id` so webhook can identify the user
- Set `success_url` and `cancel_url` to actual app routes
- For subscriptions: `mode: 'subscription'`, set `customer` (existing Stripe customer ID) or `customer_email`
- For one-time: `mode: 'payment'`
- Use `metadata` to attach app-specific data; read it in webhook

## Customer Portal

- Server endpoint creates a Portal Session: `stripe.billingPortal.sessions.create({ customer, return_url })`
- Returns the `url` to the client; client redirects
- Configure portal features in Stripe Dashboard (cancel, update payment, view invoices)

## Events to handle

For subscriptions:
- `checkout.session.completed` — initial signup; mark user as paid
- `customer.subscription.updated` — tier change, payment method update
- `customer.subscription.deleted` — cancellation took effect; downgrade
- `invoice.payment_succeeded` — renewal; extend access
- `invoice.payment_failed` — dunning; consider grace period before downgrade

For one-time payments:
- `checkout.session.completed` — fulfill the purchase

## Things to NOT do

- Don't trust amounts from the client — set them server-side from your price config
- Don't store full card numbers — Stripe handles all of that
- Don't poll Stripe API in webhooks — handle the event payload directly
- Don't return 4xx for events you don't care about — return 200 (Stripe will retry on 4xx/5xx)
- Don't put the webhook route behind auth middleware (it's authenticated by signature)
