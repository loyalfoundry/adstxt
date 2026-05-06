# Loyal `ads.txt` / `app-ads.txt`

This repo is the source of truth for the authorized-sellers file served at:

- `https://loyal.app/ads.txt`
- `https://loyal.app/app-ads.txt`

Both URLs serve the contents of [`ads.txt`](./ads.txt) in this repo. Although the file is named `ads.txt` for historical reasons, **its purpose is `app-ads.txt`** — it lists every SSP authorized to sell into the mobile and CTV apps that Loyal itself publishes.

## Background: what is this file?

`ads.txt` and `app-ads.txt` are IAB Tech Lab specifications that let publishers publicly declare which ad-tech vendors are authorized to monetize their inventory. Buyers (DSPs, agencies) and verification vendors (DoubleVerify, IAS, Pixalate) crawl these files to detect counterfeit inventory.

| File | Covers | Discovered by crawlers via |
|---|---|---|
| `ads.txt` | Web inventory on a publisher's domain | Fetching `<publisher-domain>/ads.txt` |
| `app-ads.txt` | Mobile-app and CTV-app inventory | Fetching `<developer-url>/app-ads.txt`, where `<developer-url>` is the publisher URL listed in each app's store listing (App Store, Play, Roku Channel Store, Fire TV, etc.) |

**Loyal's situation:**
- Loyal sells **no web inventory** on `loyal.app`. The `/ads.txt` URL ideally should not exist (it should 404). Currently it serves the same content as `/app-ads.txt`.
- Loyal sells **mobile and CTV inventory** through apps it publishes. Crawlers find this file because each Loyal-owned app's store listing points its publisher URL at `loyal.app`.
- Loyal also sells **CTV via syndication** — those inventory authorizations get listed in the *publisher's* `ads.txt` (not in this file).

## Authoritative spec

- **IAB Tech Lab Authorized Digital Sellers (ads.txt) Specification v1.1** — https://iabtechlab.com/wp-content/uploads/2022/04/Ads.txt-1.1.pdf
- The same spec governs `app-ads.txt`; the only difference is discovery (developer URL in app store listings).

You should read the spec before making non-trivial changes. The rules below are a summary, not a substitute.

## File format

Every non-blank, non-comment line is one of two things: a **record** (authorization) or a **variable**.

### Record lines

```
<exchange_domain>, <publisher_account_id>, <relationship>[, <tag_id>]
```

| Field | Required | Notes |
|---|---|---|
| `exchange_domain` | yes | Canonical domain of the SSP/exchange (e.g. `pubmatic.com`, `rubiconproject.com`). Lowercase. Must contain a TLD. |
| `publisher_account_id` | yes | The opaque ID the SSP assigned to Loyal's seat. Treated as a string; spec says case-sensitive but most SSPs treat as case-insensitive. |
| `relationship` | yes | Exactly `DIRECT` or `RESELLER` (uppercase canonical, but spec is case-insensitive). No other values, no annotations. |
| `tag_id` | optional but recommended | The TAG (Trustworthy Accountability Group) ID for the exchange — 16- or 32-char hex string. The same exchange always uses the same TAG-ID across all entries. |

Examples:
```
appnexus.com, 10239, RESELLER, f5ab79cb980f11d1
pubmatic.com, 163753, RESELLER, 5d62403b186f2ace
adform.com, 3035, DIRECT, 9f5210a2f0999e32
```

### Variable lines

Format: `name=value`. Spec defines a fixed set of variable names.

| Variable | Required | Purpose |
|---|---|---|
| `ownerdomain=` | recommended | Owner of the inventory — for Loyal: `loyal.app` |
| `contact=` | recommended | Email/URL for SSP/buyer disputes — for Loyal: `support@loyal.app` |
| `managerdomain=` | only if applicable | Domain of an external entity that monetizes inventory on the owner's behalf. Loyal does not currently use one. |
| `inventorypartnerdomain=` | only if applicable | For syndication relationships. Loyal does not currently use one. |
| `subdomain=` | only if applicable | Used for cross-subdomain authorizations. |

### Comments

Lines starting with `#` are comments. Inline comments after a `#` on a record line are also allowed. We use them in two ways:

