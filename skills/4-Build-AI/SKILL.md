---
name: 4-Build-AI
description: MUST BE USED when adding AI / LLM capabilities to a project. Runs model-selector to pick the right Claude model tier based on the use case (Haiku for real-time/image-gen orchestration, Sonnet for most features, Opus for deep reasoning), wires Anthropic SDK with prompt caching, and sets up LangSmith tracing + evals. Handles chat, RAG, tool use, document analysis, and image analysis. Trigger on `/add-ai`, "add AI", "add Claude", "wire LLM", "add chatbot", "add AI feature", "add evals", "wire LangSmith".
---

# /add-ai

You add AI capabilities to the current project: model selection via `model-selector`, Anthropic SDK wiring, prompt caching, and LangSmith tracing + evals.

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

## Anthropic SDK Patterns

### Install

```bash
# TypeScript/Node
npm install @anthropic-ai/sdk

# Python
pip install anthropic
```

### Env vars

```
ANTHROPIC_API_KEY=sk-ant-...
LANGCHAIN_TRACING_V2=true
LANGCHAIN_API_KEY=ls__...
LANGCHAIN_PROJECT=my-project-name
```

### TypeScript: basic client with prompt caching

```typescript
// lib/ai.ts
import Anthropic from '@anthropic-ai/sdk';

export const ai = new Anthropic({ apiKey: process.env.ANTHROPIC_API_KEY! });

export async function generate(userMessage: string, systemPrompt: string): Promise<string> {
  const response = await ai.messages.create({
    model: 'claude-sonnet-4-6',         // ← set from model-selector output
    max_tokens: 1024,
    system: [
      {
        type: 'text',
        text: systemPrompt,
        cache_control: { type: 'ephemeral' }, // cache repeated system prompts
      },
    ],
    messages: [{ role: 'user', content: userMessage }],
  });
  const block = response.content[0];
  return block.type === 'text' ? block.text : '';
}
```

### TypeScript: streaming (Next.js App Router)

```typescript
// app/api/chat/route.ts
import { ai } from '@/lib/ai';

export async function POST(req: Request) {
  const { message, systemPrompt } = await req.json();

  const stream = await ai.messages.stream({
    model: 'claude-sonnet-4-6',
    max_tokens: 2048,
    system: systemPrompt,
    messages: [{ role: 'user', content: message }],
  });

  const encoder = new TextEncoder();
  const readable = new ReadableStream({
    async start(controller) {
      for await (const chunk of stream) {
        if (chunk.type === 'content_block_delta' && chunk.delta.type === 'text_delta') {
          controller.enqueue(encoder.encode(`data: ${JSON.stringify({ text: chunk.delta.text })}\n\n`));
        }
      }
      controller.enqueue(encoder.encode('data: [DONE]\n\n'));
      controller.close();
    },
  });

  return new Response(readable, {
    headers: { 'Content-Type': 'text/event-stream', 'Cache-Control': 'no-cache' },
  });
}
```

### TypeScript: tool use (agentic loop)

```typescript
import Anthropic from '@anthropic-ai/sdk';
import { ai } from '@/lib/ai';

const tools: Anthropic.Tool[] = [
  {
    name: 'search_database',
    description: 'Search the product database by keyword',
    input_schema: {
      type: 'object',
      properties: { query: { type: 'string', description: 'Search query' } },
      required: ['query'],
    },
  },
];

export async function runAgent(userMessage: string): Promise<string> {
  const messages: Anthropic.MessageParam[] = [{ role: 'user', content: userMessage }];

  while (true) {
    const response = await ai.messages.create({
      model: 'claude-sonnet-4-6',
      max_tokens: 4096,
      tools,
      messages,
    });

    if (response.stop_reason === 'end_turn') {
      const text = response.content.find(b => b.type === 'text');
      return text?.type === 'text' ? text.text : '';
    }

    messages.push({ role: 'assistant', content: response.content });
    const results: Anthropic.ToolResultBlockParam[] = [];
    for (const block of response.content) {
      if (block.type === 'tool_use') {
        const result = await dispatchTool(block.name, block.input as Record<string, unknown>);
        results.push({ type: 'tool_result', tool_use_id: block.id, content: result });
      }
    }
    messages.push({ role: 'user', content: results });
  }
}
```

