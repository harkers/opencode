# Capacity Fallback Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add portable per-model fallback chains for retryable pre-output capacity failures.

**Architecture:** Reuse the existing model `options` map for ordered `provider/model` fallback references. Extend the session stream input with resolved fallback models and make the processor advance the chain only after the current model's existing retry policy is exhausted and before any output/tool activity has occurred.

**Tech Stack:** TypeScript, Bun, Effect, OpenCode provider catalog, Node/Bun tests.

**Spec:** `docs/superpowers/specs/2026-10-09-capacity-fallback-design.md`

## Global Constraints

- Do not import machine-specific routing files into upstream OpenCode.
- Preserve current same-model retry behavior and retry status events.
- Never replay a turn after assistant output, reasoning, or tool-call activity has been emitted.
- Treat missing or malformed fallback model references as unavailable candidates, not as new fatal errors.
- Run focused tests and `bun run --cwd packages/opencode typecheck` before completion.

---

### Task 1: Add Fallback Model Resolution

**Files:**

- Modify: `packages/opencode/src/provider/provider.ts`
- Test: `packages/opencode/test/provider/provider.test.ts`

**Interfaces:**

- Produces `Provider.fallbackModels(model: Provider.Model): Effect.Effect<Provider.Model[]>`.
- Reads `model.options.fallback_models` as an ordered string array of `provider/model` references.

- [x] **Step 1: Write the failing test**

Add a provider test with a configured primary model whose `options.fallback_models` contains one valid and one missing reference. Assert the resolver returns only the valid model in declared order and returns an empty list for a non-array value.

- [x] **Step 2: Run the focused test to verify it fails**

Run: `bun test packages/opencode/test/provider/provider.test.ts`

Expected: FAIL because `fallbackModels` does not exist.

- [x] **Step 3: Implement the minimal resolver**

Split each string at the first `/`, call the existing `getModel(providerID, modelID)` for each candidate, ignore lookup failures, and return the successful models. Do not mutate the primary model or its options.

- [x] **Step 4: Run the focused test to verify it passes**

Run: `bun test packages/opencode/test/provider/provider.test.ts`

Expected: PASS.

### Task 2: Advance the Session Model Chain

**Files:**

- Modify: `packages/opencode/src/session/llm.ts`
- Modify: `packages/opencode/src/session/processor.ts`
- Modify: `packages/opencode/src/session/prompt.ts`
- Test: `packages/opencode/test/session/processor-effect.test.ts`

**Interfaces:**

- Extends `LLM.StreamInput` with `fallbackModels?: Provider.Model[]`.
- `SessionProcessor` consumes the chain and returns the existing `Result` values.

- [x] **Step 1: Write the failing tests**

Add processor-effect tests for these behaviors:

```ts
test("uses the first fallback after retryable primary failure", async () => {
  // Arrange a stream that fails on the primary before emitting any event,
  // then succeeds on the first fallback; assert the observed model IDs.
})

test("does not fallback after output has started", async () => {
  // Arrange a primary stream that emits text then fails; assert no fallback call.
})
```

Use the existing Effect test harness and stream fixtures in `processor-effect.test.ts`; inject a deterministic `LLM.Service` layer that records model IDs and fails with a retryable API error before succeeding. Do not mock the provider catalog where the test can use the existing test provider.

- [x] **Step 2: Run the focused tests to verify they fail**

Run: `bun test packages/opencode/test/session/processor-effect.test.ts`

Expected: FAIL because the processor currently retries only the original stream model.

- [x] **Step 3: Implement the minimal chain transition**

Pass resolved fallback models from the prompt's selected model into `handle.process`. In the processor, retain the current candidate index, run the existing retry policy for that candidate, and on exhausted retryable failure advance to the next candidate only when no text, reasoning, or tool-call state was recorded. Rebuild the stream input with the new model and reset per-attempt transient state while retaining the original message and request context.

- [x] **Step 4: Run focused tests to verify the behavior**

Run: `bun test packages/opencode/test/session/processor-effect.test.ts`

Expected: PASS with the fallback attempt visible and no fallback after output.

### Task 3: Configuration and Regression Verification

**Files:**

- Modify: `packages/opencode/test/lib/test-provider.ts` if a reusable configured fallback fixture is required.
- Modify: `packages/opencode/test/provider/provider.test.ts` and `packages/opencode/test/session/processor-effect.test.ts` only for fixture cleanup.

- [x] **Step 1: Add a non-retryable regression case**

Assert authentication, context-overflow, and structured-output failures do not advance to a fallback model.

- [x] **Step 2: Run the complete focused suite**

Run: `bun test packages/opencode/test/provider/provider.test.ts packages/opencode/test/session/processor-effect.test.ts`

Expected: PASS with zero failures.

- [x] **Step 3: Run typecheck**

Run: `bun run --cwd packages/opencode typecheck`

Expected: exit 0 with no TypeScript errors.

- [x] **Step 4: Review the diff**

Run: `git diff --check && git diff --stat && git status --short`

Confirm only the planned runtime, test, and design/plan files are changed.
