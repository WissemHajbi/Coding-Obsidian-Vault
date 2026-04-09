This note documents the runtime hooks and callbacks the Agent loop expects. Use this with [[agent-types-summary]].

- convertToLlm(messages)
  - Purpose: transform app-level AgentMessage[] into provider-compatible LLM Message[] (filter UI-only messages, flatten attachments, format content).
  - Example (pseudo):
    ```ts
    function convertToLlm(messages) {
      return messages.flatMap(m => {
        if (m.role === 'artifact') return []; // UI only
        if (m.role === 'user-with-attachments') return [{ role: 'user', content: m.content + ' ' + m.attachments.map(a=>a.text).join('\n') }];
        return [m];
      });
    }
    ```

- transformContext(messages, signal)
  - Purpose: prune, compact, or inject external context before convertToLlm runs (e.g., summarize old messages, add system prompts).
  - Should be async and respect AbortSignal.
  - Example:
    ```ts
    async function transformContext(messages, signal) {
      // Keep last N tokens or replace long history with a summary
      return pruneMessages(messages, { maxTokens: 4000 });
    }
    ```

- beforeToolCall({ toolCall, args, context })
  - Run after args are validated but before executing a tool. Can block a tool call by returning { block: true, reason }.
  - Use cases: enforce policy, disable dangerous tools, confirm with user.
  - Example:
    ```ts
    async function beforeToolCall({ toolCall }) {
      if (toolCall.name === 'bash' && !allowBash) return { block: true, reason: 'bash disabled' };
      return undefined;
    }
    ```

- afterToolCall({ toolCall, result, isError, context })
  - Postprocess tool results before they are converted into the final toolResult message. Can add metadata.
  - Example:
    ```ts
    async function afterToolCall({ toolCall, result }) {
      if (!isError) return { details: { ...result.details, audited: true } };
      return undefined;
    }
    ```

- getSteeringMessages() & getFollowUpMessages()
  - Agent exposes queues for steering and follow-up messages. The loop calls these callbacks to inject queued messages while a run is in progress.
  - Steering interrupts tool execution and is prioritized; follow-ups run when no tools are pending.

Tips
- Keep convertToLlm and transformContext deterministic and fast — they run before each LLM call.
- Use beforeToolCall to enforce safety checks. Do not perform long-running side effects here.
- Use onUpdate in tool execute() to stream partial results back to the agent and UIs.

See [[Agent Runtime Flow]] for how hooks fit into the event sequence.