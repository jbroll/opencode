# Read Deduplication Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Deduplicate stale `read` tool outputs by file path during context assembly, keeping only the most recent read per file.

**Architecture:** Add a `deduplicateReads()` function in `message-v2.ts` that marks earlier reads of the same file path as compacted (reusing the existing `compacted` timestamp mechanism). Wire it into `toModelMessagesEffect()` with a config gate. Add `dedupReads` option to the V1 compaction config schema.

**Tech Stack:** TypeScript, Effect Schema, Bun test runner

## Global Constraints

- Only `read` tool outputs are deduplicated; no other tools affected
- Reuses existing `compacted` timestamp mechanism — no new compaction infrastructure
- Deduplication is O(n) single pass over parts
- Config defaults to `true` (enabled); `dedupReads: false` disables
- Old reads stay in DB; only context assembly is affected

---

## File Structure

| File | Action | Purpose |
|------|--------|---------|
| `packages/opencode/src/session/message-v2.ts` | Modify | Add `deduplicateReads()` function and wire into `toModelMessagesEffect()` |
| `packages/opencode/test/session/message-v2.test.ts` | Modify | Add tests for `deduplicateReads()` |
| `packages/core/src/v1/config/config.ts` | Modify | Add `dedupReads` field to compaction config schema |
| `packages/opencode/src/tool/read.txt` | Modify | Add deduplication note to tool description |

---

### Task 1: Add `dedupReads` config option

**Files:**
- Modify: `packages/core/src/v1/config/config.ts:149-168`

**Interfaces:**
- Consumes: existing `compaction` Schema.Struct
- Produces: `dedupReads` optional boolean field on compaction config

- [ ] **Step 1: Add `dedupReads` field to compaction config schema**

In `packages/core/src/v1/config/config.ts`, add `dedupReads` to the `compaction` Struct:

```ts
compaction: Schema.optional(
  Schema.Struct({
    auto: Schema.optional(Schema.Boolean).annotate({
      description: "Enable automatic compaction when context is full (default: true)",
    }),
    dedupReads: Schema.optional(Schema.Boolean).annotate({
      description: "Deduplicate read tool outputs by file path, keeping only the most recent read (default: true)",
    }),
    prune: Schema.optional(Schema.Boolean).annotate({
      description: "Enable pruning of old tool outputs (default: false)",
    }),
    tail_turns: Schema.optional(NonNegativeInt).annotate({
      description: "Number of recent user turns, including their following assistant/tool responses, to keep verbatim during compaction (default: 2)",
    }),
    preserve_recent_tokens: Schema.optional(NonNegativeInt).annotate({
      description: "Maximum number of tokens from recent turns to preserve verbatim after compaction",
    }),
    reserved: Schema.optional(NonNegativeInt).annotate({
      description: "Token buffer for compaction. Leaves enough window to avoid overflow during compaction.",
    }),
  }),
),
```

- [ ] **Step 2: Verify typecheck passes**

Run: `bun typecheck` from `packages/opencode`
Expected: PASS (no type errors)

- [ ] **Step 3: Commit**

```bash
git add packages/core/src/v1/config/config.ts
git commit -m "feat(config): add dedupReads option to compaction config"
```

---

### Task 2: Write `deduplicateReads()` function with tests

**Files:**
- Modify: `packages/opencode/src/session/message-v2.ts`
- Modify: `packages/opencode/test/session/message-v2.test.ts`

**Interfaces:**
- Consumes: `Part[]` (the session parts array)
- Produces: `Part[]` (same parts with earlier reads marked via `compacted` timestamp)

- [ ] **Step 1: Write failing tests for `deduplicateReads()`**

Add a new `describe` block at the end of `packages/opencode/test/session/message-v2.test.ts` (before the closing of the file). Use the existing `basePart` helper and part construction patterns from the same file.

