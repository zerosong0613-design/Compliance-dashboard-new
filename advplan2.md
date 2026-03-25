# Compliance RiskOps — advplan2.md (최종본)
# 위험평가 프로세스 2차 개편 + 실제 엑셀 구조 반영

> **기준 파일:** 현재 `index.html` (7단계 Phase 구조 반영본)  
> **핵심 목표 3가지:**
> 1. CP팀 ↔ 현업 담당자 협업 워크플로우로 재설계
> 2. 실제 운영 엑셀(`전체 Risk Profile_통제수준 진단결과.xlsx`) 컬럼 구조에 맞게 임포트/익스포트 전면 재구성
> 3. 스코어링 시스템을 실제 매트릭스 기준(5단계, 룩업 테이블)으로 전환

---

## 0. 전체 설계 원칙

### 0-1. 역할 구분
| 역할 | 설명 | 주요 작업 |
|---|---|---|
| **관리자 (admin)** | CP팀 | 리스크 등록·엑셀 업로드, A/B 평가, 효과성(Y) 평가, 의견 확정, 개선방안 확정 |
| **팀 담당자 (user)** | 각 사업팀 | 배당 리스크 의견 제출, 통제수단 내용 입력, 개선방안 제출 |

### 0-2. 페이지 분리 원칙
- Phase 상세(phase1~7): **진행 현황 + 바로가기 카드만** 표시
- 실제 데이터 입출력: **독립된 전용 view 페이지**에서 처리
- phase4/5의 인터랙티브 하단 패널(`#residual-interactive`, `#improve-interactive`): **완전 제거**

### 0-3. 기존 페이지 처리
| 기존 페이지 | 처리 |
|---|---|
| `view-master` | 유지 + 탭 2개로 확장 (리스크 목록 / A·B 평가) |
| `view-selection` | **삭제** → `view-residual`로 완전 대체 |

---

## 1. 스코어링 시스템 전면 수정 — 실제 엑셀 기준

### 1-1. 5단계 레벨 체계 (VL 추가)

기존 4단계(L/M/H/VH)를 **5단계(VL/L/M/H/VH)**로 전환.  
모든 레벨 관련 함수, 배지, 히트맵, 매트릭스에 적용.

| 레벨 | 표시 | 색상 | 배경 |
|---|---|---|---|
| VL | 매우 낮음 | `#22C55E` (진초록) | `#F0FDF4` |
| L  | 낮음 | `#84CC16` (연초록) | `#F7FEE7` |
| M  | 보통 | `#F59E0B` (황색) | `#FFFBEB` |
| H  | 높음 | `#F97316` (주황) | `#FFF7ED` |
| VH | 매우 높음 | `#EF4444` (적색) | `#FEF2F2` |

### 1-2. A 발생가능성 — 5단계 기준

| 점수 | 등급 | 기준 |
|---|---|---|
| 5 | 확신 | 과거 6개월 내 발생 경험 + 후속조치 未수행 → 지속 발생 가능성 매우 높음 |
| 4 | 높음 | 과거 1년 내 발생 경험 + 후속조치 수행 → 향후에도 발생가능성 높음 |
| 3 | 보통 | 과거 3년 내 발생 경험 + 후속조치·주기 모니터링 → 향후 발생 가능성 있음 |
| 2 | 낮음 | 과거 5년 내 발생 경험 + 근본 원인 제거·후속조치 수행 → 발생가능성 낮음 |
| 1 | 희박 | 발생경험 없음, 현재·미래 발생가능성 거의 없음 |

### 1-3. B 영향의 심각성 — 5단계 기준

| 점수 | 등급 | 법적 | 재무적 | 전략적 |
|---|---|---|---|---|
| 5 | 매우 심각 | 대표자 처벌, 100억↑ 과징금 | 100억↑ 재무 손실 | 면허 상실, 부정적 언론보도 등 심각한 영향 |
| 4 | 상당한 영향 | 2년↑ 징역 또는 2억↑ 벌금 | 50억~100억 재무 손실 | 비즈니스에 중대한 영향 |
| 3 | 보통의 영향 | 2년↓ 징역 또는 2억↓ 벌금 | 10억~50억 재무 손실 | 비즈니스에 상당한 영향 |
| 2 | 낮은 영향 | 과태료(징역·벌금 없음), 10억↓ 과징금 | 1억~10억 재무 손실 | 비즈니스에 경미한 영향 |
| 1 | 미미한 영향 | 형사처벌·과징금·과태료 없음 | 1억↓ 재무 손실 | 비즈니스에 미미한 영향 |

### 1-4. X 고유리스크 — 5×5 매트릭스 룩업 (수식 계산 아님)

```js
/* ── 실제 엑셀 "산정 Matrix" 시트 기준 ── */
const X_MATRIX = {
  //          B=1    B=2    B=3    B=4    B=5
  5: { 1:'M',  2:'H',  3:'VH', 4:'VH', 5:'VH' },
  4: { 1:'M',  2:'H',  3:'H',  4:'VH', 5:'VH' },
  3: { 1:'L',  2:'M',  3:'H',  4:'H',  5:'VH' },
  2: { 1:'VL', 2:'L',  3:'M',  4:'H',  5:'H'  },
  1: { 1:'VL', 2:'VL', 3:'L',  4:'M',  5:'H'  },
};
function xLevelFromMatrix(a, b) {
  return X_MATRIX[a]?.[b] || '';
}
// 기존 xLevel(n), scoreStyle(s) 등 수치 기반 함수는 폐기
```

### 1-5. Y 통제효과성 — 5단계 기준 및 판단 근거

| 등급 | 기준 (아래 중 해당 항목으로 판단) |
|---|---|
| VH | ① KPI 통한 주기적 모니터링 ② 경영자 보고 ③ 일상/주기적 감사 ④ CP부서 사전 검토 ⑤ 관련 경영시스템 인증 (ISO·KS·AEO·ISMS 등) |
| H  | 법무·CP팀 등 타부서 이중검토 또는 별도 위원회 검토 |
| M  | ① 문서화된 절차 수립 ② 주기적 교육 실시 ③ 투명성 제고 통제체계 수립 |
| L  | 통제수단은 없으나, 관련 업무가 조직 내에 할당되어 있음 |
| VL | 해당 의무사항에 대한 통제수단이 없음 |

```js
const Y_CRITERIA = {
  VH: 'KPI 모니터링 + 경영자 보고 + 감사 대상 + CP 사전검토 + 경영시스템 인증',
  H:  '법무·CP팀 이중검토 또는 별도 위원회 검토',
  M:  '문서화된 절차 + 주기적 교육 + 투명성 통제체계 수립',
  L:  '통제수단 없으나 담당 업무가 조직 내 할당',
  VL: '해당 의무에 대한 통제수단 없음',
};
```

### 1-6. Z 잔여리스크 — 5×5 매트릭스 룩업 (수식 계산 아님)

```js
/* ── 실제 엑셀 "통제수단의 효과성에 따른 잔여 리스크 산정 기준" 기준 ── */
const Z_MATRIX = {
  //           Y=VH    Y=H    Y=M    Y=L    Y=VL
  VH: { VH:'L',  H:'M',  M:'H',  L:'VH', VL:'VH' },
  H:  { VH:'VL', H:'L',  M:'M',  L:'H',  VL:'H'  },
  M:  { VH:'VL', H:'VL', M:'L',  L:'M',  VL:'M'  },
  L:  { VH:'VL', H:'VL', M:'VL', L:'L',  VL:'L'  },
  VL: { VH:'VL', H:'VL', M:'VL', L:'VL', VL:'VL' },
};
function zLevelFromMatrix(xLv, yLv) {
  return Z_MATRIX[xLv]?.[yLv] || '';
}
// 기존 zLevel(n), yNumeric(level) 등 수치 기반 함수는 폐기
// y_control 수치 필드 불필요 — y_level 텍스트만 사용
```

### 1-7. 히트맵 5×5로 변경

`renderHeatmap()`을 4×4에서 **5×5**로 변경.
- X축: 발생가능성 A (1희박~5확신)
- Y축: 영향의 심각성 B (1미미~5매우심각)
- 셀 내용: `xLevelFromMatrix(a, b)` 결과 레벨 배지 + 해당 리스크 건수
- `showHeatmapDrilldown(a, b)`: `r.a_likelihood === a && r.b_impact === b` 기준 필터

---

## 2. RISK 데이터 모델 — 실제 엑셀 컬럼 기준 전면 재정의

실제 운영 엑셀(`전체 Risk Profile_통제수준 진단결과.xlsx`)의 컬럼 구조를 기준으로 RISK 객체를 재정의한다.

