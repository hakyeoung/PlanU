---
name: planu-pnu-cursor-mvp
description: PlanU PNU v1 — 3단계: ①편람→JSON ②사용자 이상안·조건 인터뷰 ③대체 후보 랭킹·긴급 실행표·HTML. campus_rules.json(단방향) 연강 필터. 트리거: PlanU / 부산대 / 수강신청 / seed.yaml.
---

# PlanU (PNU) — Cursor MVP v1

## 목표

- 수강 가능 목록 + 사용자 이상안 → **과목별 대체 후보 우선순위** + **긴급 실행 순서** → HTML 출력.
- 이상안은 시스템이 새로 짜지 않는다. 근거만 사용, 추정으로 덮어쓰지 않는다.
- 전체 수강 가능 목록은 내부 후보 풀로만 사용한다. 최종 HTML에는 사용자 행동에 필요한 대체 후보와 제외/주의 항목만 노출한다.

## 3단계 파이프라인

### Stage 1 — 편람 → JSON
파일(PDF/HWP/XLSX) 수신 → `_planu_extract.py` 기반 파싱 → `available_courses_catalog` JSON 저장 후 종료.  
이 단계에서 다른 작업 없음. 파싱 실패 시 오류 + 다른 형식 권고 후 종료.

#### Data Extraction Protocol — 초대용량 파일 처리

- 수강편람 전체 데이터를 채팅/컨텍스트에 직접 로드하지 않는다. 토큰 초과 방지를 위해 정제된 JSON만 사용한다.
- 워크스페이스 루트의 `_planu_extract.py`는 PlanU 보조 실행 스크립트로 함께 버전 관리되는 파일이다. 있으면 우선 사용한다.
- `_planu_extract.py`가 없거나 현재 파일 구조와 맞지 않으면, 먼저 로컬 파일에서 헤더와 첫 3~5줄만 소량 검사해 `_planu_extract.py`를 자동 작성·보완한다. 이 소량 검사는 전체 데이터 직접 로드가 아니라 스크립트 부트스트랩 단계로 허용한다.
- 로컬 파일 접근이나 구조 자동 파악이 실패할 때만, 사용자에게 **헤더(열 이름)와 첫 3~5줄 샘플**을 채팅에 붙여넣어 달라고 요청한다.
- 자동 검사 또는 사용자 샘플을 바탕으로, 과목명·분반·요일·교시/시각·건물/강의실·담당교수 등 필요한 필드만 추출해 JSON으로 출력하는 `_planu_extract.py`를 작성·저장한다.
- `_planu_extract.py`가 준비되어 있고 사용자가 새 수강편람 파일을 지정하면, 파일 내용을 직접 읽지 말고 터미널에서 `python _planu_extract.py <파일명>`을 실행한다.
- 이후 작업은 스크립트의 stdout으로 반환된 정제 JSON만 `available_courses_catalog`에 담아 진행한다.
- 스크립트 실행 실패, 헤더 불명확, 필수 필드 누락 시 임의 추정하지 말고 오류·샘플 재요청·다른 형식 권고 중 하나로 종료하거나 보완한다.

#### `available_courses_catalog` JSON schema

`_planu_extract.py`는 stdout으로 아래 형태의 JSON만 출력한다. 필드명이 흔들리면 Stage 2·3에서 재사용하기 어렵기 때문에, 원본 열 이름이 달라도 아래 키로 정규화한다.

```json
{
  "source_file": "catalog.xlsx",
  "extracted_at": "YYYY-MM-DDTHH:mm:ss",
  "courses": [
    {
      "course_id": "미확인",
      "course_name": "과목명",
      "section": "분반",
      "credits": null,
      "day": "월",
      "period": "1-2",
      "start_time": null,
      "end_time": null,
      "building": "건물명 또는 미확인",
      "room": "강의실 또는 미확인",
      "instructor": "담당교수 또는 미확인",
      "campus_area": "미확인",
      "source_row": 2,
      "evidence": "원본 파일명:행/시트 등"
    }
  ],
  "warnings": []
}
```

