---
name: brag-doc-prep
description: Captures professional accomplishments to a personal brag doc — runs daily via cron or on-demand for any timeframe. Use when the user mentions brag doc, brag board, tracking accomplishments, perf review prep, "log a win", "I shipped X", "what did I accomplish last week/quarter", or wants to back-fill a timeframe of work — even if they don't say "brag-doc-prep" explicitly.
---

# Brag Doc Prep

The **capture layer** for an accomplishments workflow. Scans configured sources (GitHub PRs, calendar, ticket trackers, Slack, etc. — whatever MCPs are available) and appends structured entries to a personal markdown brag doc. Designed to run daily on a cron, with ad-hoc backfills for "last 2 weeks", "since April 1", quarterly perf-review prep, etc.

## What this produces

A markdown brag doc, date-grouped (`## YYYY-MM-DD`), with zero or more structured entries per day (typically 0–5). Each entry has a rich, predictable shape (`Project`, `Type`, `Role`, `Status`, `Impact`, `Metrics`, `Skills`, `Stakeholders`, `Evidence`, raw context). This is the **capture layer** — downstream skills (`brag-summary`, `brag-to-cv`, `brag-perf-review`, `brag-weekly-recap`) consume the rich shape to synthesize CV bullets, perf-review drafts, etc. without re-fetching original sources. See [REFERENCE.md](REFERENCE.md) → "Companion skills" for the consumer roster.

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

The skill keeps a tiny pointer file at `~/.brag-doc-prep/path` containing the absolute path to the user's brag-doc folder. If that pointer does not exist (or the path it points to is missing), run the setup wizard.

**Wizard UX rule (interactive mode)**: ask every wizard step via a **structured question prompt** (the `AskUserQuestion` tool or equivalent UI question component), not as conversational text. Conversational questions get skipped, ignored, or answered ambiguously. Structured prompts force a deliberate choice and give the user "Other" for free-form input when needed. One question per step — don't bundle multiple decisions into one prompt. If running headlessly (`BRAG_DOC_PREP_MODE=cron`), there's no wizard — the config must already exist or the run errors out.

**Persistent corrections rule (applies throughout the skill, not just the wizard)**: whenever the user *corrects* or *adjusts* the skill mid-flow — rejecting a candidate with a reason ("this isn't brag-worthy because X"), refining a filter ("don't include this channel"), changing a workflow note, overriding an inference, or pointing out an error in the output — **propose persisting that correction to `config.json` immediately**, via a structured prompt:

> *"Apply this as a one-time override, or save to config so it sticks for future runs?"*

