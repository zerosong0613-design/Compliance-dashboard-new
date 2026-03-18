# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

SK Chemical Compliance RiskOps Dashboard — a single-file SPA (`index.html`, ~4500 lines) for managing compliance risk assessments. Korean language UI. No build tools, no framework — vanilla HTML/CSS/JS with CDN-lazy-loaded libraries (SheetJS, PptxGenJS). Current version: v10.1.

## Architecture

### Single-File Structure

Everything lives in `index.html`:
- **CSS** (lines 33–584): Design tokens in `:root`, responsive breakpoints at 768px (mobile) and 400px (small phone). Includes archive, phase-hero, step-bar, law-alert, copilot-agent styles.
- **HTML** (lines 588–1190): Login screen (step indicator) → App shell (sidebar + topbar + content views incl. archive + mobile drawer/bottom-sheet/bottom-nav) → Excel upload modal
- **JS** (lines 1191–4507): CONFIG → sample data → SP helpers → toast/modal/loading → data loading → login/nav → helpers → dashboard (cycle card + phase hero) → archive → monthly input → eval → teams chart → master (edit/add modals) → selection → reports (PPT preview + generation) → Excel import/export → AI (Claude API) → monkey-patches (v9.2/v9.3/v10.1)

### CONFIG & Dual Mode

```javascript
const CONFIG = {
  SP_SITE_URL, LIST_RISK_MASTER, LIST_MONTHLY, LIST_EVALUATION,
  FLOW_SUBMIT_URL, FLOW_SAVE_URL, FLOW_REVISION_URL,
  FLOW_NUDGE_URL,      // F-04 독촉 메일 HTTP 트리거 URL
  COPILOT_AGENT_URL,   // Copilot Studio agent embed URL (archive page)
  OFFLINE_MODE: true,   // true = sample data, false = SharePoint live
};
CONFIG.ANTHROPIC_API_KEY = '';  // optional Claude AI summaries
```

When `OFFLINE_MODE: true`, the app uses `SAMPLE_RISKS` / `SAMPLE_TREND` arrays with localStorage persistence. When `false`, it calls SharePoint REST API directly from the browser (no backend).

### Key State Variables

```
RISKS[]       — master risk list (in-memory, reloaded per year/month)
TEAMS_DATA[]  — computed team averages
TREND[]       — 6-month trend data
appRole       — 'admin' | 'user'
appTeam       — selected team name
curPage       — current route id
curYear/curMonth — active period
_hasDirtyForm — unsaved form changes flag
_archiveFiles[] — archive file list (in-memory)
_archiveYearFilter / _lawCategoryFilter — archive page filters
```

### Navigation / Routing

SPA routing via `navigateTo(id)`. Page IDs: `dash`, `monthly`, `eval`, `teams`, `master`, `selection`, `reports`, `myteam`, `archive`. Admin sees all; user sees `monthly` + `myteam` + `archive`. Each page has a corresponding `render*()`/`init*()` function.

### Data Flow

1. `loadData(year, month)` → `_loadOfflineData()` or `_loadSPData()` → populates `RISKS`, `TREND`, `TEAMS_DATA`
2. SharePoint: `spGet/spPost/spPatch` with digest-cached auth
3. Power Automate: `callFlow(url, body)` for form submissions (F-01 submit, F-06 draft, F-07 revision)
4. localStorage: `_lsSave/_lsLoad/_lsGetAll` with key pattern `riskops_{year}_{month}_{riskId}`

### UI Component Patterns

- **Toast**: `toast(msg, type, duration)` — types: success/error/warn/info
- **Modals**: `showInputModal({...})` / `showConfirmModal({...})` — return Promises
- **Loading**: `showLoading(msg)` / `hideLoading()`
- **Charts**: `drawChart(canvasId, data, keys, colors, legend)` — manual Canvas 2D, Y-axis dynamic range, year boundary lines
- **Status helpers**: `stIcon(s)`, `stClass(s)`, `scClass(v)`, `scoreCol(v)`, `heatColor(v)`
- **XSS defense**: `_esc(s)` — escapes user input before innerHTML insertion

### Monkey-Patch Pattern (v9.2/v9.3/v10.1)