```js
{
  /* ── 시스템 관리 필드 ── */
  id:                  string,   // 시스템 자동 부여 (R001, R002...)
  spId:                number,   // SharePoint ID (SP 연동 시)
  year:                number,   // 귀속 연도 (엑셀 업로드 연도)

  /* ── 1. 리스크 식별 (Phase 1) ── */
  domain:              string,   // 영역: 'HR'|'정보보호'|'IP(지재권)'|'영업비밀'|'회계세무'|'M&A'|'IR'|'부동산'|'기타'|'안전보건'|'환경'|'공정거래'|'반독점'|'반부패'
  lawName:             string,   // 1-1. 법규명 (근로기준법 등)
  riskType:            string,   // 1-2. Risk 유형
  riskContent:         string,   // Risk 내용 (상세 설명) — 현재 title 역할
  relatedClause:       string,   // 1-3. 세부 조항 (법 제17조 제1항 등)
  relatedOrg:          string,   // 1-4. Risk 발생가능 조직 (PD팀, PP팀/PD팀 등)

  /* ── 1. 리스크 식별: 팀 의견 수렴 ── */
  identifyOpinion:       string,  // 팀 담당자 의견 내용
  identifyOpinionType:   string,  // '이의없음'|'추가요청'|'수정요청'|'삭제요청'
  identifyOpinionStatus: string,  // '미제출'|'제출완료'
  identifyOpinionAt:     string,  // 제출 일자
  identifyConfirmStatus: string,  // 'pending'|'accepted'|'rejected'
  identifyConfirmComment:string,  // CP팀 처리 의견

  /* ── 2. 리스크 평가 (Phase 2) ── */
  a_likelihood:        number,   // 2-1. 발생가능성 A (1~5)
  b_impact:            number,   // 2-2. 영향의 심각성 B (1~5)
  x_level:             string,   // 2-3. 고유Risk X — xLevelFromMatrix(a, b) 결과 (VL/L/M/H/VH)

  /* ── 3. 통제수준 진단 (Phase 3) ── */
  controlMeasure:        string,  // 3-1. 현재 통제수단 (팀 담당자 입력)
  controlMeasureStatus:  string,  // '미입력'|'입력완료'
  y_level:               string,  // 3-2. 통제효과성 Y — CP팀 평가 (VL/L/M/H/VH)
  y_evalComment:         string,  // CP팀 Y 평가 의견

  /* ── 4. 잔여리스크 (Phase 4) ── */
  z_level:               string,  // 4-1. 잔여Risk Z — zLevelFromMatrix(x, y) 결과 (VL/L/M/H/VH)
  z_selected:            boolean, // 잔여리스크 관리 대상 여부
  z_confirmedAt:         string,  // 확정 일자

  /* ── 5. 개선방안 (Phase 5) ── */
  improvePlan:           string,  // 개선방안 내용
  improvePlanStatus:     string,  // '미제출'|'제출완료'|'확정'|'반려'
  improvePlanAt:         string,  // 제출 일자
  improvePlanComment:    string,  // CP팀 검토 의견

  /* ── 6. 이행 관리 (Phase 6) ── */
  submitStatus:          string,  // 기존 이행현황 상태
  achieve:               number,  // 달성율
  // ... 기존 이행 관련 필드 유지

  /* ── 레거시 필드 (하위 호환 유지) ── */
  title:       string,  // riskContent와 동기화 (기존 코드 호환용)
  team:        string,  // relatedOrg 첫 번째 팀 기준 (배당팀)
  cat:         string,  // domain과 동기화
  owner:       string,  // 담당자
  selected:    boolean, // x_level H·VH 이상 자동 선정 (기존 선정 기능 대체)
  score:       number,  // 레거시. A×B 숫자값 (하위 호환용, 실제 계산엔 미사용)
}
```

**주요 변경:**
- `impact`, `likelihood` → `a_likelihood`, `b_impact` (1~5)로 교체
- `x_inherent`(숫자) → `x_level`(레벨 문자열)로 교체
- `y_control`(수치) → `y_level`(레벨 문자열)로 교체  
- `z_residual`(숫자) → `z_level`(레벨 문자열)로 교체
- `domain`, `lawName`, `riskType`, `riskContent`, `relatedClause`, `relatedOrg` 신규 추가

---

## 3. 엑셀 임포트 전면 재설계

### 3-1. 핵심 설계 방향

**이 엑셀 한 장을 업로드하면 모든 데이터가 시스템에 들어와야 한다.**

- 컬럼명이 한글(법규명, Risk 유형 등)이므로 다중 컬럼명 fallback 파싱
- A, B가 숫자(1~5)로 이미 입력된 경우 → x_level을 매트릭스로 자동 계산
- Y가 텍스트(VH, H, M, L, VL)로 입력된 경우 → z_level을 매트릭스로 자동 계산
- 관련 조직이 "PD팀", "PP팀/PD팀" 등으로 입력 → team 필드로 첫 번째 팀 추출

### 3-2. 엑셀 컬럼 매핑 테이블

| 엑셀 컬럼명 (한글) | 대체 컬럼명 | RISK 필드 | 처리 방식 |
|---|---|---|---|
| 영역 | domain, Domain | `domain` | 텍스트 그대로 |
| 법규명 | 관련법규, LawName | `lawName` | 텍스트 그대로 |
| Risk 유형 | 리스크유형, RiskType | `riskType` | 텍스트 그대로 |
| Risk 내용 | 리스크내용, RiskContent, Title | `riskContent` + `title` | 텍스트 그대로 |
| 세부 조항 | 관련조항, Clause | `relatedClause` | 텍스트 그대로 |
| Risk 발생가능 조직 | 관련조직, 발생가능조직 | `relatedOrg` + `team` | 텍스트(첫 팀 추출) |
| 발생가능성(A) | 2-1. 발생가능성, A, Likelihood | `a_likelihood` | 숫자 1~5 파싱 |
| 영향의 심각성(B) | 2-2. 영향의심각성, B, Impact | `b_impact` | 숫자 1~5 파싱 |
| 고유Risk값 | 2-3. 고유Risk, X, XLevel | `x_level` | VL/L/M/H/VH 또는 A×B로 자동 계산 |
| 현재 통제수단 | 3-1. 현재통제수단, ControlMeasure | `controlMeasure` | 텍스트 그대로 |
| 통제효과성(Y) | 3-2. 통제효과성, Y, YLevel | `y_level` | VL/L/M/H/VH 파싱 |
| 잔여Risk | 4-1. 잔여Risk, Z, ZLevel | `z_level` | VL/L/M/H/VH 또는 X×Y 매트릭스로 자동 계산 |

### 3-3. 파싱 로직 (`handleExcelUpload` 전면 재작성)

```js
function _parseRiskRow(r, rowNum) {
  // 1. 기본 텍스트 필드 파싱 (다중 컬럼명 fallback)
  const domain      = _col(r, ['영역','domain','Domain']) || '기타';
  const lawName     = _col(r, ['법규명','관련법규','LawName']) || '';
  const riskType    = _col(r, ['Risk 유형','리스크유형','RiskType']) || '';
  const riskContent = _col(r, ['Risk 내용','리스크내용','RiskContent','Title']) || '';
  const clause      = _col(r, ['세부 조항','관련조항','Clause']) || '';
  const relatedOrg  = _col(r, ['Risk 발생가능 조직','관련조직','발생가능조직','RelatedOrg']) || '';

  // 2. A, B 숫자 파싱 (1~5)
  const rawA = _col(r, ['발생가능성(A)','2-1. 발생가능성','A','Likelihood','LikelihoodScore']);
  const rawB = _col(r, ['영향의 심각성(B)','2-2. 영향의심각성','B','Impact','ImpactScore']);
  const a_likelihood = Math.min(5, Math.max(1, parseInt(rawA) || 0)) || null;
  const b_impact     = Math.min(5, Math.max(1, parseInt(rawB) || 0)) || null;

  // 3. X 레벨 — 엑셀에 이미 있으면 사용, 없으면 A×B 매트릭스로 계산
  const rawX = String(_col(r, ['고유Risk값','2-3. 고유Risk','X','XLevel']) || '').trim().toUpperCase();
  const x_level = ['VL','L','M','H','VH'].includes(rawX)
    ? rawX
    : (a_likelihood && b_impact ? xLevelFromMatrix(a_likelihood, b_impact) : '');

  // 4. 통제수단 텍스트
  const controlMeasure = _col(r, ['현재 통제수단','3-1. 현재통제수단','ControlMeasure']) || '';
  const controlMeasureStatus = controlMeasure ? '입력완료' : '미입력';

  // 5. Y 레벨 — 텍스트 파싱 (VL/L/M/H/VH)
  const rawY = String(_col(r, ['통제효과성(Y)','3-2. 통제효과성','Y','YLevel']) || '').trim().toUpperCase();
  const y_level = ['VL','L','M','H','VH'].includes(rawY) ? rawY : '';

  // 6. Z 레벨 — 엑셀에 있으면 사용, 없으면 X×Y 매트릭스로 계산
  const rawZ = String(_col(r, ['잔여Risk','4-1. 잔여Risk','Z','ZLevel']) || '').trim().toUpperCase();
  const z_level = ['VL','L','M','H','VH'].includes(rawZ)
    ? rawZ
    : (x_level && y_level ? zLevelFromMatrix(x_level, y_level) : '');

  // 7. 팀 추출 — "PP팀/PD팀" → "PP팀"
  const team = relatedOrg.split('/')[0].split(',')[0].trim() || '미배정';

  // 8. 유효성 검사
  if (!riskContent || riskContent.trim() === '') return null;

  return {
    id: '',  // 임포트 후 시스템 자동 부여
    year: parseInt(curYear),
    domain, lawName, riskType,
    riskContent, title: riskContent,  // title은 레거시 호환
    relatedClause: clause,
    relatedOrg, team,
    cat: domain,  // 레거시 호환
    a_likelihood, b_impact, x_level,
    controlMeasure, controlMeasureStatus,
    y_level, y_evalComment: '',
    z_level, z_selected: ['H','VH'].includes(z_level),  // H·VH는 자동 대상
    improvePlan: '', improvePlanStatus: '미제출', improvePlanComment: '',
    identifyOpinion: '', identifyOpinionType: '미제출', identifyOpinionStatus: '미제출',
    identifyConfirmStatus: 'pending',
    submitStatus: '미제출', achieve: null,
    owner: '-', selected: ['H','VH'].includes(x_level),
    score: (a_likelihood||0) * (b_impact||0),  // 레거시 호환
  };
}

// 다중 컬럼명 fallback 헬퍼
function _col(row, keys) {
  for (const k of keys) {
    if (row[k] !== undefined && row[k] !== null && row[k] !== '') return row[k];
  }
  return '';
}
```

