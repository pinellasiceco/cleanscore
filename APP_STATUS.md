# CleanScore — App Status
*Last updated: 2026-05-21 (session 47 — Inspector Intelligence) by Claude Code*

## Live App
- URL: https://pinellasiceco.github.io/cleanscore
- Repo: `pinellasiceco/cleanscore`
- Deployment: `pages.yml` triggers on every push to main → GitHub Pages

## Data Sources
All data served from Supabase Storage (public bucket `cleanscore`):
| File | Updated by | Contents |
|------|-----------|---------|
| `cleanscore_violations.json` | `export_cleanscore.py` in Pinellasiceco CI | Per-business violation records with history |
| `cleanscore_partners.json` | `export_cleanscore.py` in Pinellasiceco CI | Channel partner records |
| `cleanscore_stats.json` | `export_cleanscore.py` in Pinellasiceco CI | County-wide analytics |
| `cleanscore_inspectors.json` | `export_cleanscore.py` in Pinellasiceco CI | Per-inspector stats |

Data pipeline: `build.py` → `export_cleanscore.py` → Supabase Storage → `index.html` fetches on load

## What's Working ✅

### Score & Report
- **CleanScore (0–100)** — `calcScoreWithActions()`: starts at 100, deducts for violations by severity; disposition caps: complaint ≤40, warning ≤70, closure ≤20
- **Score hero** — color-coded (green/amber/red) with label (GOOD / WATCH / CONCERN / URGENT)
- **Inspection header** — date, disposition badge, days-since label
- **Inspection timeline** — full visit history from DBPR data, date-ascending

### Violation Cards
- Expandable cards per violation with severity badge (HIGH / MAJOR / BASIC)
- Repeat violation badge — detects `repeat===true/string/1` or `category==='ice_machine' && cit_ice_count≥2`
- `repeatBadge` detail panel shown in card body for repeat violations
- `categorize_violation()` — ICE_MACHINE_NEGATIVES guard, short-text guard (<15 chars → 'other')
- Severity legend rendered above violation cards

### Action Banners
- **Admin Complaint** (dark red, priority 0) — fires when disposition contains 'complaint'
- **Callback window** (amber, priority 1) — fires 10–28 days after warning disposition
- **Inspection window** (blue, priority 2) — fires >121 days since inspection when `cit_ice_count>0` and no active warning/complaint
- Banners sorted by priority, only highest shown

### Data Intelligence Section
- Business type risk profile (violation rate, avg violations, top category)
- Repeat violation risk by category
- Predictive inspection timing (121-day county median, overdue/due/normal/recent buckets)
- Cross-violation correlation (co-occurrence rates)
- Neighborhood benchmarking (city avg score comparison)

### Know Your Inspector Section *(data populates after daily scraper runs)*
- Inspector name pulled from `business.inspector_name` in violations export
- Stats from `cleanscore_inspectors.json`: inspection count, avg violations/visit, ice machine citation rate
- Falls back to name-only display if inspector not in analytics list
- Section hidden entirely when no inspector name on record

### Stats (County Context)
- Violation rate: ~7.3% (uses `ice_confirmed` field over comprehensive Pinellas filter)
- Median inspection interval: 121 days (matches build.py model)
- Repeat probability: 46.8% (calculated from `cit_ice_count`)

### Paywall / Subscription
- New inspection + free tier → paywall
- Paid tier: score + full violation detail visible
- Stripe return handler: polls Supabase `cs_businesses` for tier upgrade
- `checkSubscriptionValid()` via `cs_businesses` table

### Search
- By business name (substring) or license number
- Auto-navigates to report if single match
- Shows top 10 results with city + last inspection date

### AI SOP Generation
- "Remedy Guide" per violation — calls Supabase Edge Function `cs-generate-sop`
- Expandable/collapsible per card
- Retry button on failure

### Partner Box
- Shown on violation cards for relevant ice machine partners
- Verified badge, tap-to-call phone link

### Share Button
- Native share sheet (Web Share API) or clipboard fallback

## Architecture

### URL constants
```javascript
var VIOLATIONS_URL = '...supabase.co/storage/v1/object/public/cleanscore/cleanscore_violations.json';
var PARTNERS_URL   = '...supabase.co/storage/v1/object/public/cleanscore/cleanscore_partners.json';
var STATS_URL      = '...supabase.co/storage/v1/object/public/cleanscore/cleanscore_stats.json';
var INSPECTORS_URL = '...supabase.co/storage/v1/object/public/cleanscore/cleanscore_inspectors.json';
```

### Global state
```javascript
var _violations=[], _viols=[], _partners=[], _actions=[], _stats=null, _inspectors=null;
var currentBusiness=null;
```

### init() flow
1. Handle Stripe return (if `?payment=success`)
2. Fetch violations + partners in parallel; `fetchStats()` fire-and-forget; `fetchInspectors()` awaited after
3. Resolve `?id=` / `?license=` / `?q=` params
4. Load actions + business record in parallel
5. Check subscription; render paywall or full report

### renderReport() section order
1. Business name / address card
2. Score hero
3. Share button
4. Action banners
5. Inspection header
6. Inspection timeline
7. Violation cards (with severity legend)
8. County context stats
9. Data Intelligence section
10. Know Your Inspector section
11. What Happens Next section

## What's Broken / Watch List ⚠️
- **Inspector section empty on launch** — `cleanscore_inspectors.json` will have 0 entries until the scraper re-runs and captures inspector names from DBPR pages under the new code. Section is hidden when `business.inspector_name` is absent, so no visible gap.
- **iOS PWA rules apply**: all buttons in `innerHTML`-injected content must use inline `ontouchend` + `onclick`; no `addEventListener` on injected elements

## Recent Changes
- **2026-05-21 (s47 — Inspector Intelligence):**
  - Added `INSPECTORS_URL` constant and `_inspectors` global state
  - Added `fetchInspectors()` async loader (graceful null on failure)
  - Added `renderInspectorSection()` — shows inspector name, inspection count, avg violations/visit, ice machine citation rate using `intel-card` styling
  - `renderReport()` now calls `renderInspectorSection(business, _inspectors)` after Data Intelligence block
  - `init()` awaits `fetchInspectors()` after main data fetch

- **2026-05-21 (s47 — Consolidated fixes):**
  - `renderActionBanners()` fully rewritten: Admin Complaint dark-red banner (priority 0), callback 10–28 day window, inspection window guarded by `cit_ice_count>0`
  - `calcScoreWithActions()` — disposition score caps (complaint ≤40, warning ≤70, closure ≤20)
  - Repeat violation detection — handles `true/string/1` plus ice_machine+cit_ice_count≥2 fallback; `repeatBadge` detail panel in card body
  - `categorize_violation()` — `ICE_MACHINE_NEGATIVES` guard, short-text guard

## Key Files
| File | Purpose |
|------|---------|
| `index.html` | Entire CleanScore app — single-file SPA |
| `.github/workflows/pages.yml` | Deploys to GitHub Pages on every push to main |
| `APP_STATUS.md` | This file |

## Dependency on Pinellasiceco Repo
CleanScore has no build step of its own — all data is generated by `export_cleanscore.py` in the Pinellasiceco repo and uploaded to Supabase Storage. To refresh data: trigger **Rebuild Prospect Tool** in Pinellasiceco → Actions. The `cleanscore/` directory in the Pinellasiceco repo is a separate git clone of this repo used during development.
