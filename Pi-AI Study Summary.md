# 📚 Pi-AI Study Summary

> Complete study guide for the pi-ai package based on code analysis and README.

**Completed**: ✅ Comprehensive documentation of pi-ai package created in Obsidian vault

---

## What You Now Have

### 📄 Documents Created

| Document | Focus | Length |
|----------|-------|--------|
| [[Pi-AI Package]] | Core types, functions, concepts | ~18KB |
| [[Streaming & Event System]] | Event stream protocol, patterns | ~11KB |
| [[Tool Calling & Providers]] | Tool definition, execution, providers | ~13KB |
| [[Pi-AI Coding Patterns]] | 12 real-world coding patterns | ~16KB |
| **Total** | **Complete pi-ai reference** | **~58KB** |

---

## Core Concepts Covered

### 1. **Message Types** ✅
- `UserMessage` - User input (text + images)
- `AssistantMessage` - LLM response (text + thinking + tools)
- `ToolResultMessage` - Tool execution results
- Content types: Text, Thinking, Image, ToolCall

### 2. **Main Functions** ✅
- `getModel()` - Get model by provider + name
- `stream()` - Real-time event streaming
- `complete()` - Wait for full response
- `streamSimple()` - Stream with reasoning support
- `completeSimple()` - Complete with reasoning
- `getModels()` - Discover available models
- `getProviders()` - List providers

### 3. **Streaming System** ✅
- `AssistantMessageEventStream` - Async iterable + result promise
- Event types: start, text_delta, thinking_delta, toolcall_delta, done, error
- Partial state tracking at each event
- Content indexing for multiple blocks

### 4. **Tool System** ✅
- Tool definition with TypeBox schemas
- Tool calling loop (request → execute → return → repeat)
- Provider normalization (all to unified format)
- Validation with AJV

### 5. **Providers** ✅
- 20+ providers supported
- Provider selection and fallback patterns
- Custom provider registration
- API key management

### 6. **Options & Configuration** ✅
- StreamOptions: temperature, maxTokens, signal, etc.
- SimpleStreamOptions: reasoning levels and budgets
- Cache retention, session IDs, custom headers

---

## What You Can Now Do

### ✅ Understand Pi-AI

1. **Explain unified interface** - Why one API works with 20+ providers
2. **Trace message flow** - User input → LLM API → Event stream → Response
3. **Design tool systems** - Define tools that work with any LLM
4. **Implement streaming UI** - Build real-time chat interfaces
5. **Handle errors** - Deal with timeouts, cancellations, failures

### ✅ Write Pi-AI Code

1. **Simple chat** - Single request/response
2. **Multi-turn conversations** - Maintain history
3. **Real-time streaming** - Show text as it arrives
4. **Tool calling agents** - LLM requesting tools, executing, returning results
5. **Cost tracking** - Monitor API spending
6. **Provider fallback** - Try multiple providers
7. **Timeout handling** - Cancel long requests
8. **Batch processing** - Multiple requests efficiently
9. **Image analysis** - Vision models with images
10. **Custom providers** - Integrate private LLM servers

### ✅ Optimize & Extend

1. **Provider selection** - Choose best provider for task
2. **Cost optimization** - Select cheaper models
3. **Token management** - Control context window usage
4. **Reasoning budgets** - Control thinking token allocation
5. **Parallel requests** - Stream multiple at once
6. **Error recovery** - Fallbacks and retries

---

## Key Insights

### Architecture Philosophy

Pi-ai's design philosophy:

```
Problem: Different LLMs have different APIs
Solution: Unified interface + provider abstraction
Result: Swap providers with one line change
```

### Event Streaming Design

```
Traditional API:        Pi-ai Streaming:
request                 request
  ↓                       ↓
[wait...]               [start event]
  ↓                       ↓
response                [text_delta events] → Real-time UI updates
                        [done event]
                          ↓
                        response
```

### Tool Calling Pattern

```
1. LLM sees tools in context
2. LLM generates ToolCall in response
3. App executes tool
4. App creates ToolResultMessage
5. App sends back to LLM
6. Loop repeats until no more tool calls
```

---

## Quick Reference

### Get a Model
```typescript
const model = getModel("openai", "gpt-4");
```

### Simple Chat
```typescript
const response = await complete(model, { messages: [...] });
```

### Stream Text
```typescript
for await (const event of stream(model, context)) {
  if (event.type === "text_delta") console.log(event.delta);
}
```

### Define Tool
```typescript
const tool = {
  name: "read_file",
  description: "Read a file",
  parameters: Type.Object({ path: Type.String() })
};
```