### 3-4. 연도별 운영 방식 — 아카이브, 이전연도 불러오기, SharePoint 연동

#### 3-4-1. 연도별 데이터 흐름 설계

```
매년 1월:
  CP팀이 새 엑셀 업로드 (신규 법령·리스크 반영)
      ↓
  해당 연도 RISKS 데이터로 교체
      ↓
  전년도 데이터는 엑셀 파일로 아카이브 저장 (SharePoint 문서라이브러리)
      ↓
  이전 연도 불러오기 → 작년 리스크 기반으로 올해 초안 자동 생성 가능
```

#### 3-4-2. 엑셀 업로드 모달 UI 재설계

```
업로드 모달에 다음 옵션 추가:

[대상 연도 선택] ← 기본: curYear

[처리 방식 선택]
  ① 전체 교체 — 해당 연도 기존 데이터 모두 삭제 후 새 엑셀로 완전 교체
  ② 신규만 추가 — riskContent+domain 조합이 같은 건 skip, 새 건만 추가
  ③ 전체 덮어쓰기 — 기존 건은 엑셀 값으로 업데이트, 없는 건은 추가

[이전 연도 불러오기 버튼]
  → 선택한 연도의 아카이브 엑셀을 기반으로 현재 연도 초안 생성
  → A/B/X/Y/Z 값은 유지, 식별 의견·통제수단·개선방안은 초기화
  → "2024년 데이터 기반으로 2025년 초안을 생성합니다. 이후 수정 가능합니다."

업로드 완료 후 toast:
"총 587건 임포트 완료 (신규 23건 추가, 기존 564건 업데이트, 오류 0건)"
```

#### 3-4-3. 아카이브 저장 — SharePoint 연동

**아카이브 저장 시점:**
- Phase 1 완료 확정 시: 리스크 식별 완료본 자동 저장
- Phase 연도 사이클 완료(Phase 7) 시: 전체 연간 결과 자동 저장
- 관리자 수동: 언제든 "현재 상태 엑셀로 저장" 버튼

**SharePoint 저장 경로:**
```
ComplianceRiskOps/
  Documents/
    RiskMaster_Archive/
      2024_RiskMaster_v1_식별완료_20240215.xlsx
      2024_RiskMaster_v2_통제진단완료_20240430.xlsx
      2024_RiskMaster_Final_20241231.xlsx
      2025_RiskMaster_v1_식별완료_20250210.xlsx
      ...
```

**OFFLINE_MODE 대응:**
- SP 미연동 시: localStorage에 연도별 JSON 스냅샷으로 저장
- `riskops_archive_{year}_{timestamp}` 키로 관리

```js
const ARCHIVE_LS_PREFIX = 'riskops_archive_';

function saveArchiveSnapshot(year, label) {
  // 현재 RISKS 배열 + Phase 상태를 JSON으로 직렬화
  // label 예: '식별완료', '통제진단완료', 'Final'
  const snapshot = {
    year, label,
    savedAt: new Date().toISOString(),
    savedBy: appTeam,
    risks: JSON.parse(JSON.stringify(RISKS)),
    phaseStatus: _loadPhaseStatus(year),
    riskCount: RISKS.length,
    domainSummary: _buildDomainSummary(),
  };
  const key = `${ARCHIVE_LS_PREFIX}${year}_${label}_${Date.now()}`;
  localStorage.setItem(key, JSON.stringify(snapshot));

  // SP 연동 시: 엑셀 파일 생성 후 SharePoint 문서라이브러리에 업로드
  if (!CONFIG.OFFLINE_MODE) {
    _uploadArchiveToSP(snapshot, `${year}_RiskMaster_${label}_${new Date().toISOString().slice(0,10)}.xlsx`);
  }
}

async function _uploadArchiveToSP(snapshot, filename) {
  // XLSX 생성 후 SP 문서라이브러리 업로드
  // SP REST API: /_api/web/getfolderbyserverrelativeurl('Documents/RiskMaster_Archive')/files/add
}
```

#### 3-4-4. 이전 연도 불러오기 (`loadPreviousYear`)

```js
async function loadPreviousYear(targetYear) {
  // 1. SP에서 해당 연도 아카이브 파일 목록 조회
  //    OFFLINE_MODE: localStorage에서 해당 연도 스냅샷 목록 조회

  // 2. 선택 모달: "어떤 버전으로 불러오시겠습니까?"
  //    - 2024_RiskMaster_Final_20241231.xlsx (587건)
  //    - 2024_RiskMaster_v2_통제진단완료_20240430.xlsx (582건)
  //    → 선택

  // 3. 불러오기 처리:
  //    - year 를 curYear로 변경
  //    - a_likelihood, b_impact, x_level 유지 (작년 평가 기반)
  //    - y_level, z_level 유지 (작년 통제 수준 참고)
  //    - 초기화 필드: identifyOpinionStatus, controlMeasureStatus,
  //                   improvePlanStatus, submitStatus, achieve
  //    - z_selected: z_level H·VH인 건 자동 true
  //    - id 새로 부여

  // 4. 완료 안내:
  //    "2024년 데이터(587건)를 기반으로 2025년 초안이 생성되었습니다.
  //     A/B/X 평가값과 통제수단은 작년 기준으로 유지됩니다.
  //     내용을 검토하고 필요한 부분을 수정해주세요."
}
```

#### 3-4-5. view-archive 아카이브 페이지 업데이트

기존 아카이브 페이지에 다음 섹션 추가:

```
📂 연도별 RiskMaster 아카이브
┌─────────────────────────────────────────────────────┐
│ 2024년                                               │
│   📄 2024_RiskMaster_Final (587건) - 2024.12.31     │
│      [엑셀 다운로드] [이 버전으로 불러오기]           │
│   📄 2024_RiskMaster_v2 (582건) - 2024.04.30        │
│      [엑셀 다운로드]                                 │
├─────────────────────────────────────────────────────┤
│ 2025년                                               │
│   📄 2025_RiskMaster_v1 (591건) - 2025.02.10        │
│      [엑셀 다운로드] [현재 편집 중]                   │
└─────────────────────────────────────────────────────┘

[📤 현재 상태 아카이브 저장]  [📥 이전 연도 기반으로 초안 생성]
```

---

### 3-5. 엑셀 익스포트 (다운로드) 컬럼 구조

현재 엑셀과 동일한 컬럼 구조로 내보낸다.

```
영역 | 법규명 | Risk 유형 | Risk 내용 | 세부 조항 | Risk 발생가능 조직
| 발생가능성(A) | 영향의 심각성(B) | 고유Risk값(X)
| 현재 통제수단 | 통제효과성(Y) | 잔여Risk(Z) | 관리대상(z_selected)
| 개선방안 | 개선방안 상태
```

### 3-6. 엑셀 템플릿 다운로드 헤더 + 입력 안내

