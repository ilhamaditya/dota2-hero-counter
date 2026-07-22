# Dota 2 Hero Counter Picker

A free, single-page tool that recommends Dota 2 counter picks for a selected enemy draft, with lane, item, and teamfight guidance — no backend, no database, no accounts, no AI, no external API.

Live concept: search or tap up to 5 enemy heroes → get ranked, explained counter-pick recommendations → copy a draft summary. First useful result in 3 interactions or fewer.

## Product overview

- **Problem**: during drafting, players need a fast, low-friction way to sanity-check counter picks against one or more enemy heroes.
- **Solution**: a static page with a hand-curated hero/matchup dataset and a transparent, deterministic scoring engine that runs entirely in the browser.
- **Monetization**: designed for Google AdSense, with clearly marked, layout-stable ad placeholders (no real ad code included).

## Product constraints

This project intentionally has **no**:

- Backend, database, or server-side processing
- External API calls or API keys
- AI/LLM involvement in the runtime app
- Authentication or accounts
- Paid services or recurring operational cost
- Framework dependency (no React/Vue/Angular/Next.js)
- Required build step or npm install

Everything ships as static files: `index.html`, `README.md`, `robots.txt`, `sitemap.xml`, `favicon.svg`.

## Features

- Instant, case-insensitive hero search with alias support ("potm" → Mirana, "bara" → Spirit Breaker, "am" → Anti-Mage) and match highlighting
- Tap-to-toggle hero grid, up to 5 enemies, duplicate-proof by construction
- Transparent multi-enemy scoring: coverage bonus for hitting multiple enemies and a conflict penalty for picks that are themselves countered
- Position (1–5), attribute (Strength/Agility/Intelligence/Universal), and counter-type filters
- Per-recommendation breakdown: reason, lane tip, teamfight tip, suggested items, difficulty, and a plain-language score explanation
- One-click plain-text draft summary via the Clipboard API with a manual-copy fallback
- Optional `localStorage` persistence of the last draft/filters (app works fully without it)
- Runtime data validation with console diagnostics; corrupted or invalid entries are dropped instead of crashing the UI
- Semantic HTML, JSON-LD (`WebApplication` + `FAQPage`), and on-page SEO content sections

## Technology stack

- HTML5 (semantic markup, JSON-LD)
- CSS3 (custom properties, CSS Grid/Flexbox, `prefers-reduced-motion`, mobile-first)
- Vanilla JavaScript (ES5-leaning syntax, no transpilation needed, no dependencies)
- System font stack (no remote font loading)

## File structure

```
dota2-hero-counter/
├── index.html      # entire app: markup, <style>, <script>, hero + matchup data
├── README.md
├── robots.txt
├── sitemap.xml
└── favicon.svg
```

## Local usage

No build step. Just open the file:

```bash
open index.html          # macOS
# or serve it locally to avoid any local-file quirks:
python3 -m http.server 8080
```

Then visit `http://localhost:8080`.

## Free deployment

### Vercel
1. Push this repo to GitHub/GitLab/Bitbucket.
2. Import the repo in the Vercel dashboard, framework preset "Other".
3. No build command, no output directory override needed — deploy as-is.

### Netlify
1. Drag-and-drop the project folder into Netlify's "Deploys" page, or connect the git repo.
2. Build command: (leave empty). Publish directory: `/` (repo root).

### GitHub Pages
1. Push to a GitHub repo named `dota2-hero-counter` (or any name).
2. Repo Settings → Pages → Deploy from branch → select `main` and `/ (root)`.
3. Update `sitemap.xml`, `robots.txt`, and the `<link rel="canonical">`/OG tags in `index.html` with your real `https://<user>.github.io/<repo>/` URL.

### Cloudflare Pages
1. Connect the git repo in the Cloudflare Pages dashboard.
2. Build command: (leave empty). Build output directory: `/`.

In all four cases, no environment variables, serverless functions, or database provisioning are required.

## Hero data model

Defined in `index.html` inside the `HEROES` array:

```js
{
  id: "anti-mage",          // stable, kebab-case, never reused/changed once matchups reference it
  name: "Anti-Mage",
  aliases: ["am"],           // lowercase search shortcuts
  attribute: "Agility",      // "Strength" | "Agility" | "Intelligence" | "Universal"
  positions: [1],            // integers 1-5
  roles: ["Carry", "Escape"],// free-text display tags
  difficulty: "Medium",      // "Easy" | "Medium" | "Hard"
  icon: "antimage"           // optional: Valve's internal hero codename, used to hotlink
                              // that hero's official art from Dota 2's own CDN. Omit if
                              // unknown/unverified — the initials badge is the permanent
                              // fallback and is what renders if the image 404s.
}
```

