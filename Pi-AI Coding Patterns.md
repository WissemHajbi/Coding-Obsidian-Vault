# 💻 Pi-AI Coding Patterns

> Common patterns and best practices for using pi-ai in your applications.

**Prerequisites**: [[Pi-AI Package]], [[Streaming & Event System]], [[Tool Calling & Providers]]

---

## Pattern 1: Simple Chat

**Use Case**: Basic user question/answer, no tools needed.

```typescript
import { getModel, complete } from "@mariozechner/pi-ai";

async function simpleChat(userMessage: string): Promise<string> {
  const model = getModel("openai", "gpt-4");
  
  const response = await complete(model, {
    messages: [
      { role: "user", content: userMessage, timestamp: Date.now() }
    ]
  });
  
  // Extract text from response
  const text = response.content
    .filter(c => c.type === "text")
    .map(c => c.text)
    .join("");
  
  return text;
}

// Usage
const answer = await simpleChat("What is 2+2?");
console.log(answer); // "4"
```

---

## Pattern 2: Multi-Turn Conversation

**Use Case**: Chat history, maintaining context across messages.

```typescript
import { Message, Context } from "@mariozechner/pi-ai";

class ChatSession {
  private messages: Message[] = [];
  
  constructor(private systemPrompt: string = "You are a helpful assistant") {}
  
  async send(userMessage: string): Promise<string> {
    const model = getModel("anthropic", "claude-sonnet-4-20250514");
    
    // Add user message
    this.messages.push({
      role: "user",
      content: userMessage,
      timestamp: Date.now()
    });
    
    // Get response
    const response = await complete(model, {
      systemPrompt: this.systemPrompt,
      messages: this.messages
    });
    
    // Add to history
    this.messages.push(response);
    
    // Extract text
    return response.content
      .filter(c => c.type === "text")
      .map(c => c.text)
      .join("");
  }
  
  clear(): void {
    this.messages = [];
  }
}

// Usage
const session = new ChatSession("You are a coding expert");
console.log(await session.send("How do I read a file in Node.js?"));
console.log(await session.send("Can you show me an example?")); // Has context from first message
```

---

## Pattern 3: Streaming to UI

**Use Case**: Real-time chat UI, showing text as it arrives.

```typescript
import { stream } from "@mariozechner/pi-ai";

async function streamToUI(userMessage: string, onChunk: (text: string) => void) {
  const model = getModel("openai", "gpt-4");
  
  const eventStream = stream(model, {
    messages: [
      { role: "user", content: userMessage, timestamp: Date.now() }
    ]
  });
  
  let totalTokens = 0;
  
  for await (const event of eventStream) {
    switch (event.type) {
      case "text_delta":
        onChunk(event.delta);
        break;
      
      case "done":
        totalTokens = event.message.usage.totalTokens;
        console.log(`Complete. Tokens: ${totalTokens}`);
        break;
      
      case "error":
        console.error(`Error: ${event.error.errorMessage}`);
        break;
    }
  }
  
  return totalTokens;
}

// Usage (in a React component)
let displayText = "";
streamToUI("Write a poem", (chunk) => {
  displayText += chunk;
  console.log(displayText); // Update UI
});
```

---

## Pattern 4: Streaming with Thinking

**Use Case**: Deep reasoning models (Claude 3.5, OpenAI o1) that show thinking.

```typescript
import { streamSimple } from "@mariozechner/pi-ai";

async function streamWithThinking(
  userMessage: string,
  onThinking?: (text: string) => void,
  onAnswer?: (text: string) => void
) {
  const model = getModel("anthropic", "claude-3-5-sonnet-20241022");
  
  const eventStream = streamSimple(model, {
    messages: [
      { role: "user", content: userMessage, timestamp: Date.now() }
    ]
  }, {
    reasoning: "high" // Deep thinking
  });
  
  for await (const event of eventStream) {
    if (event.type === "thinking_delta") {
      onThinking?.(event.delta);
    }
    
    if (event.type === "text_delta") {
      onAnswer?.(event.delta);
    }
  }
}

// Usage
await streamWithThinking(
  "Solve this complex math problem: ...",
  (thinking) => console.log("🧠 Thinking:", thinking),
  (answer) => console.log("💬 Answer:", answer)
);
```

---

## Pattern 5: Tool Calling Agent Loop

**Use Case**: Agent that needs to call tools until question is answered.