```
A열 [영역]: HR / 정보보호 / IP(지재권) / 영업비밀 / 회계세무 / 안전보건 / 환경 / 공정거래 / 반부패 등
B열 [법규명]: 근로기준법, 개인정보보호법 등
C열 [Risk 유형]: 자유 입력
D열 [Risk 내용]: 구체적인 리스크 행위/상황 기술
E열 [세부 조항]: 법 제00조 제00항 등 (빈칸 허용)
F열 [Risk 발생가능 조직]: PD팀, PP팀/PD팀 등 (슬래시로 복수 표시)
G열 [발생가능성(A)]: 1(희박) / 2(낮음) / 3(보통) / 4(높음) / 5(확신)
H열 [영향의 심각성(B)]: 1(미미) / 2(낮음) / 3(보통) / 4(상당) / 5(매우심각)
I열 [고유Risk값(X)]: 미입력 시 A×B 매트릭스로 자동 계산 (VL/L/M/H/VH)
J열 [현재 통제수단]: 현재 시행 중인 통제 활동 기술 (빈칸 허용)
K열 [통제효과성(Y)]: VL / L / M / H / VH (빈칸 허용)
L열 [잔여Risk(Z)]: 미입력 시 X×Y 매트릭스로 자동 계산 (VL/L/M/H/VH)
```

---

## 3-7. 리스크 마스터 데이터 변경 이력 관리 (버전 관리·백업·복원)

### 3-7-1. 설계 배경

리스크 마스터는 팀의 수정 요청, CP팀 확정, 연도 변경 등으로 지속적으로 변경된다.  
**잘못 수정했을 때 이전 상태로 복원**할 수 있어야 하고,  
**누가 언제 무엇을 바꿨는지** 추적 가능해야 한다.

### 3-7-2. 리스크별 변경 이력 필드 추가

RISK 객체에 이력 배열 추가:

```js
changeLog: [
  {
    at:     string,   // 변경 일시 (ISO datetime)
    by:     string,   // 변경자 (appTeam)
    action: string,   // 'create'|'update'|'delete'|'restore'|'import'
    fields: string[], // 변경된 필드 목록 ['a_likelihood','x_level',...]
    prev:   object,   // 변경 전 값 스냅샷 (해당 fields만)
    next:   object,   // 변경 후 값 스냅샷
    note:   string,   // 변경 사유 (선택)
    source: string,   // 'manual'|'excel_import'|'team_request'|'restore'
  }
]
```

### 3-7-3. 자동 스냅샷 저장 시점

시스템이 자동으로 스냅샷을 저장하는 시점:

| 트리거 | 스냅샷 레이블 | 저장 방식 |
|---|---|---|
| 엑셀 업로드 완료 | `import_YYYYMMDD_HHmm` | localStorage + SP |
| Phase 완료 확정 | `phase{N}_complete_YYYYMMDD` | localStorage + SP |
| 수동 저장 버튼 클릭 | `manual_YYYYMMDD_HHmm` | localStorage + SP |
| 팀 수정 요청 확정 | `amendment_riskId_YYYYMMDD` | changeLog에 기록 |

### 3-7-4. 리스크 마스터 수정 요청 워크플로우 (팀 담당자 → CP팀)

Phase 1 외에도 **연중 언제든** 팀에서 리스크 추가/수정/삭제를 요청할 수 있다.

```
① 팀 담당자: view-identify-opinion 또는 리스크 목록에서 [수정 요청] 클릭
   → 리스크 ID, 요청 유형(추가/수정/삭제), 요청 내용, 사유 입력 후 제출

② CP팀: view-identify-confirm 에서 요청 검토
   → [수용]: 실제 RISK 데이터 반영 + changeLog 기록
   → [기각]: 기각 사유 입력 + 팀에 알림

③ 수용 시 자동 처리:
   - changeLog에 이력 추가
   - 현재 상태 자동 스냅샷 저장
```

### 3-7-5. 백업 복원 UI — view-master 내 "이력·복원" 탭 추가

view-master를 **탭 3개**로 확장:

```
탭 1: 📋 리스크 목록  (기존)
탭 2: 📊 A·B 평가    (신규)
탭 3: 🕐 변경 이력·복원  (신규)
```

**탭 3 — 변경 이력·복원 화면:**

```
[스냅샷 목록]
┌────────────────────────────────────────────────────────────┐
│ ● 2025-02-10 14:32  import_20250210_1432     (591건) 현재  │
│   엑셀 임포트 · 관리자                                      │
│                                        [이 시점으로 복원]   │
├────────────────────────────────────────────────────────────┤
│   2025-01-15 09:20  phase1_complete_20250115 (589건)       │
│   Phase 1 완료 확정 · 관리자                                │
│   [엑셀 다운로드] [이 시점으로 복원]                        │
├────────────────────────────────────────────────────────────┤
│   2024-12-31 18:00  manual_20241231_1800     (587건)       │
│   수동 저장 · 관리자                                        │
│   [엑셀 다운로드] [이 시점으로 복원]                        │
└────────────────────────────────────────────────────────────┘

[리스크 개별 변경 이력]
특정 리스크 ID를 선택하면 해당 리스크의 변경 이력 타임라인 표시:
  2025-02-10  엑셀 임포트  a_likelihood: 2→3, x_level: L→M
  2025-01-20  팀 수정 요청 수용  riskContent 수정  (by 생산팀)
  2024-12-01  초기 등록  (import)
```

**복원 함수:**

```js
async function restoreSnapshot(snapshotKey) {
  const ok = await showConfirmModal({
    title: '이전 상태로 복원',
    message: `선택한 시점(${label})으로 리스크 마스터 전체를 복원합니다.<br>
              현재 상태는 자동으로 백업됩니다.<br><br>
              <strong style="color:var(--red)">이 작업은 현재 데이터를 덮어씁니다.</strong>`,
    confirmText: '복원하기',
    danger: true,
  });
  if (!ok) return;

  // 1. 현재 상태 자동 백업
  saveArchiveSnapshot(curYear, `before_restore_${Date.now()}`);

  // 2. 스냅샷 로드
  const snapshot = JSON.parse(localStorage.getItem(snapshotKey));
  RISKS = snapshot.risks;

  // 3. changeLog에 복원 이력 기록
  RISKS.forEach(r => {
    if (!r.changeLog) r.changeLog = [];
    r.changeLog.push({
      at: new Date().toISOString(), by: appTeam,
      action: 'restore', source: 'restore',
      note: `${snapshot.label} 시점으로 복원`,
    });
  });

  toast(`✅ ${snapshot.label} 시점으로 복원 완료 (${RISKS.length}건)`, 'success', 5000);
  applyMaster();
}
```

### 3-7-6. 스냅샷 저장 최대 개수 관리

localStorage 용량 관리를 위해 스냅샷 보관 정책:
- 최근 10개 스냅샷만 localStorage에 유지
- 오래된 것부터 자동 삭제 (SP 연동 시 SP에는 전부 보관)
- Phase 완료 스냅샷은 삭제하지 않고 영구 보존

```js
function _pruneSnapshots() {
  const keys = [];
  for (let i = 0; i < localStorage.length; i++) {
    const k = localStorage.key(i);
    if (k && k.startsWith(ARCHIVE_LS_PREFIX)) keys.push(k);
  }
  // 타임스탬프 기준 오래된 순 정렬 후 10개 초과분 삭제
  // (phase_complete 태그 포함된 것은 제외)
  const prunable = keys
    .filter(k => !k.includes('phase') && !k.includes('Final'))
    .sort();
  if (prunable.length > 10) {
    prunable.slice(0, prunable.length - 10).forEach(k => localStorage.removeItem(k));
  }
}
```

### 3-7-7. SharePoint 연동 시 이력 저장 (SP List 활용)

SP 연동 시 `RiskMasterHistory` List를 별도로 생성:

```
SharePoint List: RiskMasterHistory
컬럼:
  - Year (number)
  - SnapshotLabel (text)
  - SavedAt (datetime)
  - SavedBy (text)
  - RiskCount (number)
  - Action (text: import/phase_complete/manual/restore)
  - FileUrl (text: 아카이브 엑셀 파일 URL)
  - Notes (text)
```

```js
CONFIG.LIST_RISK_HISTORY = 'RiskMasterHistory';  // CONFIG에 추가
```

---

## 4. 신규 독립 페이지 7개 — HTML 추가

아래 7개 `<div class="view">` 를 `#content` 내에 추가. (기존 phase1~7 view 유지)

| # | view id | 역할 | 연관 Phase |
|---|---|---|---|
| 1 | `view-identify-opinion` | 팀 담당자 | Phase 1 |
| 2 | `view-identify-confirm` | 관리자 | Phase 1 |
| 3 | `view-control-input` | 팀 담당자 | Phase 3 |
| 4 | `view-control-eval` | 관리자 | Phase 3 |
| 5 | `view-residual` | 관리자 | Phase 4 (view-selection 대체) |
| 6 | `view-improve-submit` | 팀 담당자 | Phase 5 |
| 7 | `view-improve-confirm` | 관리자 | Phase 5 |

---

## 5. 네비게이션 재설계

### 5-1. NAV_ADMIN

