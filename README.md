# lite-llm-proxy

LiteLLM proxy that maps Claude Code tiers (`fable` / `opus` / `sonnet` / `haiku`) to AWS Bedrock models. All auth is via AWS IAM keys — no OpenAI key needed.

## Models (config.yml)

| Proxy`model_name` | Bedrock`model`                            | Purpose                                                                                                                                | Bedrock region          |
| ------------------- | ------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------- | ----------------------- |
| `fable`           | `bedrock/qwen.qwen3-coder-480b-a35b-v1:0` | Largest context, heavy/long tasks                                                                                                      | `AWS_REGION_OPENCODE` |
| `opus`            | `bedrock/minimax.minimax-m2.5`            | Complex reasoning                                                                                                                      | `AWS_REGION_OPENCODE` |
| `sonnet`          | `bedrock/deepseek.v3.2`                   | Daily coding driver                                                                                                                    | `AWS_REGION_OPENCODE` |
| `haiku`           | `bedrock/global.openai.gpt-5.6-luna`      | Fastest/cheapest OpenAI tier via global CRIS (272K ctx on Bedrock). Routes from any source region (e.g.`ap-south-1`) to US capacity. | `AWS_REGION_OPENCODE` |

`litellm_settings`: `drop_params: true`, `modify_params: true`, `request_timeout: 600`, `num_retries: 2`. `modify_params` lets LiteLLM insert a dummy assistant continuation when Bedrock tool histories contain consecutive user/tool blocks. `temperature: 0.3` and `additional_drop_params: ["stop"]` on every model.

## Prerequisites

- Python `>=3.12` (see `.python-version`: `3.12`)
- `uv` `>=0.9` (or `pip`)
- AWS account with Bedrock model access enabled for all 4 models + cross-region inference profile `global.openai.gpt-5.6-luna`
- IAM principal with `bedrock:InvokeModel` on:
  - `arn:aws:bedrock:<region>:<account>:inference-profile/global.openai.gpt-5.6-luna`
  - `arn:aws:bedrock:<region>:<account>:project/default`
  - `arn:aws:bedrock:::foundation-model/openai.gpt-5.6-luna`
    (plus the 3 other model ARNs; if `bedrock:InvokeModelWithResponseStream` is used add that too)

## 1. Install

```bash
git clone <this-repo> && cd lite-llm-proxy

# with uv (recommended)
uv sync

# or pip
python -m venv .venv && source .venv/bin/activate
pip install "litellm[proxy]>=1.98.0"
```

## 2. Set environment variables

The proxy reads `os.environ/...` from `config.yml`. Do **not** commit these.

```bash
# Required — same 3 vars for all models
export AWS_REGION_OPENCODE="ap-south-1"
export AWS_ACCESS_KEY_ID_OPENCODE="AKIA..."
export AWS_SECRET_ACCESS_KEY_OPENCODE="..."

# Optional proxy auth (recommended for production)
export LITELLM_MASTER_KEY="sk-1234"
```

For persistence, create a local `.env` (already gitignored) and export it:

```bash
cat > .env <<'EOF'
AWS_REGION_OPENCODE=ap-south-1
AWS_ACCESS_KEY_ID_OPENCODE=AKIA...
AWS_SECRET_ACCESS_KEY_OPENCODE=...
LITELLM_MASTER_KEY=sk-1234
EOF

set -a; source .env; set +a
```

> `haiku` (`global.openai.gpt-5.6-luna`) works from `ap-south-1` via global CRIS. The other 3 models must be available in `AWS_REGION_OPENCODE`; if you switch to `bedrock_mantle` for Luna, set region to `us-east-1` and add `api_base: https://bedrock-mantle.us-east-1.api.aws/v1`.

## 3. Start the proxy

```bash
# uv
uv run litellm --config config.yml --port 4000

# venv
litellm --config config.yml --port 4000

# with master key auth
LITELLM_MASTER_KEY=sk-1234 litellm --config config.yml --port 4000

# detailed debug (shows Bedrock request signing)
litellm --config config.yml --port 4000 --detailed_debug

# custom host/workers
litellm --config config.yml --host 0.0.0.0 --port 4000 --num_workers 4
```

Health check:

```bash
curl http://localhost:4000/health
curl http://localhost:4000/v1/models -H "Authorization: Bearer sk-1234"
```

## 4. Use it

Point any OpenAI-compatible client at `http://localhost:4000/v1`.

```bash
# haiku → OpenAI Luna
curl http://localhost:4000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer sk-1234" \
  -d '{"model":"haiku","messages":[{"role":"user","content":"ping"}]}' | jq

# fable / opus / sonnet
curl http://localhost:4000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer sk-1234" \
  -d '{"model":"sonnet","messages":[{"role":"user","content":"write a hello world in python"}]}' | jq

# With OpenAI SDK
# openai.api_base = "http://localhost:4000/v1"
# openai.api_key = "sk-1234"
```

Claude Code / Codex example (`~/.codex/config.toml` or env):

```toml
model = "haiku"
# base URL = http://localhost:4000/v1
```

## 5. Verify Luna wiring

```bash
uv run python -c "import yaml; c=yaml.safe_load(open('config.yml')); print([(m['model_name'], m['litellm_params']['model']) for m in c['model_list']])"
uv run python -c "import litellm; print(litellm.get_model_info('bedrock/global.openai.gpt-5.6-luna'))"
```

Expected for Luna: `input_cost_per_token=2e-07` ($0.20/M), `output_cost_per_token=1.2e-06` ($1.20/M), `supports_tool_choice=True`, `mode=chat`.

## Troubleshooting

- `AccessDeniedException` / `ValidationException: inference profile not found` — enable model access in Bedrock console and attach the CRIS IAM policy above.
- `Model not found` — check `model: bedrock/global.openai.gpt-5.6-luna` spelling; list Bedrock models with `aws bedrock list-foundation-models --region us-east-1`.
- `400 UnsupportedParamsError: tool_choice` — upgrade litellm: `uv sync --upgrade` (needs cost map with Luna, bundled from `1.93+`).
- Region errors for Luna — Luna only in `us-east-1` / `us-east-2` / `us-west-2` via mantle; `global.` CRIS solves this from `ap-south-1`.
- Timeout — bump `request_timeout` in `config.yml` or add `--timeout` at client.

## Config reference

See `config.yml` for full config. `litellm_settings.drop_params` lets `stop` be dropped for models that do not support it. `litellm_settings.modify_params` enables Bedrock message normalization, including dummy assistant continuations required between consecutive user/tool blocks. All secrets stay in env vars, not in YAML.
