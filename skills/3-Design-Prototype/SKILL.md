---
name: prototype
description: MUST BE USED when the user wants to design or visualize a feature's UI before building it. Generates 3 radically different clickable HTML prototypes (TailwindCSS via CDN, mock data, fake auth, fake API). Saves to `prototypes/variant-A.html`, `variant-B.html`, `variant-C.html`. The user opens each in a browser, picks one, and that variant becomes the design reference for `/build-feature`. Do NOT use to implement features — prototype output is throwaway HTML only; use `/build-feature` once a variant is chosen.
when_to_use: "User says 'design this feature', 'what should this look like', 'show me UI options', 'make a mockup', 'prototype the UI', 'I want to see options before building'."
---

# /prototype

You generate three radically different clickable HTML prototypes for a feature. The user opens each in a browser, navigates around, picks the one they like, and that becomes the design reference for `/build-feature`.

**These are throwaways.** No backend, no real auth, no real API calls — fake everything. The point is to compare three distinct interaction models and visual approaches before committing to a design.

## Pre-flight

- Read `.claude/progress.md` (last 5 entries) and `.claude/context.md` if present
- Call `project-state-detector`; if mode is off-pattern for this skill, surface a one-line warning (do NOT block)

## Post-flight

- Append to `.claude/progress.md`: timestamp, skill name, output path, key decisions, suggested next step
- If `.claude/progress.md` is missing, create it with a header first

## Before You Start

- Agree on the feature scope with the user before generating — wide or ambiguous scope produces bloated, unfocused prototypes.
- These are disposable HTML files (no backend, no real auth); do not invest time in pixel perfection.

## When to Use

- After `/prd` and `/plan`, before `/build-feature`
- When the user asks "what should this UI look like?"
- When you've described a feature in words and want to make it concrete
- When the user wants to compare radically different design approaches

## When NOT to Use

- For a polished design system — these prototypes are deliberately rough
- For an existing feature you're tweaking (small changes don't justify 3 variants)
- For backend-only or API-only features (no UI to prototype)

## Procedure

### Step 1: Gather requirements

You need just enough to design 3 distinct variants. Ask in one batch:

> To design 3 prototypes I need:
>
> 1. **What feature?** (one-sentence description)
> 2. **Key screens** (e.g., list view, detail view, create/edit form, empty state)
> 3. **Primary user actions** (the 3-5 things users do most)
> 4. **Mock data shape** (what does one item look like? give me 3-5 example records)
> 5. **Constraints** (must use shadcn? must match an existing brand? mobile-first? desktop-first?)

If a `.claude/prd.md` exists, prefer reading it over interviewing.

### Step 2: Plan the three variants

The variants must be **radically different** — different layouts, different interaction models, different visual densities. Not just "blue vs green."

Default variant strategies (override if user has stronger requirements):

- **Variant A — Dense / power-user**: table-heavy, keyboard-friendly, lots of info per screen, inline editing. Inspiration: Linear, Notion databases, admin panels.
- **Variant B — Spacious / consumer**: card-based, generous whitespace, visual hierarchy via type and color, click-through navigation. Inspiration: Stripe Dashboard, Vercel, modern marketing-style apps.
- **Variant C — Single-pane / focused**: one thing on screen at a time, mobile-first feel, swipeable/scrollable, opinionated defaults. Inspiration: Apple Notes, mobile chat apps, conversational UIs.

Show the user the three planned strategies. Confirm. Adjust if needed.

### Step 3: Generate three self-contained HTML files

Create the `prototypes/` directory if it doesn't exist. Write three files: `variant-A.html`, `variant-B.html`, `variant-C.html`.

Each file MUST:

- Be a single self-contained HTML file (no external assets except CDNs)
- Use **TailwindCSS via CDN** (`<script src="https://cdn.tailwindcss.com"></script>`)
- Inline mock data as a JS array at the top of a `<script>` block
- Implement clickable navigation between screens using vanilla JS (`document.querySelector`, `addEventListener`) — no framework
- Have a **fake login** (a "Sign in" button that just hides itself and shows the app)
- Have **fake CRUD** (clicking "Create" pushes to the mock array; "Delete" splices it; UI re-renders)
- Include a header banner: `<div class="bg-yellow-100 ...">PROTOTYPE — Variant A — fake data, no backend</div>`
- Include 3-5 mock records with realistic-looking content
- Cover at least: list view, detail view, create form, empty state
- Be visually polished enough that the user can judge the design (not a wireframe — but not pixel-perfect either)

### Step 4: Show the user how to open them

After writing all three:

```
Three prototypes ready:

  prototypes/variant-A.html   — dense / power-user (Linear-style)
  prototypes/variant-B.html   — spacious / consumer (Stripe-style)
  prototypes/variant-C.html   — single-pane / focused (mobile-first)

Open each one in your browser:
  - Right-click the file in your file explorer → "Open with" → your browser
  - Or drag the file into a browser tab

Click around in each. Try every screen. When you've picked one, tell me which letter,
and that variant becomes the design reference for `/build-feature`.
```

### Step 5: After the user picks

Note the choice in `.claude/plan.md` (or `.claude/prd.md`) under a new `## Design Reference` section pointing to the chosen variant file. Tell the user `/build-feature` will mirror that variant's structure.

