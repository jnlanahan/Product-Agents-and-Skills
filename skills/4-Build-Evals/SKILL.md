---
name: 4-Build-Evals
description: MUST BE USED when adding LangSmith evals to an AI feature. Reads the project's AI code, determines whether evals make sense for the use case, designs a golden dataset and evaluator, and wires a runnable eval script. Step-by-step and dummy-proof.
when_to_use: "User says 'add evals', 'build evals', 'set up evals', 'wire LangSmith evals', 'how do I know if my AI is working', 'test my AI output', 'add LangSmith evals'."
---

# /4-Build-Evals

You evaluate how well your AI feature is working — and you'll do it in a way that catches problems before users do.

**What is an eval?** Think of it like a test suite for your AI. Instead of checking if a button works, you check if the AI gives good answers. LangSmith is the platform that stores these tests, runs them, and shows you a score.

**Why does this matter?** AI outputs are not deterministic — the same input can produce different outputs. Evals give you a repeatable, measurable way to know if your AI got better or worse when you changed something.

---

## Pre-flight

- Read `.claude/progress.md` (last 5 entries) and `.claude/context.md` if present
- Call `project-state-detector`; if mode is off-pattern for this skill, surface a one-line warning (do NOT block)

---

## Step 1: Find and understand your AI code

Run `stack-detector` to find what AI packages are installed, then search for the code that calls the AI.

**Look for these files/patterns:**
- Any file importing `@anthropic-ai/sdk`, `openai`, `langchain`, or `ai` (Vercel AI SDK)
- Functions named `generate`, `chat`, `complete`, `ask`, `summarize`, `classify`, `extract`
- API routes that call an LLM (usually `app/api/`, `pages/api/`, or `routes/`)
- Also check: is `langsmith` already installed? Is `LANGCHAIN_TRACING_V2` in the env vars?

**What to capture:**
1. What is the AI being asked to do? (classify, extract, summarize, answer questions, generate text, etc.)
2. What does the input look like? (a user message, a document, a structured form, etc.)
3. What does the output look like? (a category label, a JSON object, a paragraph of text, etc.)
4. Does it run repeatedly on similar inputs, or is it one-off per user session?

Share what you found. If the codebase has no AI code yet, stop here — run `/add-ai` first.

---

## Step 2: Decide if an eval is worth building

Not every AI feature benefits from a formal eval. Answer these questions:

| Question | If YES → | If NO → |
|---|---|---|
| Does the AI run the same type of task repeatedly? | Evals are worth building | Skip — one-off tasks don't need evals |
| Is there a way to tell if the output is good or bad? | Evals are possible | Skip — if "good" can't be defined, evals won't help |
| Will you change the prompt or model in the future? | Evals protect against regressions | Skip if this is a one-time throwaway |
| Do you have at least 5 example inputs you can test? | Evals can run now | Start collecting examples first |

**Use the decision table below to match your AI feature to an eval approach:**

| What your AI does | Best eval type | Effort |
|---|---|---|
| Classifies text into fixed categories | Rule-based (exact match) | Low |
| Extracts fields from documents | Rule-based (field match) | Low |
| Generates structured output (JSON, CSV) | Rule-based (schema check) | Low |
| Answers questions from a knowledge base | LLM-as-judge (accuracy) | Medium |
| Summarizes documents | LLM-as-judge (coverage, conciseness) | Medium |
| Multi-turn conversation / customer support | LLM-as-judge (helpfulness, tone) | Medium |
| Creative writing, open-ended generation | LLM-as-judge (rubric) or Human | High |
| Real-time autocomplete | Skip — use product metrics instead | — |
| One-off user request (no repetition) | Skip — run manually | — |

**If the answer is "skip":** Tell the user why and suggest the right alternative (product analytics via PostHog, manual spot-checking, or A/B testing). Do not force an eval onto a use case that doesn't need one.

**If the answer is "build":** Continue to Step 3.

---

## Step 3: Set up LangSmith (if not already done)

**If LangSmith is already installed and configured** (you see `langsmith` in `package.json` and `LANGCHAIN_API_KEY` in env), skip to Step 4.

**If not, walk through this setup:**

### 3a. Create a LangSmith account

