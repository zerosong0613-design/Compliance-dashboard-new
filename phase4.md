# Phase 4 — 잔여리스크 확정 프로세스 설명서

> 최종 업데이트: 2026-03-25 (현재 구현 상태 기준)

---

## 1. 업무 흐름

```
Phase 3 완료 (전체 Y 평가 → Z 자동 계산 완료)
    ↓
[CP팀] "잔여리스크 확정" 페이지 진입
  - Z 레벨 내림차순 정렬된 전체 리스크 확인 (VH → H → M → L → VL)
  - Z 매트릭스 기준 참고 패널로 X×Y→Z 산출 근거 확인
  - Z별 분포 KPI 카드 (VH N건 / H N건 / M N건 / L N건 / VL N건)
    ↓
[CP팀] 관리 대상 선정
  - "H·VH 자동 선정" 버튼 → z_level이 H 또는 VH인 건 자동 체크
  - 수동 체크박스로 개별 조정 가능
  - 레벨 필터 탭: 전체 / VH / H / M / L·VL
    ↓
[CP팀] 사업팀 확인 요청 메일 발송 (선택)
    ↓
[CP팀] "최종 확정" 클릭
  - 사업팀 미발송 경고 모달 표시
  - 확정 확인 모달 (확정 후에도 수정 가능 안내)
  - Phase 상태 저장 (residualConfirmed: true)
  - Phase 5 활성화
    ↓
[확정 후] 테이블 체크박스 비활성화 + 관리대상 행 강조 + 비선정 행 흐리게
  - "확정 목록 보기" 토글 → 관리대상 리스크 요약 카드
  - "수정 (재오픈)" 버튼으로 확정 취소 가능
```

---

## 2. 관련 페이지

| 페이지 | view ID | 접근 권한 | 역할 |
|--------|---------|----------|------|
| 잔여리스크 확정 | `view-residual` | CP팀 | Z 분포 확인, 관리대상 선정, 최종 확정 |

---

## 3. 주요 함수

| 함수명 | 역할 |
|--------|------|
| `renderResidualViewNew()` | 확정 화면 전체 렌더링 (KPI + Z매트릭스 + 필터 + 테이블 + 확정 섹션) |
| `_residualMatrixGuideHtml()` | Z 매트릭스 참고 패널 HTML (5×5 표 + Y 기준 설명 + 관리대상 기준 안내) |
| `autoSelectResidualNew()` | H·VH 자동 선정 |
| `resetResidualSelectionNew()` | 전체 선정 초기화 |
| `filterResidual(f, btn)` | 테이블 Z 레벨 필터 (전체/VH/H/M/L·VL) |
| `saveResidualSelectionNew()` | 선정 상태 저장 |
| `confirmResidualFinal()` | 최종 확정 (2단계 모달 + Phase 상태 저장) |
| `toggleResidualSummary()` | 확정 완료 후 관리대상 요약 카드 펼침/접힘 |
| `reopenResidual()` | 확정 취소 → 재선정 모드 |
| `sendResidualConfirm()` | 사업팀 확인 요청 메일 발송 |

---

## 4. 데이터 필드

```js
{
  z_level:        'VL' | 'L' | 'M' | 'H' | 'VH',  // Phase 3에서 자동 계산됨
  z_selected:     true | false,                      // 관리 대상 선정 여부
  z_confirmedAt:  'ISO datetime',                    // 확정 시각
}
```

Phase 상태:
```js
phaseStatus.residual = {
  residualConfirmSent: true | false,  // 사업팀 메일 발송 여부
  residualConfirmed:   true | false,  // 최종 확정 여부
}
```

---

## 5. Phase 4 완료 조건

| # | 조건 | 판정 방식 |
|---|------|----------|
| 1 | Z(잔여리스크) 계산 완료 | 자동 (z_level 있는 건 ≥ 1) |
| 2 | 관리 대상 선정 | 자동 (z_selected 건 ≥ 1) |
| 3 | 사업팀 확인 요청 발송 | 수동 체크 (residualConfirmSent) |
| 4 | 최종 확정 완료 | 자동 (residualConfirmed — confirmResidualFinal에서 설정) |

---

## 6. UI 특징

- **Z별 분포 KPI 카드**: VH/H/M/L/VL 건수 + 관리대상 건수 시각화
- **Z 매트릭스 참고 패널**: X(고유)×Y(통제)→Z 계산 기준 5×5 표 + Y 기준 + 관리대상 기준
- **확정 전**: 체크박스 활성 + Z 레벨별 필터 + 자동선정/초기화 버튼
- **확정 후**: 체크박스 비활성 + 관리대상 행 강한 배경색 + 비선정 행 흐리게(opacity .55)
- **헤더 전환**: "선정" → "관리대상" (확정 후)
- **확정 취소**: "수정 (재오픈)" 버튼으로 Phase 5 잠금 + 재선정

---

## 7. 최근 업데이트 이력

| 날짜 | 내용 |
|------|------|
| 2026-03-25 | confirmResidualFinal Phase 상태 저장 버그 수정 (핵심 버그) |
| 2026-03-25 | 2단계 확인 모달 추가 (사업팀 미발송 경고 + 최종 확정) |
| 2026-03-25 | _residualMatrixGuideHtml() Z 매트릭스 참고 패널 추가 |
| 2026-03-25 | A/B 컬럼 테이블에 추가 (X 산출 근거 확인) |
| 2026-03-25 | 확정 후 관리대상/비선정 행 시각 강조 |
| 2026-03-25 | toggleResidualSummary() 확정 목록 보기 토글 추가 |
| 2026-03-25 | reopenResidual() 확정 취소 기능 추가 |
| 2026-03-25 | Phase 4 카드 3개 구성 (Z분포/메일발송/최종확정) |
