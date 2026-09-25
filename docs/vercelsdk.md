
# Subagents

A subagent is an agent that a parent agent can invoke. The parent delegates work via a tool, and the subagent executes autonomously before returning a result.

## How It Works

1. **Define a subagent** with its own model, instructions, and tools
2. **Create a tool that calls it** for the main agent to use
3. **Subagent runs independently with its own context window**
4. **Return a result** (optionally streaming progress to the UI)
5. **Control what the model sees** using `toModelOutput` to summarize

## When to Use Subagents

Subagents add latency and complexity. Use them when the benefits outweigh the costs:

| Use Subagents When                              | Avoid Subagents When           |
| ----------------------------------------------- | ------------------------------ |
| Tasks require exploring large amounts of tokens | Tasks are simple and focused   |
| You need to parallelize independent research    | Sequential processing suffices |
| Context would grow beyond model limits          | Context stays manageable       |
| You want to isolate tool access by capability   | All tools can safely coexist   |

## Why Use Subagents?

### Offloading Context-Heavy Tasks

Some tasks require exploring large amounts of information—reading files, searching codebases, or researching topics. Running these in the main agent consumes context quickly, making the agent less coherent over time.

With subagents, you can:

- Spin up a dedicated agent that uses hundreds of thousands of tokens
- Have it return only a focused summary (perhaps 1,000 tokens)
- Keep your main agent's context clean and coherent

The subagent does the heavy lifting while the main agent stays focused on orchestration.

### Parallelizing Independent Work

For tasks like exploring a codebase, you can spawn multiple subagents to research different areas simultaneously. Each returns a summary, and the main agent synthesizes the findings—without paying the context cost of all that exploration.

### Specialized Orchestration

A less common but valid pattern is using a main agent purely for orchestration, delegating to specialized subagents for different types of work. For example:

- An exploration subagent with read-only tools for researching codebases
- A coding subagent with file editing tools
- An integration subagent with tools for a specific platform or API

This creates a clear separation of concerns, though context offloading and parallelization are the more common motivations for subagents.

## Basic Subagent Without Streaming

The simplest subagent pattern requires no special machinery. Your main agent has a tool that calls another agent in its `execute` function:

```ts
import { ToolLoopAgent, tool } from 'ai';
__PROVIDER_IMPORT__;
import { z } from 'zod';

// Define a subagent for research tasks
const researchSubagent = new ToolLoopAgent({
  model: __MODEL__,
  instructions: `You are a research agent.
Summarize your findings in your final response.`,
  tools: {
    read: readFileTool, // defined elsewhere
    search: searchTool, // defined elsewhere
  },
});

// Create a tool that delegates to the subagent
const researchTool = tool({
  description: 'Research a topic or question in depth.',
  inputSchema: z.object({
    task: z.string().describe('The research task to complete'),
  }),
  execute: async ({ task }, { abortSignal }) => {
    const result = await researchSubagent.generate({
      prompt: task,
      abortSignal,
    });
    return result.text;
  },
});

// Main agent uses the research tool
const mainAgent = new ToolLoopAgent({
  model: __MODEL__,
  instructions: 'You are a helpful assistant that can delegate research tasks.',
  tools: {
    research: researchTool,
  },
});
```

This works well when you don't need to show the subagent's progress in the UI. The tool call blocks until the subagent completes, then returns the final text response.

### Handling Cancellation

When the user cancels a request, the `abortSignal` propagates to the subagent. Always pass it through to ensure cleanup:

```ts
execute: async ({ task }, { abortSignal }) => {
  const result = await researchSubagent.generate({
    prompt: task,
    abortSignal, // Cancels subagent if main request is aborted
  });
  return result.text;
},
```

If you abort the signal, the subagent stops executing and throws an `AbortError`. The main agent's tool execution fails, which stops the main loop.

To avoid errors about incomplete tool calls in subsequent messages, use `convertToModelMessages` with `ignoreIncompleteToolCalls`:

```ts
import { convertToModelMessages } from 'ai';

const modelMessages = await convertToModelMessages(messages, {
  ignoreIncompleteToolCalls: true,
});
```

This filters out tool calls that don't have corresponding results. Learn more in the [convertToModelMessages](/docs/reference/ai-sdk-ui/convert-to-model-messages) reference.

## Streaming Subagent Progress

