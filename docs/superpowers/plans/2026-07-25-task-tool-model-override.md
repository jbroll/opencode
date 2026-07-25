# Plan: Task Tool Model Override

## Spec

`docs/superpowers/specs/2026-07-25-task-tool-model-override.md`

## Task 1: Add `model` parameter to task tool schema and use it in model resolution

**Files:**
- `packages/opencode/src/tool/task.ts`

**Changes:**
1. Add `import { Provider } from "@/provider/provider"` at the top of the file
2. Add `model` field to `BaseParameterFields` (after `subagent_type`, before `task_id`)
3. Replace the model resolution block (lines 181-184) with the override logic from the spec

**Acceptance:**
- `bun typecheck` passes from `packages/opencode`

## Task 2: Update task tool description

**Files:**
- `packages/opencode/src/tool/task.txt`

**Changes:**
1. Add line about the `model` parameter to the usage notes

**Acceptance:**
- `bun typecheck` passes from `packages/opencode`

## Task 3: Add tests for model override

**Files:**
- `packages/opencode/test/tool/task.test.ts`

**Changes:**
1. Add test: dispatch with `model` override, verify the override model is used in the prompt input
2. Add test: dispatch with agent that has a configured model + `model` override, verify override wins
3. Add test: dispatch without `model` override, verify existing behavior unchanged

**Acceptance:**
- `bun test test/tool/task.test.ts` passes from `packages/opencode`
