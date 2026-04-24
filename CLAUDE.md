# Wispr – Vendor Operations Dashboard

## What this is
A fully static GitHub Pages dashboard for GustoCare ops teams to monitor Wispr voice dictation adoption across vendor agents (Teleperformance, TaskUs). No backend. All data lives in `data/dashboard.json`, committed to this repo via the GitHub Contents API.

**Live URL:** https://roubachehab.github.io/vendor-wispr/  
**Repo:** `roubachehab/vendor-wispr`  
**Data file:** `data/dashboard.json`

---

## Architecture
- Single `index.html` — all HTML, CSS, and JS in one file
- `data/dashboard.json` — agent usage data, roster assignments, all dates
- GitHub Pages serves the static site; data updates via GitHub Contents API PUT
- No build step, no framework, no backend

### Data flow
1. Ops manager uploads Wispr CSV export + Vendor Roster CSV via the Update Data drawer
2. Dashboard merges new data with existing history in-browser
3. On publish, commits `data/dashboard.json` to GitHub via PAT stored in localStorage
4. GitHub Pages serves the updated data (~60 seconds to go live)

---

## Data Sources

### Wispr Export (CSV)
- Column format: `Name (email@gustocare.com), YYYY-MM-DD, YYYY-MM-DD, ...`
- Row values: daily word counts (0 or absent = no usage that day)
- ~581 @gustocare.com agents, date columns spanning full history

### Vendor Roster (CSV)
- Columns: `Name, Email, Team`
- Team values: `Teleperformance` or `TaskUs` (case-insensitive match)
- Used to assign agents to tabs: `tp`, `tu`, or `un` (unassigned)

---

## Vendor Tabs & Brand Colors
| Tab | Label | Color |
|---|---|---|
| `all` | All Agents | `#0A8080` (Gusto teal) |
| `tp` | Teleperformance | `#EF523C` |
| `tu` | TaskUs | `#2BABAD` |
| `un` | Unassigned | `#BABABC` |

---

## Usage Column (Agent Breakdown Table)
Based on **avg words/day over all days in the period** (total ÷ period days), not just active days. This reflects real adoption rather than peak capability.

| Level | Threshold | What it means |
|---|---|---|
| **High** | ≥ 500 avg words/day | Wispr is part of daily workflow |
| **Medium** | 150–499 | Regular use, not yet habitual |
| **Low** | 1–149 | Sporadic use |
| **None** | 0 | No usage in period |

Thresholds calibrated from a 2-month export (596 agents, Apr–May 2026). **Review next quarter** as adoption grows.

---

## Date Helpers — Important
All date parsing uses `localDate(s)` which parses `YYYY-MM-DD` as **local midnight** (not UTC). This prevents the off-by-one timezone bug where weeks started on Tuesday instead of Monday.

```js
function localDate(s) {
  const [y, mo, d] = s.split('-').map(Number);
  return new Date(y, mo-1, d);
}
```

Weeks start on **Monday**. `getMondayOf(d)` uses `day === 0 ? 6 : day - 1` offset.

---

## Filter System
- **Daily / Weekly / Monthly buttons** — chart grouping only; do NOT change the date range
- **Quick filter pills** (This Month / Last 30 Days / This Week / Last Week) — set the date range, anchored to the latest date in the dataset (not real today)
- **Manual date picker** — overrides quick filters; clears pill highlight
- Default on load: **This Month**

---

## GitHub Publish (PAT)
- Token stored in `localStorage` key `gh_token`
- Token scope required: `repo` only
- Recommend: Fine-grained PAT, 1-year expiry, rotate annually
- Token is never committed to the repo
- Anyone with a valid PAT can publish; each person stores their own token in their browser

---

## Key Functions
| Function | What it does |
|---|---|
| `applyData(data)` | Loads JSON into STATE, sets default This Month range, renders everything |
| `setQuickFilter(preset, btn)` | Sets date range from a preset, updates pill highlight |
| `setPeriod(p)` | Changes chart grouping only — never changes date range |
| `buildRows(agents, dates)` | Computes all table metrics per agent for the current period |
| `getUsageLevel(totalWords, numDays)` | Returns High/Medium/Low/None based on all-days avg |
| `publishData()` | Merges pending files, commits to GitHub via Contents API |
| `localDate(s)` | Parses YYYY-MM-DD as local midnight — never use `new Date(str)` directly |

---

## Modifying Thresholds
To update Usage level thresholds, change `getUsageLevel()` in `index.html`:

```js
function getUsageLevel(totalWords, numDays) {
  if (numDays === 0) return 'None';
  const avg = totalWords / numDays;
  if (avg >= 500) return 'High';    // ← adjust here
  if (avg >= 150) return 'Medium';  // ← adjust here
  if (avg >= 1)   return 'Low';
  return 'None';
}
```

Also update `USAGE_TOOLTIPS` object to keep hover text in sync.

---

## What NOT to do
- Don't use `new Date('YYYY-MM-DD')` directly — use `localDate(s)` to avoid UTC shift
- Don't change `setPeriod()` to also update the date range — it's grouping only by design
- Don't store the PAT in the repo or in any committed file
