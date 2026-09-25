# Stratodata Skills for Claude

This repository is a **Claude plugin marketplace**. It packages Stratodata's analytics know-how as
installable skills that run inside your own Claude environment. Once installed, Claude follows
Stratodata's methods automatically when you ask it to analyse data or build a report.

The repository is private. You reach it because Stratodata invited your GitHub account.

## What you get

| Plugin | Skills | Purpose |
|--------|--------|---------|
| `ras` | `ras-analysis`, `ras-oos-cost`, `ras-oos-risk`, `ras-scorecard`, `ras-skills-docs` | Evidence-grounded analysis of RAS / BigQuery data — every number traced to a query, every claim labelled verified / inferred / unknown. Includes out-of-stock cost/risk estimation, a recurring KPI scorecard artifact, and a skill-discovery guide. |

More plugins and skills are added over time. Updating the marketplace (below) pulls the latest.

## Quick start

1. Accept the GitHub invitation from **Stratodata** (check your email or https://github.com/notifications).
2. In Claude Code, add this marketplace and install the plugin:

   ```
   /plugin marketplace add stratodata-eu/ras-skills
   /plugin install ras@stratodata
   ```

3. Connect your RAS / BigQuery data so the skills have something to read — see
   [`docs/CONNECT-RAS-MCP.md`](docs/CONNECT-RAS-MCP.md).

Full step-by-step, including private-repo authentication, is in [`docs/SETUP.md`](docs/SETUP.md).

## How this is meant to be used

The skills describe *how* to analyse — the precision protocol, the RAS data model, the workflow.
Your **data** stays yours: it is reached through your own RAS / BigQuery MCP connector, which you
configure separately. Stratodata never sees your data through this repository; the repository only
carries the method.

## Repository layout

```
ras-skills/
├── .claude-plugin/
│   └── marketplace.json      # marketplace manifest — lists available plugins
├── plugins/
│   └── ras/                  # the RAS plugin
│       ├── .claude-plugin/
│       │   └── plugin.json    # plugin manifest
│       ├── skills/
│       │   └── ras-analysis/
│       │       └── SKILL.md   # the skill itself
│       └── README.md
├── docs/
│   ├── SETUP.md              # client onboarding & authentication
│   └── CONNECT-RAS-MCP.md    # connecting your BigQuery/RAS data
├── CLAUDE.md                 # guidance for Claude reading this repo
├── LICENSE.md                # usage terms
└── README.md                 # this file
```

## Keeping up to date

```
/plugin marketplace update stratodata
```

## Support

Questions or access issues: contact Stratodata at martin@stratodata.eu.