```js
const NAV_ADMIN = [
  { sec:'대시보드' },
  { id:'dash', icon:'▦', label:'전사 대시보드' },

  { id:'phasecontrol', icon:'🎯', label:'단계별 미션', sub:[
    { id:'phase1', label:'1. 리스크 식별' },
    { id:'phase2', label:'2. 리스크 평가' },
    { id:'phase3', label:'3. 통제수준 진단' },
    { id:'phase4', label:'4. 잔여리스크 확정' },
    { id:'phase5', label:'5. 개선방안 도출·확정' },
    { id:'phase6', label:'6. 개선방안 이행' },
    { id:'phase7', label:'7. 점검·보고' },
  ]},

  { sec:'Phase 1 — 리스크 식별' },
  { id:'master',           icon:'📋', label:'리스크 Master' },
  { id:'identify-confirm', icon:'✅', label:'식별 의견 검토·확정' },

  { sec:'Phase 3 — 통제수준 진단' },
  { id:'control-eval', icon:'🛡', label:'통제효과성 평가 (Y)' },

  { sec:'Phase 4 — 잔여리스크' },
  { id:'residual', icon:'🎯', label:'잔여리스크 확정' },

  { sec:'Phase 5 — 개선방안' },
  { id:'improve-confirm', icon:'🔧', label:'개선방안 확정' },

  { sec:'이행 관리' },
  { id:'monthly', icon:'⚖', label:'이행현황 검토', badge:0 },
  { id:'teams',   icon:'👥', label:'팀 이행현황' },

  { sec:'보고서·정보' },
  { id:'reports', icon:'📊', label:'보고서 생성' },
  { id:'archive', icon:'📚', label:'아카이브' },
];
```

### 5-2. NAV_USER

```js
const NAV_USER = [
  { sec:'Phase 1 — 의견 제출' },
  { id:'identify-opinion', icon:'💬', label:'리스크 식별 의견' },

  { sec:'Phase 3 — 통제수단' },
  { id:'control-input', icon:'🔍', label:'통제수단 입력' },

  { sec:'Phase 5 — 개선방안' },
  { id:'improve-submit', icon:'📝', label:'개선방안 제출' },

  { sec:'이행 관리' },
  { id:'monthly',    icon:'📝', label:'이행현황 입력', badge:0 },
  { id:'evalresult', icon:'📊', label:'평가결과·이력' },

  { sec:'현황·정보' },
  { id:'myteam',  icon:'📈', label:'내 팀 이행현황' },
  { id:'archive', icon:'📚', label:'아카이브' },
];
```

### 5-3. PAGE_TITLES 추가/수정

```js
'identify-opinion':  '리스크 식별 의견 제출',
'identify-confirm':  '리스크 식별 의견 검토·확정',
'control-input':     '통제수단 입력',
'control-eval':      '통제효과성 평가',
'residual':          '잔여리스크 확정',
'improve-submit':    '개선방안 제출',
'improve-confirm':   '개선방안 확정',
// selection 삭제
```

### 5-4. navigateTo() 라우팅 추가

```js
if (id === 'identify-opinion') renderIdentifyOpinionView();
if (id === 'identify-confirm') renderIdentifyConfirmView();
if (id === 'control-input')    renderControlInputView();
if (id === 'control-eval')     renderControlEvalView();
if (id === 'residual')         renderResidualView();
if (id === 'improve-submit')   renderImproveSubmitView();
if (id === 'improve-confirm')  renderImproveConfirmView();
// selection 제거
```

---

## 6. view-master 탭 2개로 확장

### 6-1. 탭 구조

```html
<div class="view" id="view-master">
  <div style="display:flex;gap:8px;margin-bottom:16px">
    <button id="master-tab-list"   onclick="switchMasterTab('list')"  >📋 리스크 목록</button>
    <button id="master-tab-assess" onclick="switchMasterTab('assess')">📊 A·B 평가</button>
  </div>
  <div id="master-panel-list">
    <!-- 기존 toolbar + filter + table 유지 -->
    <!-- 테이블 컬럼: ID | 영역 | 법규명 | 리스크내용 | 관련조직 | A | B | X | Y | Z | 의견현황 | 액션 -->
  </div>
  <div id="master-panel-assess" style="display:none">
    <!-- A/B 인라인 평가 테이블 -->
    <!-- 정렬: x_level 내림차순 (H·VH 우선) -->
    <!-- 컬럼: 순위 | 영역 | 법규명 | 리스크내용 | 관련조직 | A선택 | B선택 | X(자동) | 저장 -->
  </div>
</div>
```

### 6-2. 리스크 목록 탭 테이블 컬럼 변경

```
기존: ID | 리스크명 | 팀 | 유형 | X(고유) | Y(통제) | Z(잔여) | 담당자 | 개선과제 | 선정 | 액션
변경: ID | 영역 | 법규명 | 리스크 내용 | 관련 조직 | A | B | X | Y | Z | 의견현황 | 액션
```

의견현황 컬럼: `identifyOpinionType` 배지 (이의없음/추가요청/수정요청/삭제요청)

### 6-3. A·B 평가 탭 동작

```js
function renderMasterAssessTab() {
  // x_level 없는 건 우선 표시 (미평가 우선)
  // A 셀: <select> 1희박/2낮음/3보통/4높음/5확신
  // B 셀: <select> 1미미/2낮음/3보통/4상당/5매우심각
  // 선택 즉시: x_level = xLevelFromMatrix(a, b), 배지 실시간 갱신
  // 행 저장 또는 "전체 저장" 버튼
}

function updateRiskAB(riskId, type, val) {
  // type: 'a' or 'b'
  // val: 1~5
  // 즉시 x_level 재계산, z_level도 y_level 있으면 재계산
}
```

---

## 7. Phase 1 — 리스크 식별 워크플로우

```
① CP팀: 엑셀 업로드 → 리스크 Master 등록 → 배당 확인 메일 발송
② 팀 담당자: view-identify-opinion 에서 배당 리스크 확인
             → 이의없음 / 추가요청 / 수정요청 / 삭제요청 선택 + 의견 제출
③ CP팀: view-identify-confirm 에서 의견 수용/기각 처리
         → 최종 리스크 확정 → Phase 1 완료
```

### view-identify-opinion (팀 담당자)

```
상단: 배당 리스크 N건 / 의견 제출 M건 진행 현황 바

split-view:
좌측: 내 팀 배당 리스크 목록 (relatedOrg에 appTeam 포함된 것)
  - 표시: 영역, 법규명, 리스크 내용 요약, 의견 상태 배지

우측 폼:
  - 리스크 정보: 영역, 법규명, 세부조항, 리스크내용, 관련조직, X등급
  - 의견 유형 radio-chip 4개: ✅이의없음 / ➕추가요청 / ✏수정요청 / ❌삭제요청
  - 의견 내용 textarea (이의없음 제외 필수)
  - [임시저장] [제출하기]
```

### view-identify-confirm (관리자)

```
상단: 팀별 의견 제출 현황 카드 (미제출/이의없음/의견있음 건수)

split-view:
좌측: 의견 있는 리스크 목록 (추가/수정/삭제 요청 건, 미확인 우선)

우측:
  - 원본 리스크 정보
  - 팀 제출 의견 박스 (의견유형 + 내용)
  - [수용] [기각] 버튼 + 처리 의견 입력
  - 수용 시 실제 리스크 데이터 수정/추가/z_selected 조정
```

### Phase 1 완료 조건

```js
case 'identify':
  return [
    { label: '리스크 1건 이상 등록', met: RISKS.length >= 1, detail: `${RISKS.length}건` },
    { label: '배당 확인 메일 발송', key:'assignNotifySent', manual:true },
    { label: '팀 의견 수렴 완료',
      met: RISKS.filter(r=>r.identifyOpinionStatus==='제출완료').length >= RISKS.filter(r=>r.relatedOrg).length * 0.8,
      detail: `${RISKS.filter(r=>r.identifyOpinionStatus==='제출완료').length}건 제출` },
    { label: '의견 검토 처리 완료', key:'opinionReviewed', manual:true },
  ];
```

---

## 8. Phase 2 — 리스크 평가

```
① CP팀: view-master "A·B 평가" 탭에서 각 리스크에 A(1~5), B(1~5) 입력
         → X = xLevelFromMatrix(A, B) 자동 결정
② 엑셀 업로드 시 A/B/X 이미 채워져 있으면 자동 완료 처리
③ Phase 2 완료 확정
```

### Phase 2 완료 조건

```js
case 'assess':
  const scored = RISKS.filter(r => r.a_likelihood && r.b_impact && r.x_level);
  return [
    { label: '전체 리스크 A·B 평가 완료 (X 자동 산출)',
      met: RISKS.length > 0 && scored.length === RISKS.length,
      detail: `${scored.length}/${RISKS.length}건` },
  ];
```

---

## 9. Phase 3 — 통제수준 진단

### view-control-input (팀 담당자)

