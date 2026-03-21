# Compliance RiskOps — 운영 전환 계획서

> **작성일:** 2026-03-18 (최종 업데이트: 2026-03-21)
> **현황:** 단일 `index.html` (v10.2, 6,456줄) — 리팩토링 완료, monkey-patch 18→6개
> **목표:** SK Chemical 사내 실운영 시스템 전환
> **배포 방향:** Vercel(프론트) + SharePoint/Power Automate(데이터) 또는 경량 백엔드 추가

---

## Part 1. 현재 구현된 기능 인벤토리

> ✅ 완성 | ⚠️ UI만 있고 실제 동작 없음 | ❌ 미구현·하드코딩

---

### 1-1. 인증 / 접근제어

| 기능 | 상태 | 비고 |
|------|------|------|
| 역할 선택 UI (관리자 / 팀 담당자) | ⚠️ | 실제 인증 없이 클릭만으로 통과 |
| 팀 선택 UI (5개 팀 버튼) | ⚠️ | 선택값만 저장, 검증 없음 |
| 로그아웃 | ⚠️ | 상태 초기화만, 세션 없음 |
| 역할별 메뉴 분기 (admin/user) | ✅ | NAV_ADMIN / NAV_USER 분기 완성 |
| 실제 사용자 인증 (SSO/AD) | ❌ | 미구현 |

---

### 1-2. UI / 레이아웃

| 기능 | 상태 | 비고 |
|------|------|------|
| PC 사이드바 (접기/펼치기) | ✅ | |
| 모바일 하단 탭 (Bottom Nav) | ✅ | |
| 모바일 드로어 (햄버거 메뉴) | ✅ | |
| 모바일 Bottom Sheet (리스크 상세) | ✅ | |
| 반응형 분기점 (768px) | ✅ | |
| iOS Safe Area 대응 | ✅ | |
| 전역 JS 에러 방어 | ✅ | |
| CDN 라이브러리 Lazy Load | ✅ | XLSX, PptxGenJS |
| 페이지 전환 애니메이션 | ✅ | fadeIn / fadeUp / slideUp |
| 차트 Y축 동적 범위 | ✅ | 월별/팀별 차트 자동 범위 + 연초 경계선 |

---

### 1-3. 전사 대시보드 (관리자 전용)

| 기능 | 상태 | 비고 |
|------|------|------|
| KPI 카드 4종 (관리 리스크 / 제출완료 / 달성율 / 평가대기) | ✅ | 실데이터 연동 시 자동 반영 |
| KPI 카운터 애니메이션 | ✅ | |
| KPI 드릴다운 모달 (카드 클릭) | ✅ | showKpiDrilldown() |
| 즉각조치 배너 (Phase-aware, 월별 알림 분기) | ✅ | Phase 1~5 기간별 맞춤 알림 (v10.2) |
| KPI 카드 Phase 진행 뱃지 | ✅ | "사이클 N/5단계 확정" 클릭→미션탭 (v10.2) |
| 팀별 이행 달성율 바 차트 | ⚠️ | TEAMS_DATA 하드코딩, 실데이터 연동 필요 |
| 월별 달성율 추이 꺾은선 차트 (Canvas) | ✅ | Y축 동적 범위 + 연초 경계선 표시 |
| 리스크 히트맵 (영향도×가능성, 셀 클릭 드릴다운) | ✅ | |
| 미제출 현황 카드 (팀별 독촉 메일) | ⚠️ | F-04 미연동 |
| 연도/월 셀렉터 | ✅ | setYear()/setMonth() 데이터 갱신 연동 완료 |
| 위험평가 Full Cycle 사이클 카드 | ✅ | 현재 월 자동 감지, 단계 클릭 전환 |
| 단계별 설명 + 할 일 액션 카드 (5단계) | ✅ | previewPhase() 클릭 미리보기 포함 |
| Phase 체크리스트 + 완료 확정 버튼 | ✅ | 5단계별 완료 조건 검증 + D-day (v10.2) |
| 단계별 미션 전용 탭 (Phase Control) | ✅ | 히어로 + 아코디언 + 완료 확정/해제 (v10.2) |
| Phase 간 의존성 경고 | ✅ | 이전 단계 미완료 시 진입 경고 토스트 (v10.2) |
| 연도 전환 자동 아카이빙 | ✅ | 이전 연도 Phase 상태 자동 보관 (v10.2) |

---

### 1-4A. 이행현황 검토 (관리자, v10.2)

| 기능 | 상태 | 비고 |
|------|------|------|
| 월간입력 + 평가 통합 화면 | ✅ | 기존 eval 탭 제거, monthly 탭에서 통합 검토 |
| 상태 필터 (전체/미제출/제출완료/평가완료) | ✅ | 좌측 패널 필터 탭 |
| 미제출 건: 독촉 메일 버튼 | ✅ | |
| 제출완료 건: 팀 제출내용 읽기 + 평가 슬라이더 + AI 의견 | ✅ | |
| 임시저장 건: 작성 중 내용 미리보기 | ✅ | |
| 평가완료 건: 결과 표시 (달성율 + 의견) | ✅ | |
| 이의신청 검토: 사유 확인 + 수용(점수 조정)/기각(점수 유지) | ✅ | v10.2 |
| 퀵스코어 버튼 % 범위 표시 | ✅ | 미달성 0-40% / 부분 40-80% / 달성 80-90% / 우수 90-100% |

---

### 1-4B. 이행현황 입력 (팀 담당자, v10.2 재설계)

| 기능 | 상태 | 비고 |
|------|------|------|
| 팀 리스크 배정 확인 프로세스 | ✅ | 개별 리스크 확인/이의제기 → 전체 확인 후 입력 활성 |
| 배정 확인 화면: 4단계 프로세스 안내 | ✅ | 번호 뱃지 + 단계별 설명 |
| 배정 확인: 개선과제 상세 + [확인]/[이의제기] | ✅ | |
| 좌측 리스크 목록 패널 (전체 리스크) | ✅ | 평가완료 포함 전체 표시 |
| 미제출: 바로 입력 폼 | ✅ | |
| 임시저장/제출완료: 미리보기 + [수정]/[재제출] | ✅ | |
| 평가완료: 미리보기 + 평가결과 + [이의신청+수정]/[추가제출] | ✅ | |
| 이의신청 + 내용 수정 동시 제출 (submitWithRevision) | ✅ | |
| 입력 가이드 배너 (3단계 + D-day) | ✅ | user 모드 전용 |
| 좌측 리스크 목록 패널 | ⚠️ | 팀 필터링은 됨, 실데이터 없음 |
| 우측 입력 폼 (이행상태 칩 선택) | ⚠️ | UI 완성, 저장 안됨 |
| 이행 내용 텍스트 입력 | ⚠️ | |
| 증빙 파일 드롭존 UI | ⚠️ | 파일 선택 가능, 실제 업로드 없음 |
| 특이사항 텍스트 입력 | ⚠️ | |
| 임시저장 버튼 | ⚠️ | 동작 없음 |
| 제출하기 버튼 | ⚠️ | 버튼 색상만 변경, 저장 없음 |
| 이의신청 버튼 | ⚠️ | alert만 |
| 모바일 Bottom Sheet 연동 | ✅ | |

---

### 1-5. 이행 달성율 평가 (관리자) — v10.2에서 이행현황 검토(1-4A)로 통합

> ⚠️ 별도 eval 탭은 v10.2에서 제거됨. 관리자 '이행현황 검토' 탭에서 통합 처리.

| 기능 | 상태 | 비고 |
|------|------|------|
| 평가 대기 목록 (제출완료 건 필터) | ✅ | 이행현황 검토 탭 상태 필터로 통합 |
| 팀 제출 내용 표시 박스 | ✅ | 이행상태/이행내용/특이사항/첨부파일 섹션 분리 |
| 달성율 슬라이더 (0~100%) | ✅ | 색상 연동 |
| 퀵 스코어 버튼 (미달성/부분/달성/우수) | ✅ | |
| 평가 완료 버튼 | ⚠️ | 버튼 비활성화만, 저장 없음 |
| 수정 요청 버튼 | ⚠️ | alert만 |

---

### 1-6. 팀 이행현황 (관리자)

