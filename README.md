# Agent Ops Playbook 🛡️

**Practical, battle-tested operations playbooks for running AI agents in production.**

This repository collects hard-won operational patterns for anyone running
autonomous AI agents — memory management, post-change auditing, self-healing
watchdogs, and the discipline that keeps agents stable, cheap, and honest.

All patterns here are **written from real production incidents**, not theory.
Every playbook includes exact commands and a verification checklist.

## Why this exists

Running an AI agent is 20% prompting and 80% operations. Agents crash, configs
drift, memory leaks, costs balloon. This playbook is the ops layer that keeps
agents alive and accountable.

## Playbooks

| # | Playbook | Problem it solves |
|---|----------|-------------------|
| 1 | [Post-Change Audit](./skills/post-change-audit/README.md) | AI modified the system — how do you verify it didn't break anything? |
| 2 | [Session Archive](./skills/session-archive/README.md) | Conversation history grows forever — archive without losing data |
| 3 | [Three Rules](./skills/three-rules/README.md) | Before parsing external formats or deploying cron — read the data first |
| 4 | [Self-Healing Watchdog](./skills/system-monitoring/README.md) | Who restarts the agents when the agents are all down? |

## Principles

1. **Verify, don't assume.** Every claim ships with a reproduction command.
2. **Archive, don't delete.** History is cheap; regret is expensive.
3. **Zero-LLM for mechanical work.** Watchdogs and crons don't need a model.
4. **Post-change audit always.** If you changed it, verify it — then record it.

## License

MIT
