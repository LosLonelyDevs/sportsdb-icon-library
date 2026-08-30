# Populating `sportsdb-icon-library`

Agent guide for expanding <https://github.com/LosLonelyDevs/sportsdb-icon-library> so that
Playfy Web resolves logos from it correctly.

**Audience:** an AI agent with write access to the icon-library repo.
**Goal:** add sports / leagues / teams to `catalog.json` + committed PNG assets, using
field names and values that this app's resolver will actually match.

> ⚠ **Scope caveat added after this document landed in the library repo.** Everything below
> describes the **Playfy Web** resolver (`src/services/logoResolver.js`) — note that "this
> repo" throughout the text means Playfy Web, not the icon library you are editing.
>
> Playfy Web is **not the only consumer.** At least one other client also reads this catalog
> from `main` at runtime, and its matcher differs in ways that make some advice here
> actively unsafe to apply to the shared data — notably comma handling in
> `strTeamAlternate` (§5) and reliance on `/teams/` **filenames** as match keys (§8).
> **Read `CLAUDE.md` before editing `catalog.json` or renaming any asset.**

---

## 1. Why this repo exists

`src/services/logoResolver.js` resolves a logo for three entity kinds — `sport`, `league`,
`team` — from an event's text labels. It tries sources in a fixed order, and the icon
library is the first *remote* source:

| # | Source | Kind | Match style |
|---|--------|------|-------------|
| 1 | `NATIONAL_TEAM_BADGES` (hardcoded in resolver) | team | exact |
| 2 | `src/data/scraped-team-logos.json` | team | exact |
| 3 | **`sportsdb-icon-library` → `catalog.json`** | **team, league** | **exact** |
| 4 | `src/data/sportsdb-logo-catalog.json` (bundled) | sport, league, team | fuzzy (Fuse.js) |
| 5 | TheSportsDB live API | sport, league, team | fuzzy (Fuse.js) |
| 6 | `fallbackUrl` from the event feed | any | n/a |

Two things follow from this table, and they drive every rule in this document:

- **The icon library is only consulted for `team` and `league`.** `sport` icons are never
  read from it today (`logoResolver.js` calls `resolveIconLibraryAsset` only when
  `entityType` is `team` or `league`). Adding `strSportIconGreen` art is harmless and
  future-proof, but it will not render until the resolver is changed.
- **Icon-library matching is EXACT, not fuzzy.** Unlike sources 4 and 5, there is no
  Fuse.js scoring here. A name either normalizes to a byte-identical string or it misses
  entirely. This makes aliases (§5) the single most important field you populate.

The payoff for adding an entry: it wins over the live TheSportsDB API, which uses the
shared free key `123` and is aggressively rate-limited. Every team you add here is one
fewer network round-trip and one fewer rate-limit miss for users.

---

## 2. Current state of the library

As of the last check the repo already contains the same seed set as this app's bundled
catalog — do not re-add these, extend past them:

```
Root:
  catalog.json
  soccer/            american-football/            basketball/            ice-hockey/
```

| Sport (`strSport`) | Leagues | Teams |
|---|---|---|
| Soccer | MLS, English Premier League, UEFA Champions League, Spanish La Liga, German Bundesliga, Italian Serie A, French Ligue 1, Brazilian Serie A, Mexican Primera League | ~200 |
| American Football | NFL | 32 |
| Basketball | NBA | 30 |
| Ice Hockey | NHL | 32 |

Total 289 teams across 12 leagues. The gaps actually recorded in the shipped
`catalog.json` `missingTeams` arrays are only these four names:

| League | `missingTeams` |
|---|---|
| American Major League Soccer | `St. Louis CITY SC` |
| English Premier League | `Nottingham Forest` |
| UEFA Champions League | `Paris Saint-Germain`, `Union Saint-Gilloise` |
| French Ligue 1 | `Paris Saint-Germain` |

Note that `nottingham-forest.png` and `paris-sg.png` **already exist as committed
assets** — they are unreferenced by `catalog.json` because the free-tier API failed to
resolve their metadata, not because artwork is missing. Adding those two entries (with
real `idTeam` values) is the cheapest coverage win available.

---

## 3. WORK QUEUE — sports to add

The app can emit **13** canonical sports. The library covers **4**. These are the 9 that
are missing — this is the work.

Every `strSport` below was verified against TheSportsDB via `lookupleague.php`, not
guessed. Use these exact strings.