Late in the file (lines ~4036–4507), several functions are wrapped for enhancement without modifying originals:
```javascript
const _origFn = someFunction;
someFunction = function(...) { /* added behavior */ _origFn(...); };
```
This applies to: `navigateTo`, `openSheet`/`closeSheet`, `drawChart`, `drawTeamsChart`, `renderMyTeam`, `renderEvalList`, `parseExcelFile`, `_setDirty`, `buildMonthlyFormBody`, `buildEvalBody`, `confirmExcelImport`, `_loadOfflineData`, `renderReports`, `submitMonthly`, `saveDraft`, `showConfirmModal`, `requestRevision`.

v10.1 additions: Escape key global modal closer, Bottom Sheet dirty-form guard, requestRevision offline state persistence.

### Key Pages & Functions

| Page | ID | Init/Render | Key features |
|------|----|-------------|-------------|
| Dashboard | `dash` | `initDash()`, `renderDash()` | KPI cards, action banner, cycle card (5-phase hero), achievement chart, heatmap, missing alerts |
| Monthly Input | `monthly` | `renderMonthly()` | Risk list + form panel, chip selector, file attach, submit/draft/revision |
| Evaluation | `eval` | `renderEvalList()` | Score slider, quick-score buttons, eval comment (required <70%) |
| Teams | `teams` | `renderTeams()` | Team stat cards, trend line chart (Canvas), team filter tabs |
| Master | `master` | `applyMaster()`, `renderMaster()` | Table, search, filter, Excel upload/download, add/edit risk modals |
| Selection | `selection` | `renderSelection()` | Sorted by risk score, top-N auto-select, checkbox toggle |
| Reports | `reports` | `renderReports()` | Monthly/annual tab, PPT preview (live HTML), PPT generation (PptxGenJS) |
| My Team | `myteam` | `renderMyTeam()` | KPI 3-card, trend chart, risk status list |
| Archive | `archive` | `initArchive()` | Year-filtered files, law insight alerts, Copilot agent iframe |

## Development

### Running Locally

Open `index.html` directly in a browser. No server required for offline mode. For SharePoint-connected mode, the page must be served from a domain allowed in SP CORS settings.

### CDN Dependencies (lazy-loaded on demand)

- **SheetJS** (xlsx 0.18.5) — Excel import/export, loaded when user clicks export or upload
- **PptxGenJS** (3.12.0) — PPT generation, loaded when user clicks PPT generate

### Key Documents

- `SETUP.md` — Step-by-step SharePoint/Power Automate connection guide (F-01/F-06/F-07 setup, CONFIG reference, FAQ)
- `plan.md` — Full feature inventory (Part 1), 6-phase roadmap (Part 2), file structure (Part 3), priority matrix (Part 4), PM/Designer agent analysis & change history (Part 5)

## Conventions

- All user-facing text is in Korean
- Status values: `제출완료`, `평가완료`, `미제출`, `임시저장` (submitted/evaluated/not-submitted/draft)
- Risk scores: impact(1-5) × likelihood(1-5), high ≥15 (red), medium 8-14 (amber), low 1-7 (green) — `scClass()`, `scColor()`, `scoreStyle()`, `heatColor()` all unified on these thresholds (v10.1)
- CSS uses `var(--accent)`, `var(--green)`, `var(--amber)`, `var(--red)` color tokens
- Modals use `role="dialog"` + `aria-label`; nav buttons get `aria-current="page"`
- Form dirty tracking: `_setDirty(true/false)` triggers beforeunload warning (monthly/eval/master) + visual dot indicator
- Modal UX: Escape key closes topmost modal (6 types + Excel + Sheet + Drawer); Bottom Sheet overlay checks dirty form before closing
- Color: `--muted: #475569` (WCAG AA 7.08:1 on white, 5.1:1 on --bg)
- Sample data: SAMPLE_RISKS (8 risks, 5 teams) / SAMPLE_TREND (6 months) with localStorage overlay
- localStorage persistence: submit/eval/edit/add all save via `_lsSave()`; new risks flagged with `_isNew` for reload recovery
- Archive: ARCHIVE_SAMPLE_FILES (5 files) / LAW_ALERTS_SAMPLE (5 law insights) — in-memory only until SP connected
- Excel import: merges with existing RISKS (deduplicates by title+team), does not replace

