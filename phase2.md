# Phase 2 — 리스크 평가 설계서 및 수정 지시

---

## 1. 업무 흐름

```
Phase 1 완료 (550건 리스크 식별·확정)
    ↓
[상황 A] 작년 엑셀 불러오기로 A/B/X 값이 이미 채워져 있음
    → A·B 평가 탭 열면 기존 값 표시 → 변경 필요한 건만 수정 → 전체 저장 → 확정
    
[상황 B] 리스크 목록만 들어왔고 A/B가 없음
    → A·B 평가 탭에서 전체 입력 → X 자동 계산 → 전체 저장 → 확정
    
[공통] Phase 2 완료 확정 클릭 → Phase 3 활성화
```

**핵심 원칙:**
- A·B는 숫자(1~5)로 저장되지만 **UI에서는 반드시 레이블로 표시** (보통, 높음 등)
- X는 **직접 입력 불가** — A×B 매트릭스에서 자동 결정
- A·B 기준 설명이 **평가 화면 내에 항상 보여야** 한다 (별도 탭/모달 아님)

---

## 2. 스코어링 기준 상수 (코드에 추가)

### A — 발생가능성 레이블 (5단계)

```js
const A_LABELS = {
  5: { short: '확신',  color: '#EF4444', bg: '#FEF2F2' },
  4: { short: '높음',  color: '#F97316', bg: '#FFF7ED' },
  3: { short: '보통',  color: '#F59E0B', bg: '#FFFBEB' },
  2: { short: '낮음',  color: '#84CC16', bg: '#F7FEE7' },
  1: { short: '희박',  color: '#22C55E', bg: '#F0FDF4' },
};

const A_CRITERIA = {
  5: '과거 6개월 내 발생 경험 있고 후속조치 미수행 → 향후 지속 발생 가능성 매우 높음',
  4: '과거 1년 내 발생 경험 있고 후속조치 수행했으나 → 향후에도 발생가능성 높음',
  3: '과거 3년 내 발생 경험 있고 후속조치·주기 모니터링 이루어지나 → 향후 발생 가능성 있음',
  2: '과거 5년 내 발생 경험 있으나 근본 원인 제거·후속조치 수행 → 발생가능성 낮음',
  1: '과거 발생경험 없으며 현재 및 미래 발생가능성 거의 없음',
};
```

### B — 영향의 심각성 레이블 (5단계)

```js
const B_LABELS = {
  5: { short: '매우 심각', color: '#EF4444', bg: '#FEF2F2' },
  4: { short: '상당한 영향', color: '#F97316', bg: '#FFF7ED' },
  3: { short: '보통의 영향', color: '#F59E0B', bg: '#FFFBEB' },
  2: { short: '낮은 영향', color: '#84CC16', bg: '#F7FEE7' },
  1: { short: '미미한 영향', color: '#22C55E', bg: '#F0FDF4' },
};

const B_CRITERIA = {
  5: {
    법적: '대표자 처벌, 100억 초과 과징금',
    재무적: '손해배상·피해복구 등 100억 초과 재무 손실',
    전략적: '보유한 면허 상실, 부정적 언론보도 등 비즈니스에 심각한 영향',
  },
  4: {
    법적: '회사 또는 임직원(대표자 제외) 2년 초과 징역 또는 2억 초과 벌금',
    재무적: '손해배상·피해복구 등 50억 초과 ~ 100억 이하 재무 손실',
    전략적: '비즈니스에 중대한 영향',
  },
  3: {
    법적: '회사 또는 임직원(대표자 제외) 2년 이하 징역 또는 2억 이하 벌금',
    재무적: '손해배상·피해복구 등 10억 초과 ~ 50억 이하 재무 손실',
    전략적: '비즈니스에 상당한 영향',
  },
  2: {
    법적: '과태료의 경우(징역형·벌금 없음), 10억 이하 과징금',
    재무적: '손해배상·피해복구 등 1억 초과 ~ 10억 이하 재무 손실',
    전략적: '비즈니스에 경미한 영향',
  },
  1: {
    법적: '형사처벌, 과징금, 과태료 등 없음',
    재무적: '손해배상·피해복구 등에 의한 1억 이하의 재무적 손실',
    전략적: '비즈니스에 미미한 영향',
  },
};
```

