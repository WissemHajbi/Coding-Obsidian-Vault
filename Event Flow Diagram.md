# Agent Runtime Event Flow

> [!NOTE] Event Sequence
> User interaction triggers a complete event loop from message_start to agent_end.

## Complete Event Sequence

```mermaid
sequenceDiagram
    participant U as User
    participant A as Agent
    participant L as LLM
    participant T as Tools
    participant UI as UI

    U->>A: message_start
    Note right of A: collectMessages()
    
    A->>A: convertToLlm()
    A->>A: applySteering()
    A->>L: request(messages)
    
    L-->>A: streamResult()
    Note right of A: message_update stream
    
    loop While streaming
        A->>UI: message_update
    end
    
    A->>A: transformContext()
    Note right of A: truncate/compact messages
    
    alt Has Tool Calls
        A->>T: executeTool()
        Note right of T: Tool runs
        T-->>A: toolExecution_end
        T->>T: Stream results with onUpdate
    else No Tools
        A-->>A: toolCallComplete (none)
    end
    
    A->>L: nextMessage()
    Note right of A: awaitNextTurn()
    
    A->>UI: message_end
    A-->>A: agentLoop() end
    note right of A: turn_end
    note right of A: agentEnd
```

## Event Categories

| Event Type | When Triggered | Description |
|-----------|----------------|-------------|
| `message_start` | User sends message | Collects messages, sets initial state |
| `message_update` | LLM streaming | Shows partial responses to user |
| `message_end` | Message complete | LLM finished processing |
| `turn_end` | Between turns | Marks completion of one turn |
| `agent_start` | First user message | Beginning of agent operation |
| `agent_end` | After n loops | Agent run complete after n turns |
| `tool_execution_*` | Tool calls | Progress events during tool execution |

## State Lifecycle

```
Idle → Active → Streaming → Tooling → Processing → Completion
     ↓         ↓          ↓        ↓           ↓             ↓
Init     start         update   execute    next message   end
messages   messages    messages  results   messages    loop stop
```

## Message Collection

```typescript
collectMessages() → messages: Array
```

- Called at `message_start`
- Gathers all messages for this turn
- Used in `convertToLlm()` and `transformContext()`

---

[[Architecture Overview]] &nbsp;&nbsp; &nbsp; [[Hooks & Callbacks]]