```
split-view:
좌측: 내 팀 리스크 목록 (X등급 H·VH 우선)
  - 통제수단 입력 상태 배지 (미입력/입력완료)

우측:
  - 리스크 정보 + X등급 표시
  - 현재 통제수단 textarea (필수, 20자 이상)
    placeholder: "현재 시행 중인 통제 활동을 구체적으로 작성하세요.
                  (예: 월 1회 점검 실시, 분기별 교육 이수, 절차서 v2.0 준수 등)"
  - Y 기준 참고 안내 accordion (접혀 있음, 클릭 시 VH~VL 기준 펼쳐짐)
  - [임시저장] [제출하기]
```

### view-control-eval (관리자)

```
split-view:
좌측: 통제수단 입력된 리스크 목록 (미평가 우선)
  - X등급 배지 + Y 평가 상태 배지

우측:
  - 리스크 정보 + X등급
  - 팀 제출 통제수단 내용 (읽기전용 박스)
  - Y 효과성 평가 radio-chip 5개: VL / L / M / H / VH
    (각 chip에 tooltip으로 기준 표시)
  - Y 선택 즉시: Z = zLevelFromMatrix(X, Y) 자동 계산, 배지 미리보기
  - CP팀 평가 의견 textarea (선택)
  - [저장]
```

### Phase 3 완료 조건

```js
case 'control':
  const targetRisks = RISKS.filter(r => r.x_level && r.x_level !== '');
  const inputted    = targetRisks.filter(r => r.controlMeasureStatus === '입력완료');
  const evaluated   = targetRisks.filter(r => r.y_level);
  return [
    { label: '통제수단 입력 완료', met: inputted.length === targetRisks.length && targetRisks.length > 0,
      detail: `${inputted.length}/${targetRisks.length}건` },
    { label: 'CP팀 통제효과성(Y) 평가 완료', met: evaluated.length === targetRisks.length && targetRisks.length > 0,
      detail: `${evaluated.length}/${targetRisks.length}건` },
  ];
```

---

## 10. Phase 4 — 잔여리스크 확정 (view-residual)

**기존 view-selection 완전 대체.**

```
요약 KPI 카드: 전체 N건 / Z계산 완료 M건 / 관리대상(H·VH) K건

자동 선정:
  - [H·VH 자동 선정] — z_level이 H 또는 VH인 건 자동 z_selected=true
  - [상위 N건 선정] select

Z 정렬 테이블:
  - 컬럼: 선정 | 순위 | 영역 | 법규명 | 리스크내용 | 관련조직 | X | Y | Z | Z등급
  - 정렬: VH → H → M → L → VL 순서 (레벨 정렬)
  - 필터 탭: 전체 / VH / H / M / L·VL

하단:
  [선정 저장] [📧 사업팀 확인 요청 발송] [✅ 최종 확정]
```

레벨 정렬 순서:
```js
const LEVEL_ORDER = { VH:5, H:4, M:3, L:2, VL:1, '':0 };
risks.sort((a, b) => (LEVEL_ORDER[b.z_level]||0) - (LEVEL_ORDER[a.z_level]||0));
```

### Phase 4 완료 조건

```js
case 'residual':
  return [
    { label: 'Z 계산 완료 (X·Y 모두 입력됨)', met: RISKS.filter(r=>r.z_level).length > 0,
      detail: `${RISKS.filter(r=>r.z_level).length}건` },
    { label: '잔여리스크 관리 대상 선정', met: RISKS.filter(r=>r.z_selected).length > 0,
      detail: `${RISKS.filter(r=>r.z_selected).length}건 선정` },
    { label: '사업팀 확인 요청 발송', key:'residualConfirmSent', manual:true },
  ];
```

---

## 11. Phase 5 — 개선방안 도출·확정

**phase5 view에 인터랙티브 패널 없음. 독립 페이지 2개로 완전 분리.**  
`view-phase5` 내 `#improve-interactive` div 완전 제거.

### view-improve-submit (팀 담당자)

```
split-view:
좌측: 내 팀 잔여리스크 관리 대상 목록 (z_selected=true && relatedOrg에 appTeam 포함)
  - Z등급 배지 + 개선방안 상태 배지

우측:
  - 리스크 정보 (X/Y/Z 등급)
  - 현재 통제수단 내용 (controlMeasure) — 읽기전용 참고
  - 개선방안 textarea (필수, 10자 이상)
  - 목표 완료일 date input (선택)
  - [임시저장] [제출하기]
```

### view-improve-confirm (관리자)

```
요약: 전체 대상 N건 / 제출완료 M건 / 확정 K건

split-view:
좌측: 제출완료 목록 (미확정 우선)
  - 영역, 법규명, 리스크내용, Z등급, 팀

우측:
  - 리스크 정보 + 팀 제출 개선방안 (읽기전용)
  - CP팀 검토 의견 textarea
  - [✅ 확정] [↩ 반려 (재제출 요구)]
```

### Phase 5 완료 조건

```js
case 'improve':
  const targets   = RISKS.filter(r => r.z_selected);
  const confirmed = targets.filter(r => r.improvePlanStatus === '확정');
  return [
    { label: '전체 개선방안 제출 완료',
      met: targets.length > 0 && targets.filter(r=>r.improvePlanStatus!=='미제출').length===targets.length,
      detail: `${targets.filter(r=>r.improvePlanStatus!=='미제출').length}/${targets.length}건` },
    { label: '전체 개선방안 CP팀 확정',
      met: confirmed.length === targets.length && targets.length > 0,
      detail: `${confirmed.length}/${targets.length}건` },
  ];
```

---

## 12. 기존 코드 삭제/수정 목록

### 삭제

```
- view-selection HTML 전체 삭제
- NAV_ADMIN에서 {id:'selection'...} 삭제
- PAGE_TITLES에서 selection 삭제
- navigateTo()에서 renderSelection() 호출 삭제
- renderSelection(), autoSelectTopRisks(), resetSelection(), saveSelection(), toggleSelRisk() 함수 삭제
- view-phase4의 <div id="residual-interactive"> 삭제
- view-phase5의 <div id="improve-interactive"> 삭제
- xLevel(n), zLevel(n), yNumeric(level) 숫자 기반 함수 삭제
- scoreStyle(s), scClass(v), scColor(v), heatColor(v) 수치 기반 함수 → 레벨 기반으로 교체
- 기존 4×4 히트맵 → 5×5 재작성
```

### 수정

```
- renderMaster() tbody: X/Y/Z 레벨 배지 + 영역/법규명 컬럼 추가
- openRiskEditModal(): A/B 드롭다운(1~5) + Y 드롭다운(VL~VH) + 실시간 X·Z 미리보기
- confirmEditRisk(): 신규 필드 저장 (domain, lawName, riskType, relatedClause, relatedOrg, a_likelihood, b_impact, x_level, y_level, z_level)
- renderHeatmap(): 5×5 + xLevelFromMatrix() 기반
- _getActivePhase() Phase 카드의 nav 주소를 신규 독립 페이지로 업데이트
- _checkPhaseConditions(): 7단계 전체 기준 업데이트
- _doExportRisks(): 신규 컬럼 포함하여 실제 엑셀과 동일 형식으로 내보내기
- _doDownloadTemplate(): 동일
- SAMPLE_RISKS: 신규 필드(domain, lawName, a_likelihood 1~5, b_impact 1~5, x_level, y_level, z_level) 반영
```

---

## 13. 스코어링 상수 코드 블록 (최종 정의)

