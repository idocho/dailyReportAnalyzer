# DailyReportAnalyzer — 요구사항 명세서

**Crafted by IDO(idocho@kakao.com) · Powered by Claude AI**  
**문서 버전**: 2.1 · **앱 버전**: v0.2.0 · **최종 수정**: 2026-05-28

> Firebase 스키마: [ClassManager/documents/DB_SCHEMA.md](../../ClassManager/documents/DB_SCHEMA.md) 참조  
> 이전 버전 이력: v1.2까지 동일 파일 내 기록

---

## 변경 이력

| 버전 | 날짜 | 내용 |
|------|------|------|
| 1.0 | 2026-05-21 | 최초 작성 |
| 1.1 | 2026-05-23 | Firebase obs 노드 구조 DRW_REQUIREMENTS와 통일. 데이터 소스 키 정합성 수정. AI 모델 현행화 |
| 1.2 | 2026-05-25 | scores/ 연동 구현: 성적 추이 섹션, 성취도 레이더 축, AI 프롬프트 성적 반영 |
| 2.0 | 2026-05-27 | DB 구조 전면 재설계 반영. 학생 목록 로드 경로 변경. obs subject별 aggregation 추가. scores weekly/achievement 분리 |
| 2.1 | 2026-05-28 | nameKey = 출결번호 (불변 고유번호). 이름 기반 키 폐기. 읽기 전용 도구이므로 코드 영향 없음 — 스키마 참조 업데이트만 |

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

### 2.2 읽는 노드 및 경로 (v2.0 신규)

| 노드 | 경로 | 용도 |
|------|------|------|
| 학생 목록 | `students/?orderBy="class"&equalTo="{classId}"` | 반별 학생 필터링 |
| 반 정보 | `classes/{classId}/courses/` | 과목(subject) 목록, curriculum |
| 수업 관찰 | `obs/{nameKey}/{subject}/{date}/` | 관찰 태그 (subject별 aggregation) |
| 과제/메모 | `input/{nameKey}/{subject}/` | 과제 수행도, 특이사항 |
| 반별 성적 | `scores/weekly/{classId}/{subject}/{testKey}/` | 주간 시험 성적 |
| 학년단위 성적 | `scores/achievement/{curriculumKey}/{testKey}/` | 성취도평가 등 |
| 진도/과제 | `session/class_data/` | 진도 정보 |

### 2.3 경로 변환 (구 → 신)

| 구 경로 | 신 경로 |
|---------|---------|
| `config/sheets/` | `students/` + `classes/` |
| `obs/{group}\|{classId}\|{name}/{date}` | `obs/{nameKey}/{subject}/{date}` (subject별 aggregation) |
| `input/{group}\|{classId}\|{name}\|{subject}` | `input/{nameKey}/{subject}/` |
| `scores/{group}\|{classId}/{testKey}/` | `scores/weekly/{classId}/{subject}/{testKey}/` |

### 2.4 obs subject별 Aggregation

신규 구조에서 `obs`는 subject별로 분리 저장됨.  
월간 리포트 생성 시 **해당 학생의 모든 subject obs를 날짜 기준으로 병합**:

```js
// 모든 subject obs 병합
const allObs = {};
for (const subject of Object.keys(studentSubjects)) {
  const subjectObs = await fbGet(`obs/${nameKey}/${subject}`);
  for (const [date, data] of Object.entries(subjectObs || {})) {
    if (!allObs[date]) allObs[date] = [];
    allObs[date].push({ subject, ...data });
  }
}
```

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
| ⑤ 성취도 | `scores/weekly/` + `scores/achievement/` | 최근 시험 학급 내 백분율 |

### 4.4 성적 추이 섹션 (v2.0 변경)

- **주간 시험**: `scores/weekly/{classId}/{subject}/` — subject별 그룹으로 표시
- **학년단위 시험**: `scores/achievement/{curriculumKey}/` — 별도 섹션으로 표시
- 학급 평균 대비 표시 유지

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