| App label | `strSport` to use | Directory slug | Priority | Notes |
|---|---|---|---|---|
| Cricket | `Cricket` | `cricket/` | **high** | feeds emit this often |
| Combat | `Fighting` | `fighting/` | **high** | name differs — app already maps it |
| Baseball | `Baseball` | `baseball/` | medium | |
| Motorsport | `Motorsport` | `motorsport/` | medium | league badge only — see §3.2 |
| Tennis | `Tennis` | `tennis/` | medium | league badge only — see §3.2 |
| Rugby | `Rugby` | `rugby/` | medium | |
| Golf | `Golf` | `golf/` | low | league badge only — see §3.2 |
| Darts | `Darts` | `darts/` | low | thin upstream coverage; individual sport |
| WWE | **undecided — do not add yet** | — | blocked | see §3.3 |

Already present, but worth extending: `Soccer` (add FIFA World Cup, id `4429`),
`American Football` (add CFL, id `4405`).

### 3.1 Verified league IDs to start from

Pass these straight to the endpoints in §9. They are real and were confirmed to resolve:

| Sport | League | `idLeague` |
|---|---|---|
| Cricket | Indian Premier League | `4460` |
| Fighting | UFC | `4443` |
| Baseball | MLB | `4424` |
| Motorsport | Formula 1 | `4370` |
| Rugby | European Rugby Champions Cup | `4550` |
| Golf | PGA Tour | `4425` |
| Tennis | ATP World Tour | `4464` |
| Soccer | FIFA World Cup | `4429` |
| American Football | CFL | `4405` |
| Basketball | EuroCup Basketball | `4547` |

### 3.2 Individual sports: ship the LEAGUE badge, not teams

For Motorsport, Golf, Tennis and Fighting, do **not** expect `search_all_teams.php` to
return a useful roster — these are individual-competitor sports. The app also skips team
logos entirely for them (`NON_TEAM_SPORTS` in `src/components/EventCard.jsx` covers
motorsport, formula1, motogp, golf).

So for these, the **league badge is the only thing that renders**. A league entry with a
good `strBadge` and zero teams is a complete, correct contribution. Set
`expectedTeamCount: 0` and leave `teams: []`.

### 3.3 WWE is blocked — do not guess

The app maps `wwe` / `aew` / `wrestling` / `nxt` → `WWE`, but **TheSportsDB has no
matching sport**: its `Fighting` is MMA/boxing, and there is no "Wrestling" sport. Adding
pro wrestling under an invented `strSport` would silently fail the sport gate (§7).

If you want to add it, **pick a `strSport` and say so in your PR description** — the app
needs a matching `ICON_LIBRARY_SPORT_SYNONYMS` entry landed alongside it. Do not add it
unilaterally.

### 3.4 ⚠ `all_sports.php` is effectively broken on the free key

The shared key `123` currently returns only **2 sports** (`Soccer`, `Motorsport`) from
`all_sports.php`. This is the complete response, not truncation — do not treat it as a
network error or waste retries on it.

**You cannot enumerate sports.** Use `lookupleague.php?id=<id>` with the known IDs in
§3.1; the response's `strSport` field is the authoritative sport name.

---

## 4. `catalog.json` — exact expected format

The app fetches, unauthenticated, from the `main` branch:

```
https://raw.githubusercontent.com/LosLonelyDevs/sportsdb-icon-library/main/catalog.json
```

Assets resolve against the same base, so **`strBadge` must be a repo-relative path**, not
an absolute URL and not a leading-slash path:

```
https://raw.githubusercontent.com/LosLonelyDevs/sportsdb-icon-library/main/ + <strBadge>
```

Three-level nesting: `sports[] → leagues[] → teams[]`.

