# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Read this first, then the detailed guide

`SPORTSDB_ICON_LIBRARY.md` is the long-form guide to expanding this library: catalog schema, name-matching rules, TheSportsDB endpoints, a work queue of sports to add, and a validation checklist. Use it for all of that.

This file covers only what that document doesn't: **this repo has more than one consumer, and they disagree.** Everything in `SPORTSDB_ICON_LIBRARY.md` describes the *Playfy Web* resolver — including its uses of "this repo", which mean Playfy Web, not the library. Applying some of its advice verbatim to the shared data breaks the other consumer.

## What this repo is

312 PNG badge/icon files plus `catalog.json`, the manifest indexing them. No source code, build system, package manager, tests, or CI — only data and images. Licensed AGPL-3.0; the badges are third-party trademarks sourced from TheSportsDB.

Scope: 4 sports, 12 leagues, 289 team entries.

## `main` is production

Consumers fetch this repo over raw GitHub at runtime. There is no version, tag, or content hash in the URL, so **a push to `main` immediately changes what already-installed apps render.** Renaming or deleting a PNG breaks live clients with no release step in between.

```
https://raw.githubusercontent.com/LosLonelyDevs/sportsdb-icon-library/main
```

## The two consumers

| | Playfy Web | GAYERFy-TV |
|---|---|---|
| Stack | React | Kotlin Multiplatform (Android + desktop) |
| Resolver | `src/services/logoResolver.js` | `shared/src/commonMain/kotlin/tv/gayerfy/shared/logos/team-logos.kt` |
| Local path | not checked out here | `/home/franco/code/GAYERFy-TV` |
| Reads | `team` and `league` only | teams only (`/teams/` paths; league badges and sport icons ignored) |
| Match | exact on `compactText()`, sport-gated | exact slug → noise-stripped slug → unique substring |
| `strTeamAlternate` | whole string, **not** comma-split | **comma-split** |
| Fallback when missed | bundled catalog → live API → emoji | live TheSportsDB API → sport emoji |

Only GAYERFy-TV is verifiable from this machine; the Playfy Web column is as described by `SPORTSDB_ICON_LIBRARY.md`.

### GAYERFy-TV specifics

Fetches `<base>/catalog.json` once per process, builds a matcher, returns `<base>/<strBadge>` for Coil. The base URL is hardcoded in three places: `androidApp/build.gradle.kts` (`SPORTS_DB_LOGO_LIBRARY_BASE`), `desktopApp/src/jvmMain/kotlin/tv/gayerfy/desktop/TeamLogos.kt`, and test assertions in `shared/src/commonTest/.../team-logo-matcher-test.kt`.

## Three traps unique to the multi-consumer setup

**1. Comma lists in `strTeamAlternate` are load-bearing.** `SPORTSDB_ICON_LIBRARY.md` §5 says comma lists are useless and to spend the two alias slots carefully — true for Playfy Web, but GAYERFy-TV splits on commas, and **92 of 289 entries carry comma lists**. Never strip them to satisfy that advice; it degrades a shipped app.

The agreed fix is the other direction: **split on commas in Playfy Web's `logoResolver.js`** (still pending — that repo isn't checked out here), which gains it ~200 exact-match aliases. Bump `LOOKUP_CACHE_VERSION` when doing it or stale `localStorage` masks the change for 7 days.

Groundwork already landed: three ambiguous nicknames that would have made splitting unsafe (`Bianconeri` → Juventus/Udinese, `Giallorossi` → Roma/Lecce, `AJA` → Auxerre/Ajax) are removed from `catalog.json` and denylisted via `AMBIGUOUS_ALIASES` in GAYERFy-TV's generator. Same-sport cross-team collisions are now **1 with or without splitting** (`int` → Inter Milan / Internacional, pre-existing and untouched). Add to that denylist rather than hand-editing if more generic nicknames appear.

**2. Filenames are a match key, not just storage.** GAYERFy-TV's matcher re-implements the slug rules in Kotlin and falls back to matching the *filename* when catalog-name lookup misses; filename slugs deliberately **override** catalog aliases on collision. Consequences:

- Renaming a PNG can change which team resolves even with `catalog.json` updated correctly.
- Adding a team whose noise-stripped slug is a substring of an existing one can turn a previously unique substring match ambiguous, making a **working** logo vanish. Noise tokens stripped from both sides: `fc`, `cf`, `sc`, `afc`, `ac`, `cd`, `club`, `de`.