Defaults to **save** when the correction looks generalizable (a filter, a workflow clarification, a recurring exclude). Defaults to **one-time** when the correction is clearly context-specific (this exact PR doesn't count for some idiosyncratic reason). Always show the user the proposed config diff before writing.

Persistent corrections write to the most specific field they apply to:
- A rejected-candidate reason → append to `user_instructions` or to the source's `workflow_notes`.
- A specific channel/repo/page to exclude → append to that source's `channels_blocklist` / `repo_blocklist` / equivalent.
- A path/folder change → update `folder`, `doc_filename`, or `sync_to[].target`.
- A schedule adjustment → update `cron.schedule` / `cron.window`.

After each persistent change, log it in `.brag-log.jsonl` (`corrections_applied: [...]`) so the user can audit what the skill learned.

The wizard steps:

1. Ask for the brag-doc folder. Default: `~/Documents/brag-doc/`. The folder holds:
   - `brag-doc.md` — the primary brag doc
   - `config.json` — your decisions from this wizard
   - `.brag-log.jsonl` — run history
   - `brag-doc-adhoc-*.md` — ad-hoc backfill files
   - `cron.log` — cron stdout/stderr (if cron is enabled)

   Create the folder if missing. If `brag-doc.md` exists inside it, read it and build the dedup index from any source IDs it already contains; warn (but don't refuse) on malformed entries. If missing, create it with a schema-version header (see "Entry shape" below).
2. **Discover and ask which sources to enable.** Enumerate every MCP detected at runtime, group them by category, then **explicitly ask the user (via a multi-select structured prompt) which to enable** as accomplishment sources. Be inclusive in what you offer — don't filter for the user; let them pick. Common categories the wizard should surface if MCPs are detected:
   - **Code hosts** — GitHub, GitLab, Bitbucket (PRs, commits, merged branches).
   - **Ticket trackers** — Jira, Linear, Asana, etc. (closed/shipped tickets, epics for `Project` inference).
   - **Calendars** — Google Calendar, Outlook (events with keywords like "shipped", "launched", "demo", "presented", "1:1").
   - **Messaging** — Slack, Teams (messages *you posted* — never pull others' messages). Scope can be broad (any channel/DM you have access to). The filtering happens at the **content level**: include only messages that signal a work accomplishment (shipped, launched, decided, mentored, presented, learned-with-substance). Skip personal venting, complaints about coworkers, social chatter (birthdays, lunch plans), logistics. Optional `channels_blocklist` in config for channels to always exclude.
   - **Docs / wikis** — Notion, Confluence, Google Docs (pages you authored or substantially edited).
   - **Data / analytics** — Preset, Tableau, Looker, Metabase (dashboards you built, charts you authored, datasets you defined).
   - **Any other MCP** that exposes user activity — ask the user whether to include it.

   Sources can yield two kinds of entries:
   - **Accomplishments** — work the user did (PRs, tickets, dashboards, etc.).
   - **Shout-outs / recognition** — messages from others thanking, kudoing, or recognizing the user. These come *from others* but are still brag-worthy. The skill captures these too; see "Shout-out filter" in [REFERENCE.md](REFERENCE.md).

   **After picking sources, pre-fill per-source defaults from [REFERENCE.md](REFERENCE.md) → "Vendor defaults", then confirm with the user.** People organize their work very differently — the same MCP yields wildly different signal quality depending on workflow. The wizard should ask a structured follow-up per enabled source: *"How do you use \<source\>? What does it capture?"* Examples of what to surface:

   - **Ticket tracker (Jira/Linear/Asana)**: do you use individual tickets, epics for projects, both, or neither? Are tickets your unit of work, or is your work primarily in PRs that link back?
   - **Google Calendar**: just meetings, or focus blocks with descriptions of what you worked on? Do you label events? Are demos/presentations in specific event series?
   - **GitHub**: PRs you authored, code reviews you did, both? Repo allowlist or all repos?
   - **Slack**: which channels are work channels worth scanning? Do you announce shipped work in any specific channel? **Are there dedicated kudos / shout-out channels** (e.g., `#kudos`, `#dse-kudos`, `#team-wins`)? Capture those separately for shout-out scanning — every message in those channels mentioning you is a candidate. Stored as `kudos_channels` in the source filter.
   - **Notion/Confluence/Google Docs**: which workspaces / parent pages contain work artifacts? RFCs, design docs, weekly notes?
   - **Preset/Tableau/Looker**: which workspaces/folders contain dashboards you build vs throwaway ad-hoc charts?

   Store the answers in `sources[].workflow_notes` (free-form string). These notes are passed into candidate filtering on every run so the skill can interpret signals correctly for the user's workflow (e.g., "calendar focus blocks have detailed work descriptions — treat them as primary, not just metadata").

   Also collect any structured filters (`author=me`, `channels`, `keyword list`, `repo allowlist`, `workspace IDs`). See [REFERENCE.md](REFERENCE.md) → "Source-specific inference hints" for what each source type can yield.
3. Ask about optional sync targets. List available MCPs that could be destinations (Notion, Google Docs, etc.). For each, ask for the target identifier (page ID, doc ID). **Warn the user explicitly**: "Sync sends full entry content including raw source snippets to <target>. Continue?" — entry content may include real names, ticket IDs, and other data they may not want in the cloud target.
4. Ask about cron: schedule (default `0 9 * * 1-5` — weekdays at 9am) and default window (default `24h`).
5. Ask for the time zone to use for day-grouping. Default: local TZ. Sources report in mixed TZs (GitHub UTC, calendar local) — the skill normalizes to this TZ when bucketing entries to dates. Matches "what did I do today" intuition.
6. Ask for **any other instructions** — free-form preferences the user wants applied to every run. Examples to surface: "focus on technical leadership work", "exclude minor bug fixes from CI flakes", "prioritize cross-team coordination", "always include security-related work", "skip anything tagged `chore`". Stored as `user_instructions` in config and applied at candidate-filtering time (the skill checks each candidate against these instructions before including it).
7. Write two files:
   - `<folder>/config.json` — full schema in [REFERENCE.md](REFERENCE.md) → "Config".
   - `~/.brag-doc-prep/path` — one-line pointer file containing the absolute path to `<folder>/config.json`.

   All derived paths come from `folder` + filename conventions in config (no need for absolute paths). Set `user_instructions` to `""` if the user has no extra preferences.

8. Offer to help install the cron entry. Don't silently edit `crontab` / `launchd` / Task Scheduler — always require explicit confirmation. Flow:

   a. Detect platform (macOS / Linux / Windows) and suggest the best mechanism (`launchd` on macOS, `crontab` on Linux, Task Scheduler on Windows). See [REFERENCE.md](REFERENCE.md) for per-platform details.

   b. Show the exact command, file, or task definition the user would create. Example for Linux:

      ```
      0 9 * * 1-5 BRAG_DOC_PREP_MODE=cron <claude-invocation> >> ~/.brag-doc-prep/cron.log 2>&1
      ```

   c. Ask: install now, save the snippet for the user to install manually, or skip cron setup. If "install now", run the install command with the user confirming each step — never edit scheduler config without explicit go-ahead.

   d. After install, verify: read the scheduler back (`crontab -l`, `launchctl list`, `schtasks /query`) and confirm the entry is registered. Suggest scheduling a test run a few minutes out (see [REFERENCE.md](REFERENCE.md) → "Verifying the first cron run") before relying on it.

To re-run the wizard later: user says "reconfigure brag doc" → overwrite the config file. They can also edit the JSON directly.

## Run flow

### Mode detection

The `BRAG_DOC_PREP_MODE` environment variable is **authoritative** for interactivity. Check it first, before anything else.

- **`BRAG_DOC_PREP_MODE=cron` set** → **cron mode**. This is *non-negotiable* — never prompt, never ask for confirmation, never run the wizard. Treat the user as absent. Even if the user supplied a custom timeframe in the prompt (e.g., "for yesterday", "last 24 hours"), respect the timeframe but suppress ALL prompts. Write directly to the primary brag doc, append-only. No ad-hoc file. No promote-to-main confirmations. No post-run reflection. If you would normally ask the user something, instead log the question/decision in `.brag-log.jsonl` (`silent_decisions: [...]`) and proceed with sensible defaults.
- **Env var unset or any other value** → **on-demand mode**. Interactive. Parse the user's timeframe from the prompt. Write to a new ad-hoc file. Prompts allowed per `confirm_before_writing` config. Promote-to-main and post-run reflection happen here.

**Common bug to avoid**: do not let "for yesterday" or any natural-language timeframe in the prompt override the env-var-based mode detection. The env var wins. If `BRAG_DOC_PREP_MODE=cron` is set, the run is silent regardless of what the prompt asks for.

Then:

1. Read config. If missing, run setup wizard first.
2. Resolve the timeframe to a list of dates (most recent first). **Adaptive window for cron mode**: instead of always using `cron.window` (e.g., 24h), read the most recent successful run from `.brag-log.jsonl` and start the scan from that run's end-time. If the gap from the last run is larger than `cron.window` (e.g., a weekend, a PTO week), the window auto-widens to cover everything since. This means a Monday 4:30pm cron after a Fri 4:30pm cron scans ~72h, not 24h — closing the weekend gap without manual backfills. If there's no prior successful run logged, fall back to `cron.window`. In on-demand mode, the timeframe comes from the user's prompt (no adaptation).
3. For each date, run a **per-day scan** (see below).
4. After all days, log the run to `.brag-log.jsonl` (next to the brag doc).
5. Sync to each `sync_to` target. If a sync fails, log the failure and warn the user but don't roll back the local write.
6. **Post-run reflection** (interactive mode only — skip when `BRAG_DOC_PREP_MODE=cron`). Trigger conditions:
   - **First successful run** — always reflect after the first run, even if no issues.
   - **Errors occurred** — source failed, sync failed, parsing issue, etc.
   - **Pattern detected** — high user-rejection rate for a source (>50% of candidates rejected), repeated complaints in user feedback ("this isn't what I meant"), or unused source (consistently 0 candidates over multiple runs).

   When triggered, surface the issue/observation and ask via a structured prompt whether to update config. Examples:
   - *"Source `notion` failed twice this run — should I disable it, change filters, or leave as-is?"*
   - *"You rejected 8 of 10 calendar candidates this run. Want to refine the calendar workflow notes / keywords?"*
   - *"First-run summary: 5 entries captured across GitHub + Jira. Anything to refine before next run?"*

   Apply accepted changes to `config.json`. Log the reflection outcome in `.brag-log.jsonl` (`reflection: {prompted: ..., changes_applied: ...}`).

### Per-day scan

The atomic unit of work — same logic for cron (N=1) and ad-hoc (N=many).

For one date:

1. For each enabled source, query items in that date's window. Tolerate source-specific failures — log and continue with remaining sources.
2. Build candidate entries. Each candidate has a **source ID** (see below).
3. **Dedup**: classify each candidate against the primary brag doc by source ID:
   - **Not present** → new entry candidate.
   - **Present, sparse** (existing entry has multiple `—` fields, the candidate has richer values for those fields) → update candidate.
   - **Present, equivalent** → skip.
   Ad-hoc files are not consulted for dedup; only the primary brag doc.
4. **If 0 candidates after filtering**, run an **absence-context scan** before declaring the day quiet:
   - Calendar: look for full-day events with keywords (`PTO`, `vacation`, `OOO`, `out of office`, `holiday`, `off`, `team day`, `off-site`, `conference`, `sick`).
   - Calendar free/busy: events explicitly marked "out of office".
   - Ticket trackers: PTO tickets in the user's name if the team tracks time that way.

   If a reason is found, emit **one** entry with `Type: away`, brief `Impact` (`"PTO"`, `"Public holiday — Canada Day"`, `"Team off-site"`, `"Conference: React Conf"`), and the calendar event as `Evidence`. Most other fields stay `—`. This gives downstream skills (especially `brag-perf-review`) the context to explain gaps rather than reading them as low productivity.

   If no reason is found, the day stays absent from the doc (current behavior — no entry).
5. If `confirm_before_writing: true` and in on-demand mode, show the user the day's candidates grouped as new / update / skip / away-context. Let them accept all, skip all, or decide individually.
6. Write accepted entries to the destination file (see "Output destinations" below).

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
| `Type` | Taxonomy for filtering | Ticket type (`Story` → shipped/built, `Bug` → fixed, `Spike` → learned, etc.); PR merged → shipped; calendar "demo" → presented; @-mention with kudos signal → recognized; absence-context scan → away. Default values: `shipped`, `led`, `decided`, `fixed`, `learned`, `mentored`, `presented`, `recognized`, `away`. Users can add custom types. | `—` |
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

Stable, greppable identifiers used for dedup. Format: `<source-type>:<unique-key>` (e.g. `github:owner/repo#1234`, `jira:PROJ-1234`). If a source's natural ID isn't stable, fall back to a hash of (date + title + first evidence URL). Full list of source-ID formats in [REFERENCE.md](REFERENCE.md) → "Source-specific inference hints".

## Output destinations

**Primary brag doc** — the canonical record. Always lives at `<folder>/brag-doc.md`.

**Ad-hoc file** (on-demand only) — a scratchpad artifact for the specific query (e.g., "last 2 weeks for perf review"). Filename described below.

### Write rules per mode

**Cron mode**: append new entries to the primary brag doc. Append-only — never modifies existing entries.

**On-demand mode**: writes to *both* the ad-hoc scratchpad *and* the primary brag doc, with user confirmation:

1. Write all accepted entries to the ad-hoc file (the artifact for this query).
2. For each entry not already in the primary doc by source ID, ask: *"Add to main brag doc?"* — default yes. This is the gap-fill mechanism for days cron missed.
3. For each entry whose source ID is in the primary doc but the existing entry is sparse (multiple `—` fields where the ad-hoc has richer values), ask: *"Update main entry with richer info from this run?"* — show a diff. Default yes.
4. Record promotions in `.brag-log.jsonl` (`promoted_to_main: [<source_id>, ...]`, `updated_in_main: [<source_id>, ...]`).

The primary brag doc thus becomes the canonical "everything I've done" record — cron handles the steady state, ad-hoc backfills patch gaps and improve sparse entries.

**Sync targets (optional)**: each entry in `sync_to` is best-effort. Failures don't block the local write.

Supported sync target types:

- `notion` — appends each entry as a block to a Notion page. Required: `page_id`.
- `google-docs` — appends entries to a Google Doc. Required: `doc_id`.
- `slack` — posts to a Slack DM or channel. Required: `target` (`@me` for self-DM, `@username` for another user's DM — discouraged, or `#channel` for a channel). Optional: `format`:
  - `summary` (default) — one line per day in the form `<YYYY-MM-DD>: <prose paragraph synthesizing the day's accomplishments, with supporting links inline>`. **Format as flowing prose**, not a list. Combine the day's entries into 1–3 sentences that read naturally. Drop semicolon-joined lists in favor of connecting sentences. Example:
    > `2026-05-23: Shipped the new onboarding flow ([PR #1234](url)), which cut activation time from 4 days to 8 hours. Also fixed an auth race condition that was occasionally locking out new sessions ([JIRA-456](url)), and mentored a junior engineer on testing patterns during code review.`

    For a multi-day backfill, post one such line per day, newest first. If a day had only one entry, the summary is one sentence; if many, allow 2–3 sentences but keep it scannable.
  - `full` — each entry posted as a separate message with all fields (sensitive — see safety rules below).

**Slack sync safety rules**:
- Default `format` is `summary` — full entry content can include sensitive context (metrics, stakeholder roles, raw source bodies).
- For `target`: `@me` (self-DM) is the only safe default. Other DMs or channels require explicit user confirmation during wizard setup, and a warning at every cron run: *"Brag entries will be posted to <target>. Continue?"*
- Never post to a channel without explicit channel allowlist (same rule as messaging *sources*).

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
- **Messaging platforms: filter at the content level, not the channel level.** Scope can be broad (any channel/DM the user has access to), but **only include messages that signal a work accomplishment**. Explicit excludes: personal venting, complaints about coworkers, social chatter (birthdays, lunch), logistics, expressions of frustration. The line: a message describes *work product, decisions, learnings, or outcomes*, not feelings or social activity. When unsure, exclude — false negatives are fine, false positives are not (a misclassified vent in a brag doc is embarrassing). Never include messages authored by anyone other than the user.
- **Cron + ad-hoc concurrent runs**: if both fire simultaneously, dedup by source ID prevents duplicate entries in the primary file. Ad-hoc files are independent so they don't compete. A `.brag-doc-prep.lock` next to the doc is a nice-to-have, not required.
- **Don't silently edit the user's crontab/launchd/Task Scheduler.** Print the suggested line; the user installs it.
- **First cron tick on a fresh config**: stick to the configured window (default 24h). Don't auto-backfill — that's what ad-hoc mode is for. Suggest the user run an ad-hoc backfill if they want history.
- **Source failures are isolated**: one broken MCP shouldn't kill the whole run. Log the failure and continue with the remaining sources.
- **Confirm-then-edit UX**: when prompting per-day, support "accept all", "skip day", "edit this entry", "drop this entry". Per-day chunking keeps each prompt manageable.
