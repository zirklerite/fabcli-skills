# fabcli-skills

Claude Code plugin for [FabCLI](https://github.com/zirklerite/FabCLI) — lets
AI agents search, browse, inspect, and download assets from the Epic Games /
Fab marketplace through composable shell commands.

## Install

```
/plugin marketplace add zirklerite/fabcli-skills
/plugin install fabcli@fabcli-skills
```

## Prerequisites

FabCLI must be installed and on PATH. Download the latest release from
[FabCLI releases](https://github.com/zirklerite/FabCLI/releases), unzip,
and run `install.bat` (Windows) or copy the binary to `/usr/local/bin/`
(Linux).

Then authenticate once:

```bash
fabcli auth login
```

## What the skill teaches Claude Code

- When to use FabCLI (triggers on Fab, Epic Games Store, Unreal marketplace,
  asset search/download/library requests)
- How to authenticate and handle expired sessions
- All available commands (`auth` and `fab` subcommand groups) with flags
  and JSON output shapes
- Common recipes: search → inspect → check ownership → get download manifest
- Exit code handling so the agent knows what to do on errors
- What FabCLI does NOT do (no byte downloads, no UE project management)

## License

MIT