### Execute Tools Loop
```typescript
while (true) {
  const response = await complete(model, { messages, tools });
  const calls = response.content.filter(c => c.type === "toolCall");
  if (!calls.length) break;
  // Execute and add ToolResultMessages
}
```

---

## Next Steps

### 📖 Read in This Order

1. **[[Pi-AI Package]]** - Get the overview (30 min)
2. **[[Streaming & Event System]]** - Understand real-time updates (20 min)
3. **[[Tool Calling & Providers]]** - Learn tools & providers (25 min)
4. **[[Pi-AI Coding Patterns]]** - See it in action (30 min)

**Total**: ~2 hours to full understanding

### 💻 Try Code Snippets

1. Start with Pattern 1: Simple Chat
2. Move to Pattern 3: Streaming to UI
3. Try Pattern 5: Tool Calling Agent
4. Explore Pattern 11: Image Input

### 🔍 Dive Into Code

`packages/ai/src/` structure:
```
types.ts                    # All type definitions
stream.ts                   # Main functions
utils/event-stream.ts      # Event stream class
providers/                 # Provider implementations
  openai.ts
  anthropic.ts
  google.ts
  ...
```

---

## Topics Covered

### Core APIs
- ✅ Messages (User, Assistant, ToolResult)
- ✅ Content (Text, Thinking, Image, ToolCall)
- ✅ Context (system prompt, messages, tools)
- ✅ Tool definition and validation

### Functions
- ✅ `stream()` - Core streaming function
- ✅ `complete()` - Blocking completion
- ✅ `streamSimple()` - With reasoning
- ✅ `completeSimple()` - Reasoning + blocking
- ✅ `getModel()` - Model discovery
- ✅ `getModels()` - List models
- ✅ `getProviders()` - List providers

### Patterns
- ✅ Simple chat
- ✅ Multi-turn conversations
- ✅ Streaming to UI
- ✅ Streaming with thinking
- ✅ Tool calling loops
- ✅ Cost tracking
- ✅ Provider fallback
- ✅ Timeouts & cancellation
- ✅ Batch processing
- ✅ Parallel streaming
- ✅ Image analysis
- ✅ Custom providers

### Advanced Topics
- ✅ Event stream internals
- ✅ Partial state tracking
- ✅ Content indexing
- ✅ Provider abstraction
- ✅ Tool validation
- ✅ Error handling

---

## NOT Covered (Intentionally Omitted)

As requested, we skipped:
- ❌ OAuth providers (complex, not needed yet)
- ❌ Browser usage (desktop focus)
- ❌ Type definitions for specific providers
- ❌ Internal provider implementation details

---

## Study Statistics

**Files Analyzed**: 6 core files
- types.ts
- stream.ts
- utils/event-stream.ts
- index.ts
- README.md (in-depth)

**Code Patterns**: 12 complete examples
**Type Explanations**: 15+ core types
**Concepts Explained**: 20+ major concepts

---

## How to Use This Documentation

### For Quick Lookup
→ Use [[Pi-AI Package]] for API reference

### For Understanding Events
→ Read [[Streaming & Event System]]

### For Tools & Providers
→ Check [[Tool Calling & Providers]]

### For Implementation
→ Follow [[Pi-AI Coding Patterns]]

### For Integration with Agents
→ See [[Architecture Overview]] (how pi-ai feeds into agent-core)

---

## Connected to Broader Codebase

```
📦 Pi-Mono
├── 🤖 pi-ai (LLM abstraction) ← YOU ARE HERE
│   ├── Unified interface
│   ├── Tool calling
│   ├── Provider system
│   └── Streaming
│
├── 🏃 pi-agent-core (Agent runtime)
│   ├── Uses pi-ai for LLM calls
│   ├── Manages agent state
│   ├── Tool execution
│   └── Streaming integration
│
├── 🎯 pi-coding-agent (CLI tool)
│   ├── Uses pi-agent-core
│   ├── Built-in tools (read, write, bash)
│   └── Interactive TUI
│
├── 🎨 pi-tui (Terminal UI)
├── 📱 pi-web-ui (Web components)
└── 🤖 pi-mom (Slack integration)
```

**You now understand the bottom layer (pi-ai). Next: Study [[Architecture Overview]] to see how agents use pi-ai.**

---

## Key Files to Remember

| File | Purpose |
|------|---------|
| `packages/ai/src/types.ts` | All type definitions |
| `packages/ai/src/stream.ts` | Main export functions |
| `packages/ai/src/utils/event-stream.ts` | Event stream class |
| `packages/ai/src/index.ts` | Public API |
| `packages/ai/README.md` | Official documentation |

---

**Created**: April 10, 2026  
**Updated**: April 10, 2026  
**Status**: ✅ Complete pi-ai documentation

🎉 **You're ready to move to pi-agent-core when you want!**