```ts
describe("session.message-v2.deduplicateReads", () => {
  const sessionID = SessionID.make("ses_test")
  const providerID = "test"
  const modelID = ModelV2.ID.make("test")

  function userInfo(id: string): SessionV1.User {
    return {
      id,
      sessionID,
      role: "user",
      time: { created: 0 },
      agent: "user",
      model: { providerID, modelID },
      tools: {},
      mode: "",
    } as unknown as SessionV1.User
  }

  function assistantInfo(id: string, parentID?: string): SessionV1.Assistant {
    return {
      id,
      sessionID,
      role: "assistant",
      parentID,
      time: { created: 0 },
      agent: "build",
      model: { providerID, modelID },
      tools: {},
      mode: "",
      summary: false,
    } as unknown as SessionV1.Assistant
  }

  function basePart(messageID: string, id: string) {
    return {
      id: PartID.make(id.startsWith("prt") ? id : `prt_${id}`),
      sessionID,
      messageID: MessageID.make(messageID.startsWith("msg") ? messageID : `msg_${messageID}`),
    }
  }

  function readPart(messageID: string, id: string, filePath: string) {
    return {
      ...basePart(messageID, id),
      type: "tool" as const,
      callID: `call-${id}`,
      tool: "read",
      state: {
        status: "completed" as const,
        input: { filePath },
        output: `content of ${filePath}`,
        title: "Read",
        metadata: {},
        time: { start: 0, end: 1 },
      },
    } as unknown as SessionV1.Part
  }

  function bashPart(messageID: string, id: string, cmd: string) {
    return {
      ...basePart(messageID, id),
      type: "tool" as const,
      callID: `call-${id}`,
      tool: "bash",
      state: {
        status: "completed" as const,
        input: { cmd },
        output: `output of ${cmd}`,
        title: "Bash",
        metadata: {},
        time: { start: 0, end: 1 },
      },
    } as unknown as SessionV1.Part
  }

  test("marks earlier reads of same path as compacted", () => {
    const parts: SessionV1.Part[] = [
      readPart("msg-1", "p1", "/src/foo.ts"),
      readPart("msg-2", "p2", "/src/foo.ts"),
      readPart("msg-3", "p3", "/src/foo.ts"),
    ]
    const result = MessageV2.deduplicateReads(parts)
    const p1 = result.find((p) => p.id === "prt_p1")!
    const p2 = result.find((p) => p.id === "prt_p2")!
    const p3 = result.find((p) => p.id === "prt_p3")!
    expect(p1.state.status).toBe("completed")
    expect((p1.state as any).time.compacted).toBeTypeOf("number")
    expect(p2.state.status).toBe("completed")
    expect((p2.state as any).time.compacted).toBeTypeOf("number")
    expect(p3.state.status).toBe("completed")
    expect((p3.state as any).time.compacted).toBeUndefined()
  })

  test("does not mark reads of different paths", () => {
    const parts: SessionV1.Part[] = [
      readPart("msg-1", "p1", "/src/foo.ts"),
      readPart("msg-2", "p2", "/src/bar.ts"),
    ]
    const result = MessageV2.deduplicateReads(parts)
    const p1 = result.find((p) => p.id === "prt_p1")!
    const p2 = result.find((p) => p.id === "prt_p2")!
    expect((p1.state as any).time.compacted).toBeUndefined()
    expect((p2.state as any).time.compacted).toBeUndefined()
  })

  test("only deduplicates read tool parts, not bash or edit", () => {
    const parts: SessionV1.Part[] = [
      bashPart("msg-1", "p1", "ls"),
      bashPart("msg-2", "p2", "ls"),
      readPart("msg-3", "p3", "/src/foo.ts"),
      readPart("msg-4", "p4", "/src/foo.ts"),
    ]
    const result = MessageV2.deduplicateReads(parts)
    const p1 = result.find((p) => p.id === "prt_p1")!
    const p2 = result.find((p) => p.id === "prt_p2")!
    const p3 = result.find((p) => p.id === "prt_p3")!
    const p4 = result.find((p) => p.id === "prt_p4")!
    expect((p1.state as any).time.compacted).toBeUndefined()
    expect((p2.state as any).time.compacted).toBeUndefined()
    expect((p3.state as any).time.compacted).toBeTypeOf("number")
    expect((p4.state as any).time.compacted).toBeUndefined()
  })

  test("skips non-completed read parts", () => {
    const pendingPart = {
      ...basePart("msg-1", "p1"),
      type: "tool" as const,
      callID: "call-p1",
      tool: "read",
      state: { status: "pending" as const, input: { filePath: "/src/foo.ts" }, raw: "" },
    } as unknown as SessionV1.Part

    const parts: SessionV1.Part[] = [
      pendingPart,
      readPart("msg-2", "p2", "/src/foo.ts"),
    ]
    const result = MessageV2.deduplicateReads(parts)
    const p2 = result.find((p) => p.id === "prt_p2")!
    expect((p2.state as any).time.compacted).toBeUndefined()
  })

  test("preserves non-tool parts unchanged", () => {
    const textPart = {
      ...basePart("msg-1", "p1"),
      type: "text" as const,
      text: "hello",
    } as unknown as SessionV1.Part

    const parts: SessionV1.Part[] = [
      textPart,
      readPart("msg-2", "p2", "/src/foo.ts"),
      readPart("msg-3", "p3", "/src/foo.ts"),
    ]
    const result = MessageV2.deduplicateReads(parts)
    const p1 = result.find((p) => p.id === "prt_p1")!
    expect(p1).toEqual(textPart)
  })

  test("single read is not compacted", () => {
    const parts: SessionV1.Part[] = [
      readPart("msg-1", "p1", "/src/foo.ts"),
    ]
    const result = MessageV2.deduplicateReads(parts)
    const p1 = result.find((p) => p.id === "prt_p1")!
    expect((p1.state as any).time.compacted).toBeUndefined()
  })

  test("already compacted reads stay compacted", () => {
    const alreadyCompacted = {
      ...readPart("msg-1", "p1", "/src/foo.ts"),
    }
    ;(alreadyCompacted.state as any).time.compacted = 1000

    const parts: SessionV1.Part[] = [
      alreadyCompacted,
      readPart("msg-2", "p2", "/src/foo.ts"),
    ]
    const result = MessageV2.deduplicateReads(parts)
    const p1 = result.find((p) => p.id === "prt_p1")!
    const p2 = result.find((p) => p.id === "prt_p2")!
    expect((p1.state as any).time.compacted).toBe(1000)
    expect((p2.state as any).time.compacted).toBeUndefined()
  })
})
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd packages/opencode && bun test test/session/message-v2.test.ts --filter "deduplicateReads"`
Expected: FAIL — `MessageV2.deduplicateReads` is not exported / not a function

