---
title: "Anthropic SDK Patterns"
skill: "4-Build-AI"
---

# Anthropic SDK Patterns

## Install

```bash
# TypeScript/Node
npm install @anthropic-ai/sdk

# Python
pip install anthropic
```

## Env vars

```
ANTHROPIC_API_KEY=sk-ant-...
LANGCHAIN_TRACING_V2=true
LANGCHAIN_API_KEY=ls__...
LANGCHAIN_PROJECT=my-project-name
```

## TypeScript: basic client with prompt caching

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

## TypeScript: streaming (Next.js App Router)

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

## TypeScript: tool use (agentic loop)

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

## TypeScript: structured output (JSON)

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

## Python: basic client with prompt caching

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

# LangSmith Tracing & Evals

## Install

```bash
# TypeScript
npm install langsmith

# Python
pip install langsmith
```

## TypeScript: traceable wrapper

```typescript
// lib/ai.ts — wrap the generate function
import { traceable } from 'langsmith/traceable';

export const generateTraceable = traceable(
  async (userMessage: string, systemPrompt: string): Promise<string> =>
    generate(userMessage, systemPrompt),
  { name: 'generate', tags: ['production'] },
);
```

## TypeScript: create dataset and run evals

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

## Python: traceable + evals

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