| 기능 | 상태 | 비고 |
|------|------|------|
| 팀별 스탯 카드 (5팀) | ⚠️ | TEAMS_DATA 하드코딩 |
| 팀 카드 클릭 시 차트 하이라이트 | ✅ | |
| 전체 팀 추이 꺾은선 차트 (Canvas) | ✅ | hover 툴팁 + Y축 동적 범위 + 연초 경계선 |
| 팀별 필터 탭 | ✅ | |
| 차트 범례 | ✅ | |

---

### 1-7. 리스크 Master (관리자)

| 기능 | 상태 | 비고 |
|------|------|------|
| 리스크 목록 테이블 | ⚠️ | RISKS 배열 하드코딩 |
| 검색 (리스크명) | ✅ | masterSearch() |
| 팀별 필터 (셀렉터 + 탭) | ✅ | |
| Excel 양식 다운로드 | ✅ | SheetJS, 샘플 포함 |
| Excel 파일 업로드 (드래그&드롭) | ✅ | |
| 2단계 업로드 모달 (파싱→선정) | ✅ | |
| 위험도 자동 계산 (영향×가능성) | ✅ | |
| 고위험 자동 선정 (score≥15) | ✅ | |
| 전체/전체해제/상위N개 선정 | ✅ | |
| 선정 완료 후 RISKS 배열 갱신 | ✅ | (메모리 내, 새로고침 시 소멸) |
| 리스크 추가 (수동) | ✅ | openAddRiskModal() 커스텀 모달 구현 |
| 리스크 수정 (인라인) | ✅ | openRiskEditModal() 모달 + confirmEditRisk() |
| Owner 수정 | ✅ | 리스크 수정 모달에 포함 |
| 개선과제 열 표시 | ✅ | 테이블에 15자 미리보기 + 클릭→수정 모달 (v10.2) |
| Phase 3 통제수단 힌트 | ✅ | 3~4월 수정 모달에 입력 안내 + 필수뱃지 (v10.2) |

---

### 1-8. 점검대상 리스크 선정 (관리자)

| 기능 | 상태 | 비고 |
|------|------|------|
| 위험도 순 정렬 리스트 | ✅ | |
| 상위 N건 자동 선정 셀렉터 | ✅ | |
| 수동 체크박스 선정 | ✅ | |
| 선정 저장 버튼 | ⚠️ | 메모리에만 반영, 새로고침 시 소멸 |
| 위험도 범례 표시 | ✅ | |
| 개선과제 상태 열 (Phase 3) | ✅ | 입력됨/미입력 뱃지 + 클릭→퀵에딧 모달 (v10.2) |
| 개선과제 퀵에딧 모달 | ✅ | 리스크 클릭→개선과제 바로 입력 (Master 이동 불필요, v10.2) |
| Phase 3 입력 현황 배너 | ✅ | 미입력 건수 요약 + Master 이동 버튼 (v10.2) |

---

### 1-9. 보고서 생성 (관리자)

| 기능 | 상태 | 비고 |
|------|------|------|
| 월간/연간 탭 전환 | ✅ | |
| 월간 보고서 설정 (연도/월/총평) | ✅ | |
| 월간 PPT 생성 (PptxGenJS) | ✅ | Lazy Load, 1슬라이드 |
| 월간 PPT 미리보기 (실시간 HTML) | ✅ | 설정 변경 시 자동 갱신 |
| 연간 보고서 설정 (연도/총평) | ✅ | |
| 연간 PPT 생성 (2슬라이드) | ✅ | 종합현황 + 팀별 분석 |
| 연간 PPT 미리보기 | ✅ | |
| PPT 내 실데이터 반영 | ⚠️ | 하드코딩된 수치 삽입 |

---

### 1-10. 내 팀 이행현황 (팀 담당자)

| 기능 | 상태 | 비고 |
|------|------|------|
| KPI 카드 3종 (담당 리스크 / 달성율 / 미입력) | ⚠️ | 하드코딩 |
| 월별 달성율 추이 차트 | ⚠️ | |
| 리스크별 현황 목록 | ⚠️ | |

---

### 1-10B. 평가결과·이력 (팀 담당자, v10.2 신설)

| 기능 | 상태 | 비고 |
|------|------|------|
| split-view: 좌측 리스크 목록 + 우측 상세 | ✅ | 전체 리스크 (상태+달성율+이의 표시) |
| 개선과제 정보 표시 | ✅ | 과제 내용 + 위험도 |
| 이번달 제출 내용 표시 | ✅ | 이행 내용 + 특이사항 |
| 평가 결과 (달성율 프로그레스바 + 의견) | ✅ | 양호/주의/미달 뱃지 |
| 이의신청 버튼 + 상태 표시 (검토 중/수용/기각) | ✅ | |
| 올해 월별 이력 (1~현재월, 아코디언 펼침) | ✅ | localStorage 기반, 클릭 시 제출 내용 표시 |
| 미제출/임시저장 건: "입력 탭에서 먼저 제출" 안내 | ✅ | |

---

### 1-11. 아카이브 (신규 — v10)

| 기능 | 상태 | 비고 |
|------|------|------|
| 아카이브 탭 (사이드바·모바일 바텀탭) | ✅ | NAV_ADMIN / NAV_USER / BNAV 모두 추가 |
| 연도별 자료 보관 (파일 목록 렌더링) | ✅ | 샘플 데이터 5건 |
| 연도 필터 (전체 / 2026 / 2025 / 2024) | ✅ | archiveFilterYear() |
| 파일 업로드 UI (xlsx·pptx·pdf) | ✅ | handleArchiveUpload() |
| 파일 목록 삭제 | ✅ | archiveDeleteFile() — 목록만 제거, SP 원본 유지 |
| SharePoint 문서라이브러리 실연동 | ❌ | F-08 Power Automate Flow 필요 |
| 신규·개정 법령 인사이트 알리미 | ✅ | 샘플 데이터 5건, 카테고리 필터 |
| 법령 카테고리 필터 (공정거래/개인정보/SHE/IP/기타) | ✅ | lawFilter() |
| 법령 긴급도 표시 (즉시검토/모니터링/반영완료) | ✅ | |
| Copilot 에이전트 iframe 임베딩 영역 | ✅ | CONFIG.COPILOT_AGENT_URL 설정 시 활성화 |
| Copilot 미연결 안내 플레이스홀더 | ✅ | |
| 법령 실데이터 수집 (Copilot → SP List 연동) | ❌ | F-09 Power Automate Scheduled Flow 필요 |

---

### 1-11. 공통 기능

| 기능 | 상태 | 비고 |
|------|------|------|
| Excel 내보내기 (현재 RISKS → .xlsx) | ✅ | SheetJS |
| 연도 셀렉터 (2024~2026) | ✅ | setYear() 데이터 갱신 + 차트 재렌더 연동 |
| 데이터 영속성 (새로고침 후 유지) | ⚠️ | localStorage 임시저장 구현, SP 연동 시 완전 영속 |

---

## Part 2. 프로덕션 전환을 위한 구현 로드맵

### Phase 0 — 아키텍처 결정 (착수 전 선결)

- [ ] **데이터 저장소 선택**
  - [ ] 옵션 A: SharePoint List + Power Automate (IT 승인 최소화, 사내 친화적)
  - [ ] 옵션 B: Supabase(PostgreSQL) + Vercel (단독 SaaS형, 별도 IT 협의 필요)
  - [ ] 옵션 C: Azure SQL + Azure Functions (SK 계열사 클라우드 정책 확인)
  - [ ] **→ 결정 후 Phase 1 착수**
- [ ] 파일 첨부 저장소 선택 (SharePoint 문서 라이브러리 vs Azure Blob)
- [ ] 사내 SSO / Azure AD 연동 가능 여부 IT팀 확인
- [ ] Vercel 배포 도메인 정책 확인 (사내 도메인 CNAME 허용 여부)

---

### Phase 1 — 데이터 레이어 구축

#### 1-A. 데이터 모델 설계
- [ ] **RiskMaster** 테이블 설계
  - [ ] 필드: id, year, team, category, title, description, improvementTask, impactScore, likelihoodScore, riskScore, isSelected, owner, createdAt, updatedAt
- [ ] **MonthlyReport** 테이블 설계
  - [ ] 필드: riskId, year, month, team, status (이행완료/이행중/미이행/해당없음), content, specialNote, attachmentUrl, submitAt, submittedBy
- [ ] **Evaluation** 테이블 설계
  - [ ] 필드: reportId, achieveScore, evalComment, evalBy, evalAt, requestRevision (boolean)