```jsonc
{
  "generatedAt": "2026-07-18T06:39:37.995Z",   // ISO 8601, informational
  "source": "TheSportsDB + web-curated league seeds",
  "targetLeagues": [ /* informational: what the generator aimed to cover */ ],
  "coverage": [ /* informational: per-league expected/resolved/missing */ ],

  "sports": [
    {
      "idSport": "102",                          // TheSportsDB id; any stable string ok
      "strSport": "Soccer",                      // MUST be a TheSportsDB sport name — see §6
      "strSportIconGreen": "soccer/sport-icon.png",   // repo-relative; not yet consumed
      "strSportThumb": "https://www.thesportsdb.com/images/sports/soccer.jpg",

      "leagues": [
        {
          "idLeague": "4346",
          "strLeague": "American Major League Soccer",  // canonical league name
          "strLeagueAlternate": "MLS",                  // comma-separated aliases — CRITICAL
          "strSport": "Soccer",                         // must equal parent strSport
          "strBadge": "soccer/american-major-league-soccer/league-badge.png",
          "source": "https://www.mlssoccer.com/clubs/", // where the roster came from
          "expectedTeamCount": 30,
          "resolvedTeamCount": 29,
          "missingTeams": ["St. Louis CITY SC"],

          "teams": [
            {
              "idTeam": "135851",
              "strTeam": "Atlanta United",               // canonical team name
              "strTeamAlternate": "Atlanta Utd",         // alias — CRITICAL
              "strTeamShort": "ATL",                     // alias — CRITICAL
              "strLeague": "American Major League Soccer",
              "strSport": "Soccer",
              "strBadge": "soccer/american-major-league-soccer/teams/atlanta-united.png"
            }
          ]
        }
      ]
    }
  ]
}
```

### Fields the resolver actually reads

Everything else is documentation for humans. The resolver only reads these:

| Level | Field | Required | Used for |
|---|---|---|---|
| sport | `strSport` | **yes** | context gate on both team and league lookups |
| league | `strLeague` | **yes** | primary match key; skipped entirely if missing |
| league | `strLeagueAlternate` | strongly recommended | alias match key |
| league | `strBadge` | **yes** | the image; a league with no badge is dropped |
| team | `strTeam` | **yes** | primary match key; team dropped if missing |
| team | `strTeamAlternate`, `strTeamShort` | strongly recommended | alias match keys |
| team | `strLeague` | — | read from the **parent league**, not the team object |
| team | `strBadge` | **yes** | the image; a team with no badge is dropped |

Note the last two rows. In `ensureIconLibraryLoaded()` the record is built as
`leagueName: league.strLeague` and `sportName: sport.strSport` — taken from the **parent
objects**. A team's own `strLeague`/`strSport` keys are ignored by the resolver, so
nesting a team under the wrong league silently breaks its context gate (§7) even if the
team's own fields are right. **Nesting placement is what matters.**

Also: `if (!team.strTeam || !team.strBadge) continue;` — a team missing either field is
dropped at load with no warning. Same for leagues. Never commit a placeholder empty
string to "reserve" a slot; omit the entry and list it in `missingTeams` instead.

---

## 5. Name matching — the rule that decides everything

The resolver normalizes both sides with `compactText()` before comparing, then requires
**exact equality**:

```js
normalizeText(v) =
  v.normalize('NFKD')            // decompose accents
   .replace(/[̀-ͯ]/g,'')   // strip diacritics
   .replace(/&/g, ' and ')           // & → " and "
   .replace(/['’.]/g, '')            // drop apostrophes and periods
   .replace(/[^a-zA-Z0-9]+/g, ' ')   // everything else → space
   .toLowerCase().trim();

compactText(v) = normalizeText(v).replace(/\s+/g, '');   // then remove ALL spaces
```

So punctuation, case, accents, and spacing are already free. These all collapse to the
same key and do **not** need separate aliases:

| Written | `compactText` |
|---|---|
| `Atlético Madrid` / `Atletico Madrid` | `atleticomadrid` |
| `D.C. United` / `DC United` | `dcunited` |
| `Brighton & Hove Albion` | `brightonandhovealbion` |
| `Bodø/Glimt` | `bodoglimt` |
| `1. FC Köln` | `1fckoln` |

What is **not** free — and is exactly what aliases are for — is any change in *words*:

| Feed says | Canonical | Free? | Action |
|---|---|---|---|
| `Man Utd` | `Manchester United` | ✗ | add to `strTeamAlternate` |
| `Spurs` | `Tottenham Hotspur` | ✗ | add alias |
| `PSG` | `Paris Saint-Germain` | ✗ | add alias |
| `Inter Miami CF` | `Inter Miami` | ✗ | add alias (`CF` is a *word*) |
| `LAFC` | `Los Angeles Football Club` | ✗ | add alias |
| `Wolves` | `Wolverhampton Wanderers` | ✗ | add alias |

