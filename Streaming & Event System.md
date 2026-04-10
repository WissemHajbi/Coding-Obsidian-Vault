# 🌊 Streaming & Event System

> How pi-ai delivers real-time LLM responses through an async iterable event stream.

**See**: [[Pi-AI Package]]

---

## The Problem

LLMs take time to generate responses. You have two choices:

1. **Wait** until the full response is done
2. **Stream** partial results in real-time

Streaming is better for UX (show text as it arrives), but the API is more complex.

---

## The Solution: AssistantMessageEventStream

Pi-ai provides a single stream interface that:
- Works with **any provider** (OpenAI, Anthropic, Google, etc.)
- Emits **granular events** (text chunks, thinking blocks, tool calls)
- Provides **partial state** at each event
- Can be **iterated** or **awaited**

---

## Core Mechanics

### 1. The EventStream Class

```typescript
class EventStream<T, R = T> implements AsyncIterable<T> {
  // Iterate through events
  async *[Symbol.asyncIterator](): AsyncIterator<T>;
  
  // Get final result when done
  result(): Promise<R>;
  
  // Internal: emit event
  push(event: T): void;
  
  // Internal: finalize stream
  end(result?: R): void;
}
```

**Why this design?**
- `[Symbol.asyncIterator]` allows `for await...of` loops
- `result()` promise lets you wait for completion
- Both patterns work simultaneously

---

### 2. AssistantMessageEventStream Specialization

```typescript
class AssistantMessageEventStream 
  extends EventStream<AssistantMessageEvent, AssistantMessage> {
  constructor() {
    super(
      // Event completion check: stream ends on "done" or "error"
      (event) => event.type === "done" || event.type === "error",
      
      // Result extractor: final message from the completion event
      (event) => {
        if (event.type === "done") return event.message;
        if (event.type === "error") return event.error;
        throw new Error("Unexpected event type");
      }
    );
  }
}
```

**Translation**:
- Events are: `AssistantMessageEvent` (many types)
- Final result is: `AssistantMessage` (single unified message)
- Stream ends when event is "done" or "error"

---

## Event Types & Protocol

### Lifecycle

```typescript
// Stream starts - first event always
{ type: "start"; partial: AssistantMessage }
```

The `partial` field contains the message state so far (empty initially).

---

### Text Streaming

```typescript
{ type: "text_start"; contentIndex: number; partial: AssistantMessage }
{ type: "text_delta"; contentIndex: number; delta: string; partial: AssistantMessage }
{ type: "text_end"; contentIndex: number; content: string; partial: AssistantMessage }
```

**Flow**:
1. `text_start` → New text content block starting at index 0
2. `text_delta` → Chunk "The " arrives
3. `text_delta` → Chunk "weather " arrives
4. `text_delta` → Chunk "is..." arrives
5. `text_end` → Block complete, full text: "The weather is..."

**Why contentIndex?**
- AssistantMessage can have multiple content blocks (text + thinking + toolcalls)
- Each block streams independently with its index

---

### Thinking/Reasoning Streaming

```typescript
{ type: "thinking_start"; contentIndex: number; partial: AssistantMessage }
{ type: "thinking_delta"; contentIndex: number; delta: string; partial: AssistantMessage }
{ type: "thinking_end"; contentIndex: number; content: string; partial: AssistantMessage }
```

Same pattern as text, but for thinking blocks (Claude 3.5, OpenAI o1).

---

### Tool Call Streaming

```typescript
{ type: "toolcall_start"; contentIndex: number; partial: AssistantMessage }
{ type: "toolcall_delta"; contentIndex: number; delta: string; partial: AssistantMessage }
{ type: "toolcall_end"; contentIndex: number; toolCall: ToolCall; partial: AssistantMessage }
```

**Why streaming tool calls?**
- Tool call JSON is built incrementally
- Some providers stream the `arguments` object character-by-character
- Need to track partial JSON as it arrives

---

### Completion Events

```typescript
// Success
{ type: "done"; reason: "stop" | "length" | "toolUse"; message: AssistantMessage }

// Failure
{ type: "error"; reason: "error" | "aborted"; error: AssistantMessage }
```

**Stop Reasons**:
- `"stop"` - LLM naturally ended (no special reason)
- `"length"` - Ran out of tokens (max_tokens hit)
- `"toolUse"` - LLM is requesting tool calls (not an error)
- `"error"` - Request failed (API error, timeout, etc.)
- `"aborted"` - User cancelled via AbortSignal

---

## Usage Patterns

### Pattern 1: Iterate Events

```typescript
const stream = stream(model, context);

for await (const event of stream) {
  switch (event.type) {
    case "text_delta":
      process.stdout.write(event.delta); // Print as you go
      break;
    case "thinking_delta":
      console.log("🧠", event.delta);
      break;
    case "done":
      console.log("\n✓ Complete");
      console.log("Tokens:", event.message.usage.totalTokens);
      break;
    case "error":
      console.error("❌", event.error.errorMessage);
      break;
  }
}
```

---

### Pattern 2: Wait for Result

```typescript
const stream = stream(model, context);

// Ignore events, just wait for final message
const message = await stream.result();

console.log(message.content);
console.log(message.usage.cost.total);
```

---

### Pattern 3: Dual Pattern (Stream + Wait)

