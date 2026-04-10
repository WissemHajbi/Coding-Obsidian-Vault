# AI HARNESS

> **AI HARNESS** is a powerful orchestrator for AI agent execution. It provides lifecycle control, tool integration, streaming support, and full event monitoring.

---

## 📖 Table of Contents

- [[1.-Architecture Overview]] — Core concepts and lifecycle
- [[2.-Event Flow Diagram]] — Execution flow and debugging
- [[3.-Hooks & Callbacks]] — Hooks implementation examples
- [[4.-Quick Start]] — Get started quickly
- [[5.-Examples]] — Real usage examples
- [[6.-API Reference]] — Complete API docs
- [[7.-Migration Guide]] — Version compatibility

---

## 🚀 Quick Links

| Goal | Link |
|------|------|
| **Start Building** | [[4.-Quick Start]] |
| **Understand Flow** | [[2.-Event Flow Diagram]] |
| **Use Hooks** | [[3.-Hooks & Callbacks]] |
| **View API** | [[6.-API Reference]] |
| **Read Examples** | [[5.-Examples]] |

---

## 🔑 Core Features

- 🎯 **Lifecycle Control** — Hooks at every execution step
- 🛠️ **Tool Integration** — Automatic tool execution and management
- 📡 **Streaming Support** — Stream partial LLM responses
- 🧹 **Context Pruning** — Smart history management
- 🔊 **System Prompts** — Managed system instructions
- 🐛 **Debug Mode** — Full visibility into execution

---

## 📚 Detailed Topics

### [[1.-Architecture Overview]]

The core architecture and how AI HARNESS works.

**Key Concepts:**
- Agent lifecycle
- Hook system
- Signal-based control
- Event flow

**Lifecycle Stages:**
1. Initialization
2. Execution
3. Message Processing
4. Tool Invocation
5. Response Generation
6. Cleanup

**Hooks Available:**
- `collectMessages`
- `convertToLlm`
- `transformContext`
- `onMessageUpdate`
- `agentStart`
- `onAgentEnd`
- `start`, `end`, `before`, `after`

### [[2.-Event Flow Diagram]]

Visual representation of execution flow.

**Basic Flow:**
```
Start → collectMessages → convertToLlm
     → transformContext → LLM Request
     → onMessageUpdate → Streaming Updates
     → onAgentEnd → Cleanup
```

**With Tools:**
```
LLM Request → Tool Selection
     → Tool Execution → Tool Result
     → Context Update → LLM Update
```

**Error Handling:**
```
Error → Error Collection → Error Message
     → Optional Retry → Final Response
```

### [[3.-Hooks & Callbacks]]

Implementing hooks for custom behavior.

**Hook Categories:**

**Collection Hooks:**
- `collectMessages` — Your app messages only
- `convertToLlm` — Transform before LLM send
- `transformContext` — Context transformation

**Execution Hooks:**
- `onMessageUpdate` — Stream partial responses
- `agentStart` — Before execution begins
- `onAgentEnd` — After execution ends

**Event Hooks:**
- `agentStart` — When agent starts
- `message_start` — When message starts
- `message_end` — When message completes
- `toolExecution` — Tool completion result

### [[4.-Quick Start]]

Get started with minimal code.

**Basic Agent:**
```typescript
const agent = agent({
    llm: { ... },
    config: { ... },
    hooks: {
        onMessageUpdate: (message) => {
            console.log(message.content);
        }
    }
});
```

**With Tools:**
```typescript
const agentWithTools = agent({
    llm: { ... },
    config: {
        tools: myTools
    }
});
```

**System Prompts:**
```typescript
advancedAgent.systemMessages = [
    { role: 'system', content: 'You are a professional assistant' }
];
```

### [[5.-Examples]]

Real usage examples for common patterns.

**Simple Chat:**
- Direct message handling without tools

**Tool Execution:**
- Automatic tool invocation and result handling

**Aborting:**
- Check and handle abort requests gracefully

**Context Management:**
- Smart pruning and message history management

### [[6.-API Reference]]

Complete API documentation.

**Core Functions:**
- `agent()` — Create a new agent
- `agentWithTools()` — Create agent with tools
- `abortAgent()` — Signal abort without stopping
- `abortRun()` — Stop entire agent run

**Config Options:**
- `llm` — LLM configuration
- `config` — General config (maxTokens, temperature, etc.)
- `hooks` — Available hook callbacks
- `systemMessages` — Persistent system instructions

**Types:**
- `Agent` — Agent interface
- `Hook` — Hook callback signature
- `Tool` — Tool interface
- `Message` — Message object

### [[7.-Migration Guide]]

How to upgrade from other libraries.

**From Other Libraries:**
- Map your hooks to AI HARNESS hooks
- Adapt your config to new format
- Update error handling patterns
- Modify context management

**Version Compatibility:**
- v1 → v2 — Breaking changes, migration needed
- Features added: Tool integration, improved hooks

---

## 🔖 Related Documentation

- [[Architecture Overview]] — Core concepts
- [[Event Flow Diagram]] — Execution flow
- [[Hooks & Callbacks]] — Hook implementations
- [[Quick Start]] — Get started
- [[Examples]] — Usage examples
- [[API Reference]] — Full API docs
- [[Migration Guide]] — Version compatibility
- [[Support]] — Help and FAQ

---

## 📢 Notes

- This is the **latest documentation**
- Features are actively being added
- Check the migration guide for version updates
- Report issues on the project repository

---

[[1.-Architecture Overview]] &nbsp;&nbsp; &nbsp; [[2.-Event Flow Diagram]] &nbsp;&nbsp; &nbsp; [[3.-Hooks & Callbacks]] &nbsp;&nbsp; &nbsp; [[4.-Quick Start]] &nbsp;&nbsp; &nbsp; [[5.-Examples]]
[[6.-API Reference]] &nbsp;&nbsp; &nbsp; [[7.-Migration Guide]]
