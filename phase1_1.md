# Phase 1 버그·개선 분석 — phase1_1.md

> 분석 대상: `index.html` (Compliance RiskOps)  
> 작성일: 2026-03-25  
> 범위: Phase 1 (리스크 식별) — 엑셀 등록 → 팀 배당 메일 → 현업 의견 제출 → CP팀 검토·확정  

---

## 플로우 요약 (설계 의도)

```
[CP팀 관리자]
1. 엑셀 업로드 → 리스크 Master 등록
2. 팀별 배당 확인 요청 메일 발송 (sendAssignNotify)
   ↓
[팀 담당자 (현업)]
3. 배당된 리스크 확인 → 의견 유형 선택 + 내용 입력 → 제출
   ↓
[CP팀 관리자]
4. 팀 의견 검토 (수용 / 기각)
5. 리스크 Master 반영 (추가·수정·삭제)
6. 전체 의견 처리 완료 → "Phase 1 완료 확정" 버튼 클릭
```

---

## 🔴 BUG — 즉시 수정 필요

### BUG-1. `canConfirm` 조건 오류 — 미제출 건 있어도 확정 버튼 활성화됨

**위치:** `renderIdentifyConfirmView()` (Line 7537)

```js
// 현재 (잘못됨)
const canConfirm = pending.length === 0 && allSubmitted.length > 0;
```

- `notSubmitted`(미제출 건 수)는 이미 계산되어 있지만 `canConfirm` 조건에 포함되지 않음.
- 팀 담당자 일부가 아직 의견을 제출하지 않은 상태에서도 "Phase 1 완료 확정" 버튼이 활성화됨.
- 결과: 전수 확인이 완료되지 않은 상태로 Phase 1이 확정될 수 있는 치명적 오류.

**수정 방향:**

```js
// 수정 후
const canConfirm = pending.length === 0 && allSubmitted.length > 0 && notSubmitted === 0;
```

하단 "전체 의견 처리 현황" 카드의 비활성 버튼 설명 문구도 수정:

```js
// 수정 전
: `<button ... disabled>미처리 ${pending.length}건 남음</button>`

// 수정 후 (복합 조건 문구)
: `<button ... disabled>${
    notSubmitted > 0
      ? `미제출 ${notSubmitted}건 남음`
      : `미처리 ${pending.length}건 남음`
  }</button>`
```

---

### BUG-2. 메일 발송 버튼 — OFFLINE_MODE=false, URL 미설정 시 아무 동작 없음 (빈칸 버그)

**위치:** `sendAssignNotify()` (Lines 6700~6709)

```js
// 현재
if (CONFIG.OFFLINE_MODE) {
  // 오프라인 처리
} else if (CONFIG.FLOW_ASSIGN_NOTIFY_URL) {
  // Flow 호출
}
// else: 아무 동작 없음 — 확인 모달만 뜨고 조용히 종료
```

- `OFFLINE_MODE = false`, `FLOW_ASSIGN_NOTIFY_URL = ''` (미입력) 상태에서 발송 버튼을 누르면 확인 모달까지는 뜨지만, 이후 아무 처리 없이 종료됨.
- 사용자는 메일이 발송된 것으로 착각할 수 있음.
- 배당 확인 메일 발송 버튼이 **실질적으로 동작하지 않는** 상태.

**수정 방향:**

```js
} else {
  toast('FLOW_ASSIGN_NOTIFY_URL이 설정되지 않았습니다. CONFIG를 확인하세요.', 'error', 5000);
  return;
}
```

---

### BUG-3. 의견 제출 완료 후 의견 유형 칩(chip)이 여전히 클릭 가능

**위치:** `selectIdentifyOpinion()` (Line 7478)

```js
// 현재 — textarea는 disabled 처리되어 있지만
<textarea ... ${isSubmitted?'disabled':''}> ...

// 의견 유형 칩 버튼에는 disabled 처리 없음
${['이의없음','추가요청','수정요청','삭제요청'].map(t =>
  `<button ... onclick="selectIOType('${t}',this)"> ...`
)}
```

- 이미 "제출완료"된 리스크를 다시 선택하면 textarea는 비활성화되지만, 의견 유형 칩 버튼은 활성 상태 유지.
- `selectIOType()`으로 `window._ioType` 값이 바뀌면, 다음에 선택하는 다른 리스크의 제출 시 잘못된 타입이 사용될 위험.

**수정 방향:**

```js
`<button ...
  ${isSubmitted ? 'disabled style="opacity:.5;cursor:not-allowed;pointer-events:none"' : ''}
  onclick="selectIOType('${t}',this)"> ...`
```

---

### BUG-4. `FLOW_IDENTIFY_OPINION_URL` 선언만 있고 실제 호출이 없음

**위치:** `saveIdentifyOpinion()` (Lines 7495~7507), `CONFIG` 선언 (Line 5341)

