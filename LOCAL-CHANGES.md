# Local changes

Fork of [`danipl/opencode-jev`](https://github.com/danipl/opencode-jev) tuned to run
against a **self-hosted OpenJev** decision server instead of the hosted TypeSafe API.
Upstream behaviour is unchanged when the new defaults are not exercised.

## Why

Hosted `jev-latest` gets a compact but complete picture of the request. This fork was
tuned against `OpenJev-Flash-9B` (GGUF, served locally through Ollama + the upstream
`helper/shim.py`), and three things had to change before routing was useful:

1. **State carried no tool activity.** Upstream kept only user/assistant *text* —
   tool calls and tool results were filtered out. In a build session the state was
   ~180 characters: the original instruction and nothing else. Jev could not tell
   whether `webfetch` had already run.
2. **Long state diluted the decision.** With a full ledger of 9 entries (6.8 KB),
   the same question scored **0.535** confidence; with 3 entries, **0.886**. The
   model is a decision model, not a summarizer — recent and short beats complete.
3. **Bare tool names carried no semantics.** `criteria: { edit: "Use tool edit" }`
   is much weaker than the tool's own description.

## Changes

### `src/index.ts`

- **Tool ledger in state.** `stateOf` now emits one line per message:
  `→ tool(args)` for calls, `← tool: result` for results (300 chars),
  `ERROR` for failures, plus assistant text. Handles the AI-SDK part shape
  (`tool-call` / `tool-result` / `tool-error`) and legacy `tool_calls`.
- **Tool descriptions as criteria.** `toolDescriptions()` reads each tool's
  description (collapsed, 160 chars) and `askJev` uses it as the choice label.
- **Budgets retuned** for short-context decision models:
  `STATE_CHAR_BUDGET 8000 → 6000`, `TURN_CHAR_BUDGET 2000 → 1200`,
  `RECENT_TURNS 4 → 3`, new `TOOL_RESULT_CHARS 300`, `TOOL_DESC_CHARS 160`.
- The first user message stays pinned ahead of the recent window, as upstream did.

### `local/`

The serving stack this was tuned against:

- `front.py` — loopback front on `:3000`. Serves `GET /v1/models` (upstream shim
  has none, and OpenClaw's decision adapter needs it) and normalizes responses:
  the shim reports `model` as a long debug banner, which violates the adapter's
  `^[a-zA-Z0-9._/-]+$` / 128-char schema, and extra top-level keys are rejected by
  `additionalProperties: false`. Forwards `POST /v1/systemone` to the shim.
- `serve.sh` — starts the shim on `:3010`, waits for the port, hands off to the front.
- `openjev-shim.service` — user unit, starts at login.

## Measured effect (RTX 4070 Ti, OpenJev-Flash-9B Q4_K_M)

| | before | after |
|---|---|---|
| trim rate on real tool choices | ~10% | ~50% |
| typical confidence (clear step) | 0.12–0.36 | 0.69–0.99 |
| warm decision latency | 43–53 ms | 47–50 ms |
| state sent | 184 chars | 1.5–4 KB |

All 26 upstream tests pass with these changes (`npm test`).

## Notes

- `minConfidence` defaults to `0.75`. Against this model `0.6` was used; the
  measured band between ambiguous (≤0.36) and clear (≥0.69) is wide.
- Tool descriptions and the ledger increase prompt tokens per decision
  (~150 → ~250 in local tests), which is the cost of the accuracy.
- If you point this at hosted TypeSafe instead, the upstream defaults remain
  reasonable; the tuning here targets a smaller local model with a noisier
  confidence distribution.

## Model-tier routing

Optional, off unless configured. Alongside `next_tool`, the plugin asks a second
question in the **same** round-trip — `model_tier` — whose criteria are the tiers
you define. When the chosen tier clears `minConfidence`, the outgoing request's
`model` field is rewritten before it leaves the plugin. One decision, two
actions: tool trim and model selection.

```yaml
# ~/.config/jev/config.yaml
tiers:
  flash:
    model: deepseek-v4.1-flash
    description: Single-step work — reads, greps, small edits, quick answers
  pro:
    model: deepseek-v4-pro
    description: Multi-file refactors, architecture, hard debugging, ambiguous specs
```

`model` must be the exact id the provider puts in the request body (not
`provider/id`), because the plugin rewrites that field verbatim.

Tier switching has its own gate: `tierMinConfidence` (default: same as
`minConfidence`). Raising it keeps cheap steps cheap: a 0.65 hunch no longer
promotes the request to the expensive model while tool trims still act at 0.6. Tiers are
independent of the tool decision: a strong tier can be selected on a step whose
tool choice is still below threshold.

Log lines distinguish the two actions:

```
apply: model deepseek-v4.1-flash -> deepseek-v4-pro (pro@0.81 >= minConfidence 0.6)
apply: tools trimmed to edit only (edit@0.95 >= minConfidence 0.6)
```
