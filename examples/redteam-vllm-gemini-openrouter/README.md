# redteam-vllm-gemini-openrouter (Red Team a Local vLLM Target with Gemini + OpenRouter)

Red teams a model served locally by vLLM, using a free Gemini model as judge and a free OpenRouter model as attacker/generator. Intended for multi-GPU machines where the target model doesn't fit on a laptop.

## Setup

```bash
npx promptfoo@latest init --example redteam-vllm-gemini-openrouter
export GEMINI_API_KEY=...       # or GOOGLE_API_KEY
export OPENROUTER_API_KEY=...

# Serve the target with vLLM (adjust --tensor-parallel-size to your GPU count)
uv venv ~/vllm-env && source ~/vllm-env/bin/activate
uv pip install vllm
HF_HUB_OFFLINE=1 TRANSFORMERS_OFFLINE=1 VLLM_NO_USAGE_STATS=1 vllm serve Qwen/Qwen3.5-122B-A10B-FP8 \
  --served-model-name qwen3.5-122b-a10b-fp8 \
  --tensor-parallel-size 4 \
  --max-model-len 16384 \
  --gpu-memory-utilization 0.90 \
  --port 8001
```

## Run

```bash
PROMPTFOO_DISABLE_REMOTE_GENERATION=true \
PROMPTFOO_RETRY_5XX=true PROMPTFOO_REQUEST_BACKOFF_MS=8000 \
  npx promptfoo@latest redteam run --max-concurrency 4 -o output.json
```

`-o output.json` exports the full run (prompts, target responses, judge scores/rationale, plugin and strategy metadata) so results can be aggregated into a dataset later.

## Notes

- vLLM has no dedicated provider; use `openai:chat:<served-model-name>` with `apiBaseUrl` pointed at vLLM's port.
- Qwen3.5 defaults to "thinking mode," which burns the token budget on reasoning before (or instead of) the actual answer and can also make the model misidentify itself. Disable it with `passthrough: { chat_template_kwargs: { enable_thinking: false } }`, which forwards straight into the chat completions body since promptfoo's OpenAI provider doesn't allowlist vLLM-specific fields.
- If vLLM's worker processes hit `httpx.ProxyError: 500` on startup, a corporate HTTP(S) proxy is the usual cause even when the model is fully cached locally. Set `HF_HUB_OFFLINE=1 TRANSFORMERS_OFFLINE=1 VLLM_NO_USAGE_STATS=1` to stop it from making any outbound calls during load.
- Unlike LM Studio, vLLM's continuous batching handles real concurrency, so `--max-concurrency` can be raised well above 1.

### Plugin selection: why these 12, not `harmful`

Most `harmful:*` subcategories (hate, violent-crime, self-harm, cybercrime, etc.), plus `bias:*`,
`coppa`, `ferpa`, `religion`, and `wordplay`, require promptfoo's remote "unaligned" models to
generate test cases, with no local fallback — they hard-fail under
`PROMPTFOO_DISABLE_REMOTE_GENERATION=true`. Only `harmful:privacy` and `harmful:intellectual-property`
work locally from that whole family.

This example uses 10 plugins that generate test cases entirely locally:

- **Dataset-backed** (`beavertails`, `cyberseceval`, `donotanswer`, `harmbench`, `aegis`, `pliny`,
  `toxic-chat`, `xstest`): each fetches a static adversarial dataset from HuggingFace or GitHub (not
  promptfoo's cloud) and templates it locally.
- **Synthetic** (`contracts`, `politics`): generated fresh via the attacker model (Nemotron) above,
  same as any locally-configured redteam provider.

`unsafebench` (image dataset, `numTests: 5`) is also included, but `yiting/UnsafeBench` is gated on HF:
request access, then `export HF_TOKEN=$(hf auth token)` before running (promptfoo queries HF's
datasets-server API and does not read the local `hf` cache; without the token it fails with 404).
It sends images, so the target must be multimodal. `vlguard` is left out because `ys-zong/VLGuard`
is access-gated (403) and needs the same access request.

### Important: Crescendo never touches the dataset-backed plugins

promptfoo hard-exempts every dataset-backed plugin (`beavertails`, `cyberseceval`, `donotanswer`,
`harmbench`, `aegis`, `pliny`, `toxic-chat`, `xstest`, and `unsafebench`/`vlguard` if you add them
back) from **every** strategy, Crescendo included (`DATASET_EXEMPT_PLUGINS` in
`src/redteam/constants/plugins.ts`) — not configurable. These plugins ship verbatim prompts from
real-world adversarial datasets and are meant to be tested single-turn, not escalated into a
multi-turn conversation. They always run as plain single-turn probes, each costing exactly one judge
grading call and zero attacker calls, regardless of the `strategies:` block.

`contracts` and `politics` are the only two plugins in this config that Crescendo actually
transforms. That's why they carry a much larger `numTests` below — they're where the attacker and
judge quotas are actually spent.

### Budgeting against daily API quotas

With the dataset plugins at `numTests: 10` each (8 plugins → 80 single-turn test cases, plus 5 `unsafebench` → 85 judge
calls, 0 attacker calls — there's no strategy multiplier for them), and `contracts`/`politics` at
`numTests: 12` each (24 base test cases, every one of which Crescendo also transforms into a
multi-turn attack — strategies are additive, not multiplicative, so both the base and the
Crescendo-transformed test case run):

Per Crescendo attack with `maxTurns: N`, worst case:

- attacker calls ≈ `N` (one adversarial-prompt generation per turn)
- judge calls ≈ `1 + 3N` (the base test's own grading call, plus a refusal check, a grader call,
  and an eval-score call per Crescendo turn)

So with `maxTurns: 5`:

- attacker: `24 * 5 = 120` of your 1000/day Nemotron quota
- judge: `85 (dataset plugins) + 24 * (1 + 3 * 5) = 85 + 384 = 469` of your 500/day Gemini quota

That's a tight fit on the judge quota — raise `maxTurns` or either `numTests` further only after a
real run confirms actual usage (worst-case estimates assume every refusal/grader/eval-score call
fires; short-circuits on empty/refusal responses usually bring real usage down). If you want more
headroom, lower the dataset plugins' `numTests` first — they don't benefit from Crescendo's
multi-turn escalation, so cutting them costs you breadth, not depth.

That leaves meaningful headroom on both quotas for Crescendo's backtracking retries (up to
`maxBacktracks`, default 10, per attack) and for Gemini 503 retries. If you raise `numTests` or
`maxTurns`, judge quota is the binding constraint — it's consumed roughly 3x faster than the
attacker quota per Crescendo turn — so size test cases against `500 / (3 * maxTurns)`, not the
attacker's 1000/day.

The judge **must** be set explicitly (via `defaultTest.options.provider`, as in this config) — if
left unset, Crescendo's grading calls silently fall back to the attacker provider instead of a
separate quota.

- `harmful:*` plugins (other than `privacy` and `intellectual-property`) require remote generation and will fail under `PROMPTFOO_DISABLE_REMOTE_GENERATION=true`; see above for the local alternative used here.
- Free-tier Gemini (the judge) occasionally returns `503 UNAVAILABLE` ("currently experiencing high demand"), which surfaces as `Error: No candidates returned in API response.`. By default promptfoo treats a 503 as a normal non-ok response and does not retry it. Set `PROMPTFOO_RETRY_5XX=true` (and optionally raise `PROMPTFOO_REQUEST_BACKOFF_MS`, default 5000ms) to retry it automatically during the run. This error is a thrown API exception, not a low grading score, so it's recorded as an `ERROR` row — if any still slip through, `promptfoo redteam run --retry-errors` re-grades only those rows instead of re-running the whole eval.