1. Go to [smith.langchain.com](https://smith.langchain.com) and sign up (it's free for small usage)
2. Once logged in, click your profile icon → **Settings** → **API Keys**
3. Click **Create API Key** → copy it (you only see it once)

### 3b. Install the SDK

```bash
# TypeScript/Node
npm install langsmith

# Python
pip install langsmith
```

### 3c. Add environment variables

Add these to your `.env` or `.env.local` file (never commit this file):

```
LANGCHAIN_TRACING_V2=true
LANGCHAIN_API_KEY=ls__...         ← paste your key here
LANGCHAIN_PROJECT=my-app-evals    ← pick any name for this project
```

**Restart your dev server after adding env vars** — they are only read at startup.

### 3d. Wire tracing on your AI function

Before evals can work, LangSmith needs to see your AI calls. Wrap your existing AI function:

→ See [langsmith-eval-patterns.md](references/langsmith-eval-patterns.md#tracing-wrapper) for TypeScript and Python wrappers.

Make one real call to your AI feature, then open [smith.langchain.com](https://smith.langchain.com) → your project → **Traces**. You should see a new trace appear within 30 seconds. **Don't continue until you see a trace.** If you don't see one, check the troubleshooting section at the bottom.

---

## Step 4: Build your dataset

A dataset is a list of example inputs (and optionally expected outputs) that you want to test your AI against. Think of it as your "golden examples."

**Rules for a good dataset:**
- Minimum 5 examples — a score from 1 example is meaningless
- Include happy-path examples (typical inputs that should work well)
- Include edge cases (tricky, unusual, or borderline inputs)
- Include at least 1 example that should produce a "bad" or "I don't know" output (to verify the AI doesn't hallucinate)

**How to create examples:**

Option A — Write them yourself (easiest to start):
→ See [langsmith-eval-patterns.md](references/langsmith-eval-patterns.md#dataset-creation) for the inline dataset script.

Option B — Pull from real traces (best after you have real user traffic):
In LangSmith → **Traces** → click a trace → **Add to Dataset**. This is the fastest way to build a dataset from real usage.

**Ask the user:** "Do you have 5+ example inputs you can provide right now? If yes, walk me through them and I'll format them into the dataset script. If no, let's start with Option B — make a few real calls to your AI first, then pull from those traces."

---

## Step 5: Pick and build your evaluator

An evaluator is the function that scores each output. Pick the type that matches your decision from Step 2.

### Option A: Rule-based evaluator

Best for: classification, extraction, JSON output, anything with a clear right/wrong answer.

How it works: a regular function checks the output — no AI needed, runs instantly, costs nothing.

→ See [langsmith-eval-patterns.md](references/langsmith-eval-patterns.md#rule-based-evaluator) for exact match, field match, and JSON schema evaluators.

### Option B: LLM-as-judge evaluator

Best for: summarization, Q&A, helpfulness, tone, anything where "good" requires judgment.

How it works: a second Claude call reads the input + output and returns a score (0–1) with a reason.

**Before writing the evaluator, define your rubric.** Ask the user:
> "What makes a response good? List 3–5 specific criteria. For example: 'accurate', 'under 100 words', 'professional tone', 'cites the source', 'does not make up information'."

→ See [langsmith-eval-patterns.md](references/langsmith-eval-patterns.md#llm-as-judge-evaluator) for the Claude-based judge pattern.

### Option C: Human evaluation

Best for: creative output, high-stakes decisions, when you genuinely can't define "good" in code.

How it works: LangSmith surfaces each output in a review UI where you or a teammate clicks thumbs up/down or leaves a score.

This doesn't require writing any evaluator code. In LangSmith → **Datasets** → your dataset → **Annotate** to start a human review queue.

---

## Step 6: Wire and run the eval script

Create a script file at `scripts/run-evals.ts` (or `scripts/run_evals.py` for Python).

→ See [langsmith-eval-patterns.md](references/langsmith-eval-patterns.md#run-script) for the full runnable template.

Run it:

```bash
# TypeScript
npx tsx scripts/run-evals.ts

# Python
python scripts/run_evals.py
```

Watch for output like:
```
Experiment "baseline-20240609" complete
  Scores: { contains_expected: 0.83, ... }
```

---

## Step 7: Verify in LangSmith

1. Open [smith.langchain.com](https://smith.langchain.com) → your project
2. Click **Datasets & Testing** → find your dataset
3. Click **Experiments** → you should see your run with a score per evaluator
4. Click any row to see the individual input → output → score breakdown

**What a healthy eval looks like:**
- Score ≥ 0.8 on your primary evaluator = your AI is doing well
- Score between 0.5–0.8 = investigate the failing examples
- Score < 0.5 = your prompt needs work before you ship

**Save the eval as a regression baseline.** Next time you change the prompt or upgrade the model, re-run this script. If the score drops, you know you broke something.

---

## Post-flight

- Append to `.claude/progress.md`: timestamp, skill name, eval type chosen, dataset size, baseline score, suggested next step
- If `.claude/progress.md` is missing, create it with a header first

---

## Rules

- **At least 5 examples before running** — fewer than 5 makes scores statistically meaningless.
- **One evaluator per concern** — don't bundle "accurate AND concise AND professional" into one score. Three separate evaluators make failures easier to debug.
- **Never eval on production data directly** — use a copy or a dedicated test dataset. Evals can produce unexpected outputs.
- **LLM-as-judge uses a different model** — never use the same model you're evaluating as the judge. Use Haiku to judge Sonnet, or Sonnet to judge Haiku. Otherwise the judge is biased toward its own output style.
- **No hardcoded API keys** — always `process.env.LANGCHAIN_API_KEY`.
- **Re-run evals before any prompt change** — establish a score baseline first, then compare after the change.

---

## If Something Goes Wrong

- **No traces appearing in LangSmith** — confirm `LANGCHAIN_TRACING_V2=true` AND `LANGCHAIN_API_KEY` are both set; restart the server. Events can take 30–60 seconds to appear.
- **`ls.createDataset` throws "dataset already exists"** — add `if_exists: 'skip'` (Python) or check before creating (TS). Dataset names must be unique per project.
- **LLM-as-judge always returns 1.0** — your rubric is too easy. Make the criteria more specific and add edge cases that are genuinely hard.
- **Score varies wildly between runs** — your dataset is too small (add more examples) or the LLM temperature is too high (set `temperature: 0` for the evaluator call).
- **`npx tsx` not found** — install with `npm install -D tsx`.
- **Python `langsmith` import error** — confirm you're in the right virtualenv: `pip install langsmith anthropic`.

---

→ See [langsmith-eval-patterns.md](references/langsmith-eval-patterns.md) for all code patterns (TypeScript and Python): tracing wrapper, dataset creation, rule-based evaluator, LLM-as-judge evaluator, and the full run script.
