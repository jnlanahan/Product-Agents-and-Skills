---
name: 4-Build-Evals
description: MUST BE USED when wiring LangSmith to an AI feature. Distinguishes tracing (visibility), evals (regression testing), and online evaluations (automatic production scoring) — asks the user what they actually need, then builds only that. Step-by-step and dummy-proof.
when_to_use: "User says 'add evals', 'build evals', 'set up LangSmith', 'wire tracing', 'how do I know if my AI is working', 'test my AI output', 'add LangSmith', 'monitor my AI in production'."
---

# /4-Build-Evals

LangSmith does three separate things that are often confused for each other. Build the wrong one and you'll be disappointed. This skill routes you to the right one first.

---

## Pre-flight

- Read `.claude/progress.md` (last 5 entries) and `.claude/context.md` if present
- Call `project-state-detector`; if mode is off-pattern for this skill, surface a one-line warning (do NOT block)

---

## Step 0: What are you trying to set up?

Before touching any code, ask:

> "What are you trying to know about your AI?"

| I want to know… | That's → | What you get |
|---|---|---|
| What my AI is doing — inputs, outputs, token counts, latency | **Tracing** | A live log of every AI call. No scoring. Pure visibility. |
| Whether my AI got worse after I changed the prompt or model | **Evals** | A score that compares runs against a fixed dataset of examples. You trigger it manually. |
| Whether my AI is producing bad output right now, automatically | **Online Evaluations** | LangSmith scores every incoming production trace as it arrives. Active monitoring. |

