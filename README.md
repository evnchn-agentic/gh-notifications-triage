# gh-notifications-triage (skill — brick)

One-shot triage of an unmanageable GitHub notifications inbox down to a task list (~25 items),
via REST API. Captures the empirical quirks of the GitHub notifications API discovered while
clearing a 142-item inbox on 2026-06-01.

**Status:** brick — first draft, asking for jade. See bottom of `SKILL.md` for refinement
invitations.

**Install (Claude Code):**
```bash
ln -s "$PWD/SKILL.md" ~/.claude/skills/gh-notifications-triage/SKILL.md
```

**Audience:** maintainers of high-traffic OSS repos (nicegui, rosys, etc.) whose real
@-mentions get drowned by subscribed/CI/closed-PR noise.

Author: @evnchn · 2026-06-01 · drafted with Claude Opus 4.7
