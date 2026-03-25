# Phase 1 — 리스크 식별 프로세스 설명서

> 최종 업데이트: 2026-03-25 (현재 구현 상태 기준)

---

## 1. 업무 흐름

```
[CP팀 관리자]
1. 엑셀 업로드 → 리스크 Master 등록 (500건+)
   - 한글 컬럼명 자동 파싱 (영역/법규명/Risk내용/A/B/X/Y/Z 등)
   - 업로드 옵션: "이전 연도 평가값 유지" or "리스크 목록만 가져오기"
   - 임포트 완료 시 팀별 배정 현황 요약 배너 표시
2. 배당 확인 메일 발송 (sendAssignNotify → Power Automate)
   ↓
[팀 담당자 (현업)]
3. "내 팀 현황" 첫 화면에 "💬 리스크 식별 의견 N건 제출 필요" 알림 표시
4. "리스크 식별 의견 제출" 페이지 진입
   - 배당된 리스크 목록 확인
   - 의견 유형 선택: 이의없음 / 추가요청 / 수정요청 / 삭제요청
   - "추가요청" 선택 시 새 리스크 입력 폼 표시 (영역/법규명/내용/조항)
   - 의견 내용 입력 → 임시저장 또는 제출
   - 제출 완료 시 CP팀 알림 발송 (FLOW_IDENTIFY_OPINION_URL)
   ↓
[CP팀 관리자]
5. "식별 의견 검토·확정" 페이지 진입
   - 팀별 의견 제출 현황 요약 테이블 (전체/이의없음/의견있음/미처리/미제출)
   - 필터 탭: 전체 제출 / 의견있음 / 이의없음
   - 미처리 건: 빨간 배경 + NEW 배지(깜빡임)으로 강조
   - 수용/기각 처리 + 처리 의견 입력
   - 추가요청 수용 시 새 리스크 자동 생성 (RISKS 배열에 추가)
   - 오처리 시 "처리 취소 (재검토)" 버튼으로 복원 가능
6. 전체 의견 처리 완료 → "Phase 1 완료 확정" 버튼 클릭
   - 아카이브 스냅샷 자동 저장 → Phase 2 활성화
```

---

## 2. 관련 페이지

| 페이지 | view ID | 접근 권한 | 역할 |
|--------|---------|----------|------|
| 리스크 Master | `view-master` | CP팀 | 엑셀 업로드, 리스크 등록·수정·삭제 |
| 리스크 식별 의견 제출 | `view-identify-opinion` | 팀 담당자 | 배당 리스크 확인, 의견 제출 |
| 식별 의견 검토·확정 | `view-identify-confirm` | CP팀 | 팀 의견 수용/기각, Phase 1 완료 확정 |

---

## 3. 주요 함수

| 함수명 | 역할 |
|--------|------|
| `renderIdentifyOpinionView()` | 팀 담당자용 의견 제출 화면 렌더링 (안내 박스 + 진행률 바 + split-view) |
| `selectIdentifyOpinion(idx)` | 리스크 클릭 시 우측 폼 렌더링 (데스크탑: form-hd/body/footer, 모바일: openSheet) |
| `selectIOType(type, el, riskId, idx)` | 의견 유형 칩 선택 + 추가요청 폼 토글 |
| `saveIdentifyOpinion(riskId, isDraft)` | 의견 저장 (추가요청 시 `_newRiskRequest` 포함) + Flow 알림 |
| `renderIdentifyConfirmView()` | CP팀용 검토 화면 (팀별 현황 표 + 필터 탭 + 확정 섹션) |
| `selectIdentifyConfirm(riskId)` | 검토 폼 렌더링 (수용/기각 + Master 이동 버튼) |
| `confirmIdentifyOpinion(riskId, status)` | 수용/기각 처리 (추가요청 수용 시 새 리스크 자동 생성) |
| `reopenIdentifyConfirm(riskId)` | 처리 완료 건 재검토 (identifyConfirmStatus → 'pending') |
| `confirmPhase1Complete()` | Phase 1 완료 확정 + 아카이브 스냅샷 저장 |
| `_filterIdentifyList(mode, btn)` | 좌측 목록 필터 (전체/의견있음/이의없음) |
| `_renderIdentifyListItems(list)` | 목록 항목 렌더링 (미처리: 빨간 배경+NEW, 수용: 녹색) |

---

## 4. 데이터 필드

```js
// 리스크 객체 Phase 1 관련 필드
{
  identifyOpinionType:     '이의없음' | '추가요청' | '수정요청' | '삭제요청' | '',
  identifyOpinionStatus:   '제출완료' | '임시저장' | '미제출',
  identifyOpinion:         '의견 내용 텍스트',
  identifyOpinionAt:       'ISO datetime',
  identifyConfirmStatus:   'pending' | 'accepted' | 'rejected',
  identifyConfirmComment:  'CP팀 처리 의견',
  _newRiskRequest: {       // 추가요청 시에만 존재
    domain, lawName, riskContent, relatedClause, requestedBy, requestedAt
  },
  _deleteRequested:        true | undefined,   // 삭제요청 수용 시
}
```

---

## 5. Phase 1 완료 조건 (_checkPhaseConditions)

| # | 조건 | 판정 방식 |
|---|------|----------|
| 1 | 리스크 1건 이상 등록 (엑셀 업로드) | 자동 (RISKS.length ≥ 1) |
| 2 | 배당 확인 메일 발송 | 수동 체크 (assignNotifySent) |
| 3 | 현업 의견 전수 제출 완료 | 자동 (모든 리스크 identifyOpinionStatus === '제출완료') |
| 4 | 현업의견 전수 검토 처리 완료 | 자동 (의견있는 건 중 pending === 0) |

---

## 6. 최근 업데이트 이력

| 날짜 | 내용 |
|------|------|
| 2026-03-25 | 구버전 renderSelection monkey-patch 삭제 (스크립트 중단 버그 해결) |
| 2026-03-25 | canConfirm 조건에 notSubmitted 검증 추가 (미제출 시 확정 차단) |
| 2026-03-25 | 의견 유형 칩 제출 후 disabled 처리 (타입 오염 방지) |
| 2026-03-25 | 추가요청 시 새 리스크 입력 폼 추가 (영역/법규명/내용/조항) |
| 2026-03-25 | CP팀 수용 시 새 리스크 자동 RISKS 등록 |
| 2026-03-25 | 미처리 건 빨간 배경 + NEW 깜빡임 배지 강조 |
| 2026-03-25 | 팀별 현황 테이블에 "미처리" 컬럼 추가 |
| 2026-03-25 | 처리 완료 건 "재검토" 버튼 + Master 이동 버튼 추가 |
| 2026-03-25 | 이의없음 건 필터 탭 추가 (전체/의견있음/이의없음) |
| 2026-03-25 | 모바일 openSheet 분기 + BNAV_USER에 의견제출 탭 추가 |
| 2026-03-25 | form-panel overflow:auto 수정 (스크롤 가능) |
| 2026-03-25 | _loadOfflineData에 신규 필드 기본값 보장 (구 localStorage 호환) |
