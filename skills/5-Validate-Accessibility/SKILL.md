---
name: 5-Validate-Accessibility
when_to_use: "User says 'a11y audit', 'accessibility check', 'screen reader support', 'keyboard navigation', 'WCAG', or types /5-Validate-Accessibility."
description: MUST BE USED when auditing a project for accessibility (a11y) compliance. Calls accessibility-auditor to scan all component files for WCAG 2.1 AA violations, presents a prioritized fix list, and optionally fixes findings one at a time with user approval. Also provides a manual testing checklist for issues static analysis cannot catch. Trigger on `/5-Validate-Accessibility`, "a11y audit", "accessibility check", "screen reader support", "keyboard navigation", "WCAG".
---

# /5-Validate-Accessibility

You run an accessibility audit using the `accessibility-auditor` agent, then present findings with a prioritized fix plan.

## Pre-flight

- Read `.claude/progress.md` (last 5 entries) and `.claude/context.md` if present
- Call `project-state-detector`; if mode is off-pattern for this skill, surface a one-line warning (do NOT block)

## Post-flight

- Append to `.claude/progress.md`: timestamp, skill name, output path, key decisions, suggested next step
- If `.claude/progress.md` is missing, create it with a header first

## Important

- Run the `accessibility-auditor` agent first and present the full report before attempting any fixes.
- Get user approval on each fix before applying it — accessibility changes to markup can break visual layout.
- Do not mark issues as resolved without retesting the fixed component manually or with a screen reader.

## Procedure

### Step 1: Scan

Run in parallel:
- `stack-detector` — framework and component library (React, Vue, Svelte — affects fix patterns)
- `accessibility-auditor` — full static scan for WCAG 2.1 AA violations

### Step 2: Prioritize

Take the `ACCESSIBILITY REPORT`. Confirm the ordering:
1. **Critical** — fix before shipping (blocks keyboard/screen-reader users)
2. **High** — fix this sprint (significantly impairs experience)
3. **Medium** — fix within 30 days
4. **Low** — backlog

Add estimated effort per finding: `<5 min`, `~15 min`, `>30 min`.

### Step 3: Present and offer to fix

Show the report with effort estimates. Ask:
> Want me to fix anything? Tell me which items (e.g., "fix all Critical and High") and I'll work through them one at a time, getting confirmation before each.

### Step 4: Fix (if requested)

Go finding by finding. For each:
- Show the change before applying it
- Get explicit confirmation — accessibility fixes can change visual behavior (element type, spacing, focus ring)
- Apply the fix
- Re-read the changed file to verify the fix is structurally correct

One fix per commit.

### Step 5: Manual testing checklist

After static fixes, always output this:

> **Static analysis finds ~30% of a11y issues. These require human testing:**
>
> - [ ] **Color contrast** — use browser DevTools → Accessibility panel or [Colour Contrast Analyser](https://www.tpgi.com/color-contrast-checker/). Minimum 4.5:1 for normal text, 3:1 for large text.
> - [ ] **Keyboard navigation** — Tab through the entire app. Every interactive element must be reachable and activatable without a mouse.
> - [ ] **Focus indicators** — focused elements must have a visible outline. Search CSS for `outline: none` or `outline: 0` and remove or replace with visible styles.
> - [ ] **Screen reader** — use VoiceOver (Mac: Cmd+F5) or NVDA (Windows, free download). Navigate key flows and listen for meaningful announcements.
> - [ ] **Motion sensitivity** — if the app has animations, test with `prefers-reduced-motion: reduce` in OS settings or DevTools.
> - [ ] **Zoom at 200%** — page must remain functional at 200% browser zoom (no content clipped or lost).

---

## Common Quick Fixes

### Missing alt text

```tsx
// Before
<img src={product.imageUrl} />

// After — informative image
<img src={product.imageUrl} alt={product.name} />

// Decorative image (screen readers skip it)
<img src="/divider.svg" alt="" aria-hidden="true" />
```

### Div or span used as a button

```tsx
// Before
<div onClick={handleClick}>Submit</div>

// After — use a real button (preferred)
<button type="button" onClick={handleClick}>Submit</button>

// After — if it must stay a div
<div
  role="button"
  tabIndex={0}
  onClick={handleClick}
  onKeyDown={e => (e.key === 'Enter' || e.key === ' ') && handleClick()}
>
  Submit
</div>
```

### Input without label

```tsx
// Before
<input type="email" placeholder="Email address" />

// After — explicit label
<label htmlFor="email">Email address</label>
<input id="email" type="email" placeholder="Email address" />

// After — aria-label (when visible label would be redundant)
<input type="search" aria-label="Search products" placeholder="Search..." />
```

### Missing live region for dynamic content

```tsx
// Toast / alert / counter that updates dynamically
<div aria-live="polite" aria-atomic="true">
  {notification && <span>{notification.message}</span>}
</div>

// Loading spinner
<div role="status" aria-label="Loading...">
  <Spinner />
</div>
```

### Modal missing proper ARIA

```tsx
<div
  role="dialog"
  aria-modal="true"
  aria-labelledby="modal-title"
>
  <h2 id="modal-title">Confirm deletion</h2>
  {/* ... */}
</div>
```

### Skipped heading levels

```tsx
// Before (h1 → h3 skips h2)
<h1>Dashboard</h1>
<h3>Recent activity</h3>

// After
<h1>Dashboard</h1>
<h2>Recent activity</h2>
```

---

## Rules

- **Get confirmation before visual changes** — a11y fixes sometimes change element type, spacing, or outline styles.
- **Don't remove visible elements** to "fix" accessibility — e.g., don't `aria-hidden` a heading to avoid a duplication problem; fix the structure.
- **Don't stop at static analysis** — always end with the manual testing checklist.
- **One fix per commit** — easier to revert if a fix introduces a regression.
- **Decorative images with `alt=""` are correct** — don't add alt text to purely decorative images; screen readers should skip them.

## If Something Goes Wrong

- **accessibility-auditor agent times out** — reduce the scan scope to one directory at a time; large component trees with many dynamic elements slow static analysis significantly.
- **Fix breaks visual layout** — revert the specific change and apply a less invasive fix (e.g., use `aria-label` instead of restructuring the DOM); always re-test visually after each fix.
- **Screen reader behavior differs from static analysis** — test with an actual screen reader (NVDA + Chrome or VoiceOver + Safari); static analysis cannot detect all dynamic ARIA state issues.