### X — 고유리스크 매트릭스 (5×5, 이미 구현됨)

```js
const X_MATRIX = {
  5: { 1:'M',  2:'H',  3:'VH', 4:'VH', 5:'VH' },
  4: { 1:'M',  2:'H',  3:'H',  4:'VH', 5:'VH' },
  3: { 1:'L',  2:'M',  3:'H',  4:'H',  5:'VH' },
  2: { 1:'VL', 2:'L',  3:'M',  4:'H',  5:'H'  },
  1: { 1:'VL', 2:'VL', 3:'L',  4:'M',  5:'H'  },
};
// xLevelFromMatrix(a, b) 이미 구현됨 — 그대로 사용
```

---

## 3. renderMasterAssessTab() 전면 수정

### 3-1. 현재 문제점

| 항목 | 현재 상태 | 문제 |
|---|---|---|
| A 선택 UI | `<select>` 숫자 1~5 | 무슨 의미인지 모름 |
| B 선택 UI | `<select>` 숫자 1~5 | 무슨 의미인지 모름 |
| 기준 설명 | 없음 | 평가자가 판단 근거 없이 입력 |
| X 표시 | 레벨 배지 | 현재 맞음 |
| 미평가 건 강조 | 상단 노출 정도 | 눈에 안 띔 |
| 저장 방식 | 행별 저장 + 전체 저장 | 현재 맞음 |

### 3-2. 수정 후 화면 구조

```
┌─────────────────────────────────────────────────────────────────┐
│  📊 A·B 평가 (발생가능성 × 영향의 심각성)                         │
│                                                                 │
│  미평가 N건 / 전체 550건   [● 진행률 바]                         │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  ℹ 평가 기준 (접기/펼치기)                               │   │
│  │                                                          │   │
│  │  발생가능성 A:                                           │   │
│  │  5 확신  — 과거 6개월 내 발생, 후속조치 未수행            │   │
│  │  4 높음  — 과거 1년 내 발생, 후속조치 수행               │   │
│  │  3 보통  — 과거 3년 내 발생, 주기 모니터링 중             │   │
│  │  2 낮음  — 과거 5년 내, 근본원인 제거·후속조치 완료       │   │
│  │  1 희박  — 발생 경험 없음                                │   │
│  │                                                          │   │
│  │  영향의 심각성 B:                                         │   │
│  │  5 매우 심각  — 대표자 처벌 / 100억↑ 손실 / 면허 상실    │   │
│  │  4 상당한 영향 — 2년↑ 징역 / 50~100억 / 중대한 영향      │   │
│  │  3 보통의 영향 — 2억↓ 벌금 / 10~50억 / 상당한 영향       │   │
│  │  2 낮은 영향  — 과태료 / 1~10억 / 경미한 영향             │   │
│  │  1 미미한 영향 — 없음 / 1억↓ / 미미한 영향               │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                 │
│  [필터: 전체 | 미평가만 | VH | H | M | L·VL]                    │
│                                                                 │
│  순위 | 영역 | 법규명 | 리스크 내용 | 조직                       │
│  A 발생가능성 | B 영향심각성 | X 고유리스크                        │
│  ─────────────────────────────────────────────────────────     │
│   1  │ HR │ 근로기준법 │ 부당 해고... │ PD팀                     │
│      │    │           │              │                          │
│      A: [5 확신 ▼]   B: [4 상당한 영향 ▼]   X: [VH]             │
│  ─────────────────────────────────────────────────────────     │
│                                                           [저장] │
│                                                                 │
│              [ ✅ 전체 저장 (550건) ]                            │
└─────────────────────────────────────────────────────────────────┘
```

