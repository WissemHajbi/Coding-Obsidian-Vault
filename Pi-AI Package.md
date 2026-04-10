# 🤖 Pi-AI Package

> **pi-ai** is the unified LLM abstraction layer. It handles communication with multiple LLM providers (OpenAI, Anthropic, Google, etc.) through a single interface, enabling tool calling, streaming, and advanced features like reasoning/thinking.

---

## Overview

| Aspect | Details |
|--------|---------|
| **Purpose** | Unified multi-provider LLM API |
| **Core Types** | Message types, Content types, Tool |
| **Main Functions** | stream, complete, getModel, getModels |
| **Providers** | OpenAI, Anthropic, Google Gemini, Mistral, Bedrock, and 15+ others |

---

## Core Concepts

### 1. Unified Interface

Different LLM providers have different APIs. Pi-ai wraps them all:

- **Problem**: OpenAI uses `chat.completions`, Anthropic uses `messages`, Google uses `generateContent`, etc.
- **Solution**: Single `stream()` and `complete()` functions that work with any provider
- **Result**: Write once, swap providers by changing the model

### 2. Tool Calling (Function Calling)

Agents need to call tools. Pi-ai handles this:

- **Tool Definition**: Define tools with schema using TypeBox
- **Provider Abstraction**: Different providers format tool calls differently
- **Unified Response**: All responses are normalized to `ToolCall` objects

### 3. Streaming vs Complete

Two execution patterns:

- **`stream()`** - Real-time event stream (start → text deltas → done)
- **`complete()`** - Wait for full response (blocks until done)

---

## Key Exports

```typescript
// ✅ Do import from pi-ai
import { 
  getModel,        // Get a model by provider + name
  stream,          // Stream response events
  complete,        // Get full response
  streamSimple,    // Stream with reasoning support
  completeSimple,  // Complete with reasoning support
  Type,            // TypeBox for tool schemas
  getModels,       // List available models
  getProviders,    // List available providers
} from "@mariozechner/pi-ai";
```

---

## Core Types

#### 1. **UserMessage**
Message from the user to the LLM.

```typescript
interface UserMessage {
  role: "user";
  content: string | (TextContent | ImageContent)[];
  timestamp: number; // Unix timestamp in ms
}
```

**Examples:**
```typescript
// Simple text
{ role: "user", content: "What is 2+2?", timestamp: Date.now() }

// With images
{
  role: "user",
  content: [
    { type: "text", text: "What's in this image?" },
    { type: "image", data: "base64...", mimeType: "image/jpeg" }
  ],
  timestamp: Date.now()
}
```

---

#### 2. **AssistantMessage**
Response from the LLM.

```typescript
interface AssistantMessage {
  role: "assistant";
  content: (TextContent | ThinkingContent | ToolCall)[];
  
  // Metadata
  api: Api;               // Which API ("openai-responses", etc)
  provider: Provider;     // Which provider ("openai", "anthropic", etc)
  model: string;          // Model ID
  responseId?: string;    // Provider's response ID (if available)
  
  // Usage & stopping
  usage: Usage;           // Token counts & costs
  stopReason: StopReason; // "stop" | "length" | "toolUse" | "error" | "aborted"
  errorMessage?: string;  // If error occurred
  
  timestamp: number;      // Unix timestamp in ms
}
```

**Key Point**: An `AssistantMessage` can contain **multiple content items**:
- Text responses
- Thinking/reasoning blocks
- Tool calls (not tool results - those are ToolResultMessage)

---

#### 3. **ToolResultMessage**
Result from executing a tool that the LLM requested.

```typescript
interface ToolResultMessage<TDetails = any> {
  role: "toolResult";
  toolCallId: string;     // Which tool call this responds to
  toolName: string;       // Name of the tool
  content: (TextContent | ImageContent)[]; // Result (text or images)
  details?: TDetails;     // Optional structured result
  isError: boolean;       // True if tool execution failed
  timestamp: number;
}
```

**Pattern**:
```
1. LLM sees tool in context → generates AssistantMessage with ToolCall
2. App executes tool → creates ToolResultMessage
3. App sends ToolResultMessage back to LLM in next request
4. Loop repeats until LLM stops requesting tools
```

---

### Content Types

#### **TextContent**
Plain text response.