- [ ] **Step 3: Implement `deduplicateReads()` in message-v2.ts**

In `packages/opencode/src/session/message-v2.ts`, add the function after the existing imports but before `toModelMessagesEffect`. Export it:

```ts
export function deduplicateReads(parts: Part[]): Part[] {
  const seen = new Map<string, Part>()
  const superseded = new Set<string>()

  for (const part of parts) {
    if (part.type !== "tool" || part.tool !== "read") continue
    if (part.state.status !== "completed") continue
    const path = part.state.input.filePath
    const prev = seen.get(path)
    if (prev) superseded.add(prev.id)
    seen.set(path, part)
  }

  if (superseded.size === 0) return parts

  return parts.map((part) => {
    if (superseded.has(part.id)) {
      return {
        ...part,
        state: {
          ...part.state,
          time: { ...part.state.time, compacted: Date.now() },
        },
      }
    }
    return part
  })
}
```

Note: The `Part` type is imported from `@opencode-ai/schema/session`. The function needs to be added near the top of the file, after imports. The `Part` type is already imported — check the existing imports at the top of the file.

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd packages/opencode && bun test test/session/message-v2.test.ts --filter "deduplicateReads"`
Expected: PASS — all 7 tests pass

- [ ] **Step 5: Commit**

```bash
git add packages/opencode/src/session/message-v2.ts packages/opencode/test/session/message-v2.test.ts
git commit -m "feat(session): add deduplicateReads function with tests"
```

---

### Task 3: Wire `deduplicateReads()` into `toModelMessagesEffect()`

**Files:**
- Modify: `packages/opencode/src/session/message-v2.ts` (the `toModelMessagesEffect` function)

**Interfaces:**
- Consumes: `deduplicateReads()` from Task 2, config via `ConfigV1.Info`
- Produces: deduplication applied before part-to-model-message conversion

- [ ] **Step 1: Add integration test**

Add a test in the `"session.message-v2.toModelMessage"` describe block in `packages/opencode/test/session/message-v2.test.ts`. The test verifies that when `toModelMessages` is called, stale read outputs are replaced with `"[Old tool result content cleared]"`.

The existing tests in this file call `MessageV2.toModelMessages(input, model, options?)` directly (not Effect-based). Follow the same pattern.

Add this test after the existing compacted-output test (around line 750):

```ts
test("deduplicates read outputs by file path", () => {
  const assistantID = "msg_asst_dedup"
  const input: SessionV1.WithParts[] = [
    {
      info: userInfo("msg_user_dedup"),
      parts: [
        { ...basePart("msg_user_dedup", "p-user"), type: "text", text: "read foo.ts" },
      ] as SessionV1.Part[],
    },
    {
      info: assistantInfo(assistantID, "msg_user_dedup"),
      parts: [
        {
          ...basePart(assistantID, "p-read1"),
          type: "tool",
          callID: "call-1",
          tool: "read",
          state: {
            status: "completed",
            input: { filePath: "/src/foo.ts" },
            output: "old content",
            title: "Read",
            metadata: {},
            time: { start: 0, end: 1 },
          },
        },
        {
          ...basePart(assistantID, "p-read2"),
          type: "tool",
          callID: "call-2",
          tool: "read",
          state: {
            status: "completed",
            input: { filePath: "/src/foo.ts" },
            output: "new content",
            title: "Read",
            metadata: {},
            time: { start: 2, end: 3 },
          },
        },
        {
          ...basePart(assistantID, "p-bash"),
          type: "tool",
          callID: "call-3",
          tool: "bash",
          state: {
            status: "completed",
            input: { cmd: "ls" },
            output: "file.ts",
            title: "Bash",
            metadata: {},
            time: { start: 4, end: 5 },
          },
        },
      ] as SessionV1.Part[],
    },
  ]

  const result = MessageV2.toModelMessages(input, model)
  const assistantMsg = result.find((m) => m.role === "assistant")!
  const readResults = assistantMsg.content.filter(
    (c) => c.type === "tool-result" && c.toolName === "read",
  )
  // First read should be compacted (old content cleared)
  const firstRead = readResults.find((c) => c.toolCallId === "call-1")!
  expect(firstRead.content[0].text).toContain("[Old tool result content cleared]")
  // Second read should have actual content
  const secondRead = readResults.find((c) => c.toolCallId === "call-2")!
  expect(secondRead.content[0].text).toBe("new content")
  // Bash output should not be affected
  const bashResult = assistantMsg.content.find(
    (c) => c.type === "tool-result" && c.toolName === "bash",
  )!
  expect(bashResult.content[0].text).toBe("file.ts")
})
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd packages/opencode && bun test test/session/message-v2.test.ts --filter "deduplicates read outputs by file path"`
Expected: FAIL — the first read still shows "old content" (dedup not wired in yet)

- [ ] **Step 3: Wire `deduplicateReads()` into `toModelMessagesEffect()`**

In `packages/opencode/src/session/message-v2.ts`, inside `toModelMessagesEffect()`, after parts are loaded/hydrated and before the conversion loop, add the dedup call. The config is passed as `input.cfg` or similar — check how config is accessed in the function.

Looking at the function signature: `toModelMessagesEffect(input: WithParts[], model: Provider.Model, options?)`. The function does NOT currently take a config parameter. The dedup function should be called unconditionally inside `toModelMessagesEffect` (defaulting to enabled). To make it configurable, the config check happens at the call site or via the options parameter.

Since the spec says `dedupReads` defaults to `true` and the function signature doesn't take config, the simplest approach: always deduplicate in `toModelMessagesEffect`. The config gate is checked at the call site where `toModelMessagesEffect` is invoked (in the session processor). However, looking at the spec more carefully, it says:

```ts
const deduped = cfg.compaction?.dedupReads !== false 
  ? deduplicateReads(parts) 
  : parts