### 3-3. 수정된 renderMasterAssessTab() 코드

```js
function renderMasterAssessTab() {
  const el = document.getElementById('master-assess-content');
  if (!el) return;

  const sorted = [...RISKS].sort((a, b) => {
    const aHas = a.x_level ? 1 : 0, bHas = b.x_level ? 1 : 0;
    if (aHas !== bHas) return aHas - bHas;   // 미평가 먼저
    return (LEVEL_ORDER[b.x_level] || 0) - (LEVEL_ORDER[a.x_level] || 0); // 고위험 먼저
  });

  const assessed   = RISKS.filter(r => r.a_likelihood && r.b_impact).length;
  const pct        = RISKS.length ? Math.round(assessed / RISKS.length * 100) : 0;
  const unassessed = RISKS.length - assessed;

  // ── A 셀렉트 옵션 (레이블 포함) ──
  function aSelect(r) {
    const opts = [['', '-'],...Object.entries(A_LABELS).map(([v,l]) => [v, `${v} ${l.short}`])];
    return `<select class="fm-sel" style="min-width:110px"
      onchange="updateRiskAB('${r.id}','a',parseInt(this.value))">
      ${opts.map(([v, t]) => `<option value="${v}"${r.a_likelihood==v?' selected':''}>${t}</option>`).join('')}
    </select>`;
  }

  // ── B 셀렉트 옵션 (레이블 포함) ──
  function bSelect(r) {
    const opts = [['', '-'],...Object.entries(B_LABELS).map(([v,l]) => [v, `${v} ${l.short}`])];
    return `<select class="fm-sel" style="min-width:130px"
      onchange="updateRiskAB('${r.id}','b',parseInt(this.value))">
      ${opts.map(([v, t]) => `<option value="${v}"${r.b_impact==v?' selected':''}>${t}</option>`).join('')}
    </select>`;
  }

  // ── 평가 기준 접이식 패널 ──
  const criteriaHtml = `
    <details id="assess-criteria-panel" open style="margin-bottom:16px">
      <summary style="cursor:pointer;font-size:13px;font-weight:700;color:var(--accent);
        padding:10px 16px;background:var(--accent-l);border-radius:10px;list-style:none;
        display:flex;align-items:center;gap:8px">
        ℹ 평가 기준 보기·숨기기
      </summary>
      <div style="display:grid;grid-template-columns:1fr 1fr;gap:16px;padding:16px;
        background:var(--bg);border-radius:10px;margin-top:8px;border:1px solid var(--border)">

        <!-- A 기준 -->
        <div>
          <div style="font-size:12px;font-weight:800;color:var(--text);margin-bottom:10px">
            📈 A — 발생가능성
          </div>
          ${Object.entries(A_CRITERIA).reverse().map(([v, desc]) => {
            const lbl = A_LABELS[v];
            return `<div style="display:flex;gap:8px;margin-bottom:8px;align-items:flex-start">
              <span style="flex-shrink:0;padding:2px 8px;border-radius:6px;font-size:11px;
                font-weight:800;background:${lbl.bg};color:${lbl.color};min-width:52px;
                text-align:center">${v} ${lbl.short}</span>
              <span style="font-size:11px;color:var(--muted);line-height:1.5">${desc}</span>
            </div>`;
          }).join('')}
        </div>

        <!-- B 기준 -->
        <div>
          <div style="font-size:12px;font-weight:800;color:var(--text);margin-bottom:10px">
            💥 B — 영향의 심각성
          </div>
          ${Object.entries(B_CRITERIA).reverse().map(([v, cats]) => {
            const lbl = B_LABELS[v];
            return `<div style="display:flex;gap:8px;margin-bottom:8px;align-items:flex-start">
              <span style="flex-shrink:0;padding:2px 8px;border-radius:6px;font-size:11px;
                font-weight:800;background:${lbl.bg};color:${lbl.color};min-width:72px;
                text-align:center">${v} ${lbl.short}</span>
              <div style="font-size:11px;color:var(--muted);line-height:1.6">
                <div>⚖ 법적: ${cats.법적}</div>
                <div>💰 재무: ${cats.재무적}</div>
                <div>🏢 전략: ${cats.전략적}</div>
              </div>
            </div>`;
          }).join('')}
        </div>
      </div>
    </details>`;

  // ── 진행률 바 ──
  const progressHtml = `
    <div style="display:flex;align-items:center;gap:12px;margin-bottom:16px">
      <div style="flex:1;height:8px;background:var(--border);border-radius:4px">
        <div style="height:100%;border-radius:4px;background:var(--accent);
          width:${pct}%;transition:width .4s"></div>
      </div>
      <span style="font-size:12px;font-weight:700;color:var(--accent);white-space:nowrap">
        ${assessed}/${RISKS.length}건 완료
        ${unassessed > 0 ? `· <span style="color:var(--red)">미평가 ${unassessed}건</span>` : '· <span style="color:var(--green)">전수 완료</span>'}
      </span>
    </div>`;

  // ── 테이블 ──
  const tableHtml = `
    <div class="overflow-x">
      <table class="tbl">
        <thead>
          <tr>
            <th style="width:36px">순위</th>
            <th style="width:70px">영역</th>
            <th style="width:90px">법규명</th>
            <th>리스크 내용</th>
            <th style="width:80px">관련 조직</th>
            <th style="width:140px;text-align:center">A — 발생가능성</th>
            <th style="width:160px;text-align:center">B — 영향의 심각성</th>
            <th style="width:80px;text-align:center">X (고유)</th>
            <th style="width:50px"></th>
          </tr>
        </thead>
        <tbody>
          ${sorted.map((r, i) => {
            const rowBg = !r.x_level ? '' :
              r.x_level === 'VH' ? 'background:#FEF2F2' :
              r.x_level === 'H'  ? 'background:#FFF7ED' : '';
            const unassessedBadge = !r.a_likelihood || !r.b_impact
              ? `<span style="font-size:10px;padding:1px 6px;border-radius:4px;
                  background:#FEF2F2;color:var(--red);font-weight:700">미평가</span> ` : '';
            return `
              <tr style="${rowBg}">
                <td style="text-align:center;font-size:11px;font-weight:800;
                  color:var(--muted)">${i+1}</td>
                <td><span class="cat-tag">${_esc(r.domain||r.cat)}</span></td>
                <td style="font-size:11px;max-width:90px;white-space:nowrap;
                  overflow:hidden;text-overflow:ellipsis"
                  title="${_esc(r.lawName||'-')}">${_esc(r.lawName||'-')}</td>
                <td>
                  ${unassessedBadge}
                  <span style="font-size:12px;font-weight:600"
                    title="${_esc(r.riskContent||r.title)}">${
                      _esc((r.riskContent||r.title||'').slice(0,45) +
                        ((r.riskContent||r.title||'').length>45?'…':''))
                    }</span>
                </td>
                <td style="font-size:11px;color:var(--muted)">${_esc(r.relatedOrg||r.team)}</td>
                <td style="text-align:center">${aSelect(r)}</td>
                <td style="text-align:center">${bSelect(r)}</td>
                <td style="text-align:center">${levelBadgeHtml(r.x_level)}</td>
                <td><button class="act-btn" onclick="saveRiskAB('${r.id}')">저장</button></td>
              </tr>`;
          }).join('')}
        </tbody>
      </table>
    </div>
    <div style="padding:16px 0 2px;display:flex;justify-content:flex-end;gap:10px">
      <button class="btn btn-ghost" onclick="renderMasterAssessTab()">새로고침</button>
      <button class="btn btn-primary" onclick="saveAllRiskAB()">✅ 전체 저장 (${RISKS.length}건)</button>
    </div>`;

  el.innerHTML = `
    <div class="card" style="margin-bottom:16px">
      <div style="display:flex;align-items:center;justify-content:space-between;
        flex-wrap:wrap;gap:10px;margin-bottom:16px">
        <div class="card-title">📊 A·B 평가 (발생가능성 × 영향의 심각성 → X 자동 계산)</div>
      </div>
      ${progressHtml}
      ${criteriaHtml}
      ${tableHtml}
    </div>`;
}
```