> `strTeamAlternate` and `strLeagueAlternate` are read as **single strings**, and the
> resolver puts each in the alias list whole — it does **not** split on commas. Writing
> `"Man Utd, Man United"` creates one useless alias `manutdmanunited`. There are only two
> alias slots per team (`strTeamAlternate`, `strTeamShort`), so spend them on the two
> highest-value variants and push the rest into this app's `TEAM_ALIASES` map (§9).
> (For contrast, the *fuzzy* sources at steps 4–5 do split `strLeagueAlternate` on
> commas — so comma lists are only harmful here, in the exact-match path.)

> 🚨 **Do not act on the paragraph above by removing commas from `catalog.json`.**
> Another consumer of this repo *does* split `strTeamAlternate` on commas, and **92 of the
> 289 team entries carry comma lists** — e.g. `New York City FC` →
> `"New York City, New York City Football Club, NYCFC"`. Those lists are load-bearing
> there: flattening them to a single variant would silently drop working aliases from a
> shipped app to gain nothing but tidiness here.
>
> **The agreed resolution is to split on commas in Playfy Web's `logoResolver.js`** — one
> edit in one app — rather than degrading shared data for both. That converts the 92 lists
> from inert to useful, gaining Playfy Web **~200 additional exact-match aliases**
> (`1fckoln`, `afcajax`, `asroma`, `athleticclub`, `nycfc`, …).
>
> Two things to know when you make that change:
>
> 1. **Bump `LOOKUP_CACHE_VERSION`** (§9), or returning users keep stale `localStorage`
>    results for 7 days and the change looks like a no-op.
> 2. **The collision risk has already been removed from the data.** Splitting naively used
>    to introduce three same-sport ambiguous keys (`bianconeri` → Juventus/Udinese,
>    `giallorossi` → Roma/Lecce, `aja` → Auxerre/Ajax). Those nicknames have been stripped
>    from `catalog.json` and are denylisted in the generator, so same-sport cross-team
>    collisions stay at **1 before and after** splitting (`int` → Inter Milan /
>    Internacional, pre-existing). Splitting is now collision-neutral.
>
> See `CLAUDE.md` for the full cross-consumer picture.

### Feed vocabulary you are matching against

Names come from live stream feeds (PlayZ, SportzX, LivXow), not from TheSportsDB, so they
are messy and abbreviated. Before inventing aliases, sample the real strings:

```bash
# in this repo — see what the feeds actually emit
grep -rn "normalizePlayZEvent\|normalizeSportzXEvent" src/services/api.js
```

Priority order for aliases: whatever the **feed** calls the team first, the official name
second.

---

## 6. Sport and league naming — use TheSportsDB's vocabulary

**Use TheSportsDB's canonical `strSport` values**, i.e. what
`https://www.thesportsdb.com/api/v1/json/123/all_sports.php` returns:

```
Soccer   American Football   Basketball   Ice Hockey   Motorsport   Baseball
Cricket  Tennis   Rugby   Golf   Darts   Fighting   ...
```

Do **not** invent names, and do **not** use this app's display vocabulary.

### ⚠ Known mismatch — read before adding any sport

This app has its own display taxonomy in `src/components/eventUtils.js` (`canonicalSport`),
which is deliberately **not** TheSportsDB's. Notably:

| App's canonical sport | TheSportsDB `strSport` | Same? |
|---|---|---|
| `Football` (means soccer) | `Soccer` | **✗ differ** |
| `Hockey` | `Ice Hockey` | **✗ differ** |
| `Combat` | `Fighting` | **✗ differ** |
| `NBA` | `Basketball` | **✗ differ** |
| `American Football` | `American Football` | ✓ |
| `Cricket`, `Tennis`, `Rugby`, `Golf`, `Darts`, `Motorsport`, `Baseball` | same | ✓ |

**This is handled on the app side — you do not need to work around it.**
`logoResolver.js` folds both vocabularies onto a shared key via
`ICON_LIBRARY_SPORT_SYNONYMS` / `compactIconLibrarySport()` before the sport gate runs, so
an event labelled `Hockey` matches a library record labelled `Ice Hockey`.

What this means for you:

1. **Keep using TheSportsDB names in `catalog.json`.** They are correct and consistent with
   the existing 289 entries. Do **not** "fix" this by renaming `Soccer` → `Football` in the
   library — that would break the league path and the bundled-catalog path, and would now
   also fight the synonym map.