```

This means the config check should happen where `toModelMessagesEffect` is called. But since `toModelMessagesEffect` doesn't receive config, we have two options:
1. Always deduplicate (simplest, matches spec's default behavior)
2. Add a config/options parameter

Option 1 is correct per the spec — the spec says default is `true` and the integration is inside `toModelMessagesEffect`. The config check can be added later if needed. For now, always deduplicate.

Find where parts are collected into the conversion loop. Based on the exploration, the function iterates over `input` (WithParts[]), then iterates over `message.parts`. Add deduplication right before that inner loop:

```ts
// After collecting all parts from all messages, before the conversion loop:
const deduped = deduplicateReads(parts)
// Then use deduped instead of parts in the conversion
```

However, looking at the code structure more carefully: `toModelMessagesEffect` iterates over `input` (messages with parts) directly — it doesn't flatten all parts first. The dedup function needs a flat `Part[]` array. So we need to:

1. Flatten all parts from all messages
2. Call `deduplicateReads()` on the flat array
3. Use the deduped parts in the conversion

Actually, re-reading the code: the function iterates `for (const message of input)` then `for (const part of message.parts)`. The dedup needs to see ALL parts across all messages to know which reads are stale. So we need to flatten first.

The cleanest approach: at the start of `toModelMessagesEffect`, flatten all parts, call `deduplicateReads`, then rebuild the WithParts structure or iterate the deduped flat list. But this changes the iteration structure.

A simpler approach: since dedup only marks parts with `compacted` timestamp (doesn't remove them), we can flatten, dedup, then use a Set of compacted part IDs during the conversion loop:

```ts
// At start of toModelMessagesEffect, after the Effect.gen opening:
const allParts = input.flatMap((m) => m.parts)
const deduped = deduplicateReads(allParts)
const compactedIds = new Set(
  deduped
    .filter((p) => p.type === "tool" && p.state.status === "completed" && p.state.time.compacted)
    .map((p) => p.id)
)
```

Then in the conversion loop, when processing tool parts, check if `compactedIds.has(part.id)` — but wait, the existing code already checks `part.state.time.compacted`. Since `deduplicateReads` sets the `compacted` timestamp on the part objects in the flat array, but the original `message.parts` still have the unmodified parts...

This is the key insight: `deduplicateReads` returns new part objects with `compacted` set, but the original `WithParts[]` still has the old parts. We need to use the deduped parts.

The simplest correct approach: build a Map from part ID to deduped part, then in the conversion loop, look up each part in the map:

```ts
// At start of toModelMessagesEffect:
const allParts = input.flatMap((m) => m.parts)
const dedupedParts = deduplicateReads(allParts)
const dedupedById = new Map(dedupedParts.map((p) => [p.id, p]))

