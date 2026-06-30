# Claude Code with local / alternative backends

Claude Code talks to Anthropic's API by default, but two env vars redirect it anywhere
that speaks the Anthropic Messages API:

```bash
export ANTHROPIC_BASE_URL="http://localhost:11434"   # point at your gateway/model
export ANTHROPIC_AUTH_TOKEN="ollama"                  # any non-empty string
unset  ANTHROPIC_API_KEY                              # must be absent or empty
```

## Options

### Ollama (via LiteLLM gateway)
Ollama alone won't work — it speaks the OpenAI API, not Anthropic's. You need a
translation layer:

```bash
# 1. start LiteLLM pointing at Ollama
litellm --model ollama/qwen2.5-coder:7b --port 4000

# 2. point Claude Code at it
export ANTHROPIC_BASE_URL="http://localhost:4000"
export ANTHROPIC_AUTH_TOKEN="sk-local"
export CLAUDE_CODE_ENABLE_GATEWAY_MODEL_DISCOVERY=1
```

GLM (e.g. `glm-4-flash`) works the same way — pull the model in Ollama, swap the
model name.

### LM Studio
Has a native Anthropic-compatible endpoint — no gateway needed:

```bash
export ANTHROPIC_BASE_URL="http://localhost:1234"
export ANTHROPIC_AUTH_TOKEN="lmstudio"
```

Load any model in LM Studio, then `claude --model <lmstudio-model-name>`.

### OpenRouter (cloud, not local)
Useful when you want a non-Anthropic model but don't want to run it locally:

```bash
export ANTHROPIC_BASE_URL="https://openrouter.ai/anthropic"
export ANTHROPIC_AUTH_TOKEN="<openrouter-key>"
unset ANTHROPIC_API_KEY
```

## Caveats

- **Tool use / agentic features** degrade with weaker models. For whisperer's
  `/whisper` (single-shot completion, no tool chaining), this barely matters —
  the model just needs to read context and output text.
- **Version pinning**: Claude Code updates sometimes break gateway compatibility.
  Set `autoUpdate: false` in `~/.claude/settings.json` if things break after an upgrade.
- **vLLM**: versions ≥ 2.1.154 broke with vLLM endpoints — downgrade or pin if using vLLM.

## Why this matters for whisperer

Whisperer doesn't care what model runs the completion — the broker and extension are
model-agnostic. When Claude Code's backend is switched to Ollama/GLM/LM Studio,
`/whisper` keeps working with zero changes. A local 7B coder model is more than
enough for `low`/`medium` effort completions and costs nothing per token.

## Persist the config

```bash
claude config set -g env.ANTHROPIC_BASE_URL "http://localhost:4000"
claude config set -g env.ANTHROPIC_AUTH_TOKEN "sk-local"
```
