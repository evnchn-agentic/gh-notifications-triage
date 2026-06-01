# gh-notifications-triage

Claude Code skill for one-shot triage of a flooded GitHub notifications inbox down to a task list (~25 actionable items), via the REST API. Captures the empirical API quirks discovered while clearing a 142-thread inbox to 2.

**Key quirks captured:**
- `gh api notifications` defaults to unread-only — need `?all=true&per_page=50 --paginate`
- `DELETE /notifications/threads/<id>` archives to "Done" (verb is misleading; returns 204; reversible via web UI "Done → Restore")
- `PATCH` only flips read state; leaves threads in Inbox
- REST has no `is:done` filter — verify via web UI

**Default classifier** (task-list semantics): drop closed/merged, drop ball-in-user-court, drop subscribed-FYI, drop bot-last-actor. Keep mentions, assigned, review-requested, and active conversations where the user isn't the last actor.

**Install (Claude Code):**
```bash
mkdir -p ~/.claude/skills/gh-notifications-triage
ln -s "$PWD/SKILL.md" ~/.claude/skills/gh-notifications-triage/SKILL.md
```

**Audience:** maintainers of high-traffic OSS repos (NiceGUI, rosys, etc.) whose real @-mentions drown in subscribed/CI/closed-PR noise.

See `SKILL.md` for the full skill content.

---
Author: [@evnchn](https://github.com/evnchn) · 2026-06-01 · drafted with Claude Opus 4.7