**Each path is independent.** You can do just tracing, just evals, or combine them. Tracing is always worth adding (it's low effort and costs nothing extra). Evals and online evals are optional layers on top.

**Recommended starting point for most projects:**
- New to LangSmith → start with **Path A (Tracing)**
- Ready to protect against prompt regressions → add **Path B (Evals)**
- Have real user traffic and want automatic quality scoring → add **Path C (Online Evals)**

Jump to the path the user needs. You can do more than one in the same session — do them in order A → B → C.

---

## Path A: Tracing

**What it is:** Every call to your AI is recorded — input, output, model, latency, token count, cost. Nothing is scored. This is visibility, not testing.

**When to use:** Always. Even in development. Tracing is how you debug a bad AI response, understand your real token costs, and build the dataset you'll need for evals later.

**Effort:** 30 minutes.

### A1. Create a LangSmith account

1. Go to [smith.langchain.com](https://smith.langchain.com) and sign up (free for small usage)
2. Click your profile icon → **Settings** → **API Keys** → **Create API Key** → copy it

> **Workspace note:** Your API key is tied to a specific workspace (org). If you belong to multiple orgs, create the key from the workspace where you want traces to appear.

### A2. Install and configure

```bash
# TypeScript/Node
npm install langsmith

# Python
pip install langsmith
```

Add to your `.env` or `.env.local` (never commit this file):

```
LANGSMITH_TRACING=true
LANGSMITH_API_KEY=ls__...         ← paste your key here
LANGSMITH_PROJECT=my-app-evals    ← pick any name for this project
```

**Restart your dev server after adding env vars.**

### A3. Wrap your AI client

The fastest approach is to wrap the SDK client once — every call is then automatically traced with no further changes.

**TypeScript — Anthropic SDK:**
```typescript
// lib/ai.ts
import Anthropic from '@anthropic-ai/sdk';
import { wrapAnthropic } from 'langsmith/wrappers';

export const ai = wrapAnthropic(new Anthropic());
// Every ai.messages.create() call is now traced automatically
```

**TypeScript — Vercel AI SDK:**
```typescript
import { anthropic } from '@ai-sdk/anthropic';
import { wrapAISDKModel } from 'langsmith/wrappers/vercel';

export const model = wrapAISDKModel(anthropic('claude-sonnet-4-6'));
// Pass model to generateText / streamText as normal
```

**Python — Anthropic SDK:**
```python
# lib/ai.py
from anthropic import Anthropic
from langsmith.wrappers import wrap_anthropic

ai = wrap_anthropic(Anthropic())
# Every ai.messages.create() call is now traced automatically
```

→ For projects where you can't swap the client (e.g. third-party code), use the `traceable` decorator approach instead: [langsmith-eval-patterns.md](references/langsmith-eval-patterns.md#tracing-wrapper)

### A4. Verify

Make one real call to your AI feature. Then open [smith.langchain.com](https://smith.langchain.com) → your project → **Traces**.

A new trace should appear within 30 seconds. It shows: input, output, model used, latency, token count, cost.

**Don't continue to Path B or C until you see a trace.** If nothing appears, see the troubleshooting section at the bottom.

**Path A is done.** You now have full visibility into every AI call. If you also want regression testing, continue to Path B.

---

## Path B: Evals (regression testing)

**What it is:** You maintain a fixed dataset of example inputs (and expected outputs). Before or after any prompt or model change, you run a script that calls your AI on every example and scores the output. If the score drops, you know something broke.

**What it is NOT:** Active monitoring. Evals don't run automatically. You trigger them manually. Think of it like a test suite — it only catches regressions when you run it.

**When to use:** When you're about to change a prompt, switch models, or ship a significant update to your AI feature. Also useful as a baseline before the first deploy.

**Prerequisite:** Path A tracing must be set up first.

**Effort:** 2–4 hours for the first eval.

### Step B1: Find and understand your AI code

Run `stack-detector` to find what AI packages are installed, then search for the code that calls the AI.

**Look for:**
- Any file importing `@anthropic-ai/sdk`, `openai`, `langchain`, or `ai` (Vercel AI SDK)
- Functions named `generate`, `chat`, `complete`, `ask`, `summarize`, `classify`, `extract`
- API routes that call an LLM (usually `app/api/`, `pages/api/`, or `routes/`)

**Capture:**
1. What is the AI being asked to do? (classify, extract, summarize, answer questions, generate text, etc.)
2. What does the input look like? (a user message, a document, a structured form, etc.)
3. What does the output look like? (a category label, a JSON object, a paragraph of text, etc.)
4. Does it run repeatedly on similar inputs, or is it one-off per user session?

Share what you found. If the codebase has no AI code yet, stop here — run `/add-ai` first.

### Step B2: Decide if an eval is worth building

Not every AI feature benefits from a formal eval. Answer these questions:

| Question | If YES → | If NO → |
|---|---|---|
| Does the AI run the same type of task repeatedly? | Evals are worth building | Skip — one-off tasks don't benefit |
| Is there a way to tell if the output is good or bad? | Evals are possible | Skip — if "good" can't be defined, evals won't help |
| Will you change the prompt or model in the future? | Evals protect against regressions | Skip if this is a one-time throwaway |
| Do you have at least 5 example inputs you can test? | Evals can run now | Start collecting examples first |

**Match your AI feature to an eval approach:**

| What your AI does | Best eval type | Effort |
|---|---|---|
| Classifies text into fixed categories | Rule-based (exact match) | Low |
| Extracts fields from documents | Rule-based (field match) | Low |
| Generates structured output (JSON, CSV) | Rule-based (schema check) | Low |
| Answers questions from a knowledge base | LLM-as-judge (accuracy) | Medium |
| Summarizes documents | LLM-as-judge (coverage, conciseness) | Medium |
| Multi-turn conversation / customer support | LLM-as-judge (helpfulness, tone) | Medium |
| Creative writing, open-ended generation | LLM-as-judge (rubric) or Human | High |
| Agent with tools (web search, API calls, verification steps) | LLM-as-judge — **read the warning below** | High |
| Real-time autocomplete | Skip — use product metrics instead | — |
| One-off user request (no repetition) | Skip — run manually | — |

> **Warning — agents with live tools (web search, external APIs):** There is no mock option. The eval pipeline calls the real tool on every example, every run. A 10-example eval on an agent that does web search + verification will take **5–10 minutes** and cost roughly **$5–15 in API calls** depending on your model and tool chain. Before running, confirm: (1) you have budget for live tool calls, and (2) the tools won't have side effects on production data (writes, sends, purchases). If cost is a blocker, reduce the dataset to 5 examples for the first baseline run.

**If the answer is "skip":** Suggest the right alternative (product analytics via PostHog, manual spot-checking, or A/B testing). Do not force an eval onto a use case that doesn't need one.

### Step B3: Build your dataset

A dataset is a list of example inputs (and optionally expected outputs). Think of it as your golden examples.

**Rules:**
- Minimum 5 examples — a score from fewer is meaningless
- Include happy-path examples (typical inputs that should work)
- Include edge cases (tricky, unusual, or borderline inputs)
- Include at least 1 example that should produce a "I don't know" — to verify the AI doesn't hallucinate

**How to create examples:**

Option A — Write them yourself (easiest to start):
→ See [langsmith-eval-patterns.md](references/langsmith-eval-patterns.md#dataset-creation) for the inline dataset script.

Option B — Pull from real traces (best after you have user traffic):
In LangSmith → **Traces** → click a trace → **Add to Dataset**. Fastest way to build a dataset from real usage.

Ask the user: "Do you have 5+ example inputs you can walk me through right now? If yes, I'll format them into the script. If no, let's make a few real calls to the AI first, then pull from those traces."

### Step B4: Pick and build your evaluator

**Option A: Rule-based** — a regular function checks the output. No AI needed, runs instantly, costs nothing.
Best for: classification, extraction, JSON output — anything with a clear right/wrong answer.
→ See [langsmith-eval-patterns.md](references/langsmith-eval-patterns.md#rule-based-evaluator)

**Option B: LLM-as-judge** — a second Claude call reads the input + output and returns a score (0–1) with a reason.
Best for: summarization, Q&A, helpfulness, tone — anything where "good" requires judgment.

Before writing the evaluator, define the rubric. Ask:
> "What makes a response good? List 3–5 specific criteria — e.g. 'accurate', 'under 100 words', 'does not make up information'."

→ See [langsmith-eval-patterns.md](references/langsmith-eval-patterns.md#llm-as-judge-evaluator)

**Option C: Human** — LangSmith surfaces each output in a review UI. You click thumbs up/down.
Best for: creative output, high-stakes decisions, when you genuinely can't define "good" in code.
No evaluator code needed. In LangSmith → **Datasets** → your dataset → **Annotate**.

### Step B5: Wire and run the eval script

Create `scripts/run-evals.ts` (or `scripts/run_evals.py`).
→ See [langsmith-eval-patterns.md](references/langsmith-eval-patterns.md#run-script) for the full template.

```bash
# TypeScript
npx tsx scripts/run-evals.ts

# Python
python scripts/run_evals.py
```

### Step B6: Verify in LangSmith

1. Open [smith.langchain.com](https://smith.langchain.com) → your project
2. Click **Datasets & Testing** → find your dataset → **Experiments**
3. You should see your run with a score per evaluator
4. Click any row to see input → output → score breakdown

**What a healthy score looks like:**
- ≥ 0.8 = AI is doing well
- 0.5–0.8 = investigate the failing examples
- < 0.5 = the prompt needs work before you ship

**Save this as your baseline.** Next time you change the prompt or upgrade the model, re-run and compare.

---

## Path C: Online Evaluations (automatic production scoring)

**What it is:** LangSmith scores every incoming production trace automatically, as it arrives — no manual trigger required. You define an evaluator once; LangSmith applies it to new traces in the background.

**What it is NOT:** A replacement for evals. Online evals catch bad outputs in real user traffic. Evals catch regressions before you ship. You want both once your product has real users.

**When to use:** Once you have consistent real user traffic. Running online evals on a feature with 5 users a day is not worth the setup cost. A good signal is when you find yourself manually spot-checking traces more than once a week.

**Prerequisite:** Path A tracing must be set up first. Online evals require that traces are flowing into LangSmith.

**LangSmith plan note:** Online evaluations are available on LangSmith's paid plans (Plus and above). Check [smith.langchain.com/pricing](https://smith.langchain.com/pricing) before starting.

### C1. Define your online evaluator in LangSmith

Online evaluators are configured in the LangSmith UI (not in your code):

1. Open [smith.langchain.com](https://smith.langchain.com) → your project
2. Click **Monitoring** in the left sidebar → **Online Evaluations**
3. Click **+ New Evaluator**
4. Choose type:
   - **LLM-as-judge** — write a scoring prompt; LangSmith runs it on every new trace
   - **Exact match** — checks if a field in the output equals an expected value
5. Set the **sampling rate** — start at 10–20% to control costs. 100% scores every trace.
6. Click **Save** — LangSmith will now score new traces automatically

### C2. Define a good scoring prompt for LLM-as-judge

The prompt is the same concept as Path B's LLM-as-judge — but it runs automatically, so the rubric must be precise. Vague rubrics produce noisy scores that aren't actionable.

Ask the user:
> "What would make you want to go investigate a specific trace? Write that as a yes/no question — e.g. 'Did the AI make up a fact that isn't in the source document?'"

One clear yes/no question per evaluator. Multiple evaluators for multiple concerns.

### C3. Set up an alert (optional but recommended)

In LangSmith → **Monitoring** → **Alerts**:
- Set a threshold: e.g. "alert me if the average score drops below 0.7 over the last 50 traces"
- Connect to Slack or email

This turns online evals into active monitoring — you get paged when quality drops.

### C4. Verify

Make a few real calls to your AI feature. Open LangSmith → **Monitoring** → **Online Evaluations**. Within a few minutes, new traces should show evaluation scores attached.

---

## Post-flight

- Append to `.claude/progress.md`: timestamp, skill name, which paths were completed, baseline score (if evals), suggested next step
- If `.claude/progress.md` is missing, create it with a header first

---

## Rules

- **Tracing first, always** — evals and online evals both depend on tracing being set up correctly.
- **At least 5 examples before running evals** — fewer makes scores statistically meaningless.
- **One evaluator per concern** — don't bundle "accurate AND concise AND professional" into one score. Separate evaluators make failures easier to diagnose.
- **LLM-as-judge uses a different model** — never use the same model you're evaluating as the judge. Haiku to judge Sonnet; Sonnet to judge Haiku. Same-model judging is biased.
- **Never eval on production data directly** — use a copy or a dedicated test dataset.
- **No hardcoded API keys** — always `process.env.LANGSMITH_API_KEY`.
- **Re-run evals before any prompt change** — establish a baseline first, then compare after.

---

## If Something Goes Wrong

- **No traces appearing** — confirm `LANGSMITH_TRACING=true` AND `LANGSMITH_API_KEY` are both set; restart the server. Events can take 30–60 seconds to appear. Confirm the API key was created from the correct workspace (see Path A, step A1).
- **`wrapAnthropic` not found** — confirm you're on `langsmith` ≥ 0.1.x: `npm install langsmith@latest`.
- **`ls.createDataset` throws "dataset already exists"** — add `if_exists: 'skip'` (Python) or check before creating (TS). Dataset names must be unique per project.
- **LLM-as-judge always returns 1.0** — the rubric is too easy. Make criteria more specific and add genuinely hard edge cases.
- **Score varies wildly between runs** — dataset is too small (add examples) or judge temperature is too high (set `temperature: 0`).
- **Online evaluations tab not visible** — this feature requires a paid LangSmith plan.
- **`npx tsx` not found** — `npm install -D tsx`.
- **Python `langsmith` import error** — confirm virtualenv: `pip install langsmith anthropic`.

---

→ See [langsmith-eval-patterns.md](references/langsmith-eval-patterns.md) for all code patterns: SDK wrapping (tracing), `traceable` decorator, dataset creation, rule-based evaluator, LLM-as-judge evaluator, and the full eval run script.