- `courses`는 배열이며, 각 원소는 수업 1개 또는 동일 과목의 시간 블록 1개를 뜻한다. 한 과목이 여러 요일/교시에 걸치면 원본 구조에 맞춰 여러 행으로 분리해도 된다.
- 필수 키: `course_name`, `section`, `day`, `period`, `building`, `room`, `instructor`, `source_row`, `evidence`.
- 값이 없으면 키를 생략하지 말고 문자열 `"미확인"` 또는 `null`을 넣는다. 임의 생성 금지.
- 시각 정보가 원본에 있으면 `start_time`, `end_time`을 `HH:MM`으로 채운다. 없으면 `period`를 유지하고 Stage 3에서 75분 가정을 명시한다.
- `campus_area`는 원본이나 `campus_rules.json` 매핑으로 확정 가능할 때만 채운다. 불명확하면 `"미확인"`으로 둔다.

### Stage 2 — 사용자 조건 인터뷰
Stage 1 JSON 로드 후 순서대로 질문:
1. 최소·최대 이수 학점
2. 반드시 수강할 과목 또는 이상안에 포함된 과목
3. 선호·기피 교수
4. 가능 시간대·공강 요건
5. 이상적 시간표 (분반·요일·교시)

출력: `user_profile` + `preferences` + `must_take_courses` + `user_ideal_timetable` 저장.

### Stage 3 — 대체 후보 랭킹 → HTML
Stage 1·2 산출물 로드 후 순서 실행:

1. **Core** — Hard/Soft 분류, 이상안 과목별 대체 후보 탐색 범위 확정.
2. **campus_route_validation** — `campus_rules.json` 단방향 적용. 연강 구간(≤ 20분)에서 이동 불가 조합 탐색:
   - 이상안에 불가 조합 → **경고 + 확인 요청** (이상안 수정 금지)
   - 대체 후보에 불가 조합 → **제외 또는 주의**
   - 판정: 구역 차 ≥ 3 → 기본 불가 / rule_tree from→to 매칭 → can_connect 따름
3. **schedule_validation** — 이상안과 각 대체 후보 적용 시 시간 겹침 검사 (교시당 75분, 근거 있으면 우선).
4. **Candidate coverage** — 사용자 이상안에 포함된 각 과목에 대해, catalog에 있는 **같은 과목의 모든 다른 분반**을 반드시 평가한다. 원래 선택한 분반은 이상안에만 표시하고 대체 후보에서는 제외한다.
5. **Ranking** — 과목별 대체 후보를 순위화한다. 같은 과목/다른 분반을 우선하고, 사용자 기피 조건·시간 겹침·연강 불가·강의 수 변경 후보는 제외/주의 처리한다. 추천하지 않는 후보도 누락하지 말고 `추천`·`주의`·`제외` 중 하나로 분류한다.
6. **Organizer** — `html_report` 합성. 섹션 순서: 이상안 → 대체 후보 우선순위 → 긴급 실행 순서 → 제외·주의 목록.

## Non-negotiable constraints

- 근거 없는 주장 금지 (catalog·시간·분반·교수 = 파일 + 채팅 근거만).
- 초대용량 수강편람은 직접 컨텍스트에 적재 금지. `_planu_extract.py` stdout의 정제 JSON만 사용.
- 이상안 불변 — 불가 조합 발견 시 경고·확인만, 수정 금지.
- rule_tree는 **단방향** (from→to 명시 방향만, 역방향 자동 불가 아님).
- 최종 HTML에 전체 수강 가능 목록을 그대로 덤프하지 않는다. 필요한 근거는 대체 후보 행 안에만 연결한다.
- 이상안에 포함된 각 과목의 **모든 다른 분반**은 반드시 평가한다. 추천 후보만 골라 보여주지 않는다. 각 후보는 `추천`·`주의`·`제외` 중 하나로 판정하고 사유를 쓴다.
- 4섹션 모두 있어야 성공 종료: 이상안·대체 후보 우선순위·긴급 실행 순서·제외/주의 목록.
- 비공식 참고용 고지 필수.
- Streamlit 등 앱 코드는 요청 시에만.