---

## 4. updateRiskAB() 수정 — 저장 시 행 색상 갱신

현재 `updateRiskAB()` 호출 시 `renderMasterAssessTab()` 전체를 재렌더링하는데, 550건일 때 성능 문제가 생길 수 있습니다.

### 수정안: 행만 부분 갱신

```js
function updateRiskAB(riskId, type, val) {
  const r = RISKS.find(x => x.id === riskId);
  if (!r) return;
  if (type === 'a') r.a_likelihood = val;
  else r.b_impact = val;

  if (r.a_likelihood && r.b_impact) {
    r.x_level = xLevelFromMatrix(r.a_likelihood, r.b_impact);
    r.score    = r.a_likelihood * r.b_impact;
    r.selected = ['H', 'VH'].includes(r.x_level);
    // Y 있으면 Z도 재계산
    if (r.y_level) r.z_level = zLevelFromMatrix(r.x_level, r.y_level);
  }

  // X 배지만 해당 행에서 부분 갱신 (전체 재렌더링 방지)
  // <td>에 id 부여 필요: id="xbadge-{riskId}"
  const xCell = document.getElementById(`xbadge-${riskId}`);
  if (xCell) xCell.innerHTML = levelBadgeHtml(r.x_level);

  // 행 배경색도 갱신
  const row = xCell?.closest('tr');
  if (row) {
    row.style.background =
      r.x_level === 'VH' ? '#FEF2F2' :
      r.x_level === 'H'  ? '#FFF7ED' : '';
  }

  // 진행률 바도 갱신
  const assessed = RISKS.filter(r2 => r2.a_likelihood && r2.b_impact).length;
  const pct = RISKS.length ? Math.round(assessed / RISKS.length * 100) : 0;
  const progressBar = document.querySelector('#master-assess-content .assess-progress-bar');
  if (progressBar) progressBar.style.width = pct + '%';
}
```