Hero art is hotlinked directly from Valve's official CDN (the same source Dotabuff/OpenDota use) — never copied or rehosted in this repo. Two sizes are used from the same `<icon>` codename: a small square icon (`heroes/icons/<icon>.png`, 32×32) on the compact enemy-tray chips, and a larger "hero select screen" portrait (`heroes/<icon>.png`, 256×144) on the browsable hero grid and recommendation cards, both under `cdn.cloudflare.steamstatic.com/apps/dota2/images/dota_react/`. This is the one deliberate exception to the "no unlicensed hero artwork" rule: the images are served by Valve's own servers, not stored or distributed by this project. If a CDN path ever changes or 404s for a hero, the `<img>`'s `onerror` handler removes it and the colored-initials layer underneath (always rendered, never removed) is what the user sees — there is no broken-image state.

## Matchup data model

Defined in the `MATCHUPS` object, keyed by hero id, each value an array of counter entries:

```js
MATCHUPS["phantom-assassin"] = [
  {
    counterHeroId: "axe",         // must exist in HEROES
    score: 4,                      // see scoring rubric below
    categories: ["Disable", "Teamfight"], // from the fixed CATEGORY_VOCAB list
    reason: "Counter Helix isn't classified as an attack, so it ignores her Evasion...",
    laneTip: "Stand on creep aggro and Call her whenever she goes for last hits...",
    teamfightTip: "Call her before she can Blink Strike a squishy target...",
    suggestedItems: ["Blade Mail", "Heaven's Halberd", "Silver Edge"]
  }
]
```

The data is normalized: general hero facts live only in `HEROES`; matchup-specific reasoning lives only in `MATCHUPS`, referenced by id.

## Scoring methodology

**Matchup data source:** which heroes counter which, and how strongly, is derived from real aggregate match win-rate data pulled from OpenDota's public statistics API (patch 7.41d snapshot), not purely theorycrafted ability interactions. For each of the 31 heroes in this app, its listed counters are the opponents (restricted to this app's own 31-hero roster) it has historically had the worst win rate against, filtered to matchups with at least 15 recorded games in the sampled data. Each matchup's 0-5 `score` is derived directly from that win-rate gap:

| Countered hero's win rate vs. this counter | Score |
|---|---|
| under 30% | 5 (excellent) |
| 30–35% | 4 (strong) |
| 35–40% | 3 (useful) |
| 40–45% | 2 (minor) |
| 45% and up | 1 (slight) |

The `reason`, `laneTip`, `teamfightTip`, and `suggestedItems` text on each entry is then written to explain the mechanical, ability-level basis for that statistically-supported pairing — the pairing and score are data-derived; the explanation is authored.

**Per-draft scoring:** for the currently selected enemy draft, each candidate counter hero gets:

```
finalScore =
    sum of its matchup scores against every selected enemy it counters
  - sum of matchup scores of any selected enemy that counters it back (conflict penalty)
  + 1 point per additional enemy countered beyond the first (coverage bonus)
```

Position, attribute, and counter-type filters only narrow the candidates shown; they do not modify scores.

Recommendations are sorted by `finalScore`, then by number of enemies countered, then alphabetically — always deterministic, no randomness. This is a snapshot of aggregate statistics from one point in time, restricted to a 31-hero sample — **not** a live win-rate feed. The underlying numbers shift with every patch and as the meta evolves, so treat it as informed guidance to verify against current patch notes, not a permanent fact. See the in-page "Methodology" section for the same explanation aimed at end users.

## How to add a hero

Append an object to `HEROES` with a new, permanent `id`. Never reuse or change an existing id once matchups reference it.

## How to add an alias

Push a lowercase string into that hero's `aliases` array.

## How to add a matchup

Push an entry into `MATCHUPS[heroId]` following the shape above. Only reference `counterHeroId` values that exist in `HEROES`, and only use categories from `CATEGORY_VOCAB` (defined near the top of the `<script>`).

## How to update scores / items

Edit the `score` or `suggestedItems` field on the relevant matchup entry directly.