```typescript
import { getModel, complete, ToolCall, Tool, Type } from "@mariozechner/pi-ai";

class ToolAgent {
  private tools: Tool[];
  
  constructor(tools: Tool[]) {
    this.tools = tools;
  }
  
  async run(userMessage: string, executor: ToolExecutor): Promise<string> {
    const model = getModel("openai", "gpt-4");
    
    const context = {
      systemPrompt: "You are a helpful assistant with access to tools.",
      messages: [
        { role: "user" as const, content: userMessage, timestamp: Date.now() }
      ],
      tools: this.tools
    };
    
    let iterations = 0;
    const maxIterations = 10;
    
    while (iterations < maxIterations) {
      const response = await complete(model, context);
      context.messages.push(response);
      
      // Get tool calls
      const toolCalls = response.content.filter(
        (c): c is ToolCall => c.type === "toolCall"
      );
      
      // No tools called - we're done
      if (toolCalls.length === 0) break;
      
      // Execute tools and collect results
      for (const call of toolCalls) {
        const result = await executor.execute(call);
        context.messages.push({
          role: "toolResult" as const,
          toolCallId: call.id,
          toolName: call.name,
          content: [{ type: "text" as const, text: result }],
          isError: false,
          timestamp: Date.now()
        });
      }
      
      iterations++;
    }
    
    // Extract final answer
    const lastMsg = context.messages[context.messages.length - 1];
    if (lastMsg.role === "assistant") {
      return lastMsg.content
        .filter(c => c.type === "text")
        .map(c => c.text)
        .join("");
    }
    
    return "No response";
  }
}

interface ToolExecutor {
  execute(call: ToolCall): Promise<string>;
}

// Usage
const tools: Tool[] = [
  {
    name: "calculator",
    description: "Perform math calculations",
    parameters: Type.Object({
      expression: Type.String()
    })
  }
];

const executor: ToolExecutor = {
  async execute(call) {
    if (call.name === "calculator") {
      // Safely evaluate math expression
      return String(eval(call.arguments.expression));
    }
    return "Unknown tool";
  }
};

const agent = new ToolAgent(tools);
const result = await agent.run("What is 15 * 23?", executor);
console.log(result);
```

---

## Pattern 6: Cost Tracking

**Use Case**: Monitor and log API costs.

```typescript
import { complete, AssistantMessage } from "@mariozechner/pi-ai";

class CostTracker {
  private totalCost = 0;
  private callCount = 0;
  
  async track(userMessage: string): Promise<string> {
    const model = getModel("openai", "gpt-4");
    
    const response = await complete(model, {
      messages: [
        { role: "user", content: userMessage, timestamp: Date.now() }
      ]
    });
    
    // Track costs
    this.callCount++;
    this.totalCost += response.usage.cost.total;
    
    console.log(`Call #${this.callCount}`);
    console.log(`  Input cost: $${response.usage.cost.input.toFixed(6)}`);
    console.log(`  Output cost: $${response.usage.cost.output.toFixed(6)}`);
    console.log(`  Total cost: $${response.usage.cost.total.toFixed(6)}`);
    console.log(`  Running total: $${this.totalCost.toFixed(4)}`);
    
    return response.content
      .filter(c => c.type === "text")
      .map(c => c.text)
      .join("");
  }
  
  getStats() {
    return {
      calls: this.callCount,
      totalCost: this.totalCost,
      averageCost: this.callCount > 0 ? this.totalCost / this.callCount : 0
    };
  }
}

// Usage
const tracker = new CostTracker();
await tracker.track("What is the capital of France?");
await tracker.track("Tell me about it");
console.log(tracker.getStats());
// { calls: 2, totalCost: 0.0042, averageCost: 0.0021 }
```

---

## Pattern 7: Provider Fallback

**Use Case**: Try one provider, fallback to another if it fails.

```typescript
import { complete, getModel } from "@mariozechner/pi-ai";

async function requestWithFallback(userMessage: string) {
  const providers = [
    { provider: "openai", model: "gpt-4" },
    { provider: "anthropic", model: "claude-sonnet-4-20250514" },
    { provider: "google", model: "gemini-2.0-flash" }
  ];
  
  for (const { provider, model: modelId } of providers) {
    try {
      console.log(`Trying ${provider}...`);
      
      const model = getModel(provider as any, modelId);
      const response = await complete(model, {
        messages: [
          { role: "user", content: userMessage, timestamp: Date.now() }
        ]
      });
      
      console.log(`✓ Success with ${provider}`);
      return response;
      
    } catch (error) {
      console.log(`✗ ${provider} failed: ${error.message}`);
      // Try next provider
    }
  }
  
  throw new Error("All providers failed");
}

// Usage
const response = await requestWithFallback("Hello!");
```

---

## Pattern 8: Request Timeout & Cancellation

**Use Case**: Cancel long-running requests.

```typescript
import { stream } from "@mariozechner/pi-ai";

async function streamWithTimeout(userMessage: string, timeoutMs = 30000) {
  const controller = new AbortController();
  
  // Auto-cancel after timeout
  const timeout = setTimeout(() => controller.abort(), timeoutMs);
  
  try {
    const eventStream = stream(getModel("openai", "gpt-4"), {
      messages: [
        { role: "user", content: userMessage, timestamp: Date.now() }
      ]
    }, {
      signal: controller.signal
    });
    
    for await (const event of eventStream) {
      if (event.type === "text_delta") {
        process.stdout.write(event.delta);
      }
      if (event.type === "error" && event.reason === "aborted") {
        console.log("\n⏱️ Request cancelled due to timeout");
        return;
      }
    }
    
    console.log("\n✓ Completed in time");
    
  } finally {
    clearTimeout(timeout);
  }
}