- **Section headers** like `#33Across` to group related authorizations
- **`# REVIEW:` prefix** to flag lines that need human follow-up (see [Known issues](#known-issues--review-flags))

### Encoding

- ASCII or UTF-8
- LF line endings (no CRLF)
- No BOM
- Final newline at end of file

## Rules this repo follows

These are the conventions enforced during the May 2026 cleanup. Apply them to any new entries.

### Format rules (hard requirements)

1. Every record has 3 or 4 comma-separated fields. No more, no fewer.
2. Fields are separated by `, ` (comma + space) for readability. Either `,` or `, ` parses correctly.
3. `relationship` field is **uppercase** `DIRECT` or `RESELLER`. No suffixes (no ` - prebid`, no ` - oRTB`, no `DIRECT/RESELLER`).
4. `exchange_domain` is **lowercase** and includes a TLD (e.g. `epom.com`, not `EPOM`).
5. No record line contains two records concatenated together. One record per line.
6. No trailing whitespace on any line.

### Content rules

7. **No exact duplicates.** A record is a duplicate if `(domain, account_id, relationship, tag_id)` matches case-insensitively.
8. **No domain+account_id conflicts.** The same domain + account_id should not appear with both `DIRECT` and `RESELLER`. If it does, the data is contradictory and one is wrong.
9. **TAG-ID stays consistent per exchange.** All `pubmatic.com` lines share TAG-ID `5d62403b186f2ace`; all `rubiconproject.com` lines share `0bfd66d529a55807`; etc. New entries should use the same TAG-ID as the existing ones for that exchange.
10. **Variable lines come first**, immediately after the file header comments. Currently:
    ```
    #app-ads.txt Loyal
    #05.01.26 v1
    ownerdomain=loyal.app
    contact=support@loyal.app
    ```

### What to do with ambiguous lines

If you receive an entry from a mediation partner that you can't validate (unknown domain, malformed account ID, missing TAG-ID for an exchange that requires one), **prefix it with a `# REVIEW:` comment** explaining why, and comment out the original line:

```
# REVIEW: "eliteAppgrade" is not a known SSP domain — likely a typo, no obvious correction
# eliteAppgrade, 514554, RESELLER
```

This keeps the entry visible without breaking parsers.

## How the May 2026 cleanup worked

PR: https://github.com/loyalfoundry/adstxt/pull/1

Starting state: 3,497 lines, 3,441 records, 154,905 bytes.
Ending state: 2,385 lines, 2,322 records, 103,956 bytes (~33% smaller).

### Step 1 — Targeted line fixes (8 unambiguously broken lines)

| Before | After |
|---|---|
| `lumeriq.net 111327 RESELLER` | `lumeriq.net, 111327, RESELLER` |
| `freewheel.tv, 1585681 RESELLER` | `freewheel.tv, 1585681, RESELLER` |
| `yahoo.com, 58905, RESELLER e1a5b5b6e3255540` | `yahoo.com, 58905, RESELLER, e1a5b5b6e3255540` |
| `jio.com, 1547219295, RESELLER lemmatechnologies.com, 89, RESELLER, 7829010c5bebd1fb` | Split into two lines |
| `advlion.com, 584, RESELLERwaardex.com, 99236, RESELLER` | Split into two lines |
| `EPOM, 7e11d53b-..., Direct` | `epom.com, 7e11d53b-..., DIRECT` |
| `media.net, 8CU43768M, RESELLER - prebid` | `media.net, 8CU43768M, RESELLER` |
| `Media.net, 8CUGG4P7J, RESELLER - oRTB` | `media.net, 8CUGG4P7J, RESELLER` |

### Step 2 — Added contact line

Inserted `contact=support@loyal.app` immediately after `ownerdomain=loyal.app`.

### Step 3 — Flagged 3 lines for human review

See [Known issues](#known-issues--review-flags) below.

### Step 4 — Programmatic normalization

Single Python pass over the file:
- Stripped trailing whitespace from every line
- Normalized `Reseller`/`Direct` → `RESELLER`/`DIRECT` (9 lines)
- Deduped records by `(lower(domain), lower(account_id), upper(relationship), lower(tag_id))`, preserving first occurrence
- Preserved all comments, blanks, variable lines, and inline comments

Dropped: 1,111 exact duplicates + 7 case-only duplicates on account_id = **1,118 redundant rows**.

### Step 5 — Verification

Ran the validation commands below — confirmed 0 malformed lines and 0 remaining duplicates.

## How to manage this file going forward

### Adding new authorizations from a mediation partner

1. Receive the partner's snippet (usually pasted from email or a partner portal).
2. **Validate format before pasting:**
   - Each line has 3 or 4 comma-separated fields
   - Relationship is `DIRECT` or `RESELLER` (no suffixes)
   - Exchange domain is lowercase with a TLD
   - TAG-ID (if present) is hex and matches what we already use for that exchange
3. **Check for conflicts:** does the same domain+account_id already exist with the opposite relationship? If yes, talk to the partner before adding.
4. **Check for duplicates:** does the exact line already exist? If yes, do nothing.
5. Paste under an appropriate section header comment, or at the end of the file.
6. Run the [validation commands](#validation-commands) before committing.
7. Open a PR.

### Removing an authorization

1. Find the line(s) — usually `grep -n "<exchange-domain>" ads.txt`
2. Delete the line. Don't comment it out (keeps the file clean).
3. Validate, PR.

### Editing an existing entry

Generally avoid in-place edits — they're easy to get wrong. Prefer delete + add.

### Validation commands

Run all four before opening a PR. All should print zero hits.

```bash
F=ads.txt

# 1. Malformed records (must be 3 or 4 fields)
awk -F',' '
  /^[[:space:]]*$/ {next} /^[[:space:]]*#/ {next} /^[[:space:]]*[A-Za-z_]+=/ {next}
  { line=$0; sub(/[[:space:]]*#.*$/, "", line); n=split(line, a, ",")
    if (n<3||n>4) printf "MALFORMED L%d: %s\n", NR, $0 }' "$F"

# 2. Invalid relationship values
awk -F',' '
  /^[[:space:]]*$/ {next} /^[[:space:]]*#/ {next} /^[[:space:]]*[A-Za-z_]+=/ {next}
  { line=$0; sub(/[[:space:]]*#.*$/, "", line); n=split(line, a, ",")
    if (n>=3) {
      rel=a[3]; gsub(/^[[:space:]]+|[[:space:]]+$/, "", rel)
      if (rel != "DIRECT" && rel != "RESELLER")
        printf "BAD-REL L%d: \"%s\" :: %s\n", NR, rel, $0
    }}' "$F"

# 3. Duplicate records
awk '
  /^[[:space:]]*$/ {next} /^[[:space:]]*#/ {next} /^[[:space:]]*[A-Za-z_]+=/ {next}
  { line=$0; sub(/[[:space:]]*#.*$/, "", line); gsub(/[[:space:]]+/, "", line); line=tolower(line)
    if (line!="") c[line]++ }
  END { for (k in c) if (c[k]>1) printf "DUPE %dx %s\n", c[k], k }' "$F"

# 4. domain+account_id conflicts (same pair, both DIRECT and RESELLER)
awk -F',' '
  /^[[:space:]]*$/ {next} /^[[:space:]]*#/ {next} /^[[:space:]]*[A-Za-z_]+=/ {next}
  { line=$0; sub(/[[:space:]]*#.*$/, "", line); n=split(line, a, ",")
    if (n>=3) {
      d=tolower(a[1]); id=tolower(a[2]); rel=toupper(a[3])
      gsub(/[[:space:]]/, "", d); gsub(/[[:space:]]/, "", id); gsub(/[[:space:]]/, "", rel)
      key=d","id
      if (rels[key] && rels[key]!=rel) printf "CONFLICT %s rels: %s vs %s\n", key, rels[key], rel
      rels[key]=rel
    }}' "$F"
```

### Online validators

After merging to `master`, also run the file through:
- IAB Tech Lab validator: https://iabtechlab.com/ads-txt-validator/
- adstxt.guru (free): https://adstxt.guru/

These catch subtle issues the awk scripts miss (e.g. duplicate variable lines, unknown variable names).

## Hosting

**Current setup (problematic):**

```
https://loyal.app/ads.txt
  → 301 → https://www.loyal.app/ads.txt
  → 301 → https://raw.githubusercontent.com/loyalfoundry/adstxt/refs/heads/master/ads.txt
```

This means commits to `master` here update the live file immediately, with no deploy step.

**Why this is a problem:** the IAB spec requires redirects to stay on the same root domain. `raw.githubusercontent.com` is a different root, so spec-compliant crawlers (Google AdX, IAB Tech Lab validator, most CTV verification vendors) treat the file as missing. This silently breaks SSP authorization in production.

**Fix needed:** serve the file directly from `loyal.app` — most likely via a Cloudflare Worker that proxies the GitHub raw file. This preserves the git workflow but makes the file IAB-compliant.

## Known issues / REVIEW flags

### Lines flagged in this repo with `# REVIEW:` (need human follow-up)

```
# REVIEW: "eliteAppgrade" is not a known SSP domain — likely a typo, no obvious correction
# eliteAppgrade, 514554, RESELLER

# REVIEW: same account_id+TAG-ID as lemmatechnologies.com — almost certainly a typo for "lemmatechnologies.com"
# emmatechnologies.com, 89, RESELLER, 7829010c5bebd1fb

# REVIEW: malformed line — looks like a stray TAG-ID glued to an incomplete e-volution.ai entry; original intent unclear
# 03113cd04947736dssp.e-volution.ai, AJxF6R378a9M6CaTvK
```

If you can identify the source partner for any of these, contact them for the correct entry and replace the `# REVIEW:` block with the corrected line.

### 8 DIRECT/RESELLER conflicts (need business decision)

The same domain+account_id is currently authorized as both DIRECT and RESELLER. One is wrong. To resolve, decide whether Loyal has a direct seat with that SSP for that account, or sells through a reseller, then delete the other line.

```
adform.com, 3035             DIRECT and RESELLER
appnexus.com, 16912          DIRECT and RESELLER
conversantmedia.com, 100141  DIRECT/RESELLER (also has invalid relationship value)
onetag.com, 61d88450bdb25bc  DIRECT and RESELLER
smartadserver.com, 4717      DIRECT and RESELLER
video.unrulymedia.com, 8167205979129043832  DIRECT and RESELLER
yahoo.com, 58905             RESELLER and RESELLERE1A5B5B6E3255540 (parse bug)
```

### `loyal.app/ads.txt` should 404

Loyal sells no web inventory. The `/ads.txt` URL currently serves the app-ads content, which advertises web authorizations Loyal does not actually use. Verification vendors can flag this. Fix is at the hosting layer (Cloudflare/Webflow), not in this repo.

## Repo layout

```
.
├── README.md   (this file)
└── ads.txt     (the served file)
```

That's it. There's intentionally no build step, no CI, no formatter — the file is hand-edited and the validation commands above are the test suite.