## HTML 필수 항목

- 면책·가정 (75분 교시 사용 시 명시).
- **§ 이상적 시간표** — 요일별 표 (교시·과목·분반·담당교수) + 겹침 검증 + campus_route_validation 요약.
- **§ 대체 후보 우선순위** — 원래 과목별로 별도 소표를 만든다. 각 과목 블록은 원래 분반/시간, 후보 표, 짧은 판단 메모를 포함한다. 후보 표 행은 `순위 | 대체 후보 | 시간 | 담당 | 시간 겹침 | 동선 | 상태 | 근거`를 포함한다.
- 각 과목 블록에는 catalog에 존재하는 같은 과목의 모든 다른 분반이 등장해야 한다. 후보가 추천 불가라면 표에서 `제외`로 표시하고 제외 사유를 쓴다.
- **§ 긴급 실행 순서** — 수강신청 현장에서 누를 순서. 예: `자료구조 061 실패 → 자료구조 062 → 없으면 보류/질문`.
- **§ 제외·주의 목록** — 사용자 기피, 시간 겹침, 연강 불가, 근거 부족, 강의 수 변경 등으로 제외/주의 처리한 후보.
- campus_route_validation 요약 형식: `수업A건물 → 수업B건물 | 구역차 | can_connect | 근거`.
- 근거 부족 셀 → **미확인**, 임의 생성 금지.

## Ontology

| Key | Role |
|-----|------|
| `available_courses_catalog` | 개설 강의 목록 (Stage 1 출력) |
| `user_ideal_timetable` | 사용자 이상안 (불변 입력) |
| `must_take_courses` | 병목 과목 / B·C 실패 시나리오 대상 |
| `user_profile` | 학과·학년·학점 |
| `preferences` | 공강·최대수업·교수 등 Soft 조건 |
| `evidence_sources` | 파일 경로·채팅 인용 |
| `extracted_corpus` | 파싱 결과 |
| `graduation_audit` | 졸업 요건 참고 |
| `course_candidates` | 대체 후보 필터 결과 |
| `schedule_validation` | 이상안·대체 후보 적용 시 시간 겹침 검사 |
| `campus_route_validation` | 연강 이동 가능 여부 (단방향) |
| `replacement_rankings` | 과목별 대체 후보 우선순위 |
| `emergency_action_order` | 수강신청 현장 실행 순서 |
| `html_report` | 최종 HTML |

## Quality bar

- 4섹션 존재; 이상안 / 대체 후보 우선순위 / 긴급 실행 순서 / 제외·주의 목록.
- 이상안 미변경 (사용자 승인 수정 제외).
- 추천 후보에 연강 불가 조합 없음 (campus_rules.json 단방향 적용).
- 이상안 내 불가 조합은 경고만, 수정 없음.
- 사용자 기피 조건과 시간 겹침 후보는 추천 랭킹에서 제외하거나 주의로 명확히 표시.
- 같은 과목의 다른 분반 누락 없음. 모든 후보가 추천·주의·제외 중 하나로 설명됨.
- 전체 수강 가능 목록 대신 행동 가능한 후보만 노출.

## Exit conditions

- **성공**: 4섹션 html_report 전달. 이상안 재현 + 과목별 대체 후보 랭킹 + 긴급 실행 순서 + 제외/주의 목록.
- **오류 종료**: 파일 추출 실패 → 메시지 + 종료.
- **v2 연기**: 상호평가·경쟁률·교수스타일 분석.

## 참조

- `seed.yaml` v2.2.0 — 권위 있는 제약 소스.
- `campus_rules.json` — 연강 판단 rule_engine (단방향).
