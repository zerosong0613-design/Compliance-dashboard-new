# Phase 1 오류 분석 및 수정 지침 — error.md

> 분석 대상: `index.html` (Compliance RiskOps)  
> 작성일: 2026-03-25  
> 핵심 증상: **현업 팀 담당자가 "리스크 식별 의견 제출" 버튼을 눌러도 제출이 안 됨**

---

## 🔴 ROOT CAUSE — 제출 버튼이 안 눌리는 직접 원인 3가지

### [RC-1] `form-panel` 내 스크롤 컨테이너 누락 → 버튼이 `overflow:hidden`으로 잘림 ★★★

**근본 원인**

```css
/* CSS — form-panel */
.form-panel {
  flex: 1;
  overflow: hidden;       /* ← 범인 */
  display: flex;
  flex-direction: column;
}
```

`form-panel`이 `overflow:hidden`이면 내부 콘텐츠가 패널 높이를 초과할 때 **잘려서 보이지 않음**.

다른 정상 작동하는 폼(월간 입력 `monthly`, 통제수단 입력 `control-input`)은 반드시 아래 구조를 씀:

```html
<!-- 정상 패턴 (monthly, eval 등) -->
<div class="form-hd">...</div>             <!-- 헤더, flex-shrink:0 -->
<div class="form-body">...</div>           <!-- overflow-y:auto; flex:1 ← 스크롤 가능 -->
<div class="form-footer">...</div>         <!-- 버튼, flex-shrink:0 -->
```

그런데 `selectIdentifyOpinion()`은:

```js
// 현재 코드 (잘못됨)
form.innerHTML = `
  <div style="padding:20px">       ← form-body 없음, 스크롤 없음
    ...내용...
    <button onclick="saveIdentifyOpinion(...)">제출하기</button>   ← 화면 아래로 잘림
  </div>`;
```

**높이 계산으로 확인**:

| 화면 높이 | split-view 위 헤더+안내박스 | 가용 form-panel 높이 | 폼 콘텐츠 높이 | 결과 |
|---|---|---|---|---|
| 900px | ~155px | ~615px | ~380px | ✅ 보임 |
| 768px | ~155px | ~483px | ~380px | ✅ 겨우 보임 |
| 700px | ~155px | ~415px | ~380px | ⚠ 버튼 절반 잘림 |
| 600px | ~155px | ~315px | ~380px | ❌ 버튼 완전 숨김 |

→ **화면이 작거나 브라우저 줌이 높으면 제출 버튼이 invisible/unclickable**

**수정 방법**: `selectIdentifyOpinion()` 및 `selectIdentifyConfirm()` 내부를 `form-hd` / `form-body` / `form-footer` 구조로 교체

```js
// 수정 후 — selectIdentifyOpinion
form.innerHTML = `
  <div class="form-hd">
    <div class="form-hd-id">${r.id} · ${_esc(r.domain||r.cat)}</div>
    <div class="form-hd-title">${_esc(r.riskContent||r.title)}</div>
    <div style="font-size:12px;color:var(--muted);margin-top:4px">법규명: ${_esc(r.lawName||'-')} · 조항: ${_esc(r.relatedClause||'-')}</div>
  </div>
  <div class="form-body">
    <!-- 의견 유형 칩 + textarea 영역 -->
    <div style="font-size:12px;font-weight:700;color:var(--muted);margin-bottom:6px">의견 유형</div>
    <div id="io-type-chips" ...>...칩 버튼들...</div>
    <div style="font-size:12px;font-weight:700;...">의견 내용</div>
    <textarea id="io-content" ...></textarea>
    ${...CP팀 처리 결과 박스...}
  </div>
  <div class="form-footer">
    <button class="btn btn-ghost" onclick="saveIdentifyOpinion('${r.id}',true)">임시저장</button>
    <button class="btn btn-primary" onclick="saveIdentifyOpinion('${r.id}',false)" ${isSubmitted?'disabled':''}>제출하기</button>
  </div>`;
```

동일 패턴을 `selectIdentifyConfirm()`에도 적용.

---

### [RC-2] 모바일에서 `openSheet` 미사용 → 폼이 리스트 아래로 밀려 버튼 못 찾음 ★★★

**근본 원인**

모바일(≤768px)에서는 `split-view`가 `display:block`으로 바뀜. 다른 정상 폼들은 모바일 시 Bottom Sheet를 열어 처리함:

```js
// 정상 패턴 (monthly, residual 등)
if (isMobile()) {
  openSheet(r.id, r.title, body, footer);  // ← Bottom Sheet 팝업
} else {
  form.innerHTML = `<div class="form-hd">...</div>...`;
}
```

