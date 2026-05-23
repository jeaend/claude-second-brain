---
name: brag-board
description: Captures professional accomplishments to a personal brag doc — runs daily via cron or on-demand for any timeframe. Use when the user mentions brag doc, brag board, tracking accomplishments, perf review prep, "log a win", "I shipped X", "what did I accomplish last week/quarter", or wants to back-fill a timeframe of work — even if they don't say "brag-board" explicitly.
---

# Brag Board

Scans configured sources (GitHub PRs, calendar, etc. — whatever MCPs are available) and appends accomplishments to a personal markdown brag doc. Designed to run daily on a cron, with ad-hoc backfills for "last 2 weeks", "since April 1", quarterly perf-review prep, etc.

## When to use

Trigger phrases:

- "log a win" / "add to brag doc" / "update brag doc"
- "I shipped X" / "I just launched Y" / "track this win"
- "I need to keep track of my accomplishments"
- "what did I accomplish last week / this quarter / since X"
- "draft a perf review from my brag doc"
- "back-fill the last two weeks"
- "set up daily brag tracking"

## First-run setup

If `~/.brag-board/config.json` does not exist, run the setup wizard:

1. Ask for the primary brag doc path. Default: `~/Documents/brag-doc.md`.
   - If the file exists: read it, build the dedup index from any source IDs it already contains, warn (but don't refuse) on malformed entries.
   - If missing: create it with a schema-version header (see "Entry shape" below).
2. **Discover sources broadly.** Enumerate every MCP detected at runtime and offer any that could plausibly surface accomplishments — be inclusive, not selective. Common categories:
   - **Code hosts** — GitHub, GitLab, Bitbucket (PRs, commits, merged branches).
   - **Ticket trackers** — Jira, Linear, Asana, etc. (closed/shipped tickets, epics for `Project` inference).
   - **Calendars** — Google Calendar, Outlook (events with keywords like "shipped", "launched", "demo", "presented", "1:1").
   - **Messaging** — Slack, Teams (messages *you posted* with shipping language; never pull others' messages).
   - **Docs / wikis** — Notion, Confluence, Google Docs (pages you authored or substantially edited).
   - **Any other MCP** that exposes user activity — ask the user whether to include it.

   For each enabled source, ask for filters (e.g., author=me, channel allowlist, keyword list). See [REFERENCE.md](REFERENCE.md) → "Source-specific inference hints" for what each source type can yield.
3. Ask about optional sync targets. List available MCPs that could be destinations (Notion, Google Docs, etc.). For each, ask for the target identifier (page ID, doc ID). **Warn the user explicitly**: "Sync sends full entry content including raw source snippets to <target>. Continue?" — entry content may include real names, ticket IDs, and other data they may not want in the cloud target.
4. Ask about cron: schedule (default `0 9 * * 1-5` — weekdays at 9am) and default window (default `24h`).
5. Ask for the time zone to use for day-grouping. Default: local TZ. Sources report in mixed TZs (GitHub UTC, calendar local) — the skill normalizes to this TZ when bucketing entries to dates. Matches "what did I do today" intuition.
6. Write `~/.brag-board/config.json`. Schema:

```json
{
  "schema_version": 1,
  "doc_path": "~/Documents/brag-doc.md",
  "timezone": "America/Toronto",
  "sources": [
    {"type": "github", "filter": {"author": "me"}},
    {"type": "calendar", "filter": {"keywords": ["shipped", "launched", "demo"]}}
  ],
  "sync_to": [
    {"type": "notion", "page_id": "abc123"}
  ],
  "cron": {"schedule": "0 9 * * 1-5", "window": "24h"},
  "confirm_before_writing": true
}
```

7. Offer to help install the cron entry. Don't silently edit `crontab` / `launchd` / Task Scheduler — always require explicit confirmation. Flow:

   a. Detect platform (macOS / Linux / Windows) and suggest the best mechanism (`launchd` on macOS, `crontab` on Linux, Task Scheduler on Windows). See [REFERENCE.md](REFERENCE.md) for per-platform details.

   b. Show the exact command, file, or task definition the user would create. Example for Linux:

      ```
      0 9 * * 1-5 BRAG_BOARD_MODE=cron <claude-invocation> >> ~/.brag-board/cron.log 2>&1
      ```

   c. Ask: install now, save the snippet for the user to install manually, or skip cron setup. If "install now", run the install command with the user confirming each step — never edit scheduler config without explicit go-ahead.

   d. After install, verify: read the scheduler back (`crontab -l`, `launchctl list`, `schtasks /query`) and confirm the entry is registered. Suggest scheduling a test run a few minutes out (see [REFERENCE.md](REFERENCE.md) → "Verifying the first cron run") before relying on it.

To re-run the wizard later: user says "reconfigure brag doc" → overwrite the config file. They can also edit the JSON directly.

## Run flow

Detect mode:

- `BRAG_BOARD_MODE=cron` env var → **cron mode**: use config defaults, no prompts, write to the primary brag doc.
- Otherwise → **on-demand mode**: parse the user's timeframe, write to a new file (see below), prompts according to `confirm_before_writing`.

Then:

1. Read config. If missing, run setup wizard first.
2. Resolve the timeframe to a list of dates (most recent first).
3. For each date, run a **per-day scan** (see below).
4. After all days, log the run to `.brag-log.jsonl` (next to the brag doc).
5. Sync to each `sync_to` target. If a sync fails, log the failure and warn the user but don't roll back the local write.

### Per-day scan

The atomic unit of work — same logic for cron (N=1) and ad-hoc (N=many).

For one date:

1. For each enabled source, query items in that date's window. Tolerate source-specific failures — log and continue with remaining sources.
2. Build candidate entries. Each candidate has a **source ID** (see below).
3. **Dedup**: skip any candidate whose source ID already appears in the primary brag doc. (Ad-hoc files are not consulted for dedup — each ad-hoc artifact is independent.)
4. If `confirm_before_writing: true` and in on-demand mode, show the user the day's candidates. Let them accept all, skip all, or edit individually.
5. Append accepted entries to the destination file (see "Output destinations" below).

## Entry shape

Markdown with date-grouped sections. Append-only — don't rewrite existing entries.

Each entry **always includes all fields** — use `—` (em dash) when a value can't be inferred. Downstream skills (project summaries, CV updates, perf-review drafts) treat `—` as null. Never omit a field just because the value is unknown.

### Day grouping

Group entries by the date in the configured `timezone` (see config in setup wizard). Sources report timestamps in mixed TZs; convert to the configured TZ before bucketing.

```markdown
## 2026-05-23

### Shipped the new onboarding flow
- **Date**: 2026-05-23
- **Project**: onboarding-redesign
- **Type**: shipped
- **Role**: lead engineer
- **Status**: done
- **Impact**: cut activation time from 4 days to 8 hours
- **Metrics**: activation_time_p50: 96h → 8h (-92%); first-week retention: 41% → 58%
- **Skills**: technical leadership, postgres, cross-team coordination
- **Stakeholders**: PM (1), design (2), eng team (6)
- **Evidence**: [PR #1234](url), [launch doc](url)
- **Tags**: onboarding, leadership
- **Source**: github:owner/repo#1234
- **Logged**: 2026-05-23 (cron)

<details>
<summary>Raw context</summary>

> [verbatim PR description / calendar event / Slack message / etc.]

</details>
```

### Field semantics

| Field | Purpose | Inference strategy | Null marker |
|-------|---------|-------------------|-------------|
| `Date` | When it happened (not when logged) | From source timestamp | n/a (required) |
| `Project` | Grouping slug for project summaries | Ticket-tracker epic name (best signal — epics already group work into projects). Fallback: repo name / calendar title → slug | `—` |
| `Type` | Taxonomy for filtering | Ticket type (`Story` → shipped/built, `Bug` → fixed, `Spike` → learned, etc.); PR merged → shipped; calendar "demo" → presented. Default values: `shipped`, `led`, `decided`, `fixed`, `learned`, `mentored`, `presented`. Users can add custom types. | `—` |
| `Role` | Distinguishes "I shipped this" from "I helped" — critical for honest CV. Values: `initiator`, `lead`, `contributor`, `reviewer`, `advisor` | Ticket assignee/reporter; PR author/reviewer; calendar role | `—` |
| `Status` | Projects don't end on ship date. Values: `done`, `in-progress`, `ongoing`, `abandoned` | Ticket status field (best signal); else default `done` for shipped items, `in-progress` otherwise | `—` |
| `Impact` | The "so what" in one line | Best summary from source description | `—` |
| `Metrics` | Numeric outcomes for quotability | Extract numbers from source if present | `—` |
| `Skills` | Competencies demonstrated (CV pull) | Free-form. If `skills_allowlist` in config is set, validate against it | `—` |
| `Stakeholders` | Anonymized roles + counts. **Never real names** | Counts from PR reviewers, calendar attendees | `—` |
| `Evidence` | At least one canonical link | Always from source URL | n/a (required) |
| `Tags` | Free-form labels for searching | User-added | `—` |
| `Source` | Stable source ID (dedup key) | Built from source type + ID | n/a (required) |
| `Logged` | When this entry was written + mode | `YYYY-MM-DD (cron|on-demand)` | n/a (required) |
| Raw context | Verbatim source content so future skills can re-summarize without re-fetching. Collapsed via `<details>` so the markdown stays scannable. | Copy source body verbatim | Omit the `<details>` block if source has no body content |

### Inference rule

When inferring a field: pick the best-confidence value. If confidence is low (multiple plausible values, no clear signal), prefer `—` over a guess. Downstream skills handle null gracefully; they don't recover from wrong values.

## Source IDs

Stable, greppable identifiers used for dedup. Format: `<source-type>:<unique-key>`.

- GitHub PR: `github:owner/repo#1234`
- GitHub issue: `github:owner/repo!1234`
- Ticket tracker: `<tracker>:<ticket-id>` (e.g. `linear:ENG-5678`, `jira:PROJ-1234`)
- Calendar event: `calendar:<event-id>`
- Slack message: `slack:<workspace>:<channel>:<ts>`
- Notion page edit: `notion:<page-id>` (use revision ID if available)
- Google Doc edit: `gdoc:<doc-id>:<revision>`
- Manual entry: `manual:<short-slug>-<date>`

If a source's natural ID isn't stable, fall back to a hash of (date + title + first evidence URL). See [REFERENCE.md](REFERENCE.md) for per-source ID and inference details.

## Output destinations

**Primary (always)**: the local markdown file from `doc_path` (cron) or a generated ad-hoc filename (on-demand). Write to local first — it's the source of truth and the dedup index.

**Sync targets (optional)**: each entry in `sync_to` is best-effort. Failures don't block the local write.

### Ad-hoc filename

For on-demand runs, generate the filename as:

```
brag-doc-<gen-date>-<timeframe-slug>.md
```

Examples:

- "last 2 weeks" on 2026-05-23 → `brag-doc-2026-05-23-last-2-weeks.md`
- "since April 1" on 2026-05-23 → `brag-doc-2026-05-23-since-april-1.md`
- "2026-04-01 to 2026-05-15" on 2026-05-23 → `brag-doc-2026-05-23-2026-04-01-to-2026-05-15.md`

Slug rules: lowercase, hyphenate spaces, strip punctuation. If the file already exists, append `-2`, `-3`, etc.

Place ad-hoc files in the same directory as the primary brag doc.

## Run log (`.brag-log.jsonl`)

One JSON line per run, next to the brag doc. Use for audit and resume.

```json
{"run_at":"2026-05-23T08:00Z","mode":"cron","window":"24h","sources":["github","calendar"],"candidates":5,"appended":3,"skipped_dup":2,"sync_results":[{"type":"notion","ok":true}]}
```

If the run is interrupted, the last successful date is recoverable from this log — next run can resume from there.

## Gotchas

- **Don't bake in any company, tool, or coworker name.** This skill is published publicly. All source/sync target identifiers come from user config at runtime.
- **Cron + ad-hoc concurrent runs**: if both fire simultaneously, dedup by source ID prevents duplicate entries in the primary file. Ad-hoc files are independent so they don't compete. A `.brag-board.lock` next to the doc is a nice-to-have, not required.
- **Don't silently edit the user's crontab/launchd/Task Scheduler.** Print the suggested line; the user installs it.
- **First cron tick on a fresh config**: stick to the configured window (default 24h). Don't auto-backfill — that's what ad-hoc mode is for. Suggest the user run an ad-hoc backfill if they want history.
- **Source failures are isolated**: one broken MCP shouldn't kill the whole run. Log the failure and continue with the remaining sources.
- **Confirm-then-edit UX**: when prompting per-day, support "accept all", "skip day", "edit this entry", "drop this entry". Per-day chunking keeps each prompt manageable.
