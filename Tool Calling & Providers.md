# 🔧 Tool Calling & Provider System

> How pi-ai abstracts tool calling across 20+ providers and enables agent tool execution.

---

## Problem: Provider Differences

Each LLM provider formats tools differently:

| Provider | Format | Tool Call Response |
|----------|--------|-------------------|
| **OpenAI** | `tools: [{type: "function", function: {...}}]` | `tool_calls: [{id, type, function: {name, arguments: JSON string}}]` |
| **Anthropic** | `tools: [{name, description, input_schema}]` | `tool_use: [{id, name, input: object}]` |
| **Google** | `tools: [{functionDeclarations}]` | `toolCalls: [{name, args}]` |
| **Mistral** | OpenAI-like | OpenAI-like |
| **Bedrock** | Claude-like | Claude-like |

**The Solution**: Pi-ai normalizes all to a unified format.

---

## The Tool Interface

```typescript
interface Tool<TParameters extends TSchema = TSchema> {
  name: string;              // Unique identifier
  description: string;       // What the tool does (used in LLM context)
  parameters: TParameters;   // TypeBox schema for validation
}
```

**Key Point**: Use TypeBox (`Type`) for schemas. It's:
- Type-safe (TypeScript integration)
- JSON Schema compatible (works with all providers)
- Validated by AJV automatically

---

## Defining Tools

### Basic Tool

```typescript
import { Type } from "@mariozechner/pi-ai";

const readFileTool: Tool = {
  name: "read_file",
  description: "Read contents of a file",
  parameters: Type.Object({
    path: Type.String({
      description: "File path"
    }),
    encoding: Type.Optional(Type.Enum({
      utf8: "utf-8",
      ascii: "ascii"
    }))
  })
};
```

### Complex Tool with Nested Objects

```typescript
const executeCodeTool: Tool = {
  name: "execute_code",
  description: "Execute JavaScript code",
  parameters: Type.Object({
    language: Type.Enum({
      javascript: "javascript",
      python: "python",
      bash: "bash"
    }),
    code: Type.String(),
    timeout: Type.Optional(Type.Number({ minimum: 1, maximum: 30000 })),
    environment: Type.Optional(Type.Object({
      NODE_ENV: Type.Optional(Type.String()),
      DEBUG: Type.Optional(Type.Boolean())
    }))
  })
};
```

### Enum Tool

```typescript
const weatherTool: Tool = {
  name: "get_weather",
  description: "Get weather for a location",
  parameters: Type.Object({
    location: Type.String(),
    unit: Type.Enum({
      celsius: "C",
      fahrenheit: "F",
      kelvin: "K"
    }, { description: "Temperature unit" })
  })
};
```

### Array Tool

```typescript
const searchTool: Tool = {
  name: "search_documents",
  description: "Search through documents",
  parameters: Type.Object({
    query: Type.String(),
    tags: Type.Optional(Type.Array(Type.String())),
    limit: Type.Optional(Type.Number({ minimum: 1, maximum: 100 }))
  })
};
```

---

## Tool Calling Flow

### Step 1: Provide Tools to LLM

```typescript
const context: Context = {
  systemPrompt: "You are a coding assistant",
  messages: [
    { role: "user", content: "Read config.json", timestamp: Date.now() }
  ],
  tools: [readFileTool, writeFileTool, executeCodeTool]
};

const response = await complete(model, context);
```

### Step 2: LLM Requests Tools

The LLM sees the tools and decides when to use them. Response:

```typescript
// response is AssistantMessage with:
content: [
  {
    type: "toolCall",
    id: "call_abc123",
    name: "read_file",
    arguments: { path: "config.json", encoding: "utf-8" }
  }
]
stopReason: "toolUse" // ← Indicates tools were requested
```

### Step 3: Execute the Tool

```typescript
async function executeTool(call: ToolCall): Promise<string> {
  if (call.name === "read_file") {
    const { path, encoding } = call.arguments;
    return fs.readFileSync(path, encoding || "utf-8");
  }
  if (call.name === "write_file") {
    // ...
  }
  throw new Error(`Unknown tool: ${call.name}`);
}

const toolResult = await executeTool(toolCall);
```

### Step 4: Return Result to LLM