그런데 `selectIdentifyOpinion()`은:

```js
// 현재 코드 (잘못됨)
function selectIdentifyOpinion(idx) {
  // isMobile() 분기 없음
  // openSheet 호출 없음
  const form = document.getElementById('identify-opinion-form');
  form.innerHTML = `<div style="padding:20px">...</div>`;  // 리스트 아래에 렌더링
}
```

→ 모바일에서 리스크를 클릭해도 폼은 **리스트 아래 화면 밖에** 렌더링됨. 사용자는 "안 되네"라고 인식.

**수정 방법**: 모바일 분기 추가

```js
function selectIdentifyOpinion(idx) {
  const myRisks = RISKS.filter(r => (r.relatedOrg||r.team||'').includes(appTeam));
  const r = myRisks[idx]; if (!r) return;
  
  const bodyHtml = `...의견 유형 칩 + textarea...`;
  const footerHtml = `
    <button class="btn btn-ghost" onclick="saveIdentifyOpinion('${r.id}',true)">임시저장</button>
    <button class="btn btn-primary" onclick="saveIdentifyOpinion('${r.id}',false)"
      ${isSubmitted?'disabled':''}>제출하기</button>`;

  if (isMobile()) {
    openSheet(r.id + ' · ' + r.domain, r.riskContent||r.title, bodyHtml, footerHtml);
  } else {
    const form = document.getElementById('identify-opinion-form');
    form.innerHTML = `
      <div class="form-hd">...</div>
      <div class="form-body">${bodyHtml}</div>
      <div class="form-footer">${footerHtml}</div>`;
  }
  window._ioType = r.identifyOpinionType || '';
}
```

동일 패턴을 `selectIdentifyConfirm()`에도 적용.

---

### [RC-3] 모바일 바텀 내비게이션에 `identify-opinion` 진입 경로 없음 ★★

**근본 원인**

```js
// 현재 BNAV_USER (바텀 탭, 모바일)
const BNAV_USER = [
  {id:'monthly',   icon:'📝', label:'입력',   badge:true},
  {id:'evalresult',icon:'📊', label:'평가·이력'},
  {id:'myteam',    icon:'📈', label:'내 팀'},
  {id:'archive',   icon:'📚', label:'아카이브'},
];
// ← 'identify-opinion' 없음! 드로어 메뉴만으로만 접근 가능
```

현업 담당자가 모바일에서 가장 자주 써야 하는 "① 리스크 식별 의견 제출" 화면이 바텀 탭에 없음.

**수정 방법**: `monthly` 대신 또는 추가로 식별 의견 탭 노출

```js
// Phase 1 기간 중 동적 노출 (또는 항상 노출)
const BNAV_USER = [
  {id:'identify-opinion', icon:'💬', label:'의견제출'},
  {id:'monthly',          icon:'📝', label:'이행입력', badge:true},
  {id:'evalresult',       icon:'📊', label:'이력'},
  {id:'myteam',           icon:'📈', label:'내 팀'},
];
```

---

## 🔴 기능 오류 (버튼 비활성·동작 안 함)

### [ERR-1] 배당 확인 메일 발송 버튼 — URL 미설정 시 아무 동작 없음

**위치**: `sendAssignNotify()` (Line 6693~6711)

```js
// 현재 — else 분기 없음
if (CONFIG.OFFLINE_MODE) {
  // 오프라인 처리
} else if (CONFIG.FLOW_ASSIGN_NOTIFY_URL) {
  // Flow 호출
}
// OFFLINE_MODE=false, URL='' → 모달 확인 후 아무 일도 안 일어남
```

→ 운영 환경에서 URL 미입력 시 메일 발송 버튼 클릭이 **무반응**

**수정**:

```js
} else {
  toast('📧 FLOW_ASSIGN_NOTIFY_URL이 설정되지 않았습니다. CONFIG를 확인하세요.', 'error', 6000);
  return;
}
```

---

### [ERR-2] Phase 1 완료 확정 버튼 — 미제출 건 있어도 활성화됨

**위치**: `renderIdentifyConfirmView()` (Line 7537)

```js
// 현재 (잘못됨) — notSubmitted 조건 없음
const canConfirm = pending.length === 0 && allSubmitted.length > 0;

// notSubmitted는 계산되어 있지만 canConfirm에 포함 안 됨
const notSubmitted = RISKS.filter(r => r.identifyOpinionStatus !== '제출완료').length;
```

