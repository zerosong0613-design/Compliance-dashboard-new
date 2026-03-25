# 기능 업데이트 요청 — update.md

> 작성일: 2026-03-25  
> 대상 파일: `index.html` (Compliance RiskOps)

---

## UPDATE-1. 이행현황 입력/검토 — Phase 5 확정 개선방안이 있는 리스크만 표시

### 현재 동작 (문제)

`renderMonthly()` Line 3038:
```js
const base = appRole === 'admin'
  ? RISKS
  : RISKS.filter(r => r.team === appTeam);
```

현재는 **전체 RISKS** 또는 **내 팀 전체 리스크**를 목록에 표시함.  
Phase 5에서 개선방안이 확정되지 않은 리스크까지 이행입력 목록에 나타나는 문제 발생.

이행 대상은 Phase 5에서 `improvePlanStatus === '확정'`이 된 리스크에만 해당됨.  
(`_getPhase6Targets()` 함수가 이미 이 조건으로 정의되어 있음 — Line 8173)

```js
// 기존 정의 (활용 안 됨)
function _getPhase6Targets() {
  return RISKS.filter(r => r.z_selected && r.improvePlanStatus === '확정');
}
```

### 수정 방향

**`renderMonthly()` Line 3038 `base` 정의 수정**

```js
// 수정 전
const base = appRole === 'admin'
  ? RISKS
  : RISKS.filter(r => r.team === appTeam);

// 수정 후
const phase5Done = _loadPhaseStatus(curYear)?.improve?.completed;
const phase6Targets = RISKS.filter(r => r.z_selected && r.improvePlanStatus === '확정');

const base = appRole === 'admin'
  ? (phase5Done ? phase6Targets : RISKS)  // 관리자: Phase 5 완료 시 확정 건만
  : phase6Targets.filter(r => r.team === appTeam); // 현업: 내 팀 확정 건만
```

**빈 목록 안내 문구 추가** (base.length === 0일 때)

```js
// monthly-list가 비었을 때 표시
if (!base.length) {
  document.getElementById('monthly-list').innerHTML = `
    <div style="text-align:center;padding:48px 20px;color:var(--muted)">
      <div style="font-size:36px;margin-bottom:12px">📋</div>
      <div style="font-size:14px;font-weight:700;margin-bottom:6px">이행 대상 리스크 없음</div>
      <div style="font-size:12px;line-height:1.7">
        Phase 5(개선방안 확정)가 완료된 후<br>이행 대상 리스크가 이 목록에 표시됩니다.
      </div>
    </div>`;
  return;
}
```

**가이드 배너 문구 수정** (user 모드, Line 3064)

```js
// 수정 전
`${curYear}년 ${curMonth}월 · ${appTeam} · 제출 ${submitted}/${total}건`

// 수정 후
`${curYear}년 ${curMonth}월 · ${appTeam} · 확정 개선방안 ${total}건 이행 중 · 제출 ${submitted}/${total}건`
```

---

## UPDATE-2. Phase 4 잔여리스크 확정 — 확정된 관리대상 리스크 목록 강조 및 클릭 조회

### 현재 동작 (문제)

확정 완료 시 하단에 아래 한 줄만 표시됨 (Line 7961):
```js
`✅ Phase 4 완료 · 관리 대상 ${zSelected}건 확정됨`
```

- 숫자만 보이고 **어떤 리스크가 확정됐는지** 목록을 볼 수 없음
- 확정 후 테이블의 체크박스가 `disabled` 처리되어 시각적으로 구분은 되지만 강조가 부족
- 관리대상 리스크(H·VH)와 비관리대상이 같은 테이블에 섞여 보임

### 수정 방향 A — 확정 완료 시 "관리대상 리스크" 요약 카드 노출

`confirmResidualFinal()` 실행 후 `renderResidualViewNew()` 안에서, `isConfirmed === true`일 때 하단 확정 완료 섹션을 아래처럼 교체:

```js
// 수정 전 (Line 7960~7963)
`<div style="display:flex;align-items:center;gap:10px">
  <span ...>✅ Phase 4 완료 · 관리 대상 ${zSelected}건 확정됨</span>
  <button ... onclick="reopenResidual()">수정 (재오픈)</button>
</div>`

// 수정 후
`<div>
  <div style="display:flex;align-items:center;gap:10px;margin-bottom:12px">
    <span style="...green">✅ Phase 4 완료 · 관리 대상 ${zSelected}건 확정됨</span>
    <button ... onclick="reopenResidual()">수정 (재오픈)</button>
    <button class="btn btn-ghost" style="font-size:12px"
      onclick="toggleResidualSummary()">📋 확정 목록 보기 ▾</button>
  </div>
  <div id="residual-confirmed-summary" style="display:none">
    <!-- 관리대상 리스크 카드 목록 렌더링 -->
  </div>
