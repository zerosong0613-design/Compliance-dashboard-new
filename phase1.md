# Phase 1 — 리스크 식별 전체 업무 프로세스 설계서

---

## 1. 업무 흐름 전체 그림

```
[CP팀] 작년 엑셀 업로드
    ↓ 500+ 리스크 자동 매핑 (영역·법규·조직·A/B/X/Y/Z 포함)
[CP팀] 담당 조직 확인 · 조정
    ↓ 엑셀의 "Risk 발생가능 조직" 컬럼 → team/relatedOrg 자동 할당
[CP팀] 배당 확인 메일 발송 (Power Automate)
    ↓ 각 팀 담당자에게 "배당 리스크 확인 요청" 메일
[팀 담당자] 배당된 리스크 확인 → 의견 제출
    ↓ 이의없음 / 추가요청 / 수정요청 / 삭제요청 선택 + 내용 입력
[CP팀] 팀 의견 검토 → 수용 / 기각 처리
    ↓ 수용 시 RISKS 데이터 실제 반영 (추가/수정/삭제)
[CP팀] Phase 1 완료 확정 클릭
    ↓ 아카이브 스냅샷 저장 → Phase 2 활성화
```

---

## 2. 필요한 페이지 3개

### 페이지 A — 리스크 Master (view-master) [CP팀]

**역할:** 엑셀 업로드 → 전체 리스크 목록 등록·관리

**현재 상태:** 구현 완료 (탭 2개 구조 있음)

**문제점 및 수정 사항:**

#### 문제 1 — 엑셀 업로드 시 팀 할당 현황이 보이지 않음
- 업로드 완료 후 "팀별 배정 현황 요약" 배너가 없음
- **수정:** 업로드 완료 toast 대신 "임포트 결과 요약 카드" 표시
  - 총 N건 등록, 팀별 건수 (PD팀 45건, PP팀 38건 …)
  - "배당 확인 메일 발송" 버튼 바로 노출

#### 문제 2 — relatedOrg가 "PP팀/PD팀" 형식일 때 team 필드가 첫 팀만 저장됨
- team 필드: `relatedOrg.split('/')[0]` → 배당 기준으로는 맞지만
- 의견 제출 뷰에서 `(r.relatedOrg||r.team||'').includes(appTeam)` 로 필터링하므로 relatedOrg에 여러 팀이 있을 때 정상 작동
- **수정 불필요** (현재 로직 맞음)

#### 문제 3 — 엑셀에 이미 A/B/X/Y/Z가 있으면 Phase 진행 상태가 애매함
- 작년 엑셀에는 A·B·X·Y·Z가 모두 채워져 있음
- 업로드하면 값들이 들어오지만, Phase 2/3 완료 상태는 아님
- **수정:** 엑셀 업로드 모달에 옵션 추가:
  ```
  [옵션] 이전 연도 평가값 유지 (A/B/X/Y/Z 그대로 가져오기)
  [옵션] 리스크 목록만 가져오기 (A/B/X/Y/Z 초기화)
  ```
  → 기본값: "이전 연도 평가값 유지"

#### 문제 4 — 의견 현황 컬럼이 마스터 테이블에 없음
- **수정:** 테이블에 "의견현황" 컬럼 추가
  - `identifyOpinionStatus` 기준으로 배지 표시
  - 미제출(회색) / 제출완료(파랑) / 이의없음(초록)

---

### 페이지 B — 배당 리스크 확인·의견 제출 (view-identify-opinion) [팀 담당자]

**현재 상태:** 구현 완료 (renderIdentifyOpinionView 함수 존재)

**문제점 및 수정 사항:**

#### 문제 1 — 담당자가 처음 들어왔을 때 "배당된 리스크가 없음"처럼 보일 수 있음
- 필터 조건: `(r.relatedOrg||r.team||'').includes(appTeam)`
- 엑셀의 `relatedOrg`가 "PD팀"인데 appTeam이 "PD팀"이면 정상 작동
- 하지만 appTeam = "생산팀", relatedOrg = "PP팀/PD팀"처럼 불일치하면 빈 목록
- **수정:** 팀명 매핑 테이블 또는 로그인 시 팀명을 엑셀 조직명과 일치시키는 드롭다운 제공