→ 일부 팀이 의견 미제출 상태여도 Phase 1 완료 확정 가능 → 데이터 무결성 파괴

**수정**:

```js
const canConfirm = pending.length === 0 && allSubmitted.length > 0 && notSubmitted === 0;
```

비활성 버튼 문구도 복합 조건으로 수정:

```js
: `<button class="btn btn-ghost" disabled style="...opacity:.5">
    ${notSubmitted > 0
      ? `⚠ 미제출 ${notSubmitted}건 남음`
      : `미처리 ${pending.length}건 남음`}
  </button>`
```

---

### [ERR-3] 제출완료 건의 의견 유형 칩 — 여전히 클릭 가능 (타입 오염)

**위치**: `selectIdentifyOpinion()` Line 7478

```js
// 현재 — isSubmitted 시 textarea는 disabled지만 칩은 활성
${['이의없음','추가요청','수정요청','삭제요청'].map(t =>
  `<button ... onclick="selectIOType('${t}',this)">${t}</button>`
).join('')}
```

→ 제출완료 건의 칩을 클릭하면 `window._ioType`이 바뀌어, 다음 미제출 건 제출 시 잘못된 type이 전송될 수 있음

**수정**:

```js
`<button ...
  ${isSubmitted ? 'disabled style="opacity:.5;pointer-events:none"' : ''}
  onclick="selectIOType('${t}',this)">${t}</button>`
```

---

### [ERR-4] 의견 제출 완료 시 CP팀 알림 발송 안 됨

**위치**: `saveIdentifyOpinion()` (Line 7495~7507)

```js
// CONFIG에는 선언됨
CONFIG.FLOW_IDENTIFY_OPINION_URL = '';

// 하지만 saveIdentifyOpinion 내 callFlow 호출 없음
function saveIdentifyOpinion(riskId, isDraft) {
  ...
  toast(`[${riskId}] 의견이 제출되었습니다.`);
  // ← 여기서 callFlow 누락
}
```

→ 팀 담당자가 제출해도 CP팀 관리자에게 알림이 가지 않음

**수정**: 정식 제출 시 Flow 호출 추가

```js
if (!isDraft) {
  if (!CONFIG.OFFLINE_MODE && CONFIG.FLOW_IDENTIFY_OPINION_URL) {
    callFlow(CONFIG.FLOW_IDENTIFY_OPINION_URL, {
      year: parseInt(curYear),
      riskId: r.id,
      team: r.team,
      opinionType: r.identifyOpinionType,
    }).catch(e => console.warn('의견 제출 알림 발송 실패:', e));
  }
  toast(`[${riskId}] 의견이 제출되었습니다.`, 'success');
} else {
  toast('임시저장 완료', 'info');
}
```

---

### [ERR-5] 수용·기각 오처리 후 재처리 방법 없음

**위치**: `selectIdentifyConfirm()` (Line 7617)

```js
// 현재 — done=true면 버튼 영역 자체가 없음
${!done ? `
  <button onclick="confirmIdentifyOpinion('${r.id}','rejected')">기각</button>
  <button onclick="confirmIdentifyOpinion('${r.id}','accepted')">수용</button>
` : ''}
```

→ 잘못 수용/기각 처리 시 수정 방법 없음. CP팀 관리자 전용 기능.

**수정**: 처리 완료 건에 "재처리" 버튼 추가 + 함수 신규 작성

```js
// selectIdentifyConfirm 내 done=true 영역에 추가
${done ? `
  <div style="display:flex;justify-content:flex-end;margin-top:12px">
    <button class="btn btn-ghost" style="font-size:12px"
      onclick="reopenIdentifyConfirm('${r.id}')">↩ 처리 취소 (재검토)</button>
  </div>
` : ''}

// 신규 함수 추가
function reopenIdentifyConfirm(riskId) {
  const r = RISKS.find(x => x.id === riskId); if (!r) return;
  r.identifyConfirmStatus = 'pending';
  r.identifyConfirmComment = '';
  if (CONFIG.OFFLINE_MODE) _lsSave(curYear, curMonth, r.id, { ...r });
  toast(`[${riskId}] 처리가 취소되었습니다. 재검토해주세요.`, 'info');
  renderIdentifyConfirmView();
}
```

---

### [ERR-6] CP팀 검토 화면에서 "이의없음" 제출 건 조회 불가

**위치**: `renderIdentifyConfirmView()` (Line 7512~7556)

```js
// 좌측 목록에 의견있는 건만 표시
const withOpinion = allSubmitted.filter(r =>
  r.identifyOpinionType && r.identifyOpinionType !== '이의없음'
);
```

