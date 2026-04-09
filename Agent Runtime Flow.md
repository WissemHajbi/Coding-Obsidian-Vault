Simple sequence of what happens when you call agent.prompt("Hello")

1. UI calls `agent.prompt("Hello")`.
2. Agent builds a LoopConfig that contains:
   - convertToLlm (how to turn app messages into LLM messages)
   - transformContext (prune or inject context)
   - hooks: beforeToolCall, afterToolCall
   - callbacks to fetch queued steering/follow-ups
3. Low-level `agentLoop` starts and emits `agent_start` and `turn_start` events.
4. It emits `message_start` for the user message and then calls the LLM using `streamFn`.
5. While the model streams partial tokens, `message_update` events are emitted with the deltas — UIs append these to the assistant message.
6. If the assistant requests a tool, the loop:
   - emits `tool_execution_start`
   - runs `beforeToolCall` (can block)
   - executes tool(s) (parallel or sequential)
   - emits `tool_execution_update` as tool streams progress
   - emits `tool_execution_end` and inserts a `toolResult` message
7. The loop resumes LLM turns; assistant sees tool results and may continue.
8. When done: `message_end`, `turn_end`, then possibly another turn or `agent_end`.

ASCII diagram

UI → Agent.prompt()
Agent → agentLoop → LLM stream → Agent emits message_update → UI
Assistant -> ToolCall -> Agent executes tool -> emits tool_execution_* -> LLM continues

Key events: agent_start, message_start, message_update, message_end, tool_execution_start/update/end, turn_end, agent_end.
