# Phase 3 — 통제수준 진단 프로세스 설명서

> 최종 업데이트: 2026-03-25 (현재 구현 상태 기준)

---

## 1. 업무 흐름

```
Phase 2 완료 (전체 A/B/X 평가 완료)
    ↓
[CP팀] Phase 3 시작 → 통제수단 입력 요청 메일 발송 (sendControlInputNotify)
    ↓
[팀 담당자]
  - 작년 엑셀에서 불러온 경우: 기존 통제수단 내용이 채워져 있음
    → "작년 통제수단 그대로 유지" 체크박스 or 수정 후 제출
  - 신규 리스크: 빈칸 → 새로 작성
  - 제출 완료 후 수정 필요 시 "수정 요청" 버튼으로 재오픈
    ↓
[CP팀] 팀이 입력한 통제수단 내용을 보고 효과성(Y) 평가
  - Y = VL / L / M / H / VH (각 레이블 + 기준 한 줄 표시)
  - Z = zLevelFromMatrix(X, Y) 자동 계산 + 관리대상 자동 판정
    ↓
[CP팀] 전수 Y 평가 완료 → Phase 3 완료 확정 → Phase 4 활성화
```

---

## 2. 관련 페이지

| 페이지 | view ID | 접근 권한 | 역할 |
|--------|---------|----------|------|
| 통제수단 입력 | `view-control-input` | 팀 담당자 | 통제수단 확인·입력·제출 |
| 통제효과성 평가 | `view-control-eval` | CP팀 | Y 효과성 평가, Z 자동 계산 |

---

## 3. Y 통제효과성 기준

| 등급 | 레이블 | 색상 | 기준 |
|------|--------|------|------|
| VH | 매우 높음 | 녹색 | KPI 모니터링 + 경영자 보고 + 감사 + CP 사전검토 + 경영시스템 인증 |
| H | 높음 | 연초록 | 법무·CP팀 이중검토 또는 별도 위원회 검토 |
| M | 보통 | 황색 | 문서화된 절차 + 주기적 교육 + 투명성 통제체계 |
| L | 낮음 | 주황 | 통제수단 없으나 담당 업무가 조직 내 할당 |
| VL | 매우 낮음 | 빨강 | 해당 의무에 대한 통제수단이 없음 |

> **주의**: Y는 높을수록 통제 우수 → 잔여리스크(Z) 낮아짐. 색상이 X/Z와 반대.

### Z 잔여리스크 매트릭스 (X × Y → Z)

```
        X=VH   X=H   X=M   X=L   X=VL
Y=VH     L     VL    VL    VL    VL
Y=H      M      L    VL    VL    VL
Y=M      H      M     L    VL    VL
Y=L     VH      H     M     L    VL
Y=VL    VH      H     M     L    VL
```

---

## 4. 주요 함수

### 팀 담당자용

| 함수명 | 역할 |
|--------|------|
| `renderControlInputView()` | 통제수단 입력 화면 (안내 배너 + 진행률 바 + split-view) |
| `selectControlInput(riskId)` | 우측 폼 (X등급+A/B + 작년 내용 안내 + "그대로 유지" 체크 + Y기준 참고) |
| `saveControlInput(riskId, isDraft)` | 저장 (그대로 유지 옵션 지원, 최소 10자 검증) |
| `toggleControlUnchanged(riskId, checked)` | "그대로 유지" 체크 시 textarea 비활성화 |
| `reopenControlInput(riskId)` | 제출 완료 건 수정 모드 전환 |

### CP팀용

| 함수명 | 역할 |
|--------|------|
| `renderControlEvalView()` | Y 평가 화면 (팀별 입력 현황 표 + 진행률 바 + split-view) |
| `selectControlEval(riskId)` | Y 평가 폼 (통제수단 내용 + Y 5단계 버튼 + Z 미리보기 + 평가 의견) |
| `selectCEY(lv, el, riskId)` | Y 버튼 선택 시 Z 자동 계산 + 관리대상 여부 표시 |
| `saveControlEval(riskId)` | Y + Z 저장 + z_selected 자동 판정 |
| `sendControlInputNotify()` | 팀에 통제수단 입력 요청 메일 발송 |

---

## 5. 데이터 필드

```js
{
  controlMeasure:       '현재 운영 중인 통제 활동 설명',
  controlMeasureStatus: '입력완료' | '임시저장' | '미입력',
  y_level:              'VL' | 'L' | 'M' | 'H' | 'VH' | '',
  y_evalComment:        'CP팀 Y 평가 의견',
  z_level:              'VL' | 'L' | 'M' | 'H' | 'VH' | '',  // 자동 계산
  z_selected:           true | false,  // ['H','VH'].includes(z_level) 시 자동 true
}
```

---

## 6. Phase 3 완료 조건

| # | 조건 | 판정 방식 |
|---|------|----------|
| 1 | 팀 통제수단 입력 완료 (80% 이상) | 자동 (inputted/target ≥ 0.8) |
| 2 | CP팀 통제효과성(Y) 평가 완료 (전수) | 자동 (evaluated === target.length) |

> 통제수단 입력은 **80%** — 미입력 건은 CP팀이 Y=VL로 평가 가능. Y 평가는 **전수 필수**.

---

## 7. UI 특징

- **팀 담당자**: 작년 내용 자동 채움 + "그대로 유지" 체크박스 + Y 기준 상시 참고
- **CP팀**: 팀별 입력 현황 표 (전체/입력완료/임시저장/미입력/Y평가완료)
- **Y 버튼**: 세로 배열, 각 버튼에 레이블 + 기준 한 줄 표시 (VL~VH)
- **Z 미리보기**: Y 선택 즉시 Z 계산 + "관리 대상" 또는 "관리 대상 아님" 표시
- **통제수단 미입력 건**: 빨간 테두리 + "VL 선택 권장" 안내

---

## 8. 최근 업데이트 이력

| 날짜 | 내용 |
|------|------|
| 2026-03-25 | Y_LABELS 상수 추가 (색상이 X/Z와 반전) |
| 2026-03-25 | sendControlInputNotify() 함수 신규 추가 |
| 2026-03-25 | renderControlInputView 전면 개선 (안내배너 + 진행률 + 상태구분) |
| 2026-03-25 | "작년 통제수단 그대로 유지" 체크박스 추가 |
| 2026-03-25 | reopenControlInput() 제출 후 수정 기능 |
| 2026-03-25 | renderControlEvalView 전면 개선 (팀별 현황표 + 전체 대상 표시) |
| 2026-03-25 | Y 버튼에 레이블 + 기준 한 줄 추가 (세로 배열) |
| 2026-03-25 | 통제수단 미입력 건 안내 (VL 선택 권장) |
| 2026-03-25 | 완료 조건: 통제수단 80% + Y 전수 |
