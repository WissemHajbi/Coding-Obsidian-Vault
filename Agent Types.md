# Agent Types Overview

> [!NOTE] Quick Reference
> Agents differ in structure and behavior. Choose the right type for your app.

---

## Basic Agents

### 1. `agent`

**Purpose:** Run LLM requests and stream results.

**Lifecycle:**

```
User input → collectMessages → convertToLlm
    ↓
convertToLlm → LLM streaming → collectMessages → ...
    ↓
LLM streaming → transformContext → nextMessage
```

**Key events:**
- `message_start`
- `message_update`
- `message_end`
- `turn_end`

**Use case:** Simple chat apps, basic assistants.

---

### 2. `agentWithTools`

**Purpose:** Execute tools during streaming.

**Additional behavior:**

```
Basic agent flow + executeTool + Stream tool results
```

**Key events:**
- All basic agent events
- `tool_call_complete`
- `toolExecution_*`

**Use case:** Apps that need to run external tools/functions.

---

### 3. `agentWithSystemPrompt`

**Purpose:** System prompt managed by harness.

**Key change:**

- System prompt is stored in `messages.systemMessages`
- Harness injects into LLM context automatically
- Changes only take effect at next agent start

**Use case:** Multi-turn conversations with consistent system instructions.

---

## Advanced Agent

### 4. `agentFlow` (Basic + Advanced)

**Purpose:** Combine all features with enhanced control.

**Features:**
- All basic/advanced agent features
- Tool execution with streaming
- Enhanced system prompt management
- Full lifecycle control

**Lifecycle:**

```
start → collectMessages → convertToLlm
    ↓
LLM streaming → collectMessages / transformContext
    ↓
executeTool (if tools) → Stream results → collectMessages
    ↓
turn_end → onAgentEnd → onSteeringEnd
```

**Use case:** Full-featured apps needing maximum control.

---

[[1.-Architecture Overview]] &nbsp;&nbsp; &nbsp; [[2.-Event Flow Diagram]] &nbsp;&nbsp; &nbsp; [[3.-Hooks & Callbacks]]