2. **If you add a sport whose TheSportsDB name differs from the app's label, say so in your
   PR.** The synonym map needs a matching entry in `src/services/logoResolver.js`, or the
   sport gate will reject that sport's teams. Currently mapped: soccer, ice hockey,
   basketball, american football, fighting, baseball, motorsport.

A genuinely different sport still correctly fails the gate — the synonym map only unifies
names for the *same* sport, so a Baseball "Arizona Cardinals" can never satisfy an
American Football lookup.

### League names

Use TheSportsDB's `strLeague` verbatim, including its quirky prefixes — e.g.
`American Major League Soccer` (not `MLS`), `Spanish La Liga` (not `La Liga`),
`English Premier League`, `Mexican Primera League`. Then put the short/common form the
feeds actually use in `strLeagueAlternate` (`MLS`, `La Liga`, `EPL`, `Liga MX`).

---

## 7. League/sport context gates

`resolveIconLibraryAsset` applies one context filter after the name match:

```js
if (!names.includes(normalizedInput)) return false;              // name must match (exact)
if (normalizedSport && candidate.sportName &&
    compactIconLibrarySport(candidate.sportName) !== normalizedSport) return false;  // sport gate
```

**There is no league gate.** It was removed deliberately: the library nests a club under
its **domestic** league, while an event's league is the **competition** it is playing in,
and those routinely differ.

- Real Madrid in a Champions League fixture → feed league `UEFA Champions League`, but the
  team is nested under `Spanish La Liga`. This now resolves correctly.

This mirrors the escape hatch the fuzzy path already had via `hasExactNameMatch` — an
exact hit on a highly specific team name is evidence enough on its own, so a mismatched
competition should not veto it.

What this means for populating:

- **Nest a club under its domestic league** (its TheSportsDB primary league). This is now
  unambiguously correct, and it is what the fuzzy fallback expects too, so both paths stay
  consistent.
- **Never duplicate a club under every cup it plays in.** It bloats the catalog, and since
  `records.find()` returns the first hit, duplicates make the winner arbitrary.

  > ⚠ **The shipped catalog already violates this rule.** 21 clubs are nested under both
  > their domestic league *and* `UEFA Champions League` (Arsenal, Chelsea, Liverpool,
  > Manchester City, Real Madrid, …), each with a byte-identical PNG committed under both
  > league directories — 29 duplicate file pairs in total. Every duplicate involves UCL; no
  > other pair of leagues duplicates.
  >
  > So this is a rule for *new* work, not a description of current state. Do not "fix" the
  > existing duplicates by deleting one side: the `/teams/` filenames are a match key for
  > another consumer (see `CLAUDE.md`), so removing a path that resolver can reach breaks
  > logos in a shipped app. Deduping requires coordinating both consumers.
- **Do** still add cup/continental competitions as **league entries** with a badge —
  league lookups are how `UEFA Champions League` gets its own artwork.

---

## 8. Assets — layout, format, size

Directory layout mirrors the catalog nesting, all slugified
(`lowercase`, non-alphanumerics → `-`, trimmed):

```
<sport-slug>/sport-icon.png
<sport-slug>/<league-slug>/league-badge.png
<sport-slug>/<league-slug>/teams/<team-slug>.png
```

Concretely:

```
soccer/sport-icon.png
soccer/english-premier-league/league-badge.png
soccer/english-premier-league/teams/manchester-united.png
american-football/nfl/teams/arizona-cardinals.png
```

### Worked example — one team, end to end

The NFL team **Arizona Cardinals**, as it exists in the library today:

```jsonc
// catalog.json → sports[strSport="American Football"] → leagues[strLeague="NFL"] → teams[]
{
  "idTeam": "134946",
  "strTeam": "Arizona Cardinals",          // canonical name → drives the slug
  "strTeamAlternate": "Cardinals",         // alias: feeds often drop the city
  "strTeamShort": "ARI",                   // alias: feeds often use the 3-letter code
  "strLeague": "NFL",
  "strSport": "American Football",
  "strBadge": "american-football/nfl/teams/arizona-cardinals.png"
}
```

| | |
|---|---|
| Committed file | `american-football/nfl/teams/arizona-cardinals.png` |
| Browsable (GitHub UI) | `https://github.com/LosLonelyDevs/sportsdb-icon-library/blob/main/american-football/nfl/teams/arizona-cardinals.png` |
| **What the app fetches** | `https://raw.githubusercontent.com/LosLonelyDevs/sportsdb-icon-library/main/american-football/nfl/teams/arizona-cardinals.png` |
| Verified | HTTP 200 · 512×512 · 8-bit RGBA PNG · ~43 KB |

