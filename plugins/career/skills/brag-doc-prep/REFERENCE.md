# Brag Doc Prep — Reference

Supplementary details for the `brag-doc-prep` skill. SKILL.md is the entrypoint; load this when the user needs to wire up the cron job, debug headless runs, or extend the skill with a new source/sync type. See [README.md](README.md) for a visual overview.

## Running headlessly (cron, launchd, Task Scheduler)

The cron path invokes Claude Code non-interactively. This is supported but has a few setup concerns that don't apply to normal interactive use.

### Invocation

Use Claude Code's print/headless mode. Exact flag depends on your Claude Code version — check `claude --help`. The common patterns:

```bash
# Print mode (one-shot)
claude -p "Run the brag-doc-prep skill"

# With explicit input
echo "Run the brag-doc-prep skill" | claude -p
```

In all cases, set `BRAG_DOC_PREP_MODE=cron` so the skill suppresses prompts and uses config defaults.

### Authentication in the cron environment

Cron runs with a minimal environment — interactive shell configs (`~/.bashrc`, `~/.zshrc`, `~/.bash_profile`) are **not sourced** by default. If your `ANTHROPIC_API_KEY` (or whichever auth method) only lives in those files, the cron job can't see it.

Options:

1. **Inline in the cron line** — least secure, but simplest for personal-machine use:
   ```
   0 9 * * 1-5 ANTHROPIC_API_KEY=sk-... BRAG_DOC_PREP_MODE=cron claude -p "..."
   ```

2. **Source from a file** — keep the key out of crontab:
   ```
   0 9 * * 1-5 . ~/.claude/env && BRAG_DOC_PREP_MODE=cron claude -p "..."
   ```
   With `~/.claude/env` containing `export ANTHROPIC_API_KEY=sk-...` (chmod 600).

3. **macOS keychain / Linux secret-tool** — most secure, requires a wrapper script that fetches the key at runtime.

### MCP availability

MCP servers configured for interactive Claude Code don't automatically work in headless invocations. Two failure modes to watch for:

- MCP server isn't reachable because no one started it (interactive sessions start MCPs on demand; cron may not).
- MCP server requires OAuth or interactive auth that hasn't been refreshed.

**Mitigation**: before relying on a source in cron, test it once with a manual headless invocation:

```bash
BRAG_DOC_PREP_MODE=cron claude -p "Run the brag-doc-prep skill for today only"
```

If a source fails, the skill logs the failure to `.brag-log.jsonl` and continues with remaining sources — the cron job won't silently produce empty days.

### Output and debugging

Redirect stdout + stderr to a log file so failures are diagnosable after the fact. The cron-install step bakes the user's brag-doc folder path directly into the cron line, so `cron.log` ends up inside that folder:

```
0 9 * * 1-5 ... >> ~/Documents/brag-doc/cron.log 2>&1
```

Rotate `cron.log` occasionally (logrotate, or a periodic `mv` in another cron job) — it can grow.

To diagnose a failing cron job:

1. Read `<brag-doc-prep-folder>/cron.log` — captures shell-level errors (auth missing, command not found).
2. Read `<brag-doc-prep-folder>/.brag-log.jsonl` — captures skill-level info (sources scanned, failures per source).
3. If both are silent, the cron itself isn't firing. Check with `grep CRON /var/log/syslog` (Linux) or `log show --predicate 'process == "cron"' --last 1d` (macOS).

### Platform-specific schedulers

- **macOS**: `cron` works but Apple deprecates it in favor of `launchd`. For new setups, write a `~/Library/LaunchAgents/com.user.brag-doc-prep.plist` with a `StartCalendarInterval`. `launchctl load` it once.
  - **Sleep behavior**: `cron` *misses* jobs scheduled during sleep and never catches up. `launchd` doesn't *wake* the Mac, but **catches up on wake** — if the scheduled time fell during sleep, the job fires when the Mac next wakes. For a job that needs to run at the exact wall-clock time even while asleep, pair `launchd` with `pmset schedule` to wake the Mac shortly before the run. For brag-doc-prep at weekday 9am this is usually unnecessary — Mac is almost always awake by then, and a slight catch-up delay is harmless.
