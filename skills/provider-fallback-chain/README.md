# Provider Fallback Chain — Your API Key Goes Down in 7 Seconds

> **Problem:** Your AI agent's API provider returns a 402 (Insufficient Balance), 429 (Rate Limited), or 503 (Service Down). The agent stops working — but you have a backup key. How do you switch in 7 seconds, not 30 minutes?

## The core insight

**Don't wait for the agent to fail and then scramble.** Configure your agent with a fallback provider hierarchy *before* the primary key dies. When a 402 hits, the machine switches; a human shouldn't be in the loop.

Most agent frameworks support fallback configuration at startup — but the default is usually "no fallback." You need to set it up once, then forget it.

## The setup

### Config structure (Hermes)

```yaml
model:
  default: deepseek-v4-flash
  provider: deepseek        # primary
fallback_providers:
  - provider: custom
    model: deepseekv4flash-0731
    base_url: https://backup-api.example.com/v1
```

This tells the agent: try DeepSeek official first; if it fails, retry with the custom endpoint.

### Who owns the fallback chain

The fallback chain is **user-owned** — the agent configures it at your direction, not on its own initiative. The agent should never add/remove fallback entries without asking. Rationale: fallback costs real money (different provider = different pricing), and the user needs to know which provider is burning their budget.

## Key diagnostics

| Symptom | Diagnosis | Fix |
|---------|-----------|-----|
| Agent "works" but feels worse | `grep "switching to fallback" agent.log` — silently falling back due to quota | Check primary billing; switch default or top up |
| 402 errors in logs | `curl` test with primary key yields `Insufficient Balance` | Top up or swap the default model |
| Requests route to wrong endpoint | `grep "base_url=" agent.log` — compare with `providers.<name>.base_url` in config | Sync model.base_url with providers.base_url |

## Verification

```bash
# Test primary key
curl -s https://api.primary.com/v1/chat/completions \
  -H "Authorization: Bearer $KEY" \
  -d '{"model":"your-model","messages":[{"role":"user","content":"hi"}],"max_tokens":5}' \
  | grep -c "choices"

# Confirm fallback is configured
grep -A5 "fallback_providers:" ~/.hermes/config.yaml

# Check if fallback has been recently used
grep -c "switching to fallback" ~/.hermes/logs/agent.log
```

## Real incident

A multi-agent system ran 6 agents all pointed at the same primary key. When the key hit its monthly quota, **every agent silently fell back** to the same secondary endpoint — but only *some* fell back correctly. Others continued hitting the primary, burning retries and timing out.

Root cause: the fallback chain was only configured on profiles where an operator explicitly set it. Profiles created by automation had `fallback_providers: []` — an empty array, which Hermes interprets as "no fallback."

**Lesson:** fallback chains must be set *per profile*, not just at the global level. A centralized config change doesn't propagate to agent profiles automatically.

## Checklist

- [ ] Primary provider key is configured and tested
- [ ] Fallback provider chain is set in config
- [ ] `base_url` is consistent between `model` and `providers` sections
- [ ] Per-profile fallback entries exist for every agent
- [ ] Fallback was confirmed with an actual curl test (not just config inspection)
- [ ] Agent logs show no unexpected "switching to fallback" noise