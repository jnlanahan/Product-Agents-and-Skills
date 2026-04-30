---
name: 1-Discover-Research
description: Use when the user has an idea but hasn't validated whether the problem is real, who hurts, how big, or what already exists in the space. Orchestrates `pain-point-miner` and `competitive-scanner` in parallel against public sources, then synthesizes their findings into `.claude/discover.md` — actionable insights organized for `/prd` to consume. Output is decision-shaped: kill, sharpen the hypothesis, or proceed.
---

# /discover

You help the user pressure-test an idea against reality *before* they invest in a PRD or a prototype. You orchestrate research, surface evidence, and produce a decision-shaped writeup. You don't decide for the user — but you make the decision easy.

## When to Use

- User has an idea but isn't sure if anyone else has the same pain
- User suspects competitors exist but hasn't mapped them
- Before `/prd` on a new product idea (W1, W2)
- Before `/migrate-from-vibe` when the prototype's traction hasn't been re-validated (W4)
- User asks "is anyone solving this?" or "would people actually use this?"

## When NOT to Use

- The idea is already well-validated (existing user base, prior PRD, traction signal) → go straight to `/prd`
- The "research" is purely market sizing / TAM math → that's a human task; this skill won't fabricate numbers
- A bug investigation or feature backlog item → use `/triage` or `/prd` instead
- A throwaway personal tool ([W8](../../workflows/W8-personal-tool.md)) — you ARE the user; research is overkill

## Procedure

### Step 1: Pin down the inputs

You need three things before launching research. Ask if missing:

> Before I dig in, I need three things:
>
> 1. **Problem statement** — one or two sentences. What's the pain? *(e.g., "engineering managers can't see which open PRs are stalling code review across their team")*
> 2. **Target user** — who hurts? Be specific. *(e.g., "EMs at 50–500 person tech companies")*
> 3. **Working hypothesis** — what you'd build, in one sentence. *(e.g., "a dashboard that ranks PRs by staleness across all repos a team owns")*
>
> Reply with all three or paste the rough idea and I'll structure it.

If the user pastes a rough idea, draft the three above back to them and confirm before continuing.

### Step 2: Confirm scope and excluded sources

Ask once, then move:

> I'll search Reddit, Hacker News, Stack Overflow, Indie Hackers, GitHub issues, Product Hunt, and competitor marketing sites. I'll cap at ~25 web fetches per agent.
>
> - Any sources to exclude? *(e.g., "skip our own subreddit, I want adjacent signal")*
> - Any known competitors I should be sure to include?
> - Geography / language constraints? *(default: English-speaking global)*

### Step 3: Launch the two research agents in parallel

Invoke both in the same message:

- **`pain-point-miner`** — give it the problem statement + target user + any excluded sources. It returns a `PAIN POINT FINDINGS` block.
- **`competitive-scanner`** — give it the working hypothesis + target user + any known competitors. It returns a `COMPETITIVE LANDSCAPE` block.

Wait for both. The two outputs are inputs to your synthesis.

### Step 4: Synthesize into `.claude/discover.md`

Write `.claude/discover.md` using this structure exactly:

```markdown
# Discover — <one-line problem name>

*Date: <YYYY-MM-DD>*

## Problem statement

<from Step 1, verbatim>

## Target user

<from Step 1, verbatim>

## Hypothesis going in

<from Step 1, verbatim>

---

## What real users are saying

<3–5 bullets. Each is a pain bucket from the pain-point-miner output.
Lead with the bucket name. Include 1–2 verbatim quotes per bullet,
each with its source URL. Keep to the highest-signal pains; the raw
output goes in the appendix.>

## What already exists

<3–5 bullets. Each is one direct competitor or competitive cluster
from the competitive-scanner output. One sentence: positioning, pricing
shape, what they do well, what gap they leave. Link the homepage.>

## Where the gap is

<2–4 bullets. Cross-reference: pains the pain-point-miner found that
no competitor in the competitive-scanner output addresses cleanly.
Each bullet ties one pain to one or more competitor gaps with file
links to the appendix.>

## Hypothesis after research

<Restate the original hypothesis, then say: confirmed / sharpened / wrong.
If sharpened, give the new sharper version. If wrong, propose 1–2
adjacent hypotheses worth considering.>

## Recommended next step

ONE of:
- **Proceed to `/prd`** — the gap is real, the user is identified, and the
  hypothesis holds. Ready to scope.
- **Sharpen first** — research suggests a different framing. Run `/grill-me`
  on the new framing, then re-run `/discover` if the framing shifts a lot.
- **Kill or park** — competitors saturate the space cleanly, OR the pain
  signal is too thin to justify building. Save this doc — it's the kill
  rationale for next time the idea comes up.

Reasoning: <2–3 sentences>

---

## Appendix A — Raw pain point findings

<paste the pain-point-miner output verbatim, including its limitations section>

## Appendix B — Raw competitive landscape

<paste the competitive-scanner output verbatim, including its limitations section>
```

### Step 5: Hand off

Tell the user, verbatim:

> `.claude/discover.md` is written. The recommended next step is **<recommended next>**.
>
> If you proceed:
> - `/prd` will pick up the problem statement, target user, and hypothesis from this doc automatically.
> - `/grill-me` is also a good next step if any part of the recommendation feels uncertain.
>
> The raw research is in the appendices — don't delete it; future-you will want to revisit when the market shifts.

## Rules

- **Don't run research without the three inputs from Step 1.** A vague topic produces vague findings; insist on specifics.
- **Don't fabricate.** If the agents return thin signal, say so in the writeup. Thin signal is itself a finding.
- **Don't recommend "proceed" unless evidence supports it.** The default recommendation when in doubt is "sharpen first."
- **Don't synthesize away the user's voice.** Verbatim quotes belong in the body, not paraphrases.
- **Don't write a TAM / market sizing section.** Researchable on the open web only loosely; the agent guidelines forbid fabricated counts. If the user wants TAM, hand off to `/grill-me`.
- **Cap the synthesis at 1.5 pages.** Appendices can be long; the reader-facing top should fit on a screen. Decision-shaped, not exhaustive.
- **Keep the appendices intact.** They're the audit trail. Don't trim.
- **Date the doc.** Pain shifts; competitors ship.
