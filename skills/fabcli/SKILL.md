---
name: fabcli
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
| `fabcli auth login` | Interactive Epic OAuth login (opens browser) |
| `fabcli auth fab-login` | Establish a Fab web session — needed for `fab claim` and rich `fab ownership`. Opens a WebView; Epic auto-approves if your `auth login` session is still fresh. |
| `fabcli auth logout` | Invalidate both Epic and Fab sessions; delete token + WebView data folder |
| `fabcli auth status` | Check if Epic session is valid (headless) |
| `fabcli auth whoami` | Print account_id, display_name, email as JSON |

Two-step onboarding (once per ~90 days):
```bash
fabcli auth login       # Epic OAuth — unlocks search, library, download, ...
fabcli auth fab-login   # Fab web session — unlocks claim, rich ownership
```

`auth fab-login` skips the WebView if a fresh session already exists
(>7 days of validity). Pass `--force` to rerun anyway.

Check session before running commands:
```bash
fabcli auth status
# → {"authenticated":true,"expires_at":"...","refreshed":false}
```

## Commands

### Search

```bash
fabcli search --query "medieval kitbash" --channel unreal-engine --free
```

Flags:
- `-q, --query` — text search
- `--channel` — e.g. "unreal-engine"
- `--type` — e.g. "3d-model", "tool-and-plugin", "audio"
- `--category` — category filter
- `--sort` — `-relevance` (default), `-createdAt`, `createdAt`, `-price`, `price`. Leading `-` means descending (Fab API convention). Both `--sort=-createdAt` and `--sort "-createdAt"` work.
- `--count` — results per page
- `--cursor` — pagination cursor from previous results
- `--free` — only free assets
- `--discounted` — only discounted assets
- `--seller` — filter by seller name
- `--since <DATE|Nd>` — return only assets published on or after the cutoff. Accepts `YYYY-MM-DD`, RFC 3339, or relative `7d`/`30d`. Auto-paginates through Fab's cursors and stops at the first page whose oldest item falls before the cutoff — dramatically cutting API pages for "last week" / "this month" queries. Pair with `--sort=-createdAt` (or leave `--sort` unset) for early-stop to kick in; other sorts fall back to full-traversal with a stderr hint.

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

Reports whether a listing is owned. With a Fab session
(`auth fab-login` has been run), queries
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
UE5CLI integration. Shows progress on stderr.

Flags: `--artifact-id`, `--namespace`, `--asset-id` (required, from
manifest/library output), `-o, --output` (required), `--platform`
(optional), `--jobs` (default 8).

Response: `{"ok":true,"files":N,"total_bytes":M,"elapsed_seconds":T,"output_dir":"...","sidecar":".fabcli-asset.json"}`

## Fab Pricing Model

Fab's pricing has three "free" conditions — understanding them is
essential for correct `--free` filtering and `claim` behavior:

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

**The `--free` flag on `search`** sends `is_free=true` to the API AND
applies client-side filtering to also include `price==0` and
`discountedPrice==0` results. This catches all three free conditions.

**`claim`** uses the same three-condition check. Paid assets are
hard-blocked — the POST is never sent. The response includes the
price and a `purchase_url` for the user to buy manually.

**Note:** prices are in the user's local currency (determined by
their Epic account region), not USD.

## Common Recipes

### Recipe 1: Search and inspect an asset

```bash
# Search for assets
fabcli search -q "sci-fi props" --channel unreal-engine

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

# 3. If UE5CLI is available, install into project:
# ue5cli install-asset ./my-asset/ --project C:\Projects\MyGame
```

### Recipe 4: Browse free Unreal Engine assets

```bash
fabcli search --free --channel unreal-engine --sort -createdAt --count 20
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

## Recipes

> **Tip**: for any recipe that touches the library (directly or via
> `--from-library`), preface with `export FABCLI_LIBRARY_CACHE=1`
> so repeated library reads come from disk instead of re-paginating.
> See "Library cache (opt-in)" in the Library section for details.

### Find new free assets this week

```bash
fabcli search --free --since 7d
```

`--since 7d` auto-paginates and stops at the first page whose oldest
item is more than 7 days old, so you get just the recent slice
instead of the entire catalog. Combine with relative windows (`30d`,
`90d`) or ISO dates (`--since 2026-04-01`) as needed.

### Claim every new free asset this month

```bash
export FABCLI_LIBRARY_CACHE=1     # speed up any library reads below

fabcli search --free --since 30d \
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
  files in an output directory. Use UE5CLI (`ue5cli install-asset`)
  for automated project placement, or move files manually.
- **Does not manage Unreal Engine projects** — no project creation,
  plugin installation, or build management.
