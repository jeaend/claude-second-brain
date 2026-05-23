# brag-board

Captures professional accomplishments to a personal brag doc — runs daily via cron or on-demand for any timeframe.

## How it works

```mermaid
flowchart TD
    Start([Invoke skill]) --> Cfg{Configured?}
    Cfg -->|No| Wiz[Setup wizard:<br/>folder, sources, sync,<br/>cron, TZ, instructions]
    Wiz --> CronInstall{Install cron?}
    CronInstall -->|Yes, w/ confirm| Ready([Ready])
    CronInstall -->|Skip| Ready

    Cfg -->|Yes| Mode{Mode}
    Mode -->|BRAG_BOARD_MODE=cron| C[Cron mode<br/>24h window, no prompts]
    Mode -->|interactive| O[On-demand<br/>user timeframe]

    C --> Loop[For each date,<br/>latest first]
    O --> Loop

    Loop --> Scan[Per-day scan:<br/>query sources →<br/>apply user_instructions →<br/>content filter →<br/>classify vs main doc<br/>new / update / skip]

    Scan --> W{Mode}
    W -->|cron| MainAppend[Append new entries<br/>to main brag doc<br/>append-only]
    W -->|on-demand| Adhoc[Write accepted<br/>to ad-hoc file]
    Adhoc --> Promote{Promote new /<br/>upgrade sparse<br/>to main?}
    Promote -->|user accepts| MainAppend
    Promote -->|user skips| Next
    MainAppend --> Next[Next date]

    Next --> More{More dates?}
    More -->|yes| Scan
    More -->|no| Sync[Sync to targets<br/>Notion / Google Docs / Slack<br/>best-effort]
    Sync --> Log[Append run to<br/>.brag-log.jsonl]
    Log --> Done([Done])
```

## Trigger phrases

- "set up daily brag tracking"
- "update brag doc for last week"
- "I shipped X today, log it"
- "what did I accomplish this quarter"
- "back-fill the last two weeks"

## What it produces

A folder (default `~/Documents/brag-board/`) containing:

- `brag-doc.md` — the canonical record, append-only on cron runs.
- `brag-doc-adhoc-<gen-date>-<timeframe-slug>.md` — scratchpad artifacts for ad-hoc queries (perf review prep, "last 2 weeks", etc.).
- `config.json` — your decisions from the setup wizard.
- `.brag-log.jsonl` — one record per run for audit + resume.
- `cron.log` — cron stdout/stderr (if cron is enabled).

## Files in this skill

- [SKILL.md](SKILL.md) — the skill itself (loaded by Claude when activated).
- [REFERENCE.md](REFERENCE.md) — supplementary notes: headless/cron setup, source-specific inference hints, schema reference, companion-skill placeholders.