</div>`
```

**`toggleResidualSummary()` 함수 신규 추가**

```js
function toggleResidualSummary() {
  const el = document.getElementById('residual-confirmed-summary');
  if (!el) return;
  const isOpen = el.style.display !== 'none';
  if (isOpen) {
    el.style.display = 'none';
    return;
  }
  const targets = RISKS.filter(r => r.z_selected);
  el.innerHTML = `
    <div style="margin-top:12px;padding:16px;background:#FEF9F9;border:1px solid #FECACA;border-radius:12px">
      <div style="font-size:13px;font-weight:800;color:var(--red);margin-bottom:10px">
        🎯 관리대상 리스크 ${targets.length}건
      </div>
      ${targets.map(r => `
        <div style="display:flex;align-items:center;gap:8px;padding:8px 10px;
          background:#fff;border:1px solid var(--border);border-radius:8px;margin-bottom:6px">
          <span style="font-size:11px;color:var(--accent);font-weight:700;flex-shrink:0">${r.id}</span>
          ${levelBadgeHtml(r.z_level)}
          <span style="font-size:12px;font-weight:600;color:var(--text);flex:1;
            white-space:nowrap;overflow:hidden;text-overflow:ellipsis">
            ${_esc(r.riskContent||r.title)}
          </span>
          <span style="font-size:11px;color:var(--muted);flex-shrink:0">${_esc(r.team)}</span>
        </div>`).join('')}
    </div>`;
  el.style.display = 'block';
}
```

### 수정 방향 B — 테이블에서 관리대상 행 시각적 강조 강화

현재 체크된 행의 배경색만 다를 뿐 구분이 약함. 확정 이후(`isConfirmed === true`)에는:

```js
// 수정 전 (Line 7940~7941)
const rowBg = r.z_level==='VH' ? 'background:#FEF9F9'
            : r.z_level==='H'  ? 'background:#FFFCF5' : '';

// 수정 후 — z_selected 여부 추가
const rowBg = r.z_selected
  ? (r.z_level==='VH' ? 'background:#FEE2E2' : 'background:#FEF3C7')
  : 'background:#F8FAFC;opacity:.55';  // 비선정 행은 흐리게
```

또한 `선정` 컬럼 헤더를 확정 후에는 `관리대상`으로 교체:

```js
// 수정 전 헤더
<th style="width:40px;text-align:center">선정</th>

// 수정 후 (isConfirmed 여부로 분기)
<th style="width:40px;text-align:center">${isConfirmed ? '관리대상' : '선정'}</th>
```

---

## UPDATE-3. 사이드바 "위험평가 작업" 섹션 — Phase 5 완료 후 접기 가능

### 현재 동작 (문제)

`NAV_ADMIN` (Line 1912~1917):
```js
{sec:'위험평가 작업'},
{id:'master',           icon:'📋', label:'① 리스크 등록·관리'},
{id:'identify-confirm', icon:'✅', label:'① 식별 의견 검토·확정'},
{id:'control-eval',     icon:'🛡', label:'③ 통제효과성 평가'},
{id:'residual',         icon:'🎯', label:'④ 잔여리스크 확정'},
{id:'improve-confirm',  icon:'🔧', label:'⑤ 개선방안 확정'},
```

`buildNav()` (Line 2039):
```js
if (n.sec) return `<div class="sb-section">${n.sec}</div>`;
```

현재 `.sb-section`은 단순 텍스트 레이블이라 클릭/접기 기능이 없음.

### 수정 방향

**① `NAV_ADMIN`에 접기 제어용 속성 추가**

```js
// 수정 후 — collapsible 속성 부여
{sec:'위험평가 작업', collapsible:true, secId:'eval-work'},
```

**② `buildNav()` 내 `sec` 렌더링 분기 추가**

