# Connecting your RAS / BigQuery data

The skills in this repository describe *how* to analyse data. They do not contain any data. To
actually analyse your numbers, Claude needs a live connection to your RAS / BigQuery dataset — an
**MCP connector** — in your Claude environment.

This connector is set up separately from the plugin, and your data never passes through this
repository.

## What you need

- Access to your RAS BigQuery project (Stratodata provisions this for you).
- A service account or credentials with read access to your dataset, provided by Stratodata.

## Options

> Stratodata will tell you which of these applies to your account and hand over the exact
> connection details. This section is an overview so you know what to expect.

**Option A — Managed connector (recommended).** Stratodata gives you a ready-made MCP connector
configuration for your project. You add it to your Claude environment and authenticate once.

**Option B — Self-hosted BigQuery MCP.** If you prefer to run the connector yourself, you configure
a BigQuery MCP server pointed at your project and dataset, using credentials Stratodata provides.

## Verify the connection

Once connected, ask Claude:

> List the tables available in my RAS dataset.

If Claude can list your tables, the connector is live and the `ras-analysis` skill has data to work
with. If it can't, the connector isn't connected yet — contact Stratodata.

## Security notes

- Your credentials and data stay in *your* environment. Stratodata does not receive your data
  through this plugin repository.
- Grant the connector **read-only** access to your dataset.
- If you ever rotate or revoke credentials, the connector simply stops working until reconnected —
  it can't be used to reach anything you didn't grant.

Questions: martin@stratodata.eu.
