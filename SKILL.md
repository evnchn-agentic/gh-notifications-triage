---
name: gh-notifications-triage
description: Use when a GitHub user's notifications inbox is unmanageable (50+ threads, drowning, "can't see what needs my attention") and they want a one-shot cleanup. Triggers on "clean up notifications", "dismiss notifications", "use notifications as task list", "inbox is overflowing", "what GitHub stuff actually needs me", or pre-emptive triage before going on/off rotation. Maintainer-shaped problem: nicegui/upstream repo owners receive a flood of subscribed/CI/closed-PR notifications around their real @-mentions.
---

# GitHub Notifications Triage

> **Brick status:** First pass, derived from one real cleanup (142 → 2 actionable on @evnchn's inbox, 2026-06-01). Heuristics are opinionated; rules of thumb, not laws. Refine on contact.

## Core idea

The default GitHub notifications inbox conflates four orthogonal axes — *unread/read*, *open/closed*, *ball-in-your-court/theirs*, *FYI-watching/active-participation*. Most users want a fifth axis on top: **inbox = task list = things requiring my action now**. This skill operationalizes that view via the REST API.

## Step 1 — Ask the user, propose defaults

Before triaging, surface two choices. Propose defaults; user confirms or modifies. Don't skip this step — the right rule depends on how the user *intends* to use the inbox.

| Question | Default to propose | When user might pick differently |
|---|---|---|
| **What should the inbox represent?** | Task list — only items that need your action | Reading list (keep informational mentions); audit log (keep nothing read) |
| **Target inbox count after cleanup?** | ~25 (manageable in one sitting) | Smaller if they only want hot items; larger if they read context broadly |

Phrase the ask roughly as:

> "Before I touch anything: I'll default to treating the inbox as a task list, targeting ~25 items. Drop everything that's closed, where you replied last, or where you're only subscribed-as-FYI. Sound right, or shape it differently?"

## Step 2 — Fetch the real inventory

```bash
gh api 'notifications?all=true&per_page=50' --paginate \
  -q '.[] | [.id, .unread, .reason, .updated_at, .repository.full_name, .subject.type, .subject.url, .subject.title] | @tsv'
```

**Gotcha #1:** `gh api notifications` without `all=true` defaults to unread-only. Likely 10–20× smaller than the real inbox.

## Step 3 — Classify each thread

For every thread with a `subject.url`, fetch the PR/issue object to get `state`, `merged`, `assignees`, `requested_reviewers`, plus the last comment's author.

**Default classifier — "task list" semantics:**

| Drop if | Reason |
|---|---|
| State is closed or merged | Done, not actionable |
| Last comment author == the user | Ball is in their court |
| `reason == subscribed` AND not @-mentioned in latest activity | FYI-watching, not assigned work |
| Last actor is a bot (`Copilot`, `github-actions[bot]`, `dependabot[bot]`) | Wait for human |

**Keep if** open AND last actor ≠ user AND last actor ≠ bot AND `reason ∈ {mention, comment, author, assign, review_requested}` — OR — user is explicitly in `assignees` / `requested_reviewers`.

**Gotcha #2:** Threads of type `CheckSuite` and `RepositoryAdvisory` have empty `subject.url`. The naive classifier "keeps" them by default. Handle separately: dropping by repo+reason works (e.g., all `ci_activity` from a superseded project).

## Step 4 — Present and confirm

Show the proposed drop list grouped by repo + reason. Get explicit go-ahead. **Do not auto-fire** — this is shared-system-state mutation.

## Step 5 — Execute via DELETE (NOT PATCH)

```bash
DELETE /notifications/threads/<id>   # archives to "Done" (web UI Inbox loses it)
```

**Gotcha #3 — the verb is misleading.** `DELETE` returns 204 No Content and is *equivalent to clicking "Done"* in the web UI. It does NOT permanently delete; threads remain reachable in the Done tab and via `GET notifications/threads/<id>`.

**Gotcha #4:** `PATCH /notifications/threads/<id>` only flips `unread → read`. Marked-read items STAY in the web UI Inbox. The web UI does occasionally auto-archive read threads but anecdotally unreliable — for inbox-as-task-list, you must `DELETE`, not `PATCH`.

```bash
cat /tmp/drop_ids.txt | xargs -P 8 -I{} gh api -X DELETE "notifications/threads/{}" --silent
```

## Step 6 — Verify

Ask the user to refresh `github.com/notifications` in their browser and confirm the count. REST `?all=true` will still show the archived threads — REST has no `is:done` filter and is a poor verification surface.

## Quirk reference

| Concern | Truth |
|---|---|
| Default `gh api notifications` page | unread-only (need `all=true`) |
| Mark Done via REST | `DELETE /notifications/threads/<id>` → 204 |
| Mark unread via REST | ❌ not exposed |
| Filter by Done via REST | ❌ no `is:done` |
| GraphQL `markNotificationAsDone` | ❌ not in public schema (only `updateNotificationRestrictionSetting`) |
| Done auth | classic PAT / `gh` OAuth ✅; fine-grained PATs ❌ per docs; GitHub App tokens untested |
| Recovery | Web UI "Done" tab → Restore |

## Edge cases beyond the 99%

When the inbox shape stretches the rule (a PR where the last meaningful action is a force-push, not a comment; a `review_requested` PR the user already reviewed-with-nits, then the author pushed a fix without re-requesting — user is no longer in `requested_reviewers` but the PR genuinely needs them; a thread where the user is `reason=subscribed` *and* @-mentioned in the latest update; a maintainer-grade inbox with 10K+ threads facing the 5K/hr rate limit), the skill expects the agent to sort it out with the user at runtime — don't bake speculative branches into the rule. The 142 → 2 reference run had no such edge cases.

## Implementation skeleton

Two-file split works fine: classifier in Python (state lookup + rule), executor in bash (xargs DELETE). See `~/rich-renders/gh-notifications-triage-2026-06-01.html` and `/tmp/triage*.py` on the originating machine for the working reference impl.

## When NOT to use

- One specific PR is making noise → use `gh api -X PUT repos/<owner>/<repo>/subscription -f ignored=true` for that thread
- Inbox has <20 items → manual review is faster than triage setup
- User wants notifications-as-archive (audit trail) — this skill drops aggressively
