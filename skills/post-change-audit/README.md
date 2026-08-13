# Post-Change Audit — Verify Before You Trust the AI

> **Problem:** You let an AI agent modify your system (config, gateway, cron,
> environment). It says "done." How do you know it actually works — and didn't
> silently break something else?

## The core insight

An AI agent's *self-report* is not verification. "I updated the config" is a
claim; `grep` showing the new value in the file is evidence. Never confuse the
two. The audit flow below turns every change into a verifiable, reversible,
recorded event.

## The 9-step flow

### 0. Back up before you touch anything

```bash
mkdir -p ~/.hermes/backups/$(date +%Y%m%d_%H%M%S)
cp -a <config-or-data-path> ~/.hermes/backups/$(date +%Y%m%d_%H%M%S)/
```

If a change goes wrong, you roll back in seconds. **No backup, no change.**

### 1. Grep for secrets in the wrong places

Config edits accidentally leak keys into files they don't belong in:

```bash
grep -rn "API_KEY\|TOKEN\|SECRET" <modified-files> | grep -v ".env"
```

### 2. Validate the config file syntax

```bash
python3 -c "import yaml; yaml.safe_load(open('config.yaml'))"   # YAML
python3 -m json.tool config.json                                 # JSON
plutil -lint ~/Library/LaunchAgents/*.plist                      # macOS plist
```

### 3. Check the architecture map matches reality

If you have an architecture doc, diff the port table against actual listening
ports:

```bash
lsof -i -P -n | grep LISTEN | awk '{print $1, $9}'
```

### 4. Sync to persistent memory (NocMem)

Durable knowledge goes to the knowledge tree — the agent's long-term memory —
so future sessions know what changed and why.

### 5. Notify sibling agents

If your change affects other agents (a moved port, a renamed service), tell
them through the inter-agent channel. Silent changes break other agents.

### 6. Verify at runtime, not just at rest

A file being correct is not the same as the service working:

```bash
# config looks right, but does the service actually serve?
curl -s http://127.0.0.1:<port>/health
# does the cron actually fire?
hermes cron run <job-id>
```

### 7. Confirm the process is alive and staying alive

```bash
launchctl list | grep <service>    # exit code + PID
```

KeepAlive services should show a PID and be respawned if killed.

### 8. Clean up residue

Deleted profiles leave trails: launchd entries, cron references, doc mentions,
scripts. Grep the whole system for the old name:

```bash
grep -rn "<deleted-thing>" ~/Library/LaunchAgents/ ~/.hermes/scripts/ 2>/dev/null
```

### 9. Record the decision

- **Decision** → SOUL (the agent's operating constitution)
- **Fact** → MEMORY (cross-session facts)
- **Preference** → USER (who you're helping)

## Real incident example

An agent migrated a trading agent between frameworks. The migration "succeeded"
per the agent's report — but the scheduled jobs silently failed for a day:

- Root cause: the cron `--script` path resolved to the *new profile's* scripts
  directory, while the actual scripts lived in the *old* profile's directory.
- The audit's "verify at runtime" step (`hermes cron run <job-id>`) caught it
  in 30 seconds — a full day earlier than waiting for the morning report.

**Lesson:** if you only check that the file exists, you miss that the *job*
can't find it. Verify the running system, not the resting file.

## Checklist

- [ ] Backup exists (Step 0)
- [ ] No secrets leaked (Step 1)
- [ ] Config syntax valid (Step 2)
- [ ] Architecture doc matches reality (Step 3)
- [ ] Memory/NocMem synced (Step 4)
- [ ] Sibling agents notified (Step 5)
- [ ] Service verified at runtime (Step 6)
- [ ] Process alive + self-healing (Step 7)
- [ ] No residue (Step 8)
- [ ] Decision recorded (Step 9)