→ `renderMasterAssessTab()` 내 테이블 td X 셀에 `id="xbadge-${r.id}"` 추가 필요

---

## 5. Phase 2 완료 체크포인트 (수정안)

```js
case 'assess': {
  const total     = RISKS.length;
  const assessed  = RISKS.filter(r => r.a_likelihood && r.b_impact && r.x_level).length;
  const highRisks = RISKS.filter(r => ['H','VH'].includes(r.x_level));
  const highAll   = highRisks.length === 0 ||
                    highRisks.every(r => r.a_likelihood && r.b_impact);
  const pct       = total > 0 ? assessed / total : 0;

  return [
    {
      label:  `A·B 평가 완료 90% 이상 (${assessed}/${total}건 · ${Math.round(pct*100)}%)`,
      met:    total > 0 && pct >= 0.9,
      detail: `${assessed}/${total}건`,
    },
    {
      label:  `H·VH 등급 리스크 전수 평가 완료 (${highRisks.length}건)`,
      met:    highAll && total > 0,
      detail: highRisks.length > 0 ? `H이상 ${highRisks.length}건` : '해당 없음',
    },
  ];
}
```

**기존 대비 변경:**
- 전체 100% 강제 → **90% 이상** 으로 완화 (일부 미분류 리스크 허용)
- H·VH 등급은 반드시 전수 평가 (핵심 리스크 누락 방지)

---

## 6. Phase 2 상세 페이지(phase2) 카드 수정