- [ ] **User** 테이블 설계
  - [ ] 필드: id, name, email, role (admin/user), team

#### 1-B. SharePoint List 구성 (옵션 A 선택 시)
- [ ] RiskMaster 리스트 생성 및 컬럼 매핑
- [ ] MonthlyReport 리스트 생성
- [ ] Evaluation 리스트 생성
- [ ] 리스트별 권한 설정 (팀 담당자는 자기 팀 데이터만 쓰기 가능)
- [ ] Power Automate — F-01: 월간 입력 제출 시 트리거 Flow 구성 (관리자 알림 + MonthlyReport 저장)
- [ ] Power Automate — F-02: 평가 완료 시 팀 담당자 알림 Flow 구성
- [ ] Power Automate — F-03: 수정 요청 시 팀 담당자 알림 Flow (RevisionRequested 트리거)
- [ ] Power Automate — F-04: 미제출 독촉 메일 자동 발송 Flow (매월 26일, D-5)
- [ ] Power Automate — F-05: 미제출 최종 독촉 Flow (매월 30일, D-1, 관리자 CC)
- [ ] Power Automate — F-06: 임시저장 HTTP 트리거 Flow
- [ ] Power Automate — F-07: 이의신청 알림 Flow (팀→관리자, HTTP 트리거) ← v2.0 신설
- [ ] Power Automate — F-08: 아카이브 파일 업로드 시 SharePoint 문서라이브러리 저장 Flow ← v10 신설
- [ ] Power Automate — F-09: 법령 인사이트 수집 Scheduled Flow (Copilot→SP List, 주 1회) ← v10 신설

#### 1-C. API 레이어 구성
- [ ] 프론트 → 데이터 연동 방식 결정
  - [ ] 옵션 A-1: Power Automate HTTP 트리거 (GET/POST REST)
  - [ ] 옵션 B-1: Vercel API Routes (Node.js, Supabase 클라이언트)
- [ ] API 엔드포인트 목록 정의
  - [ ] GET  `/api/risks?year=&team=`
  - [ ] POST `/api/risks` (리스크 등록)
  - [ ] PUT  `/api/risks/:id`
  - [ ] GET  `/api/reports/monthly?year=&month=&team=`
  - [ ] POST `/api/reports/monthly` (월간 입력 제출 — F-01 연동)
  - [ ] PUT  `/api/reports/monthly/:id` (임시저장 / 수정 — F-06 연동)
  - [ ] POST `/api/reports/monthly/:id/revision` (이의신청 — F-07 연동)
  - [ ] POST `/api/evaluations` (평가 완료)
  - [ ] POST `/api/notifications/nudge` (독촉 메일)
  - [ ] GET  `/api/dashboard?year=&month=`
  - [ ] GET  `/api/archive/files?year=` (아카이브 파일 목록 — F-08 연동)
  - [ ] POST `/api/archive/files` (파일 업로드 → SP 문서라이브러리)
  - [ ] GET  `/api/archive/laws?category=` (법령 인사이트 목록 — F-09 연동)

---

### Phase 2 — 인증 및 권한 관리

- [ ] **로그인 방식 결정**
  - [ ] 옵션 A: Microsoft SSO (MSAL.js) — Azure AD 계정으로 자동 로그인
  - [ ] 옵션 B: 이메일+패스워드 (Supabase Auth / Firebase Auth)
  - [ ] 옵션 C: 사내 발급 토큰 기반 (IT팀과 협의)
- [ ] MSAL.js 연동 구현 (옵션 A)
  - [ ] Azure AD 앱 등록 (App Registration)
  - [ ] 로그인 화면을 SSO 리다이렉트로 대체
  - [ ] 로그인 후 사용자 이메일 기준으로 role/team 자동 매핑
- [ ] 역할별 API 접근 제어
  - [ ] 관리자: 전체 데이터 읽기/쓰기
  - [ ] 팀 담당자: 자기 팀 데이터만 읽기/쓰기
- [ ] 세션 관리 (JWT 토큰 또는 쿠키)
- [ ] 로그아웃 시 토큰 무효화

---

### Phase 3 — 핵심 기능 실동작 구현

#### 3-A. 리스크 Master 관리
- [ ] 리스크 목록 API 연동 (GET 요청으로 실데이터 표시)
- [ ] 연도 셀렉터 변경 시 해당 연도 데이터 재조회
- [x] 리스크 수동 추가 폼 구현 (`openAddRiskModal()` 커스텀 모달)
  - [x] 팀 / 카테고리 / 제목 / 설명 / 개선과제 / 영향도 / 가능성 입력
  - [x] 위험도 자동 계산 및 미리보기
  - [ ] 등록 API 호출 (현재 메모리만, SP 연동 필요)
- [x] 리스크 수정 모달 (`openRiskEditModal()` + `confirmEditRisk()`)
- [ ] 리스크 삭제 (소프트 딜리트)
- [x] Owner 지정/변경 (수정 모달에 포함)
- [ ] Excel 업로드 → API POST 연동 (현재 메모리만 갱신)
- [ ] 점검대상 선정 저장 → API PUT 연동

#### 3-B. 월간 현황 입력
- [ ] 폼 제출 시 API POST 연동
  - [ ] 이행상태 / 이행내용 / 특이사항 포함
- [ ] 임시저장 기능 구현 (PUT, status='draft')
- [ ] 제출 후 상태 즉시 반영 (좌측 목록 status chip 업데이트)
- [ ] 증빙 파일 업로드
  - [ ] 파일 → SharePoint 문서 라이브러리 또는 Azure Blob 업로드
  - [ ] 업로드 후 URL을 MonthlyReport에 저장
  - [ ] 업로드 진행 상태 표시 (프로그레스 바)
  - [ ] 파일 미리보기 / 다운로드 링크
- [ ] 이의신청 기능 구현
  - [ ] 이의 사유 입력 모달 ✅ 완료 (커스텀 모달 적용, v9)
  - [ ] F-07 전용 Flow 연동 (`CONFIG.FLOW_REVISION_URL`) ← v2.0 신설, URL 입력 필요
  - [ ] 관리자에게 알림 이메일 발송 (F-07 내부 동작)
  - [ ] MonthlyReport RevisionRequest 필드 기록 (이의신청 이력)
- [ ] 재제출 가능 조건 처리 (수정요청 상태일 때만)

#### 3-C. 이행 달성율 평가
- [ ] 팀 제출 내용 API로 실제 데이터 표시
  - [ ] 이행내용 / 특이사항 / 첨부파일 링크
- [ ] 달성율 슬라이더 → API POST 연동 (평가 완료)
- [ ] 수정 요청 기능 구현
  - [ ] 수정요청 사유 입력
  - [ ] 팀 담당자에게 알림 발송
- [ ] 평가 완료 후 목록에서 제거 및 카운트 갱신

#### 3-D. 대시보드 실데이터 연동
- [ ] KPI 수치 → API GET 응답으로 계산
- [ ] 팀별 달성율 바 → 실데이터
- [ ] 월별 추이 차트 → 실데이터 (최근 6개월 자동 계산)
- [ ] 미제출 현황 알림 카드 → 실데이터
- [ ] 독촉 메일 발송 → Power Automate 트리거 연동
- [ ] 연도/월 셀렉터 → 해당 기간 데이터로 전체 대시보드 갱신

#### 3-E. 팀 이행현황 실데이터 연동
- [ ] 팀별 스탯 카드 → 실데이터
- [ ] 추이 차트 → 실데이터

#### 3-F. 내 팀 이행현황 실데이터 연동
- [ ] 로그인한 사용자 팀 기준으로 KPI 계산
- [ ] 차트 실데이터 연동
- [ ] 리스크별 상태 목록 실데이터 연동

---

### Phase 4 — 알림 자동화

- [ ] **미제출 독촉 알림 (F-04, F-05)**
  - [ ] F-04: 매월 26일(D-5) 미제출 팀 담당자에게 자동 이메일 발송
  - [ ] F-05: 매월 30일(D-1) 잔여 미제출 팀에 최종 독촉 + 관리자 CC
  - [ ] Power Automate Scheduled Flow 구성 (되풀이 트리거)
- [ ] **제출 완료 알림 (F-01)** — 관리자 검토 요청
  - [ ] 팀 담당자 제출 시 → 관리자에게 즉시 이메일 알림