→ "이의없음"으로 제출된 건은 CP팀 화면에서 볼 수 없음. 특정 팀 전체가 이의없음으로 처리했는지 확인 불가.

**수정**: 탭 필터 추가

```js
// 상태 변수 추가
let _identifyConfirmTab = 'opinion'; // 'opinion' | 'noIssue' | 'all'

// 좌측 목록 상단에 탭 삽입
<div style="display:flex;gap:4px;padding:10px 14px;border-bottom:1px solid var(--border)">
  ${['opinion','noIssue','all'].map(tab => {
    const labels = {opinion:`의견있음 (${withOpinion.length})`, noIssue:`이의없음 (${noIssueList.length})`, all:`전체 (${allSubmitted.length})`};
    return `<button onclick="_identifyConfirmTab='${tab}';renderIdentifyConfirmView()"
      style="padding:5px 10px;border-radius:6px;font-size:11px;font-weight:700;border:1px solid ${_identifyConfirmTab===tab?'var(--accent)':'var(--border)'};background:${_identifyConfirmTab===tab?'var(--accent-l)':'#fff'};color:${_identifyConfirmTab===tab?'var(--accent)':'var(--muted)'};cursor:pointer;font-family:inherit">
      ${labels[tab]}</button>`;
  }).join('')}
</div>
```

---

## 수정 우선순위 및 체크리스트

| 우선순위 | 번호 | 항목 | 수정 위치 | 영향 |
|---------|------|------|----------|------|
| **P0** | RC-1 | `form-panel` 내 `form-body`/`form-footer` 구조 적용 | `selectIdentifyOpinion()`, `selectIdentifyConfirm()` | 제출 버튼 안 눌림 직접 원인 |
| **P0** | RC-2 | 모바일 `openSheet` 분기 추가 | `selectIdentifyOpinion()`, `selectIdentifyConfirm()` | 모바일 제출 불가 직접 원인 |
| **P0** | RC-3 | `BNAV_USER`에 `identify-opinion` 추가 | `const BNAV_USER` 선언 (Line 1936) | 모바일 접근 불가 |
| **P1** | ERR-1 | `sendAssignNotify` else 분기 추가 | `sendAssignNotify()` (Line 6700~6709) | 메일 버튼 무반응 |
| **P1** | ERR-2 | `canConfirm`에 `&& notSubmitted === 0` 추가 | `renderIdentifyConfirmView()` (Line 7537) | 무결성 오류 |
| **P1** | ERR-3 | 제출완료 건 의견유형 칩 `disabled` 처리 | `selectIdentifyOpinion()` (Line 7478) | 타입 오염 |
| **P1** | ERR-4 | `saveIdentifyOpinion`에 `callFlow` 추가 | `saveIdentifyOpinion()` (Line 7495~) | 알림 미발송 |
| **P2** | ERR-5 | 재처리 버튼 + `reopenIdentifyConfirm` 함수 추가 | `selectIdentifyConfirm()` (Line 7617~) | 오처리 복구 불가 |
| **P2** | ERR-6 | 이의없음 건 탭 필터 추가 | `renderIdentifyConfirmView()` | 전수 조회 불가 |

---

## 수정 후 검증 시나리오

```
[팀 담당자 로그인 → identify-opinion 진입]
1. 모바일: 바텀 탭에서 "의견제출" 탭 → 리스크 목록 표시 확인
2. 리스크 항목 클릭 → 모바일: Bottom Sheet 팝업 확인 / 데스크탑: 우측 폼 열림 확인
3. 의견 유형 칩 선택 → 스타일 변경 확인
4. 내용 입력 → 임시저장 클릭 → "임시저장 완료" toast 확인
5. 제출하기 클릭 → "제출되었습니다" toast 확인 → 목록 상태 "제출완료"로 변경 확인
6. 제출 완료된 건 재클릭 → 의견유형 칩 비활성화 확인 / 제출 버튼 disabled 확인

[CP팀 관리자 로그인 → identify-confirm 진입]
7. 팀별 현황 테이블에서 미제출 건 수 확인
8. 좌측 목록에서 의견있는 건 클릭 → 우측 검토 폼 열림 확인
9. 수용/기각 클릭 → 처리 완료 상태 확인
10. 처리 완료 건에서 "↩ 처리 취소" 버튼 확인
11. 전체 제출 + 전체 처리 완료 시 → "Phase 1 완료 확정" 버튼 활성화 확인
12. 배당 확인 메일 발송 버튼 → OFFLINE 모드: toast 성공 / URL 없음: 경고 toast 확인
```
