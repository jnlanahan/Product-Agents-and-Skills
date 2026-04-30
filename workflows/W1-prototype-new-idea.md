# W1 — Prototype a New Idea

> *"Could this be a thing? Let me see it."*

## When to use

You have an idea. You want to validate the **concept** — flow, value prop, look-and-feel — before investing real engineering. Output is a clickable HTML prototype someone can navigate in a browser, share with a stakeholder, or hand to a designer. The HTML is throwaway; the *thinking* you do in Define and Design is not.

This workflow is **not a shortcut around the PDLC** — it walks the same phases as W2, just faster and lighter. You still write a PRD (a short one), still pressure-test the idea, still plan, still pick a design. You just stop after Design instead of continuing into Build. If the prototype lands well, you graduate to [W2](W2-production-saas.md) and the PRD/plan/glossary you already produced come with you. If it doesn't, you cost yourself a day, not half a quarter.

## PDLC mapping

| Phase | Skills | Why |
|---|---|---|
| 1 · Discover | *(optional)* `/discover` — quick pass through Reddit / HN / competitor sites | 30-minute reality check; surfaces whether the pain is real and what already exists. Skip only if the idea is for an audience you ARE. |
| 2 · Define | `/grill-me` → `/prd` (lightweight) → `/glossary` *(if domain-heavy)* → `/plan` *(brief)* | Even a one-page PRD forces you to name the user, the job, the success signal |
| 3 · Design | `/prototype` | Three HTML variants → pick one — this is where the workflow earns its keep |
| 4 · Build | *(skipped — or hand the HTML to Replit / V0 for a live demo)* | If Build is needed, you've graduated to W2 |
| 5 · Validate | *(human)* | Show the variants to users / stakeholders; collect reactions |
| 6 · Deploy | *(optional)* `/deploy` | Only if you want a public URL — GitHub Pages, Cloudflare Pages, or the bare HTML works |
| 7 · Learn | *(human-led decision gate)* | Graduate to W2, iterate on the prototype, or kill the idea |

## Skill sequence

1. **`/grill-me`** — stress-test the idea before you write anything. Surfaces unknowns and trade-offs. Skip only if the idea is already crisp.
2. **`/prd`** — write a *lightweight* PRD: who, what, why, success signal, out-of-scope. Half a page is fine. This output graduates with you to W2 if the prototype lands.
3. **`/glossary`** — only if the domain has its own vocabulary worth pinning down now (helps the prototype use consistent labels).
4. **`/plan`** — brief enough to be useful: which screens, which interactions, what's mocked. Don't TDD-plan a throwaway, but do plan the surface area.
5. **`/prototype`** — generates `prototypes/variant-A.html`, `variant-B.html`, `variant-C.html` (Tailwind via CDN, mock data, fake auth, fake API).
6. **(Manual)** open each variant in a browser; pick a winner.
7. **(Optional)** `/deploy` — only if you want a public URL for the chosen variant.

## Diagram

```mermaid
flowchart LR
    A[Idea] --> B[/grill-me/]
    B --> C[/prd/<br/>lightweight]
    C --> D[/glossary/<br/>optional]
    D --> E[/plan/<br/>brief]
    E --> F[/prototype/]
    F --> G[3 HTML variants]
    G --> H[Open in browser]
    H --> I{Pick winner?}
    I -- Yes --> J[Graduate to W2<br/>PRD + plan come with you]
    I -- No --> K[Iterate or kill]
```

## Agents called

- If `/discover` runs: `pain-point-miner` and `competitive-scanner` (both network-touching, parallel).
- The Define-phase skills work in conversation; `/prototype` is self-contained — no codebase to detect.

## Gaps surfaced

- **No `/interview-synthesis` skill** to fold first-party interviews / sales calls into the Discover output. Open-source research is covered by `/discover`; voice-of-customer interviews remain human-led. → [GAPS.md](../GAPS.md#discover-phase)
- **No `/post-launch-review` skill** to capture prototype feedback structurally. → [GAPS.md](../GAPS.md#learn-phase)

## Example walkthrough

You're chatting with a colleague: "We should have an internal tool that shows everyone's GitHub PR review queue, color-coded by age." Sounds reasonable. One day later:

1. **`/grill-me` (15 min)** — discover you actually want **stale PRs only**, sorted by stalest, scoped to your team's repos.
2. **`/prd` (20 min)** — half-page PRD: user is "anyone who reviews PRs," success signal is "fewer PRs older than 5 days," out-of-scope is approvals/merges. The PRD makes the kill criteria explicit: if no one opens it twice, it dies.
3. **`/plan` (10 min)** — three screens (queue, PR detail, settings), all mocked, one Tailwind theme.
4. **`/prototype`** — produces three variants: a kanban board, a list with red/yellow/green age dots, a calendar heatmap.
5. You open all three. Kanban feels wrong. The list is the obvious winner. The heatmap is interesting but premature.
6. You share the list HTML in Slack with two engineers. Both say "yes, build this."

Now you can confidently kick off [W2](W2-production-saas.md) (if it warrants production rigor) or [W8](W8-personal-tool.md) (if it's just for your team's eyes). The PRD and plan you wrote in step 2–3 carry over — they're not throwaway even though the HTML is.
