# Example Code Snippets

---

## Basic Agent Setup

```typescript
import { agent, agentWithTools } from '@harne';

// Create basic agent
const basicAgent = agent({
    llm: {
        model: {
            name: 'gpt-4o',
            apiProvider: { name: 'openai' }
        }
    },
    config: {
        maxTokens: 2048
    }
});

// Create agent with tools
const toolsAgent = agentWithTools({
    llm: { ... }
}, {
    tools: [
        {
            name: 'fetch',
            description: 'Retrieve URL content',
            inputSchema: { type: 'object', properties: { url: { type: 'string' } } }
        }
    ]
});
```

---

## Basic Hooks Implementation

```typescript
const myAgent = agent({
    llm: { ... },
    config: {
        maxTokens: 2048
    },
    hooks: {
        // Message collection
        collectMessages: () => {
            return myAgent.messages;
        },
        
        // Convert messages to LLM format
        convertToLlm: (messages) => {
            return {
                messages: [
                    { role: 'system', content: 'You are helpful' },
                    ...messages
                ]
            };
        },
        
        // Transform context (prune history)
        transformContext: (messages) => {
            if (messages.length > 20) {
                return messages.slice(-20);
            }
            return messages;
        },
        
        // Block dangerous tools
        beforeToolCall: (definition) => {
            const dangerous = ['bash', 'write'];
            if (dangerous.includes(definition.name)) {
                console.log(`⚠ Blocked tool: ${definition.name}`);
                return false;
            }
            return true;
        },
        
        // Stream partial responses
        onMessageUpdate: (message) => {
            document.getElementById('response').textContent = message.content;
        },
        
        // Add metadata
        afterToolCall: (result) => {
            result.metadata = { executedAt: new Date() };
            return result;
        }
    }
});
```

---

## Advanced Agent with System Prompt

```typescript
const advancedAgent = agent({
    llm: { ... },
    config: { ... },
    hooks: {
        // System prompt management
        collectMessages: () => {
            // Your app messages only (not including system)
            return myAppMessages;
        }
    }
});

// System prompts injected automatically into context
// They take effect at next agent start
advancedAgent.systemMessages = [
    { role: 'system', content: 'You are a professional assistant' }
];
```

---

## Debug Hooks

```typescript
const debugAgent = agent({
    llm: { ... },
    config: { ... },
    hooks: {
        // Debug every step
        collectMessages: () => {
            console.log('📦 Messages collected:', myAgent.messages.length);
            return myAgent.messages;
        },
        
        convertToLlm: (messages) => {
            console.log('📤 Sending to LLM:', messages.length, 'messages');
            return myAgent.messages;
        },
        
        transformContext: (messages) => {
            console.log('🔄 Context transformed:', messages.length, 'messages');
            return messages;
        },
        
        onMessageUpdate: (message) => {
            console.log('💬 Streaming:', message.content.substring(0, 50));
            return message;
        },
        
        agentStart: () => {
            console.log('🎬 Agent started');
        },
        
        onAgentEnd: () => {
            console.log('✋ Agent ended');
        }
    }
});
```

---

## Event Monitoring

```typescript
const eventAgent = agent({
    llm: { ... },
    config: { ... },
    hooks: {
        agentStart: () => {
            // Called when agent execution starts
            console.log('Agent running');
        },
        
        message_start: () => {
            // Called at message start
            console.log('New message event');
        },
        
        message_end: () => {
            // Called when message processing complete
            console.log('Message processed');
        },
        
        toolExecution: (result) => {
            // Called after tool execution complete
            console.log('Tool completed:', result.data);
        },
        
        onAgentEnd: () => {
            // Called when agent execution stops
            console.log('Agent execution stopped');
        }
    }
});
```

---

## Aborting Execution

```typescript
const abortableAgent = agent({
    llm: { ... },
    config: { ... },
    hooks: {
        onMessageUpdate: (message) => {
            // Check abort request
            if (userRequestedAbort) {
                abortAgent();  // Stop streaming but allow resume
                return message;
            }
            return message;
        },
        
        onAgentEnd: () => {
            // Check if user requested abort
            if (userRequestedAbort) {
                abortRun();  // Stop entire run
            }
        }
    }
});
```

---

[[Architecture Overview]] &nbsp;&nbsp; &nbsp; [[Event Flow Diagram]] &nbsp;&nbsp; &nbsp; [[Hooks & Callbacks]]
