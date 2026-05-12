---
name: add-email
description: MUST BE USED when adding transactional email to a project. Detects existing email setup; if none, wires Resend per stack preferences. Handles welcome emails, password resets, notification emails, and rich HTML templates with React Email. Always covers DKIM/SPF DNS setup, send-on-event wiring, and delivery verification. Trigger on `/add-email`, "add email", "send email", "transactional email", "welcome email", "password reset email", "email notifications".
---

# /add-email

You add transactional email. Preference is Resend. If a different provider is detected, adapt to it — never migrate.

## Pre-flight

- Read `.claude/progress.md` (last 5 entries) and `.claude/context.md` if present
- Call `project-state-detector`; if mode is off-pattern for this skill, surface a one-line warning (do NOT block)

## Post-flight

- Append to `.claude/progress.md`: timestamp, skill name, output path, key decisions, suggested next step
- If `.claude/progress.md` is missing, create it with a header first

## Important

- Use Resend's sandbox or test mode for all initial integration work — never send real emails to real addresses during development.
- DNS records (SPF, DKIM, DMARC) must be verified in the email provider's dashboard before marking the integration complete; unverified domains get spam-filtered.
- Confirm the "from" address domain is one the user owns and controls before configuring it.

## Procedure

### Step 1: Detect

In parallel:
- `stack-detector` — email provider (Resend, SendGrid, Postmark, Nodemailer, etc.)
- `pattern-finder` — "Find existing email code: send functions, email templates, event triggers"

Read `_stack-preferences.md`.

### Step 2: Determine mode

| Detected | Action |
|---|---|
| No email setup | Install Resend (preference) |
| Resend wired | Extend it |
| SendGrid wired | Extend SendGrid |
| Postmark wired | Extend Postmark |
| Nodemailer wired | Extend Nodemailer |

### Step 3: Ask what to add

> What email feature?
> 1. **Initial setup** (no email exists yet)
> 2. **Welcome / onboarding email** (triggered on sign-up)
> 3. **Password reset email**
> 4. **Notification email** (triggered by in-app events)
> 5. **Digest / summary email** (scheduled — e.g., weekly activity)
> 6. **Custom rich HTML template**

### Step 4: Walk through DNS setup

Before any code, explain what the user needs to do in Resend dashboard:

> Before email will land in inboxes rather than spam, add these DNS records for your sending domain:
>
> 1. **Resend dashboard** → Domains → Add domain → follow verification
> 2. Add the DKIM TXT record Resend generates (`resend._domainkey`)
> 3. Add/update SPF: `v=spf1 include:amazonses.com ~all`
> 4. Add DMARC: `v=DMARC1; p=quarantine; rua=mailto:dmarc@yourdomain.com`
>
> Reply when DNS records are propagated (Resend dashboard shows "Verified").

### Step 5: Execute

Wire code per the patterns below.

### Step 6: Verify

- Send a real test email to the user's address via the new code
- Check Resend dashboard → Logs for delivery status
- Check spam folder if not received
- For triggered emails: trigger the event in the app, confirm email arrives within 60 seconds

---

## Resend Patterns

### Install

```bash
npm install resend @react-email/components react react-dom
```

### Env vars

```
RESEND_API_KEY=re_...
EMAIL_FROM=noreply@yourdomain.com
NEXT_PUBLIC_APP_URL=https://yourdomain.com
```

### Email client

```typescript
// lib/email.ts
import { Resend } from 'resend';
import { render } from '@react-email/render';

const resend = new Resend(process.env.RESEND_API_KEY!);
const FROM = process.env.EMAIL_FROM ?? 'noreply@yourdomain.com';

export async function sendEmail({
  to,
  subject,
  template,
}: {
  to: string;
  subject: string;
  template: React.ReactElement;
}) {
  const html = await render(template);
  const text = await render(template, { plainText: true });
  const { data, error } = await resend.emails.send({ from: FROM, to, subject, html, text });
  if (error) throw new Error(`Email failed: ${error.message}`);
  return data;
}
```

### Welcome email template

