![Moonlight](header.png)

# Moonlight

Ideation-to-evaluation pipeline for Claude. Submit raw ideas, get back ranked, sourced executive summaries — all in your own Notion workspace.

## What it does

Run `/moonlight <your idea>` in Claude. The skill will:

1. Interview you to capture the idea (intake)
2. Research the market live (Reddit, competitors, sizing)
3. Run the idea through 4 expert perspectives (Business Analyst, CTO, CPO, Marketing)
4. Score it (RICE + Opportunity Score with moat, unfair advantage, revenue potential, time-to-revenue, competitive window)
5. Write a 1–2 page executive summary into a ranked Notion database

Every claim is sourced. Verdict is `🟢 Go` / `🟡 Investigate` / `🔴 Kill`.

## Install

```
claude plugin install <this-repo-url>
```

## Prerequisites

- **Notion connector authorized in Claude.** Open Claude → settings → connectors → enable Notion. The skill needs read/write access to the workspace where you want Moonlight to live.
- That's it. No API keys, no template duplication, no manual page setup.

## First run

Run `/moonlight` once with any idea (or just `/moonlight` to set up). On first invocation the skill will:

1. Ask where to create the workspace (paste a Notion page URL as the parent, or `root` for workspace root)
2. Programmatically create:
   - 🌙 **Moonlighter's Den** (root page)
   - 🧰 **Resources** (your skills, time, budget, tech stack — fill this in to improve evaluations)
   - 📋 **Changelog** (log of every change)
   - 📊 **Ideas Database** (inline DB with full schema and three computed formulas)
3. Cache the IDs at `~/.claude/moonlight/config.json` so future runs skip setup

## Configuration

The per-user config is stored at:

- POSIX: `~/.claude/moonlight/config.json`
- Windows: `%USERPROFILE%\.claude\moonlight\config.json`

```json
{
  "schema_version": 1,
  "root_page_id": "...",
  "resources_page_id": "...",
  "changelog_page_id": "...",
  "ideas_db_id": "..."
}
```

To re-run onboarding (e.g., to set up in a different parent page), delete this file. To point Moonlight at an existing workspace you've moved or renamed, edit the IDs directly.

## What gets created in your Notion

| Item | Type | Purpose |
|---|---|---|
| Moonlighter's Den | Page (parent) | Root container |
| Resources | Page | Your constraints — Moonlight reads this on every evaluation |
| Changelog | Page | Append-only log of skill activity |
| Ideas Database | Inline database | 18 properties (15 inputs + 3 computed formulas) |

The Ideas Database includes 3 Notion formula properties: **RICE Score**, **Opportunity Score (/5)**, **Verdict**. They auto-compute from the inputs you (or Claude) write.

## Limits / known issues

- **Formulas may need verification.** The Notion API doesn't expose formula source code, so the formulas in [`skills/moonlight/assets/ideas_db_schema.json`](skills/moonlight/assets/ideas_db_schema.json) are reasonable defaults derived from the skill's spec. Open the database after first run and confirm the formulas match what you want.
- **First-run requires Notion connector.** If you skip the connector authorization, the skill will tell you and stop.
- **Workspace search depends on visibility.** If your Notion workspace has many pages, the skill searches for `"Moonlighter's Den"` to detect existing setups. If you renamed the root page, edit the config directly.

## Troubleshooting

**The skill keeps asking me to set up the workspace.**
Check the config file exists and has all four IDs. If a page was deleted in Notion, the skill will detect it on next fetch and re-onboard.

**Database creation failed on formula properties.**
Some Notion MCP versions don't support formula creation. The skill will fall back to creating the database without formulas and tell you. Add the three formulas manually using the expressions documented in [`skills/moonlight/assets/ideas_db_schema.json`](skills/moonlight/assets/ideas_db_schema.json).

**I want to start over.**
Delete `~/.claude/moonlight/config.json` (or the Windows equivalent) and run `/moonlight` again. The skill will re-discover or re-create the structure. Your existing Notion pages are not deleted.

## Contributing

The skill body is [`skills/moonlight/SKILL.md`](skills/moonlight/SKILL.md). The DB schema is [`skills/moonlight/assets/ideas_db_schema.json`](skills/moonlight/assets/ideas_db_schema.json). Page templates are in [`skills/moonlight/assets/`](skills/moonlight/assets/).

To bump the schema, increment `schema_version` in both the JSON and any logic in SKILL.md, and add migration handling in Phase 0.

## License

MIT.
