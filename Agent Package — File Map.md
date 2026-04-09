- src/index.ts
  - Barrel that re-exports the public API (Agent, agentLoop, types).

- src/types.ts
  - Public interfaces and types: contracts other code relies on.

- src/agent.ts
  -  *=**prompt()/continue()/steer()/followUp()*,** manages subscribers and run lifecycle.
  - Look at: ***constructor, prompt(), createLoopConfig(), runWithLifecycle(), subscribe()***.

- src/agent-loop.ts
  - Low-level loop that runs LLM turns, streams assistant responses, detects and executes tool calls, and emits AgentEvent streams.
  - Key functions: agentLoop(), agentLoopContinue(), streamAssistantResponse(), executeToolCalls().

- src/proxy.ts
  - streamProxy helper for client/server proxying of model streams — reconstructs partial assistant messages from compact proxy events.

- README.md
  - Examples and usage; good for learning how Agent is used by UIs. 