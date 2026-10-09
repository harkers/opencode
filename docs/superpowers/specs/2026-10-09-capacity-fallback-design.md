# Capacity Fallback Design

## Goal

When a configured model fails with a retryable capacity or transport error before producing usable output, OpenCode should try the model's explicitly configured fallback chain instead of exhausting the request on the original model.

## Configuration Contract

Provider model entries may declare fallback models in their existing arbitrary `options` map:

```jsonc
{
  "provider": {
    "local": {
      "models": {
        "swift-qwen": {
          "options": {
            "fallback_models": ["local/qwen3-coder-30b-a3b-30g"],
          },
        },
      },
    },
  },
}
```

Each value is a `provider/model` reference. Fallbacks are ordered, resolved through the existing provider catalog, and invalid or missing references fail closed by ignoring that candidate and preserving the original error. The chain is explicit and portable; it does not read an external machine-specific routing file.

## Runtime Behavior

1. Existing same-model retry behavior remains unchanged for retryable failures.
2. Once the current model's retry policy gives up, the processor tries the next configured fallback model.
3. Fallback is allowed only when no assistant text, reasoning, or tool call has been emitted for the failed attempt. This prevents replaying a partially executed turn.
4. A fallback attempt receives the original request, messages, tools, and agent context, but uses the fallback model.
5. Non-retryable errors, context overflow, content filters, and structured-output failures do not select a fallback.
6. When all candidates fail, the original terminal error is surfaced.

## Verification

Unit tests cover fallback option parsing/resolution, ordered model selection after a retryable failure, no fallback after emitted output, and no fallback for non-retryable errors. Existing retry tests and the package typecheck must pass.
