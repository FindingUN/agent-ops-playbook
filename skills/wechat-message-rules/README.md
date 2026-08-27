# WeChat Message Rules — Keep an Agent Alive on Chat Platforms

> **Problem:** Your AI agent sends messages to users via WeChat. The platform has strict rate limits, character caps, and retry behavior. You lose message delivery — or worse, the agent gets rate-limited for 30 seconds and all subsequent delivery fails. The user sees silence and thinks the agent is dead.

## The core insight

**Chat platforms are not LLM outputs.** A model generates text in milliseconds; a chat platform requires pacing, chunking, retry logic, and cooldown awareness. Treat the delivery channel as a first-class system, not a simple pipe.

## Platform constraints (WeChat-specific)

| Constraint | Limit | What happens if you break it |
|-----------|-------|------------------------------|
| Single message length | ≤2000 characters | Delivery fails silently |
| Inter-message interval | ≥3 seconds | Rate limited (30s + cooldown) |
| Max messages per burst | ≤5 | Later messages dropped |
| File size | varies per type | Upload silently truncated |
| Retry | 3 attempts, exponential backoff | 4th attempt = permanent failure |

## Rules

### 1. Split long output, don't truncate

If the LLM output exceeds 2000 characters, split it into chunks of ≤1900 chars (leave margin). Each chunk is a separate delivery with the ≥3s gap.

```bash
# Example: chunk a long message
split -l 50 long_message.txt chunk_
```

### 2. Pace, don't dump

Even if the LLM returns 6 paragraphs, deliver them over ≥15 seconds with gaps. Don't pipe the full output through in one burst — the rate limit will cause the *entire batch* to fail, not just the last message.

### 3. Retry with meaning

```python
# Pseudo: retry with jitter and max_attempts
for attempt in 1..3:
  try:
    send(message)
    break
  except RateLimited:
    wait(attempt * 5 + random(0, 3))
```

A single rate limit is not an emergency. 3 consecutive rate limits with the same message means the message is too long or the burst is too tight — adjust and retry.

### 4. Diagnose delivery failures

When a user says "I didn't see your message":

```bash
# Check the delivery output
ls -lt ~/.hermes/profiles/<profile>/cron/output/<job_id>/
# Look for "delivery error" in agent.log
grep "delivery error\|rate limited" ~/.hermes/logs/agent.log | tail -5
```

Two common patterns:
- **"rate limited; cooldown active for 30.0s"** — burst was too fast, let the cooldown pass
- **"delivery error: Weixin send failed"** — could be message too long, network issue, or the iLink bridge disconnect

### 5. Know your category, act accordingly

The agent should self‑identify its message category (alert, report, daily briefing, reply) and apply the appropriate priority. High-priority messages (crash alerts, 402s) can shorten the interval but never below 1.5s.

## Verification

```bash
# 1. Check recent delivery health
grep "delivery" ~/.hermes/logs/agent.log | tail -10

# 2. Check if rate limits occurred
grep -c "rate limited" ~/.hermes/logs/agent.log

# 3. Check message lengths for rule compliance
# (monitor script that warns if any message >1900 chars)
```

## Real incident

A daily inspection cron produced a well-formatted report, delivered to WeChat at 22:00. The report triggered a 30-second rate limit because the cron output was delivered as a single-line burst through the gateway. The rate-limit cooldown coincided with the next cron (22:05), causing two consecutive delivery failures. The user saw nothing for 3 days and assumed the system was broken.

Root cause: the cron job's output was delivered as multiple chunks without pacing — the gateway sent chunks at LLM speed (milliseconds between messages), which WeChat treated as a spam burst.

**Lesson:** Always pace multi-message delivery to chat platforms. A 3-second gap costs nothing and prevents the entire report from being lost.

## Checklist

- [ ] Single message ≤2000 chars (with 100-char safety margin)
- [ ] Multi-message delivery has ≥3s interval between chunks
- [ ] Burst ≤5 messages per delivery window
- [ ] Retry logic implemented (3 attempts with jitter)
- [ ] Delivery errors are logged and diagnosable
- [ ] Agent self-identifies message category for priority routing