```js
// _getActivePhase() 내 assess phase cards 수정
{ id:'assess', label:'리스크 평가', cards:[
  {
    icon: '📊',
    title: 'A·B 평가 입력',
    why: '발생가능성(A)과 영향의 심각성(B)을 선택하면 고유리스크(X)가 자동 계산됩니다.',
    nav: 'master',
    count: () => RISKS.filter(r => r.a_likelihood && r.b_impact && r.x_level).length,
    unit_fn: () => `/${RISKS.length}건 평가 완료`,
    color: 'var(--accent)',
    action: () => { navigateTo('master'); setTimeout(() => switchMasterTab('assess'), 80); }
  },
  {
    icon: '🔥',
    title: '히트맵으로 위험 분포 확인',
    why: '5×5 매트릭스에서 고위험 리스크 분포를 한눈에 파악합니다.',
    nav: 'dash',
    count: () => RISKS.filter(r => ['H','VH'].includes(r.x_level)).length,
    unit: '건 H·VH 등급',
    color: 'var(--red)',
  },
]}
```

---

## 7. 메뉴 연결 수정

### navigateTo() 내 추가

```js
// Phase 2 카드 클릭 시 Master A·B 탭 자동 선택
if (id === 'phase2') {
  renderPhaseDetail(1);
  // Phase 2 상세 페이지의 "A·B 평가" 카드가 클릭되면 아래 실행
  // navigateTo('master') + switchMasterTab('assess')
}
```

### phase2 상세 페이지에서 카드 클릭 시 처리

현재 `nav: 'master'`로 가기만 하고 탭 전환이 안 됩니다.

**수정:** `action` 속성을 카드 렌더링에 반영하거나, phase2 카드 onclick에 직접 기술:

```js
onclick="navigateTo('master');setTimeout(()=>switchMasterTab('assess'),80)"
```

---

## 8. 추가: X 분포 요약 카드 (A·B 평가 탭 상단)

평가 완료 후 결과 요약을 바로 볼 수 있도록 X 분포 카드 추가:

```js
function _renderXDistSummary() {
  const dist = { VH:0, H:0, M:0, L:0, VL:0 };
  RISKS.forEach(r => { if (r.x_level) dist[r.x_level]++; });
  const total = Object.values(dist).reduce((a,b) => a+b, 0);
  return `
    <div style="display:flex;gap:10px;flex-wrap:wrap;margin-bottom:16px">
      ${['VH','H','M','L','VL'].map(lv => {
        const c = LEVEL_COLOR[lv];
        const n = dist[lv];
        const pct = total ? Math.round(n/total*100) : 0;
        return `
          <div style="flex:1;min-width:80px;padding:12px;border-radius:10px;
            background:${c.bg};border:1px solid ${c.border};text-align:center">
            <div style="font-size:18px;font-weight:900;color:${c.text}">${n}</div>
            <div style="font-size:10px;font-weight:700;color:${c.text}">${lv}</div>
            <div style="font-size:10px;color:${c.text};opacity:.7">${pct}%</div>
          </div>`;
      }).join('')}
    </div>`;
}
```

→ `criteriaHtml` 위에 삽입

---

## 9. 수정 우선순위

| 우선순위 | 항목 | 공수 |
|---|---|---|
| 🔴 즉시 | `A_LABELS`, `B_LABELS`, `A_CRITERIA`, `B_CRITERIA` 상수 추가 | 소 |
| 🔴 즉시 | `renderMasterAssessTab()` — 선택지를 숫자→레이블로 변경 | 소 |
| 🔴 즉시 | `renderMasterAssessTab()` — 평가 기준 접이식 패널 추가 | 중 |
| 🔴 즉시 | `renderMasterAssessTab()` — 진행률 바 추가 | 소 |
| 🟡 중요 | `updateRiskAB()` — 부분 갱신으로 성능 개선 | 중 |
| 🟡 중요 | X 분포 요약 카드 추가 | 소 |
| 🟡 중요 | phase2 카드 클릭 → Master A·B 탭 자동 전환 | 소 |
| 🟢 권장 | `_checkPhaseConditions` assess 케이스 업데이트 | 소 |
| 🟢 권장 | 히트맵 5×5 수정 (현재 4×4) | 중 |