```typescript
interface TextContent {
  type: "text";
  text: string;
  textSignature?: string; // Provider metadata (OpenAI)
}
```

---

#### **ThinkingContent**
Extended thinking/reasoning (Claude 3.5+, OpenAI o1).

```typescript
interface ThinkingContent {
  type: "thinking";
  thinking: string;           // The actual thinking
  thinkingSignature?: string; // Provider ID for redacted thinking
  redacted?: boolean;         // True if safety-filtered
}
```

**Use Case**: Models that benefit from explicit reasoning:
```typescript
// Request thinking
streamSimple(model, context, { reasoning: "high" })

// Response includes thinking block
// [{ type: "thinking", thinking: "Let me work through this..." }, 
//  { type: "text", text: "The answer is..." }]
```

---

#### **ToolCall**
Request from LLM to execute a tool.

```typescript
interface ToolCall {
  type: "toolCall";
  id: string;                // Unique ID for this call
  name: string;              // Tool name
  arguments: Record<string, any>; // JSON args
  thoughtSignature?: string; // Google-specific (for thought reuse)
}
```

---

### Usage & Cost Tracking

```typescript
interface Usage {
  input: number;      // Input tokens
  output: number;     // Output tokens
  cacheRead: number;  // Cached tokens (prompt caching)
  cacheWrite: number; // Newly cached tokens
  totalTokens: number;
  
  cost: {
    input: number;      // $ for input
    output: number;     // $ for output
    cacheRead: number;  // $ for cache reads
    cacheWrite: number; // $ for cache writes
    total: number;      // Total $
  };
}
```

---

### Tool Definition (Tool Interface)

```typescript
interface Tool<TParameters extends TSchema = TSchema> {
  name: string;              // Tool name (unique)
  description: string;       // What the tool does
  parameters: TParameters;   // TypeBox schema for args
}
```

**Example**:
```typescript
import { Type } from "@mariozechner/pi-ai";

const weatherTool: Tool = {
  name: "get_weather",
  description: "Get weather for a location",
  parameters: Type.Object({
    location: Type.String({ description: "City name" }),
    unit: Type.Enum({ celsius: "C", fahrenheit: "F" })
  })
};
```

---

### Context (What Gets Sent to LLM)

```typescript
interface Context {
  systemPrompt?: string;   // System instructions
  messages: Message[];     // Conversation history
  tools?: Tool[];          // Available tools
}
```

**Example**:
```typescript
const context: Context = {
  systemPrompt: "You are a helpful coding assistant",
  messages: [
    { role: "user", content: "Write a function", timestamp: ... },
    { role: "assistant", content: [...], ... },
    { role: "toolResult", toolCallId: "123", ... }
  ],
  tools: [readFileTool, writeFileTool]
};
```

---

## Main Functions

### stream() - Real-Time Streaming

**Purpose**: Get response as a stream of events (for real-time UI updates).

```typescript
function stream<TApi extends Api>(
  model: Model<TApi>,
  context: Context,
  options?: ProviderStreamOptions
): AssistantMessageEventStream;
```

**Returns**: `AssistantMessageEventStream` - Async iterable + `.result()` promise

**Events Emitted**:
- `start` - Response begins
- `text_start` / `text_delta` / `text_end` - Text chunks
- `thinking_start` / `thinking_delta` / `thinking_end` - Reasoning blocks
- `toolcall_start` / `toolcall_delta` / `toolcall_end` - Tool calls
- `done` - Success completion
- `error` - Failure with error message

**Usage Pattern**:
```typescript
const eventStream = stream(model, context);

// Option 1: Iterate events
for await (const event of eventStream) {
  if (event.type === "text_delta") {
    console.log(event.delta); // Print text chunk
  }
}

// Option 2: Get final message
const message = await eventStream.result();
console.log(message.content); // Full AssistantMessage
```

---

### complete() - Full Response

**Purpose**: Wait for complete response (simpler API, no streaming).

```typescript
async function complete<TApi extends Api>(
  model: Model<TApi>,
  context: Context,
  options?: ProviderStreamOptions
): Promise<AssistantMessage>;
```

**Returns**: Full `AssistantMessage` (blocks until complete)

**Usage**:
```typescript
const response = await complete(model, context);
console.log(response.content);
console.log(response.usage.cost.total); // $ spent
```

---

### streamSimple() - Stream with Reasoning

**Purpose**: Stream with extended thinking/reasoning support.