- [ ] **평가 완료 알림 (F-02)** — 팀 담당자 통보
  - [ ] 평가 결과(달성율 + 의견) 포함하여 팀 담당자에게 발송
- [ ] **수정 요청 알림 (F-03)** — 관리자→팀 담당자
  - [ ] 수정 요청 사유 포함, RevisionRequested 트리거
- [ ] **이의신청 알림 (F-07)** — 팀 담당자→관리자 ← v2.0 신설
  - [ ] `CONFIG.FLOW_REVISION_URL` 입력 후 활성화
  - [ ] 이의신청 사유 이메일 발송 + MonthlyReport RevisionRequest 필드 기록
  - [ ] F-03(수정 요청)과 역할 구분: F-07은 팀→관리자, F-03은 관리자→팀
- [ ] **Teams 채널 알림** (선택, Microsoft Teams Connector)
  - [ ] 미제출 현황 주간 요약 카드
  - [ ] 전사 달성율 월말 마감 알림

---

### Phase 5 — 고도화 기능

#### 5-A. AI 요약 생성
- [ ] Claude API 연동 (Anthropic)
  - [ ] 월간 이행 내용 텍스트 → AI 요약 생성
  - [ ] "AI 요약 보기" 버튼 추가 (월간 현황 입력 폼 / 평가 폼)
  - [ ] 전사 달성율 기반 → AI 전체 코멘트 자동 생성 (보고서용)
- [ ] PPT 보고서 총평 AI 자동 완성 (입력 필드 옆 "AI 작성" 버튼)

#### 5-B. 보고서 고도화
- [ ] PPT 내 실데이터 완전 연동 (현재 하드코딩 수치 제거)
- [ ] 보고서 생성 이력 저장 (언제 누가 생성했는지)
- [ ] PDF 내보내기 옵션 추가
- [ ] 보고서 링크 공유 기능 (생성된 PPT SharePoint 자동 업로드)

#### 5-C. 연도 관리
- [ ] 연도 초기화 기능 (새 연도 리스크 Master 세팅)
- [ ] 전년도 데이터 아카이브 조회
- [ ] 연도 간 달성율 비교 차트

#### 5-D. 리스크 카테고리 커스터마이징
- [ ] 관리자 설정 화면에서 카테고리 추가/수정/삭제
- [ ] 팀 목록 수정 (현재 5개 팀 고정)

#### 5-E. 감사 로그 (Audit Trail)
- [ ] 데이터 변경 이력 기록 (누가, 언제, 무엇을)
- [ ] 관리자 감사 로그 조회 화면

#### 5-F. 아카이브 고도화 ← v10 신설
- [ ] SharePoint 문서라이브러리 실연동 (F-08)
  - [ ] 파일 업로드 → SP 자동 저장
  - [ ] 연도별 폴더 자동 생성
  - [ ] 파일 목록 SP에서 실시간 조회
- [ ] 법령 인사이트 실데이터 연동 (F-09)
  - [ ] Copilot 에이전트 → SP List(LawInsights) 자동 수집 (주 1회 Scheduled Flow)
  - [ ] 대시보드에서 SP List 조회 → 실시간 표시
  - [ ] 긴급도 자동 분류 (시행일 기준 D-30 이내 = 즉시검토)
- [ ] Copilot 에이전트 iframe 연결
  - [ ] CONFIG.COPILOT_AGENT_URL 입력 후 활성화
  - [ ] Copilot Studio 에이전트 → 로펌 뉴스레터 SharePoint 폴더 연동

---

### Phase 6 — 배포 및 운영 안정화

- [ ] **Vercel 배포 설정**
  - [ ] 환경변수 설정 (API URL, API Key 등)
  - [ ] 사내 도메인 연결 (예: riskops.skchemical.internal)
  - [ ] Preview 배포 환경 구성 (개발/운영 분리)
- [ ] **보안**
  - [ ] API Key 서버사이드 보관 (클라이언트 노출 금지)
  - [ ] CORS 설정
  - [ ] HTTPS 강제
  - [ ] 입력값 검증 (XSS, SQL Injection 방어)
- [ ] **성능**
  - [ ] 대용량 리스크 목록 가상 스크롤 (100건 이상 대비)
  - [ ] API 응답 캐싱 전략
  - [ ] 이미지/파일 업로드 용량 제한 처리
- [ ] **오류 처리**
  - [ ] API 오류 시 사용자 친화적 메시지 표시
  - [ ] 네트워크 단절 시 재시도 로직
  - [ ] 폼 데이터 손실 방지 (페이지 이탈 전 경고)
- [ ] **테스트**
  - [ ] 크로스브라우저 테스트 (Chrome / Edge / Safari / iOS Safari)
  - [ ] 모바일 기기 실기 테스트 (iPhone / Android)
  - [ ] 사용자 수용 테스트 (UAT) — 팀 담당자 1~2명 파일럿
- [ ] **운영 가이드 문서**
  - [ ] 사용자 매뉴얼 (팀 담당자용)
  - [ ] 관리자 운영 가이드
  - [ ] 연도 초기화 절차서

---

## Part 3. 파일 구조 (권장)

```
compliance-riskops/
├── index.html                  ← 현재 파일 (프로토타입)
│
├── src/
│   ├── index.html              ← 진입점 (SSO 리다이렉트 처리)
│   ├── app.js                  ← 앱 초기화, 라우팅
│   ├── auth.js                 ← MSAL.js 인증 모듈
│   ├── api.js                  ← API 클라이언트 (fetch wrapper)
│   │
│   ├── views/
│   │   ├── dashboard.js        ← 전사 대시보드
│   │   ├── monthly.js          ← 월간 현황 입력
│   │   ├── evaluation.js       ← 이행 달성율 평가
│   │   ├── teams.js            ← 팀 이행현황
│   │   ├── master.js           ← 리스크 Master
│   │   ├── selection.js        ← 점검대상 선정
│   │   ├── reports.js          ← 보고서 생성
│   │   ├── archive.js          ← 아카이브 (v10 신설)
│   │   └── myteam.js           ← 내 팀 현황
│   │
│   ├── components/
│   │   ├── chart.js            ← 차트 공통 (Canvas)
│   │   ├── table.js            ← 테이블 공통
│   │   ├── modal.js            ← 모달 공통
│   │   └── toast.js            ← 토스트 알림
│   │
│   └── style/
│       ├── tokens.css          ← CSS 변수
│       ├── layout.css          ← 레이아웃 (사이드바/모바일)
│       └── components.css      ← 공통 컴포넌트
│
├── api/
│   ├── proxy.js                ← Vercel API Route (기존 Claude proxy 구조 참고)
│   ├── risks.js
│   ├── reports.js
│   ├── evaluations.js
│   └── notifications.js
│
└── docs/
    ├── user-manual.md
    ├── admin-guide.md
    └── data-model.md
```

---

## Part 4. 우선순위 요약

| 순서 | 작업 | 이유 |
|------|------|------|
| 1 | Phase 0 — 아키텍처 결정 | 이후 모든 작업의 전제 |
| 2 | Phase 1-A — 데이터 모델 설계 | SharePoint 구성 전 확정 필요 |
| 3 | Phase 2 — 인증 (SSO 또는 간이 로그인) | 실운영의 최소 요건 |
| 4 | Phase 1-B/C + Phase 3-A — 리스크 Master 실연동 | 나머지 기능의 데이터 기반 |
| 5 | Phase 3-B — 월간 현황 입력 실저장 | 팀 담당자 핵심 업무 |
| 6 | Phase 3-C — 평가 실저장 | 관리자 핵심 업무 |
| 7 | Phase 3-D/E/F — 대시보드/현황 실데이터 | 경영진 보고 기반 |
| 8 | Phase 4 — 알림 자동화 (F-01~F-07) | Power Automate 연동 |
| 9 | Phase 5-B — 보고서 실데이터 반영 | PPT 자동화 완성 |
| 10 | Phase 5-F — 아카이브 실연동 (F-08/F-09) | 법령 알리미·과거 자료 |
| 11 | Phase 5-A — AI 요약 | 고도화 (Azure OpenAI via PA) |
| 12 | Phase 6 — 배포/보안/테스트 | 전체 완성 후 |

---

> **현실적 구현 순서 제안:** SharePoint 옵션 A 선택 시, Phase 0→1(SharePoint List 구성)→Phase 2(간이 로그인 or SSO)→3-A(Master 실연동)→3-B(제출)→4(Power Automate 알림 F-01~F-07) 순으로 진행하면 최소 6~8주 내 실운영 가능한 MVP 완성 가능. 아카이브(Phase 5-F)는 SharePoint 연동 완료 후 F-08/F-09를 추가하면 자동 활성화됨.

