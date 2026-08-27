# Model Routing Diagnostics — Your Agent Doesn't Know What Model It's On

> **Problem:** You changed the model config. The agent is still using the old model. HTTP 400/401/404 errors appear. You spend 30 minutes debugging — and discover it was a base URL mismatch.

## The core insight

**Config changes don't take effect until the gateway restarts.** But even after restart, *existing sessions* cache the model name they were created with. There are three layers that must all agree:

1. `config.yaml` — what you edited
2. Running gateway — what was loaded at startup
3. Session route cache — what the session uses for API calls

A mismatch at any layer causes a failure that looks like "the model doesn't exist."

## The one-command diagnosis

```bash
# Check all three layers at once
echo "=== Config ===" && grep -E "model:|base_url:" config.yaml | head -4 && \
echo "=== Running sessions (last 5 API calls) ===" && \
grep "model=" ~/.hermes/logs/agent.log | tail -5
```

Output tells you exactly which layer is out of sync.

## Common mismatch patterns

### Pattern A: base_url mismatch
```yaml
model:
  default: deepseekv4flash-0731
  base_url: https://api.old.com/v1       # ← old, never updated
providers:
  custom:
    base_url: https://api.new.com/v1     # ← new, but model doesn't use it
```

**Symptom:** 400 "model not found" even though you "just updated the config."
**Fix:** model.base_url must match the provider it routes through.

### Pattern B: stale session cache
**Symptom:** Config shows `deepseekv4flash-0731`, gateway restarted after config change — but old sessions still log `model=deepseek-v4-flash` and get 404.

**Fix:** Delete the stale session (`hermes sessions delete <session-id>`). New sessions pick up the correct model name.

### Pattern C: gateway didn't restart
**Symptom:** Config was modified 3 days ago, gateway PID shows it started before that.

```bash
ps -p $(lsof -iTCP:8643 -sTCP:LISTEN -t) -o lstart=   # e.g. "Tue Aug 11"  
stat -f "%Sm" config.yaml                              # e.g. "Thu Aug 14"  
```
Config changed after gateway started → changes never loaded.

**Fix:** Restart the gateway. On macOS: `launchctl kickstart -k gui/$(id -u)/ai.hermes.gateway` (or use the profile-specific variant).

## Verification

```bash
# 1. Confirm config matches reality
echo "model.base_url:"; grep "base_url:" config.yaml | head -1
echo "providers.<current>.base_url:"; grep -A2 "custom:" config.yaml | grep base_url

# 2. Confirm recent API calls are going where you expect
grep "API call\|base_url=" ~/.hermes/logs/agent.log | tail -3

# 3. Quick inference test (not just /v1/models which doesn't authenticate)
curl -s POST https://<your-endpoint>/v1/chat/completions \
  -H "Authorization: Bearer $KEY" \
  -d '{"model":"your-model","messages":[{"role":"user","content":"OK"}],"max_tokens":5}'
```

## Real incident

An operator changed `model.base_url` to point at a new aggregator (YHAPI), but `providers.custom.base_url` still pointed at the old endpoint (Mimo). The request routed to Mimo, which rejected the model name (which only existed at YHAPI). The operator spent 30 minutes debugging "why doesn't the model exist at YHAPI?" — it never hit YHAPI at all.

**Lesson:** A change to `model.base_url` is meaningless unless `providers.<name>.base_url` is also in sync. Check both.

## Checklist

- [ ] `model.base_url` matches `providers.<used>.base_url`
- [ ] Gateway restart is newer than config change (`ps lstart` vs `stat mtime`)
- [ ] API call logs show correct `base_url=` and `model=` for recent requests
- [ ] A curl inference test returns 200, not 400/401/404
- [ ] Stale sessions using old model names have been identified (delete if problematic)