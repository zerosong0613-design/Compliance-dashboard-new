# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

SK Chemical Compliance RiskOps Dashboard — a single-file SPA (`index.html`, ~8,400 lines) for managing compliance risk assessments across a 7-phase annual cycle. Korean language UI. No build tools, no framework — vanilla HTML/CSS/JS with CDN-lazy-loaded libraries (SheetJS, PptxGenJS). Current version: v11.0 (advplan2 적용).

## Architecture

### Single-File Structure

Everything lives in `index.html`:
- **CSS** (lines 33–620): Design tokens in `:root`, responsive breakpoints at 768px/400px. Includes phase-hero, step-bar, law-alert, copilot-agent, pulse animation styles.
- **HTML** (lines 620–1270): Login screen → App shell (sidebar + topbar + content views) → Excel upload modal → 7 new independent phase views
- **JS** (lines 1270–8500): CONFIG → scoring constants → sample data → SP helpers → toast/modal/loading → data loading → login/nav → helpers → dashboard → archive → monthly → eval → teams → master (3-tab) → reports → Excel import/export → AI → monkey-patches → **7 independent phase page renderers** → archive/snapshot system

### Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                    index.html (Single-File SPA)                 │
├─────────────────────────────────────────────────────────────────┤
│  CSS (~600 lines)                                               │
│  ├── Design tokens (:root variables)                            │
│  ├── Layout (sidebar, topbar, split-view, form-panel)           │
│  ├── Components (kpi-card, status-chip, level-badge, heatmap)   │
│  └── Responsive (768px mobile, 400px small phone)               │
├─────────────────────────────────────────────────────────────────┤
│  HTML (~650 lines)                                              │
│  ├── Login screen (role/team selection)                         │
│  ├── App shell                                                  │
│  │   ├── Sidebar (collapsible sections, phase submenu)          │
│  │   ├── Topbar (year/month selector, export)                   │
│  │   └── Content views (19 view divs)                           │
│  ├── Mobile (drawer, bottom-sheet, bottom-nav)                  │
│  └── Excel upload modal                                         │
├─────────────────────────────────────────────────────────────────┤
│  JS (~7,100 lines)                                              │
│  ├── CONFIG + Scoring Constants                                 │
│  │   ├── X_MATRIX (5×5), Z_MATRIX (5×5)                        │
│  │   ├── A_LABELS, B_LABELS, A_CRITERIA, B_CRITERIA             │
│  │   ├── Y_LABELS, Y_CRITERIA, LEVEL_COLOR, LEVEL_ORDER         │
│  │   └── DOMAINS (14 categories)                                │
│  ├── SAMPLE_RISKS (8 risks, new data model)                    │
│  ├── SharePoint REST helpers (spGet/spPost/spPatch)             │
│  ├── UI Framework (toast, modal, loading, chart)                │
│  ├── Data Loading (_loadOfflineData with field defaults)        │
│  ├── Navigation (buildNav, navigateTo, collapsible sections)    │
│  ├── Dashboard (KPI 4-card, heatmap 5×5, action banner)        │
│  ├── Master (3-tab: list/assess/history)                        │
│  ├── Phase Pages (7 independent views)                          │
│  │   ├── Phase 1: identify-opinion / identify-confirm           │
│  │   ├── Phase 3: control-input / control-eval                  │
│  │   ├── Phase 4: residual (standalone)                         │
│  │   └── Phase 5: improve-submit / improve-confirm              │
│  ├── Monthly/Eval/Teams/MyTeam/Reports                          │
│  ├── Excel Import/Export (Korean column fallback)               │
│  ├── Archive/Snapshot system                                    │
│  └── Monkey-patches (v10.1+)                                   │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────── Data Flow ───────────────────┐
│                                                  │
│  SAMPLE_RISKS ──┐                                │
│                 ├──→ RISKS[] (in-memory)          │
│  localStorage ──┘        │                       │
│                          ├──→ render*() functions │
│                          └──→ _lsSave() → localStorage
│                                                  │
│  SharePoint ←──→ spGet/spPost/spPatch (online)   │
│  Power Automate ←── callFlow() (email triggers)  │
└──────────────────────────────────────────────────┘

┌──────────── 7-Phase Annual Cycle ────────────┐
│                                               │
│  Phase 1: 리스크 식별                          │
│    CP팀: 엑셀 업로드 → 배당 메일               │
│    현업: 의견 제출 (이의없음/추가/수정/삭제)     │
│    CP팀: 수용/기각 → 완료 확정                  │
│           ↓                                   │
│  Phase 2: 리스크 평가                          │
│    CP팀: A(1~5) × B(1~5) → X(VL~VH) 매트릭스  │
│           ↓                                   │
│  Phase 3: 통제수준 진단                        │
│    현업: 통제수단 입력                          │
│    CP팀: Y(VL~VH) 평가 → Z = X×Y 자동 계산     │
│           ↓                                   │
│  Phase 4: 잔여리스크 확정                       │
│    CP팀: Z 기준 관리대상 선정 → 최종 확정        │
│           ↓                                   │
│  Phase 5: 개선방안 도출·확정                    │
│    현업: 개선방안 제출                          │
│    CP팀: 확정/반려 → Phase 5 완료               │
│           ↓                                   │
│  Phase 6: 개선방안 이행 (확정 건만 대상)         │
│  Phase 7: 점검·보고                            │
└───────────────────────────────────────────────┘
```

### CONFIG & Dual Mode

```javascript
const CONFIG = {
  SP_SITE_URL, LIST_RISK_MASTER, LIST_MONTHLY, LIST_EVALUATION,
  FLOW_SUBMIT_URL, FLOW_SAVE_URL, FLOW_REVISION_URL,
  FLOW_NUDGE_URL, FLOW_ASSIGN_NOTIFY_URL, FLOW_IDENTIFY_OPINION_URL,
  FLOW_CONTROL_INPUT_URL, FLOW_RESIDUAL_CONFIRM_URL, FLOW_IMPROVE_CONFIRM_URL,
  COPILOT_AGENT_URL, LIST_RISK_HISTORY, SP_ARCHIVE_FOLDER, MAX_LOCAL_SNAPSHOTS,
  OFFLINE_MODE: true,
};
```

### 5-Level Scoring System (VL/L/M/H/VH)

```
X(고유리스크) = X_MATRIX[A발생가능성][B영향심각성]  (5×5 lookup)
Z(잔여리스크) = Z_MATRIX[X레벨][Y통제효과성]       (5×5 lookup)

