# Session Archive — Keep History, Lose the Noise

> **Problem:** Your agent's conversation history grows forever. The list is
> cluttered, the database balloons, and you're tempted to *delete* old
> sessions to save space. But deleting history is irreversible — and usually
> unnecessary.

## The core insight

**Archiving ≠ deleting.** Archiving soft-hides old sessions (they leave the
list, stay in the database, and remain fully recoverable). Deleting removes
them forever. For "I don't want to lose history," archiving is always the
right answer.

## The mechanics

| Operation | Command | Semantics |
|-----------|---------|-----------|
| **Archive** (recommended) | `hermes sessions archive --older-than 7d --yes` | Soft-hide. Zero data loss, fully recoverable |
| **Prune** (destructive) | `hermes sessions prune --older-than 90d --yes` | Real delete. Irrecoverable, not searchable |
| **Backup** | `hermes sessions export --format jsonl <path>` | Full JSONL export to external storage |

Always `--dry-run` first to preview what would be archived:

```bash
hermes sessions archive --older-than 7d --dry-run
```

## Workflow: backup → preview → archive → verify

### 1. Backup all profiles to external storage

```bash
for P in guardian ops shangzhou; do
  hermes -p $P sessions export --format jsonl \
    "/backup-dir/${P}-sessions-$(date +%Y%m%d).jsonl"
done
```

### 2. Preview the archive scope

```bash
hermes sessions archive --older-than 7d --dry-run
# "N session(s) match (last active before ...)" — review the list
```

### 3. Archive for real

```bash
hermes sessions archive --older-than 7d --yes
# "Archived N session(s). They're hidden from listings but fully recoverable."
```

### 4. Verify nothing was deleted

```bash
hermes sessions stats
# Total sessions / messages should be UNCHANGED — archiving only hides
```

## Automate it (weekly, zero-token)

A pure-script cron job can run this every week with no LLM cost:

```bash
# session-archive-weekly.sh (no-agent cron mode)
# - Exports backup per profile (skip if today's backup exists)
# - Archives sessions older than 7 days
# - Silent when nothing archived; prints summary when something changed
```

Key design: **the script only produces output when it archived something** —
in no-agent cron mode, empty stdout means no notification, so a healthy week
doesn't spam you.

## Pitfalls

- **Parsing the archive count:** `grep -oE '[0-9]+'` will match the year in
  timestamps ("2026" → "2026 sessions"). Use
  `sed -n 's/^Archived \([0-9][0-9]*\) session.*/\1/p'` instead.
- **Export count vs. file count differ:** Hermes counts logical sessions
  (branches/fragments merged), so the number of session *files* on disk is
  usually higher than the exported session count. Don't be alarmed.
- **Desktop delete button = real delete:** unlike "closing a tab," the delete
  button in a desktop UI calls the real backend deletion. Don't click it
  casually.

## Cross-session continuity (why you can delete the tab)

New sessions don't need the old conversation context to be useful. Agents stay
continuous through **memory layers**, not through keeping every conversation
open:

1. **MEMORY.md** — compact cross-session facts, injected every session
2. **NocMem** — structured knowledge tree, retrieved on demand
3. **Session search** — "what did we do about X" finds it in history
4. **Skills** — procedures that survive any session

**Daily habit:** start a fresh session each day (saves tokens — no giant
context window to pay for), archive old ones weekly, never delete.
