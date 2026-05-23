# Jeanne's Claude Second Brain

This is the public-facing subset of my personal claude skills.

These are the skills deemed safe and reusable for public release—general-purpose tools that avoid company-specific data and can be adapted by future me or anyone else. (Skills I don't want to lose 😉)

Installable with:

```text
/plugin marketplace add jeaend/claude-second-brain
```

## Repository layout

- `.claude-plugin/marketplace.json` — marketplace catalog (lists plugins).
- `plugins/<plugin>/` — one folder per plugin, each with its own `.claude-plugin/plugin.json` and `skills/`.
- `CLAUDE.md` — skill shape, writing principles, and the pre-commit sensitive-content checklist.
- `CHANGELOG.md` — tracks public releases.
- `LICENSE` — MIT open-source license.

## Skills

### Overview

| Plugin | Skills | Description |
|--------|--------|-------------|
| **career** | brag-board | Career-oriented skills, brag board, and other professional self-tracking, cv helpers. |

### By plugin

<details>
<summary><strong>career</strong></summary>

| Skill | Description |
|-------|-------------|
| [brag-board](plugins/career/skills/brag-board/SKILL.md) | Captures professional accomplishments to a personal brag doc — runs daily via cron or on-demand for any timeframe. |

</details>

## Adding a new skill

First decide: does the skill fit an **existing plugin**, or does it need a **new plugin**?

- **Existing plugin** — drop the skill into `plugins/<plugin>/skills/<skill-name>/`.
- **New plugin** — scaffold `plugins/<new-plugin>/.claude-plugin/plugin.json` and add an entry to `.claude-plugin/marketplace.json`, then add the skill under `plugins/<new-plugin>/skills/<skill-name>/`.

See [CLAUDE.md](CLAUDE.md) for the full checklist — skill folder shape, `SKILL.md` frontmatter template, and the sensitive-content check to run before committing.


## License

This project is licensed under the MIT License.
