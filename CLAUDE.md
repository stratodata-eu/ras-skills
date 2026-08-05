# Guidance for Claude

This repository is a Stratodata plugin marketplace that distributes analytics skills to clients.

## What this repo is

- `.claude-plugin/marketplace.json` lists the available plugins.
- Each plugin lives under `plugins/<name>/` with its own `.claude-plugin/plugin.json` and a
  `skills/` folder. Skills are auto-discovered from `skills/<skill-name>/SKILL.md`.
- The skills carry the *method* only. Client data is never stored here. Data is read through the
  client's own RAS / BigQuery MCP connector, configured separately in their Claude environment.

## When helping in a client environment

- If the RAS / BigQuery MCP connector is not connected, the analysis skills cannot read data. Say
  so plainly and point to `docs/CONNECT-RAS-MCP.md` rather than guessing at numbers.
- The `ras-analysis` skill is the governing standard for any data work: ground every number in a
  query, label each claim verified / inferred / unknown, and never fill a data gap with a guess.
  Follow it even when the user does not explicitly ask for precision.

## When helping maintain this repo (Stratodata internal)

- To add a skill: create `plugins/<plugin>/skills/<skill-name>/SKILL.md` with `name` and
  `description` frontmatter. No need to register it in `plugin.json` — skills auto-discover.
- To add a plugin: create `plugins/<name>/.claude-plugin/plugin.json` and add an entry to
  `marketplace.json` with its `source` path.
- Bump the plugin `version` on every meaningful change so installed clients receive the update.
- Keep everything in this repo generic. Nothing client-specific — no client names, dataset IDs,
  credentials, or private figures. Client-specific material belongs in a separate private location.