**3. `paris-sg.png` must not be deleted.** GAYERFy-TV hardcodes a `builtinAliases` map pointing `"Paris Saint-Germain"` and `"PSG"` at `soccer/french-ligue-1/teams/paris-sg.png` — a path absent from `catalog.json`. Any "remove unreferenced assets" cleanup breaks PSG logos in production.

## The generator lives elsewhere — hand-edits are temporary

Every PNG and `catalog.json` itself is output of `scripts/generate-sportsdb-logo-catalog.mjs` + `scripts/priority-league-seeds.mjs`. **This repo contains neither.** The current `catalog.json` is byte-identical to the one deleted from GAYERFy-TV in commit `e2d4c96` ("Remove logos") — that commit is the split that created this repo.

Two copies of that generator now exist (GAYERFy-TV's and the one `SPORTSDB_ICON_LIBRARY.md` §9 describes in Playfy Web), and they **disagree on `strBadge` format**: this repo requires repo-relative paths with no leading slash, while Playfy Web's writes `/sportsdb-logos/...`. Any regeneration must say which copy ran.

Practical rules:

- A hand-edit to `catalog.json` **forks it from its generator** and is overwritten by the next regeneration, which rebuilds the file from scratch. Durable fixes go in `priority-league-seeds.mjs` (add an alias to a team's array).
- GAYERFy-TV's script writes to `assets/sportsdb-logos/`, which no longer exists there and is *not* gitignored — regenerating recreates a full duplicate tree in that repo. Move output here; don't commit it there.
- `downloadAsset()` skips files already on disk, so re-running never refreshes existing badges. Delete a PNG to force a re-fetch.
- Regeneration is slow and network-bound (1.8 s between calls, backoff on 429) and needs `SPORTS_DB_BASE`/`SPORTS_DB_API_KEY`. `missingTeams` entries are free-tier API resolution failures, **not** missing artwork.

## Catalog invariants worth knowing

`targetLeagues[]` and `coverage[]` are denormalized views of data that also lives in `sports[].leagues[]`. Editing a league's expected/resolved/missing counts requires mirroring the change in all three, or the file contradicts itself.

Diacritics are folded to ASCII rather than stripped (`Köln` → `koln.png`, `São Paulo` → `sao-paulo.png`), so naive `[^a-z0-9]+ → -` slugification of `strTeam` will not reproduce 17 of the filenames. Always read the path from `catalog.json`.

21 clubs are nested under both their domestic league and UEFA Champions League with byte-identical PNGs in each league directory (29 duplicate file pairs). Every duplicate involves UCL. This contradicts `SPORTSDB_ICON_LIBRARY.md` §7's "never duplicate" rule — that rule governs new work; the existing duplicates are reachable match keys and can't be pruned without coordinating both consumers.

## Verifying changes

No linter or test suite exists. `SPORTSDB_ICON_LIBRARY.md` §10 has `jq`-based checks for JSON validity, path format, and missing fields. Beyond those, reconcile catalog against disk:

```bash
python3 - <<'EOF'
import json, os
d = json.load(open('catalog.json'))
ref = set()
for sp in d['sports']:
    if sp['strSportIconGreen']: ref.add(sp['strSportIconGreen'])
    for lg in sp['leagues']:
        ref.add(lg['strBadge'])
        ref.update(t['strBadge'] for t in lg['teams'])
disk = {os.path.relpath(os.path.join(r, f), '.')
        for r, _, fs in os.walk('.') if '.git' not in r
        for f in fs if f.endswith('.png')}
print('in catalog, missing on disk:', sorted(ref - disk))
print('on disk, unreferenced:', sorted(disk - ref))
EOF
```

11 PNGs are currently unreferenced and that is **expected** — see trap 3 above before deleting any of them. They are alias duplicates (`fc-koln.png` alongside the referenced `koln.png`, etc.), the two API-gated teams (`nottingham-forest.png`, `paris-sg.png`), and `soccer/sport-icon.png` (referenced via `strSportIconGreen`, not `strBadge`).

Asset norm is 512×512 RGBA with transparency; current outliers are one 2560×2560, one 1024×1024, one 256×256, fifteen 500×500, and the 64×64 sport icon.

```bash
file $(find . -name '*.png' -not -path './.git/*') | sed 's/.*PNG image data, //' | cut -d, -f1 | sort | uniq -c
```

## Note on AGENTS.md

`AGENTS.md` defers to `.agents/AGENTS.md` as authoritative, but that path does not exist in this repo and is gitignored. Its fallback mentions `README.md` and `package.json` scripts, neither of which exists here either. Treat `SPORTSDB_ICON_LIBRARY.md` plus this file as the actual guidance.