### TypeScript: structured output (JSON)

```typescript
export async function extractStructured<T>(text: string, schema: string): Promise<T> {
  const response = await ai.messages.create({
    model: 'claude-haiku-4-5',
    max_tokens: 512,
    messages: [{
      role: 'user',
      content: `Extract the following from this text and return ONLY valid JSON matching this schema: ${schema}\n\nText: ${text}`,
    }],
  });
  const content = response.content[0];
  if (content.type !== 'text') throw new Error('Unexpected response type');
  return JSON.parse(content.text) as T;
}
```

### Python: basic client with prompt caching

```python
import anthropic

client = anthropic.Anthropic()

def generate(user_message: str, system_prompt: str) -> str:
    response = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=1024,
        system=[{"type": "text", "text": system_prompt, "cache_control": {"type": "ephemeral"}}],
        messages=[{"role": "user", "content": user_message}],
    )
    return response.content[0].text
```

---

## LangSmith Tracing & Evals

### Install

```bash
# TypeScript
npm install langsmith

# Python
pip install langsmith
```

### TypeScript: traceable wrapper

```typescript
// lib/ai.ts — wrap the generate function
import { traceable } from 'langsmith/traceable';

export const generateTraceable = traceable(
  async (userMessage: string, systemPrompt: string): Promise<string> =>
    generate(userMessage, systemPrompt),
  { name: 'generate', tags: ['production'] },
);
```

### TypeScript: create dataset and run evals

```typescript
// scripts/run-evals.ts
import { Client } from 'langsmith';
import { evaluate } from 'langsmith/evaluation';
import { generateTraceable } from '@/lib/ai';

const ls = new Client();
const SYSTEM_PROMPT = 'You are a helpful assistant.'; // replace with real prompt

async function createDataset(name: string) {
  const dataset = await ls.createDataset(name, {
    description: 'Golden examples for regression testing',
  });
  await ls.createExamples([
    // Replace with real golden examples
    {
      inputs: { user_message: 'Summarize: "The sky is blue."' },
      outputs: { expected: 'blue sky' },
      datasetId: dataset.id,
    },
  ]);
  return dataset;
}

function containsExpected({ outputs, referenceOutputs }: {
  outputs: Record<string, string>;
  referenceOutputs?: Record<string, string>;
}) {
  const actual = (outputs.result ?? '').toLowerCase();
  const expected = (referenceOutputs?.expected ?? '').toLowerCase();
  return { key: 'contains_expected', score: actual.includes(expected) ? 1 : 0 };
}

async function runEvals(datasetName: string) {
  await evaluate(
    (inputs: { user_message: string }) =>
      generateTraceable(inputs.user_message, SYSTEM_PROMPT).then(result => ({ result })),
    { data: datasetName, evaluators: [containsExpected], experimentPrefix: 'baseline' },
  );
}
```

### Python: traceable + evals

```python
from langsmith import traceable, Client
from langsmith.evaluation import evaluate

ls = Client()

@traceable(name="generate", tags=["production"])
def generate_traced(user_message: str, system_prompt: str) -> str:
    return generate(user_message, system_prompt)

def contains_expected(outputs: dict, reference_outputs: dict) -> dict:
    actual = outputs.get("result", "").lower()
    expected = reference_outputs.get("expected", "").lower()
    return {"key": "contains_expected", "score": int(expected in actual)}

def run_evals(dataset_name: str, system_prompt: str):
    evaluate(
        lambda inputs: {"result": generate_traced(inputs["user_message"], system_prompt)},
        data=dataset_name,
        evaluators=[contains_expected],
        experiment_prefix="baseline",
    )
```

---

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