---

## Part 5. PM Agent / Designer Agent 분석 결과 및 개선 이력

> **분석일:** 2026-03-15 (최종 업데이트: 2026-03-21)
> **방법:** PM Agent(제품 관리 관점) + Designer Agent(UX/UI 관점) 교차 분석
> **대상:** index.html(v8→v10.2), plan.md, SETUP.md, architecture.svg

---

### 5-1. PM Agent 식별 이슈

#### P1 — 실운영 차단 요소

| # | 이슈 | 상태 | 비고 |
|---|------|------|------|
| P1-1 | 인증 없이 역할 선택 — 클릭만으로 관리자 진입 가능 | ⬜ 미해결 | Phase 2(MSAL.js) 선결 필요 |
| P1-2 | 데이터 영속성 전무 — 새로고침 = 전체 초기화 | ✅ 부분해결 | localStorage 오프라인 영속 구현 + 이탈 경고 + 월/연도별 동적 변동 |
| P1-3 | 수정 요청 워크플로 미구현 — alert()만 실행 | ✅ 해결 | 커스텀 모달로 교체, 사유 입력 후 전송 |
| P1-4 | Phase 0 아키텍처 결정 지연 — 후속 작업 전체 블로킹 | ⬜ 미해결 | IT팀 제출 필요 (인프라 의존) |

#### P2 — 기능 갭 및 운영 리스크

| # | 이슈 | 상태 | 비고 |
|---|------|------|------|
| P2-1 | 증빙 파일 업로드 — UI만, 실제 저장 없음 | ⬜ 미해결 | SharePoint 연동 선행 필요 |
| P2-2 | 독촉 메일 — 버튼 색상만 변경, 실제 미발송 | ⬜ 미해결 | Power Automate F-04 연동 필요 |
| P2-3 | 감사 로그(Audit Trail) 부재 | ⬜ 미해결 | Phase 5-E → MVP 포함 격상 권고 |
| P2-4 | 연도 셀렉터 — UI만, 데이터 필터링 없음 | ✅ 해결 | setYear()/setMonth() 데이터 갱신 + 차트 재렌더 연동 |
| P2-5 | PPT 보고서 — 하드코딩 수치 경영진 보고 리스크 | ✅ 해결 | 샘플 데이터 경고 배너 + PPT 파일 내 워터마크 추가 |
| P2-6 | 파일럿 계획 부재 — UAT 없이 전사 배포 위험 | ⬜ 미해결 | 아래 체크리스트 추가 |

#### P3 — 고도화 제안

| # | 이슈 | 상태 |
|---|------|------|
| P3-1 | Teams 알림 카드 (이메일 대비 열람율 높음) | ⬜ Phase 4에 포함 |
| P3-2 | 연도 간 달성율 비교 차트 (YoY) | ⬜ Phase 5-C에 포함 |
| P3-3 | AI 요약 — 평가 부담 경감 | ✅ 기존 구현됨 |

---

### 5-2. Designer Agent 식별 이슈

#### 치명적 — 즉시 수정 대상

| # | 이슈 | 상태 | 적용 내용 |
|---|------|------|----------|
| D1-1 | 제출 확인 단계 없음 — 실수 제출 후 취소 불가 | ✅ 해결 | 제출 전 확인 모달 추가 (이행상태/내용 요약 표시) |
| D1-2 | 색상만으로 상태 구분 — 색맹 접근 불가 | ✅ 해결 | 모든 상태 칩에 아이콘 prefix 추가 (✓/✕/↗/◎/★) |
| D1-3 | 액션 후 피드백 없음 — Toast 부재 | ✅ 기존 구현됨 | toast() 함수 확인 |

#### 사용성 개선

| # | 이슈 | 상태 | 적용 내용 |
|---|------|------|----------|
| D2-1 | 빈 상태(Empty State) 화면 없음 | ✅ 해결 | renderMyTeam/renderEvalList 개선 (v9.3→v10.2 원본 통합) |
| D2-2 | 로그인 흐름 — 역할·팀 2단계가 어색 | ✅ 해결 | Step 1/2/3 인디케이터 추가 |
| D2-3 | 모바일 키보드 가림 문제 | ⬜ 미해결 | visualViewport 연동 향후 구현 |
| D2-4 | 달성율 슬라이더 — 평가 근거 입력란 없음 | ✅ 해결 | 평가 코멘트 란 존재 확인 + 70% 미만 시 필수 처리 추가 |
| D2-5 | 대시보드 정보 밀도 — 첫 화면 인지 과부하 | ✅ 해결 | 상단 즉각 조치 배너 추가 (미제출/평가대기 수치 강조) |

#### 고도화

| # | 이슈 | 상태 |
|---|------|------|
| D3-1 | 히트맵 — 셀 클릭 시 해당 리스크 드릴다운 | ✅ 해결 |
| D3-2 | PPT 미리보기 인라인 편집 | ⬜ 향후 구현 |
| D3-3 | 온보딩 가이드 (Tooltip Tour) | ⬜ 향후 구현 |

---

### 5-3. 적용 완료 목록

#### 2026-03-15 1차 적용 (index.html v8 → v9)

백엔드 없이 즉시 적용 가능한 항목 15개 수정 완료.

| # | 수정 내용 | 분류 |
|---|----------|------|
| 1 | 로그인 Step 인디케이터 CSS + HTML (Step 1→2→3) | UX |
| 2 | Step 인디케이터 JS 연동 (`_updateStepUI()`) | UX |
| 3 | `prompt()` 완전 제거 → `showInputModal()` 커스텀 모달 | UX |
| 4 | `confirm()` 완전 제거 → `showConfirmModal()` 커스텀 모달 | UX |
| 5 | 수정 요청 (`requestEdit`) — 커스텀 모달로 교체 | UX/기능 |
| 6 | 이의신청 (`requestRevision`) — 커스텀 모달로 교체 | UX/기능 |
| 7 | 제출하기 2단계 확인 모달 (이행상태/내용 요약 표시) | UX/안전 |
| 8 | 상태 칩 아이콘 prefix 전체 추가 (`stIcon()` 헬퍼) | 접근성 |
| 9 | 평가 코멘트 70% 미만 시 필수 입력 검증 | 기능 |
| 10 | 히트맵 셀 클릭 → 해당 리스크 드릴다운 모달 | 기능 |
| 11 | 대시보드 상단 즉각 조치 배너 (미제출/평가대기 수치) | UX |
| 12 | PPT 샘플 데이터 경고 배너 (월간/연간, OFFLINE_MODE 연동) | 안전 |
| 13 | PPT 파일 내 SAMPLE DATA 워터마크 (오프라인 모드 시) | 안전 |
| 14 | 페이지 이탈 경고 (`beforeunload` + SPA 내부 이탈 커스텀 모달) | 안전 |
| 15 | localStorage 초기화 `confirm()` → 커스텀 모달 교체 | UX |

#### 2026-03-15 2차 적용 (index.html v9 → v9.1 / 설계서 v1.0 → v2.0)

F-07 이의신청 Flow 신설에 따른 코드·문서 전면 정합성 수정.

| # | 파일 | 수정 내용 | 분류 |
|---|------|----------|------|
| 16 | index.html | `CONFIG`에 `FLOW_REVISION_URL` 변수 추가 | 기능 |
| 17 | index.html | `requestRevision()` — `FLOW_SUBMIT_URL` 재사용 제거, `FLOW_REVISION_URL` 전용 호출로 분리 | 기능/정합성 |
| 18 | index.html | 이의신청 payload에 `type: 'revision_request'`, `riskTitle`, `year`, `month` 필드 추가 | 기능 |
| 19 | index.html | `FLOW_REVISION_URL` 미설정 시 명확한 에러 메시지 출력 | 안전 |
| 20 | index.html | 이의신청 성공 토스트 문구 구체화 ("컴플라이언스팀에서 검토 후 연락" 안내) | UX |
| 21 | PowerAutomate_Flow_설계서.docx | v2.0 신규 발행 — F-07 상세 설계 추가 (7종으로 확장) | 문서 |
| 22 | PowerAutomate_Flow_설계서.docx | `submittedBy` 값 주석 명시 (현재 팀명, SSO 후 사용자명으로 교체 예정) | 문서 |
| 23 | SETUP.md | F-07 생성 방법 섹션 신설 (JSON 스키마 + 단계별 절차) | 문서 |
| 24 | SETUP.md | CONFIG 블록에 `FLOW_REVISION_URL` 항목 추가 | 문서 |
| 25 | SETUP.md | 체크리스트 F-07 및 `CONFIG.FLOW_REVISION_URL` 2개 항목 추가 | 문서 |
| 26 | plan.md | Phase 1-B Flow 목록 F-01~F-07 명세화 | 문서 |
| 27 | plan.md | Phase 1-C API 엔드포인트에 이의신청 경로 추가 | 문서 |
| 28 | plan.md | Phase 3-B 이의신청 항목 F-07 연동 내용으로 업데이트 | 문서 |
| 29 | plan.md | Phase 4 알림 자동화에 F-07 이의신청 알림 항목 추가, F-04/F-05 일정 정정 (25일→26일) | 문서 |