```js
/* ═══════════════════════════════════════════════════════════
   📊  리스크 스코어링 상수 — 실제 운영 기준
   모든 레벨 계산은 이 상수를 참조한다.
═══════════════════════════════════════════════════════════ */

// X 고유리스크 매트릭스 (A=발생가능성, B=영향심각성, 각 1~5)
const X_MATRIX = {
  5: { 1:'M',  2:'H',  3:'VH', 4:'VH', 5:'VH' },
  4: { 1:'M',  2:'H',  3:'H',  4:'VH', 5:'VH' },
  3: { 1:'L',  2:'M',  3:'H',  4:'H',  5:'VH' },
  2: { 1:'VL', 2:'L',  3:'M',  4:'H',  5:'H'  },
  1: { 1:'VL', 2:'VL', 3:'L',  4:'M',  5:'H'  },
};

// Z 잔여리스크 매트릭스 (X레벨, Y레벨)
const Z_MATRIX = {
  VH: { VH:'L',  H:'M',  M:'H',  L:'VH', VL:'VH' },
  H:  { VH:'VL', H:'L',  M:'M',  L:'H',  VL:'H'  },
  M:  { VH:'VL', H:'VL', M:'L',  L:'M',  VL:'M'  },
  L:  { VH:'VL', H:'VL', M:'VL', L:'L',  VL:'L'  },
  VL: { VH:'VL', H:'VL', M:'VL', L:'VL', VL:'VL' },
};

// 레벨 정렬 순서 (숫자 클수록 위험)
const LEVEL_ORDER = { VH:5, H:4, M:3, L:2, VL:1, '':0 };

// 레벨별 색상
const LEVEL_COLOR = {
  VH: { text:'#EF4444', bg:'#FEF2F2', border:'#FECACA' },
  H:  { text:'#F97316', bg:'#FFF7ED', border:'#FED7AA' },
  M:  { text:'#F59E0B', bg:'#FFFBEB', border:'#FDE68A' },
  L:  { text:'#84CC16', bg:'#F7FEE7', border:'#D9F99D' },
  VL: { text:'#22C55E', bg:'#F0FDF4', border:'#BBF7D0' },
};

function xLevelFromMatrix(a, b) { return X_MATRIX[a]?.[b] || ''; }
function zLevelFromMatrix(xLv, yLv) { return Z_MATRIX[xLv]?.[yLv] || ''; }
function levelBadgeHtml(lv, label) {
  if (!lv) return '<span style="color:var(--muted)">—</span>';
  const c = LEVEL_COLOR[lv] || LEVEL_COLOR.VL;
  const txt = label || lv;
  return `<span style="display:inline-flex;align-items:center;padding:2px 8px;
    border-radius:99px;font-size:11px;font-weight:800;
    background:${c.bg};color:${c.text};border:1px solid ${c.border}">${txt}</span>`;
}
```

---

## 14. 도메인 영역 상수 (드롭다운·필터 공통 사용)

```js
const DOMAINS = [
  'HR','정보보호','IP(지재권)','영업비밀',
  'M&A','IR','회계세무','부동산','기타',
  '안전보건','환경','공정거래','반독점','반부패'
];
```

리스크 추가/편집 모달, 마스터 필터, 대시보드 카테고리 분류에 공통 사용.

---

## 15. CONFIG Flow URL

```js
FLOW_ASSIGN_NOTIFY_URL:    '',  // Phase 1: 배당 확인 메일
FLOW_IDENTIFY_OPINION_URL: '',  // Phase 1: 팀 의견 제출 알림
FLOW_CONTROL_INPUT_URL:    '',  // Phase 3: 통제수단 입력 완료 알림
FLOW_RESIDUAL_CONFIRM_URL: '',  // Phase 4: 잔여리스크 사업팀 확인
FLOW_IMPROVE_SUBMIT_URL:   '',  // Phase 5: 개선방안 제출 알림
FLOW_IMPROVE_CONFIRM_URL:  '',  // Phase 5: 개선방안 확정 알림

// 아카이브·이력 관련 (3-7절 참조)
LIST_RISK_HISTORY:  'RiskMasterHistory',  // SP: 스냅샷 이력 List
SP_ARCHIVE_FOLDER:  'Documents/RiskMaster_Archive',  // SP 아카이브 폴더
MAX_LOCAL_SNAPSHOTS: 10,  // localStorage 최대 스냅샷 보관 수
```

---

## 16. 작업 순서 (권장)

1. **스코어링 상수** 정의 (섹션 13) + `DOMAINS` 상수 (섹션 14)
2. **기존 수치 기반 함수 폐기** + `levelBadgeHtml()` 교체 (섹션 12)
3. **RISK 데이터 모델** 재정의 (섹션 2) + SAMPLE_RISKS 업데이트 (changeLog 포함)
4. **CONFIG** Flow URL + 아카이브 상수 추가 (섹션 15)
5. **엑셀 임포트 파싱 로직** 전면 재작성 (섹션 3)
6. **아카이브·백업 함수** 구현 (섹션 3-7): `saveArchiveSnapshot`, `restoreSnapshot`, `loadPreviousYear`, `_pruneSnapshots`
7. **view-selection 삭제** + view-phase4/5 패널 삭제 (섹션 12)
8. **NAV_ADMIN / NAV_USER / PAGE_TITLES** 교체 (섹션 5)
9. **navigateTo()** 라우팅 추가 (섹션 5-4)
10. **view-master 탭 3개 구조** 추가 (섹션 6: 리스크 목록 / A·B 평가 / 변경 이력·복원)
11. **히트맵 5×5** 재작성 (섹션 1-7)
12. **7개 신규 독립 페이지** HTML + 함수 구현 (섹션 4, 7~11)
13. **_checkPhaseConditions()** 7단계 업데이트 (섹션 7~11)
14. **렌더링 함수 업데이트** (renderMaster, renderResidualView 등)
15. **엑셀 익스포트** 컬럼 업데이트 (섹션 3-5)
16. **view-archive** 아카이브 섹션 추가 (연도별 스냅샷 목록 + 불러오기 버튼)
17. JS 문법 검사 + 전체 동작 확인

---

## 17. 검증 체크리스트

- [ ] 기존 엑셀(`전체 Risk Profile_통제수준 진단결과.xlsx`) 업로드 시 domain, lawName, riskContent, A/B/X/Y/Z 모두 파싱되는지 확인
- [ ] A·B만 있고 X가 없는 엑셀 → X_MATRIX로 자동 계산 확인
- [ ] Y만 있고 Z가 없는 엑셀 → Z_MATRIX로 자동 계산 확인
- [ ] A/B/Y/X/Z 모두 있는 엑셀 → 엑셀 값 그대로 사용 확인
- [ ] 업로드 후 587건 전체 임포트, 오류 건 별도 표시 확인
- [ ] 업로드 완료 시 자동 스냅샷 저장 확인
- [ ] 이전 연도 불러오기 → 새 연도 초안 생성, 평가값 유지·진행 필드 초기화 확인
- [ ] view-master "변경 이력·복원" 탭에서 스냅샷 목록 표시 확인
- [ ] 특정 스냅샷으로 복원 → 복원 전 자동 백업 생성 후 복원 확인
- [ ] 팀 수정 요청 → CP팀 수용 시 changeLog 기록 확인
- [ ] view-archive 연도별 아카이브 목록 표시 + 엑셀 다운로드 확인
- [ ] localStorage 스냅샷 10개 초과 시 자동 정리(phase_complete 보존) 확인
- [ ] 히트맵 5×5 정상 렌더링, 셀 클릭 시 리스크 목록 표시 확인
- [ ] 레벨 배지 VL/L/M/H/VH 색상 구분 확인
- [ ] view-selection 완전 삭제, 사이드바에서 미노출 확인
- [ ] Phase 4에서 Z 레벨 내림차순(VH→VL) 정렬 확인
- [ ] 팀 담당자 7개 독립 페이지 정상 접근 확인
- [ ] 관리자 전용 페이지 팀 담당자 접근 차단 확인
- [ ] OFFLINE_MODE=true에서 전체 기능 동작 확인

---

## 18. 대시보드(view-dash) KPI 카드 재설계

### 18-1. 카드 구성 (4개)

기존 KPI 카드(관리 리스크 / 제출 완료 / 전사 달성율 / 평가 대기)를 아래 4개로 교체한다.

```
카드 1        카드 2          카드 3        카드 4
━━━━━━━━━━━  ━━━━━━━━━━━━━  ━━━━━━━━━━━  ━━━━━━━━━━━━━━━
📚 법령·리스크  🎯 잔여리스크    📊 전사 달성율   ⚡ 현재 업무
━━━━━━━━━━━  ━━━━━━━━━━━━━  ━━━━━━━━━━━  ━━━━━━━━━━━━━━━
103개 법령    40건             73%          28건 대기
587개 리스크  확정 관리 대상                ─────────────
(2025년 기준)                              이행 평가  12건
                                           이의 검토   3건
                                           방안 확정   8건
                                           의견 검토   5건
```

### 18-2. 카드별 상세 스펙

#### 카드 1 — 법령·리스크 현황

```js
// 표시 데이터
const lawCount  = new Set(RISKS.map(r => r.lawName).filter(Boolean)).size;
const riskCount = RISKS.length;
const yearLabel = curYear + '년 기준';

// UI
상단: 📚 아이콘 + "법령·리스크"
메인 숫자 1: lawCount  → "103개 법령"
메인 숫자 2: riskCount → "587개 리스크"
하단 서브: yearLabel
클릭 시: navigateTo('master') 이동
```

#### 카드 2 — 잔여리스크 확정 대상

```js
// 표시 데이터
const residualCount = RISKS.filter(r => r.z_selected).length;
// z_selected 아직 없는 경우 (Phase 4 이전) fallback:
const fallbackCount = RISKS.filter(r => ['H','VH'].includes(r.x_level)).length;
const displayCount  = residualCount > 0 ? residualCount : fallbackCount;
const isConfirmed   = residualCount > 0;  // Phase 4 완료 여부

// UI
상단: 🎯 아이콘 + "잔여리스크 관리 대상"
메인 숫자: displayCount → "40건"
서브: isConfirmed
  ? "Phase 4 확정 완료"
  : `고유리스크 H·VH ${fallbackCount}건 (잠정)`
