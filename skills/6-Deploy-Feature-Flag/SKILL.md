---
name: 6-Deploy-Feature-Flag
when_to_use: "User says 'add feature flag', 'gate this feature', 'staged rollout', 'kill switch', 'progressive rollout', 'dark launch', or types /6-Deploy-Feature-Flag."
description: MUST BE USED when wiring feature flags to gate a new feature for staged rollout, A/B testing, or kill-switch control. Uses PostHog feature flags (stack preference) or extends detected flag tooling. Generates flag-guarded code and a staged rollout plan. Trigger on `/6-Deploy-Feature-Flag`, "add feature flag", "gate this feature", "staged rollout", "kill switch", "progressive rollout", "dark launch".
---

# /6-Deploy-Feature-Flag

You wire feature flags to control feature visibility — for staged rollouts, A/B tests, or kill switches.

## Pre-flight

- Read `.claude/progress.md` (last 5 entries) and `.claude/context.md` if present
- Call `project-state-detector`; if mode is off-pattern for this skill, surface a one-line warning (do NOT block)

## Post-flight

- Append to `.claude/progress.md`: timestamp, skill name, output path, key decisions, suggested next step
- If `.claude/progress.md` is missing, create it with a header first

## Critical

- Flags must default to OFF (false/null) on creation — never ship a flag that defaults to enabled for all users.
- Test the flag in staging with it both ON and OFF before enabling it in production; a flag that only works one way is not a flag.
- Keep the flag-guarded code path clean — do not let flag conditions accumulate without a removal plan (flag debt causes outages).

## Procedure

### Step 1: Detect

In parallel:
- `stack-detector` — PostHog, LaunchDarkly, Statsig, Unleash, or no flag tooling
- `pattern-finder` — "Find existing feature flag checks: isFeatureEnabled, useFeatureFlag, posthog.isFeatureEnabled, FEATURE_FLAGS"

Read `_stack-preferences.md` — PostHog is the preferred provider.

### Step 2: Determine mode

| Detected | Action |
|---|---|
| No flags | Wire PostHog feature flags (stack preference) |
| PostHog present | Add flag via PostHog |
| LaunchDarkly present | Extend LaunchDarkly |
| Custom flags (env var or DB table) | Extend existing |

### Step 3: Gather flag details

Ask:
> 1. Feature name (used as the flag key — snake_case, e.g., `new-dashboard-v2`)
> 2. Initial rollout: 0% (off for everyone), 5% (internal), or 100% (on for everyone)?
> 3. Targeting: by user ID percentage, user property (e.g., `plan = "pro"`), or org/team?

### Step 4: Create the flag in PostHog

Walk the user through:
1. PostHog dashboard → Feature Flags → New Feature Flag
2. Key: `<feature-name>` (snake_case, must match code)
3. Set rollout percentage or targeting rules
4. Save

### Step 5: Wire in code

Add flag checks per the patterns below. Add a PostHog event when the feature is used so adoption is trackable.

### Step 6: Verify

- Test with a user/session in the flag's target group → feature visible
- Test with a user NOT in the target group → feature hidden
- Set flag to 0% in PostHog → feature hidden for everyone (confirms kill switch works)
- Check PostHog → Feature Flags → the flag shows usage

### Step 7: Save rollout plan

Save a staged rollout plan to `.claude/flag-<name>-rollout.md`.

---

## PostHog Feature Flag Patterns

### Client-side React hook

```typescript
// hooks/useFeatureFlag.ts
'use client';
import { usePostHog } from 'posthog-js/react';

export function useFeatureFlag(key: string): boolean {
  const posthog = usePostHog();
  return posthog?.isFeatureEnabled(key) ?? false;
}
```

```tsx
// Usage in component
import { useFeatureFlag } from '@/hooks/useFeatureFlag';

export function Dashboard() {
  const showNewDashboard = useFeatureFlag('new-dashboard-v2');
  return showNewDashboard ? <NewDashboard /> : <OldDashboard />;
}
```

### Server-side flag check (Next.js / Node)

```bash
npm install posthog-node
```

```typescript
// lib/flags.ts
import { PostHog } from 'posthog-node';

const posthogClient = new PostHog(process.env.NEXT_PUBLIC_POSTHOG_KEY!, {
  host: process.env.NEXT_PUBLIC_POSTHOG_HOST,
});

export async function isFeatureEnabled(flagKey: string, userId: string): Promise<boolean> {
  const enabled = await posthogClient.isFeatureEnabled(flagKey, userId);
  return enabled ?? false;
}
```

```typescript
// In an API route
const enabled = await isFeatureEnabled('new-dashboard-v2', userId);
if (!enabled) return new Response('Not found', { status: 404 });
```

### Track flag exposure

```typescript
// Always capture when the flag meaningfully controls behavior
posthog.capture('feature_viewed', {
  feature_flag: 'new-dashboard-v2',
  flag_enabled: isEnabled,
});
```

---

## Rollout Plan Template

```markdown
# Feature Flag Rollout: <feature-name>

## Flag key
`<flag-key>`

## Why this feature is flagged
<one sentence: risk level, target audience, or what we're testing>

## Rollout stages
1. **0%** — code ships but flag is off; validate no side effects
2. **5%** — internal users + early adopters; watch for errors
3. **25%** — broader rollout; monitor PostHog funnel + Sentry error rate
4. **50%** — majority rollout; watch p95 latency
5. **100%** — full rollout
6. **Flag removal** — 2+ weeks after 100%, clean up flag code and delete from PostHog

## Metrics to watch
- Sentry: new errors in <feature area>
- PostHog: <key event> count and funnel completion
- PostHog: session replays for users with flag enabled

## Kill switch
If any metric degrades: set flag to 0% in PostHog dashboard. **No code deploy needed.**

## Removal criteria
Flag can be deleted when: 100% rollout for 2+ weeks, no incident since rollout, metrics stable.
```

---

## Rules

- **Code ships before the flag turns on** — merge flag-gated code with the flag at 0%; enable separately after verified.
- **Flags are temporary** — schedule removal once 100% rollout is confirmed stable. See `/schedule`.
- **Never gate the critical path (auth, billing)** without verifying the kill switch at 0% first.
- **Server-side flags for sensitive features** — client-side flags are visible in the browser bundle; don't gate admin-only or billing features client-side only.
- **Always track an exposure event** — without it, PostHog can't tell if flagged users actually saw the feature.
- **One flag, one feature** — don't reuse a flag key across multiple features.

## If Something Goes Wrong

- **Flag not activating for expected users** — confirm the flag condition in PostHog matches the user properties being sent; check that `posthog.identify()` is called with the correct user ID before flag evaluation.
- **Feature leaks to all users despite flag being OFF** — check for missing flag checks in server-side routes; a flag guard on the client only is not sufficient if the API is also accessible.
- **Flag cannot be toggled without a deploy** — the flag must be wired to the PostHog API, not hardcoded; verify the SDK call returns the live flag value on each request.