Note the slug comes from `strTeam` (`Arizona Cardinals` → `arizona-cardinals`), **not** from
`strTeamAlternate` or `strTeamShort` — those exist purely to be matched against, never to
name files.

> ⚠ **Never put a `github.com/.../blob/...` URL in `strBadge`.** That is the HTML page for
> the file, not the file: it returns `text/html`, so the `<img>` fails and the app silently
> falls back to an emoji. `strBadge` is always a repo-relative path (§4), and the app
> prepends the `raw.githubusercontent.com` base itself. If you are copying paths out of the
> GitHub web UI, strip the leading
> `https://github.com/LosLonelyDevs/sportsdb-icon-library/blob/main/`.

The slug is derived from the **resolved canonical name** (`strTeam`), not the alias:

```js
slugify(v) = v.normalize('NFKD').replace(/[̀-ͯ]/g,'')
              .replace(/[^a-zA-Z0-9]+/g,'-').replace(/^-+|-+$/g,'').toLowerCase();
```

Rules:

- **PNG, transparent background.** Badges are rendered over card backgrounds in both
  light and dark themes — a white matte will look like a bug.
- Square-ish, **≥128px**, ideally 256px. These render small (card logos) but on HiDPI.
- Keep files lean — the whole repo is served over `raw.githubusercontent.com` with no CDN
  in front. Run PNGs through `oxipng`/`pngquant` before committing.
- **`strBadge` path must exactly equal the committed file path**, including case. Raw
  GitHub is case-sensitive; a `Teams/` vs `teams/` mismatch 404s and the app falls back
  silently, which is painful to debug.

Source images from TheSportsDB (`strBadge`, `strLogo`, `strTeamBadge`), which is what the
existing entries did. Respect its terms of use and keep attribution in the repo README.

---

## 9. Recommended workflow for adding a league

1. **Pick the league and get its TheSportsDB id.**

   ```bash
   curl -s "https://www.thesportsdb.com/api/v1/json/123/all_leagues.php" \
     | jq '.leagues[] | select(.strLeague|test("Eredivisie";"i"))'
   ```

2. **Pull league details (for the badge and the canonical name).**

   ```bash
   curl -s "https://www.thesportsdb.com/api/v1/json/123/lookupleague.php?id=4337" | jq '.leagues[0]'
   ```

3. **Pull the roster.**

   ```bash
   curl -s "https://www.thesportsdb.com/api/v1/json/123/search_all_teams.php?l=Dutch%20Eredivisie" \
     | jq '.teams[] | {idTeam,strTeam,strTeamAlternate,strTeamShort,strBadge}'
   ```

4. **Rate-limit yourself.** Key `123` is a shared free key that returns HTTP 429 under
   load. This repo's own generator waits **1800 ms between requests** with up to 8 retries
   and honors `Retry-After` — match that. Do not parallelize these calls.

