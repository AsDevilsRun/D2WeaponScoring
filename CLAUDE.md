# D2 Vault Cleaner

A Destiny 2 vault cleaner: it scores your legendary weapons and armor against your own lists (Google Sheets published as CSV) and builds a DIM search for the junk. Hosted on GitHub Pages at asdevilsrun.github.io.

## Layout

- `vault.html` is currently the whole app: HTML, CSS and JS in one file, with no build step or dependencies. That's how it grew, not a rule; splitting files or adding libraries is open for discussion.
- There is no `dev.html`. Dev features live in `vault.html` behind a flag (see Dev mode).
- The script runs top to bottom in these sections (each opens with a `/* ==== NAME ==== */` banner):
  - **SETUP:** config, `DEV`, the sheet links, `BUILD`, `state`.
  - **HELPERS:** `el()` builds DOM elements, plus `store`, the per-update caches (`loadCache`/`saveCache`), `fetchText`, `parseCSV`, and `renderStats`/`dimSearch`/`tag`/`icon`, which both pages use.
  - **BUNGIE:** `bungie()`, `itemDef`, `manifestFile`, `eachLimited`, `profileItems`, sign-in, `loadDefs`, `loadWeapons`.
  - **WEAPONS:** the list, perk columns and pools, scoring, and rendering (`render` → `weaponRow` → `perkGrid`).
  - **ARMOR:** set and plug data, fit scoring, `solveGroup`, `armorBuilds`/`keepSpares`/`knockout`, rendering (`renderArmor` → `targetBox`, `armorRow`), and exotics.
  - **FARMING:** `loadFarmList`, `farmScores`, `renderFarming` → `farmRow`.
  - **LIST EDITOR** (dev) and **DIM EXPORT INPUT** (dev).
  - **SCANNING:** `scan` (weapons, then `scanFarming`, then `scanArmor`), `readAllArmor`.
  - **PAGE:** the `wire*` functions and `init`.

## Rules for changes

- **Bump `BUILD`** (`const BUILD = "YYYY-MM-DD HH:MM"`, America/Chicago time) on every change, so the running version can be identified in the browser.
- Build DOM with `createElement` and `textContent`. Never put sheet, CSV or Bungie text into `innerHTML`.
- The comments explain the *why* in plain language, written to the owner ("your list", "your sheet"). Match that style.
- Anything the user can't rebuild (their list, their settings) is saved with `store.set(key, value, true)` and the return value is checked. Caches use plain `store.set` and may fail silently.
- Never cache a failed lookup. Add it to the session-only `failed` set instead, so the next visit retries.
- Bungie's `/Platform` calls go through `bungie()`, which handles throttling and expired sessions.

## Dev mode

- Open `vault.html?dev` to turn it on and `?dev=0` to turn it off. It's kept in `sessionStorage` (`d2v_dev`), so it survives the OAuth round trip and is limited to one tab.
- Dev-only elements carry `class="devonly"`; CSS hides them unless `body.dev` is set. In JS, check `DEV`.
- Dev adds: DIM CSV export as a data source, the perk-ranking editor (under Advanced), and "my list" (`d2v_mylist`), which replaces sheet rows when scanning.
- The editor is intentionally hidden from the public page for now. Don't expose it without being asked.
- This is hiding, not security: the repo is public.

## Bungie

- `CONFIG.apiKey` and `CONFIG.clientId` belong to a Public OAuth app. The key only works from the site's origin, so the page can't be tested from `file://` against the live API.
- The sign-in redirect URL is `vault.html`. Public clients get a 1-hour token with no refresh token.
- Game data comes from the manifest: the light item definitions, plug sets, equipable item sets and collectibles are downloaded once per session. Full item definitions are fetched one at a time and cached per manifest version.

## Data sources

The lists live in Google Sheets. `.github/workflows/sync-lists.yml` copies the published tabs into `data/` daily (09:17 UTC) and on demand (Actions → Sync lists → Run workflow), committing only when something changed.
- A tab that fails to download, or comes back as a web page instead of CSV, keeps its old file and marks the run failed.
- vault.html reads `data/*.csv` (`LISTS`, `fetchList`). If a file is missing it falls back to the sheet's published link, and a link entered in Settings overrides both.
- The tab ids (gids) live in both the workflow and `LISTS`, so keep them in step.
- GitHub turns off scheduled workflows after 60 days without repo activity; running it once by hand turns it back on.

The lists (constants near the top of the script):

- **Weapon list** `TIERED_CSV` (gid 1453679621): `Weapon, Priority 1..11`. One row per wanted roll; a weapon may have several rows (e.g. PvE and PvP).
- **Armor list** `ARMOR_CSV` (gid 1186443884): `Class, Set, Archetype, Tertiary, Tuning`, with an optional Set Bonus Y/N column. A field may list alternatives ("Melee/Grenade"); a blank field means any.
- **Exotic list** `EXOTIC_CSV` (gid 133060317): `Class, Exotic, Archetype, Tertiary`.
- **Farming list** `FARM_CSV` (gid 1626558071, the Worth Farming tab): only `Source, Item, Why` are read, to give each weapon its activity and reason. The weapon list decides what's worth farming: Worth Farming rows for anything not on it (armor sets, dropped weapons) are ignored. The tab's other columns (Have?, the unnamed TRUE/FALSE, Score) are worked out live instead.
- `parseCSV` repairs text that went through the wrong encoding when pasted into Sheets ("Ir YÃ»t" becomes "Ir Yût", "HÃ¤kke" becomes "Häkke"); see `fixText`.
- **DIM exports** (dev only): the Weapons and Armor CSVs from DIM's Settings → Spreadsheets. They rely on the columns `Perks N` (a `*` suffix marks the selected perk), `Id`, `Hash`, `Rarity`, `Crafted`, `Locked`, `Archetype`, `Tertiary Stat`, `Tuning Stat`, `Total (Base)` and `Source`.

