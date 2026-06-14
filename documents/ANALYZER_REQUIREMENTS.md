# DailyReportAnalyzer — 요구사항 명세서

**Crafted by IDO(idocho@kakao.com) · Powered by Claude AI**  
**문서 버전**: 2.9 · **앱 버전**: v0.4 · **최종 수정**: 2026-06-14

> Firebase 스키마: [ClassManager/documents/DB_SCHEMA.md](../../ClassManager/documents/DB_SCHEMA.md) 참조  
> 이전 버전 이력: v1.2까지 동일 파일 내 기록

---

## 변경 이력

| 버전 | 날짜 | 내용 |
|------|------|------|
| 2.9 | 2026-06-14 | **자기주도 다과목 병합 보정 (§4.5)** — 실데이터(조이도 22명) 레이더 검증 중 발견: 과제수행도를 같은 날 과목 중 최악값 병합 → 다과목 학생이 한 교재만 빠뜨려도 autonomy 0 추락(이승진 4과목, 실제 most·half 수행했으나 3점). 병합 시 과목별 등급 모음(`__hwAll`) 누적 후 **전 과목-세션 평균**으로 autonomy 계산(3→20). 표시·AI노트용 `assign_grade`(최악값)은 유지. condition·understand 병합은 별건으로 잔존 |
| 2.8 | 2026-06-14 | **스탯 Formula 개정 + "그럴듯" 균형 보정 (§4.5)** — 감사 결함 수정. 신태그 카운터 추가 후: ② 참여도 `+deep_try*12` + **결측=중립50 baseline**(희소 positive-only 축 dent 해소, `50+min(50,engRate*5)`), ④ 이해응용 `+process_good*8`·`writeup_weak` 약한 감점(cap8), ⑤ 성취 죽은 perfect/improved→`effort`·성적결측 50중립, ① 학습태도 condition결측 65중립, ③ 자기주도 과제가중 **0.7→0.85**(폐기태그 의존 천장70 해소). slow·calc_miss 관찰용 무패널티. 표시·AI프롬프트·빈도차트 신태그 반영. 전형 학생 5종 5축 시뮬 편차 11~18 균형 검증·JS문법 OK. 미반영: 가중치 config화 |
| 2.7 | 2026-06-14 | **스탯 Formula 감사 박제 (§4.5 신설, 수정 대기)** — `computeWindowData` 레이더 5축·성적 집계 전반 점검. 🔴 v8.30 폐기 6태그(present·help·preview·error_fix·perfect·improved)가 공식 입력으로 잔존해 참여도·자기주도·성취 보너스 무력화 + 신태그(deep_try·process_good·slow·calc_miss·writeup_weak) 미배선(analyzer.html 0회) — 태그 3중 정의 동기화 부채. 🟠 결측 폴백 비대칭(attitude 20·achievement 0 추락 vs 이해도 65 중립). 🟡 참여도 천장 85·매직넘버 산재·다과목 보수병합. ✅ 백분위 중립·클램프·날짜정렬·동점rank 정상. **코드 미반영** — 가중치 합의 후 수정 |
| 1.0 | 2026-05-21 | 최초 작성 |
| 1.1 | 2026-05-23 | Firebase obs 노드 구조 DRW_REQUIREMENTS와 통일. 데이터 소스 키 정합성 수정. AI 모델 현행화 |
| 1.2 | 2026-05-25 | scores/ 연동 구현: 성적 추이 섹션, 성취도 레이더 축, AI 프롬프트 성적 반영 |
| 2.0 | 2026-05-27 | DB 구조 전면 재설계 반영. 학생 목록 로드 경로 변경. obs subject별 aggregation 추가. scores weekly/achievement 분리 |
| 2.1 | 2026-05-28 | nameKey = 출결번호 (불변 고유번호). 이름 기반 키 폐기. 읽기 전용 도구이므로 코드 영향 없음 — 스키마 참조 업데이트만 |
| 2.2 | 2026-05-30 | 역량 레이더 비포·애프터 오버레이 추가 (비교 기간 끔/직전 동기간/직접 지정 3모드, 변화량 병기). 축 계산 `computeWindowData`로 일원화 |
| 2.6 | 2026-06-12 | **v0.4 — 학년단위 시험(`scores/achievement/`) 연동**: bulk fetch에 achievement 전체 추가, `aggregateStudentScores`를 weekly·achievement 공용 수집기로 리팩터(`collect()` — grade 플래그). 학년 시험 코호트 = 노드 내장 students 맵(과정 수강생 전체) — 백분위·등수·평균이 학년 기준. testKey 날짜 없음(`유형\|회차`, DRW v8.23) — 날짜는 meta 대표값 사용. 표시: classId 「학년」, subject=curriculumKey, 라벨 「학년평균」(리뷰·리포트·AI 프롬프트), 범례 「평균(학급/학년)」. 스텁 검증: 학년 5명 코호트 3등→백분위 50, 구형 평면 폴백 동작 |
| 2.5 | 2026-06-12 | **v0.3.2 — 성적 집계 견고화** (`aggregateStudentScores`): ① 구형 평면 레코드 폴백 — `meta` 서브키 없으면 루트 필드 사용(DRW 읽기 로직과 정합, 종전엔 무음 스킵돼 성적 추이 누락) ② `myPct`/`avgPct` 0~100 클램프 — 만점 초과·음수 데이터가 막대 오버플로·성취성장 축 왜곡하지 않도록 ③ 단독 응시(코호트 1명) 백분위 100→**50 중립** (성취성장 축 부풀림 방지) |
| 2.4 | 2026-06-11 | **v0.3.1 — Security Rules 전환 사전 배선(#15)**: ① `fbE()` DB Secret(`?auth=`) 옵션 부가 — 설정 패널 「DB 시크릿」 입력란(`drw_fb_secret` localStorage), 「DRW 설정 가져오기」가 시크릿(`drw_db_secret`)도 복사. 미설정 시 종전과 동일(no-op). ② `schema_version` 게이트 — `loadClasses()` 진입 시 DB 노드 확인, 클라 `SCHEMA_MAX`(14) 초과면 **경고 토스트만**(read-only라 차단 없음). 노드 부재·읽기 실패=통과. 전환 절차: DRW `documents/SECURITY_RULES_PLAN.md` |
| 2.3 | 2026-06-11 | **v0.3 — nameKey-first 종단 비교**: ① 과목 순회를 nameKey-first로 전환 (obs 실존 과목 ∪ 현재 반 과목 union — 반 이동·과목 삭제 후에도 과거 데이터 포함), ② `scores/weekly` 전체 1회 GET 후 역수집 (classId 무관, students에 nameKey 있는 시험 전부 — 성적 추이가 반 이동 생존, 백분위 코호트는 시험 노드 내장 students 맵), ③ 레이더 비교 기간 선택 확장 (지난달·직전 동기간 기본 / 3개월 전 / 6개월 전 / 작년 동월 / 직접 지정 — `offsetWindow()` 말일 클램프), ④ 문서 정합: `scores/achievement`·`session/class_data`는 실제 코드 미사용임을 명시 |

---

## 1. 프로젝트 개요

### 1.1 목적

DRW2가 Firebase에 축적한 일별 관찰 데이터와 성적 데이터를 기반으로,  
담임교사가 학부모에게 발송할 **월간 학생 성장 리포트**를 생성·검토·출력한다.

### 1.2 구성 파일

| 파일 | 역할 |
|------|------|
| `analyzer.html` | 단일 파일 — 생성·검토·출력 전체 (브라우저 실행) |

### 1.3 운용 시나리오

```
[월말] 담임교사가 웹브라우저에서 analyzer.html 접속
  1. 대상 반·기간 선택
  2. 일괄 생성 (AI가 전체 학생 리포트 순차 작성, ~2분)
  3. 학생별 검토 + 튜닝
  4. 학생별 컨펌
  5. 컨펌된 학생 리포트 HTML 링크 생성 → 카카오톡 수동 전송
```

---

## 2. Firebase 연동

### 2.1 읽기 전용

`analyzer.html`은 Firebase를 **읽기 전용**으로만 사용. 쓰기 없음.

### 2.2 읽는 노드 및 경로 (v0.3 현행)

| 노드 | 경로 | 용도 |
|------|------|------|
| 학생 목록 | `students/` (전체 GET 후 class 필터) | 반별 학생 필터링 |
| 반 정보 | `classes/` (전체 GET) | 과목(subject) 목록, curriculum |
| 수업 관찰 + 과제수행도 | `obs/{nameKey}/` (학생 전체 GET) | 관찰 태그 + `assign_grade`(과제수행도 단일 소스). **하위 과목 키 = 과목 발견 소스**(v0.3) |
| 특이사항(당일) | `input/{nameKey}/__note__/note` | **학생 단위 단일**(v2.1.2). 구 `{subject}.note` 는 fallback |
| 전송 코멘트(누적) | `history/{nameKey}/{date}/` | 과거 전송 최종 코멘트 `{note,instructor}` — 반복 회피·맥락 (v2.1.2 신규) |
| 주간 성적 | `scores/weekly/` (**전체 1회 GET, v0.3**) | 모든 시험을 역수집 — students에 nameKey 있는 시험 전부 (classId 무관) |

> **미사용 노드 (v0.4 정합)**: `session/class_data/`(진도/과제)는 현재 코드가 **읽지 않음**. `scores/achievement/`는 v0.4에서 연동 완료 — 성적 추이·성취성장 축에 학년단위 시험 포함(코호트=과정 수강생 전체, 라벨 「학년평균」).

> **v2.1.2 정합 주의**: 특이사항은 과목 종속이 아니라 **학생 단위 단일**(`__note__`). 과목별 루프로 `.note` 읽으면 빈값(회귀) — `__note__` 직접 읽기. 과제수행도는 `input/.assign`(폐기) 아닌 `obs/assign_grade`.

### 2.3 경로 변환 (구 → 신)

| 구 경로 | 신 경로 |
|---------|---------|
| `config/sheets/` | `students/` + `classes/` |
| `obs/{group}\|{classId}\|{name}/{date}` | `obs/{nameKey}/{subject}/{date}` (subject별 aggregation) |
| `input/{group}\|{classId}\|{name}\|{subject}` | `input/{nameKey}/{subject}/` |
| `scores/{group}\|{classId}/{testKey}/` | `scores/weekly/{classId}/{subject}/{testKey}/` |

### 2.4 obs subject별 Aggregation — nameKey-first (v0.3)

신규 구조에서 `obs`는 subject별로 분리 저장됨 (학생 grain — 반 삭제에도 생존).  
**과목 발견은 nameKey-first**: 학생의 과목 목록 = `obs/{nameKey}` 하위에 실존하는 과목 키 ∪ 현재 반 course 키.

```js
// v0.3: obs 실존 과목 ∪ 현재 반 과목 (반 이동·과목 삭제 후 과거 데이터 포함)
const obsSubs = Object.keys(allObs[nameKey] || {});
r.subjects = [...new Set([...currentCourseKeys, ...obsSubs])];
```

- 현재 반 course 목록만 순회하던 구버전은 학기 간 반/교재 변경 시 과거 obs가 누락됨 → v0.3에서 해소
- `archived:true`(소프트 삭제) course 데이터도 포함
- 과목 키: 신형 `"{curriculum} {textbook}"` + 레거시 교재명 단독 키 혼재 — 키 형식 무관하게 obs 실존 키 그대로 순회

월간 리포트 생성 시 **해당 학생의 모든 subject obs를 날짜 기준으로 병합**.

같은 날짜에 여러 subject 수업이 있는 경우:
- `condition`: 가장 낮은 값 우선 (보수적 집계)
- `understand`, `engage`, `caution`, `extra`, `highlight`: 전체 union

---

## 3. 화면 구성

### 3.1 전체 플로우

```
[Step 1] 생성 설정   →   [Step 2] 검토·튜닝   →   [Step 3] 출력
반·기간·항목 선택         학생별 검토·수정           HTML 링크 생성
일괄 생성 실행             컨펌                       (카카오 수동 전송)
```

### 3.2 Step 1 — 생성 설정

**대상 반 선택**
- 반 목록: `classes/`에서 로드 → group(M/T) 기준 필터 가능
- 각 반 카드: 반명 + 학생 수 (students/?orderBy=class로 집계)

**기간 선택**: 이번 달 / 지난 달 / 최근 3개월 / 직접 입력

**포함 항목 선택** (기본 전체 체크):
역량 레이더 / 수업 컨디션 달력 / 관찰 집계 / 성적 추이 / AI 서술 평가 / 기억에 남는 순간 / 다음달 집중 포인트 / 선생님 한마디

---

## 4. 학부모 리포트 HTML

### 4.1 대상

- 모바일 세로 스크롤 (카카오톡 인앱 브라우저)
- 최대 너비 380px

### 4.2 섹션 구성

```
[커버]        학원명 · 학생명 · 반/기간 · AI 커버 문장
[01]          선생님이 본 이번 달 (AI 서술 2~3문장)
[02]          역량 레이더 (5각형 스파이더 차트)
[03]          한 달 수업 컨디션 (날짜별 달력)
[04]          이달에 보여준 것들 (긍정 태그 횟수 바)
[05]          성적 추이 (주간/학년단위 구분)
[06]          선생님이 기억하는 순간 (AI 선별 2~3건)
[07]          이달의 키워드
[08]          다음달 집중 포인트 (AI 3가지)
[푸터]        선생님 한마디 · 학원명 · 담임 · 발송일
```

### 4.3 역량 레이더 5축

| 축 | 원천 | 산출 |
|---|---|---|
| ① 수업 태도 | `obs.condition` (aggregated) | great+good 비율 |
| ② 참여도 | `obs.engage` (발표+질문) | 발생 횟수 / 수업수 × 100 |
| ③ 과제 성실도 | `input.assign` | 성실 계열 / 수업수 × 100 |
| ④ 이해도 | `obs.understand` + `understand_sub` | top/good 비율 + 태그 가중 |
| ⑤ 성취도 | `scores/weekly/` (역수집, v0.3) | 최근 시험 학급 내 백분율 — 코호트는 시험 노드 내장 students 맵 (`scores/achievement/` 미연동) |

#### 4.3.1 비포·애프터 오버레이 (비교 기간)

- 선택 기간(애프터)과 **비교 기간**(비포)의 5축을 한 차트에 겹쳐 표시.
- **비교 기간 모드** (`compareMode`, UI: "레이더 비교 기간") — v0.3 확장:
  - `off` — 비교 안 함 (단일 폴리곤)
  - `auto` — **지난달(직전 동일 길이 기간)** = `prevWindow(start,end)` (기본값 — 기존 동작 동일)
  - `m3` / `m6` / `y1` — **3개월 전 / 6개월 전 / 작년 동월(12개월 전)** = `offsetWindow(start,end,N)` — 기준 기간을 N개월 평행이동, 말일 클램프(예: 5/31→2/28)
  - `custom` — 날짜 2개 직접 지정 (`compareRange`, `cmp-start`/`cmp-end`)
  - 해석: `resolveCompareWindow()` → `{start,end,label}` 또는 `null`
- 축 계산 로직은 `computeWindowData(mergedObs, r, winStart, winEnd)`로 일원화 → 두 기간에 동일 적용.
- 비교 기간에 관찰/성적 데이터가 전혀 없으면 오버레이 생략(단일 폴리곤).
- 시각: 현재=파란 실선, 비교=회색 점선. 각 축 라벨에 변화량(`▲`/`▼`) 병기. 차트 하단 범례에 기간 라벨 표기.
- 검토 화면(`radarSVG`, 다크)·최종 리포트(`buildReportRadarSVG`, 라이트) 양쪽 적용.
- 데이터: `aggregateStudentData`가 `radarAxesPrev`, `radarPrevLabel`, `radarPrevCount`, `radarRangeLabel` 추가 반환.

### 4.4 성적 추이 섹션 (v0.3 변경)

- **주간 시험**: `scores/weekly/` 전체를 1회 GET 후 **역수집** — `students`에 해당 nameKey가 있는 시험을 classId 무관하게 수집 (반 개편 후 orphan classId의 과거 시험도 포함 → 추이 생존)
- 학급 평균·등수·백분위 코호트 = 해당 시험 노드의 `students` 맵 (응시 당시 학급이 내장됨)
- **학년단위 시험**(`scores/achievement/`): 미구현 — 향후 과제
- 학급 평균 대비 표시 유지

---

## 4.5 스탯 Formula 감사 + 개정 (2026-06-14 · **반영 완료** v2.8)

`computeWindowData`(analyzer.html) 레이더 5축·성적 집계 전반 점검 결과와 개정 내역. 아래 🔴🟠 결함은 **2026-06-14 수정 반영됨**(개정 매핑은 맨 아래 표). 🟡 일부(매직넘버 config화)는 후속 과제로 잔존.

### 🔴 Critical — v8.30 태그 개편이 공식 3축을 무력화

DRW v8.30에서 폐기된 6태그(`present`·`help`·`preview`·`error_fix`·`perfect`·`improved`)가 **여전히 스탯 공식 입력**인데 웹 입력에서 제거돼 신규 데이터에선 항상 0. 대체 신태그(`deep_try`·`process_good`)는 **미배선**. 집계 루프(`analyzer.html:774-783`)가 `if(obj[k]!==undefined)` 가드라 미지 키는 무음 누락.

| 축 | 공식 | line | 결함 |
|----|------|------|------|
| ② 참여도 | `present*14+question*10+help*6` | 794 | present·help 사장 → 사실상 question만. `deep_try` 0 기여 |
| ③ 자기주도 | `preview*18+self_study*15+error_fix*12+retry*10` | 801 | preview·error_fix 사장 → 보너스 반감 |
| ⑤ 성취성장 | `(perfect+improved)*15` | 815 | 둘 다 사장 → hlBonus≈0. `process_good` 미반영 |

신태그 5종(`deep_try`·`process_good`·`slow`·`calc_miss`·`writeup_weak`) analyzer.html 0회 출현 — 카운터(767-771)·표시(1064-1077)·빈도차트(1261-1266/1770-1774) 전부 구 택소노미. **태그 정의 3중화(웹 `app-core.js`·PC `constants.py`·Analyzer)** 동기화 부채.

### 🟠 Correctness — 결측 폴백 비대칭

| 축 | 결함 | line |
|----|------|------|
| ① 학습태도 | condition 0건이면 condAvg=0 → **attitude 20 추락** (이해도는 65 중립 폴백인데 비대칭) | 789-792 |
| ⑤ 성취성장 | 성적 0건이면 **achievement=0** (결측을 "최악"으로 오독) | 816 |

→ 결측은 "데이터 없음"(중립 또는 축 제외)으로 통일 필요.

### 🟡 Design / 품질

- **참여도 천장 85** (`35+min(50,…)`, 795) — 타축 100과 비대칭. `engRate*8` 포화 빨라 세션당 태그 1개면 만점 → 변별력 저하
- **매직넘버 산재** (패널티 8/5/6/10·참여 14/10/6·자주 18/15/12/10·점수 0.6/0.4 등) — 중앙 config 부재로 튜닝·검증 난해
- **다과목 결측병합 보수적**(840-863): 같은 날 과목 간 condition/understand/assign을 **최악값** 병합 → 다과목 학생 체계적 불리(의도됐으나 영향 큼)

### ✅ 검증 OK (정상)

- 단독응시 백분위 50 중립(966) · myPct/avgPct 0~100 클램프(959) · 최근3 시험 날짜정렬 후 slice(978→812) · rank 동점 공유(963-966)

### 개정 반영 (2026-06-14, v2.8)

신태그 카운터 추가(767-771) 후 공식 배선·결측 폴백 수정. 폐기 키는 옛 데이터 호환 위해 카운터에 잔존(공식 기여는 신규 데이터서 0).

| 축 | 개정 |
|----|------|
| ① 학습태도 | condition 0건 → condAvg **65 중립**(종전 0→attitude 20추락 해소). slow·calc_miss·writeup_weak 패널티 미반영 |
| ② 참여도 | `+deep_try*12` 배선. **희소 positive-only 축 → 결측=중립 50 baseline**(`50+min(50,round(engRate*5))`). 태그 부재가 floor-30 dent 만들던 문제 해소, 50→100 매끄럽게 |
| ③ 자기주도 | autoBehavior 주력(preview·error_fix)이 v8.30 폐기로 사실상 0 → 만점 과제도 천장 70. **과제 가중 0.7→0.85**(cap 15)로 천장 해소. **과제평균 = 날짜 최악값 병합 → 전 과목-세션 평균**(`__hwAll`): 다과목 학생이 한 교재만 빠뜨려도 0으로 깎이던 문제 해소(예: 4과목 학생 autonomy 3→20). 단, 표시·AI노트용 `assign_grade`·`hwCounts`는 최악값 유지(빠뜨린 교재 적시 목적) |
| ④ 이해응용 | `+process_good*8` 보너스, `writeup_weak` **약한 감점**(cap 8, process_good의 대칭축), 하한 20 |
| ⑤ 성취성장 | 죽은 perfect·improved 대신 `effort` 배선, 성적 결측 → **중립 50**(종전 0) |
| 표시/AI | 수업참여에 심화도전, 이해응용에 풀이과정우수, 주의에 풀이정확성·서술 보완 추가. 빈도차트 deep_try 칩(폐기 칩 제거) |

**"그럴듯" 2차 점검 (펜타곤 균형)**: 1차 수정 후 전형 학생 시뮬레이션에서 ②참여(희소태그 floor-30 dent)·③자주(폐기태그로 천장 70)가 축 불균형 유발 발견 → 위 표대로 재보정. 결과 전형 4종 **편차 11~18 균형 펜타곤**, 큰 편차는 "참여태그 0" 결측 엣지(중립 50으로 정직 표현)뿐.

**실데이터 검증(조이도 22명)**: 실저장 obs/scores로 레이더 생성 중 자기주도 병합 결함 발견 — 4과목 학생(이승진)이 한 교재(라이트쎈) 미완으로 매 세션 최악값 none 병합돼 autonomy 3, 실제론 우공비표준 most·half 수행. 위 ③ 과목평균 보정으로 3→20 교정(22명 중 다과목 학생 전반 상향).

**미반영(후속)**: 가중치 중앙 config화, condition·understand 도 다과목 최악값 병합(단일등급이라 논쟁 여지·별건). **검증**: 전형 5종 시뮬 + 실데이터 22명 레이더(편차·결측 정직성 확인), JS 문법 OK.

`slow`·`calc_miss`·`writeup_weak` 패널티 정책: slow·calc_miss는 v8.30 관찰용(무패널티), writeup_weak만 이해응용 약한 감점(스파이더 차트 균형 — process_good 보너스와 대칭).

---

## 5. AI 생성

| 항목 | 내용 |
|------|------|
| 모델 | `claude-sonnet-4-6` |
| 호출 방식 | 학생별 순차 처리 (rate limit 고려) |
| API Key | 클라이언트 로컬 보관 (내부 도구) |

**생성 섹션**: 커버 문장 / AI 서술 평가 / 기억에 남는 순간 / 다음달 집중 포인트 / 선생님 한마디

**caution 처리**:
- `late`: 직접 언급 허용
- `sleepy` / `chat` / `attitude`: 완곡 표현만

---

## 6. 스코프 제외

| 항목 | 사유 |
|------|------|
| 카카오톡 자동 전송 | DRW PC 앱 역할. Analyzer는 링크 생성만 |
| Firebase 쓰기 | 읽기 전용 도구 |
| 실시간 동기화 | 월말 1회 생성 구조 |
