---
name: fabcli
version: 0.1.0
description: Use FabCLI to search, browse, inspect, and download assets from the Epic Games / Fab marketplace.
when_to_use: |
  Use when the user mentions: Fab, Epic Games Store, Unreal marketplace,
  Unreal assets, Fab library, asset search, asset download, marketplace
  pricing, asset reviews, or checking asset ownership.
argument-hint: "[search|library|listing|claim|download|auth status|...]"
---

# FabCLI — Agent Guide

FabCLI is a command-line tool that wraps the Epic Games / Fab marketplace
APIs. All commands output compact JSON to stdout (add `--pretty` for
human-readable output). Errors go to stderr as structured JSON.

## Prerequisites

FabCLI must be on PATH. Verify: `fabcli --version`

If not installed, the user needs to:
1. Download the latest release ZIP (Windows) or tar.gz (Linux) from
   the releases page
2. Unpack to a permanent location
3. Run the installer inside the extracted folder:
   - Windows: `install.bat`
   - Linux: `./install.sh` (symlinks `~/.local/bin/fabcli`)
4. Open a new terminal

### Linux support

**Supported Linux platform: Ubuntu 24.04 LTS.** This is the only
distribution FabCLI is built and tested against. Other distros
are unsupported — they may work but are not validated.

One Linux binary ships per release: `fabcli-v*-linux64.tar.gz`
(gnu/glibc, with the webview-backed Fab session capture compiled
in). Runtime deps: `libgtk-3-0`, `libwebkit2gtk-4.1-0`,
`libsoup-3.0-0`. Install:
`sudo apt install libgtk-3-0 libwebkit2gtk-4.1-0 libsoup-3.0-0`.

The daemon path that accelerates `claim-batch` is Windows-only
today — on Linux each call spins up a fresh WebView (~1–2 s per
UID).

## Authentication

FabCLI uses a **two-phase auth flow**:

1. **One-time interactive login** — the user must run this themselves:
   ```bash
   fabcli auth login
   ```
   A single `auth login` establishes **both** the Epic OAuth session
   (unlocks search, library, download, ...) and the Fab web session
   (unlocks `claim` and rich `ownership`) in one WebView flow.

   If both sessions are already valid, prints
   `{"ok":true,"already_authenticated":true}` immediately — no
   window opens. Otherwise the login window appears, the user signs in
   with Epic credentials + 2FA once, and the code is captured
   automatically.

   Use `fabcli auth login --manual` for the paste flow if the WebView
   isn't available.

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
| `fabcli auth login` | Combined Epic + Fab login (one WebView, one 2FA). Skips the window if both sessions are already fresh. |
| `fabcli auth logout` | Invalidate both Epic and Fab sessions; delete token + WebView data folder |
| `fabcli auth status` | Check Epic + Fab session health in one call (headless) |
| `fabcli auth whoami` | Print account_id, display_name, email as JSON |

Single-step onboarding (once per ~90 days):
```bash
fabcli auth login       # unlocks everything — search, library, claim, ownership, download, ...
```

Check session before running commands:
```bash
fabcli auth status
# → {
#     "authenticated": true,
#     "expires_at": "...", "refreshed": false,
#     "fab": {
#       "session_present": true,
#       "expires_at": "2026-07-16T05:58:58+00:00",
#       "days_remaining": 82,
#       "needs_refresh": false
#     }
#   }
```

#### Session health

`auth status` reports **both** sessions in one JSON object:

| Session | Lifetime | Auto-refresh? |
| --- | --- | --- |
| Epic OAuth (top-level `expires_at`) | ~36h access + ~1y refresh | Yes — every command silently refreshes |
| Fab (`fab` block) | 90 days | No — must re-run `fabcli auth login` manually |

Use `fab.needs_refresh: true` as the signal to re-run
`fabcli auth login`. It's `true` when the session is expired
**or** expires within the warn threshold (default 7 days,
overridable via `FABCLI_FAB_SESSION_WARN_DAYS`). When no Fab session
is persisted, the `fab` block is `{"session_present": false,
"needs_refresh": true}`.

**Proactive stderr warning.** When you run a Fab-gated command
(`claim`, `claim-batch`, rich `ownership`) with a session within
the warn threshold, FabCLI prints one line to stderr (not stdout,
so JSON pipelines are unaffected):