## Scoring (intended behavior; don't "fix" it)

- **Weapons:**
  - Priority 1 is worth 25 points and Priority 2 is worth 20. The filled Priorities 3–11 score 9, 8, 7, and so on.
  - Perks are matched by name against every perk option on the copy, wherever its column. Copies with more options scoring higher is intended.
  - Each row keeps its best-scoring copy. Copies tied with identical scored perks keep only the oldest (lowest item ID); tied copies with different perks are all kept.
  - Adept, Timelost, Harrowed and Baroque copies count as the base weapon (`baseName`).
  - Enhanced perks count as the base perk (`cleanPerk`). DIM writes them as "X Enhanced"; perks really named "Enhanced Battery" or "Enhanced Heatsink" keep their names.
  - Options: weapons not on the list are junk; crafted and locked copies can be protected.
- **Columns:** Priorities 1/3/5 are trait 1, 2/4/6 are trait 2, 7/8 are the magazine, 9/10 the barrel and 11 the origin trait (`PERK_COLS`). Columns are only used for the perk display.
- **Armor:**
  - Rows with the same class and set form 5-row builds.
  - Each piece's fit is ranked: archetype (16) > tertiary isn't Health (8) > tertiary matches (4) > tuning isn't Health (2) > tuning matches (1).
  - Assignment priority per piece: filled > fit > preferred set > base stat total > oldest.
  - `solveGroup` solves all builds of a group together. Every slot is an exact assignment problem across all builds (Hungarian, `assignMax`). Layouts are searched exhaustively up to 20,000 combinations, and by repeated one-build improvement from several orders above that.
  - Spares are kept until a set is perfect.
  - Old raid armor (no archetype, raid source) can be protected.
- **Exotics:** for each exotic on the list, only the best copy is kept. Exotic class items are never judged, because their random perks make every copy different.
- **Farming page:**
  - Use A: pick an activity, see what you most need there. Use B: read down the worst scores and pick where to go.
  - Score = your best copy's points / the row's max. The max is the sum of the row's points, which is 90 when all 11 priorities are filled, matching the sheet's formula.
  - A weapon with several rolls shows the roll you're furthest from (the minimum). This differs from the sheet, which uses one particular roll.
  - Exactly the weapons on the weapon list are shown; one missing from Worth Farming goes under "Not on Worth Farming".
  - Activity buttons are A–Z, each showing your worst score there. Items are sorted worst first.
  - Clicking an item you own opens its copies on the Weapons page.
- The DIM search is always `-tag:keep (id:… OR …)`.

## Browser storage (localStorage, about 5 MB per origin)

- **Irreplaceable:** `d2v_mylist`.
- **Settings:** `d2v_page`, `d2v_tiered_url`, `d2v_armor_url`, `d2v_farm_url`, `d2v_excluded`, `d2v_useMine`, `d2v_opt*`.
- **Session:** `d2v_token`, `d2v_oauth_state`.
- **Rebuildable** (cleared first when an important save doesn't fit):
  - the per-manifest caches `d2v_wpools_<v>`, `d2v_wperk_<v>`, `d2v_plugs_<v>` and `d2v_wmeta4_<v>`, pruned by `pruneCaches` when the version changes
  - the saved DIM exports `d2v_dev_weapons` and `d2v_dev_armor`
- If you add a key, decide which group it belongs to, and update `REBUILDABLE` if it's rebuildable.

## Testing

There are no automated tests in the repo. Before handing back a change:

1. Run a syntax check: extract the `<script>` body and run `node --check` on it.
2. Test logic changes (scoring, `solveGroup`) in Node with `vm`, using stub DOM and `localStorage` globals. Append the test code to the script text so it can see the top-level `const`s, and strip the final `init();`. For the solver, compare against a brute-force assignment on small random cases.
3. Check the page in headless Chrome (Playwright) served from localhost. Mock `www.bungie.net/Platform/...`, the manifest JSON files and the `docs.google.com` CSVs with `page.route`. Check both the public view (`vault.html`) and dev mode (`vault.html?dev` with DIM CSVs loaded) for page errors.
4. For a refactor, save the old file first and run both versions on the same mock data. Use vault mode with a mocked profile, and dev mode with mock DIM exports. Compare the stats, DIM searches, table and set-box HTML (before and after option toggles), and the editor's HTML; they must match exactly.
5. The real Bungie sign-in can only be checked on the live site.

## Deploying

Commit `vault.html` to the default branch; GitHub Pages serves it, along with `data/`. The owner usually uploads files through the GitHub web UI.
