# Agent Ops Playbook 🛡️

**Running AI agents in production? You'll spend 80% of your time on operations, not prompting.**

This repo is a living collection of battle-tested ops patterns — zero theory, exact commands, and a verification checklist with every playbook. Every pattern here was **written from a real incident** that cost time or money before we figured it out.

---

## 👀 What's inside

| # | Playbook | Problem it solves | New? |
|---|----------|-------------------|------|
| 1 | [Post-Change Audit](./skills/post-change-audit/README.md) | AI modified the system — how do you verify it didn't break everything? | |
| 2 | [Session Archive](./skills/session-archive/README.md) | Conversation history grows forever — archive without losing data | |
| 3 | [Three Rules](./skills/three-rules/README.md) | Before parsing external formats or deploying cron — read the data first | |
| 4 | [Self-Healing Watchdog](./skills/system-monitoring/README.md) | Who restarts the agents when every agent is down? | |
| 5 | [Provider Fallback Chain](./skills/provider-fallback-chain/README.md) | Your API provider goes down — 7 seconds of config keeps you running | 🆕 |
| 6 | [Model Routing Diagnostics](./skills/model-routing-diagnostics/README.md) | Base URL mismatch? Session cache stale? One command finds out | 🆕 |
| 7 | [WeChat Message Rules](./skills/wechat-message-rules/README.md) | Rate limits, retries, dedup — keep an agent alive on chat platforms | 🆕 |

> **"AI wrote the code. The watchdog caught the crash. The audit proved it was fixed."**

---

## 🇨🇳 中文说明

> 这个仓库是 **AI Agent 运维实战手册**——所有内容来自真实线上事故。
> 你跑 agent 时遇到过的：API 挂了自动降级没生效？配置改了没重启？会话日志炸了磁盘？
> 每个 playbook 都是一个具体问题的检查清单，**复制粘贴就能用**。
>
> 7 篇已发布，持续更新中。欢迎 Star ⭐

---

## 🚀 Quick start — 30 seconds

```bash
# Clueless about what's running on your system?
lsof -i -P -n | grep LISTEN

# Need to know if your agent actually restarted after a config change?
grep -c "listening.*port 8642" ~/.hermes/logs/gateway.log

# Cron job that should produce output but you never see it?
hermes cron run <job-id>
```

If those commands gave you useful information, **this repo is for you**. Pick any playbook above and follow its checklist.

---

## ⚡ How to use this repo

1. **Browse** — each playbook is a standalone README with: problem → core insight → step-by-step flow → verification checklist
2. **Copy/paste** — every command is copy-paste safe (tested on macOS/Linux)
3. **Contribute** — hit a new pattern? Open an issue or PR. All patterns must include the real incident that produced them

---

## 📦 5 playbooks in 14 days — what's next

This repo launched 8/13 with 4 playbooks. By 8/27 we've added 3 more from real production hits:

- **Provider fallback chain** — discovered when a $400 bill taught us the auto-fallback didn't exist
- **Model routing diagnostics** — discovered when 30 minutes of debugging was actually a one-command fix
- **WeChat message rules** — discovered when rate-limited delivery took out a week's worth of daily reports

The repo grows as operations happen. **Star it if you want to watch it grow** ⭐

---

## Principles

1. **Verify, don't assume.** Every claim ships with a reproduction command.
2. **Archive, don't delete.** History is cheap; regret is expensive.
3. **Zero-LLM for mechanical work.** Watchdogs and crons don't need a model.
4. **Post-change audit always.** If you changed it, verify it — then record it.

## License

MIT