Level colors:  VH=#EF4444  H=#F97316  M=#F59E0B  L=#84CC16  VL=#22C55E
Y colors (반전): VH=#22C55E  H=#84CC16  M=#F59E0B  L=#F97316  VL=#EF4444
```

### Risk Data Model (advplan2)

```javascript
{
  id, domain, lawName, riskType, riskContent, relatedClause, relatedOrg,
  title, team, cat, owner, score,                    // legacy compat
  a_likelihood(1~5), b_impact(1~5), x_level,         // Phase 2
  controlMeasure, controlMeasureStatus,               // Phase 3
  y_level, y_evalComment, z_level, z_selected,        // Phase 3-4
  identifyOpinionType/Status, identifyConfirmStatus,  // Phase 1
  improvePlan, improvePlanStatus, improvePlanComment,  // Phase 5
  submitStatus, achieve,                              // Phase 6
  changeLog[], _newRiskRequest,                       // audit trail
}
```

### Navigation / Routing

SPA routing via `navigateTo(id)`. 19 page IDs:

**Admin**: `dash`, `phasecontrol`, `phase1~7`, `master`, `identify-confirm`, `control-eval`, `residual`, `improve-confirm`, `monthly`, `eval`, `teams`, `reports`, `archive`

**User**: `myteam`(첫 화면), `identify-opinion`, `control-input`, `improve-submit`, `monthly`, `evalresult`, `archive`

Sidebar sections with collapsible toggle (`toggleSbSection`): "위험평가 작업" (admin/user), "이행 관리" (user)

### Key Pages & Functions

| Page | ID | Render | Role |
|------|----|--------|------|
| Dashboard | `dash` | `initDash()` | KPI 4-card (법령/잔여리스크/달성율/업무대기), heatmap 5×5 |
| Phase Control | `phasecontrol` | `renderPhaseControl()` | 7-phase overview with accordion submenu |
| Master | `master` | `applyMaster()` | 3-tab: 리스크목록 / A·B평가 / 변경이력·복원 |
| Identify Opinion | `identify-opinion` | `renderIdentifyOpinionView()` | 팀 의견 제출 (추가요청 폼 포함) |
| Identify Confirm | `identify-confirm` | `renderIdentifyConfirmView()` | CP팀 검토 (필터탭 + 확정 섹션) |
| Control Input | `control-input` | `renderControlInputView()` | 팀 통제수단 입력 |
| Control Eval | `control-eval` | `renderControlEvalView()` | CP팀 Y 평가 |
| Residual | `residual` | `renderResidualViewNew()` | Z 확정 (매트릭스 가이드 + 관리대상 선정) |
| Improve Submit | `improve-submit` | `renderImproveSubmitView()` | 팀 개선방안 제출 |
| Improve Confirm | `improve-confirm` | `renderImproveConfirmView()` | CP팀 확정/반려 + Phase 5 완료 |
| My Team | `myteam` | `renderMyTeam()` | 할 일 알림 배너 + KPI + 리스크 현황 |
| Archive | `archive` | `initArchive()` | Admin: 파일보관/법령/Copilot, User: 자료실 |

## Development

### Running Locally

Open `index.html` directly in a browser. No server required for offline mode.

### CDN Dependencies (lazy-loaded)

- **SheetJS** (xlsx 0.18.5) — Excel import/export
- **PptxGenJS** (3.12.0) — PPT generation

### Key Documents

- `phase1.md ~ phase5.md` — Phase별 프로세스 설명서 (현재 구현 기준)
- `advplan2.md` — 위험평가 프로세스 2차 개편 설계서
- `SETUP.md` — SharePoint/Power Automate 연동 가이드

## Conventions

- All user-facing text is in Korean
- 5-level system: VL/L/M/H/VH with matrix lookup (not multiplication)
- Status values: `제출완료`, `평가완료`, `미제출`, `임시저장`, `확정`, `반려`
- Phase 6 target filter: `z_selected && improvePlanStatus === '확정'`
- CSS color tokens: `var(--accent)`, `var(--green)`, `var(--amber)`, `var(--red)`
- localStorage: `riskops_{year}_{month}_{riskId}` + `riskops_archive_` snapshots
- Archive snapshots: max 10 in localStorage, phase-complete preserved
- Mobile: `isMobile()` → `openSheet()` for form panels
- form-panel: `form-hd/form-body/form-footer` 3-part structure for scroll
