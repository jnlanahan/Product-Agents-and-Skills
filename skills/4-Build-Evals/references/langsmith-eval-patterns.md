---
title: "LangSmith Eval Patterns"
skill: "4-Build-Evals"
---

# LangSmith Eval Patterns

## Install

```bash
# TypeScript/Node
npm install langsmith
npm install -D tsx   # to run .ts scripts directly

# Python
pip install langsmith anthropic
```

## Env vars

```
LANGCHAIN_TRACING_V2=true
LANGCHAIN_API_KEY=ls__...
LANGCHAIN_PROJECT=my-app-evals
ANTHROPIC_API_KEY=sk-ant-...    # needed if using LLM-as-judge
```

---

## Tracing wrapper

Wrap your existing AI function so LangSmith records every call.

### TypeScript

```typescript
// lib/ai.ts — add this import and wrap your existing function
import { traceable } from 'langsmith/traceable';

// Your existing function (unchanged):
async function generate(userMessage: string, systemPrompt: string): Promise<string> {
  // ... your existing Anthropic / OpenAI call
}

// Traced version — use this everywhere instead of generate():
export const generateTraceable = traceable(generate, { name: 'generate' });
```

### Python

```python
# lib/ai.py — wrap your existing function
from langsmith import traceable

# Your existing function (unchanged):
def generate(user_message: str, system_prompt: str) -> str:
    # ... your existing Anthropic / OpenAI call
    pass

# Traced version — use this everywhere instead of generate():
generate_traced = traceable(generate, name="generate")
```

---

## Dataset creation

### TypeScript — inline dataset

```typescript
// scripts/create-dataset.ts
import { Client } from 'langsmith';

const ls = new Client();

const DATASET_NAME = 'my-feature-evals';

const examples = [
  // Each example: inputs = what goes into the AI, outputs = what you expect back
  {
    inputs: { user_message: 'What is the return policy?' },
    outputs: { expected: 'You can return items within 30 days' },
  },
  {
    inputs: { user_message: 'How do I reset my password?' },
    outputs: { expected: 'Click Forgot Password on the login page' },
  },
  // Add at least 5 examples total
];

async function createDataset() {
  const dataset = await ls.createDataset(DATASET_NAME, {
    description: 'Golden examples for my AI feature',
  });

  await ls.createExamples({
    inputs: examples.map(e => e.inputs),
    outputs: examples.map(e => e.outputs),
    datasetId: dataset.id,
  });

  console.log(`Created dataset "${DATASET_NAME}" with ${examples.length} examples`);
}

createDataset();
```

### Python — inline dataset

```python
# scripts/create_dataset.py
from langsmith import Client

ls = Client()

DATASET_NAME = "my-feature-evals"

examples = [
    {
        "inputs": {"user_message": "What is the return policy?"},
        "outputs": {"expected": "You can return items within 30 days"},
    },
    {
        "inputs": {"user_message": "How do I reset my password?"},
        "outputs": {"expected": "Click Forgot Password on the login page"},
    },
    # Add at least 5 examples total
]

dataset = ls.create_dataset(DATASET_NAME, description="Golden examples for my AI feature")

ls.create_examples(
    inputs=[e["inputs"] for e in examples],
    outputs=[e["outputs"] for e in examples],
    dataset_id=dataset.id,
)

print(f'Created dataset "{DATASET_NAME}" with {len(examples)} examples')
```

---

## Rule-based evaluator

### TypeScript — exact match

```typescript
import { EvaluationResult } from 'langsmith/evaluation';

// Checks if the actual output contains the expected string (case-insensitive)
function containsExpected(
  { outputs, referenceOutputs }: { outputs: Record<string, string>; referenceOutputs?: Record<string, string> }
): EvaluationResult {
  const actual = (outputs.result ?? '').toLowerCase();
  const expected = (referenceOutputs?.expected ?? '').toLowerCase();
  return {
    key: 'contains_expected',
    score: actual.includes(expected) ? 1 : 0,
  };
}
```

### TypeScript — JSON schema check

```typescript
import Ajv from 'ajv';
const ajv = new Ajv();

const EXPECTED_SCHEMA = {
  type: 'object',
  properties: {
    category: { type: 'string' },
    confidence: { type: 'number' },
  },
  required: ['category', 'confidence'],
};

function validJsonSchema(
  { outputs }: { outputs: Record<string, string> }
): EvaluationResult {
  try {
    const parsed = JSON.parse(outputs.result ?? '{}');
    const valid = ajv.validate(EXPECTED_SCHEMA, parsed);
    return { key: 'valid_json_schema', score: valid ? 1 : 0 };
  } catch {
    return { key: 'valid_json_schema', score: 0 };
  }
}
```

### Python — exact match

```python
def contains_expected(outputs: dict, reference_outputs: dict) -> dict:
    actual = (outputs.get("result") or "").lower()
    expected = (reference_outputs.get("expected") or "").lower()
    return {"key": "contains_expected", "score": int(expected in actual)}
```

---

## LLM-as-judge evaluator

Use a **different model** from the one you are evaluating. If you're evaluating Sonnet, judge with Haiku (fast, cheap). If evaluating Haiku, judge with Sonnet.

### TypeScript — Claude as judge

