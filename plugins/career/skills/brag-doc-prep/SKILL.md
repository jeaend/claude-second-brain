---
name: brag-doc-prep
description: Captures professional accomplishments to a personal brag doc — runs daily via cron or on-demand for any timeframe. Use when the user mentions brag doc, brag board, tracking accomplishments, perf review prep, "log a win", "I shipped X", "what did I accomplish last week/quarter", or wants to back-fill a timeframe of work — even if they don't say "brag-doc-prep" explicitly.
---

# Brag Doc Prep

The **capture layer** for an accomplishments workflow. Scans configured sources (GitHub, ticket trackers, calendar, Slack, etc. — whatever MCPs are available) and appends structured entries to a personal markdown brag doc. Runs daily via cron + on-demand backfills.

## What this produces

A date-grouped markdown brag doc (`## YYYY-MM-DD` per day, 0–5 entries per day). Each entry has a rich, predictable shape (`Project`, `Type`, `Role`, `Status`, `Impact`, `Metrics`, `Skills`, `Stakeholders`, `Evidence`, raw context). Downstream skills (`brag-summary`, `brag-to-cv`, `brag-perf-review`, `brag-weekly-recap`) consume the shape to produce CV bullets, perf-review drafts, etc., without re-fetching original sources.

## When to use

- "log a win" / "add to brag doc" / "update brag doc"
- "I shipped X" / "track this win"
- "what did I accomplish last week / this quarter / since X"
- "draft a perf review from my brag doc"
- "back-fill the last two weeks"
- "set up daily brag tracking"

## Meta-rules

**Mode is set by `BRAG_DOC_PREP_MODE` env var, authoritatively**:

- `=cron` → silent. Never prompt, never run the wizard. If you would prompt, log to `silent_decisions` in `.brag-log.jsonl` and proceed with sensible defaults. The env var wins over any natural-language timeframe in the prompt.
- unset → on-demand, interactive. Wizard if needed; prompts per `confirm_before_writing`.

**Wizard prompts use structured input** (`AskUserQuestion` or equivalent), not conversational text. One question per step.

**Persistent corrections**: when the user corrects something mid-run (rejects a candidate with reason, refines a filter, overrides inference), propose persisting to `config.json` via a structured prompt: *"One-time override, or save to config?"* Default **save** for generalizable corrections (filters, workflow notes, recurring excludes), **one-time** for context-specific. Show the diff before writing. Log to `corrections_applied` in `.brag-log.jsonl`.

## First-run setup

If `~/.brag-doc-prep/path` (pointer) doesn't exist or its target is missing, run the wizard. Five steps:

1. **Folder** — default `~/Documents/brag-doc/`. Holds `brag-doc.md`, `config.json`, `.brag-log.jsonl`, ad-hoc files, `cron.log`. If `brag-doc.md` exists, read it and build the dedup index from existing source IDs.
2. **Sources & sync** — enumerate detected MCPs, present a multi-select prompt to enable, then for each enabled source: pre-fill defaults from [REFERENCE.md](REFERENCE.md) → "Vendor defaults" + ask one workflow follow-up (*"how do you use this source?"*). Store in `sources[].workflow_notes`. Then ask about optional sync targets (Notion, Google Docs, Slack); for each, warn *"Sync sends full entry content; continue?"* before recording.
3. **Schedule, TZ, instructions** — cron schedule (default `0 9 * * 1-5`), `cron.window` (default `24h`), `timezone` (default local), and free-form `user_instructions` (e.g., *"focus on technical leadership, skip flake fixes"*).
4. **Write** — `<folder>/config.json` (full schema in [REFERENCE.md](REFERENCE.md) → "Config") + `~/.brag-doc-prep/path` pointer file.
5. **Cron install** — detect platform, show the snippet, offer **install now** (with explicit confirmation), **save snippet**, or **skip**. After install, verify by reading the scheduler back. Per-platform details in [REFERENCE.md](REFERENCE.md) → "Running headlessly".

Re-run with *"reconfigure brag doc"* (overwrites config).

## Run flow

1. Read config. Run mode detection (meta-rules above).
2. **Resolve timeframe**:
   - **Cron mode**: adaptive window — start from the end-time of the most recent successful run in `.brag-log.jsonl`. If none, fall back to `cron.window`. Auto-widens after weekends / PTO.
   - **On-demand mode**: parse from prompt.
   - Then **retention pre-flight**: check sources against retention limits ([REFERENCE.md](REFERENCE.md) → "Source retention reference"). Warn in on-demand mode; silent log in cron mode (`source_retention_gaps`).
3. **Per-day scan** for each date, latest first (see below).
4. **Log run** to `.brag-log.jsonl`.
5. **Sync** to each `sync_to` target — best-effort, failures don't block local write.
6. **Post-run reflection** (on-demand only). Triggers: first successful run, errors, high rejection rate, unused source. When triggered, ask via structured prompt whether to update config; apply accepted changes; log to `reflection: {...}`.

### Per-day scan

The atomic unit. For one date:

