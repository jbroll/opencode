# Task Tool Model Override

## Goal

Add an optional `model` parameter to the OpenCode `task` tool so that the calling agent can override the subagent's model at dispatch time, matching Claude Code's behavior.

## Why

The superpowers plugin annotates every task in an implementation plan with a `Model:` tier (haiku/sonnet/opus) to control cost. Claude Code accepts a `model` parameter on subagent dispatch. OpenCode's `task` tool currently has no such parameter — the model is fixed per agent definition or inherited from the parent. This means all subagents in OpenCode run on the same model regardless of task complexity.

## Changes

### 1. Add `model` to task tool parameters

**File:** `packages/opencode/src/tool/task.ts`

Add an optional `model` string field to `BaseParameterFields` (line 43-52):

```typescript
const BaseParameterFields = {
  description: Schema.String.annotate({ description: "A short (3-5 words) description of the task" }),
  prompt: Schema.String.annotate({ description: "The task for the agent to perform" }),
  subagent_type: Schema.String.annotate({ description: "The type of specialized agent to use for this task" }),
  model: Schema.optional(Schema.String).annotate({
    description: "Override the model for this subagent. Format: provider/model-name (e.g. anthropic/claude-haiku-4-20250514). Overrides the agent's configured model.",
  }),
  task_id: Schema.optional(Schema.String).annotate({
    description:
      "This should only be set if you mean to resume a previous task (you can pass a prior task_id and the task will continue the same subagent session as before instead of creating a fresh one)",
  }),
  command: Schema.optional(Schema.String).annotate({ description: "The command that triggered this task" }),
}
```

### 2. Use the model override in model resolution

**File:** `packages/opencode/src/tool/task.ts`

Replace lines 181-184:

```typescript
const model = next.model ?? {
  modelID: msg.info.modelID,
  providerID: msg.info.providerID,
}
```

With:

```typescript
const override = params.model ? Provider.parseModel(params.model) : undefined
const model = override ?? next.model ?? {
  modelID: msg.info.modelID,
  providerID: msg.info.providerID,
}
```

This requires adding the import at the top of the file:

```typescript
import { Provider } from "@/provider/provider"
```

### 3. Update tool description

**File:** `packages/opencode/src/tool/task.txt`

Add a line about the model parameter:

```
8. You can override the subagent's model by passing a `model` parameter (format: provider/model-name). This is useful for cost optimization — use cheaper models for simple tasks and more capable models for complex ones.
```

## Model Resolution Order

1. `params.model` (override from calling agent) — new
2. `next.model` (agent's configured model from definition)
3. Parent's current model (inherited fallback)

## What This Does NOT Change

- Agent definitions continue to work as before — the `model` field in agent frontmatter is still respected
- The `@agent` syntax model override in `prompt.ts` is unchanged
- No validation that the model exists is added (the LLM call will fail naturally if the model is unavailable, matching the existing behavior for agent-configured models)

## Testing

1. Create a test agent with no model configured, dispatch it with and without the `model` parameter, verify the correct model is used
2. Create a test agent with a model configured, dispatch it with a `model` override, verify the override takes precedence
3. Verify existing agents with configured models continue to work without the override
