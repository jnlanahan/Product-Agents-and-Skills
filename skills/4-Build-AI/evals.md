# Evals: /add-ai

Binary pass/fail criteria. Grading agent: check output against each criterion and return PASS or FAIL.
For each FAIL provide one line of reason. Do not add criteria beyond what is listed.

1. `model-selector` was run before choosing a model tier — model was not chosen by default
2. Prompt caching is wired for system prompts that repeat across calls
3. `ANTHROPIC_API_KEY` is read from environment — not hardcoded anywhere in application code
4. LangSmith tracing is wired (unless user explicitly skipped with a documented reason)
5. If streaming: Server-Sent Events are used; response is not buffered in memory before returning
6. If tool use: tool invocations are confirmed visible in the LangSmith trace
7. If evals were set up: dataset contains at least 5 golden examples
8. AI client is isolated in a dedicated module (`lib/ai.ts` or equivalent) — not inline API calls in routes
9. A real API call was made and a valid response was confirmed during the verify step
10. `.claude/progress.md` was updated on completion
