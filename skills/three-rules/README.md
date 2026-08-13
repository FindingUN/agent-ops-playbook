# Three Rules — The Discipline That Keeps Agents from Wasting Your Time

> **Problem:** An AI agent is working on your system. It guesses formats, waits
> on timers, and declares victory before verifying. Every guess costs you time
> and money. How do you install discipline?

## Rule 0 — Align every action with a project goal

Before doing anything, the agent must answer: **"What project goal does this
action serve?"** If it can't say "after I do X, you can do Y," it shouldn't do X.

Signs of misalignment:
- Executing a TODO mechanically without asking why it exists
- Switching to a random other task because the current one is stuck
- Answering "what now?" by picking a corner to clean up

## Rule 1 — Read the data before writing code

When handling any unfamiliar file/API/protocol, the first step is always to
**dump the real data and look at its structure**.

```bash
# Never assume the format — look at it first
tail -1 file.jsonl | python3 -c "import sys,json; print(json.load(sys.stdin).keys())"
```

> Real incident: an agent assumed a log file stored messages under
> `data.messages`, but the actual field was `data.assistantTexts`. Dumping one
> line would have taken 15 seconds; the assumption cost 2 hours.

**Data decides code. Experience doesn't.**

## Rule 2 — Second-level test loops, not minute-level

Never validate logic by waiting for a scheduler (cron, watchdog, timer).

```bash
# ❌ Edit code → wait for the 60s launchd tick → discover it's wrong
# ✅ Run the script directly, read exit code + output, iterate
bash script.sh
```

10 manual runs = 1 minute of iteration. 1 scheduled run = 1 hour of waiting.

Corollaries that compound:
- **Use port-level kills, not pattern matching:** `lsof -ti :PORT | xargs kill`
  is reliable; `pkill -f pattern` leaks (misses or overkills).
- **Diagnose yourself, don't ping-pong commands to the user:** batch 3-5
  diagnostic commands in one turn, read all output, deliver one conclusion.
- **Verify architecture before downloading:** `uname -m` first, then pick the
  right binary. Downloading arm64 to an Intel Mac wastes a session.
- **Long-running tasks must detach from the session:** background processes
  bound to a chat session die when the session ends. Use the OS scheduler
  (launchd/systemd) with the real command as a direct child — not a bash
  wrapper, which the scheduler cleans up.

## Rule 3 — Don't leave until the loop is closed

A task isn't done until it's verified end-to-end, from source to the user's
eyes. "The file was written" is not "the user saw the result."

- Never propose stopping with one step left
- Never accept a second-hand system-state claim ("agent X says it works" —
  verify with your own tools)
- If asked "are you sure?" and the answer contains "should/maybe/probably" —
  you haven't verified. Go check.

> Real incident: a scanner wrote "result appended to session" in its log, and
> the agent declared success. The user had never seen the result in their
> chat — the gateway runs sessions in memory and never rereads the file.
> Log-level success is not user-visible success.

## When these rules apply

Any task involving **external data formats, scheduled triggers, or
multi-component pipelines**:

- Writing parsers (third-party logs, format conversion)
- Deploying scheduled jobs (cron/launchd/watchdogs)
- Testing multi-component chains (A → B → C end-to-end)

Pure algorithm/math work doesn't need Rule 1 (no external format). Casual chat
doesn't need Rule 3 (no deliverable).

## The cost of skipping discipline

Every skipped rule shows up as the same pattern: the agent says "done," you
discover it isn't, and you pay for another round-trip — often another session,
another model call, another hour. Discipline is the cheapest optimization an
agent can have.