클릭 시: navigateTo('residual') 이동
```

#### 카드 3 — 전사 달성율

```js
// 기존 달성율 계산 로직 유지
const achRisks   = RISKS.filter(r => r.achieve != null);
const overallAch = achRisks.length
  ? Math.round(achRisks.reduce((s,r) => s + r.achieve, 0) / achRisks.length)
  : 0;
const achColor   = overallAch >= 80 ? 'var(--green)' : overallAch >= 60 ? 'var(--amber)' : 'var(--red)';
const achLabel   = overallAch >= 80 ? '양호' : overallAch >= 60 ? '주의' : '위험';

// UI
상단: 📊 아이콘 + "전사 달성율"
메인 숫자: overallAch + "%" (색상: achColor)
서브: achLabel + " · 최고: {best팀} {best}%"
클릭 시: navigateTo('teams') 이동
```

#### 카드 4 — 현재 업무 대기

```js
// 표시 데이터 — 관리자 기준
const pendingItems = [
  {
    label: '이행 평가 대기',
    count: RISKS.filter(r => r.submitStatus === '제출완료').length,
    nav: 'eval',
    color: 'var(--amber)',
  },
  {
    label: '이의신청 검토',
    count: RISKS.filter(r => r._revisionRequested && r._revisionStatus === 'pending').length,
    nav: 'monthly',
    color: 'var(--red)',
  },
  {
    label: '개선방안 확정 대기',
    count: RISKS.filter(r => r.improvePlanStatus === '제출완료').length,
    nav: 'improve-confirm',
    color: '#8B5CF6',
  },
  {
    label: '식별 의견 검토',
    count: RISKS.filter(r => r.identifyConfirmStatus === 'pending'
                          && r.identifyOpinionStatus === '제출완료'
                          && r.identifyOpinionType !== '이의없음').length,
    nav: 'identify-confirm',
    color: 'var(--accent)',
  },
];
const totalPending = pendingItems.reduce((s, it) => s + it.count, 0);

// UI
상단: ⚡ 아이콘 + "현재 업무"
메인 숫자: totalPending + "건 대기"
세부 목록: pendingItems 중 count > 0인 항목만 표시
  → 각 항목 클릭 시 해당 nav로 이동
  → count = 0인 항목은 회색으로 표시하거나 숨김
하단: count === 0이면 "✓ 처리 대기 없음" 초록 텍스트
```

### 18-3. 렌더링 함수 — `initDash()` 수정

기존 KPI 카드 4개(관리 리스크 / 제출 완료 / 전사 달성율 / 평가 대기) 렌더링 로직을 **완전히 교체**한다.

```js
function initDash() {
  _renderKpiCard1_LawRisk();
  _renderKpiCard2_Residual();
  _renderKpiCard3_Achieve();
  _renderKpiCard4_PendingWork();

  renderDashSummary();
  renderMissingAlerts();
  renderHeatmap();
  setTimeout(() => { drawChart('trend-canvas', TREND, TKEYS, TCOLORS, true); runCounters(); }, 80);
}
```

### 18-4. HTML KPI 카드 구조 변경

기존 `<div class="kpi-grid">` 내 4개 카드를 교체한다.

```html
<div class="kpi-grid">

  <!-- 카드 1: 법령·리스크 현황 -->
  <div class="kpi-card" style="cursor:pointer" onclick="navigateTo('master')">
    <div class="kpi-top"><span class="kpi-icon">📚</span></div>
    <div style="display:flex;flex-direction:column;gap:2px">
      <div style="display:flex;align-items:baseline;gap:6px">
        <span class="kpi-num" id="kpi-law-count" style="font-size:28px">0</span>
        <span style="font-size:13px;color:var(--muted);font-weight:600">개 법령</span>
      </div>
      <div style="display:flex;align-items:baseline;gap:6px">
        <span class="kpi-num" id="kpi-risk-count" style="font-size:28px">0</span>
        <span style="font-size:13px;color:var(--muted);font-weight:600">개 리스크</span>
      </div>
    </div>
    <div class="kpi-label">법령·리스크 현황</div>
    <div class="kpi-sub" id="kpi-year-label">2025년 기준</div>
  </div>

  <!-- 카드 2: 잔여리스크 확정 대상 -->
  <div class="kpi-card" style="animation-delay:80ms;cursor:pointer" onclick="navigateTo('residual')">
    <div class="kpi-top"><span class="kpi-icon">🎯</span></div>
    <div class="kpi-num" id="kpi-residual">0</div>
    <div class="kpi-label">잔여리스크 관리 대상</div>
    <div class="kpi-sub" id="kpi-residual-sub">Phase 4 확정 기준</div>
  </div>

  <!-- 카드 3: 전사 달성율 -->
  <div class="kpi-card" style="animation-delay:160ms;cursor:pointer" onclick="navigateTo('teams')">
    <div class="kpi-top">
      <span class="kpi-icon">📊</span>
      <span class="badge-up" id="kpi-ach-badge">—</span>
    </div>
    <div class="kpi-num" data-pct="1" id="kpi-ach">0</div>
    <div class="kpi-label">전사 달성율</div>
    <div class="kpi-sub" id="kpi-ach-sub">팀별 달성율</div>
  </div>

  <!-- 카드 4: 현재 업무 대기 -->
  <div class="kpi-card" style="animation-delay:240ms;cursor:pointer" id="kpi-pending-card">
    <div class="kpi-top"><span class="kpi-icon">⚡</span></div>
    <div class="kpi-num" id="kpi-pending-total">0</div>
    <div class="kpi-label">현재 업무 대기</div>
    <!-- 세부 항목 목록 (JS로 동적 생성) -->
    <div id="kpi-pending-detail"
         style="margin-top:8px;display:flex;flex-direction:column;gap:4px;width:100%"></div>
  </div>

</div>
```

### 18-5. 카드 4 세부 항목 렌더링

```js
function _renderKpiCard4_PendingWork() {
  const items = [
    { label:'이행 평가',   count: RISKS.filter(r=>r.submitStatus==='제출완료').length,           nav:'eval',             color:'var(--amber)' },
    { label:'이의 검토',   count: RISKS.filter(r=>r._revisionStatus==='pending').length,          nav:'monthly',          color:'var(--red)'   },
    { label:'방안 확정',   count: RISKS.filter(r=>r.improvePlanStatus==='제출완료').length,       nav:'improve-confirm',  color:'var(--purple)'},
    { label:'의견 검토',   count: RISKS.filter(r=>r.identifyConfirmStatus==='pending'
                                              && r.identifyOpinionStatus==='제출완료'
                                              && r.identifyOpinionType!=='이의없음').length,      nav:'identify-confirm', color:'var(--accent)'},
  ];

  const total = items.reduce((s, it) => s + it.count, 0);
  const totalEl = document.getElementById('kpi-pending-total');
  if (totalEl) { totalEl.textContent = total; totalEl.style.color = total > 0 ? 'var(--red)' : 'var(--green)'; }

  const detailEl = document.getElementById('kpi-pending-detail');
  if (!detailEl) return;

  if (total === 0) {
    detailEl.innerHTML = `<div style="font-size:12px;color:var(--green);font-weight:700;text-align:center">✓ 처리 대기 없음</div>`;
    return;
  }

  detailEl.innerHTML = items.map(it => `
    <div onclick="event.stopPropagation();navigateTo('${it.nav}')"
         style="display:flex;justify-content:space-between;align-items:center;
                padding:3px 0;cursor:pointer;
                ${it.count === 0 ? 'opacity:.35;pointer-events:none' : ''}">
      <span style="font-size:11px;color:var(--muted)">${it.label}</span>
      <span style="font-size:12px;font-weight:800;color:${it.count > 0 ? it.color : 'var(--muted)'}">
        ${it.count}건
      </span>
    </div>`).join('');
}
```

### 18-6. 기존 KPI 카드 관련 삭제·정리

```
삭제:
- 기존 id="kpi-total", id="kpi-submitted", id="kpi-eval" 요소 및 관련 JS
- initDash() 내 기존 setKpi() 호출 4개
- kpi-submitted-badge, kpi-eval-badge 관련 코드

수정:
- runCounters(): kpi-ach 카운터 애니메이션 유지
                  kpi-law-count, kpi-risk-count, kpi-residual, kpi-pending-total 카운터 추가
- updateBadges(): 변경 없음 (사이드바 배지는 기존 로직 유지)
```

### 18-7. 검증 항목 추가

- [ ] 법령수: `new Set(RISKS.map(r => r.lawName))` 고유값 카운트 정확한지 확인
- [ ] 잔여리스크: Phase 4 이전에는 H·VH 건수 fallback 표시 확인
- [ ] 카드 4 각 항목 클릭 시 해당 페이지로 정확히 이동하는지 확인
- [ ] 대기 건수 0일 때 "✓ 처리 대기 없음" 초록 텍스트 표시 확인
- [ ] 대기 건수 0인 항목은 회색 + 클릭 비활성화 확인
