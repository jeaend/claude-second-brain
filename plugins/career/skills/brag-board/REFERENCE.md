# Brag Board — Reference

Supplementary details for the `brag-board` skill. SKILL.md is the entrypoint; load this when the user needs to wire up the cron job, debug headless runs, or extend the skill with a new source/sync type. See [README.md](README.md) for a visual overview.

## Running headlessly (cron, launchd, Task Scheduler)

The cron path invokes Claude Code non-interactively. This is supported but has a few setup concerns that don't apply to normal interactive use.

### Invocation

Use Claude Code's print/headless mode. Exact flag depends on your Claude Code version — check `claude --help`. The common patterns:

```bash
# Print mode (one-shot)
claude -p "Run the brag-board skill"

# With explicit input
echo "Run the brag-board skill" | claude -p
```

In all cases, set `BRAG_BOARD_MODE=cron` so the skill suppresses prompts and uses config defaults.

### Authentication in the cron environment

Cron runs with a minimal environment — interactive shell configs (`~/.bashrc`, `~/.zshrc`, `~/.bash_profile`) are **not sourced** by default. If your `ANTHROPIC_API_KEY` (or whichever auth method) only lives in those files, the cron job can't see it.

Options:

1. **Inline in the cron line** — least secure, but simplest for personal-machine use:
   ```
   0 9 * * 1-5 ANTHROPIC_API_KEY=sk-... BRAG_BOARD_MODE=cron claude -p "..."
   ```

2. **Source from a file** — keep the key out of crontab:
   ```
   0 9 * * 1-5 . ~/.claude/env && BRAG_BOARD_MODE=cron claude -p "..."
   ```
   With `~/.claude/env` containing `export ANTHROPIC_API_KEY=sk-...` (chmod 600).

3. **macOS keychain / Linux secret-tool** — most secure, requires a wrapper script that fetches the key at runtime.

### MCP availability

MCP servers configured for interactive Claude Code don't automatically work in headless invocations. Two failure modes to watch for:

- MCP server isn't reachable because no one started it (interactive sessions start MCPs on demand; cron may not).
- MCP server requires OAuth or interactive auth that hasn't been refreshed.

**Mitigation**: before relying on a source in cron, test it once with a manual headless invocation:

```bash
BRAG_BOARD_MODE=cron claude -p "Run the brag-board skill for today only"
```

If a source fails, the skill logs the failure to `.brag-log.jsonl` and continues with remaining sources — the cron job won't silently produce empty days.

### Output and debugging

Redirect stdout + stderr to a log file so failures are diagnosable after the fact. The cron-install step bakes the user's brag-board folder path directly into the cron line, so `cron.log` ends up inside that folder:

```
0 9 * * 1-5 ... >> ~/Documents/brag-board/cron.log 2>&1
```

Rotate `cron.log` occasionally (logrotate, or a periodic `mv` in another cron job) — it can grow.

To diagnose a failing cron job:

1. Read `<brag-board-folder>/cron.log` — captures shell-level errors (auth missing, command not found).
2. Read `<brag-board-folder>/.brag-log.jsonl` — captures skill-level info (sources scanned, failures per source).
3. If both are silent, the cron itself isn't firing. Check with `grep CRON /var/log/syslog` (Linux) or `log show --predicate 'process == "cron"' --last 1d` (macOS).

### Platform-specific schedulers

