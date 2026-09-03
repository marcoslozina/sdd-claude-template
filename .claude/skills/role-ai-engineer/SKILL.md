---
name: role-ai-engineer
description: Building on the Anthropic Claude API — SDK setup, model selection, prompt caching, tool use, structured outputs, streaming, the Batch API, multi-turn chat, error handling and token counting. Use when writing code against `anthropic` / `@anthropic-ai/sdk`, choosing between Opus/Sonnet/Haiku, or cutting LLM latency and cost.
---

# Skill: AI Engineer (Claude API — Official Anthropic)

> Based exclusively on the official documentation: https://docs.anthropic.com

## SDK Setup

```python
# Python
import anthropic
client = anthropic.Anthropic()  # reads ANTHROPIC_API_KEY from the env
```

```typescript
// TypeScript
import Anthropic from '@anthropic-ai/sdk'
const client = new Anthropic() // reads ANTHROPIC_API_KEY from the env
```

**Never hardcode the API key. Always an env var.**

---

## Available models

| Model | ID | When to use |
|--------|----|-------------|
| Claude Opus 5 | `claude-opus-5` | Complex reasoning, agents, hard tasks |
| Claude Sonnet 5 | `claude-sonnet-5` | Cost/capability balance, general use |
| Claude Haiku 4.5 | `claude-haiku-4-5-20251001` | Simple tasks, low latency, minimum cost |

**Rule:** start with Sonnet. Move up to Opus if quality isn't good enough. Move down to Haiku if cost matters.

---

## Basic call

```python
response = client.messages.create(
    model="claude-sonnet-5",
    max_tokens=1024,
    system="You are a technical assistant specialized in Python.",
    messages=[
        {"role": "user", "content": "How do I implement a thread-safe singleton?"}
    ]
)
print(response.content[0].text)
```

---

## Prompt Caching (cut costs by up to 90%)

Source: https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching

```python
# Explicit cache on long content (docs, large context)
response = client.messages.create(
    model="claude-sonnet-5",
    max_tokens=1024,
    system=[
        {
            "type": "text",
            "text": "You are an expert on this codebase.",
        },
        {
            "type": "text",
            "text": "<codebase>" + full_codebase_content + "</codebase>",
            "cache_control": {"type": "ephemeral"}  # TTL: 5 min
        }
    ],
    messages=[{"role": "user", "content": user_question}]
)
```

**Minimum to cache:** 4096 tokens (Opus/Sonnet), 2048 (Haiku)
**TTL:** 5 minutes (default) or 1 hour (double write cost, useful for long sessions)
**Cache hit pricing:** ~90% cheaper than regular input

---

## Tool Use (function calling)

Source: https://docs.anthropic.com/en/docs/build-with-claude/tool-use

```python
tools = [
    {
        "name": "search_database",
        "description": "Searches database records by criteria",
        "input_schema": {
            "type": "object",
            "properties": {
                "query": {"type": "string", "description": "Search criteria"},
                "limit": {"type": "integer", "default": 10}
            },
            "required": ["query"]
        }
    }
]

messages = [{"role": "user", "content": "Find all users from Argentina"}]

while True:
    response = client.messages.create(
        model="claude-sonnet-5",
        max_tokens=1024,
        tools=tools,
        messages=messages
    )

    if response.stop_reason == "end_turn":
        print(response.content[0].text)
        break

    # Process tool use
    for block in response.content:
        if block.type == "tool_use":
            result = execute_tool(block.name, block.input)
            messages.append({"role": "assistant", "content": response.content})
            messages.append({
                "role": "user",
                "content": [{
                    "type": "tool_result",
                    "tool_use_id": block.id,
                    "content": str(result)
                }]
            })
```

---

## Structured Outputs

```python
from pydantic import BaseModel

class ExtractedData(BaseModel):
    name: str
    email: str
    intent: str
    priority: int

response = client.messages.parse(
    model="claude-sonnet-5",
    max_tokens=512,
    messages=[{"role": "user", "content": raw_text}],
    output_format=ExtractedData,
)
data = response.parsed_output  # guaranteed ExtractedData type
```

**Limitations:**
- No recursive schemas or external `$ref`
- No numeric constraints (`minimum`/`maximum`)
- Incompatible with citations and message prefilling

---

## Streaming

```python
with client.messages.stream(
    model="claude-sonnet-5",
    max_tokens=2048,
    messages=[{"role": "user", "content": prompt}]
) as stream:
    for text in stream.text_stream:
        print(text, end="", flush=True)

# Or to process the complete final message:
final = stream.get_final_message()
```

**Use streaming when:** max_tokens > 1000, you need a responsive UX, long tasks.

---

## Batch API (50% discount)

```python
from anthropic.types.message_create_params import MessageCreateParamsNonStreaming
from anthropic.types.messages.batch_create_params import Request

# Create batch (up to 100k requests or 256MB)
batch = client.messages.batches.create(
    requests=[
        Request(
            custom_id=f"item-{i}",
            params=MessageCreateParamsNonStreaming(
                model="claude-haiku-4-5-20251001",
                max_tokens=512,
                messages=[{"role": "user", "content": text}]
            )
        )
        for i, text in enumerate(texts)
    ]
)

# Polling (completes in < 1h, 24h maximum)
import time
while True:
    batch = client.messages.batches.retrieve(batch.id)
    if batch.processing_status == "ended":
        break
    time.sleep(60)

# Process results (streaming so we don't load everything into memory)
for result in client.messages.batches.results(batch.id):
    if result.result.type == "succeeded":
        handle(result.custom_id, result.result.message.content[0].text)
```

**When to use batch:** bulk processing with no urgency, classification, dataset analysis.

---

## Multi-turn conversation

```python
messages = []

def chat(user_input: str) -> str:
    messages.append({"role": "user", "content": user_input})

    response = client.messages.create(
        model="claude-sonnet-5",
        max_tokens=1024,
        messages=messages
    )

    assistant_reply = response.content[0].text
    messages.append({"role": "assistant", "content": assistant_reply})
    return assistant_reply
```

**Watch out:** the history grows. For long conversations: summarize periodically.

---

## Error handling

```python
import anthropic

try:
    response = client.messages.create(...)
except anthropic.RateLimitError:
    # Wait and retry with exponential backoff
    time.sleep(60)
except anthropic.APIConnectionError:
    # Network problem, retry
    pass
except anthropic.APIStatusError as e:
    print(f"API error {e.status_code}: {e.message}")
```

---

## Estimating cost before running

```python
token_count = client.messages.count_tokens(
    model="claude-sonnet-5",
    messages=[{"role": "user", "content": prompt}]
)
print(f"Estimated tokens: {token_count.input_tokens}")
```

---

## Common AI Engineering decisions

Apply the decision protocol from CLAUDE.md when facing:
- **Model:** Opus vs Sonnet vs Haiku depending on complexity and cost
- **Caching:** automatic vs explicit breakpoints
- **Tool use:** how many tools per call, granularity
- **Streaming vs sync:** depending on required latency
- **Batch vs real-time:** depending on how urgent the processing is
