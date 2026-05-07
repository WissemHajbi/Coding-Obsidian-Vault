# AI Harness Architecture

> [!INFO] Core Concept
> An agent is a stateful loop that translates user messages to LLM requests, executes tools, and streams results to the UI.

## High-Level Flow

```
┌────────────────────────────────────────────────────────────────────┐
│  User Input                                                          │
└────────────────────────────────────────────────────────────────────┘
                               ▼
┌────────────────────────────────────────────────────────────────────┐
│  Agent.createAgent() → Agent Instance                               │
│  • Converts app messages → LLM format                               │
│  • Applies hooks                                                    │
│  • Creates streaming loop                                          │
└────────────────────────────────────────────────────────────────────┘
                               ▼
┌────────────────────────────────────────────────────────────────────┐
│  agentLoop() → emits AgentEvent stream                              │
│  • Each event: message_start → LLM → streaming → message_end       │
│  • Events: toolExecution*, messageUpdate, turn_end, agent_end      │
└────────────────────────────────────────────────────────────────────┘
```

## LLM Conversion Point

**`convertToLlm(messages)`** is the primary hook that transforms your app's message format to what the LLM expects:

```typescript
messages = convertToLlm(messages);  // app format → LLM format
```

This happens **before every LLM call**, giving you full control over:
- What messages are sent
- Message formatting/ordering
- System prompts injection

---

[[Event Flow Diagram]] &nbsp;&nbsp; &nbsp; [[Hooks & Callbacks]] &nbsp;&nbsp; &nbsp; [[Safety & Control]]
