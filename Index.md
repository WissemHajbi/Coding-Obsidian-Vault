# AI HARNESS

> **AI HARNESS** is a powerful orchestrator for AI agent execution. It provides lifecycle control, tool integration, streaming support, and full event monitoring.

---

## 📖 Table of Contents

### Pi-AI (LLM Abstraction Layer)
- [[Pi-AI Package]] — Unified LLM API reference
- [[Streaming & Event System]] — Real-time response events
- [[Tool Calling & Providers]] — Tool definitions and execution
- [[Pi-AI Coding Patterns]] — Common patterns and best practices

### Agent Framework (Built on Pi-AI)
- [[Architecture Overview]] — Core agent concepts and lifecycle
- [[Event Flow Diagram]] — Agent execution flow
- [[Hooks & Callbacks]] — Agent hooks implementation
- [[Example Code]] — Agent usage examples

---

## 🚀 Quick Links

| Goal                     | Link                         |
| ------------------------ | ---------------------------- |
| **Learn LLM API**        | [[Pi-AI Package]]            |
| **Understand Streaming** | [[Streaming & Event System]] |
| **Use Tools**            | [[Tool Calling & Providers]] |
| **Learn Agents**         | [[Architecture Overview]]    |
| **See Examples**         | [[Example Code]]             |

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

### [[Architecture Overview]]

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

### [[Event Flow Diagram]]

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

### [[Hooks & Callbacks]]

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

### Quick Start

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

### [[Example Code]]

Real usage examples for common patterns.

**Simple Chat:**
- Direct message handling without tools

**Tool Execution:**
- Automatic tool invocation and result handling

**Aborting:**
- Check and handle abort requests gracefully

**Context Management:**
- Smart pruning and message history management

**Types:**
- `Agent` — Agent interface
- `Hook` — Hook callback signature
- `Tool` — Tool interface
- `Message` — Message object

---
## 🔖 All Topics

### Pi-AI (Foundation Layer)
- [[Pi-AI Package]] — API reference & types
- [[Streaming & Event System]] — Real-time events
- [[Tool Calling & Providers]] — Tools & 20+ providers
- [[Pi-AI Coding Patterns]] — 12 coding patterns
- [[Pi-AI Study Summary]] — Complete study guide ← **Start here**

### Agent Framework (Built on Pi-AI)
- [[Architecture Overview]] — Agent concepts & lifecycle
- [[Event Flow Diagram]] — Agent execution flow
- [[Hooks & Callbacks]] — Agent hooks implementation