```typescript
function streamSimple<TApi extends Api>(
  model: Model<TApi>,
  context: Context,
  options?: SimpleStreamOptions
): AssistantMessageEventStream;
```

**New Options**:
```typescript
interface SimpleStreamOptions extends StreamOptions {
  reasoning?: ThinkingLevel;  // "minimal" | "low" | "medium" | "high" | "xhigh"
  thinkingBudgets?: {         // Custom token budgets per level
    minimal?: number;
    low?: number;
    medium?: number;
    high?: number;
  };
}
```

**Example**:
```typescript
// Request deep thinking
const stream = streamSimple(model, context, { reasoning: "high" });

// Iterate and handle thinking blocks
for await (const event of stream) {
  if (event.type === "thinking_delta") {
    console.log("🧠 Model thinking:", event.delta);
  }
  if (event.type === "text_delta") {
    console.log("💬 Model says:", event.delta);
  }
}
```

---

### completeSimple() - Complete with Reasoning

**Purpose**: Wait for full response with reasoning.

```typescript
async function completeSimple<TApi extends Api>(
  model: Model<TApi>,
  context: Context,
  options?: SimpleStreamOptions
): Promise<AssistantMessage>;
```

---

### getModel() - Get Model by Name

**Purpose**: Find a model configuration by provider and name.

```typescript
function getModel(provider: Provider, model: string): Model<Api>;
```

**Usage**:
```typescript
const model = getModel("anthropic", "claude-sonnet-4-20250514");
const response = await complete(model, context);

// Get OpenAI
const gpt = getModel("openai", "gpt-4-turbo");
```

---

### getModels() - List Available Models

**Purpose**: Discover all available models.

```typescript
function getModels(filters?: {
  provider?: Provider;
  reasoning?: boolean;
  input?: ("text" | "image")[];
}): Model[];
```

**Usage**:
```typescript
// All models
const all = getModels();

// Models with reasoning
const reasoning = getModels({ reasoning: true });

// Anthropic models that support images
const anthropicWithImages = getModels({
  provider: "anthropic",
  input: ["text", "image"]
});
```

---

### getProviders() - List Available Providers

**Purpose**: Discover available providers and their models.

```typescript
function getProviders(): ProviderInfo[];

interface ProviderInfo {
  name: Provider;
  models: Model[];
}
```

---

## Stream Protocol

### How It Works

The `AssistantMessageEventStream` is an **async iterable + result promise**:

```typescript
class AssistantMessageEventStream {
  // Iterate through events
  async *[Symbol.asyncIterator](): AsyncIterator<AssistantMessageEvent>;
  
  // Get final result when done
  result(): Promise<AssistantMessage>;
}
```

### Event Types

```typescript
type AssistantMessageEvent =
  // Lifecycle
  | { type: "start"; partial: AssistantMessage }
  
  // Text streaming
  | { type: "text_start"; contentIndex: number; partial: AssistantMessage }
  | { type: "text_delta"; contentIndex: number; delta: string; partial: AssistantMessage }
  | { type: "text_end"; contentIndex: number; content: string; partial: AssistantMessage }
  
  // Thinking/reasoning
  | { type: "thinking_start"; contentIndex: number; partial: AssistantMessage }
  | { type: "thinking_delta"; contentIndex: number; delta: string; partial: AssistantMessage }
  | { type: "thinking_end"; contentIndex: number; content: string; partial: AssistantMessage }
  
  // Tool calls
  | { type: "toolcall_start"; contentIndex: number; partial: AssistantMessage }
  | { type: "toolcall_delta"; contentIndex: number; delta: string; partial: AssistantMessage }
  | { type: "toolcall_end"; contentIndex: number; toolCall: ToolCall; partial: AssistantMessage }
  
  // Completion
  | { type: "done"; reason: "stop" | "length" | "toolUse"; message: AssistantMessage }
  | { type: "error"; reason: "error" | "aborted"; error: AssistantMessage };
```

**Key Point**: Every event includes `partial: AssistantMessage` - the message **so far** (accumulated state).

---

## Stream Options

### Common Options (StreamOptions)

