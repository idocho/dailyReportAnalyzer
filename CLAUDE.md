# DailyReportAnalyzer — Claude 작업 지침

## 프로젝트 개요

DailyReportWizard 2.0이 Firebase에 축적한 수업 관찰 데이터를 기반으로  
월간 학부모 리포트를 생성·검토·출력하는 단일 파일 웹 앱.

## 필수 규칙

### 요구사항 문서 동기화
**코드 수정 시 반드시 `documents/ANALYZER_REQUIREMENTS.md`도 함께 업데이트.**

### 프로젝트 파일 위치
| 항목 | 경로 |
|------|------|
| 메인 앱 | `analyzer.html` (단일 파일 — CSS/JS 모두 인라인) |
| 요구사항 문서 | `documents/ANALYZER_REQUIREMENTS.md` |

## 아키텍처

- **단일 파일**: `analyzer.html` — CSS/JS 인라인, 외부 의존성 없음
- **Firebase 읽기 전용** — 쓰기 없음
- **Claude API 직접 호출** (브라우저, `anthropic-dangerous-direct-browser-access: true`)
- **로컬 저장**: localStorage (`drw_fb_url`, `drw_fb_path`, `drw_analyzer_claude_key`)

## Firebase 데이터 구조 (DRW 2.0 공유)

```
students/{nameKey}            학생 명단 {name, class}  (nameKey = 출결번호, unique)
classes/{classId}             학급 {group, courses:{subject:{curriculum}}}
input/{nameKey}/__note__      당일 특이사항 (학생 단위 단일) {note}   ← v2.1.2: 과목 종속 아님
obs/{nameKey}/{subject}/{YYYY-MM-DD}  관찰 태그 + 과제수행도 (날짜별 히스토리)
  assign_grade, assign_tags[], condition, understand, understand_sub[], engage[], caution[], extra[], highlight[]
scores/weekly/{classId}/{subject}/{testKey}   주간 시험 점수
session/class_data/{classId|subject}          반 공통 진도/과제 {progress, homework}
history/{nameKey}/{YYYY-MM-DD}  전송된 최종 데일리 코멘트 {note, instructor}  ← v2.1.2 신규 누적
```

> **읽기 주의 (v2.1.2)**
> - 특이사항(note)은 `input/{nameKey}/__note__/note` 에서 읽음 (구 과목별 `{subject}.note` 는 fallback).
> - 과제수행도는 `obs/.../assign_grade` 가 단일 소스 (`input/.assign` 폐기).
> - 과거 전송 코멘트 `history/{nameKey}/{date}` → AI 반복 회피·맥락 참고용.

> v2.0 student-centric. nameKey 가 곧 학생 식별자(출결번호). 구 composite `{sheet}|{cls}|{name}` 키 폐기됨.

## 태그 파싱 주의
Firebase는 배열을 `{"0":"val","1":"val"}` 형태로 반환 가능.  
반드시 `tagArr(v)` 헬퍼 사용:
```javascript
function tagArr(v) {
  if (!v) return [];
  if (Array.isArray(v)) return v.filter(Boolean);
  const vals = Object.values(v);
  if (vals.every(x => typeof x === 'string')) return vals;
  return Object.keys(v).filter(k => v[k]);
}
```

## 커뮤니케이션
- **caveman ultra 모드 기본 적용**