When you want to show incremental progress as the subagent works, use [**preliminary tool results**](/docs/ai-sdk-core/tools-and-tool-calling#preliminary-tool-results). This pattern uses a generator function that yields partial updates to the UI.

### How Preliminary Tool Results Work

Change your `execute` function from a regular function to an async generator (`async function*`). Each `yield` sends a preliminary result to the frontend:

```ts
execute: async function* ({ /* input */ }) {
  // ... do work ...
  yield partialResult;
  // ... do more work ...
  yield updatedResult;
}
```

### Building the Complete Message

Each `yield` **replaces** the previous output entirely (it does not append). This means you need a way to accumulate the subagent's response into a complete message that grows over time.

The `readUIMessageStream` utility handles this. It reads each chunk from the stream and builds an ever-growing `UIMessage` containing all parts received so far:

```ts
import { readUIMessageStream, toUIMessageStream, tool } from 'ai';
import { z } from 'zod';

const researchTool = tool({
  description: 'Research a topic or question in depth.',
  inputSchema: z.object({
    task: z.string().describe('The research task to complete'),
  }),
  execute: async function* ({ task }, { abortSignal }) {
    // Start the subagent with streaming
    const result = await researchSubagent.stream({
      prompt: task,
      abortSignal,
    });

    // Each iteration yields a complete, accumulated UIMessage
    for await (const message of readUIMessageStream({
      stream: toUIMessageStream({ stream: result.stream }),
    })) {
      yield message;
    }
  },
});
```

Each yielded `message` is a complete `UIMessage` containing all the subagent's parts up to that point (text, tool calls, and tool results). The frontend simply replaces its display with each new message.

## Controlling What the Model Sees

Here's where subagents become powerful for context management. The full `UIMessage` with all the subagent's work is stored in the message history and displayed in the UI. But you can control what the main agent's model actually sees using `toModelOutput`.

### How It Works

The `toModelOutput` function maps the tool's output to the tokens sent to the model:

```ts
const researchTool = tool({
  description: 'Research a topic or question in depth.',
  inputSchema: z.object({
    task: z.string().describe('The research task to complete'),
  }),
  execute: async function* ({ task }, { abortSignal }) {
    const result = await researchSubagent.stream({
      prompt: task,
      abortSignal,
    });

    for await (const message of readUIMessageStream({
      stream: toUIMessageStream({ stream: result.stream }),
    })) {
      yield message;
    }
  },
  toModelOutput: ({ output: message }) => {
    // Extract just the final text as a summary
    const lastTextPart = message?.parts.findLast(p => p.type === 'text');
    return {
      type: 'text',
      value: lastTextPart?.text ?? 'Task completed.',
    };
  },
});
```

With this setup:

- **Users see**: The full subagent execution—every tool call, every intermediate step
- **The model sees**: Just the final summary text

The subagent might use 100,000 tokens exploring and reasoning, but the main agent only consumes the summary. This keeps the main agent coherent and focused.

### Write Subagent Instructions for Summarization

For `toModelOutput` to extract a useful summary, your subagent must produce one. Add explicit instructions like this:

```ts
const researchSubagent = new ToolLoopAgent({
  model: __MODEL__,
  instructions: `You are a research agent. Complete the task autonomously.

IMPORTANT: When you have finished, write a clear summary of your findings as your final response.
This summary will be returned to the main agent, so include all relevant information.`,
  tools: {
    read: readFileTool,
    search: searchTool,
  },
});
```

Without this instruction, the subagent might not produce a comprehensive summary. It could simply say "Done", leaving `toModelOutput` with nothing useful to extract.

## Rendering Subagents in the UI (with useChat)

To display streaming progress, check the tool part's `state` and `preliminary` flag.

### Tool Part States

| State              | Description                                |
| ------------------ | ------------------------------------------ |
| `input-streaming`  | Tool input being generated                 |
| `input-available`  | Tool ready to execute                      |
| `output-available` | Tool produced output (check `preliminary`) |
| `output-error`     | Tool execution failed                      |

### Detecting Streaming vs Complete

```tsx
const hasOutput = part.state === 'output-available';
const isStreaming = hasOutput && part.preliminary === true;
const isComplete = hasOutput && !part.preliminary;
```

### Type Safety for Subagent Output

Export types alongside your agents for use in UI components:

```ts filename="lib/agents.ts"
import { ToolLoopAgent, InferAgentUIMessage } from 'ai';

export const mainAgent = new ToolLoopAgent({
  // ... configuration with researchTool
});

// Export the main agent message type for the chat UI
export type MainAgentMessage = InferAgentUIMessage<typeof mainAgent>;
```

### Render Messages and Subagent Output

This example uses the types defined above to render both the main agent's messages and the subagent's streamed output:

```tsx
'use client';

import { useChat } from '@ai-sdk/react';
import type { MainAgentMessage } from '@/lib/agents';

export function Chat() {
  const { messages } = useChat<MainAgentMessage>();

  return (
    <div>
      {messages.map(message =>
        message.parts.map((part, i) => {
          switch (part.type) {
            case 'text':
              return <p key={i}>{part.text}</p>;
            case 'tool-research':
              return (
                <div>
                  {part.state !== 'input-streaming' && (
                    <div>Research: {part.input.task}</div>
                  )}
                  {part.state === 'output-available' && (
                    <div>
                      {part.output.parts.map((nestedPart, i) => {
                        switch (nestedPart.type) {
                          case 'text':
                            return <p key={i}>{nestedPart.text}</p>;
                          default:
                            return null;
                        }
                      })}
                    </div>
                  )}
                </div>
              );
            default:
              return null;
          }
        }),
      )}
    </div>
  );
}
```

## Caveats

### No Tool Approvals in Subagents

Subagent tools cannot use approval flows such as `toolApproval` (or the
deprecated `needsApproval`). All tools must execute automatically without user
confirmation.

### Subagent Context is Isolated

Each subagent invocation starts with a fresh context window. This is one of the key benefits of subagents: they don't inherit the accumulated context from the main agent, which is exactly what allows them to do heavy exploration without bloating the main conversation.

If you need to give a subagent access to the conversation history, the `messages` are available in the tool's execute function alongside `abortSignal`:

```ts
execute: async ({ task }, { abortSignal, messages }) => {
  const result = await researchSubagent.generate({
    messages: [
      ...messages, // The main agent's conversation history
      { role: 'user', content: task }, // The specific task for this invocation
    ],
    abortSignal,
  });
  return result.text;
},
```

Use this sparingly since passing full history defeats some of the context isolation benefits.

### Streaming Adds Complexity

The basic pattern (no streaming) is simpler to implement and debug. Only add streaming when you need to show real-time progress in the UI.

===================================================


# Tool Approvals

By default, tools with an `execute` function run automatically when the model calls them. Use `toolApproval` on `ToolLoopAgent` to review, approve, or deny selected tool calls before they execute.

`toolApproval` is useful for tools that can modify data, spend money, execute code, send messages, access private data, or perform any other sensitive action.

<Note>
  `toolApproval` applies to tools executed by the AI SDK. Provider-executed
  tools run provider-side and do not use AI SDK tool approvals.
</Note>

## Statuses

Every approval rule returns one of these statuses, either as a string or as an object with a `type` field:

- `'not-applicable'`: execute the tool normally without approval metadata. This is the default.
- `'approved'`: record an automatic approval, then execute the tool.
- `'denied'`: record an automatic denial and return a denied tool output.
- `'user-approval'`: emit an approval request and wait for an explicit response.

For automatic approvals and denials, use the object form when you want to include a reason:

```ts
toolApproval: {
  deleteFile: {
    type: 'denied',
    reason: 'Deleting files is disabled in this workspace',
  },
}
```

Approval functions can also return `undefined`, which is treated the same as `'not-applicable'`.

## Require Approval for a Tool

Use a per-tool map when each tool has a simple policy.

```ts highlight="13-15"
import { ToolLoopAgent, tool } from 'ai';
__PROVIDER_IMPORT__;
import { z } from 'zod';

const agent = new ToolLoopAgent({
  model: __MODEL__,
  tools: {
    runCommand: tool({
      inputSchema: z.object({ command: z.string() }),
      execute: async ({ command }) => runCommand(command),
    }),
  },
  toolApproval: {
    runCommand: 'user-approval',
  },
});
```

When `runCommand` is called, the agent returns a `tool-approval-request` instead of executing the tool.

## Decide Based on Tool Input

Use a per-tool approval function when the decision depends on the parsed tool input. The function receives the typed input plus `toolCallId`, `messages`, `toolContext`, and `runtimeContext`.

```ts highlight="17-24"
import { ToolLoopAgent, tool } from 'ai';
__PROVIDER_IMPORT__;
import { z } from 'zod';

const agent = new ToolLoopAgent({
  model: __MODEL__,
  tools: {
    processPayment: tool({
      inputSchema: z.object({
        amount: z.number(),
        recipient: z.string(),
      }),
      execute: async ({ amount, recipient }) =>
        processPayment({ amount, recipient }),
    }),
  },
  toolApproval: {
    processPayment: async ({ amount }, { runtimeContext }) => {
      if (runtimeContext.role !== 'admin') {
        return { type: 'denied', reason: 'Only admins can send payments' };
      }
      return amount > 1000 ? 'user-approval' : undefined;
    },
  },
});
```

In this example, non-admin users are denied automatically, large payments require manual approval, and smaller admin payments execute normally.

## Use One Policy for All Tools

Pass a function directly as `toolApproval` when approval depends on the full tool call, shared state across tools, or the complete tool set. This is called a `GenericToolApprovalFunction`.

```ts highlight="13-16,18-20,22-23"
const agent = new ToolLoopAgent({
  model: __MODEL__,
  tools: {
    readFile: tool({
      inputSchema: z.object({ path: z.string() }),
      execute: async ({ path }) => readFile(path),
    }),
    deleteFile: tool({
      inputSchema: z.object({ path: z.string() }),
      execute: async ({ path }) => deleteFile(path),
    }),
  },
  toolApproval: ({ toolCall }) => {
    if (toolCall.dynamic) {
      return 'user-approval';
    }

    if (toolCall.toolName === 'deleteFile') {
      return 'user-approval';
    }

    return undefined;
  },
});
```

The generic function receives:

- `toolCall`: the full tool call, including `toolName`, `toolCallId`, `input`, and whether it is dynamic.
- `tools`: all tools available to the model.
- `toolsContext`: context for all tools.
- `messages`: the messages sent to the model for the step that produced the tool call.
- `runtimeContext`: the call's shared runtime context.

## Configure Approval per Request

Because `toolApproval` is an agent setting, you can also return it from `prepareCall`. This is useful when approval policy depends on call options, tenant policy, or user permissions.

```ts highlight="7-17"
import { ToolLoopAgent, tool } from 'ai';
__PROVIDER_IMPORT__;
import { z } from 'zod';

const agent = new ToolLoopAgent({
  model: __MODEL__,
  callOptionsSchema: z.object({
    canRunCommands: z.boolean(),
  }),
  prepareCall: ({ options, ...settings }) => ({
    ...settings,
    toolApproval: {
      runCommand: options.canRunCommands
        ? 'user-approval'
        : { type: 'denied', reason: 'Command access is disabled' },
    },
  }),
  tools: {
    runCommand: tool({
      inputSchema: z.object({ command: z.string() }),
      execute: async ({ command }) => runCommand(command),
    }),
  },
});
```

## Handle Manual Approvals

Manual approval requires two calls:

1. Call `agent.generate()` or `agent.stream()` with `toolApproval`.
2. Read the `tool-approval-request` from the result or UI stream.
3. Ask the user or your approval system for a decision.
4. Add a `tool-approval-response` to the messages.
5. Call the agent again with the updated messages.

```ts highlight="5-6,10-18,21-24,26"
import { type ModelMessage, type ToolApprovalResponse } from 'ai';

const messages: ModelMessage[] = [{ role: 'user', content: 'Delete temp.txt' }];

const result = await agent.generate({ messages });
messages.push(...result.responseMessages);

const approvalResponses: ToolApprovalResponse[] = [];

for (const part of result.content) {
  if (part.type === 'tool-approval-request' && !part.isAutomatic) {
    approvalResponses.push({
      type: 'tool-approval-response',
      approvalId: part.approvalId,
      approved: true,
      reason: 'User confirmed the file can be deleted',
    });
  }
}

messages.push({
  role: 'tool',
  content: approvalResponses,
});

const finalResult = await agent.generate({ messages });
```

If approved, the tool runs on the second call. If denied, the model receives the denial and can respond without the tool result.

<Note>
  When a tool execution is denied, consider adding an instruction such as "When
  a tool execution is not approved, do not retry it" to prevent repeated
  approval requests for the same action.
</Note>

## Use with `useChat`

When streaming an agent to a chat UI, approval requests appear as tool parts with `state: 'approval-requested'`. Respond with `addToolApprovalResponse`.

```tsx highlight="7-9,17-40"
'use client';

import { useChat } from '@ai-sdk/react';
import { lastAssistantMessageIsCompleteWithApprovalResponses } from 'ai';

export default function Chat() {
  const { messages, addToolApprovalResponse } = useChat({
    sendAutomaticallyWhen: lastAssistantMessageIsCompleteWithApprovalResponses,
  });

  return messages.map(message =>
    message.parts.map(part => {
      if (part.type !== 'tool-runCommand') {
        return null;
      }

      if (part.state === 'approval-requested' && !part.approval.isAutomatic) {
        return (
          <div key={part.toolCallId}>
            {part.approval.requestReason && (
              <p>{part.approval.requestReason}</p>
            )}
            <button
              onClick={() =>
                addToolApprovalResponse({
                  id: part.approval.id,
                  approved: true,
                })
              }
            >
              Approve
            </button>
            <button
              onClick={() =>
                addToolApprovalResponse({
                  id: part.approval.id,
                  approved: false,
                })
              }
            >
              Deny
            </button>
          </div>
        );
      }
    }),
  );
}
```

Only call `addToolApprovalResponse` for manual approvals. Automatic approvals and denials already include approval state in the stream.
When a manual approval status includes a reason, it is available as
`part.approval.requestReason`. A reason supplied with
`addToolApprovalResponse` is stored separately as `part.approval.reason`.

### Schema transforms and persisted approvals

Approval requests preserve the original schema input in `inputSchemaInput` when
it differs from the input presented for approval. Keep this field when persisting
`responseMessages` or UI messages; the UI message conversion functions preserve it
automatically.

On continuation, the SDK reconstructs the transformed input and checks that it
matches the approved input. It never replaces the approved input with a different
value. Older or projected histories that omit the original input are rejected as
invalid tool input if revalidation fails or would change the approved value.

The original input is included in persisted messages and UI approval streams,
including fields removed by schema transforms. A transform that removes a field
does not redact it from approval metadata.

## Security Considerations

### Trust model

In the standard `useChat` pattern, the server rebuilds the conversation from the messages the client sends each turn. The server does not persist conversation state between requests. This means the message history is client-controlled input.

Tool approvals reconstructed from this history are re-validated before execution: the tool input is checked against the tool's schema, and the approval policy is re-evaluated. However, without additional protection, a client that crafts a valid-looking approval for a schema-conforming input can bypass the human-in-the-loop step.

If your tools perform sensitive operations (modifying data, spending money, calling external APIs, accessing private resources), use `experimental_toolApprovalSecret` to cryptographically bind approvals to the server that issued them.

### Signing approvals with `experimental_toolApprovalSecret`

When you provide a secret, the server HMAC-signs each approval request at issuance and verifies the signature when the approval is replayed. A forged or tampered approval is rejected before the tool executes. Configure the secret on `ToolLoopAgent` (or pass it directly to `generateText` or `streamText`).

```ts highlight="6"
const agent = new ToolLoopAgent({
  model: __MODEL__,
  tools: { deleteFile, runQuery },
  toolApproval: { deleteFile: 'user-approval', runQuery: 'user-approval' },
  experimental_toolApprovalSecret: process.env.TOOL_APPROVAL_SECRET,
});

const result = await agent.generate({
  messages,
});
```

The signature binds the approval to the exact tool name, tool call ID, and input arguments. Changing any of these after signing invalidates the approval.

**Setting up the secret:**

1. Generate a high-entropy random string (at least 32 bytes):
   ```bash
   openssl rand -base64 32
   ```
2. Store it as an environment variable accessible to all server instances:
   ```
   TOOL_APPROVAL_SECRET=your-generated-secret-here
   ```
3. Pass it to `ToolLoopAgent`, `generateText`, or `streamText` via `experimental_toolApprovalSecret`.

Every serverless instance that might handle a request needs the same secret, since one instance signs the approval and a different instance may verify it on the next turn.

**Behavior when configured:**

- Approval requests without a valid signature are rejected (fail-closed)
- No secret configured: approvals work as before (backward compatible)
- The secret is never sent to the client or included in the stream

<Note>
  `WorkflowAgent` also supports `experimental_toolApprovalSecret`. It signs in
  a workflow step before writing the durable approval request. Pass an
  environment variable reference, such as
  `{ environmentVariable: 'TOOL_APPROVAL_SECRET' }`, so the raw secret is read
  only inside signing and verification steps. Only the signature is persisted
  and sent to the client.
</Note>

## Related APIs

- Use `toolApproval` with `ToolLoopAgent`, `generateText`, and `streamText`.
- Author approval rules as code with [Policy-Based Tool Approvals](/docs/agents/policy-tool-approvals) (`@ai-sdk/policy-opa`).
- Use `needsApproval` only with [`WorkflowAgent`](/docs/agents/workflow-agent), where approvals suspend and resume durable workflow execution.
- Subagent tools cannot use `toolApproval`; see [Subagents](/docs/agents/subagents#no-tool-approvals-in-subagents).
