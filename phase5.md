# Phase 5 — 개선방안 도출·확정 프로세스 설명서

> 최종 업데이트: 2026-03-25 (현재 구현 상태 기준)

---

## 1. 업무 흐름

```
Phase 4 완료 (z_selected = true인 관리 대상 확정)
    ↓
[팀 담당자] "개선방안 제출" 페이지 진입
  - 내 팀 관리 대상 리스크 목록 (z_selected && 내 팀)
  - 안내: "Phase 4에서 확정된 관리 대상에 개선방안을 작성해 제출해주세요"
  - 각 리스크에 개선방안 작성 → 임시저장 또는 제출
  - 반려된 건: 빨간 테두리 + CP팀 반려 사유 표시 → 수정 후 재제출
    ↓
[CP팀] "개선방안 확정" 페이지 진입
  - 진행률 바: 확정 비율
  - 팀 제출 개선방안 내용 검토
  - 통제수단 참고 표시 (Phase 3 입력 내용)
  - 확정 또는 반려 (반려 시 검토 의견 필수)
    ↓
[CP팀] 전체 확정 완료 → "Phase 5 완료 확정" 버튼 활성화
  - 미확정 건 있으면 버튼 비활성화 + 안내 문구
  - 확정 시 Phase 6(이행 관리) 활성화
  - 사이드바 "위험평가 작업" 섹션 자동 접힘
    ↓
[Phase 6] 확정된 개선방안(improvePlanStatus === '확정')만 이행 관리 대상
```

---

## 2. 관련 페이지

| 페이지 | view ID | 접근 권한 | 역할 |
|--------|---------|----------|------|
| 개선방안 제출 | `view-improve-submit` | 팀 담당자 | 개선방안 작성·제출 |
| 개선방안 확정 | `view-improve-confirm` | CP팀 | 검토·확정·반려 + Phase 5 완료 |

---

## 3. 주요 함수

### 팀 담당자용

| 함수명 | 역할 |
|--------|------|
| `renderImproveSubmitView()` | 제출 화면 (안내배너 + 진행률 + 반려 표시 + split-view) |
| `selectImproveSubmitItem(riskId)` | 우측 폼 (X→Z 배지 + 통제수단 참고 + 반려 안내 + 확정 잠금) |
| `saveImprovePlanNew(riskId, isDraft)` | 저장 (최소 10자 검증, 리스크별 고유 textarea ID) |

### CP팀용

| 함수명 | 역할 |
|--------|------|
| `renderImproveConfirmView()` | 확정 화면 (진행률 + split-view + Phase 5 완료 확정 섹션) |
| `selectImproveConfirmItem(riskId)` | 검토 폼 (통제수단 참고 + 팀 개선방안 + 검토 의견 + 확정/반려) |
| `confirmImprovePlanNew(riskId, status)` | 확정('확정') 또는 반려('반려') 처리 (반려 시 의견 필수) |
| `completePhase5()` | Phase 5 완료 확정 + 사이드바 자동 접기 |

### 유틸

| 함수명 | 역할 |
|--------|------|
| `_getPhase6Targets()` | Phase 6 이행 대상 필터 (z_selected && improvePlanStatus === '확정') |

---

## 4. 데이터 필드

```js
{
  improvePlan:         '개선방안 내용 텍스트',
  improvePlanStatus:   '확정' | '제출완료' | '반려' | '임시저장' | '미제출',
  improvePlanAt:       'ISO datetime',       // 제출 시각
  improvePlanComment:  'CP팀 검토/반려 의견',
}
```

---

## 5. Phase 5 완료 조건

| # | 조건 | 판정 방식 |
|---|------|----------|
| 1 | 팀 개선방안 전수 제출 완료 | 자동 (모든 z_selected 건 improvePlanStatus ≠ '미제출') |
| 2 | CP팀 개선방안 전수 확정 | 자동 (모든 z_selected 건 improvePlanStatus === '확정') |

---

## 6. Phase 5 → Phase 6 연결

Phase 5 완료 후:
- **이행현황 입력** (`renderMonthly`): `z_selected && improvePlanStatus === '확정'` 리스크만 표시
- **평가결과·이력** (`renderEvalResult`): 동일 필터 적용
- **내 팀 현황** (`renderMyTeam`): 동일 필터 적용
- Phase 5 미완료 시: 기존처럼 전체 RISKS 표시 (호환성)

---

## 7. UI 특징

### 팀 담당자
- **반려 건 강조**: 빨간 테두리 + CP팀 반려 사유 미리보기 + "재작성 후 재제출해주세요" 안내
- **확정 건 잠금**: textarea readonly + "✅ CP팀 확정 완료" 배지
- **통제수단 참고**: 현재 통제수단(Phase 3 입력) 읽기전용 박스로 참고 표시

### CP팀
- **진행률 바**: 확정 비율 (녹색)
- **제출완료 건 강조**: amber 테두리
- **Phase 5 완료 섹션**: 전수 확정 시 녹색 배경 + 활성화 버튼, 미완료 시 비활성 + 남은 건수

---

## 8. 구버전 함수 정리

Phase 5는 구버전(`improve-interactive` div 기반)과 신버전(`view-improve-submit/confirm` 기반)이 공존했던 이력이 있습니다.

**삭제된 구버전 함수:**
- `renderImproveView()` → stub만 남음 (`improve-interactive` div 제거됨)
- `switchImproveTab()`, `selectImprove()`, `saveImproveDraft()`, `submitImprovePlan()`
- `selectImproveConfirm(riskIdx)`, `confirmImprovePlan(riskIdx)` (인덱스 기반)

**현재 사용 중인 신버전:**
- `selectImproveSubmitItem(riskId)` / `saveImprovePlanNew(riskId, isDraft)` (ID 기반)
- `selectImproveConfirmItem(riskId)` / `confirmImprovePlanNew(riskId, status)` (ID 기반)

---

## 9. 최근 업데이트 이력

| 날짜 | 내용 |
|------|------|
| 2026-03-25 | 구버전 함수 전체 삭제 (이중 정의 버그 해결) |
| 2026-03-25 | renderImproveSubmitView 전면 개선 (안내배너 + 진행률 + 반려 표시) |
| 2026-03-25 | selectImproveSubmitItem 반려 상태 안내 + 확정 잠금 |
| 2026-03-25 | renderImproveConfirmView 전면 개선 (진행률 + Phase 5 완료 섹션) |
| 2026-03-25 | confirmImprovePlanNew 반려 시 의견 필수 검증 |
| 2026-03-25 | completePhase5() Phase 5 완료 처리 + 사이드바 자동 접기 |
| 2026-03-25 | _getPhase6Targets() Phase 6 이행 대상 필터 함수 추가 |
| 2026-03-25 | 현업 전체 화면에 최종 선정 리스크 필터 일괄 적용 |