- **Linux**: `crontab -e` is the standard path. `systemd` user timers are the modern alternative if you're already using systemd-user services.
- **Windows**: Task Scheduler. Create a basic task with a trigger (daily 9am, weekdays) and an action (run `claude.exe -p "..."` with the env var set in the task's environment).

### Verifying the first cron run

Don't trust cron until you've seen it work. Schedule the first run a few minutes out:

```
*/5 * * * * BRAG_DOC_PREP_MODE=cron claude -p "..." >> ~/.brag-doc-prep/cron.log 2>&1
```

Watch `cron.log` and the brag doc for ~10 minutes. Once you've seen one good run, change the schedule to your real one (`0 9 * * 1-5`).

## Companion skills (planned)

The rich entry shape exists because `brag-doc-prep` is intended to be the *capture layer* — future skills will be the *output layer*. Documenting them here keeps the field set honest: if a field doesn't earn its keep across these consumers, drop it.

All consumer skills should output, per item: a **short summary line** (1–2 sentences) plus **supporting links** carried verbatim from the entry's `Evidence` field (PRs, tickets, Notion pages, etc.). Never strip the source links — they're the proof and the path back to context.

- **`brag-summary`** — group entries by `Project` over a window. Output per project: short summary of what shipped + supporting links (PR #s, ticket IDs, doc URLs) + timeline + outcome. Reads: `Project`, `Status`, `Date`, `Impact`, `Metrics`, `Evidence`.
- **`brag-to-cv`** — pull achievements suitable for a resume bullet. Output per achievement: metric-grounded one-liner + supporting links as references. Reads: `Type` (shipped/led/decided), `Role` (lead/initiator over contributor), `Skills`, `Metrics`, `Impact`, `Evidence`.
- **`brag-perf-review`** — draft a perf-review self-review for a half/quarter. Output: structured sections by competency, each claim backed by entry summary + supporting links. Reads: `Skills`, `Role`, `Stakeholders`, `Impact`, `Metrics`, `Evidence`, plus raw context for tone.
- **`brag-weekly-recap`** — quick "what did I do this week" summary for a team update or 1:1. Output per item: short summary + supporting links. Reads: `Date`, `Title`, `Impact`, `Project`, `Evidence`.

None of these exist yet — when adding one, validate the field set against its actual needs. If a consumer never reads a field, the field probably shouldn't be required at capture time.

## Source-specific inference hints

What each source type can plausibly yield. Use this when implementing the per-day scan for a given source. If a signal isn't reliably present, leave the field `—` rather than guess.

### Source retention reference

What history each source typically holds. Used by the retention pre-flight check in the run flow to warn users when a requested window exceeds what a source can return.

| Source | Typical retention | Notes |
|--------|------------------|-------|
| Slack | 90 days (Free / many corp policies) | Hard cap. Cron captures continuously; backfills > 90d are incomplete. |
| Notion (edit history) | 7d (Free) / 30d (Plus) / 90d (Business) / 1yr (Enterprise) | Current page state is always available; only edit-history queries are bounded. |
| Gmail | Workspace-defined (often 1–3 years corp; indefinite personal) | Treat older mail as potentially missing. |
| Google Calendar | Generally indefinite; some corp policies cap at 1–3 years | Deleted recurring series lose instance history. |
| Jira / Linear / Asana | Indefinite on paid plans | Archived projects may restrict access. |
| Confluence | Indefinite on paid plans | Rarely an issue. |
| GitHub / GitLab / Bitbucket | Indefinite (commits, PRs, issues) | Deleted repos lose everything. |
| Preset / Tableau / Looker | As long as the artifact exists | Deleted dashboards lose everything. |

**Behavior when a window exceeds retention**:

- **Cron mode**: log silently in `.brag-log.jsonl` as `source_retention_gaps: [{ "source": "<x>", "missing_before": "<date>" }]`. Don't block the run.
- **On-demand mode**: warn the user before scanning: *"\<source\> typically retains \<window\>. Entries from before \<date\> won't be captured from this source."* Then proceed.

**Other operational limits worth knowing** (not retention but related):

- **GitHub API rate limits**: 5000 req/hr authenticated. A 90-day backfill across many repos can hit this. MCPs typically paginate + cache; only a concern for very large backfills.
- **Archived projects/repos**: source IDs may 404 when fetching evidence. Treat as `source_failures` and continue.
- **Deleted records**: source IDs remain in the brag doc (dedup still works), but `Evidence` links may 404. Acceptable trade-off — the doc remembers the work happened.

### Source-ID formats (dedup keys)

| Source type | Format |
|-------------|--------|
| GitHub PR | `github:owner/repo#1234` |
| GitHub issue | `github:owner/repo!1234` |
| Ticket tracker | `<tracker>:<ticket-id>` (e.g. `linear:ENG-5678`, `jira:PROJ-1234`) |
| Calendar event | `calendar:<event-id>` |
| Slack message | `slack:<workspace>:<channel>:<ts>` |
| Notion page edit | `notion:<page-id>` (revision ID if available) |
| Google Doc edit | `gdoc:<doc-id>:<revision>` |
| Dashboard/chart | `preset:<workspace>:<chart-or-dashboard-id>` |
| Manual entry | `manual:<short-slug>-<date>` |

Fallback when no stable natural ID: hash of (date + title + first evidence URL).

### Code hosts (GitHub, GitLab, Bitbucket)
The PR description is the **primary source for entry content**. Read it carefully — most fields below can be inferred from it.

Capture two categories of activity:

1. **PRs you authored** (most entries).
2. **Code reviews you performed on others' PRs** — but only *substantive* ones (see filter below).

For PRs you authored:

- **Title** → PR title (cleaned of conventional prefixes like `feat:`, `fix:`, `[ABC-123]`).
- **Impact** → from PR description: look for "why", "motivation", "context", or business-value language. Extract the 1-line outcome. If the description has a "Summary" or "Why" section, prefer that.
- **Skills** → from PR description + changed files: technologies mentioned, patterns (refactor, migration, perf optimization), domain (auth, billing, search). Free-form, not a fixed allowlist.
- **Project** → repo name (`owner/repo` → `repo` slug). Fallback if no ticket-tracker source.
- **Type** → `shipped` (PR merged), `fixed` (PR with "fix"/"bug" labels).
- **Role** → `lead` or `contributor` (you authored).
- **Status** → `done` (PR merged), `in-progress` (PR open).
- **Metrics** → +/- LOC, files changed, review approvals. Also pull any numeric claims from the PR description.
- **Evidence** → PR URL, linked issues, linked tickets.
- **Raw context** → full PR description body, verbatim.

For code reviews (PRs you reviewed, did not author):

Capture **all reviews** — even `lgtm`-only ones. Review volume is itself a signal (e.g., "reviewed 40 PRs this quarter, including 12 substantive design reviews"). Downstream skills can group/filter by substance using the `Impact` field.

- **Title** → `Reviewed: <PR title>` (cleaned).
- **Impact** → describe what kind of review it was. Examples: `"Approved without comment"` (lgtm-only), `"Approved with minor stylistic comments"`, `"Requested changes — caught race condition before merge"`, `"Design discussion — guided cross-team API choice"`, `"Mentored junior eng through testing patterns"`. Honest assessment, not inflation.
- **Type** → `reviewed`, or `mentored` if teaching-heavy (multiple explanatory comments for a junior author).
- **Role** → `reviewer`.
- **Stakeholders** → PR author role + count (e.g., "junior eng (1)") — never name them.
- **Skills** → technical areas the review touched on (free-form; `—` if `lgtm`-only).
- **Evidence** → PR URL + permalink to your most substantive comment if any.
- **Raw context** → your review comments verbatim (not the PR description). Empty if `lgtm`-only.

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
- **Type** → infer from keywords: `presented` (demo, present, showcase), `mentored` (1:1 mentee, coaching), `decided` (decision review, RFC review), `away` (PTO/holiday/off-site — see "Absence-context scan" below).
- **Role** → `initiator` (organizer), `contributor` (attendee), `presenter` (explicit role in event title).
- **Stakeholders** → attendee count + roles (never names).
- **Evidence** → calendar event URL, attached docs.

#### Absence-context scan

Triggered when a per-day scan returns 0 candidates. Calendar is the primary source for absence context. Signals (in priority order):

- Full-day events with keywords: `PTO`, `vacation`, `OOO`, `out of office`, `holiday`, `off`, `team day`, `off-site`, `conference`, `sick`, `personal day`.
- Events explicitly marked free/busy = "out of office".
- Recurring company holidays (if the calendar has a "Holidays in <country>" subscription).

When a signal is found, emit one entry per day with:
- **Type**: `away`
- **Impact**: short reason — `"PTO"`, `"Public holiday — Canada Day"`, `"Team off-site"`, `"Conference: React Conf"`. Pull the human-readable phrasing from the event title.
- **Evidence**: link to the triggering calendar event.
- All other fields: `—`.

If multiple signals overlap (e.g., a PTO + a team off-site on the same day), prefer the more specific one (off-site over PTO). Don't emit multiple `away` entries for the same day.

### Messaging (Slack, Teams)
- **Default rule**: filter to messages *you posted*. Don't pull others' messages — privacy + signal-to-noise.
- **Exception — shout-outs**: pull others' messages when they explicitly recognize the user (see "Shout-out filter" below). This is the one allowed case of capturing others' content.
- **Scope**: broad — any channel/DM the user has access to. The filter happens at the content level (below). Optional `channels_blocklist` in config to always exclude specific channels. Separate `kudos_channels` allowlist for dedicated shout-out channels (treated specially — see below).
- **Retention limit (important)**: Slack workspaces commonly retain only **the last 90 days** of message history (Free tier default + many corporate retention policies). Beyond that window, Slack returns nothing — not because there's nothing to find, but because the data is gone.
  - **Implication for cron**: not a problem. Daily runs always look at the last 24-72h, well within retention.
  - **Implication for ad-hoc backfills**: requesting a window > 90 days returns incomplete data from Slack. Other sources (Jira, GitHub, Notion) typically have longer retention and still work.
  - **Skill behavior**: when an ad-hoc window exceeds 90 days and Slack is an enabled source, **warn the user explicitly**: *"Slack data beyond 90 days isn't retained — entries from before \<date\> won't be captured from Slack. Other sources will still scan the full window."* Log the limitation in `.brag-log.jsonl` (`source_retention_gaps: [{ "source": "slack", "missing_before": "<date>" }]`).
  - **Mitigation**: rely on cron to capture Slack data continuously; don't expect ad-hoc backfills to recover historical Slack content beyond the retention window.
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

### Shout-out filter (cross-source)

A separate capture category — messages from others that explicitly recognize the user. Yields entries with `Type: recognized`.

Detection rules (in priority order):

1. **Dedicated kudos channels** (Slack `kudos_channels` config): every message in these channels that @-mentions the user OR contains their name is a candidate. These channels exist *for* recognition — high signal, low noise.
2. **General channels / PR comments / ticket comments**: opportunistic detection. Candidate must have BOTH:
   - A direct mention of the user (@-handle or name).
   - At least one recognition signal: "thanks for", "shoutout to", "kudos", "great work", "amazing job", "really appreciated", "well done", "huge thanks", "saved the day", etc.

   Filter out low-substance signals: "thanks!" alone in a transactional reply doesn't count. The recognition must be about specific work or impact.

Entry shape for shout-outs:

- **Title** → `Recognized for <topic>` synthesized from the message body (e.g., "Recognized for the onboarding launch", "Kudos for incident response on auth outage").
- **Impact** → the recognition itself, in 1 line (e.g., "EM publicly thanked me for unblocking the team on auth migration").
- **Type** → `recognized`.
- **Role** → best-guess of your role in the work being recognized; `—` if no specific work is referenced.
- **Stakeholders** → role of the person recognizing you, anonymized (`EM`, `PM`, `tech lead`, `senior eng`, `team`). Never their name. Count if multiple people piled on.
- **Skills** → if the recognition mentions specific technical or soft skills demonstrated, capture them.
- **Source** → the channel/PR/ticket where the shout-out happened.
- **Evidence** → permalink to the message.
- **Raw context** → verbatim recognition message(s).

Skip rules:
- Generic team-wide acknowledgements not directed at you ("thanks everyone for a great quarter").
- Reactions/emojis without text recognition (a 🎉 isn't a brag-doc entry).
- Recognition you already gave yourself (your own shipping announcement isn't a shout-out).

### Docs / wikis (Notion, Confluence, Google Docs)
- **Project** → parent page / workspace section.
- **Type** → `learned` (research/exploration docs), `decided` (RFC, ADR, design doc), `presented` (slide decks).
- **Role** → `lead` (author of substantial edits — measure by diff size if available), `contributor` (minor edits only — usually skip).
- **Status** → `done` (published/locked), `in-progress` (draft).
- **Evidence** → doc URL, linked discussions.

### Data / analytics (Preset, Tableau, Looker, Metabase)
- **Title** → chart/dashboard/dataset name.
- **Impact** → from dashboard description or commit message; otherwise synthesize from chart titles + dataset purpose. Numeric outcomes if mentioned ("powers daily exec review", "drives Q2 OKR tracking").
- **Skills** → SQL, dimensional modeling, viz design, data domain (finance, growth, ops). Visualization library if specified.
- **Project** → workspace / folder / dashboard-group name.
- **Type** → `built` (new chart/dashboard/dataset), `shipped` (published to a team-visible workspace), `decided` (when adding a metric that codifies a business definition).
- **Role** → `lead`/`contributor` (creator), `reviewer` (commenter on someone else's chart).
- **Status** → `done` (published), `in-progress` (draft / private workspace).
- **Stakeholders** → owning team / consuming team if known from dashboard metadata.
- **Metrics** → number of charts in a new dashboard, number of unique consumers if known, dataset row count if architecturally significant.
- **Evidence** → chart URL, dashboard URL, dataset URL.
- **Raw context** → dashboard description + chart titles list.
- **Filter**: skip ad-hoc one-off charts that won't survive a week. Only include work likely to be referenced again (dashboards, named metrics, shared datasets). Heuristic: was it shared, named, or added to a team workspace? If no, probably skip.

### Cross-source enrichment
When the same accomplishment appears in multiple sources (PR linked to a ticket linked to a calendar demo), **merge inference**: pick the highest-confidence value per field. Ticket-tracker fields usually win for `Project` and `Status`; code-host fields win for `Metrics` (LOC); calendar wins for `Type: presented`. Source ID should be the *primary* source (the one the user is most likely to remember) — usually the ticket if present, else the PR.

## Vendor defaults

When the wizard detects a known vendor MCP, it pre-fills suggestions from this table instead of asking everything cold. The user can override any default. Add new entries here when introducing new vendors.

### GitHub
- **Default capture**: PRs authored by user + all code reviews (review volume is itself a signal; substance is captured in `Impact`).
- **Default repo filter**: all repos the user has activity in. Ask if they want to restrict.
- **Default workflow note** (proposed to user): *"PRs are my primary unit of work; capture both authored PRs and all reviews."*

### Ticket trackers (Jira, Linear, Asana)
- **Default capture**: any ticket activity by the user in the window — tickets created, status changes (`To Do` → `In Progress` → `Done`/`Closed`/`Shipped`), assignment changes, substantive comments. Movement is signal; ticket lifecycle tells the story of work.
- **Default Project inference**: epic name (best signal). Fallback to ticket title.
- **Default workflow question**: *"Are tickets your unit of work, or are you mostly PR-driven with tickets as context? Should I capture tickets you reported but didn't work on?"*
- **Default workflow note** (proposed): *"Tickets group work via epics; capture all activity (creation, status transitions, substantive comments) since ticket movement signals progress."*

### Google Calendar
- **Default capture**: events the user **accepted** (RSVP = yes). Skip declined, tentative, and no-response events — those weren't actually attended.
- **Capture priority depends on usage** (the wizard's question below sets this):
  - **Primary time management tool** — user blocks focus time with descriptions of work, time-blocks for tasks, uses calendar as their daily plan. Treat as **high-signal source**: scan all accepted events for accomplishment content, mine focus-block descriptions as primary text.
  - **Mostly just meetings** — user doesn't time-block, calendar is meetings + appointments. Treat as **low-signal source**: only capture events matching strong keywords (`demo`, `launch`, `present`, `showcase`, `kickoff`, `retro`) and 1:1s with mentorship signals.
- **Default capture keywords** (for low-signal mode): `shipped`, `launched`, `demo`, `presented`, `1:1`, `review`, `kickoff`.
- **Absence detection**: always enabled (PTO/holiday/OOO/team-day) — see "Absence-context scan" above. Doesn't depend on the primary-vs-meetings classification.
- **Default workflow question**: *"Is your calendar your primary time management tool — do you block focus time with descriptions of what you're working on, time-block for tasks? Or is it mostly just meetings? This sets how heavily I weight calendar as an accomplishment source."*

### Slack / Teams
- **Default scan scope**: all public channels the user is in.
- **Default DM scope**: include the user's own messages; never others'. **Shout-out exception** (see "Shout-out filter" above): allow others' messages that directly recognize the user.
- **Default `kudos_channels`**: empty — explicitly ask: *"Do you have any dedicated kudos / shout-out channels (e.g., `#kudos`, `#team-wins`)?"*
- **Default `channels_blocklist`**: empty — the user can add channels they want to exclude.
- **Default workflow question**: *"Do you announce shipped work in any specific channel? Are there channels worth always excluding (social, off-topic)?"*
- **Retention awareness**: surface during setup: *"Slack often retains only the last 90 days. Cron will capture continuously; ad-hoc backfills beyond 90 days won't recover historical Slack data."* See "Retention limit" in the Messaging inference section above.

### Notion / Confluence / Google Docs
- **Default capture**: pages the user authored or substantially edited (heuristic: ≥ 50% of diff is theirs, or they're the page creator).
- **Default workflow question**: *"Which workspaces / parent pages contain work-related artifacts (RFCs, design docs, weekly notes)? Which are personal and should be excluded?"*
- **Default skip**: comments-only edits, formatting-only edits, draft scratchpads in personal workspaces.

### Preset / Tableau / Looker / Metabase
- **Default capture**: dashboards + datasets created or substantially modified in the window.
- **Default heuristic** for "substantial": chart is named (not auto-titled), published to a non-personal workspace, or has > 1 consumer.
- **Default workflow question**: *"Which workspaces / folders contain dashboards you want tracked? Which are scratch space?"*

**Wizard usage**: for each enabled source, the wizard shows the defaults above as a structured prompt (*"Apply these defaults for \<vendor\>? Adjust?"*), asks the workflow follow-up question, then saves answers to `sources[].filter` and `sources[].workflow_notes`. New vendor not in this table → wizard asks from scratch; add to this table afterward to make future setups smoother.

## Schema reference

### Config

Lives at `<folder>/config.json` (folder set by user in setup, default `~/Documents/brag-doc/`). A one-line pointer file at `~/.brag-doc-prep/path` records the absolute path to this config so subsequent runs can find it.

```jsonc
{
  "schema_version": 1,
  "folder": "~/Documents/brag-doc/",
  "doc_filename": "brag-doc.md",
  "timezone": "America/Toronto",
  "user_instructions": "Focus on technical leadership and cross-team work. Skip minor flake fixes.",
  "sources": [
    {
      "type": "github",
      "filter": { "author": "me" },
      "workflow_notes": "PRs are my primary unit of work; capture both authored and substantive reviews."
    },
    {
      "type": "calendar",
      "filter": { "keywords": ["shipped", "launched", "demo"] },
      "workflow_notes": "I block focus time with detailed descriptions of what I'm working on — treat focus blocks as primary content, not just metadata."
    },
    {
      "type": "slack",
      "filter": { "channels_blocklist": [] },
      "workflow_notes": "I announce shipped work in #eng-platform; team updates go to #team-foo."
    }
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
- `sources[].workflow_notes`: per-source free-form description of how the user uses that source. Captured during the wizard's per-source follow-up ("how do you use \<source\>?"). Passed into candidate filtering so the skill interprets signals correctly for the user's workflow. Updated by post-run reflection when patterns emerge.

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
  "reflection": {                          // present when post-run reflection was triggered
    "trigger": "first_run",                // or "errors", "pattern_high_rejection", "pattern_unused_source"
    "prompted": "First-run summary. Anything to refine?",
    "changes_applied": ["calendar.workflow_notes updated"]
  },
  "output_file": "brag-doc.md"             // or "brag-doc-adhoc-2026-05-23-last-2-weeks.md" for on-demand
}
```

### Entry shape

See [SKILL.md](SKILL.md) → "Entry shape" for the canonical field list and inference rules.