1. Query each enabled source for the date's window. Source failures isolated — log and continue.
2. Build candidates, each with a stable source ID (format: `<source-type>:<unique-key>`; full list in [REFERENCE.md](REFERENCE.md)).
3. **Classify against the primary brag doc**:
   - Not present → **new**.
   - Present but sparse (existing has `—` fields the candidate fills) → **update**.
   - Present and equivalent → **skip**.
4. **If 0 candidates after filtering**, run an absence-context scan: calendar for PTO/holiday/OOO/off-site/conference signals, ticket-tracker PTO entries. If found, emit one entry with `Type: away`, brief `Impact` (`"PTO"`, `"Public holiday — Canada Day"`, `"Team off-site"`), calendar event as `Evidence`. If nothing, day stays absent.
5. **On-demand + `confirm_before_writing`**: show candidates grouped new / update / skip / away-context. User accepts/skips/edits.
6. Write to destinations (next section).

## Entry shape

Markdown with date-grouped sections. Append-only. Every entry includes **all** fields — use `—` for unknowns. Downstream skills treat `—` as null.

Day-grouping uses the configured `timezone`; convert source timestamps before bucketing.

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
| `Project` | Grouping slug for project summaries | Ticket-tracker epic name (best signal). Fallback: repo name / calendar title → slug | `—` |
| `Type` | Taxonomy for filtering | See per-source inference in REFERENCE.md. Values: `shipped`, `led`, `decided`, `fixed`, `learned`, `mentored`, `presented`, `recognized`, `away`. Custom types OK. | `—` |
| `Role` | Distinguishes "I shipped this" from "I helped" — critical for honest CV. Values: `initiator`, `lead`, `contributor`, `reviewer`, `advisor` | Ticket assignee/reporter; PR author/reviewer; calendar role | `—` |
| `Status` | Projects don't end on ship date. Values: `done`, `in-progress`, `ongoing`, `abandoned` | Ticket status field (best signal); else default `done` for shipped items, `in-progress` otherwise | `—` |
| `Impact` | The "so what" in one line | Best summary from source description | `—` |
| `Metrics` | Numeric outcomes for quotability | Extract numbers from source if present | `—` |
| `Skills` | Competencies demonstrated (CV pull) | Free-form; if `skills_allowlist` in config is set, validate against it | `—` |
| `Stakeholders` | Anonymized roles + counts. **Never real names** | Counts from PR reviewers, calendar attendees | `—` |
| `Evidence` | At least one canonical link | Always from source URL | n/a (required) |
| `Tags` | Free-form labels for searching | User-added | `—` |
| `Source` | Stable source ID (dedup key) | Built from source type + ID | n/a (required) |
| `Logged` | When this entry was written + mode | `YYYY-MM-DD (cron|on-demand)` | n/a (required) |
| Raw context | Verbatim source content so future skills can re-summarize without re-fetching. Collapsed via `<details>` so the markdown stays scannable. | Copy source body verbatim | Omit the `<details>` block if source has no body |

**Inference rule**: pick the best-confidence value. If confidence is low (multiple plausible values, no clear signal), prefer `—` over a guess. Downstream skills handle null gracefully; they don't recover from wrong values.

## Output destinations

**Primary brag doc** at `<folder>/brag-doc.md` — the canonical record, append-only.

**Ad-hoc files** (on-demand only) at `<folder>/brag-doc-<gen-date>-<timeframe-slug>.md`. Slug rules: lowercase, hyphenate spaces, strip punctuation. Suffix `-2`, `-3` on collision.

Examples: `brag-doc-2026-05-23-last-2-weeks.md`, `brag-doc-2026-05-23-since-april-1.md`.

**Write rules**:

- **Cron mode**: append new entries to the primary doc.
- **On-demand mode**: write all accepted entries to the ad-hoc file. Then for each entry not in the primary doc, ask *"promote to main?"*; for each sparse-but-present entry, ask *"update with richer info?"* (show diff). Log to `promoted_to_main` / `updated_in_main` in `.brag-log.jsonl`.

**Sync targets** (`sync_to[]`): best-effort; failures don't block local write. Types:

- `notion` — appends per-entry block. Required: `page_id`.
- `google-docs` — appends to a doc. Required: `doc_id`.
- `slack` — DM or channel. Required: `target` (`@me` self-DM is the safe default; channels / other DMs require explicit setup confirmation + cron-time warning). `format`: `summary` (default — one line per day in flowing prose with inline links) or `full` (per-entry; sensitive, opt-in).

Example `summary` format:

> `2026-05-23: Shipped the new onboarding flow ([PR #1234](url)), which cut activation time from 4 days to 8 hours. Also fixed an auth race condition ([JIRA-456](url)), and mentored a junior engineer on testing patterns during code review.`

## Gotchas

- **No company/tool/coworker names baked into the skill.** Source identifiers all come from runtime config.
- **Messaging filter is content-level**: capture only work-accomplishment messages. Skip venting, complaints, social chatter, logistics. When unsure, exclude — false positives in a brag doc are embarrassing. Never include messages authored by anyone other than the user (except shout-outs — see [REFERENCE.md](REFERENCE.md)).
- **Never silently edit `crontab` / `launchd` / Task Scheduler.** Always require explicit confirmation.
- **Source failures isolated**: one broken MCP doesn't kill the run.