```typescript
import Anthropic from '@anthropic-ai/sdk';

const judge = new Anthropic({ apiKey: process.env.ANTHROPIC_API_KEY! });

// Customize the RUBRIC for your specific use case
const RUBRIC = `
You are evaluating an AI assistant response. Score it on this rubric:

1 point — The response is accurate and directly answers the question.
0.5 points — The response is partially correct or vague but not wrong.
0 points — The response is incorrect, irrelevant, or makes up information.

Return ONLY a JSON object: { "score": <0, 0.5, or 1>, "reason": "<one sentence>" }
`;

async function llmJudge(
  { inputs, outputs }: { inputs: Record<string, string>; outputs: Record<string, string> }
): Promise<EvaluationResult> {
  const response = await judge.messages.create({
    model: 'claude-haiku-4-5',   // cheaper model as judge
    max_tokens: 256,
    temperature: 0,              // deterministic scoring
    system: RUBRIC,
    messages: [
      {
        role: 'user',
        content: `Question: ${inputs.user_message}\n\nResponse to evaluate: ${outputs.result}`,
      },
    ],
  });

  const text = response.content[0].type === 'text' ? response.content[0].text : '{}';
  const { score, reason } = JSON.parse(text);
  return { key: 'llm_accuracy', score, comment: reason };
}
```

### Python — Claude as judge

```python
import json
import anthropic

judge_client = anthropic.Anthropic()

# Customize this rubric for your specific use case
RUBRIC = """
You are evaluating an AI assistant response. Score it on this rubric:

1 point — The response is accurate and directly answers the question.
0.5 points — The response is partially correct or vague but not wrong.
0 points — The response is incorrect, irrelevant, or makes up information.

Return ONLY a JSON object: { "score": <0, 0.5, or 1>, "reason": "<one sentence>" }
"""

def llm_judge(inputs: dict, outputs: dict) -> dict:
    response = judge_client.messages.create(
        model="claude-haiku-4-5",   # cheaper model as judge
        max_tokens=256,
        temperature=0,              # deterministic scoring
        system=RUBRIC,
        messages=[{
            "role": "user",
            "content": f"Question: {inputs['user_message']}\n\nResponse to evaluate: {outputs['result']}",
        }],
    )
    result = json.loads(response.content[0].text)
    return {"key": "llm_accuracy", "score": result["score"], "comment": result["reason"]}
```

---

## Run script

This is the file that actually runs your eval. Create it at `scripts/run-evals.ts` (or `scripts/run_evals.py`).

### TypeScript — full run script

```typescript
// scripts/run-evals.ts
import { evaluate } from 'langsmith/evaluation';
import { generateTraceable } from '../lib/ai';   // ← your traced AI function

const DATASET_NAME = 'my-feature-evals';
const SYSTEM_PROMPT = 'You are a helpful assistant...';  // ← your actual system prompt

// Import your evaluator(s) from above
import { containsExpected } from './evaluators/contains-expected';
// import { llmJudge } from './evaluators/llm-judge';

async function runEvals() {
  const results = await evaluate(
    // This calls your AI function for each example in the dataset:
    (inputs: { user_message: string }) =>
      generateTraceable(inputs.user_message, SYSTEM_PROMPT).then(result => ({ result })),
    {
      data: DATASET_NAME,
      evaluators: [containsExpected],   // add llmJudge here too if using both
      experimentPrefix: 'baseline',     // change this when re-running after changes
    }
  );

  console.log('Eval complete. Results:');
  console.log(JSON.stringify(results.results, null, 2));
}

runEvals().catch(console.error);
```

Run it:
```bash
npx tsx scripts/run-evals.ts
```

### Python — full run script

```python
# scripts/run_evals.py
from langsmith.evaluation import evaluate
from lib.ai import generate_traced   # ← your traced AI function

DATASET_NAME = "my-feature-evals"
SYSTEM_PROMPT = "You are a helpful assistant..."  # ← your actual system prompt

# Import your evaluator(s) from above
from evaluators.contains_expected import contains_expected
# from evaluators.llm_judge import llm_judge

def run_evals():
    results = evaluate(
        # This calls your AI function for each example in the dataset:
        lambda inputs: {"result": generate_traced(inputs["user_message"], SYSTEM_PROMPT)},
        data=DATASET_NAME,
        evaluators=[contains_expected],  # add llm_judge here too if using both
        experiment_prefix="baseline",    # change this when re-running after changes
    )
    print("Eval complete.")
    print(results)

if __name__ == "__main__":
    run_evals()
```

Run it:
```bash
python scripts/run_evals.py
```

---

## Running evals before deploys (optional)

Add to your CI pipeline (`package.json`):

```json
{
  "scripts": {
    "eval": "tsx scripts/run-evals.ts"
  }
}
```

Or add to `.github/workflows/ci.yml`:

```yaml
- name: Run AI evals
  run: npm run eval
  env:
    LANGCHAIN_API_KEY: ${{ secrets.LANGCHAIN_API_KEY }}
    LANGCHAIN_TRACING_V2: "true"
    LANGCHAIN_PROJECT: my-app-evals
    ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
```

Add `LANGCHAIN_API_KEY` and `ANTHROPIC_API_KEY` to your GitHub repo secrets (Settings → Secrets → Actions).