```js
// CONFIG에 선언되어 있음
CONFIG.FLOW_IDENTIFY_OPINION_URL = '';  // Phase 1: 팀 의견 제출 알림

// 그러나 saveIdentifyOpinion() 함수에는 callFlow 호출 없음
function saveIdentifyOpinion(riskId, isDraft) {
  ...
  toast(isDraft ? '임시저장 완료' : `[${riskId}] 의견이 제출되었습니다.`, ...);
  renderIdentifyOpinionView();
  // callFlow(CONFIG.FLOW_IDENTIFY_OPINION_URL, ...) 누락
}
```

- 팀 담당자가 의견을 "제출완료"해도 CP팀 관리자에게 알림 흐름이 없음.
- 배당 확인 메일 발송 후 CP팀이 능동적으로 의견 제출 현황을 확인해야만 알 수 있는 구조.

**수정 방향:** `isDraft === false` (정식 제출) 시점에 Flow 호출 추가

```js
if (!isDraft && (CONFIG.FLOW_IDENTIFY_OPINION_URL || CONFIG.OFFLINE_MODE)) {
  if (!CONFIG.OFFLINE_MODE && CONFIG.FLOW_IDENTIFY_OPINION_URL) {
    callFlow(CONFIG.FLOW_IDENTIFY_OPINION_URL, {
      year: parseInt(curYear),
      riskId: r.id,
      team: r.team,
      opinionType: r.identifyOpinionType,
    }).catch(e => console.warn('의견 제출 알림 발송 실패:', e));
  }
}
```

---

### BUG-5. CP팀이 수용/기각 오처리 후 재처리 방법 없음

**위치:** `selectIdentifyConfirm()` (Lines 7598~7648)

```js
// done = true이면 버튼 영역 자체가 렌더링되지 않음
${!done ? `
  <div style="display:flex;gap:10px;margin-top:16px;justify-content:flex-end">
    <button ... onclick="confirmIdentifyOpinion('${r.id}','rejected')">기각</button>
    <button ... onclick="confirmIdentifyOpinion('${r.id}','accepted')">수용</button>
  </div>
` : ''}
```

- 수용·기각 처리 후에는 "처리 완료" 뱃지만 표시되고 변경 수단이 없음.
- 잘못 기각한 건을 수용으로 바꾸거나, 반대 방향 수정이 불가능.

**수정 방향:** 처리완료 상태에도 "재처리" 버튼 노출 (관리자 권한 기준)

```js
${done ? `
  <div style="display:flex;justify-content:flex-end;margin-top:12px">
    <button class="btn btn-ghost" style="font-size:12px"
      onclick="reopenIdentifyConfirm('${r.id}')">↩ 처리 취소 (재검토)</button>
  </div>
` : ''}
```

`reopenIdentifyConfirm` 함수를 추가해 `identifyConfirmStatus`를 `'pending'`으로 재설정.

---

### BUG-6. CP팀 검토 화면에서 "이의없음" 제출 건 조회 불가

**위치:** `renderIdentifyConfirmView()` (Lines 7512~7556)

```js
const withOpinion = allSubmitted.filter(r =>
  r.identifyOpinionType && r.identifyOpinionType !== '이의없음'
);
// 좌측 목록: withOpinion 기준으로만 렌더링
```

- 의견없음(이의없음)으로 제출된 건들은 CP팀 화면에서 아예 보이지 않음.
- CP팀이 "이의없음" 제출 건의 내용을 확인하거나, 팀별 전체 제출 현황을 파악할 수단이 없음.
- 예: 특정 팀이 이의없음으로 다수 제출했는데 CP팀이 내용을 검토할 수 없음.

**수정 방향:** 좌측 목록 상단에 탭 또는 필터 토글 추가

```
[전체 제출 (N건)] [의견있음 (N건)] [이의없음 (N건)]
```

탭 선택에 따라 목록을 필터링하여 렌더링.

---

## 🟡 IMPROVE — 기능 개선 권고

### IMPROVE-1. 수용 처리 후 리스크 Master 이동 버튼 없음

**위치:** `confirmIdentifyOpinion()` / `selectIdentifyConfirm()` (Lines 7618~7641)

현재 코드:
```js
if (r.identifyOpinionType === '추가요청') {
  toast(`[${riskId}] 추가 요청을 수용했습니다. 리스크 Master에서 직접 추가해주세요.`, 'info', 5000);
}
```

- "리스크 Master에서 직접 해주세요"라고 toast만 띄우지만, Master로 이동하는 버튼이 없음.
- 관리자가 toast를 놓치거나, 어디서 작업해야 하는지 혼란 발생.

**수정 방향:** 수용 처리 완료 후 폼 하단에 바로 이동 버튼 표시

```js
if (status === 'accepted') {
  const actionMap = {
    추가요청: '<button onclick="navigateTo(\'master\')" class="btn btn-ghost" style="margin-top:10px">➕ 리스크 Master에서 추가하기</button>',
    수정요청: `<button onclick="navigateTo('master')" class="btn btn-ghost" style="margin-top:10px">✏ 리스크 Master에서 수정하기</button>`,
    삭제요청: `<button onclick="navigateTo('master')" class="btn btn-ghost" style="margin-top:10px;border-color:var(--red);color:var(--red)">🗑 리스크 Master에서 삭제 확인</button>`,
  };
}
```

