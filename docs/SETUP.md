# Setup guide

How to install Stratodata's skills into your own Claude environment. One-time setup, ~10 minutes.

## 1. Accept the GitHub invitation

Stratodata invites your GitHub account to the private `stratodata-eu/ras-skills` repository.
You'll get an email, or find it at https://github.com/notifications. Accept it. If you don't have a
GitHub account yet, create a free one at https://github.com/signup first and send Stratodata the
username so they can invite it.

## 2. Make sure Claude Code can reach the private repo

Because the repository is private, Claude Code needs your Git credentials. The simplest, most
reliable way is the GitHub CLI:

```bash
# install gh if needed: https://cli.github.com
gh auth login          # choose GitHub.com → HTTPS → login with browser
gh auth setup-git      # lets git (and Claude Code) reuse your GitHub login
```

Verify access:

```bash
git ls-remote https://github.com/stratodata-eu/ras-skills >/dev/null && echo "OK: access works"
```

If that prints `OK`, you're ready.

## 3. Add the marketplace and install the plugin

In Claude Code:

```
/plugin marketplace add stratodata-eu/ras-skills
/plugin install ras@stratodata
```

Then restart Claude Code (or reload) if prompted. Confirm it's active:

```
/plugin
```

You should see the `ras` plugin listed as installed, and `/ras:ras-analysis` available.

## 4. Connect your data

The skills analyse *your* RAS / BigQuery data through your own MCP connector, which you set up
separately. Follow [`CONNECT-RAS-MCP.md`](CONNECT-RAS-MCP.md).

## 5. Try it

Ask Claude something like:

> Using RAS, what were net sales last month vs the month before, and what drove the change?

Claude will follow the RAS analysis protocol: query for every number, label each claim, and cite
its sources.

## Staying up to date

When Stratodata publishes an update:

```
/plugin marketplace update stratodata
```

## Troubleshooting

- **`/plugin marketplace add` fails with an authentication error** — your Git credentials aren't set
  up for the private repo. Re-run step 2 (`gh auth login` + `gh auth setup-git`).
- **Plugin installs but skills don't trigger** — restart Claude Code so it re-scans plugins.
- **Skills run but say data is unavailable** — the RAS / BigQuery MCP connector isn't connected. See
  [`CONNECT-RAS-MCP.md`](CONNECT-RAS-MCP.md).
- **Anything else** — contact Stratodata at martin@stratodata.eu.