5. **Cross-check the roster against a current, authoritative source** (the league's
   official clubs page or Wikipedia's current-season article) rather than trusting
   TheSportsDB's membership — it lags promotions/relegations. Record that URL in the
   league's `source` field.

6. **Download badges, slugify, commit** under the §8 layout.

7. **Update `catalog.json`**: append the league (and the sport, if new) and set
   `expectedTeamCount` / `resolvedTeamCount` / `missingTeams` honestly. Those three fields
   are how the next agent knows what is still open — an accurate `missingTeams` is worth
   more than a padded `resolvedTeamCount`.

8. **Validate before committing** (see §10).

A working reference implementation of this whole loop already exists —
`scripts/generate-sportsdb-logo-catalog.mjs`, driven by `scripts/priority-league-seeds.mjs`.
Read it before writing anything new; it handles the 429 backoff, the persistent lookup
cache, alias-based team resolution, and the exact slug/path scheme described above. The
main difference is that it writes `strBadge` as `/sportsdb-logos/...` (a leading-slash web
path for this app's `public/`), whereas **the icon library must use repo-relative paths
with no leading slash.**

> ⚠ **A script of that same name in another consumer repo is what actually produced the
> shipped `catalog.json`** — the current file is byte-identical to a copy deleted from that
> repo when this one was split out of it. Its output paths are already repo-relative with no
> leading slash.
>
> At least two copies of this generator therefore exist, and they disagree on `strBadge`
> format. Any regeneration must state which copy was used, or `catalog.json` will silently
> flip path conventions and break one consumer. Hand-edits here are also overwritten by
> whichever generator runs next — durable fixes belong in that generator's
> `priority-league-seeds.mjs`.

### What belongs in this app instead

Not every naming problem should be solved in the library. Add to `src/services/logoResolver.js`:

- `TEAM_ALIASES` / `LEAGUE_ALIASES` / `SPORT_ALIASES` — for feed-specific abbreviations
  beyond the two alias slots the catalog gives you.
- `NATIONAL_TEAM_BADGES` — national teams; the icon library has **no** national-team
  badges today, and a bare `France` would otherwise fuzzy-match a club.
- `src/data/scraped-team-logos.json` — one-off exact-name overrides.

**When changing resolver scoring or cache shape, bump `LOOKUP_CACHE_VERSION`** in
`logoResolver.js`, or returning users keep the stale results in `localStorage` (hits cache
for 7 days; misses for 30 minutes).

---

## 10. Validation checklist

Before opening a PR on the icon library:

```bash
# 1. Valid JSON
jq empty catalog.json

# 2. Every strBadge points at a file that exists, with exact case
jq -r '[.sports[] | .strSportIconGreen,
        (.leagues[] | .strBadge, (.teams[]? | .strBadge))]
       | map(select(. != null and . != "" and (startswith("http") | not)))
       | unique[]' catalog.json \
  | while read -r p; do [ -f "$p" ] || echo "MISSING: $p"; done

# 3. No leading slashes and no absolute URLs in strBadge (breaks URL joining)
jq -r '.sports[].leagues[] | .strBadge, (.teams[]? | .strBadge)' catalog.json \
  | grep -E '^(/|https?://)' && echo "BAD PATH ^^"

# 4. No team missing a name or badge (these are silently dropped by the app)
jq -r '.sports[].leagues[].teams[]? | select(.strTeam == null or .strTeam == ""
       or .strBadge == null or .strBadge == "") | .idTeam' catalog.json

# 5. No duplicate compacted team names within one league (first match wins arbitrarily)
jq -r '.sports[].leagues[] as $l | $l.teams[]? |
       "\($l.strLeague)\t\(.strTeam|ascii_downcase|gsub("[^a-z0-9]";""))"' catalog.json \
  | sort | uniq -d
```

Then verify end-to-end against this app:

```bash
# in gayfy_web
npm run lint
npm test
npm run dev     # browse an event in the league you added; the badge should render
```

In the browser console, a failed lookup logs `[logoResolver] resolution failed`. Silence
plus a fallback emoji means the entry loaded but **did not match** — nearly always a §5
alias problem or a §7 context-gate rejection.

> Note: `src/data/sportsdb-logo-catalog.json` (the bundled fallback) references
> `/sportsdb-logos/...` under `public/`, but **`public/sportsdb-logos/` does not exist in
> this repo** — those assets were never committed. So the bundled catalog currently
> contributes broken image URLs, and `ResolvedLogoImage`'s `onError` handler falls back to
> the emoji. That makes the icon library the only working badge source today, and it is
> the reason to prioritize populating it.

---

## 11. Quick reference

| Thing | Value |
|---|---|
| Catalog URL | `https://raw.githubusercontent.com/LosLonelyDevs/sportsdb-icon-library/main/catalog.json` |
| Asset base | `https://raw.githubusercontent.com/LosLonelyDevs/sportsdb-icon-library/main/` |
| Branch | `main` (hardcoded in the app) |
| Consumed kinds | `team`, `league` only — **not** `sport` |
| Match style | exact, on `compactText()`; gated on sport only (no league gate) |
| Required team fields | `strTeam`, `strBadge` |
| Required league fields | `strLeague`, `strBadge` |
| Alias fields | `strTeamAlternate`, `strTeamShort`, `strLeagueAlternate` (each a single string, **not** comma-split) |
| `strBadge` format | repo-relative, no leading slash |
| Sport vocabulary | TheSportsDB (`Soccer`, `Ice Hockey`); the app translates its own labels (`Football`, `Hockey`) via a synonym map |
| App-side integration | `src/services/logoResolver.js`, `src/components/ResolvedLogoImage.jsx` |
| Reference generator | `scripts/generate-sportsdb-logo-catalog.mjs` |