```typescript
context.messages.push({
  role: "toolResult",
  toolCallId: toolCall.id,      // Link to the tool call
  toolName: toolCall.name,
  content: [{ type: "text", text: toolResult }],
  isError: false,               // true if tool execution failed
  timestamp: Date.now()
});
```

### Step 5: Loop Until Done

```typescript
let done = false;

while (!done) {
  const response = await complete(model, context);
  context.messages.push(response);
  
  const toolCalls = response.content.filter(c => c.type === "toolCall");
  
  if (toolCalls.length === 0) {
    done = true; // No more tools needed
  } else {
    // Execute all tool calls and add results
    for (const call of toolCalls) {
      const result = await executeTool(call);
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
}

// Final response
const finalMessage = context.messages[context.messages.length - 1];
console.log(finalMessage.content[0].text); // Final answer
```

---

## Complete Tool Loop Example

```typescript
import { getModel, complete, Type } from "@mariozechner/pi-ai";

const tools = [
  {
    name: "read_file",
    description: "Read a file",
    parameters: Type.Object({
      path: Type.String()
    })
  },
  {
    name: "list_directory",
    description: "List files in a directory",
    parameters: Type.Object({
      path: Type.String()
    })
  }
];

async function toolExecutor(call: ToolCall): Promise<string> {
  if (call.name === "read_file") {
    return fs.readFileSync(call.arguments.path, "utf-8");
  }
  if (call.name === "list_directory") {
    return fs.readdirSync(call.arguments.path).join(", ");
  }
  return "Unknown tool";
}

async function runAgent(userMessage: string) {
  const model = getModel("openai", "gpt-4");
  
  const context: Context = {
    systemPrompt: "You are a file system assistant. Help the user work with files.",
    messages: [
      { role: "user", content: userMessage, timestamp: Date.now() }
    ],
    tools
  };
  
  let iterations = 0;
  const maxIterations = 10;
  
  while (iterations < maxIterations) {
    const response = await complete(model, context);
    context.messages.push(response);
    
    // Check if LLM requested tools
    const toolCalls = response.content.filter(
      (c): c is ToolCall => c.type === "toolCall"
    );
    
    if (toolCalls.length === 0 || response.stopReason !== "toolUse") {
      // Done - no more tool calls
      break;
    }
    
    // Execute tools
    for (const call of toolCalls) {
      const result = await toolExecutor(call);
      context.messages.push({
        role: "toolResult",
        toolCallId: call.id,
        toolName: call.name,
        content: [{ type: "text", text: result }],
        isError: false,
        timestamp: Date.now()
      });
    }
    
    iterations++;
  }
  
  // Extract final response
  const lastMessage = context.messages[context.messages.length - 1];
  if (lastMessage.role === "assistant") {
    return lastMessage.content
      .filter((c): c is TextContent => c.type === "text")
      .map(c => c.text)
      .join("");
  }
  
  return "No response";
}

// Usage
const answer = await runAgent("What files are in my home directory?");
console.log(answer);
```

---

## Streaming Tool Calls

Tool calls can be streamed too (important for large JSON arguments):

```typescript
const stream = stream(model, context);

for await (const event of stream) {
  if (event.type === "toolcall_start") {
    console.log("Tool call starting...");
  }
  
  if (event.type === "toolcall_delta") {
    console.log("Building:", event.delta);
  }
  
  if (event.type === "toolcall_end") {
    // Now we have the complete tool call
    const call = event.toolCall;
    console.log(`Execute: ${call.name}(${JSON.stringify(call.arguments)})`);
  }
}
```

---

## Provider System

Pi-ai supports providers via a registry:

```typescript
type Provider = "openai" | "anthropic" | "google" | "mistral" | ...

interface ApiProvider {
  // Stream implementation
  stream(model: Model, context: Context, options?: StreamOptions): AssistantMessageEventStream;
  
  // Stream with reasoning
  streamSimple(model: Model, context: Context, options?: SimpleStreamOptions): AssistantMessageEventStream;
}
```

### Available Providers

