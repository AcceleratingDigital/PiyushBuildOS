# Model Access Verification

> **When to run:** Before creating any crons for a new project. Before a major release.
> After any LiteLLM proxy config change. After a provider outage.
> **Who runs it:** Process Coordinator.
> **Goal:** Confirm every model in the project's tool matrix is reachable and responding
> before agents start building. Discover failures now, not mid-build.

---

## Step 1: Verify LiteLLM Proxy is Reachable

```bash
PROXY_URL="http://[LITELLM_HOST]:[PORT]"

# Health check
curl -s "$PROXY_URL/health" | python3 -m json.tool

# Model list
curl -s "$PROXY_URL/v1/models" -H "Authorization: Bearer $LITELLM_API_KEY" \
  | python3 -c "import json,sys; [print(m['id']) for m in json.load(sys.stdin)['data']]"
```

**Expected:** HTTP 200 with model list. If proxy is down: stop, fix proxy before proceeding.

---

## Step 2: Test Each Role's Primary Model

For each model in your project's Tool & Model Matrix, run a minimal completion:

```bash
# Generic test — replace MODEL_NAME and PROXY_URL
curl -s "$PROXY_URL/v1/chat/completions" \
  -H "Authorization: Bearer $LITELLM_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "[MODEL_NAME]",
    "messages": [{"role": "user", "content": "Reply with OK only."}],
    "max_tokens": 5
  }' | python3 -c "import json,sys; r=json.load(sys.stdin); print(r['choices'][0]['message']['content'])"
```

Run for each model:
- [ ] Primary model for Build Manager
- [ ] Primary model for Requirements agent
- [ ] Primary model for Coder (claude CLI)
- [ ] Primary model for QA (opencode CLI)
- [ ] Primary model for Reviewer security (codex CLI)
- [ ] Primary model for Reviewer performance (codex CLI)
- [ ] Primary model for Docs/Site (claude CLI)

**Expected:** Each returns "OK" or similar. Log any failures.

---

## Step 3: Test Each Fallback Model

For any primary that failed, verify its fallback immediately:

```bash
# Same test, with fallback model name
```

- [ ] All primaries pass → proceed
- [ ] Some primaries fail, fallbacks pass → proceed with warning, fix primaries
- [ ] Any fallback fails → STOP. Do not create crons until at least one model per role is confirmed.

---

## Step 4: Test CLI Tool Access

Each agent role uses a different CLI. Verify each is installed and can reach the proxy:

```bash
# claude CLI
ANTHROPIC_BASE_URL="$PROXY_URL/v1" ANTHROPIC_API_KEY="$LITELLM_API_KEY" \
  claude -p "Reply with OK only." --model [MODEL_NAME] --dangerously-skip-permissions

# opencode CLI (QA)
opencode run --model [MODEL_NAME] "Reply with OK only."

# codex CLI (Reviewers)
codex --model [MODEL_NAME] "Reply with OK only."
```

- [ ] claude CLI responds ✅
- [ ] opencode CLI responds ✅
- [ ] codex CLI responds ✅

---

## Step 5: Test Hermes Cron Model Access

Cron jobs run in a fresh session. Verify the model they'll use is accessible from a cron context:

```bash
# Check ~/.hermes/config.yaml for the configured default model
grep -A5 "model:" ~/.hermes/config.yaml

# Verify it matches what you intend for cron agents
```

---

## Step 6: Document Results

Before proceeding, write a one-line summary to COORDINATION.md in the project:

```
[DATE] PROCESS COORDINATOR: Model verification complete.
  Primaries: [list pass/fail per role]
  Fallbacks: [list pass/fail per role]
  CLIs: claude ✅ / opencode ✅ / codex ✅
  Proxy: [PROXY_URL] ✅
  Result: [GO / NO-GO]
```

---

## Troubleshooting

| Symptom | Fix |
|---|---|
| Proxy returns 502/503 | Check LiteLLM process: `ps aux | grep litellm`; restart if needed |
| Model returns 404 | Model name wrong or not configured in proxy; check proxy model list |
| Model returns 429 | Rate limit hit on upstream provider; check provider quotas |
| CLI hangs indefinitely | Check `ANTHROPIC_BASE_URL` env var points to proxy, not api.anthropic.com |
| Cron model differs from CLI model | Check cron job's `--model` flag vs `~/.hermes/config.yaml` default |
| Fallback not triggering | Verify `fallback_model` is set in `~/.hermes/config.yaml` and the fallback model is registered in the proxy |
