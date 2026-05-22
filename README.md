# CleanScore — App Status
*Last updated: 2026-05-22 (session 50 — Know Your Inspector section hidden) by Claude Code*

## Live App
- URL: https://pinellasiceco.github.io/cleanscore
- Deployed via: `pages.yml` — triggers on every push to main
- Data source: Supabase (violations, stats, partners JSON) built by `Pinellasiceco/export_cleanscore.py` at CI time
- Build repo: `pinellasiceco/Pinellasiceco` — CleanScore data is exported from there and pushed to Supabase

## What It Does
Consumer-facing restaurant health inspection report page. Business owners search by name or license number to see their CleanScore (0–100), violation breakdown, inspection history chart, action banners, county context, and intelligence summary. Also supports shareable inspector-ready verification reports (Stripe-gated).

## Data Pipeline (from Pinellasiceco repo)
- `scrape_dbpr.py` — scrapes DBPR inspection detail pages for violation narratives; output: `full_inspection_narratives.json`
- `export_cleanscore.py` — reads DBPR bulk CSV + narratives cache; builds violations export, stats export, partners export, inspector export; pushes to Supabase via REST API
- `build_violations_list.py` — generates `data/pinellas_all_violations_to_scrape.csv` (Pinellas businesses with violations in last 180 days)
- Supabase tables: `pic_violations` (per-business violation data), `pic_stats` (county-wide stats), `pic_partners` (partner referral data)
- Supabase Storage: `cleanscore/cleanscore_violations.json`, `cleanscore/cleanscore_stats.json`, `cleanscore/cleanscore_partners.json`

## What's Working ✅

### Core Report
- Business search by name, license number, or address (`?q=`, `?id=`, `?license=`)
- CleanScore (0–100) with hero display, color-coded bar, grade label (GOOD / WARNING / POOR / CRITICAL)
- Violation cards: each violation with severity badge, observation text, action guidance
- Severity legend (High Priority / Intermediate / Basic)
- Inspection disposition banner (Warning Issued / Emergency Order / Admin Complaint)
- What Happens Next timeline (callback inspection, enforcement escalation)

### Intelligence Section
- County context: how this business compares to Pinellas average
- Per-violation-type intelligence: what inspectors commonly find with this violation code
- Repeat-offender detection: badges for chronic/repeat ice machine violations

### Inspection History Chart
- Bar chart per inspection: red (V22/ice), amber (other violations), green (clean)
- Trend label: first-time / repeat / chronic ice history
- Timeline shows up to 10 most recent inspections

### Action Banners
- Emergency closure banner (red) when disposition is Emergency Order
- Callback warning banner (amber) when inspection is Callback
- Score cap: Emergency closure caps score at 20; multiple high-priority violations cap at 40

### Partner Referrals
- Relevant partner types surface based on violation codes found
- Hood cleaning, pest control, refrigeration, beverage equipment, HVAC

### Shareable Reports
- Stripe-gated "Inspector Report" — generates shareable verification link
- `?verify=<token>` URL renders verification view for inspector

## What's Hidden / Not Working ⚠️

### Know Your Inspector (hidden as of s50)
- Section removed from `renderReport()` and `fetchInspectors()` call removed from `init()`
- **Root cause**: Inspector names are not available from any DBPR public source
  - Bulk CSV (82 columns): no inspector name or ID field
  - `inspectionDetail.asp` (static + JS-rendered): no labeled inspector field; "Inspector" only appears inside violation observation text
  - `LicenseDetail.asp`: returns "request cannot be processed" error
  - `wl11.asp` search: returns empty search form only
  - `inspectionSearch.asp`: 404
  - Playwright Phase 1 + Phase 2 diagnostic: zero AJAX calls on any page, no inspector name in rendered DOM anywhere
- `renderInspectorSection()` function and `fetchInspectors()` kept as dead code — re-enable if a data source surfaces

## What's Missing 🔲
- **Know Your Inspector**: permanently hidden until DBPR or another source exposes inspector names publicly
- **Mobile-optimized share flow**: current share button is functional but not styled for mobile-first UX
- **Business claim flow**: no mechanism for owners to claim/verify their business listing

## Recent Changes
- **2026-05-22 (s50)**: Know Your Inspector section hidden — `renderInspectorSection()` call removed from `renderReport()`, `fetchInspectors()` call removed from `init()`. All DBPR public sources confirmed to not expose inspector names.
- **2026-05-21 (s47)**: Inspector Intelligence feature added — `renderInspectorSection()`, `fetchInspectors()`, `INSPECTORS_URL` wired up. (Subsequently hidden in s50.)
- **2026-05-21 (s47)**: Score cap fixes — emergency closure caps score at 20; action banners for admin complaint and warning disposition.
- **2026-05-21 (s48)**: Inspection history chart fixed — `_safe_col()` prevents KeyError on short DBPR files; `had_v22=True` + `num_total=0` bumped to 1.
- **2026-05-21 (s47)**: Stats fixes — `build_stats_export()` now uses all Pinellas records (was filtering to 2,661 of 9,400); repeat rate denominator fixed.

## Key Files
| File | Purpose |
|------|---------|
| `index.html` | Entire CleanScore app — single-file SPA |
| `pages.yml` | GitHub Pages deploy — triggers on push to main |
| `README.md` | This file |

## Data URLs (Supabase Storage — public read)
| File | Content |
|------|---------|
| `cleanscore/cleanscore_violations.json` | Per-business violation data array |
| `cleanscore/cleanscore_stats.json` | County-wide stats (ice citation rates, repeat rates, etc.) |
| `cleanscore/cleanscore_partners.json` | Partner referral businesses by type |
| `cleanscore/cleanscore_inspectors.json` | Inspector analytics (empty / not populated — section hidden) |