## How to disable an outdated recommendation

Add `disabled: true` to that matchup entry. It is skipped everywhere in the app (search, scoring, sharing) without deleting its history — useful right after a balance patch while you re-verify it.

## How to update patch metadata

Edit the three fields in `APP_META` near the top of the `<script>` block:

```js
var APP_META = {
  patch: "7.41d",
  lastReviewed: "2026-07-20",
  dataVersion: "1.0.0"
};
```

This single edit updates the header badge, footer text, and stays consistent everywhere it's displayed.

## Data validation

`validateData()` runs on every page load and checks: hero id uniqueness, required fields, valid attribute/position values, matchup references to real heroes, no self-counters, no duplicate counters per hero, valid numeric scores, and recognized categories. Issues are logged to the browser console (`[Dota 2 Hero Counter Picker] Data validation found N issue(s)`); `sanitizeMatchups()` then silently drops anything invalid so the UI never crashes on bad data.

## AdSense integration notes

Five placeholder slots are marked with HTML comments (`<!-- AdSense placement: ... -->`) and a `.ad-slot` div with a reserved `min-height` to avoid layout shift:

1. Below the introduction
2. Below the selected enemy draft
3. Between filters and recommendations
4. After the top recommendations
5. Before the FAQ/footer boundary

To go live, replace each `.ad-slot` div's contents with your AdSense `<ins class="adsbygoogle">` snippet and add the AdSense loader script — no other markup changes are needed. No real publisher ID or ad code is included in this repo.

## SEO checklist

- [x] Descriptive `<title>` and meta description (primary keyword: "Dota 2 hero counter")
- [x] Canonical URL, Open Graph, and Twitter Card tags (placeholder domain — update before launch)
- [x] Mobile viewport + `theme-color`
- [x] Logical heading hierarchy (single `h1`, `h2` per section, `h3` per recommendation card)
- [x] JSON-LD `WebApplication` + `FAQPage` structured data
- [x] Indexable static content: How it works, Drafting principles, Laning vs. teamfight counters, Patch impact, Methodology, FAQ
- [x] `robots.txt` and `sitemap.xml`
- [ ] Replace all `https://dota2-hero-counter.example.com/` placeholders with your real domain before launch
- [ ] Add a real 1200×630 social share image (currently falls back to the favicon)
- [ ] Replace the placeholder `hello@dota2-hero-counter.example.com` Contact address in `index.html` with a real, monitored inbox before launch

## Accessibility checklist

- [x] Skip-to-content link, visible focus states (`:focus-visible`)
- [x] Labeled form controls (`<label for>`, `<fieldset>/<legend>` for the hero grid)
- [x] `aria-live` regions for the enemy tray, recommendations, and share status
- [x] `aria-pressed` on hero toggle buttons; counter level always shown as text, never color-only
- [x] `<output>` for dynamic status text instead of ARIA-role-only markup
- [x] `prefers-reduced-motion` respected (disables smooth scroll/transitions)
- [x] Keyboard-operable: search, hero grid, filters, share button, `<details>` disclosures
- [x] Sufficient color contrast on the dark palette

## Trademark disclaimer

This is an independent, fan-made tool. It is **not affiliated with, endorsed by, or sponsored by Valve Corporation**. Dota and Dota 2 are trademarks or registered trademarks of Valve Corporation. Matchups and recommendations may change after gameplay or balance patches; treat them as general guidance, not guaranteed outcomes, and verify against current patch notes and your own match context.

## Maintenance guide

- After each balance patch: re-pull matchup win rates for the 31-hero roster from `https://api.opendota.com/api/heroes/<opendota_id>/matchups` (hero id mapping via `https://api.opendota.com/api/heroes`), re-derive scores per the table in "Scoring methodology" above, review/update `reason`/`laneTip`/`teamfightTip`/`suggestedItems`, or `disabled: true` stale entries, then bump `APP_META`.
- OpenDota is queried only as a one-time authoring step when refreshing the dataset — the shipped app itself makes no external calls at runtime.
- Keep the dataset curated rather than exhaustive — this release covers 31 commonly picked heroes (all 4 attributes, all 5 positions), not the full roster. Coverage is explicitly labeled as a sample in-page and here.
- Run the app in a browser and check the console after any data edit — `validateData()` will flag broken references immediately.
- No dependency updates are ever required since there are no dependencies.
