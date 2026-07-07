---
title: Vercel AI SDK
description: Capture Vercel AI SDK OpenTelemetry spans in Logfire from Node.js and Next.js applications.
---

# Vercel AI SDK

The Vercel AI SDK can emit OpenTelemetry spans for model calls, tools, token usage, and streaming operations. Logfire can receive those spans through either `@pydantic/logfire-node` in Node.js scripts or `@vercel/otel` in Next.js applications.

AI SDK 7 uses `@ai-sdk/otel` and the GenAI semantic conventions. AI SDK 5 and 6 use the older `experimental_telemetry` option.

## Node.js Scripts With AI SDK 7

```bash
npm install @pydantic/logfire-node ai @ai-sdk/otel @ai-sdk/openai
```

Replace `@ai-sdk/openai` with the provider package you use, such as `@ai-sdk/anthropic` or `@ai-sdk/google`.

Configure Logfire before registering the AI SDK telemetry integration:

```ts title="instrumentation.ts"
import { OpenTelemetry } from '@ai-sdk/otel'
import * as logfire from '@pydantic/logfire-node'
import { registerTelemetry } from 'ai'

logfire.configure({
  serviceName: 'ai-worker',
})

registerTelemetry(new OpenTelemetry())
```

Import your instrumentation file before calling the AI SDK. Once the telemetry integration is registered, AI SDK 7 emits telemetry by default:

```ts
import './instrumentation.ts'
import { openai } from '@ai-sdk/openai'
import { generateText } from 'ai'

const result = await generateText({
  model: openai('gpt-4.1-mini'),
  prompt: 'Write a short haiku about traces.',
  telemetry: {
    functionId: 'haiku-agent',
  },
})

console.log(result.text)
```

## Next.js With AI SDK 7

In Next.js, configure `@vercel/otel` as shown in [Next.js](nextjs.md), then register `@ai-sdk/otel` in the same `instrumentation.ts` file. The file must live in the project root, or in `src` if your Next.js app uses `src`.

```bash
npm install @vercel/otel @opentelemetry/api ai @ai-sdk/otel @ai-sdk/openai
```

```ts title="instrumentation.ts"
import { OpenTelemetry } from '@ai-sdk/otel'
import { registerOTel } from '@vercel/otel'
import { registerTelemetry } from 'ai'

export function register() {
  registerOTel({ serviceName: 'nextjs-ai-app' })
  registerTelemetry(new OpenTelemetry())
}
```

Then call the AI SDK normally:

```ts
const result = await generateText({
  model,
  prompt: 'Write a short haiku about traces.',
  telemetry: {
    functionId: 'haiku-agent',
  },
})
```

## Example: AI SDK 7 Text Generation With Tools

```ts
import { openai } from '@ai-sdk/openai'
import { generateText, tool } from 'ai'
import { z } from 'zod'

const result = await generateText({
  model: openai('gpt-4.1-mini'),
  telemetry: {
    functionId: 'weather-agent',
  },
  tools: {
    weather: tool({
      description: 'Get the weather in a location',
      inputSchema: z.object({
        location: z.string().describe('The location to get the weather for'),
      }),
      execute: async ({ location }) => ({
        location,
        temperature: 72,
      }),
    }),
  },
  prompt: 'What is the weather in San Francisco?',
})

console.log(result.text)
```

## What You Will See

With AI SDK 7 and `@ai-sdk/otel`, Logfire captures GenAI semantic spans such as:

- `invoke_agent <model>` root spans
- `step <n>` spans for multi-step calls
- `chat <model>` spans for provider requests
- `execute_tool <toolName>` spans for tool calls

Provider names are emitted as `gen_ai.provider.name`. Request models are emitted as `gen_ai.request.model`, and some providers also emit `gen_ai.response.model`. Logfire understands these fields alongside the older `gen_ai.system` attribute.

Depending on the AI SDK provider and call type, traces can include:

- model and provider details
- input and output token usage
- timing information
- tool call arguments and results
- prompts and responses when the AI SDK emits them

## AI SDK 5 and 6

For AI SDK 5 and 6, install the same core packages without `@ai-sdk/otel` and enable telemetry on each call with `experimental_telemetry.isEnabled`:

```bash
npm install @pydantic/logfire-node ai @ai-sdk/openai
```

```ts
const result = await generateText({
  model,
  prompt: 'Write a short haiku about traces.',
  experimental_telemetry: { isEnabled: true },
})
```

Legacy AI SDK spans usually use names such as `ai.generateText`, `ai.generateText.doGenerate`, and `ai.toolCall`, with provider information in `gen_ai.system`.

## Metadata

In AI SDK 7, use `telemetry.functionId` to identify the operation. This is emitted as `gen_ai.agent.name`; it no longer controls the default GenAI span names, which include the operation and model or tool name.

Use `runtimeContext` with `telemetry.includeRuntimeContext` to include selected custom fields in telemetry:

```ts
await generateText({
  model,
  prompt,
  runtimeContext: {
    tenant: 'acme',
    userId: 'user_123',
  },
  telemetry: {
    functionId: 'support-reply',
    includeRuntimeContext: {
      tenant: true,
    },
  },
})
```

For AI SDK 5 and 6, use `experimental_telemetry.functionId` and `experimental_telemetry.metadata`.