// Usage
await streamWithTimeout("Write a novel", 5000); // 5 second limit
```

---

## Pattern 9: Batch Processing

**Use Case**: Process multiple requests efficiently.

```typescript
async function processBatch(messages: string[]): Promise<string[]> {
  const model = getModel("openai", "gpt-4");
  const results: string[] = [];
  
  // Process sequentially to avoid rate limits
  for (const msg of messages) {
    const response = await complete(model, {
      messages: [
        { role: "user", content: msg, timestamp: Date.now() }
      ]
    });
    
    results.push(
      response.content
        .filter(c => c.type === "text")
        .map(c => c.text)
        .join("")
    );
    
    // Rate limit: 100ms between requests
    await new Promise(r => setTimeout(r, 100));
  }
  
  return results;
}

// Usage
const results = await processBatch([
  "What is 2+2?",
  "What is 3+3?",
  "What is 4+4?"
]);

console.log(results);
// ["4", "6", "8"]
```

---

## Pattern 10: Streaming Multiple Requests in Parallel

**Use Case**: Multiple concurrent requests with streaming.

```typescript
async function streamMultiple(messages: string[]) {
  const model = getModel("openai", "gpt-4");
  
  const streams = messages.map(msg =>
    stream(model, {
      messages: [
        { role: "user", content: msg, timestamp: Date.now() }
      ]
    })
  );
  
  // Stream all in parallel
  const promises = streams.map(async (eventStream, index) => {
    let fullText = "";
    
    for await (const event of eventStream) {
      if (event.type === "text_delta") {
        fullText += event.delta;
        // Real-time update for message index
        console.log(`[${index}] ${fullText}`);
      }
    }
    
    return fullText;
  });
  
  return Promise.all(promises);
}

// Usage
const results = await streamMultiple([
  "Tell me a joke",
  "Tell me a riddle",
  "Tell me a quote"
]);
```

---

## Pattern 11: Image Input

**Use Case**: Process images with vision models.

```typescript
import fs from "fs";

async function analyzeImage(imagePath: string, question: string): Promise<string> {
  // Read image as base64
  const imageData = fs.readFileSync(imagePath, "base64");
  const mimeType = imagePath.endsWith(".png") ? "image/png" : "image/jpeg";
  
  const model = getModel("openai", "gpt-4-vision");
  
  const response = await complete(model, {
    messages: [
      {
        role: "user",
        content: [
          { type: "image", data: imageData, mimeType },
          { type: "text", text: question }
        ],
        timestamp: Date.now()
      }
    ]
  });
  
  return response.content
    .filter(c => c.type === "text")
    .map(c => c.text)
    .join("");
}

// Usage
const analysis = await analyzeImage("photo.jpg", "What's in this image?");
console.log(analysis);
```

---

## Pattern 12: Custom Provider

**Use Case**: Use a custom/private LLM server.

```typescript
import { registerApiProvider, createAssistantMessageEventStream } from "@mariozechner/pi-ai";

registerApiProvider("my-server", {
  stream(model, context, options) {
    const eventStream = createAssistantMessageEventStream();
    
    // Fetch from your server
    fetch(`${model.baseUrl}/stream`, {
      method: "POST",
      body: JSON.stringify({
        messages: context.messages,
        tools: context.tools,
        model: model.id
      }),
      headers: { "Authorization": `Bearer ${options?.apiKey}` }
    }).then(async (response) => {
      const reader = response.body!.getReader();
      
      while (true) {
        const { done, value } = await reader.read();
        if (done) break;
        
        // Parse your server's events and emit them
        const text = new TextDecoder().decode(value);
        eventStream.push({
          type: "text_delta",
          contentIndex: 0,
          delta: text,
          partial: { /* your partial message */ } as any
        });
      }
      
      eventStream.push({
        type: "done",
        reason: "stop",
        message: { /* final message */ } as any
      });
    });
    
    return eventStream;
  },
  
  streamSimple(model, context, options) {
    return this.stream(model, context, options);
  }
});

// Now use it
const model = getModel("my-server", "my-model");
const response = await complete(model, context);
```

---

## Key Takeaways

1. **Always check stop reasons** - Different completion types (stop, length, error, toolUse)
2. **Handle streaming** - Events let you build UIs that feel responsive
3. **Track costs** - Know what you're spending on LLM calls
4. **Batch carefully** - Respect rate limits
5. **Use TypeBox for schemas** - Type safety + JSON Schema compatibility
6. **Error handling** - Fallbacks, timeouts, retries
7. **Tool validation** - LLM arguments are validated automatically

---

## Related

- [[Pi-AI Package]]
- [[Tool Calling & Providers]]
- [[Streaming & Event System]]