```tsx
// emails/WelcomeEmail.tsx
import { Html, Head, Body, Container, Heading, Text, Button, Hr } from '@react-email/components';

export function WelcomeEmail({ firstName, appUrl }: { firstName: string; appUrl: string }) {
  return (
    <Html>
      <Head />
      <Body style={{ fontFamily: 'sans-serif', backgroundColor: '#f6f9fc', margin: 0 }}>
        <Container style={{ margin: '40px auto', padding: '24px', maxWidth: '560px', backgroundColor: '#fff', borderRadius: '8px' }}>
          <Heading style={{ fontSize: '24px', margin: '0 0 16px' }}>Welcome, {firstName}!</Heading>
          <Text style={{ color: '#555', lineHeight: '1.6' }}>
            You're all set. Click below to get started.
          </Text>
          <Button
            href={`${appUrl}/dashboard`}
            style={{ backgroundColor: '#000', color: '#fff', padding: '12px 24px', borderRadius: '6px', textDecoration: 'none', display: 'inline-block', marginTop: '16px' }}
          >
            Get started
          </Button>
          <Hr style={{ margin: '32px 0', borderColor: '#eee' }} />
          <Text style={{ fontSize: '12px', color: '#999' }}>
            You received this because you signed up. Questions? Reply to this email.
          </Text>
        </Container>
      </Body>
    </Html>
  );
}
```

### Sending welcome email on sign-up

```typescript
// Call this wherever new users are created (auth webhook, sign-up handler, etc.)
import { sendEmail } from '@/lib/email';
import { WelcomeEmail } from '@/emails/WelcomeEmail';

export async function onUserCreated(email: string, displayName: string) {
  const firstName = displayName.split(' ')[0] ?? 'there';
  await sendEmail({
    to: email,
    subject: 'Welcome!',
    template: <WelcomeEmail firstName={firstName} appUrl={process.env.NEXT_PUBLIC_APP_URL!} />,
  });
}
```

### Password reset email

```typescript
// lib/password-reset.ts
import crypto from 'crypto';
import { sendEmail } from '@/lib/email';
import { Html, Body, Container, Text, Button } from '@react-email/components';

function PasswordResetEmail({ resetUrl }: { resetUrl: string }) {
  return (
    <Html>
      <Body style={{ fontFamily: 'sans-serif' }}>
        <Container style={{ maxWidth: '560px', margin: '40px auto', padding: '24px' }}>
          <Text>Click the link below to reset your password. It expires in 1 hour.</Text>
          <Button href={resetUrl} style={{ backgroundColor: '#000', color: '#fff', padding: '12px 24px', borderRadius: '6px' }}>
            Reset password
          </Button>
          <Text style={{ fontSize: '12px', color: '#999', marginTop: '16px' }}>
            If you didn't request this, ignore this email.
          </Text>
        </Container>
      </Body>
    </Html>
  );
}

export async function initiatePasswordReset(to: string) {
  const token = crypto.randomBytes(32).toString('hex');
  const expiresAt = new Date(Date.now() + 60 * 60 * 1000); // 1 hour
  // TODO: save { token, expiresAt } to DB keyed by email
  const resetUrl = `${process.env.NEXT_PUBLIC_APP_URL}/reset-password?token=${token}`;
  await sendEmail({ to, subject: 'Reset your password', template: <PasswordResetEmail resetUrl={resetUrl} /> });
  return { token, expiresAt };
}
```

---

## SendGrid Adaptation (when extending existing)

```typescript
import sgMail from '@sendgrid/mail';
sgMail.setApiKey(process.env.SENDGRID_API_KEY!);

export async function sendEmail({ to, subject, html, text }: { to: string; subject: string; html: string; text: string }) {
  await sgMail.send({ to, from: process.env.EMAIL_FROM!, subject, html, text });
}
```

---

## Rules

- **Never migrate email providers** — deliverability reputation is hard-won and easy to break.
- **Always include plain text** alongside HTML — spam filters trust it; render `plainText: true` from React Email.
- **Verify DNS before sending** — DKIM/SPF failures route to spam. Confirm "Verified" in Resend dashboard first.
- **Don't send from personal email addresses** — use a verified custom domain.
- **Rate-limit send endpoints** — especially password reset (prevents enumeration attacks and spam abuse).
- **Log send failures silently** — don't surface email errors to end users; don't crash user flows over email.
- **Password reset tokens**: random (`crypto.randomBytes`), short expiry (1 hour), stored hashed in DB, single-use.

## If Something Goes Wrong

- **Email not delivered** — check the Resend dashboard for delivery status; common causes are an unverified domain, a blocked recipient, or a spam filter.
- **DNS records not verifying** — propagation can take up to 48 hours; use `dig TXT yourdomain.com` to check current records; do not change records again until propagation completes.
- **Webhook events from Resend not received** — confirm the webhook endpoint is publicly reachable (not localhost) and that the signing secret matches the one in your env.
- **React Email template renders blank** — check that the template is exported as a default export and that `@react-email/render` is called with `await`.