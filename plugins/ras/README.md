# RAS plugin

Skills for analysing RAS / BigQuery data with Claude, the way Stratodata does it: every number
traced to a query, every claim labelled, no guesses dressed up as facts.

## What's inside

| Skill | Invoke as | What it does |
|-------|-----------|--------------|
| `ras-analysis` | `/ras:ras-analysis` | Evidence-grounded analysis protocol and step-by-step workflow for any RAS / BigQuery data question or client-facing deliverable. |

Skills trigger automatically when Claude recognises a matching task (analysing data, building a
report, explaining a metric change). You can also invoke one explicitly with its `/ras:<name>`
command.

## Requirements

This plugin provides the *analysis method*, not the data connection. To analyse your data you also
need your RAS / BigQuery MCP connector active in your Claude environment — see
[`../../docs/CONNECT-RAS-MCP.md`](../../docs/CONNECT-RAS-MCP.md).

## Adding a new skill

Create `skills/<skill-name>/SKILL.md` with YAML frontmatter (`name`, `description`). Optional
supporting files (`references/`, `scripts/`) live in the same folder. Skills are auto-discovered —
no need to list them in `plugin.json`. Bump `version` in `.claude-plugin/plugin.json` so clients
pick up the update.
