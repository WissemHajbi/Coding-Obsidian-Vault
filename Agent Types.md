This note lists core data types used by the Agent runtime. Hooks and callbacks are documented in [[Agent Hooks & Callbacks]].

- AgentMessage
  - Basic transcript unit. Common roles: `user`, `assistant`, `toolResult`.
  - Messages may contain content blocks, attachments, timestamps, and custom fields via declaration merging.

- AgentEvent (common ones)
  - agent_start / agent_end
  - turn_start / turn_end
  - message_start / message_update / message_end
  - tool_execution_start / tool_execution_update / tool_execution_end
  - Events include metadata (timestamps, message ids, deltas for streaming).

- AgentTool (shape)
  - name: string
  - label?: string
  - description?: string
  - parameters: Type.Object(...) // TypeBox schema
  - execute(toolCallId: string, params: any, signal: AbortSignal, onUpdate?: (update)=>void): Promise<{content, details}>
  - execute should throw on failure; return content blocks and optional details on success.

- AgentToolCall / ToolResult
  - Tool calls appear as structured toolCall objects; results are inserted back as `toolResult` messages consumed by the LLM.

- AgentState (summary)
  - systemPrompt, model, thinkingLevel, tools, messages, isStreaming, streamingMessage?, pendingToolCalls, errorMessage?

- AgentLoopConfig (overview)
  - convertToLlm(messages): AgentMessage[] → LLM Message[]
  - transformContext(messages, signal): optional pruning/compaction
  - Other runtime hooks are in [[Agent Hooks & Callbacks]]

Why read this

- Types define contracts across packages (agent, ai, coding-agent, web-ui).
- When building tools or UIs, implement shapes compatible with these types.