## HTML Skeleton (use as starting point for each variant)

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>[Feature Name] — Variant [A|B|C] Prototype</title>
  <script src="https://cdn.tailwindcss.com"></script>
</head>
<body class="bg-slate-50 min-h-screen">
  <div class="bg-yellow-100 border-b border-yellow-300 text-yellow-900 text-sm px-4 py-2 text-center">
    PROTOTYPE — Variant [A|B|C] — fake data, no backend
  </div>

  <!-- Sign-in screen -->
  <div id="signin-screen" class="min-h-[80vh] flex items-center justify-center">
    <button id="signin-btn" class="px-6 py-3 bg-blue-600 text-white rounded-lg font-medium">
      Sign in (fake)
    </button>
  </div>

  <!-- Main app -->
  <div id="app-screen" class="hidden">
    <header class="border-b bg-white px-6 py-4 flex items-center justify-between">
      <h1 class="font-semibold">[Feature Name]</h1>
      <button id="create-btn" class="px-4 py-2 bg-blue-600 text-white rounded-md text-sm">+ New</button>
    </header>

    <main id="main-content" class="p-6">
      <!-- List / detail / form / empty state rendered here -->
    </main>
  </div>

  <script>
    const mockData = [
      { id: '1', title: 'First Example Item', body: 'This is sample content...', createdAt: '2026-04-01' },
      { id: '2', title: 'Second Example', body: 'More sample content...', createdAt: '2026-04-15' },
      { id: '3', title: 'Third Example', body: 'Yet more content...', createdAt: '2026-04-22' },
    ];

    let currentView = 'list'; // list | detail | create
    let selectedId = null;

    document.getElementById('signin-btn').addEventListener('click', () => {
      document.getElementById('signin-screen').classList.add('hidden');
      document.getElementById('app-screen').classList.remove('hidden');
      render();
    });

    document.getElementById('create-btn').addEventListener('click', () => {
      currentView = 'create';
      render();
    });

    function render() {
      const main = document.getElementById('main-content');
      if (mockData.length === 0 && currentView === 'list') {
        main.innerHTML = renderEmpty();
      } else if (currentView === 'list') {
        main.innerHTML = renderList();
      } else if (currentView === 'detail') {
        main.innerHTML = renderDetail();
      } else if (currentView === 'create') {
        main.innerHTML = renderCreate();
      }
      bindEvents();
    }

    function renderList() { /* variant-specific list layout */ return ''; }
    function renderDetail() { /* variant-specific detail */ return ''; }
    function renderCreate() { /* variant-specific form */ return ''; }
    function renderEmpty() { /* variant-specific empty state */ return ''; }

    function bindEvents() {
      // wire up clicks for items, form submit, delete, back, etc.
    }
  </script>
</body>
</html>
```

Each variant fills in `renderList`, `renderDetail`, `renderCreate`, `renderEmpty`, and `bindEvents` differently to express the chosen design strategy.

## Variant-Specific Notes

### Variant A (dense / power-user)

- List = `<table>` with sortable columns
- Detail = side panel that slides in (don't navigate away from the list)
- Create = inline at top of list, not a separate screen
- Use `text-xs` and `text-sm` heavily; tight padding
- Show many fields per row (created date, status, owner, tags)
- Add a `Cmd+K`-style command bar mock (visual only, doesn't have to work)

### Variant B (spacious / consumer)

- List = grid of cards with one item per card
- Detail = full-page navigation (replace the list)
- Create = full-page form
- Use generous padding (`p-8`, `gap-6`); larger fonts (`text-base`, `text-lg`)
- Use color and hierarchy to guide the eye
- Show fewer fields per item, but make them prominent

### Variant C (single-pane / focused)

- List = single scrollable column, full-bleed items
- Detail = takes over the screen entirely
- Create = sheet that slides up from the bottom
- Mobile-first: design at 375px width then scale up
- One primary action at a time
- Use a bottom tab bar for navigation if there are multiple top-level views

## Rules

- **Three variants, radically different.** Not three colors of the same design. The user is choosing an interaction model, not a palette.
- **TailwindCSS via CDN only.** No build step. The user must be able to open the file directly.
- **Fake everything.** No backend, no real auth, no real API. Mock data inline.
- **Self-contained.** Each variant in one file. No imports.
- **Polished enough to judge.** Not a wireframe. Real-looking text, real spacing, real interactions. But not pixel-perfect — that comes later.
- **Don't carry into the build.** When `/build-feature` runs, it builds against the project's actual stack and component library. The prototype is a reference for layout/interaction, not code to copy.
- **Save to `prototypes/`** at the repo root. Add `prototypes/` to `.gitignore` unless the user wants to commit them as design reference.

## If Something Goes Wrong

- **Prototype file does not open in browser** — check that the file is saved to `prototypes/` and open it directly from the filesystem (`File > Open`); it does not need a server.
- **TailwindCSS CDN does not load** — the user may be offline; fall back to inline styles for the prototype session.
- **User cannot decide between variants** — ask two specific questions: "Which one felt fastest to navigate?" and "Which one would you show to a potential user today?"