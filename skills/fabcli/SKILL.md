---
name: fabcli
description: Use FabCLI to search, browse, inspect, and download assets from the Epic Games / Fab marketplace.
when_to_use: |
  Use when the user mentions: Fab, Epic Games Store, Unreal marketplace,
  Unreal assets, Fab library, asset search, asset download, marketplace
  pricing, asset reviews, or checking asset ownership.
argument-hint: "[fab search|fab library|fab listing|auth status|...]"
---

# FabCLI — Agent Guide

FabCLI is a command-line tool that wraps the Epic Games / Fab marketplace
APIs. All commands output compact JSON to stdout (add `--pretty` for
human-readable output). Errors go to stderr as structured JSON.

## Prerequisites

FabCLI must be on PATH. Verify: `fabcli --version`

If not installed, the user needs to:
1. Download the latest release ZIP from the releases page
2. Unzip to a permanent location
3. Run `install.bat` inside the extracted folder (adds to PATH)
4. Open a new terminal

## Authentication

FabCLI uses a **two-phase auth flow**:

1. **One-time interactive login** — the user must run this themselves:
   ```bash
   fabcli auth login
   ```
   If already authenticated, prints `{"ok":true,"already_authenticated":true}`
   immediately — no window opens.

   Otherwise opens a login window (native WebView). The user signs in
   with Epic credentials + 2FA. The code is captured automatically — no
   copy-paste needed. Window closes and the session is saved. Each login
   uses a fresh session (no persistent cookies — enables account switching).

   Use `fabcli auth login --manual` for the old paste flow if the
   WebView isn't available.

2. **Headless refresh** — every subsequent command loads the token,
   refreshes if expired, and runs without prompts. No user involvement.

**If you get exit code 2** (`auth_required`), tell the user:
> "Your FabCLI session has expired. Please run `fabcli auth login`
> in your terminal to re-authenticate, then I'll retry."

Do NOT attempt to run `fabcli auth login` yourself — it requires
user interaction (WebView window or TTY for paste).

### Auth commands

| Command | What it does |
|---|---|
| `fabcli auth login` | Interactive login (opens browser, requires TTY) |
| `fabcli auth logout` | Invalidate session + delete token file |
| `fabcli auth status` | Check if session is valid (headless) |
| `fabcli auth whoami` | Print account_id, display_name, email as JSON |

Check session before running commands:
```bash
fabcli auth status
# → {"authenticated":true,"expires_at":"...","refreshed":false}
```

## Fab Marketplace Commands

### Search

```bash
fabcli fab search --query "medieval kitbash" --channel unreal-engine --free
```

Flags:
- `-q, --query` — text search
- `--channel` — e.g. "unreal-engine"
- `--type` — e.g. "3d-model", "tool-and-plugin", "audio"
- `--category` — category filter
- `--sort` — "-relevance" (default), "-createdAt", "price", etc.
- `--count` — results per page
- `--cursor` — pagination cursor from previous results
- `--free` — only free assets
- `--discounted` — only discounted assets
- `--seller` — filter by seller name

Response: `{ "results": [{ "uid", "title", "listing_type", "is_free", "is_discounted", ... }], "count", "cursors": { "next", "previous" } }`

### Library

```bash
fabcli fab library
```

Lists all Fab assets owned by the authenticated account. No arguments.

Response: `{ "results": [{ "uid", "title", ... }], "cursors": {...} }`

### Listing (detail)

```bash
fabcli fab listing <uid>
fabcli fab listing --stdin    # read UID from stdin pipe
```

Fetches full detail for one listing: title, description, seller, ratings,
tags, thumbnails, pricing, category.

Response: `{ "uid", "title", "description", "listing_type", "user", "category", "ratings", "is_free", "tags", "thumbnails", ... }`

### Formats

```bash
fabcli fab formats <uid>
fabcli fab formats --stdin
fabcli fab formats <uid> --format unreal-engine
```

Shows every format a listing exposes, enriched with the format-specific
metadata: artifact versions (with `engineVersions` and
`targetPlatforms`), distribution method, and technical details. Pass
`--format <code>` (e.g. `unreal-engine`) to fetch a single format
directly — cheaper and faster than the default fan-out.

Exits `3` (`not_found`) if the listing or the requested `--format`
code doesn't exist.

Response: `[{ "assetFormatType": {"code":"unreal-engine", …}, "distributionMethod":"complete_project", "technicalDetails":"<p>…</p>", "versions": [{"artifactId":"…", "engineVersions":["UE_5.4","UE_5.5"], "targetPlatforms":["Windows"], "fileType":"source", "uid":"…"}, …] }]`

