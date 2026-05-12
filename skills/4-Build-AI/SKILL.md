---
name: add-ai
description: MUST BE USED when adding AI / LLM capabilities to a project. Runs model-selector to pick the right Claude model tier based on the use case (Haiku for real-time/image-gen orchestration, Sonnet for most features, Opus for deep reasoning), wires Anthropic SDK with prompt caching, and sets up LangSmith tracing + evals. Handles chat, RAG, tool use, document analysis, and image analysis. Trigger on `/add-ai`, "add AI", "add Claude", "wire LLM", "add chatbot", "add AI feature", "add evals", "wire LangSmith".
---

# /add-ai

You add AI capabilities to the current project: model selection via `model-selector`, Anthropic SDK wiring, prompt caching, and LangSmith tracing + evals.

## Pre-flight

- Read `.claude/progress.md` (last 5 entries) and `.claude/context.md` if present
- Call `project-state-detector`; if mode is off-pattern for this skill, surface a one-line warning (do NOT block)

## Post-flight

- Append to `.claude/progress.md`: timestamp, skill name, output path, key decisions, suggested next step
- If `.claude/progress.md` is missing, create it with a header first

## Important

- Run `model-selector` first — do not choose a model tier by default; the use case determines cost and latency tradeoffs.
- Set `ANTHROPIC_API_KEY` in the environment before testing; the SDK fails silently or throws unhelpful errors without it.
- Enable prompt caching from the start — retrofitting it later on high-volume features is error-prone and costly.

## When to Use vs. Vercel AI SDK

- **Use this skill (Anthropic SDK directly)** when you need LangSmith evals, full tool use control, prompt caching, extended thinking, or the Anthropic Files API.
- **Extend Vercel AI SDK** (`ai` + `@ai-sdk/anthropic`) when the project already uses it for streaming UI components (`useChat`, `useCompletion`). Check `_stack-preferences.md`.

## Procedure

### Step 1: Detect

In parallel:
- `stack-detector` — language, framework, existing AI packages (`@anthropic-ai/sdk`, `ai`, `langchain`, `openai`)
- `model-selector` — classify use case and get model recommendation (pass the stack-detector output in the prompt)

Read `_stack-preferences.md`.

### Step 2: Determine mode

| Detected | Action |
|---|---|
| No AI wiring | Install Anthropic SDK fresh |
| `@anthropic-ai/sdk` present | Extend it — don't reinstall |
| Vercel AI SDK (`ai`) present | Ask: extend with `@ai-sdk/anthropic` OR add direct Anthropic SDK alongside |
| `openai` present | Add Anthropic alongside; note the duplication |
| LangChain present | Wire Anthropic as the LLM provider; LangSmith tracing is already available |

### Step 3: Confirm feature scope

Ask:
> What should this AI integration do?
> 1. **Chat / conversational** — multi-turn chat with message history
> 2. **Single-turn generation** — one-shot text, JSON, or structured output
> 3. **Document analysis** — summarize, extract, or Q&A over uploaded documents
> 4. **Agentic / tool use** — Claude calls tools to complete multi-step tasks
> 5. **RAG** — retrieval-augmented generation over a knowledge base
> 6. **Image analysis** — describe or extract information from images
> 7. **Code assistant** — code generation, review, or explanation

### Step 4: Plan

Show concrete file changes before writing any code:
- SDK installation + env var list
- AI client module (`lib/ai.ts` or `lib/ai.py`)
- The primary feature route / function
- LangSmith tracing wrapper
- Eval scaffold (dataset + at least one evaluator)

### Step 5: Execute

Wire per the patterns below. Use the model constant from `model-selector`'s `AI REQUIREMENTS PROFILE`.

### Step 6: Wire LangSmith

Always wire LangSmith tracing unless the user explicitly skips. Wire evals if the user confirms golden examples exist or can be created now.

### Step 7: Verify

- Call the AI endpoint; confirm a real response is returned
- Check LangSmith dashboard → Traces for the run
- If streaming: confirm tokens arrive incrementally in the UI
- If tool use: confirm tool invocations appear in the LangSmith trace

---

→ See [anthropic-sdk-patterns.md](references/anthropic-sdk-patterns.md) for implementation patterns (TypeScript and Python clients with prompt caching, streaming, tool use/agentic loop, structured output, LangSmith tracing, and eval scaffolding).

## Model Quick Reference (from model-selector)

| Use Case | Model | Notes |
|---|---|---|
| Image-gen app (Claude = orchestrator only) | `claude-haiku-4-5` | Claude's role is minimal — use cheapest |
| Real-time autocomplete, live suggestions | `claude-haiku-4-5` | Latency-first |
| Simple classification, extraction, translation | `claude-haiku-4-5` | High-volume, cost-sensitive |
| Chat, Q&A, RAG, code, most features | `claude-sonnet-4-6` | Best balance — the default |
| Multi-step agents, complex long documents | `claude-sonnet-4-6` | |
| Legal/medical analysis, hard reasoning | `claude-opus-4-7` | Max capability |
| Extended thinking required | `claude-opus-4-7` | Enable `thinking` parameter |

---

## Rules

- **Always wire prompt caching** for system prompts that repeat across calls — reduces latency and cost.
- **Always add LangSmith tracing** unless the user explicitly skips — even a prototype benefits from observability.
- **One model constant, not a switch** — pick one model per use case; don't add model-selection logic in app code.
- **Claude cannot generate images** — for image generation, DALL-E 3, Stability AI, or Replicate are needed. Claude analyzes images and orchestrates generation calls.
- **No hardcoded API keys** — always `process.env.ANTHROPIC_API_KEY`.
- **For streaming**: use Server-Sent Events in Next.js API routes; never buffer a streaming response in memory before returning it.
- **For evals**: create a dataset with at least 5 golden examples before running. A 1-example eval score is meaningless.

## If Something Goes Wrong

- **`ANTHROPIC_API_KEY` not recognized** — confirm the key is in `.env.local` (Next.js) or `.env` and that the server was restarted after the key was added.
- **model-selector agent fails** — run it again with explicit context about the use case; if it still fails, use Sonnet as the safe default and note the deviation.
- **LangSmith events not appearing** — verify `LANGCHAIN_API_KEY`, `LANGCHAIN_TRACING_V2=true`, and `LANGCHAIN_PROJECT` are all set; events can take 30–60 seconds to appear.
- **Prompt caching not hitting** — confirm the cached block is at least 1024 tokens and that the cache-control header is set correctly on the `system` message.