```
WARNING: Fab session expires in 5 days; run 'fabcli auth login' to refresh.
```

Fires at most once per CLI invocation — a 30-UID `claim-batch`
doesn't spam. Disable entirely with `FABCLI_FAB_SESSION_WARN_DAYS=0`.
`auth status` itself does NOT emit the stderr warning (its JSON is
the signal for that command).

## Commands

### Search

```bash
fabcli search --query "medieval kitbash" --filter channels=unreal-engine --filter is_free=1
```

**Mental model:** every search parameter is a Fab URL key. Use
`--filter <KEY=VALUE>` (repeatable) for any filter. The four other
flags are non-filter concerns: `--query` (text), `--sort` (ordering),
`--count` and `--cursor` (pagination).

Flags (five total):
- `-q, --query` — text search.
- `--sort` — see "Known sort values" below. Leading `-` means
  descending (Fab API convention). Both `--sort=-createdAt` and
  `--sort "-createdAt"` work.
- `--count` — results per page.
- `--cursor` — pagination cursor from previous results.
- `--filter <KEY=VALUE>` — repeatable. Each occurrence emits one
  `?KEY=VALUE` query param (URL-encoded value, raw key). Repeated
  same-key invocations preserve order — Fab's multi-valued
  convention. See "How to do X" recipes and "Known Fab filter keys"
  below.

#### How to do X (recipes)

| Use case | Invocation |
| --- | --- |
| Free assets only | `--filter is_free=1` |
| Discounted assets | `--filter is_discounted=true` |
| Limited Time Free (100% off paid items) | `--filter min_discount_percentage=100` |
| Filter by channel | `--filter channels=unreal-engine` |
| Multiple channels (OR / union) | `--filter channels=unity --filter channels=unreal-engine` |
| Filter by listing type | `--filter listing_types=3d-model` |
| Filter by category | `--filter categories=<slug>` |
| Filter by seller | `--filter seller=<name>` |
| Multi-style (AND / narrowing) | `--filter styles=anime --filter styles=lowpoly` |
| Technical feature | `--filter technical_features=rigged` |
| Asset format | `--filter asset_formats=metahuman` |
| License | `--filter licenses=cc-by` |
| Rating range | `--filter min_average_rating=4 --filter max_average_rating=5` |
| Price range | `--filter min_price=0 --filter max_price=20` |
| Published in last 7 days | `--filter published_since=YYYY-MM-DD` (caller computes `today - 7d`) |
| Published since specific date | `--filter published_since=2026-04-01` |

#### Date filtering (no `--since` flag)

FabCLI does **not** have a `--since` flag. Date filtering is done via
`--filter published_since=YYYY-MM-DD` — and **the caller is responsible
for computing the date**.

- "Published in the last 7 days": compute `today - 7 days`, format as
  `YYYY-MM-DD`, then `--filter published_since=YYYY-MM-DD`.
- "Published since April 1st 2026":
  `--filter published_since=2026-04-01`.
- There is no relative-window shorthand (`7d`, `30d`); the server only
  accepts ISO dates.

If you encounter older docs or recipes that mention `fabcli search
--since 7d`, replace with the explicit-date `--filter` form above.

#### Known Fab filter keys

Multi-valued (repeatable, but semantics vary per key):
- `styles` — **AND** (results must match all). Examples: `anime`,
  `cartoon`, `stylized`, `lowpoly`, `realistic`, `pixelart`, …
- `technical_features` — examples: `animated`, `rigged`,
  `destructible`, `customizable`, …
- `asset_formats` — examples: `metahuman`, `3ds-max`, `blender`,
  `fbx`, `unreal-engine`, `unity`, …
- `licenses` — examples: `cc-by`, `uefn-reference-only`, `epic-only`, …
- `channels` — **OR / union** (results from any channel). Known slugs:
  `unreal-engine`, `unity`, `uefn`, `metahuman`.

**Multi-value semantics are not uniform across keys.** Fab applies AND
to `styles` and OR to `channels`. When in doubt, smoke-test with
single-filter vs. multi-filter and inspect the result intersection.