```typescript
const stream = stream(model, context);

// Start async iteration (doesn't block)
(async () => {
  for await (const event of stream) {
    if (event.type === "text_delta") {
      // Update UI in real-time
      updateUI(event.delta);
    }
  }
})();

// Meanwhile, wait for completion elsewhere
const message = await stream.result();
saveToDatabase(message); // Happens when stream is done
```

Both patterns run concurrently. This is the power of the design!

---

### Pattern 4: Building Text from Deltas

```typescript
const stream = stream(model, context);
let fullText = "";

for await (const event of stream) {
  if (event.type === "text_delta") {
    fullText += event.delta;
  }
}

// fullText is now complete
console.log(fullText);
```

**Better**: Just use `stream.result()` - it accumulates for you!

---

### Pattern 5: Tool Call Processing

```typescript
const stream = stream(model, context);
let toolCalls: ToolCall[] = [];

for await (const event of stream) {
  if (event.type === "toolcall_end") {
    toolCalls.push(event.toolCall);
    console.log(`Tool call: ${event.toolCall.name}(${JSON.stringify(event.toolCall.arguments)})`);
  }
}

// Execute all tool calls
for (const call of toolCalls) {
  await executeTool(call);
}
```

---

### Pattern 6: Partial State Tracking

Every event includes `partial: AssistantMessage` - the message as it exists **so far**.

```typescript
const stream = stream(model, context);

for await (const event of stream) {
  // partial is the message accumulated up to this event
  console.log("Content blocks so far:", event.partial.content.length);
  console.log("Current text:", event.partial.content
    .filter(c => c.type === "text")
    .map(c => c.text)
    .join("")
  );
}
```

---

## Content Index & Multiple Blocks

Why does each event include a `contentIndex`?

**Scenario**: Model generates thinking, then text:

```
START
  ↓
THINKING_START (contentIndex: 0)
THINKING_DELTA (contentIndex: 0, "Let me think...")
THINKING_END (contentIndex: 0)
  ↓
TEXT_START (contentIndex: 1)     ← New block
TEXT_DELTA (contentIndex: 1, "The answer...")
TEXT_END (contentIndex: 1)
  ↓
DONE
```

**Result**:
```typescript
content: [
  { type: "thinking", thinking: "Let me think..." },
  { type: "text", text: "The answer..." }
]
```

The index tells you which block in the array is being updated.

---

## Practical Example: Building a Chat UI

```typescript
async function chatStream(userMessage: string) {
  const model = getModel("openai", "gpt-4");
  const context: Context = {
    messages: [
      { role: "user", content: userMessage, timestamp: Date.now() }
    ]
  };
  
  const eventStream = stream(model, context);
  
  // Stream events to UI
  (async () => {
    for await (const event of eventStream) {
      if (event.type === "start") {
        ui.showLoading();
      }
      
      if (event.type === "text_delta") {
        ui.appendToMessage(event.delta);
      }
      
      if (event.type === "thinking_delta") {
        ui.appendToThinkingPanel(event.delta);
      }
      
      if (event.type === "done") {
        ui.hideLoading();
        console.log(`Spent: $${event.message.usage.cost.total}`);
      }
      
      if (event.type === "error") {
        ui.showError(event.error.errorMessage);
      }
    }
  })();
  
  // Wait and return final message (optional)
  return await eventStream.result();
}
```

---

## Error Handling in Streams

```typescript
const stream = stream(model, context);

for await (const event of stream) {
  if (event.type === "error") {
    // event.reason: "error" or "aborted"
    // event.error: AssistantMessage with errorMessage
    
    console.error(`Failed (${event.reason}):`, event.error.errorMessage);
    
    // Stop iterating
    break;
  }
}
```

---

## Cancellation with AbortSignal

```typescript
const controller = new AbortController();

const stream = stream(model, context, {
  signal: controller.signal
});

// Cancel after 5 seconds
setTimeout(() => controller.abort(), 5000);

for await (const event of stream) {
  if (event.type === "error" && event.reason === "aborted") {
    console.log("User cancelled");
  }
}
```

---

## Performance Considerations

### When to use streaming:
- ✅ Interactive UIs (show text as it arrives)
- ✅ Long responses (show progress)
- ✅ Real-time interactions

### When to use complete():
- ✅ Batch processing (don't need intermediate updates)
- ✅ Simpler code (less event handling)
- ✅ When you need final state (complete() is just a wrapper)

---

## Under the Hood: Provider Implementation

Each provider implements the `StreamFunction`:

```typescript
type StreamFunction<TApi extends Api> = (
  model: Model<TApi>,
  context: Context,
  options?: StreamOptions
) => AssistantMessageEventStream;
```

**For example, OpenAI provider**:
1. Calls OpenAI API with streaming enabled
2. For each chunk from OpenAI:
   - Parse the delta
   - Emit appropriate event (`text_delta`, `toolcall_delta`, etc.)
   - Update `partial` AssistantMessage
3. On stream end:
   - Emit `done` with complete message
   - Stream ends automatically

---

## Key Takeaways

1. **Unified across providers** - Same event types whether you use OpenAI, Anthropic, or Google
2. **Flexible consumption** - Iterate events OR wait for result OR do both
3. **Partial state** - Every event includes message-so-far
4. **Granular updates** - Track text, thinking, and tool calls separately
5. **Cancellation support** - Use AbortSignal to stop streams

---

## Related

- [[Pi-AI Package]]
- [[Tool Calling & Providers]]
- [[Pi-AI Coding Patterns]]
