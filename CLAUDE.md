# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

SK Chemical Compliance RiskOps Dashboard — a single-file SPA (`index.html`, ~3500 lines) for managing compliance risk assessments. Korean language UI. No build tools, no framework — vanilla HTML/CSS/JS with CDN-lazy-loaded libraries (SheetJS, PptxGenJS).

## Architecture

### Single-File Structure

Everything lives in `index.html`:
- **CSS** (lines 33–436): Design tokens in `:root`, responsive breakpoints at 768px (mobile) and 400px (small phone)
- **HTML** (lines 438–930): Login screen → App shell (sidebar + topbar + content views + mobile drawer/bottom-sheet/bottom-nav) → Excel upload modal
- **JS** (lines 932–3527): CONFIG → SP helpers → UI components → data loading → navigation → view renderers → patches

### CONFIG & Dual Mode

```javascript
const CONFIG = {
  SP_SITE_URL, LIST_RISK_MASTER, LIST_MONTHLY, LIST_EVALUATION,
  FLOW_SUBMIT_URL, FLOW_SAVE_URL, FLOW_REVISION_URL,
  OFFLINE_MODE: true,  // true = sample data, false = SharePoint live
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
```

### Navigation / Routing

SPA routing via `navigateTo(id)`. Page IDs: `dash`, `monthly`, `eval`, `teams`, `master`, `selection`, `reports`, `myteam`. Admin sees all; user sees `monthly` + `myteam`. Each page has a corresponding `render*()` function.

### Data Flow

1. `loadData(year, month)` → `_loadOfflineData()` or `_loadSPData()` → populates `RISKS`, `TREND`, `TEAMS_DATA`
2. SharePoint: `spGet/spPost/spPatch` with digest-cached auth
3. Power Automate: `callFlow(url, body)` for form submissions (F-01 submit, F-06 draft, F-07 revision)
4. localStorage: `_lsSave/_lsLoad/_lsGetAll` with key pattern `riskops_{year}_{month}_{riskId}`

### UI Component Patterns

- **Toast**: `toast(msg, type, duration)` — types: success/error/warn/info
- **Modals**: `showInputModal({...})` / `showConfirmModal({...})` — return Promises
- **Loading**: `showLoading(msg)` / `hideLoading()`
- **Charts**: `drawChart(canvasId, data, keys, colors, legend)` — manual Canvas 2D
- **Status helpers**: `stIcon(s)`, `stClass(s)`, `scClass(v)`, `scoreCol(v)`, `heatColor(v)`

### Monkey-Patch Pattern (v9.2)

Late in the file, several functions are wrapped for enhancement without modifying originals:
```javascript
const _origFn = someFunction;
someFunction = function(...) { /* added behavior */ _origFn(...); };
```
This applies to: `navigateTo`, `openSheet`/`closeSheet`, `drawChart`, `drawTeamsChart`, `renderMyTeam`, `renderEvalList`, `parseExcelFile`, `_setDirty`, `buildMonthlyFormBody`, `buildEvalBody`.

## Development

### Running Locally

Open `index.html` directly in a browser. No server required for offline mode. For SharePoint-connected mode, the page must be served from a domain allowed in SP CORS settings.

### CDN Dependencies (lazy-loaded on demand)

- **SheetJS** (xlsx 0.18.5) — Excel import/export, loaded when user clicks export or upload
- **PptxGenJS** (3.12.0) — PPT generation, loaded when user clicks PPT generate

### Key Documents

- `SETUP.md` — Step-by-step SharePoint/Power Automate connection guide
- `plan.md` — Full feature inventory, 6-phase roadmap, PM/Designer agent analysis with change history

## Conventions

- All user-facing text is in Korean
- Status values: `제출완료`, `평가완료`, `미제출`, `임시저장` (submitted/evaluated/not-submitted/draft)
- Risk scores: impact(1-5) × likelihood(1-5), high ≥15 (red), medium 8-14 (amber), low 1-7 (green)
- CSS uses `var(--accent)`, `var(--green)`, `var(--amber)`, `var(--red)` color tokens
- Modals use `role="dialog"` + `aria-label`; nav buttons get `aria-current="page"`
- Form dirty tracking: `_setDirty(true/false)` triggers beforeunload warning + visual dot indicator