---

### IMPROVE-2. 의견 제출 시 알림 발송 미구현 (BUG-4 연계 개선)

현재 `CONFIG.FLOW_IDENTIFY_OPINION_URL`이 설정되어 있어도 활용되지 않음.  
BUG-4 수정과 함께, UI 상 "제출 시 CP팀에 알림이 발송됩니다" 문구를 의견 제출 안내 박스에 추가 권고.

---

### IMPROVE-3. 팀별 미제출 현황에서 독촉 메일 발송 버튼 없음 (Phase 1 한정)

`renderIdentifyConfirmView()`의 팀별 현황 테이블에서 "미제출" 컬럼에 숫자만 표시됨.  
Phase 2~3에서는 `sendNudge()` 버튼이 있는데, Phase 1 현업 의견 미제출 팀에 대한 독촉 기능이 없음.

**수정 방향:** 팀별 현황 테이블의 "미제출" 셀에 독촉 버튼 추가

```js
<td>
  ${d.notSubmit > 0
    ? `<span style="color:var(--red)">${d.notSubmit}</span>
       <button onclick="sendNudge('${t}',this)" style="...">독촉</button>`
    : `<span style="color:var(--green)">✓</span>`}
</td>
```

---

### IMPROVE-4. Phase 1 단계별 미션(phasecontrol) 완료 조건에 "전체 제출 완료" 항목 없음

**위치:** `_checkPhaseConditions('identify')` (Lines 5851~5861)

현재 완료 조건:
1. 리스크 1건 이상 등록
2. 배당 확인 메일 발송 ← manual 체크
3. 현업의견 전수 검토 처리 완료

"현업 전원 의견 제출 완료" 조건이 없음.  
일부 팀이 미제출 상태여도 나머지 조건이 충족되면 완료로 표시될 수 있음 (BUG-1과 연동).

**수정 방향:** 조건 4번 추가

```js
const allTeamsSubmitted = RISKS.length > 0 &&
  RISKS.every(r => r.identifyOpinionStatus === '제출완료');
return [
  ...기존 조건들,
  {
    label: `현업 의견 전수 제출 완료 (${allSubmitted.length}/${RISKS.length}건)`,
    met: allTeamsSubmitted,
    detail: allTeamsSubmitted
      ? '전수 제출 완료'
      : `${RISKS.length - allSubmitted.length}건 미제출`,
  },
];
```

---

## 수정 우선순위 요약

| 순위 | 구분 | 항목 | 영향 |
|------|------|------|------|
| P0 | BUG | BUG-1: canConfirm 조건 누락 (notSubmitted 미검증) | Phase 1 데이터 무결성 파괴 |
| P0 | BUG | BUG-2: 메일 발송 버튼 빈칸 (URL 없을 때 아무 동작 없음) | 사용자 혼란, 미발송 착오 |
| P1 | BUG | BUG-3: 제출완료 건 의견유형 칩 disabled 미처리 | 잘못된 타입 overwrite 위험 |
| P1 | BUG | BUG-4: FLOW_IDENTIFY_OPINION_URL 호출 누락 | 실시간 알림 불동작 |
| P1 | BUG | BUG-5: 처리완료 건 재처리 수단 없음 | 오처리 시 수정 불가 |
| P2 | BUG | BUG-6: "이의없음" 건 CP팀 화면 조회 불가 | 전수 검토 불가 |
| P2 | IMPROVE | IMPROVE-1: 수용 후 Master 이동 버튼 없음 | 흐름 단절 |
| P3 | IMPROVE | IMPROVE-3: Phase 1 독촉 메일 없음 | 편의 기능 |
| P3 | IMPROVE | IMPROVE-4: phasecontrol 완료 조건에 전수 제출 항목 누락 | 완료 기준 불완전 |

---

## 파일 수정 체크리스트

- [ ] `renderIdentifyConfirmView()` — canConfirm 조건에 `&& notSubmitted === 0` 추가
- [ ] `renderIdentifyConfirmView()` — 비활성 버튼 문구 복합 조건으로 수정
- [ ] `sendAssignNotify()` — else 분기 추가 (URL 미설정 경고 toast)
- [ ] `selectIdentifyOpinion()` — 의견유형 칩에 `isSubmitted` disabled 처리 추가
- [ ] `saveIdentifyOpinion()` — `!isDraft` 시 `FLOW_IDENTIFY_OPINION_URL` callFlow 추가
- [ ] `selectIdentifyConfirm()` — done=true 시 "재처리" 버튼 추가
- [ ] `reopenIdentifyConfirm(riskId)` 함수 신규 추가
- [ ] `renderIdentifyConfirmView()` — 좌측 목록에 탭(전체/의견있음/이의없음) 추가
- [ ] `confirmIdentifyOpinion()` — 수용 처리 후 Master 이동 버튼 렌더링 추가
- [ ] `_checkPhaseConditions('identify')` — 전수 제출 완료 조건 항목 추가
- [ ] 팀별 현황 테이블 "미제출" 셀에 독촉 버튼 추가 (선택)