### Prices

```bash
fabcli fab prices <uid>                         # single listing
fabcli fab prices --offer-ids id1,id2,id3       # bulk (comma-separated)
```

Response (single): `[{ "offer_id", "price", "discounted_price", "discount_percentage", "currency_code" }]`

### Ownership

```bash
fabcli fab ownership <uid>
fabcli fab ownership --stdin
```

Response: `{ "licenses": [{ ... }] }` — check whether `acquired` is true.

### Reviews

```bash
fabcli fab reviews <uid>
fabcli fab reviews <uid> --sort-by newest --cursor <c>
fabcli fab reviews --stdin
```

Response: `{ "results": [{ "uid", "rating", "title", "content", "user": { "display_name" }, "created_at" }], "count", "cursors" }`

### Manifest (download URLs)

```bash
fabcli fab manifest --artifact-id X --namespace Y --asset-id Z [--platform Windows]
```

Returns signed CDN URLs for downloading the asset's files.

Response: `[{ "artifact_id", "asset_format", "distribution_points": [{ "url", "signature", ... }], "manifest_hash" }]`

### Download

```bash
fabcli fab download --artifact-id X --namespace Y --asset-id Z -o ./my-asset/
fabcli fab download --artifact-id X --namespace Y --asset-id Z -o ./my-asset/ --platform Windows --jobs 16
```

Downloads all files: checks disk space first (needs ~2x asset size),
parallel chunk fetching to temp files, SHA1 verification, file
reassembly, directory structure preserved. No memory limit — works for
any asset size. Writes `.fabcli-asset.json` sidecar with metadata for
UE5CLI integration. Shows progress on stderr.

Flags: `--artifact-id`, `--namespace`, `--asset-id` (required, from
manifest/library output), `-o, --output` (required), `--platform`
(optional), `--jobs` (default 8).

Response: `{"ok":true,"files":N,"total_bytes":M,"elapsed_seconds":T,"output_dir":"...","sidecar":".fabcli-asset.json"}`

## Common Recipes

### Recipe 1: Search and inspect an asset

```bash
# Search for assets
fabcli fab search -q "sci-fi props" --channel unreal-engine

# Inspect a specific result (pipe the UID)
fabcli fab search -q "sci-fi props" | jq -r '.results[0].uid' | fabcli fab listing --stdin --pretty
```

### Recipe 2: Check ownership before recommending

```bash
UID="some-listing-uid"
fabcli fab ownership "$UID"
# Check if .licenses is non-empty or .acquired is true
```

### Recipe 3: Download an owned asset

```bash
# 1. List library to find artifact info
fabcli fab library --pretty

# 2. Download (artifact_id, namespace, asset_id from library output)
fabcli fab download --artifact-id "$AID" --namespace "$NS" --asset-id "$ASID" -o ./my-asset/

# 3. If UE5CLI is available, install into project:
# ue5cli install-asset ./my-asset/ --project C:\Projects\MyGame
```

### Recipe 4: Browse free Unreal Engine assets

```bash
fabcli fab search --free --channel unreal-engine --sort -createdAt --count 20
```

## Exit Codes

| Code | Meaning | What to do |
|---|---|---|
| 0 | Success | Parse stdout JSON normally |
| 1 | Generic error | Report the error message to the user |
| 2 | Auth required | Tell user to run `fabcli auth login` |
| 3 | Not found | The listing/asset doesn't exist — check the UID |
| 4 | Rate limited | Wait ~30 seconds and retry |
| 5 | Network error | Report connectivity issue to the user |
| 6 | Invalid args | Fix the command (wrong flags, missing UID, etc.) |

Error JSON on stderr: `{"error":{"kind":"<kind>","message":"<description>"}}`

## Token Storage

- **Windows:** `%APPDATA%\fabcli\token.json`
- **Linux:** `~/.config/fabcli/token.json`

Override with `FABCLI_TOKEN_PATH` environment variable for multi-account
use or testing:
```bash
FABCLI_TOKEN_PATH=/tmp/alt-token.json fabcli auth status
```

Prefer `fabcli auth status` to check auth state rather than inspecting
the token file directly.

## What FabCLI Does NOT Do

- **Does not install assets into UE projects** — `fab download` puts
  files in an output directory. Use UE5CLI (`ue5cli install-asset`)
  for automated project placement, or move files manually.
- **Does not manage Unreal Engine projects** — no project creation,
  plugin installation, or build management.