- **macOS**: `cron` works but Apple deprecates it in favor of `launchd`. For new setups, write a `~/Library/LaunchAgents/com.user.brag-board.plist` with a `StartCalendarInterval`. `launchctl load` it once. More reliable across sleep/wake cycles than `cron`.
- **Linux**: `crontab -e` is the standard path. `systemd` user timers are the modern alternative if you're already using systemd-user services.
- **Windows**: Task Scheduler. Create a basic task with a trigger (daily 9am, weekdays) and an action (run `claude.exe -p "..."` with the env var set in the task's environment).

### Verifying the first cron run

Don't trust cron until you've seen it work. Schedule the first run a few minutes out:

```
*/5 * * * * BRAG_BOARD_MODE=cron claude -p "..." >> ~/.brag-board/cron.log 2>&1
```

Watch `cron.log` and the brag doc for ~10 minutes. Once you've seen one good run, change the schedule to your real one (`0 9 * * 1-5`).

## Companion skills (planned)

The rich entry shape exists because `brag-board` is intended to be the *capture layer* — future skills will be the *output layer*. Documenting them here keeps the field set honest: if a field doesn't earn its keep across these consumers, drop it.

All consumer skills should output, per item: a **short summary line** (1–2 sentences) plus **supporting links** carried verbatim from the entry's `Evidence` field (PRs, tickets, Notion pages, etc.). Never strip the source links — they're the proof and the path back to context.

- **`brag-summary`** — group entries by `Project` over a window. Output per project: short summary of what shipped + supporting links (PR #s, ticket IDs, doc URLs) + timeline + outcome. Reads: `Project`, `Status`, `Date`, `Impact`, `Metrics`, `Evidence`.
- **`brag-to-cv`** — pull achievements suitable for a resume bullet. Output per achievement: metric-grounded one-liner + supporting links as references. Reads: `Type` (shipped/led/decided), `Role` (lead/initiator over contributor), `Skills`, `Metrics`, `Impact`, `Evidence`.
- **`brag-perf-review`** — draft a perf-review self-review for a half/quarter. Output: structured sections by competency, each claim backed by entry summary + supporting links. Reads: `Skills`, `Role`, `Stakeholders`, `Impact`, `Metrics`, `Evidence`, plus raw context for tone.
- **`brag-weekly-recap`** — quick "what did I do this week" summary for a team update or 1:1. Output per item: short summary + supporting links. Reads: `Date`, `Title`, `Impact`, `Project`, `Evidence`.

None of these exist yet — when adding one, validate the field set against its actual needs. If a consumer never reads a field, the field probably shouldn't be required at capture time.

## Source-specific inference hints

What each source type can plausibly yield. Use this when implementing the per-day scan for a given source. If a signal isn't reliably present, leave the field `—` rather than guess.

### Code hosts (GitHub, GitLab, Bitbucket)
The PR description is the **primary source for entry content**. Read it carefully — most fields below can be inferred from it.

- **Title** → PR title (cleaned of conventional prefixes like `feat:`, `fix:`, `[ABC-123]`).
- **Impact** → from PR description: look for "why", "motivation", "context", or business-value language. Extract the 1-line outcome. If the description has a "Summary" or "Why" section, prefer that.
- **Skills** → from PR description + changed files: technologies mentioned (frameworks, languages, infra), patterns described (refactor, migration, perf optimization), domain (auth, billing, search). Free-form, not a fixed allowlist.
- **Project** → repo name (`owner/repo` → `repo` slug). Fallback if no ticket-tracker source.
- **Type** → `shipped` (PR merged), `fixed` (PR with "fix"/"bug" labels), `reviewed` (PR I reviewed but didn't author — promote with caution; reviewer ≠ contributor of substance).
- **Role** → `lead`/`contributor` (PR author), `reviewer` (PR reviewer).
- **Status** → `done` (PR merged), `in-progress` (PR open).
- **Metrics** → +/- LOC, files changed, review approvals. Also pull any numeric claims from the PR description (e.g., "reduced p95 latency by 40%").
- **Evidence** → PR URL, linked issues, linked tickets (rich cross-source signal).
- **Raw context** → full PR description body, verbatim.

### Ticket trackers (Jira, Linear, Asana, etc.)
The ticket description + comments are the **primary source for entry content**, often richer than a PR description because they include problem framing, constraints, and acceptance criteria.

- **Title** → ticket title (cleaned of prefixes like `[ABC-123]`).
- **Impact** → from ticket description: look for "acceptance criteria", "outcome", "goal", "definition of done". Synthesize the 1-line business or technical outcome. Resolution comment (when present) often has the cleanest impact statement.
- **Skills** → from ticket description + labels/components: domain (search, auth, billing), technologies, methodologies (incident response, design review, cross-team coordination).
- **Project** → epic name (best signal — epics already group work into projects).
- **Type** → `shipped`/`built` (Story closed), `fixed` (Bug closed), `learned` (Spike closed), `decided` (RFC/Decision closed).
- **Role** → `lead`/`contributor` (assignee), `initiator` (reporter), `reviewer` (commenter only).
- **Status** → direct from ticket status field.
- **Stakeholders** → ticket watchers/commenters (roles only, never real names).
- **Metrics** → story points, time-in-status, linked PR count. Also pull any numeric claims from the ticket description or resolution comment.
- **Evidence** → ticket URL, linked PRs, linked docs.
- **Raw context** → full ticket description + resolution comment, verbatim.

### Calendars (Google, Outlook)
- **Project** → calendar event series name / parent meeting.
- **Type** → infer from keywords: `presented` (demo, present, showcase), `mentored` (1:1 mentee, coaching), `decided` (decision review, RFC review).
- **Role** → `initiator` (organizer), `contributor` (attendee), `presenter` (explicit role in event title).
- **Stakeholders** → attendee count + roles (never names).
- **Evidence** → calendar event URL, attached docs.

### Messaging (Slack, Teams)
- **Hard rule**: filter strictly to messages *you posted*. Never pull others' messages.
- **Scope**: broad — any channel/DM the user has access to. The filter happens at the content level (below). Optional `channels_blocklist` in config to always exclude specific channels.
- **Content filter** (the real work): only include messages that signal a **work accomplishment** — shipped, launched, decided, mentored, presented, learned-with-substance. Explicit excludes:
  - Personal venting / frustration ("ugh this is awful")
  - Complaints about coworkers
  - Social chatter (birthdays, lunch plans, weekend banter)
  - Logistics ("moving the meeting to 3pm")
  - "I'm so tired" / mood content
  - One-line acknowledgements without substance ("nice!", "lgtm", "thx")
- **The line**: the message describes *work product, decisions, learnings, or outcomes* — not feelings, social activity, or coordination. When unsure, exclude. False negatives (missed brags) are fine; false positives (vent surfacing in a brag doc) are not.
- **Type** → `shipped` (keywords: shipped, launched, deployed, ship-it), `learned` (keywords: TIL, learned, figured out, with concrete content), `decided` (keywords: decided, going with, plan is, with rationale).
- **Role** → usually `lead` if you're announcing the work.
- **Stakeholders** → channel scope only (e.g., "#eng-platform announcement"), not individual reactions.
- **Evidence** → permalink to the message.

### Docs / wikis (Notion, Confluence, Google Docs)
- **Project** → parent page / workspace section.
- **Type** → `learned` (research/exploration docs), `decided` (RFC, ADR, design doc), `presented` (slide decks).
- **Role** → `lead` (author of substantial edits — measure by diff size if available), `contributor` (minor edits only — usually skip).
- **Status** → `done` (published/locked), `in-progress` (draft).
- **Evidence** → doc URL, linked discussions.

### Cross-source enrichment
When the same accomplishment appears in multiple sources (PR linked to a ticket linked to a calendar demo), **merge inference**: pick the highest-confidence value per field. Ticket-tracker fields usually win for `Project` and `Status`; code-host fields win for `Metrics` (LOC); calendar wins for `Type: presented`. Source ID should be the *primary* source (the one the user is most likely to remember) — usually the ticket if present, else the PR.

## Extending the skill

### Adding a new source type

Sources are anything that can yield candidate accomplishments for a date window. To add one:

1. Pick a source-ID prefix that won't collide (`figma:`, `confluence:`, `gmail:`, etc.).
2. In the per-day scan, when the source type is in `config.sources`, query items in the day's window.
3. Map each item to a candidate entry with: title, impact (best-guess from item description), evidence (link to the item), source ID, tags.
4. Document the query/filter shape in `config.sources` so the user knows what to put there.

Don't hard-code provider names beyond the source-ID prefix. The actual querying goes through whatever MCP is configured.

### Adding a new sync target

Sync targets receive entries after the local write succeeds. To add one:

1. Pick a `type` string for the config (`notion`, `google-docs`, `obsidian`, etc.).
2. Document the required identifier fields in `config.sync_to` (page ID, doc ID, vault path).
3. Implement entry rendering for that target (markdown → Notion blocks, markdown → Google Docs API requests, etc.).
4. On failure, log to `.brag-log.jsonl` `sync_results` and continue — never roll back the local write.

## Schema reference

### Config

Lives at `<folder>/config.json` (folder set by user in setup, default `~/Documents/brag-board/`). A one-line pointer file at `~/.brag-board/path` records the absolute path to this config so subsequent runs can find it.

```jsonc
{
  "schema_version": 1,
  "folder": "~/Documents/brag-board/",
  "doc_filename": "brag-doc.md",
  "timezone": "America/Toronto",
  "user_instructions": "Focus on technical leadership and cross-team work. Skip minor flake fixes.",
  "sources": [
    { "type": "github", "filter": { "author": "me" } },
    { "type": "calendar", "filter": { "keywords": ["shipped", "launched", "demo"] } },
    { "type": "slack", "filter": { "channels_blocklist": [] } }
  ],
  "sync_to": [
    { "type": "notion", "page_id": "abc123" },
    { "type": "slack", "target": "@me", "format": "summary" }
  ],
  "cron": { "schedule": "0 9 * * 1-5", "window": "24h" },
  "confirm_before_writing": true
}
```

- `user_instructions`: free-form preferences from the wizard's "any other instructions" step. Applied to every run's candidate filter. Empty string if the user has no extra preferences.
- `folder` + `doc_filename` together resolve the brag doc path. All sibling files (`.brag-log.jsonl`, ad-hoc files, `cron.log`) live in the same folder.

### `.brag-log.jsonl` (one record per run)

Lives at `<folder>/.brag-log.jsonl`.

```jsonc
{
  "run_at": "2026-05-23T08:00Z",
  "mode": "cron",                          // "cron" or "on-demand"
  "window": "24h",
  "dates_scanned": ["2026-05-23"],
  "sources": ["github", "calendar", "slack"],
  "candidates": 5,
  "appended": 3,
  "skipped_dup": 2,
  "promoted_to_main": [],                  // on-demand: entries added to primary doc from ad-hoc
  "updated_in_main": [],                   // on-demand: sparse entries upgraded with richer info
  "source_failures": [],
  "sync_results": [
    { "type": "notion", "ok": true },
    { "type": "slack", "ok": true }
  ],
  "output_file": "brag-doc.md"             // or "brag-doc-adhoc-2026-05-23-last-2-weeks.md" for on-demand
}
```

### Entry shape (markdown)

See [SKILL.md](SKILL.md) → "Entry shape" for the canonical field list and inference rules. Summary:

- Every entry includes **all** fields, with `—` for unknowns.
- Required fields (no `—` allowed): `Date`, `Evidence` (≥1 link), `Source`, `Logged`.
- All other fields are nullable via `—`.
- Raw context goes in a collapsed `<details>` block, omitted if the source has no body.

The rich shape exists so downstream skills (project summaries, CV updates, perf-review drafts) can group/filter by `Project`, `Type`, `Role`, `Status`, `Skills` without re-fetching the original source.