```typescript
interface StreamOptions {
  temperature?: number;           // 0.0 - 2.0 (creativity)
  maxTokens?: number;             // Max output tokens
  signal?: AbortSignal;           // Cancel the request
  apiKey?: string;                // Override API key
  
  transport?: "sse" | "websocket" | "auto"; // Transport method
  cacheRetention?: "none" | "short" | "long"; // Prompt cache strategy
  sessionId?: string;             // Session-aware caching
  
  onPayload?: (payload, model) => unknown; // Inspect/modify request
  headers?: Record<string, string>; // Custom HTTP headers
  metadata?: Record<string, unknown>; // Provider metadata
  
  maxRetryDelayMs?: number;       // Max retry wait time
}
```

---

## Model Type

```typescript
interface Model<TApi extends Api> {
  id: string;                 // Unique model ID
  name: string;               // Display name
  api: TApi;                  // API type
  provider: Provider;         // Provider name
  baseUrl: string;            // API endpoint
  
  reasoning: boolean;         // Supports extended thinking?
  input: ("text" | "image")[]; // Input types
  
  cost: {
    input: number;      // $/million tokens
    output: number;
    cacheRead: number;
    cacheWrite: number;
  };
  
  contextWindow: number;      // Total context size
  maxTokens: number;          // Max output tokens
  headers?: Record<string, string>; // Custom headers
  compat?: OpenAICompletionsCompat; // Compatibility overrides
}
```

---

## Common Patterns

### 1. Simple Chat

```typescript
const model = getModel("openai", "gpt-4");
const response = await complete(model, {
  messages: [
    { role: "user", content: "Hello!", timestamp: Date.now() }
  ]
});
console.log(response.content[0].text);
```

---

### 2. Streaming with Real-Time Updates

```typescript
const stream = stream(model, context);

for await (const event of stream) {
  if (event.type === "text_delta") {
    process.stdout.write(event.delta); // Print as it comes
  }
}

const finalMsg = await stream.result();
console.log(`\nTotal cost: $${finalMsg.usage.cost.total}`);
```

---

### 3. Tool Calling Loop

```typescript
let context: Context = {
  systemPrompt: "You are a helpful assistant",
  messages: [
    { role: "user", content: "What's the weather?", timestamp: Date.now() }
  ],
  tools: [weatherTool]
};

while (true) {
  const msg = await complete(getModel("openai", "gpt-4"), context);
  context.messages.push(msg);
  
  // If LLM requested tools
  const toolCalls = msg.content.filter(c => c.type === "toolCall");
  if (toolCalls.length === 0) break; // Done
  
  // Execute tools
  for (const call of toolCalls) {
    const result = await executeTool(call.name, call.arguments);
    context.messages.push({
      role: "toolResult",
      toolCallId: call.id,
      toolName: call.name,
      content: [{ type: "text", text: result }],
      isError: false,
      timestamp: Date.now()
    });
  }
}

// Final response
console.log(context.messages.at(-1)!.content[0].text);
```

---

### 4. Error Handling

```typescript
const stream = stream(model, context);

for await (const event of stream) {
  if (event.type === "error") {
    console.error("LLM error:", event.error.errorMessage);
    console.error("Stop reason:", event.reason); // "error" or "aborted"
  }
}
```

---

### 5. Cost Tracking

```typescript
const response = await complete(model, context);
const { cost } = response.usage;

console.log(`Input: $${cost.input.toFixed(4)}`);
console.log(`Output: $${cost.output.toFixed(4)}`);
console.log(`Total: $${cost.total.toFixed(4)}`);
```

---

## Provider Ecosystem

Pi-ai supports **20+ providers**:

| Provider | Models | Notes |
|----------|--------|-------|
| **OpenAI** | GPT-4, GPT-4o, o1 | Full featured |
| **Anthropic** | Claude 3.5, Opus, Sonnet | Excellent reasoning |
| **Google** | Gemini 2.0, Ultra, Pro | Multimodal |
| **Mistral** | Mistral Large, 7B | European GDPR-friendly |
| **Bedrock** | Claude, Llama, Titan | AWS integration |
| **Groq** | Mixtral, LLaMA | Speed-focused |
| **XAI** | Grok | Alternative |
| **OpenRouter** | 100+ models | Aggregator |

---

## Related Topics

- [[Streaming & Event System]]
- [[Tool Calling & Providers]]
- [[Pi-AI Coding Patterns]]
- [[Pi-AI Study Summary]]

---

**Next**: Learn how [[Streaming & Event System]] works under the hood.
