# CLAUDE.md

This is a **public** skills marketplace (`jeaend/claude-second-brain`). Treat everything in this repo as world-readable. Skills here must be general-purpose and free of personal or company-specific content.

## Repository layout

- `.claude-plugin/manifest.json` — marketplace manifest.
- `skills/<skill-name>/` — one folder per skill.
- `CHANGELOG.md` — bumped per public release.
- `LICENSE` — MIT.

## Skill shape

Each skill lives at `skills/<skill-name>/`. Naming: lowercase, hyphen-separated (`meeting-notes`, not `MeetingNotes`).

Files:

- `SKILL.md` — **required**. The skill itself.
- `REFERENCE.md` — *optional*. Deeper notes, examples, links.
- `scripts/` — *optional*. Helper scripts if the skill needs them.

`SKILL.md` is YAML frontmatter + markdown body:

```markdown
---
name: skill-name
description: What this skill does. Use when the user mentions <trigger phrase 1>, <trigger phrase 2>, <trigger phrase 3>, or asks to <related task> — even if they don't say "<skill-name>" explicitly.
---

# Skill Name

One- or two-sentence blurb on what this is for.

## When to use

Concrete trigger phrases the user would actually type. List them — Claude under-triggers by default, so be generous and specific.

Example for a `meeting-notes` skill:
- "summarize this Zoom transcript"
- "pull action items out of these notes"
- "give me a recap of the meeting"
- "what did we decide in standup"

## Steps

1. Imperative steps.
2. Be specific where it matters (exact commands, flags).
3. Be loose where multiple approaches work — explain why, let Claude choose.

## Output

What success looks like. Concrete input → output example.

## Gotchas

Project-specific edge cases. Skip the section if there aren't any.
```

### Writing principles

- **Spend context wisely.** Every line loaded competes with conversation. Cut anything Claude already knows.
- **Teach procedures, not one-shot answers.** Reusable method > specific solution.
- **Explain why instead of MUSTs.** A reason carries over to edge cases; a rule doesn't.
- **Default, don't menu.** Pick one approach; mention alternatives briefly.

## When adding a new skill — checklist

1. Create the folder: `skills/<skill-name>/SKILL.md` (plus optional `REFERENCE.md` and `scripts/`).
2. Write the skill using the shape above. Make the `description` pushy with explicit trigger phrases.
3. Append a row to the skills table in [README.md](README.md) (`Skill | Description`). Remove the `_No skills yet_` placeholder row when the first real skill is added.
4. Bump `CHANGELOG.md` with a one-line entry under the next version.
5. Run the sensitive-content check below before staging anything.

## Before staging any change — sensitive-content check

This repo is **public**. Before staging or committing any new or modified file, scan it for the following and either rewrite with generic placeholders or move the skill out of this repo entirely:

- **Company-specific content** — internal company names, internal product names, internal team or project names, internal tickets (e.g. `ABC-1234`-style), internal Slack channels, internal Linear/Jira/Notion identifiers, internal repo names, internal tool names.
- **Internal infrastructure** — `.internal` hostnames, internal IPs, internal dashboard URLs, internal VPN/SSO endpoints, internal database names, internal schema names, dev schemas tied to an individual.
- **Personal data** — real names of coworkers or clients, email addresses, phone numbers, customer or account identifiers, employee IDs.
- **Secrets** — API keys, tokens, OAuth client secrets, `.env` contents, credential files, private keys.
- **Real production data** — actual cohort IDs, segment names tied to business logic, sample rows that originated from production, real query results.

Generic placeholders are fine: `<your-database>`, `your_schema`, `example-project`, `team@example.com`.

If a skill cannot be cleanly genericized, it does not belong in this repo — it belongs in a private skills location (`~/.claude/skills/` or a private repo). When in doubt, flag it and ask before committing.

## References

- [Agent Skills specification](https://agentskills.io/specification) — the authoritative format spec (frontmatter fields, directory structure, progressive disclosure).
- [agentskills.io best practices](https://agentskills.io/skill-creation/best-practices) — patterns for writing skills that activate reliably.
- [Anthropic `skill-creator`](https://github.com/anthropics/skills/blob/main/skills/skill-creator/SKILL.md) — Anthropic's own scaffolding skill, including a systematic description optimizer.
- [Anthropic skills repo](https://github.com/anthropics/skills) — reference implementations from Anthropic.