#### 2026-03-18 3차 적용 (index.html v9.1 → v10)

대시보드 첫 화면 전면 재설계 + 아카이브 탭 신설.

| # | 파일 | 수정 내용 | 분류 |
|---|------|----------|------|
| 30 | index.html | 대시보드 첫 화면 순서 재배치: 배너→KPI→달성율→사이클→히트맵+미제출 | UX |
| 31 | index.html | KPI 카드 4종 복원 (kpi-total/submitted/ach/eval ID 추가, 실데이터 연동 준비) | 기능 |
| 32 | index.html | 위험평가 Full Cycle 사이클 카드 설명형 전환 (단계별 desc·why 문장 추가) | UX |
| 33 | index.html | 사이클 타임라인 클릭 → previewPhase() 페이지 이동 없이 Hero 전환 | 기능 |
| 34 | index.html | _getActivePhase() 데이터에 shortLabel·desc·why 속성 전면 추가 | 데이터 |
| 35 | index.html | 위험평가 Full Cycle 스텝바 CSS 전면 교체 (ps-connector·ps-sublabel·ps-step-selected 등) | CSS |
| 36 | index.html | Phase Hero CSS 전면 교체 (phase-hero-banner·phase-hero-desc·action-card-why 등) | CSS |
| 37 | index.html | initDash()에 즉각조치 배너·KPI 업데이트·히트맵·미제출 알림 복구 | 기능 |
| 38 | index.html | 아카이브 탭 신설: NAV_ADMIN·NAV_USER·BNAV_ADMIN·BNAV_USER·PAGE_TITLES 추가 | 기능 |
| 39 | index.html | view-archive HTML 추가 (연도별 자료 + 법령 알리미 + Copilot iframe) | 기능 |
| 40 | index.html | 아카이브 CSS 추가 (archive-file-row·law-alert-row·law-st-badge 등) | CSS |
| 41 | index.html | 아카이브 JS 추가 (initArchive·renderArchiveFiles·renderLawAlerts·initCopilotAgent 등 8개 함수) | 기능 |
| 42 | index.html | CONFIG에 COPILOT_AGENT_URL 추가 | 설정 |
| 43 | index.html | navigateTo()에 archive 라우팅 연결 (양쪽 모두) | 기능 |
| 44 | plan.md | Part 1-3 대시보드 기능 표 업데이트 | 문서 |
| 45 | plan.md | Part 1-11 아카이브 기능 표 신설 | 문서 |
| 46 | plan.md | Phase 1-B F-08·F-09 Flow 추가 | 문서 |
| 47 | plan.md | Phase 1-C API 엔드포인트에 아카이브 경로 추가 | 문서 |
| 48 | plan.md | Phase 5-F 아카이브 고도화 섹션 신설 | 문서 |
| 49 | plan.md | Part 3 파일 구조에 archive.js 추가 | 문서 |
| 50 | plan.md | Part 4 우선순위 표에 Phase 5-F·AI 수단 변경 반영 | 문서 |
| 51 | index.html | 샘플 데이터 연간 사이클 재설계 — 1월 낮음→12월 높음 | 데이터 |
| 52 | index.html | 오프라인 샘플 데이터 월/연도별 동적 변동 | 데이터 |
| 53 | index.html | 월별 달성율 차트 Y축 동적 범위 + 연초 경계선 표시 | 차트 |
| 54 | index.html | 팀 이행현황 차트 Y축 동적 범위 + 연초 경계선 적용 | 차트 |
| 55 | plan.md | v10 전체 문서 동기화 — 기능 인벤토리/로드맵/이력 반영 | 문서 |
| 56 | CLAUDE.md | v10 아키텍처 정보 업데이트 (4420줄, 신규 섹션/함수 반영) | 문서 |
| 57 | SETUP.md | v10 CONFIG 항목 추가 (COPILOT_AGENT_URL), 변경사항 요약 추가 | 문서 |

#### 2026-03-18 4차 적용 (index.html v10 → v10.1)

PM Agent 5차 + Designer Agent 5차 분석 기반 개선 — 11개 항목 적용.

| # | 파일 | 수정 내용 | 분류 |
|---|------|----------|------|
| 58 | index.html | [P1-5] `completeEval()` 오프라인 모드 `_lsSave()` 추가 — 평가 데이터 영속 | 데이터 |
| 59 | index.html | [P1-6] `confirmEditRisk()`/`confirmAddRisk()` localStorage 저장 + 신규 리스크 복원 로직 | 데이터 |
| 60 | index.html | [P1-7] `sendNudge()` 잘못된 URL 치환 제거 → `CONFIG.FLOW_NUDGE_URL` 별도 분리 | 기능/안전 |
| 61 | index.html | [P2-8] `scClass()`/`scColor()` 중간 기준 10→8로 수정 (시스템 기준 통일) | 데이터 |
| 62 | index.html | [P2-11] Excel 업로드 시 기존 RISKS 병합 (중복 제외) — 데이터 소실 방지 | 안전 |
| 63 | index.html | [P2-16] 관리자 월간 제출 `submittedBy`를 리스크 소속팀으로 변경 | 기능 |
| 64 | index.html | [P2-17] `beforeunload` 경고를 eval/master 페이지에도 적용 | 안전 |
| 65 | index.html | [D1-4] 전역 Escape 키 모달 닫기 (6종 모달 + Excel + Bottom Sheet + Drawer) | 접근성 |
| 66 | index.html | [D1-7] Bottom Sheet 오버레이 클릭 시 dirty form 확인 모달 | 안전/UX |
| 67 | index.html | [D2-8] 터치 타겟 44px 확보 (`.af-del`, `.f-btn`, `.cft-btn`) | 접근성 |
| 68 | index.html | [D2-9] `--muted` 색상 `#64748B` → `#475569` (WCAG AA 대비비 7.08:1) | 접근성 |
| 69 | index.html | [D2-6] 아카이브 그리드 브레이크포인트 900px → 768px 통일 | CSS |
| 70 | index.html | [D2-11] 내보내기 버튼 archive/eval 페이지에서 숨김 | UX |
| 71 | index.html | [D2-12] `SHOW_FILTER`에 `monthly` 추가 — 모바일 월간 입력 연/월 변경 가능 | UX |
| 72 | index.html | [D3-12] `.ps-step`/`.ps-dot`/`.ps-label` CSS 중복 선언 제거 | CSS |
| 73 | index.html | [P2-12] `requestRevision()` 오프라인 모드 상태 반영 + localStorage 저장 | 데이터 |
| 74 | index.html | [P2-13] `requestEdit()` 오프라인 모드 localStorage 저장 | 데이터 |

#### 2026-03-21 5차 적용 (index.html v10.1 → v10.2)

Phase 미션 컨트롤 시스템 + 코드 리팩토링 + 가시성 개선.

