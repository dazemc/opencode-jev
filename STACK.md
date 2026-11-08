# Local Jev stack

Everything needed to run one local decision model behind two different agents.
Nothing in this stack sends conversation data off the machine.

```
                    ┌──────────────────────────────┐
  opencode ────────▶│ forked plugin (this repo)    │──┐
  (per tool step)   │  http.request: trim + tier   │  │
                    └──────────────────────────────┘  │
                                                      ▼
                    ┌──────────────────────────────┐  ┌────────────────────┐
  OpenClaw ────────▶│ jev-router plugin            │─▶│ 127.0.0.1:3000     │
  (per user turn)   │  before_model_resolve: tier  │  │ front.py           │
                    └──────────────────────────────┘  │  /v1/models        │
                                                      │  /v1/systemone     │
                                                      └─────────┬──────────┘
                                                                ▼
                                                      ┌────────────────────┐
                                                      │ 127.0.0.1:3010     │
                                                      │ shim.py (upstream) │
                                                      └─────────┬──────────┘
                                                                ▼
                                                      ┌────────────────────┐
                                                      │ Ollama: qwen       │
                                                      │ OpenJev-Flash-9B   │
                                                      └────────────────────┘
```

## Why the front exists

Upstream `shim.py` serves `POST /v1/systemone` and nothing else. Two consumers
need more:

- **Model discovery** — OpenClaw's decision adapter will not advertise a
  decision model until it can enumerate ids, so the front answers
  `GET /v1/models` (`kev-latest`, `jev-latest`, `openjev-flash-9b`).
- **Strict response shaping** — the adapter validates with
  `additionalProperties: false` and requires `model` to match
  `^[a-zA-Z0-9._/-]+$` / ≤128 chars. The shim reports `model` as a long debug
  banner and adds a top-level `id`. One bad field rejects the whole answer as
  `invalid-response`. The front rebuilds `{model, answers, usage}` and coerces
  usage to integers.

The shim itself stays byte-identical to upstream (`65ba37…` in the model card).

## Install

```sh
# 1. model + server
ollama pull hf.co/openjev/OpenJev-Flash-9B-GGUF:Q4_K_M
ollama cp hf.co/openjev/OpenJev-Flash-9B-GGUF:Q4_K_M qwen
# tokenizer files: download tokenizer.json / tokenizer_config.json /
# chat_template.jinja / generation_config.json / config.json from
# openjev/OpenJev-Flash-9B into ~/.local/share/openjev/tokenizer

# 2. python deps for the shim
uv venv ~/.local/share/openjev/venv && uv pip install --python ~/.local/share/openjev/venv/bin/python openai httpx transformers

# 3. run it
cp local/front.py local/serve.sh ~/.local/share/openjev/
cp local/openjev-shim.service ~/.config/systemd/user/
systemctl --user enable --now openjev-shim.service

# 4. opencode plugin (fork of danipl/opencode-jev)
git clone https://github.com/dazemc/opencode-jev ~/.local/share/opencode-jev
cd ~/.local/share/opencode-jev && bun install --production
# opencode.json: "plugins": ["/home/<user>/.local/share/opencode-jev"]

# 5. OpenClaw model-tier router
cp -r openclaw/jev-router ~/.openclaw/extensions/
openclaw plugins reload jev-router
```

## Configuration

`~/.config/jev/config.yaml` — read by the opencode plugin:

```yaml
apiKey: local-openjev
apiUrl: http://127.0.0.1:3000/v1/systemone
model: openjev-flash-9b
minConfidence: 0.6        # tool trimming
tierMinConfidence: 0.75   # model switching
timeoutMs: 2500
tiers:
  flash:
    model: deepseek-v4.1-flash
    description: Single-step work — reads, greps, small edits, quick answers
  pro:
    model: deepseek-v4-pro
    description: Multi-file refactors, architecture, hard debugging, ambiguous specs
```

`openclaw.json` — `plugins.entries.jev-router.config` mirrors `apiUrl`,
`tierMinConfidence` and `tiers`; it uses **0.65** because
`before_model_resolve` receives only the prompt, not a ledger.

Tier models must be **bare ids** (`deepseek-v4-pro`), not `provider/id` — both
consumers rewrite the request's `model` field verbatim.

## Measured tuning (RTX 4070 Ti)

| change | effect |
|---|---|
| tool ledger in state | confidence 0.33 → 0.58 on identical questions |
| 3 turns instead of 8 | 0.535 → 0.886 on the identical question; trim rate ~10% → ~50% |
| tool descriptions in criteria | disambiguates `edit`/`write`/`patch`, `webfetch`/`websearch` |
| model tiers | `pro@0.78` on a monorepo refactor, `flash@0.98` on a file read |
| warm latency | 47–50 ms per decision |

## Failure behaviour

Both consumers fail open: any error, timeout, or low confidence leaves the
request exactly as the agent built it. The gate is deliberately asymmetric —
a wrong tool trim wastes a turn; a wrong tier promotion bills the expensive
model.
