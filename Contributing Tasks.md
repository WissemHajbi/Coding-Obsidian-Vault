Small, progressive tasks to learn the code and contribute.

1. Read & annotate
   - Read `src/types.ts` and add inline notes about each type in this vault (see [[Agent Types]]).
   - Read hooks and callbacks in [[Agent Hooks & Callbacks]] and add examples.

2. Reproduce the runtime flow
   - Paste key function signatures from `src/agent.ts` and `src/agent-loop.ts` into a note and annotate the steps.

3. Add a toy tool
   - Implement a simple `read_file` tool in a local example (reads file contents). Test by wiring it into a small script that uses the Agent API.

4. Tests
   - Add a unit test for `streamAssistantResponse` mocking a fake stream that sends tokens and check emitted events.

5. Docs
   - Improve README examples with a small tutorial showing tool calls in the coding-agent CLI.

How to mark progress

- Add checkboxes below as you finish tasks and tag issues in GitHub when ready to submit PRs.

- When ready for a PR, run `npm run check` and fix TypeScript/biome warnings before opening PR.