| # | 파일 | 수정 내용 | 분류 |
|---|------|----------|------|
| 75 | index.html | Phase 미션 컨트롤: 5단계별 체크리스트 + 완료 확정/해제 시스템 | 기능 |
| 76 | index.html | Phase-aware 즉각 조치 배너 (월별→Phase별 알림 분기) | UX |
| 77 | index.html | Phase Step Bar에 완료 상태(확정됨) 반영 | 기능 |
| 78 | index.html | Phase 결정 기준을 curMonth → 실제 달력 월로 분리 (`_getRealMonth()`) | 기능 |
| 79 | index.html | Phase 3 통제수단 조건을 `r.task`(개선 과제) 필드 기반으로 변경 | 기능 |
| 80 | index.html | Master 수정 모달에 Phase 3 기간 통제수단 입력 힌트/뱃지 | UX |
| 81 | index.html | 단계별 미션 전용 탭(`phasecontrol`) 추가 — 히어로 + 아코디언 구조 | 기능 |
| 82 | index.html | Phase 4 체크리스트 간소화 (달성율 60% 체크만) | UX |
| 83 | index.html | 점검대상 선정 페이지: 개선과제 퀵에딧 모달 + 미입력 표시 | 기능 |
| 84 | index.html | Master 테이블에 개선과제 열 추가 (15자 미리보기) | 기능 |
| 85 | index.html | Phase 간 의존성 경고 (이전 단계 미완료 시 토스트) | UX |
| 86 | index.html | 대시보드 KPI에 "사이클 N/5단계 확정" 뱃지 | UX |
| 87 | index.html | 연도 전환 시 이전 연도 Phase 자동 아카이빙 | 기능 |
| 88 | index.html | D-day 완료 후 플러스 표시 제거 | UX |
| 89 | index.html | Master 수정 모달 dirty form 경고 잔존 버그 수정 | 버그 |
| 90 | index.html | **리팩토링 Phase A**: v9.2/v9.3 monkey-patch 16개 원본 통합 | 리팩토링 |
| 91 | index.html | **리팩토링 Phase B**: v10.1 monkey-patch 2개 원본 통합 | 리팩토링 |
| 92 | index.html | **리팩토링 Phase C**: v10.2 initDash 2겹→1겹, navigateTo 흡수 | 리팩토링 |
| 93 | index.html | 전반적 가시성 개선: 최소 폰트 11px + #94A3B8→var(--muted) 통일 | 접근성 |
| 94 | architecture.svg | 아키텍처 다이어그램 v10.2 기준 재작성 | 문서 |
| 95 | plan.md | v10.2 기능 인벤토리 + 적용 이력 업데이트 | 문서 |
| 96 | index.html | 구조적 결함 수정: loadData 동시실행 방지, _lsSave 실패 시 롤백 등 8건 | 버그 |
| 97 | index.html | User 아카이브: 올해 이전 연도만 표시, 법령/Copilot 숨김 | UX |
| 98 | index.html | 현업 월간입력 안내 가이드 배너 (3단계 설명 + D-day) | UX |
| 99 | index.html | 관리자 '이행현황 검토' 탭 통합 (월간입력+평가 합침, eval 탭 제거) | 기능/UX |
| 100 | index.html | 관리자 검토 화면: 상태 필터 + 제출내용 읽기 + 평가 슬라이더 통합 | 기능 |
| 101 | index.html | 샘플 데이터 보강: content/evalComment/task/_revisionRequested 추가 (8건) | 데이터 |
| 102 | index.html | 샘플 시나리오 데이터 월별 진행률 덮어쓰기 버그 수정 (hasScenarioData) | 버그 |
| 103 | index.html | localStorage가 시나리오 데이터 덮어쓰는 버그 수정 (safeOverride) | 버그 |
| 104 | index.html | 관리자 검토 제출내용 상세화: 이행상태/내용/특이사항/첨부 섹션 분리 | UX |
| 105 | index.html | 퀵스코어 버튼에 % 범위 표시 (미달성 0-40%, 달성 80-90% 등) | UX |
| 106 | index.html | 이의신청 검토 프로세스: 수용(점수 조정)/기각(점수 유지) + 검토 의견 | 기능 |
| 107 | index.html | 현업 '평가결과·이력' 탭: split-view + 개선과제 + 평가 + 월별 이력 | 기능 |
| 108 | index.html | 현업 입력화면: 평가완료 포함 전체 리스크 + 미리보기/수정/이의+수정 | 기능/UX |
| 109 | index.html | 팀 리스크 확정 프로세스: 개별 확인/이의제기 + 전체 확인 후 입력 활성 | 기능 |
| 110 | index.html | 배정 확인 화면: 이행 프로세스 4단계 안내 + 진행 상태 프로그레스 | UX |
| 111 | index.html | 올해 월별 이력: localStorage 기반 실데이터 + 클릭 아코디언 펼침 | 기능 |
| 112 | plan.md | v10.2 후반 적용 이력 업데이트 (#96~#112) | 문서 |

---

### 5-4. 해결 완료 항목

```
☑ [P1-2] 데이터 영속성 — localStorage 오프라인 저장 (v9)
☑ [P1-3] 수정 요청 워크플로 — 커스텀 모달 교체 (v9)
☑ [P1-5] 평가 완료 데이터 localStorage 저장 (v10.1)
☑ [P1-6] 리스크 추가/수정 데이터 localStorage 저장 (v10.1)
☑ [P1-7] sendNudge() URL 분리 → FLOW_NUDGE_URL (v10.1)
☑ [P2-4] 연도/월 셀렉터 실데이터 필터링 (v9.3)
☑ [P2-5] PPT 샘플 데이터 경고 배너 + 워터마크 (v9)
☑ [P2-8] scClass/scColor 위험도 기준 통일 — 8점 (v10.1)
☑ [P2-16] 관리자 submittedBy 권한 분리 (v10.1)
☑ [D1-1] 제출 확인 단계 모달 (v9)
☑ [D1-2] 색상+아이콘 상태 구분 접근성 (v9)
☑ [D1-4] 모달 Escape 닫기 + 포커스 관리 (v10.1)
☑ [D1-7] Bottom Sheet dirty form 확인 모달 (v10.1)
☑ [D2-2] 로그인 Step 인디케이터 (v9)
☑ [D2-4] 평가 코멘트 70% 미만 필수 (v9)
☑ [D2-5] 대시보드 즉각조치 배너 (v9)
☑ [D2-6] 아카이브 브레이크포인트 768px 통일 (v10.1)
☑ [D2-8] 터치 타겟 44px 확보 (v10.1)
☑ [D2-9] --muted 색상 WCAG AA 대비비 7.08:1 (v10.1)
☑ [D2-11] 내보내기 버튼 불필요 페이지 숨김 (v10.1)
☑ [D2-12] monthly 페이지 필터바 활성화 (v10.1)
☑ [D3-1] 히트맵 셀 클릭 드릴다운 (v9)
☑ [D3-12] CSS 중복 선언 정리 (v10.1)
☑ [D2-1] Empty State 개선 — renderMyTeam/renderEvalList (v10.2)
☑ [NEW] Phase 미션 컨트롤 시스템 — 5단계 체크리스트+완료 확정 (v10.2)
☑ [NEW] 단계별 미션 전용 탭 — 히어로+아코디언 구조 (v10.2)
☑ [NEW] Phase-aware 즉각 조치 배너 (v10.2)
☑ [NEW] 개선과제 퀵에딧 모달 — 선정 페이지에서 바로 입력 (v10.2)
☑ [NEW] Master 테이블 개선과제 열 추가 (v10.2)
☑ [NEW] Phase 간 의존성 경고 (v10.2)
☑ [NEW] 연도 전환 자동 아카이빙 (v10.2)
☑ [REFACTOR] v9.2/v9.3 monkey-patch 16개 원본 통합 (v10.2)
☑ [REFACTOR] v10.1 monkey-patch 2개 원본 통합 (v10.2)
☑ [REFACTOR] navigateTo 3겹→1겹, initDash 2겹→1겹 (v10.2)
☑ [A11Y] 전반적 가시성 개선 — 최소 11px + 색상 대비 강화 (v10.2)
☑ [NEW] 관리자 이행현황 검토 통합 — 월간입력+평가 합침 (v10.2)
☑ [NEW] 이의신청 검토 프로세스 — 수용/기각 + 점수 조정 (v10.2)
☑ [NEW] 현업 평가결과·이력 탭 — split-view + 월별 이력 아코디언 (v10.2)
☑ [NEW] 현업 입력화면 — 미리보기/수정/이의+수정 동시 제출 (v10.2)
☑ [NEW] 팀 리스크 배정 확인 — 개별 확인/이의제기 프로세스 (v10.2)
☑ [NEW] 배정 확인 4단계 프로세스 안내 (v10.2)
☑ [NEW] 올해 월별 이력 — localStorage 실데이터 + 클릭 펼침 (v10.2)
☑ [FIX] 구조적 결함 8건 — loadData 동시실행, _lsSave 롤백 등 (v10.2)
☑ [FIX] 샘플 시나리오 데이터 보호 — hasScenarioData + safeOverride (v10.2)
```

---

### 5-5. 미해결 항목 — 우선순위별 정리

> 총 37건 미해결. 백엔드 없이 즉시 가능한 항목(★)과 인프라 의존 항목(◆)으로 구분.

#### Tier 1 — 즉시 수정 가능 (백엔드 불필요) ★

프론트엔드 코드 수정만으로 해결 가능한 항목. 다음 패치(v10.2)에서 일괄 적용 권장.

| # | 이슈 | 분류 | 설명 |
|---|------|------|------|
| P2-7 | PPT TREND 인덱스 하드코딩 | 데이터 | `getReportData()` / `getAnnualData()` — TREND[3~5]을 현재 월 기반으로 동적 계산 |
| P2-9 | PPT `event` 암묵적 참조 | 안정성 | `generatePPT()` 등 4곳 — `onclick`에서 event를 명시적 파라미터로 전달 |
| P2-14 | 연도 셀렉터 하드코딩 | 기능 | HTML 2024~2026 고정 → JS 동적 생성 (현재 연도 ±2 자동) |
| P2-15 | `renderRiskTable()` 데드 코드 | 정리 | 존재하지 않는 `#risk-tbody` 참조 — 함수 및 `filterTbl()` 제거 |
| P3-4 | 아카이브 localStorage 저장 | 데이터 | `_archiveFiles` 추가/삭제를 localStorage에 영속화 |
| P3-5 | API Key 노출 경고 | 보안 | ANTHROPIC_API_KEY 설정 시 화면에 "⚠ 운영환경에서는 서버 프록시 권장" 배너 |
| P3-6 | 팀 목록 중앙화 | 유지보수 | 8곳+ 하드코딩 → `CONFIG.TEAMS` 단일 배열, HTML 동적 생성 |
| P3-7 | Canvas HiDPI | UI | `drawChart()` — canvas CSS `height` 속성 명시적 설정 |
| P3-9 | `_esc()` 미적용 innerHTML | 보안 | `showRiskDetail()`, `showHeatmapDrilldown()`, `selectMonthly()` 등 사용자 입력 이스케이프 |
| P1-8 | head `_loadLib` 이중 정의 | 정리 | head 내 중복 제거 (body측이 최종 사용) |
| D1-5 | 리스크 수정 모달 XSS | 보안 | `openRiskEditModal()` — `replace(/"/g)` → `_esc()` 전환 |
| D1-6 | 숫자 필드 범위 초과 경고 | UX | 영향도/가능성 input — `oninput` 실시간 범위 경고 |
| D2-7 | Canvas aria-label 동적화 | 접근성 | `drawChart()` 후 `aria-label`에 실제 데이터 텍스트 반영 |
| D2-10 | 사이클 가로 스크롤 인디케이터 | UX | 모바일 `.phase-timeline` — 좌우 페이드 그라데이션 오버레이 |
| D2-13 | 관리자 Step 2 건너뛰기 | UX | `_updateStepUI()` — 관리자 시 Step 2를 "건너뜀" 표시 |
| D2-14 | 페이지 타이틀 통일 | UI | 모든 페이지에 일관된 헤더 컴포넌트 (제목+설명) 적용 |
| D2-15 | 스켈레톤 로딩 | UX | KPI 카드·차트·테이블에 스켈레톤 패턴, 전체 블로킹 스피너는 API 전용 |
| D3-4 | 키보드 단축키 | 생산성 | `Ctrl+1~8` 페이지 전환 바인딩 |
| D3-5 | KPI 전월 대비 변화량 | 데이터 | `initDash()` — 이전 월 대비 "+3건" / "-5%p" 동적 표시 |
| D3-8 | PPT 미리보기 모바일 | UI | `renderSlidePreview()` — `vh` → `em`/`cqw` 전환, 모바일 텍스트 가독성 |
| D3-10 | 오프라인 배너 모바일 | UI | 모바일 시 `#bottom-nav`과 겹침 방지 — 인라인 배너 전환 |
| D3-11 | 법령 긴급도 라벨 | 접근성 | 8px dot → 텍스트 라벨 ("🔴 긴급" / "🟡 주의" / "⚪ 참고") |

#### Tier 2 — 인프라 의존 (IT팀/SharePoint 필요) ◆

| # | 이슈 | Phase | 설명 |
|---|------|-------|------|
| P1-1 | MSAL.js SSO 연동 | 2 | Azure AD 앱 등록 → 로그인 화면 SSO 리다이렉트 전환 |
| P1-4 | 아키텍처 결정 | 0 | IT팀 SharePoint 구성요청서 제출, 인프라 방향 확정 |
| P2-1 | 증빙 파일 업로드 | 3-B | SharePoint 문서 라이브러리 or Azure Blob 연동 |
| P2-2 | 독촉 메일 실발송 | 4 | Power Automate F-04/F-05 + `FLOW_NUDGE_URL` 연동 |
| P2-3 | Audit Trail | 5-E | 데이터 변경 이력 기록 + 관리자 조회 화면 |
| F-07 | 이의신청 Flow | 4 | Power Automate 생성 → `CONFIG.FLOW_REVISION_URL` 입력 |
| F-08 | 아카이브 파일 SP 연동 | 5-F | Power Automate → SharePoint 문서라이브러리 자동 저장 |
| F-09 | 법령 인사이트 수집 | 5-F | Copilot → SP List Scheduled Flow (주 1회) |
| Copilot | 에이전트 연결 | 5-F | Copilot Studio 에이전트 생성 → `CONFIG.COPILOT_AGENT_URL` 입력 |

#### Tier 3 — 고도화 (MVP 이후)

| # | 이슈 | Phase | 설명 |
|---|------|-------|------|
| ~~D2-1~~ | ~~Empty State~~ | ~~-~~ | ~~v10.2에서 해결: renderMyTeam/renderEvalList 개선~~ |
| D2-3 | 모바일 키보드 가림 | - | `visualViewport` 기반 Bottom Sheet 높이 자동 보정 |
| D3-2 | PPT 인라인 편집 | 5-B | 미리보기에서 직접 텍스트 수정 |
| D3-3 | 온보딩 Tooltip Tour | - | 첫 로그인 사용자 가이드 투어 |
| D3-6 | 다크 모드 | - | CSS 변수 기반 prefers-color-scheme or 토글 |
| D3-7 | 바 애니메이션 성능 | - | `width` transition → `transform:scaleX()` |
| D3-9 | 테이블 행 클릭 상세 | - | Master 테이블 행 클릭 → `showRiskDetail()` |
| P3-8 | 사이클 매핑 CONFIG화 | - | 월-단계 매핑을 CONFIG 테이블로 유연화 |
| P3-10 | spGet() 페이지네이션 | - | `$top=500` 제한 → nextLink 기반 페이징 |
| 파일럿 | 팀 선정 + 피드백 | 6 | 1개 팀 2주 파일럿, 피드백 채널 구축 |
| 매뉴얼 | 사용자 가이드 | 6 | 팀 담당자용 1페이지 + 관리자 운영 가이드 |

---

### 5-6. 파일럿 배포 전 최소 요건 (Go/No-Go 기준)

실운영 파일럿 배포 전 반드시 완료해야 할 항목:

| 항목 | 기준 | 현재 상태 |
|------|------|----------|
| 인증 | SSO 또는 이메일 도메인 검증 | ❌ 미구현 (Phase 2) |
| 데이터 저장 | 제출 내용이 새로고침 후에도 유지 | ⚠️ localStorage 구현 완료, SP 연동 필요 |
| 알림 | 제출 시 관리자 이메일 발송 | ❌ PA Flow 필요 (Phase 4) |
| 보안 | XSS 방어, API Key 서버사이드 보관 | ⚠️ `_esc()` 부분 적용, Tier 1에서 완료 필요 |
| 위험도 기준 | 시스템 전체 일관성 (8~14=중간) | ✅ v10.1에서 통일 |
| 접근성 | WCAG AA 색상 대비, 터치 44px | ✅ v10.1에서 개선 |
| 사용자 매뉴얼 | 팀 담당자 1페이지 가이드 | ❌ 미작성 |
| 브라우저 테스트 | Chrome/Edge/iOS Safari | ⬜ 미수행 |
