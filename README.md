# fabcli-skills

Claude Code plugin for [FabCLI](https://github.com/zirklerite/FabCLI) — lets
AI agents search, browse, inspect, and download assets from the Epic Games /
Fab marketplace through composable shell commands.

## Install

Two equivalent paths — pick whichever fits your workflow:

### Option A — install from the FabCLI binary (recommended for binary users)

If you already installed FabCLI from a release zip, the binary embeds this
SKILL.md at compile time. Run:

```bash
fabcli skill install
```

This writes `~/.claude/skills/fabcli/SKILL.md` from the binary's embedded
copy, so the skill version is always in lock-step with your binary's CLI
surface. Works offline, no GitHub access required.

### Option B — install from this marketplace plugin

```
/plugin marketplace add zirklerite/fabcli-skills
/plugin install fabcli@fabcli-skills
```

Pulls the latest `master` of this repo. Useful if you want a SKILL.md
update between FabCLI binary releases (e.g. recipe additions or trigger
phrasing tweaks that don't depend on new CLI flags).

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