#### 문제 2 — 안내 메시지가 없어 처음 접속한 담당자가 뭘 해야 할지 모름
- **수정:** 상단에 안내 박스 추가:
  ```
  📋 CP팀에서 배당한 리스크를 확인하고 의견을 제출해주세요.
  - 내용에 이의가 없으면 "이의없음"을 선택하세요.
  - 수정·추가·삭제가 필요하면 의견 내용을 작성해 주세요.
  - 제출 후에는 수정이 불가합니다. 신중하게 제출해주세요.
  제출 기한: {기한 표시} (Phase 1 완료 확정 전까지)
  ```

#### 문제 3 — 좌측 목록에 리스크 내용이 너무 길어서 잘림
- **수정:** 좌측 패널에 법규명 + 리스크 내용 앞 30자 + X등급 표시로 정리

#### 문제 4 — 제출 후 재수정 불가 처리가 UI상 명확하지 않음
- 제출 완료 시 textarea disabled 처리는 되어 있음 ✅
- **수정:** 제출 완료 항목에 "CP팀 처리 대기 중" 배지 추가

#### 문제 5 — 팀 담당자용 NAV에 이 페이지만 있고 현황 요약이 없음
- **수정:** 팀 담당자 로그인 후 첫 화면(myteam)에 Phase 1 안내 카드 추가
  - "📋 N건의 리스크 식별 의견을 제출해야 합니다 → [의견 제출하기]"

---

### 페이지 C — 식별 의견 검토·확정 (view-identify-confirm) [CP팀]

**현재 상태:** 구현 완료 (renderIdentifyConfirmView 함수 존재)

**문제점 및 수정 사항:**

#### 문제 1 — "이의없음" 제출 건은 목록에 안 나옴 → 전체 진행 현황 파악 불가
- 현재: `withOpinion = RISKS.filter(r => r.identifyOpinionType !== '이의없음')`
- **수정:** 화면 상단에 팀별 의견 제출 현황 요약 카드 추가

  ```
  ┌─────────────┬─────────┬─────────┬─────────┬────────┐
  │ 팀명         │ 전체    │ 이의없음 │ 의견있음 │ 미제출 │
  ├─────────────┼─────────┼─────────┼─────────┼────────┤
  │ PD팀         │  91건   │  85건   │   4건   │  2건   │
  │ PP팀         │  45건   │  40건   │   3건   │  2건   │
  │ ...          │ ...     │ ...     │  ...    │ ...    │
  └─────────────┴─────────┴─────────┴─────────┴────────┘
  전체 의견 처리 현황: 의견있음 N건 중 M건 처리 완료
  ```

#### 문제 2 — 수용 처리 시 실제 RISKS 데이터가 변경되지 않음
- 현재 `confirmIdentifyOpinion()`: `identifyConfirmStatus`만 바꿈
- 추가 요청이 수용됐을 때 실제로 새 리스크가 RISKS에 추가되어야 함
- 수정 요청이 수용됐을 때 해당 리스크의 내용이 변경되어야 함
- 삭제 요청이 수용됐을 때 해당 리스크에 `deleted: true` 플래그 또는 실제 삭제
- **수정:** `confirmIdentifyOpinion(riskId, 'accepted')` 내부에 opinionType별 처리 분기 추가:

  ```js
  if (status === 'accepted') {
    if (r.identifyOpinionType === '추가요청') {
      // 팀 담당자의 의견 내용을 기반으로 신규 리스크 draft 생성
      // → CP팀이 직접 Master에서 추가하도록 안내 토스트
      toast('추가 요청을 수용했습니다. 리스크 Master에서 직접 추가해주세요.', 'info');
    } else if (r.identifyOpinionType === '수정요청') {
      // 수정 요청된 내용을 적용할 수 있는 인라인 편집 UI 제공
      // → 또는 "Master에서 직접 수정해주세요" 안내
    } else if (r.identifyOpinionType === '삭제요청') {
      // r.identifyDeleteRequested = true 플래그 설정
      // CP팀이 Master에서 최종 삭제 확인
    }
  }
  ```