```js
// 수정 전
if (n.sec) return `<div class="sb-section">${n.sec}</div>`;

// 수정 후
if (n.sec) {
  if (n.collapsible) {
    const ps = _loadPhaseStatus(curYear);
    const phase5Done = !!ps?.improve?.completed;
    const isCollapsed = phase5Done
      ? (localStorage.getItem(`sb_sec_${n.secId}`) !== 'open')   // Phase 5 완료 → 기본 접힘
      : (localStorage.getItem(`sb_sec_${n.secId}`) === 'closed'); // Phase 5 미완료 → 기본 펼침
    return `
      <div class="sb-section sb-section-toggle"
        onclick="toggleSbSection('${n.secId}', this)"
        style="cursor:pointer;display:flex;align-items:center;justify-content:space-between;padding-right:12px">
        <span>${n.sec}</span>
        <span id="sb-sec-arrow-${n.secId}" style="font-size:9px;color:#4A6280;transition:transform .2s;
          ${isCollapsed ? '' : 'transform:rotate(90deg)'}">▶</span>
      </div>
      <div id="sb-sec-body-${n.secId}" style="overflow:hidden;transition:max-height .25s ease;
        max-height:${isCollapsed ? '0' : '400px'}">`;
    // 닫는 </div>는 이 섹션 마지막 항목 뒤에 삽입 필요 (아래 ③ 참조)
  }
  return `<div class="sb-section">${n.sec}</div>`;
}
```

**③ 해당 섹션 마지막 항목 뒤 닫기 div 삽입**

`NAV_ADMIN`에서 `improve-confirm` 다음에 섹션 종료 마커 추가:

```js
{id:'improve-confirm', icon:'🔧', label:'⑤ 개선방안 확정'},
{secEnd:'eval-work'},   // ← 새 마커
{sec:'이행 관리'},
...
```

`buildNav()` 내 처리:
```js
if (n.secEnd) return `</div>`; // sb-sec-body 닫기
```

**④ `toggleSbSection()` 함수 신규 추가**

```js
function toggleSbSection(secId, headerEl) {
  const body = document.getElementById(`sb-sec-body-${secId}`);
  const arrow = document.getElementById(`sb-sec-arrow-${secId}`);
  if (!body) return;
  const isNowOpen = body.style.maxHeight !== '0px' && body.style.maxHeight !== '0';
  if (isNowOpen) {
    body.style.maxHeight = '0';
    if (arrow) arrow.style.transform = '';
    localStorage.setItem(`sb_sec_${secId}`, 'closed');
  } else {
    body.style.maxHeight = '400px';
    if (arrow) arrow.style.transform = 'rotate(90deg)';
    localStorage.setItem(`sb_sec_${secId}`, 'open');
  }
}
```

**⑤ Phase 5 완료 확정 시 자동 접기 트리거 추가**

`completePhase5()` 함수 (Line 8164~8171) 내 toast 이후에:
```js
// 기존
toast(`✅ Phase 5 완료! ...`, 'success', 5000);
renderImproveConfirmView();

// 추가
setTimeout(() => {
  const body = document.getElementById('sb-sec-body-eval-work');
  const arrow = document.getElementById('sb-sec-arrow-eval-work');
  if (body) { body.style.maxHeight = '0'; }
  if (arrow) { arrow.style.transform = ''; }
  localStorage.setItem('sb_sec_eval-work', 'closed');
  buildNav(); // 사이드바 재구성으로 상태 반영
}, 500);
```

**드로어(모바일)에도 동일 패턴 적용**

`buildNav()` 내 드로어 렌더링(`dr-section`)도 동일한 분기 처리 추가 (Line 2062):
```js
if (n.sec && n.collapsible) { /* 동일 패턴, dr-sec-body-{id} ID 사용 */ }
```

---

## 수정 우선순위 및 체크리스트

| 우선순위 | 번호 | 수정 위치 | 작업 내용 |
|---------|------|----------|----------|
| P0 | UPDATE-1 | `renderMonthly()` Line 3038 | `base` 필터에 `improvePlanStatus==='확정'` 조건 추가 |
| P0 | UPDATE-1 | `renderMonthly()` | Phase 5 미완료 시 빈 목록 안내 문구 추가 |
| P1 | UPDATE-2-A | `renderResidualViewNew()` Line 7960 | 확정 완료 영역에 "확정 목록 보기" 버튼 추가 |
| P1 | UPDATE-2-A | 신규 함수 | `toggleResidualSummary()` 추가 |
| P1 | UPDATE-2-B | `renderResidualViewNew()` Line 7940 | 비선정 행 흐리게, 관리대상 행 강조 |
| P2 | UPDATE-3 | `NAV_ADMIN` Line 1912 | `collapsible:true, secId:'eval-work'` 속성 추가 |
| P2 | UPDATE-3 | `buildNav()` Line 2039 | `sec` 렌더링에 접기 분기 추가 |
| P2 | UPDATE-3 | 신규 함수 | `toggleSbSection(secId)` 추가 |
| P2 | UPDATE-3 | `completePhase5()` Line 8169 | Phase 5 완료 시 섹션 자동 접기 트리거 추가 |
| P2 | UPDATE-3 | `buildNav()` Line 2062 | 드로어(모바일)에도 동일 접기 패턴 적용 |
