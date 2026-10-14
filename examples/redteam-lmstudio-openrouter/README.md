# redteam-lmstudio-openrouter (Red Team a Local LM Studio Model)

Red teams a model served by LM Studio, using OpenRouter models as attacker, generator and judge.

## Setup

```bash
npx promptfoo@latest init --example redteam-lmstudio-openrouter
export OPENROUTER_API_KEY=...
lms server start --port 1234   # LM Studio's API server is off by default
```

## Run

```bash
PROMPTFOO_DISABLE_REMOTE_GENERATION=true npx promptfoo@latest redteam run --max-concurrency 1
```

## Notes

- LM Studio has no dedicated provider; use `openai:chat:<model>` with `apiBaseUrl`.
- `harmful:*` plugins require remote generation, so they fail with `PROMPTFOO_DISABLE_REMOTE_GENERATION=true`. This example uses `politics` and `hallucination`.
- Reasoning models return `reasoning_content`. Set `showThinking: false` and a large `max_tokens`, otherwise the answer can come back empty or prefixed with `Thinking:`.
- Free OpenRouter models can return 503s. Judge failures are recorded as failed results, not ERRORs, so `--retry-errors` will not pick them up; swap in a paid judge or re-run.
- LM Studio serves one request at a time, so keep concurrency low.
