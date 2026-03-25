# 리스크 편집 모달 수정 의견 — modal.md

> 대상 함수: `openRiskEditModal(id)` · `confirmEditRisk(riskId)` · `openTaskQuickEdit(id)` · `saveTaskQuickEdit(riskId)`  
> 작성일: 2026-03-25

---

## 현재 문제 요약

스크린샷에서 보이는 모달의 필드 순서와 라벨이 두 가지 문제를 동시에 가지고 있습니다.

**문제 1 — 통제수단 입력 칸이 없음**

현재 모달 필드 순서:
```
A (발생가능성) → B (영향심각성) → X 자동계산 → Y (통제효과성) → Z 자동계산 → 담당자 → 개선 과제
```
X와 Y 사이에 **통제수단(controlMeasure)** 입력 필드가 없습니다.  
Y(통제효과성)를 평가하려면 "현재 어떤 통제수단이 있는가"를 먼저 입력해야 평가 근거가 생기는 구조임에도, 그 입력 칸이 빠져 있습니다.

**문제 2 — 통제수단과 개선과제가 하나로 뒤섞임**

스크린샷 하단에 `개선 과제 (통제수단) [입력됨]`으로 표시된 필드는 실제로 데이터 모델의 `r.task` 필드입니다.  
그런데 코드 Line 6998에서 라벨을 아래처럼 동적으로 덮어씁니다:

```js
labelEl.innerHTML = `개선 과제 (통제수단) <span ...>${badgeText}</span>`;
```

두 필드는 완전히 다른 개념입니다:

| 필드 | 데이터 키 | 입력 주체 | 입력 시점 | 의미 |
|------|----------|----------|----------|------|
| 통제수단 | `r.controlMeasure` | 팀 담당자 | Phase 3 | 현재 운영 중인 예방·점검 활동 |
| 개선과제 | `r.task` | CP팀 / 관리자 | Phase 4 확정 이후 | Z 확정 후 도출된 개선 목표 |

지금 UI에서는 이 둘을 하나의 필드에 혼용하여 표시 중 → **현업 담당자가 개선과제란에 통제수단을 입력하거나, 반대로 CP팀이 통제수단 위치에 개선과제를 기입하는 혼선이 발생**합니다.

---

## 수정 1 — `openRiskEditModal()` 필드 순서 및 구성 변경

### 수정 전 필드 순서 (Line 4448~4472)

```
X preview
Y (통제효과성) 칩
Z preview
담당자
개선 과제  ← r.task
점검 대상 체크박스
```

### 수정 후 필드 순서 (목표)

```
X preview
[NEW] 통제수단  ← r.controlMeasure  (Phase 3 · 팀 담당자 입력 영역)
Y (통제효과성) 칩
Z preview
담당자
[RENAMED] 개선과제  ← r.task  (Phase 4 확정 후 CP팀 기입 영역)
점검 대상 체크박스
```

### 삽입할 HTML 블록 — X preview 바로 아래, Y 칩 바로 위

```html
<!-- X preview 아래 이 블록 추가 -->
<div style="border:1px solid #BFDBFE;border-radius:10px;padding:14px;background:#F8FBFF">
  <div style="display:flex;align-items:center;gap:8px;margin-bottom:8px">
    <label style="font-size:12px;font-weight:700;color:#1D4ED8">
      🔍 통제수단 <span style="font-weight:400;color:var(--muted)">(Phase 3 · 팀 담당자 입력)</span>
    </label>
    <!-- 입력 상태 뱃지: r.controlMeasureStatus === '입력완료' 여부로 분기 -->
    ${r.controlMeasureStatus === '입력완료'
      ? `<span style="font-size:11px;font-weight:700;padding:2px 8px;border-radius:99px;background:#ECFDF5;color:var(--green)">✓ 입력됨</span>`
      : `<span style="font-size:11px;font-weight:700;padding:2px 8px;border-radius:99px;background:#FEF9EE;color:var(--amber)">미입력</span>`
    }
  </div>
  <textarea id="edit-risk-control"
    style="width:100%;padding:10px 12px;border:1px solid #BFDBFE;border-radius:8px;font-size:13px;font-family:inherit;outline:none;height:80px;resize:vertical;background:#fff"
    placeholder="예: 월 1회 점검 실시, 분기별 교육 이수율 95% 유지, 절차서 v2.1 준수"
    onfocus="this.style.borderColor='var(--accent)'"
    onblur="this.style.borderColor='#BFDBFE'"
  >${r.controlMeasure || ''}</textarea>
  <div style="font-size:11px;color:var(--muted);margin-top:5px">
    현재 운영 중인 예방·점검 활동을 구체적으로 작성하세요. Y(통제효과성) 평가의 근거가 됩니다.
  </div>
</div>
```

### 개선과제 라벨 변경 — Phase 4 영역임을 명확히 표시

현재 (Line 4469):
```html
<label ...>개선 과제</label>
```

수정 후:
```html
<label ...>
  📋 개선과제 <span style="font-weight:400;color:var(--muted)">(Phase 4 확정 후 · CP팀 기입)</span>
</label>
```

개선과제 textarea에 placeholder도 변경:
```
placeholder="Z(잔여리스크) 확정 후 도출된 개선 목표를 입력하세요&#10;예: 분기별 동의서 양식 점검 + 마케팅 수신 동의 프로세스 자동화"
```