#### 문제 3 — 전체 확정 버튼이 없음
- 현재: 개별 수용/기각만 있고 "전체 리스크 식별 확정" 버튼이 없음
- **수정:** 하단에 확정 섹션 추가:

  ```
  ┌───────────────────────────────────────────────────┐
  │  전체 의견 처리 현황                                │
  │  수용 7건 · 기각 2건 · 미처리 0건                   │
  │                                                   │
  │  [✅ 리스크 식별 완료 확정]   ← Phase 1 완료 처리   │
  │  (미처리 건이 있으면 비활성화)                       │
  └───────────────────────────────────────────────────┘
  ```
  - 이 버튼이 `confirmPhaseComplete('identify')`를 호출
  - 미처리 pending 건이 있으면 버튼 disabled + 안내 텍스트

---

## 3. Phase 1 완료 체크포인트 (수정안)

```js
case 'identify': {
  const hasRisks         = RISKS.length >= 1;
  const assignNotifySent = phaseStatus.identify?.assignNotifySent || false;

  // 분모: 전체 RISKS.length (relatedOrg 여부 무관)
  const opinionSubmitted = RISKS.filter(r => r.identifyOpinionStatus === '제출완료').length;
  const opinionDone      = RISKS.length > 0 && (opinionSubmitted / RISKS.length) >= 0.8;

  // 의견있는 건 전수 처리 완료 여부 (이의없음 제외한 제출 건)
  const withOpinion     = RISKS.filter(r =>
    r.identifyOpinionStatus === '제출완료' && r.identifyOpinionType !== '이의없음'
  );
  const allReviewed     = withOpinion.length === 0 ||
                          withOpinion.every(r => r.identifyConfirmStatus !== 'pending');

  return [
    {
      label: '리스크 1건 이상 등록 (엑셀 업로드)',
      met:    hasRisks,
      detail: `${RISKS.length}건 등록`,
    },
    {
      label:  '배당 확인 메일 발송',
      met:    assignNotifySent,
      detail: assignNotifySent ? '발송 완료' : '미발송',
      manual: true,
      key:    'assignNotifySent',
    },
    {
      label:  `팀 의견 수렴 80% 이상 (${opinionSubmitted}/${RISKS.length}건)`,
      met:    opinionDone,
      detail: `${opinionSubmitted}건 제출`,
    },
    {
      label:  '의견 있는 건 전수 검토 처리 완료',
      met:    allReviewed,
      // manual이 아님 — 데이터 기반으로 자동 판정
      detail: allReviewed
        ? `${withOpinion.length}건 처리 완료`
        : `${withOpinion.filter(r=>r.identifyConfirmStatus==='pending').length}건 미처리`,
    },
  ];
}
```

### 기존 조건 대비 변경점

| 항목 | 기존 | 수정 |
|---|---|---|
| 의견 수렴 분모 | `relatedOrg 있는 건 수` → 0이면 조건 자동 충족 버그 | `RISKS.length` 고정 |
| 의견 검토 완료 | 수동 체크 (`opinionReviewed` key) | **데이터 자동 판정** (pending 건 0이면 충족) |
| 확정 버튼 위치 | `confirmPhaseComplete` 직접 호출 | `view-identify-confirm` 하단 버튼에서 호출 |

---

## 4. 단계별 미션 (phase1) 페이지 카드 수정안

현재 phase1 상세 페이지에는 카드 2개가 있음:
- 📋 RiskMaster에 리스크 등록
- 🏷 팀별 담당 배분 확인

**수정 후 카드 4개:**

```
카드 1: 📋 리스크 등록 (엑셀 업로드)
  설명: 작년 엑셀을 업로드하여 전체 리스크를 시스템에 등록합니다.
  카운트: RISKS.length건 등록됨
  이동: navigateTo('master')

카드 2: 📧 배당 확인 메일 발송
  설명: 팀 담당자에게 리스크 확인 요청 메일을 발송합니다.
  카운트: 배당된 팀 수 (Set(RISKS.map(r=>r.team)).size)개 팀
  버튼: sendAssignNotify() 직접 실행
  상태: 발송 완료 시 ✅ 배지 표시

카드 3: 💬 팀 의견 수렴 현황
  설명: 팀 담당자가 배당 리스크를 확인하고 의견을 제출합니다.
  카운트: 제출완료 N / 전체 M건
  이동: navigateTo('identify-confirm') [현황 확인용]

카드 4: ✅ 의견 검토 후 확정
  설명: 팀 의견을 검토하고 최종 리스크 목록을 확정합니다.
  카운트: 미처리 pending 건수
  이동: navigateTo('identify-confirm')
```