```typescript
// OpenAI ecosystem
"openai"              // OpenAI GPT-4, GPT-4o
"azure-openai"        // Azure OpenAI
"github-copilot"      // GitHub Copilot
"xai"                 // X.ai Grok
"groq"                // Groq (fast inference)

// Anthropic ecosystem
"anthropic"           // Claude models
"amazon-bedrock"      // AWS Bedrock with Claude/others

// Google ecosystem
"google"              // Google Gemini
"google-vertex"       // Google Vertex AI
"google-gemini-cli"   // Local Google Gemini CLI

// Mistral ecosystem
"mistral"             // Mistral AI

// OpenRouter
"openrouter"          // Aggregates 100+ models

// Other
"cerebras"            // Fast inference
"huggingface"         // HF hosted models
"minimax"             // Chinese models
"kimi-coding"         // Coding-specific
```

---

## Getting Models from Providers

### Single Model

```typescript
const model = getModel("anthropic", "claude-sonnet-4-20250514");
```

### List All Models from Provider

```typescript
const providers = getProviders();
const anthropic = providers.find(p => p.name === "anthropic");
console.log(anthropic.models); // All Claude models
```

### Find Models by Criteria

```typescript
// Models with reasoning
const reasoning = getModels({ reasoning: true });

// Models that support images
const multimodal = getModels({ input: ["text", "image"] });

// OpenAI models only
const openai = getModels({ provider: "openai" });
```

---

## Provider Configuration

### API Keys

Each provider needs API key configuration (via environment or options):

```typescript
// Environment variables
process.env.OPENAI_API_KEY = "sk-...";
process.env.ANTHROPIC_API_KEY = "sk-ant-...";

// Or pass in options
const response = await complete(model, context, {
  apiKey: "sk-..."
});
```

### Custom Headers

Some providers support custom headers:

```typescript
await complete(model, context, {
  headers: {
    "User-Agent": "MyApp/1.0",
    "X-Custom-Header": "value"
  }
});
```

### Timeout & Retry

```typescript
await complete(model, context, {
  maxRetryDelayMs: 60000 // 60 second max retry wait
});
```

---

## Tool Validation

Pi-ai automatically validates tool arguments using AJV (JSON Schema):

```typescript
// If LLM provides invalid arguments, validation fails
// Error is returned with stopReason: "error"

const response = await complete(model, context);

if (response.stopReason === "error") {
  console.error("Validation error:", response.errorMessage);
}
```

**Best Practice**: Always check `stopReason` before processing.

---

## Tool Best Practices

### 1. Clear Descriptions

```typescript
// ❌ Bad
{ name: "tool1", description: "Do stuff", parameters: ... }

// ✅ Good
{ 
  name: "read_file_content",
  description: "Read the complete contents of a file. Returns text or binary data.",
  parameters: ...
}
```

### 2. Document Parameters

```typescript
parameters: Type.Object({
  path: Type.String({
    description: "Absolute file path. Example: /home/user/config.json"
  }),
  maxSize: Type.Optional(Type.Number({
    description: "Maximum bytes to read. Default: 1MB",
    minimum: 1,
    maximum: 10485760
  }))
})
```

### 3. Limit Tool Count

```typescript
// ❌ Bad - too many choices
const tools = [100 tools...]; // LLM gets confused

// ✅ Good - focused toolset
const tools = [readFile, writeFile, bash]; // 3-5 well-defined tools
```

### 4. Handle Errors Gracefully

```typescript
const result = await executeTool(call);

if (error) {
  context.messages.push({
    role: "toolResult",
    toolCallId: call.id,
    toolName: call.name,
    content: [{ type: "text", text: `Error: ${error.message}` }],
    isError: true,  // ← Mark as error
    timestamp: Date.now()
  });
}
```

---

## Advanced: Custom Providers

To add a custom provider (e.g., a private LLM server):

```typescript
import { registerApiProvider } from "@mariozechner/pi-ai";

registerApiProvider("my-custom-api", {
  stream(model, context, options) {
    // Your implementation that returns AssistantMessageEventStream
    return customImplementation(model, context, options);
  },
  
  streamSimple(model, context, options) {
    // Your implementation with reasoning support
    return customImplementationSimple(model, context, options);
  }
});

// Now use it
const model = getModel("my-custom-api", "my-model");
```

---

## Related

- [[Pi-AI Package]]
- [[Streaming & Event System]]
- [[Pi-AI Coding Patterns]]