---

## 수정 2 — `confirmEditRisk()` 저장 로직에 controlMeasure 추가

현재 (Line 4530~4566): `task`(개선과제)는 저장하지만 `controlMeasure`는 저장하지 않음.

### 추가할 읽기 코드

```js
// 기존 task 읽기 코드 아래에 추가
const controlMeasure = document.getElementById('edit-risk-control')?.value.trim() || '';
const controlMeasureStatus = controlMeasure ? '입력완료' : (r.controlMeasureStatus || '미입력');
```

### Object.assign 수정

현재:
```js
Object.assign(r, { title, riskContent: title, team, cat, ..., task, selected, ... });
```

수정 후:
```js
Object.assign(r, {
  title, riskContent: title, team, cat, ..., task, selected,
  controlMeasure,                    // ← 추가
  controlMeasureStatus,              // ← 추가
  ...
});
```

### SP 연동 시 (비오프라인 모드) 패치 항목 추가

```js
await spPatch(CONFIG.LIST_RISK_MASTER, r.spId, {
  ...,
  ControlMeasure: controlMeasure,           // ← 추가
  ControlMeasureStatus: controlMeasureStatus // ← 추가
});
```

---

## 수정 3 — `openTaskQuickEdit()` 라벨 혼용 수정

**위치**: Line 6853, 6998

### 현재 (두 곳 모두)
```js
`개선 과제 (통제수단)`
```

이 라벨이 두 다른 개념을 하나로 묶고 있어서 혼란을 야기합니다.

### 수정 방향

`openTaskQuickEdit()`는 `r.task`(개선과제) 전용 퀵에딧 모달이므로 **통제수단 참조를 완전히 제거**합니다.

Line 6853:
```html
<!-- 수정 전 -->
<label ...>개선 과제 (통제수단)</label>

<!-- 수정 후 -->
<label ...>📋 개선과제 <span style="font-size:11px;color:var(--muted);font-weight:400">(Phase 4 · CP팀 기입)</span></label>
```

Line 6998:
```js
// 수정 전
labelEl.innerHTML = `개선 과제 (통제수단) <span ...>${badgeText}</span>`;

// 수정 후
labelEl.innerHTML = `📋 개선과제 <span style="font-size:11px;color:var(--muted);font-weight:400">(Phase 4)</span> <span class="phase-task-badge" style="${badgeColor}">${badgeText}</span>`;
```

Line 7004 — 힌트 텍스트도 수정:
```js
// 수정 전
hint.innerHTML = `<strong>Phase 3 · 통제수준 진단 중</strong><br>이 리스크에 대해 현재 시행 중인 통제수단(예방 활동, 점검 절차 등)을 기록하세요.`;

// 수정 후
hint.innerHTML = `<strong>Phase 4 · 잔여리스크 확정 후</strong><br>Z(잔여리스크)가 확정된 이후, 관리 대상 리스크에 대한 구체적인 개선 목표와 조치 계획을 기입하세요.`;
```

---

## 수정 후 모달 최종 필드 구조

```
┌─────────────────────────────────────────┐
│ 리스크명 *                               │
├─────────────────────────────────────────┤
│ 팀               │  유형                │
├─────────────────────────────────────────┤
│ 발생가능성 A (1~5)  [칩 선택]             │
├─────────────────────────────────────────┤
│ 영향심각성 B (1~5)  [칩 선택]             │
├─────────────────────────────────────────┤
│ X(고유리스크) =  [자동 계산 뱃지]          │
├─────────────────────────────────────────┤  ← 여기에 삽입
│ 🔍 통제수단  (Phase 3 · 팀 담당자 입력)   │  ← NEW
│   [textarea — r.controlMeasure]         │
│   [상태 뱃지: 입력됨 / 미입력]             │
├─────────────────────────────────────────┤
│ 통제효과성 Y  [칩 선택]                   │
├─────────────────────────────────────────┤
│ Z(잔여리스크) =  [자동 계산 뱃지]          │
├─────────────────────────────────────────┤
│ 담당자                                   │
├─────────────────────────────────────────┤
│ 📋 개선과제  (Phase 4 확정 후 · CP팀 기입) │  ← 라벨 변경
│   [textarea — r.task]                   │
├─────────────────────────────────────────┤
│ ☑ 이번 연도 점검 대상으로 선정             │
└─────────────────────────────────────────┘
```

---

## 수정 체크리스트

- [ ] `openRiskEditModal()` — X preview 아래에 `통제수단` textarea 블록 삽입 (`id="edit-risk-control"`)
- [ ] `openRiskEditModal()` — `개선 과제` 라벨을 `개선과제 (Phase 4 확정 후 · CP팀 기입)`으로 수정
- [ ] `confirmEditRisk()` — `controlMeasure` / `controlMeasureStatus` 읽기 및 `Object.assign` 반영
- [ ] `confirmEditRisk()` — SP 비오프라인 모드 패치에 `ControlMeasure` 필드 추가
- [ ] `openTaskQuickEdit()` Line 6853 — 라벨에서 `(통제수단)` 제거, Phase 4 명시
- [ ] `openTaskQuickEdit()` Line 6998 — `labelEl.innerHTML` 동적 라벨에서 `통제수단` 참조 제거
- [ ] `openTaskQuickEdit()` Line 7004 — 힌트 텍스트를 Phase 4 맥락으로 교체