---

## 5. 메뉴 정리 (NAV) 수정안

현재 NAV의 혼란 원인:
- "단계별 미션" 접이식 (phase1~7 overview)
- 별도로 "Phase 1 — 리스크 식별" 섹션에 Master, 의견 검토·확정
- Phase 2는 섹션 없이 누락

**수정안 — 단계별 미션 서브메뉴를 실제 작업 페이지로 교체:**

```js
const NAV_ADMIN = [
  {sec:'대시보드'},
  {id:'dash', icon:'▦', label:'전사 대시보드'},
  {id:'phasecontrol', icon:'🔄', label:'위험평가 진행 현황'},  // 서브메뉴 없음 — 단일 페이지

  {sec:'위험평가 작업'},
  // Phase 1
  {id:'master',           icon:'📋', label:'① 리스크 등록·관리'},
  {id:'identify-confirm', icon:'✅', label:'① 식별 의견 검토·확정'},
  // Phase 2: master A/B 탭으로 이동 — 별도 메뉴 불필요
  // Phase 3
  {id:'control-eval', icon:'🛡', label:'③ 통제효과성 평가'},
  // Phase 4
  {id:'residual', icon:'🎯', label:'④ 잔여리스크 확정'},
  // Phase 5
  {id:'improve-confirm', icon:'🔧', label:'⑤ 개선방안 확정'},

  {sec:'이행 관리'},
  {id:'monthly', icon:'⚖', label:'이행현황 검토', badge:0},
  {id:'teams',   icon:'👥', label:'팀 이행현황'},

  {sec:'보고서·정보'},
  {id:'reports', icon:'📊', label:'보고서 생성'},
  {id:'archive', icon:'📚', label:'아카이브'},
];
```

**팀 담당자 NAV:**
```js
const NAV_USER = [
  {sec:'현황'},
  {id:'myteam', icon:'📈', label:'내 팀 현황'},

  {sec:'내 업무'},
  {id:'identify-opinion', icon:'💬', label:'① 리스크 식별 의견 제출'},
  {id:'control-input',    icon:'🔍', label:'③ 통제수단 입력'},
  {id:'improve-submit',   icon:'📝', label:'⑤ 개선방안 제출'},
  {id:'monthly',          icon:'📝', label:'이행현황 입력', badge:0},
  {id:'evalresult',       icon:'📊', label:'평가결과·이력'},

  {sec:'정보'},
  {id:'archive', icon:'📚', label:'아카이브'},
];
```

**핵심 변경:**
- "단계별 미션" 서브메뉴(phase1~7) 완전 제거
- `phasecontrol` 페이지 → 7단계 전체 진행 현황 한 눈에 보는 단일 페이지
- 실제 작업 페이지에 번호(①③④⑤) 붙여서 Phase 번호 직관적으로 표시
- Phase 2는 Master 탭으로 처리 (별도 메뉴 없음 — 혼란 제거)

---

## 6. 수정 우선순위

| 우선순위 | 항목 | 공수 |
|---|---|---|
| 🔴 즉시 | `_checkPhaseConditions` identify 케이스 수정 (분모 버그 + 자동 판정) | 소 |
| 🔴 즉시 | `view-identify-confirm` 하단 전체 현황 요약 + 확정 버튼 추가 | 중 |
| 🟡 중요 | 엑셀 업로드 모달 — 옵션(값 유지/초기화) + 임포트 결과 요약 카드 | 중 |
| 🟡 중요 | NAV 정리 — 서브메뉴 제거, 번호 표시 | 소 |
| 🟡 중요 | `confirmIdentifyOpinion` 수용 시 opinionType별 처리 분기 | 중 |
| 🟢 권장 | `view-identify-confirm` 상단 팀별 현황 요약 표 | 중 |
| 🟢 권장 | `view-identify-opinion` 상단 안내 메시지 박스 | 소 |
| 🟢 권장 | phase1 카드 4개로 확장 | 소 |
| 🟢 권장 | Master 테이블 의견현황 컬럼 추가 | 소 |
