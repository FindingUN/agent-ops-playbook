# Self-Healing Watchdog — Who Restarts the Agents When They're All Down?

> **Problem:** You run several AI agent gateways. One crashes. Then another.
> Then the one that would have noticed is also down. Who's left to bring them
> back?

## The core insight

**The watchdog must not be one of the things it watches.** A pure-bash,
zero-LLM process scheduled by the OS (launchd/systemd) is the last line of
defense — it survives even when every agent gateway is dead, and it can
restart them all.

Two layers of self-healing:

| Layer | Speed | What it does | Survives? |
|-------|-------|--------------|-----------|
| **launchd KeepAlive** | seconds | Restarts a crashed service automatically | Yes (OS-level) |
| **Watchdog script** | minutes | Checks all services, restarts what's down, alerts | Yes (independent) |

## Layer 1: launchd KeepAlive (per-service)

```xml
<key>KeepAlive</key>
<dict>
    <key>SuccessfulExit</key>
    <false/>
</dict>
```

`SuccessfulExit: false` means: restart unless the process exited cleanly.
This catches crashes within seconds. Verified: kill a gateway → it's back in
~15 seconds with a new PID.

## Layer 2: watchdog script (system-wide)

A bash script scheduled every 5 minutes that:

1. **Checks every gateway** via health endpoint (`curl http://127.0.0.1:<port>/health`)
2. **Restarts what's down** via `launchctl kickstart -k gui/$(id -u)/<service>`
   (independent of the agent processes — it can even restart the agent that
   "owns" it)
3. **Checks disk/memory** and alerts on thresholds
4. **Cleans orphaned processes** (children whose parent died)
5. **Verifies API keys exist** (existence check only — never prints values)
6. **Tracks 24h cost** from the session database

```bash
# launchd plist: run watchdog every 300 seconds
<key>StartInterval</key>
<integer>300</integer>
<key>RunAtLoad</key>
<true/>
```

## Key design decisions

### Health check via port, not process name

`curl http://127.0.0.1:PORT/health` tells you the service actually *serves*.
`pgrep` only tells you a process exists — a wedged process can still be alive
and useless.

### Restart via launchctl, not direct spawn

`launchctl kickstart` goes through the OS service manager: proper env, proper
logging, KeepAlive still applies. Spawning `hermes gateway run &` directly
from the watchdog bypasses all of that.

### Zero LLM

The watchdog is pure bash. No model calls, no tokens, no cost. It runs on a
5-minute timer forever, whether or not any agent is awake.

### Alert channel independent of the agents

If all gateways are down, you can't get an alert through them. Use a
push-notification service (ntfy.sh, Telegram bot) directly from bash — the
alert must not depend on the thing being watched.

## Real incident: the fix that broke the watchdog

A watchdog update added a cost-checking section with a hardcoded key and a
broken quote — the whole script died on syntax error from that point on, and
all later checks (including the new ones) silently stopped running. **A
watchdog that can't even parse is worse than no watchdog** — it looks like it's
watching while doing nothing.

Fix discipline:
- Always `bash -n <script>` after every edit (syntax check)
- Never hardcode secrets in the watchdog (use env files)
- Log every run to a file so you can see it's actually executing
- Run-lock to prevent concurrent executions

## Checklist

- [ ] Every service has KeepAlive with `SuccessfulExit: false`
- [ ] Watchdog scheduled by OS (launchd/systemd), not by an agent
- [ ] Watchdog is pure bash — zero LLM dependency
- [ ] `bash -n` passes after every edit
- [ ] Watchdog logs every run (you can see it's alive)
- [ ] Alert channel does not depend on the watched services
- [ ] Killed a gateway recently and watched it come back (test it!)