Scalar booleans (use `1` / `true`):
- `is_free=1` — permanently free assets only (Fab's `is_free=true`).
  Does *not* return paid items that are temporarily 100% off — use
  `min_discount_percentage=100` for those.
- `is_discounted=true` — any current discount (1%+ off).

Scalar pairs:
- `min_average_rating` / `max_average_rating` (0-5)
- `min_price` / `max_price` (decimal)

Scalar singletons:
- `min_discount_percentage` (1-100). `100` matches Fab's "Limited
  Time Free" page (`fab.com/limited-time-free`).
- `listing_types` (`3d-model`, `tool-and-plugin`, `audio`,
  `material`, …)
- `categories` (slug)
- `seller` (seller name)

Date scalar:
- `published_since=YYYY-MM-DD` — caller computes the date. No
  relative-window shorthand.

Unknown keys are forwarded raw and yield empty results — Fab does not
echo or validate filter keys. If you typo `style=anime` (singular)
instead of `styles=anime`, you'll see zero matches, not an error.

#### Known sort values

`--sort` is a passthrough string. Observed accepted values:

- `-relevance` (default)
- `-createdAt`, `createdAt`
- `firstPublishedAt`
- `price`, `-price`
- `-min_discount_percentage`
- `title`, `-title`
- `-ratings.averageRating`

Fab's accepted set may grow; this is a best-effort enumeration. The
leading `-` indicates descending.

Response: `{ "results": [{ "uid", "title", "listing_type", "is_free", "is_discounted", "user", "ratings", "seller", "rating", ... }], "count", "cursors": { "next", "previous" } }`

Each result row carries two convenience fields alongside the raw Fab
shape:
- `seller`: `user.sellerName` trimmed; `null` if absent or blank
- `rating`: `ratings.averageRating` as a number, or `null`

Raw `user` and `ratings` objects are preserved unchanged for full-fidelity consumers.

### Library

```bash
fabcli library
fabcli library --count 500        # larger per-page, fewer round-trips
```

Lists all Fab assets owned by the authenticated account. Internally
paginates through Fab's `/e/accounts/<id>/ue/library` endpoint until
exhausted.

Response: `{ "results": [{ "uid", "title", ... }], "cursors": {...} }`

**Tuning `--count`** (empirical, as of 2026-04-21; re-test if Fab
changes behaviour):

| `--count` | behaviour |
| --- | --- |
| 100 (default) | Safe; works everywhere. ~12 round-trips for a 1k-item library. |
| **500** | Accepted; ~3× fewer requests; measurably faster on large libraries. Recommended for bulk workflows. |
| 1000 | Accepted but returns slightly fewer items than exist (off-by-a-few); don't rely on it. |
| 2000+ | Fab returns an empty page; effectively the cap. |

If Fab changes the cap, re-probe with `fabcli library --count <N>`
and compare `results.length` across values — the largest value that
still returns the full library is today's sweet spot. Drop back to
100 whenever something looks off.

#### Library cache (opt-in)

The library endpoint is the slowest thing FabCLI talks to — Fab
serves it at ~10 items/second, so a 1k-item library takes
~100 seconds to paginate. To skip redundant fetches, FabCLI ships an
opt-in on-disk cache:

```bash
export FABCLI_LIBRARY_CACHE=1    # enable cache for this session
fabcli library                   # first call: fetches, writes cache
fabcli library                   # second call: reads cache (~10 ms)
fabcli ownership --from-library  # also reads cache
fabcli claim-batch --from-library # also reads cache
```

Flags (all override the env per-call):

| Flag | Meaning |
| --- | --- |
| `--cache` | Read-if-fresh / write on miss for this invocation. |
| `--no-cache` | Bypass read and write for this invocation. |
| `--refresh` | Force a live fetch and overwrite the cache. |
| `--clear` | Delete the cache file and exit. No network call. |

Env vars:

- `FABCLI_LIBRARY_CACHE` — `1` / `true` / `yes` enables cache
  reads and writes. Anything else leaves today's always-live
  behaviour intact.
- `FABCLI_LIBRARY_CACHE_TTL` — seconds; default `3600` (1 hour).
  `0` effectively disables reads without unsetting the main env.

**Strict env gating.** If `FABCLI_LIBRARY_CACHE` is not set,
FabCLI will NOT read an existing cache file, even if it's fresh —
the cache only activates when you explicitly opt in. Set it once
per script/session (or use `--cache` per call).

**Which commands read the cache** (when enabled):

- `fabcli library`
- `fabcli ownership --from-library`
- `fabcli ownership --batch <uids>` (bearer-only fallback path
  without a Fab session)
- `fabcli ownership <uid>` (same bearer-only fallback)
- `fabcli claim-batch --from-library`

Every other command is unaffected (they don't touch the library).

**Invalidation guarantee.** A successful `fabcli claim` or
`fabcli claim-batch` deletes the cache file before returning, so
the next library read sees the newly-owned asset without waiting
for the TTL to lapse. `fabcli auth logout` also deletes the cache
alongside the webview-data folder.

**Data-at-rest.** The file contains asset metadata only — no
tokens, cookies, or CSRF secrets. On Unix, written with mode
`0600` (user-only read/write), mirroring the token file's
hygiene. A leak is a privacy concern (asset list disclosure),
not a credential concern.

Account safety: the cache file carries the Epic `account_id` in
its envelope and self-invalidates on mismatch. Swapping
`token.json` between Fab accounts without `auth logout` still
triggers a fresh fetch for the new account on first read.

### Listing (detail)

```bash
fabcli listing <uid>
fabcli listing --stdin    # read UID from stdin pipe
```

Fetches full detail for one listing: title, description, seller, ratings,
tags, thumbnails, pricing, category.

Response: `{ "uid", "title", "description", "listing_type", "user", "category", "ratings", "is_free", "tags", "thumbnails", ... }`

### Formats

```bash
fabcli formats <uid>
fabcli formats --stdin
fabcli formats <uid> --format unreal-engine
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
fabcli prices <uid>                         # single listing
fabcli prices --offer-ids id1,id2,id3       # bulk (comma-separated)
```

Response (single): `[{ "offer_id", "price", "discounted_price", "discount_percentage", "currency_code" }]`

### Ownership

```bash
fabcli ownership <uid>
fabcli ownership --stdin
fabcli ownership --batch uid1,uid2,uid3     # batch (CSV)
fabcli ownership --from-stdin                # batch (newline-delimited)
fabcli ownership --from-library              # every UID in the library
```

Reports whether a listing is owned. With a Fab session (established
automatically by `auth login`), queries
`/i/users/me/listings-states/{uid}` through a hidden WebView and
returns richer state (entitlement id, held licenses, wishlist
flag). Without a session, falls back to matching the listing UID
against the user's library (`customAttributes[].ListingIdentifier`)
— works on headless systems too.

Response shape depends on which path ran; both include
`listingUid`, `owned`, and a `source` field identifying the path:

```json
// Fab-session path
{"listingUid":"...","owned":true,"source":"fab_session","state":{"acquired":true,"entitlementId":"...","ownership":[...],"wishlisted":false,...}}

// Library-derived path (no session, or session expired)
{"listingUid":"...","owned":true,"source":"library","entitlement":{"assetId":"...","title":"...","projectVersions":[...]}}

// Batch (any `--batch` / `--from-stdin` / `--from-library` flag)
{"ok":true,"results":[{...},{...},{...}],"meta":{"total":3}}
```

Single-UID calls still emit the flat shape for back-compat.

### Claim (free assets only)

```bash
fabcli claim <uid>
fabcli claim --stdin
```

Adds a free listing to the library. Requires a Fab session
(established during `auth login`). **Cannot be used to purchase
paid assets** — any non-free listing returns a structured "not_free"
response without issuing any claim request.

Responses (all exit 0 unless noted):

```json
// Free asset successfully claimed
{"ok":true,"claimed":true,"title":"FREE Mayonnaise","uid":"..."}

// Already in library (idempotent re-run)
{"ok":true,"already_owned":true,"title":"...","uid":"..."}

// Paid asset — no purchase made; informational
{"ok":false,"reason":"not_free","title":"...","uid":"...","price":3777.73,"currency":"TWD","purchase_url":"https://www.fab.com/listings/..."}
```

**IMPORTANT for agents:** Exit code 0 does NOT always mean "claimed."
Always check the `ok` field: `ok: true` = claimed or already owned,
`ok: false` = not claimed (paid asset). Parse the `reason` field for
specifics.

Exits `2` (auth_required) with a clear message if no Fab session exists.
Exits `1` if the POST succeeded but ownership couldn't be
verified afterward.

### Claim-batch (many free assets in one go)

```bash
fabcli claim-batch --uids uid1,uid2,uid3
fabcli claim-batch --stdin                    # newline-delimited UIDs
fabcli claim-batch --from-stdin-json          # JSON: {"results":[{"uid":…}]} or [{"uid":…}]
fabcli claim-batch --from-library             # every UID from the library
```

Runs the full `claim` pre-flight + POST + verify sequence for each
UID. Response shape:

```json
{
  "ok": true,
  "results": [
    {"uid":"...","ok":true,"claimed":true,"title":"..."},
    {"uid":"...","ok":true,"already_owned":true,"title":"..."},
    {"uid":"...","ok":false,"reason":"not_free","price":3777.73,"currency":"TWD","purchase_url":"..."}
  ],
  "meta": {"total":3,"claimed":1,"already_owned":1,"skipped_paid":1,"failed":0}
}
```

Exit `0` unless any UID errored (`meta.failed > 0`), in which case
exit `1` with partial results still emitted. All per-UID safety
properties of `claim` hold: no paid asset is ever POSTed.

**Batching tip:** All batch commands (`claim-batch`, `ownership
--batch*`) keep a background browser daemon alive between UIDs on
Windows. First UID pays the ~1–2s WebView startup; subsequent UIDs
are ~100ms each. Use batches over loops whenever you have more
than 2 UIDs. Set `FABCLI_NO_DAEMON=1` to bypass the daemon for
debugging.

### Reviews

```bash
fabcli reviews <uid>
fabcli reviews <uid> --sort-by newest --cursor <c>
fabcli reviews --stdin
```

Response: `{ "results": [{ "uid", "rating", "title", "content", "user": { "display_name" }, "created_at" }], "count", "cursors" }`

### Manifest (download URLs)

```bash
fabcli manifest --artifact-id X --namespace Y --asset-id Z [--platform Windows]
```

Returns signed CDN URLs for downloading the asset's files.

Response: `[{ "artifact_id", "asset_format", "distribution_points": [{ "url", "signature", ... }], "manifest_hash" }]`

### Download

```bash
fabcli download --artifact-id X --namespace Y --asset-id Z -o ./my-asset/
fabcli download --artifact-id X --namespace Y --asset-id Z -o ./my-asset/ --platform Windows --jobs 16
```

Downloads all files: checks disk space first (needs ~2x asset size),
parallel chunk fetching to temp files, SHA1 verification, file
reassembly, directory structure preserved. No memory limit — works for
any asset size. Writes `.fabcli-asset.json` sidecar with metadata for
future UE5CLI integration (companion tool, not yet released). Shows
progress on stderr.

Flags: `--artifact-id`, `--namespace`, `--asset-id` (required, from
manifest/library output), `-o, --output` (required), `--platform`
(optional), `--jobs` (default 8).

Response: `{"ok":true,"files":N,"total_bytes":M,"elapsed_seconds":T,"output_dir":"...","sidecar":".fabcli-asset.json"}`

## Fab Pricing Model

Fab's pricing has three "free" conditions — understanding them
matters when reading search results and predicting `claim` behavior:

| Field | Type | Meaning |
|---|---|---|
| `isFree` | bool | Permanently free asset (rare) |
| `startingPrice.price` | float | Full price in user's local currency |
| `startingPrice.discountedPrice` | float/null | Price after discount (null if no active discount) |
| `startingPrice.currencyCode` | string | User's currency code (e.g. "TWD", "USD", "EUR") |

**An asset is claimable for free if ANY of these are true:**
- `isFree == true` (permanently free)
- `price == 0` ($0 asset, `isFree` may still be `false`)
- `discountedPrice == 0` (100% discount, temporarily free)

**Search filtering:** `--filter is_free=1` sends `is_free=true` to
the server and matches **only permanently free** items — Fab's
`is_free=true` server filter does not return temporarily-100%-off
items. To find "Free for the Month" / "Limited Time Free" assets,
use `--filter min_discount_percentage=100` instead — that matches
Fab's own `/limited-time-free` page. Combine the two queries if you
want the union.

**`claim`** uses the same three-condition check. Paid assets are
hard-blocked — the POST is never sent. The response includes the
price and a `purchase_url` for the user to buy manually.

**Note:** prices are in the user's local currency (determined by
their Epic account region), not USD.

## Common Recipes

### Recipe 1: Search and inspect an asset

```bash
# Search for assets
fabcli search -q "sci-fi props" --filter channels=unreal-engine

# Inspect a specific result (pipe the UID)
fabcli search -q "sci-fi props" | jq -r '.results[0].uid' | fabcli listing --stdin --pretty
```

### Recipe 2: Check ownership before recommending

```bash
UID="some-listing-uid"
fabcli ownership "$UID"
# Check if .licenses is non-empty or .acquired is true
```

### Recipe 3: Download an owned asset

```bash
# 1. List library to find artifact info
fabcli library --pretty

# 2. Download (artifact_id, namespace, asset_id from library output)
fabcli download --artifact-id "$AID" --namespace "$NS" --asset-id "$ASID" -o ./my-asset/

# 3. Import into your UE project manually (UE5CLI companion tool
#    that automates this step is planned but not yet released).
```

### Recipe 4: Browse free Unreal Engine assets

```bash
fabcli search --filter is_free=1 --filter channels=unreal-engine --sort -createdAt --count 20
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

The file is **encrypted at rest** using the OS user keystore
(DPAPI on Windows, libsecret on Linux). It is no longer
directly parseable JSON — do **not** try to `cat` / `jq` it.
Use `fabcli auth status` to check session state.

Override with `FABCLI_TOKEN_PATH` for multi-account workflows or
testing:
```bash
FABCLI_TOKEN_PATH=/tmp/alt-token.json fabcli auth status
```

If `auth login` fails with "OS keystore unavailable," the user
needs to start their keyring daemon (`gnome-keyring-daemon` /
`kwalletd5` on Linux; DPAPI is always available on Windows 10+).

## Update FabCLI

`fabcli update` swaps the running binary in place from the public
GitHub release. Use it when:

- The user asks how to update FabCLI.
- The agent sees a stderr line like
  `fabcli: a newer version (X.Y.Z) is available. Run 'fabcli update' to upgrade.`

| Command | What it does |
|---|---|
| `fabcli update` | Download the latest release for the host triple, verify SHA-256 against `SHA256SUMS.txt`, swap the binary in place. Exits 0 with `{"updated": true, "from": "...", "to": "...", "asset": "..."}`. If already at latest, exits 0 with `unchanged: true`. |
| `fabcli update --check` | Print `{"running": "...", "latest": "...", "newer_available": bool}`; no download. |
| `fabcli update --to X.Y.Z` | Pin to a specific tag (forward or back). |
| `fabcli update --force` | Re-download and swap even if already at the latest. |

One archive per platform: Windows ZIP or Linux tar.gz. The asset
matcher resolves to the host's platform automatically.

The once-a-day **stderr hint** is opt-out:

- `FABCLI_NO_UPDATE_CHECK=1` — disable the hint entirely.
- `FABCLI_UPDATE_CHECK_TTL_HOURS=<int>` — tune the cache (default 24,
  `0` disables).
- `FABCLI_UPDATE_REMOTE=<owner/repo>` — override the GitHub repo
  (testing/forks).

The hint goes to stderr only — stdout JSON is never touched. If the
agent sees the hint, surface it to the user; don't try to run
`fabcli update` automatically (the user's binary may live in a path
that requires elevated rights to overwrite).

## Manage the Skill from the Binary

`fabcli skill` lets the user install, update, or remove this very
SKILL.md without leaving the terminal. The binary embeds the canonical
copy at compile time, so the offline default works on any host that
already has FabCLI installed.

| Command | What it does |
|---|---|
| `fabcli skill install` | Write the embedded SKILL.md to `~/.claude/skills/fabcli/SKILL.md`. Refuses to overwrite a different file unless `--force`. |
| `fabcli skill update` | Same as `install --force`; reports `old_version → new_version` on stderr. |
| `fabcli skill uninstall` | Delete the installed SKILL.md (and the `fabcli/` directory if empty). Refuses to delete a file whose frontmatter `name:` isn't `fabcli`, unless `--force`. |
| `fabcli skill status` | JSON: embedded version, installed version, target path, `matches_embedded`. With `--remote`, also fetches the latest version from the public GitHub repo. |
| `fabcli skill path` | Print just the resolved install path (single line, no JSON envelope) for scripting. |

**Sources** (`--source <s>`):
- `embedded` (default) — uses the binary's compile-time copy. Works offline.
- `github` — fetches from `https://raw.githubusercontent.com/zirklerite/fabcli-skills/master/skills/fabcli/SKILL.md`. Override the URL via `FABCLI_SKILLS_REMOTE_URL` (testing/forks).
- `path=<file>` — read from a local file.

**Scope** (`--scope <s>`): `user` (default, `~/.claude/skills/`) or `project` (`./.claude/skills/` relative to cwd). `--path <dir>` overrides everything; `FABCLI_SKILLS_DIR=<dir>` is the env-level override (priority: `--path` > env > `--scope project` > `--scope user`).

If the user mentions wanting the FabCLI skill in Claude Code, suggest `fabcli skill install`. The marketplace path (`/plugin marketplace add zirklerite/fabcli-skills`) still works for users who prefer it — both produce the same SKILL.md.

## Recipes

> **Tip**: for any recipe that touches the library (directly or via
> `--from-library`), preface with `export FABCLI_LIBRARY_CACHE=1`
> so repeated library reads come from disk instead of re-paginating.
> See "Library cache (opt-in)" in the Library section for details.

### Find new free assets this week

```bash
# Caller computes today - 7 days as YYYY-MM-DD, then passes it through.
fabcli search --filter is_free=1 --filter published_since=2026-04-23
```

The server applies `published_since` itself; FabCLI does not paginate
or post-process by date. For wider windows, compute the date and pass
it as `published_since=YYYY-MM-DD`.

### Find Fab's "Limited Time Free" / "Free for the Month" assets

```bash
fabcli search --filter min_discount_percentage=100 --sort=-createdAt
```

This is the canonical, programmatic way to get the list shown on
`https://www.fab.com/limited-time-free` — paid items currently
100%-off, claimable for free until the promotion ends. **Do not use
`--filter is_free=1`** for this; that only returns *permanently* free
items and misses the temporary promos. Pipe straight into
`claim-batch` to grab them all:

```bash
fabcli search --filter min_discount_percentage=100 \
  | fabcli claim-batch --from-stdin-json
```

To find the union of permanently-free + temporarily-100%-off,
run both queries and merge the UIDs client-side:

```bash
( fabcli search --filter is_free=1 --filter published_since=2026-04-01 --count 500 ;
  fabcli search --filter min_discount_percentage=100 --count 100 ) \
  | jq -s '.[0].results + .[1].results | unique_by(.uid)' \
  | fabcli claim-batch --from-stdin-json
```

### Claim every new free asset this month

```bash
export FABCLI_LIBRARY_CACHE=1     # speed up any library reads below

# Caller computes 30 days back as YYYY-MM-DD, e.g. 2026-04-01.
fabcli search --filter is_free=1 --filter published_since=2026-04-01 \
  | fabcli claim-batch --from-stdin-json
```

Search returns only the last 30 days' worth of free assets (fewer
Fab API pages). `claim-batch --from-stdin-json` extracts the UIDs,
reuses the browser daemon across every UID, and emits a single
aggregate `{ok, results, meta}` envelope. Typically ~100ms per UID
after daemon warm-up vs. ~1–2s if you'd looped `fabcli claim <uid>`
one at a time. The `claim-batch` auto-invalidates the library cache
on any successful claim, so subsequent `ownership --from-library`
calls see the newly-owned asset without waiting for the TTL.

### Re-verify ownership across the entire library

```bash
fabcli ownership --from-library
```

Useful as a periodic sanity check that every library entry is still
marked acquired on the server side — or as a source for `fabcli
download` on everything owned.

## What FabCLI Does NOT Do

- **Does not install assets into UE projects** — `fab download` puts
  files in an output directory. UE5CLI (planned companion tool, not
  yet released) will automate project placement; for now, move files
  manually.
- **Does not manage Unreal Engine projects** — no project creation,
  plugin installation, or build management.