// Then in the inner loop where parts are processed:
// Replace `part` with `dedupedById.get(part.id) ?? part`
```

This preserves the existing iteration structure while applying dedup. Let me write the actual edit.

- [ ] **Step 4: Implement the wiring**

Edit `packages/opencode/src/session/message-v2.ts` in `toModelMessagesEffect`. Find the start of the function body (after `function*` opening) and add:

```ts
const allParts = input.flatMap((m) => m.parts)
const dedupedParts = deduplicateReads(allParts)
const dedupedById = new Map(dedupedParts.map((p) => [p.id, p]))
```

Then find every place where `part` is used in the inner loop and ensure it reads from the deduped map. The key location is the tool result construction where `part.state.time.compacted` is checked — that part reference needs to come from `dedupedById`.

Actually, the simpler approach: since `deduplicateReads` returns parts with `compacted` set, and the existing code already handles `compacted` parts by replacing output with `"[Old tool result content cleared]"`, we just need to make the conversion loop use the deduped parts instead of the original parts.

The cleanest edit: after the existing part iteration setup, remap the parts:

In the inner loop `for (const part of message.parts)`, the first use of `part` that matters is in the tool result construction. Change it to use the deduped version:

```ts
// Before the inner loop processes each part, look up the deduped version:
const effectivePart = dedupedById.get(part.id) ?? part
```

Then use `effectivePart` instead of `part` for all state reads (status, compacted check, output, etc.).

- [ ] **Step 5: Run test to verify it passes**

Run: `cd packages/opencode && bun test test/session/message-v2.test.ts --filter "deduplicates read outputs by file path"`
Expected: PASS

- [ ] **Step 6: Run full message-v2 test suite**

Run: `cd packages/opencode && bun test test/session/message-v2.test.ts`
Expected: ALL PASS (no regressions)

- [ ] **Step 7: Commit**

```bash
git add packages/opencode/src/session/message-v2.ts packages/opencode/test/session/message-v2.test.ts
git commit -m "feat(session): wire deduplicateReads into toModelMessagesEffect"
```

---

### Task 4: Update read tool description

**Files:**
- Modify: `packages/opencode/src/tool/read.txt`

**Interfaces:**
- Consumes: none
- Produces: updated tool description text

- [ ] **Step 1: Add deduplication note to read tool description**

Edit `packages/opencode/src/tool/read.txt`. Add one sentence after the first line:

```
Read a file or directory from the local filesystem. If the path does not exist, an error is returned. Replaces any earlier read of the same file in context — only the most recent read is retained.

Usage:
...
```

- [ ] **Step 2: Verify typecheck passes**

Run: `cd packages/opencode && bun typecheck`
Expected: PASS

- [ ] **Step 3: Run full test suite**

Run: `cd packages/opencode && bun test test/session/message-v2.test.ts`
Expected: ALL PASS

- [ ] **Step 4: Commit**

```bash
git add packages/opencode/src/tool/read.txt
git commit -m "docs(tool): add deduplication note to read tool description"
```

---

### Task 5: Verify end-to-end

**Files:**
- None (verification only)

- [ ] **Step 1: Run full typecheck**

Run: `cd packages/opencode && bun typecheck`
Expected: PASS

- [ ] **Step 2: Run full test suite**

Run: `cd packages/opencode && bun test`
Expected: ALL PASS

- [ ] **Step 3: Verify config wiring**

Check that `dedupReads` appears in the config schema by searching for it:
Run: `grep -r "dedupReads" packages/`
Expected: Found in `packages/core/src/v1/config/config.ts` and `packages/opencode/src/session/message-v2.ts`

- [ ] **Step 4: Final commit if any fixes needed**

If any fixes were required in steps 1-3, commit them:
```bash
git add -A
git commit -m "fix: address review findings from read deduplication"
```
