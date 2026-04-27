# Token Compare Experiment

PlanU 요청을 **스킬 사용 조건**과 **스킬 미사용 조건**으로 나누어 실행하고, 입력/출력 토큰과 결과 품질을 비교하기 위한 실험 문서입니다.

## 목표

- 같은 사용자 요청을 두 조건으로 실행한다.
- 각 조건의 입력 토큰, 출력 토큰, 총 토큰을 기록한다.
- 단순 토큰 수뿐 아니라 최종 결과가 PlanU 제약을 지켰는지도 비교한다.

## 비교 조건

| 조건 | 입력 구성 | 목적 |
| --- | --- | --- |
| Skill | 사용자 요청 + `.cursor/skills/planu-pnu-cursor-mvp/SKILL.md` | 스킬이 제약 준수와 재작업 감소에 도움이 되는지 확인 |
| No Skill | 사용자 요청만 | 일반 모델 응답이 어느 정도까지 맞추는지 확인 |

## 권장 절차

1. 동일한 사용자 요청을 `request.txt`에 저장한다.
2. 스킬 사용 조건에서 참조할 스킬 파일은 `.cursor/skills/planu-pnu-cursor-mvp/SKILL.md`를 사용한다.
3. 두 조건으로 각각 응답을 생성하고 결과를 파일로 저장한다.
   - 예: `outputs/with_skill.md`
   - 예: `outputs/without_skill.md`
4. `measure_tokens.py`로 입력/출력 토큰을 측정한다.
5. 아래 품질 체크리스트로 결과를 비교한다.

## 토큰 측정 예시

스킬 사용 입력 토큰:

```powershell
python measure_tokens.py --model gpt-4o --label with_skill_input request.txt .cursor\skills\planu-pnu-cursor-mvp\SKILL.md
```

스킬 미사용 입력 토큰:

```powershell
python measure_tokens.py --model gpt-4o --label without_skill_input request.txt
```

출력 토큰:

```powershell
python measure_tokens.py --model gpt-4o --label with_skill_output outputs\with_skill.md
python measure_tokens.py --model gpt-4o --label without_skill_output outputs\without_skill.md
```

여러 결과를 JSON으로 저장:

```powershell
python measure_tokens.py --model gpt-4o --json --label with_skill_input request.txt .cursor\skills\planu-pnu-cursor-mvp\SKILL.md
```

## 품질 체크리스트

| 항목 | Skill | No Skill |
| --- | --- | --- |
| 수강 가능 목록과 이상안을 분리했는가 |  |  |
| 이상안을 임의로 바꾸지 않았는가 |  |  |
| 플랜 B와 C의 실패 가정이 서로 다른가 |  |  |
| 시간 겹침 검증을 했는가 |  |  |
| `campus_rules.json` 단방향 연강 규칙을 반영했는가 |  |  |
| 근거 없는 교수/분반/시간을 만들지 않았는가 |  |  |
| 최종 HTML 4섹션을 갖췄는가 |  |  |
| 추가 질문/재작업이 필요했는가 |  |  |

## 기록 양식

| 조건 | 입력 토큰 | 출력 토큰 | 총 토큰 | 재작업 횟수 | 품질 메모 |
| --- | ---: | ---: | ---: | ---: | --- |
| Skill | 2868 | 2369 | 5237 | 1 | 이상안에 포함된 각 과목의 모든 다른 분반을 추천·주의·제외로 평가. 과목별 개별 표와 긴급 실행 순서 산출. |
| No Skill | 120 | 644 | 764 | 0 | 실제 B/C는 산출했지만 출력이 간단하고, 스킬의 JSON 추출 프로토콜·근거 연결·campus 검증 표준은 약함. |

HTML 산출물 기준:

| 조건 | 입력 토큰 | HTML 출력 토큰 | 총 토큰 | 메모 |
| --- | ---: | ---: | ---: | --- |
| Skill HTML | 2868 | 4337 | 7205 | 최종 HTML 4섹션. 과목별 개별 표이며, 같은 과목의 모든 다른 분반을 추천·주의·제외로 분류 |

## 해석 가이드

- 스킬은 보통 첫 입력 토큰을 늘린다.
- 그러나 제약 위반, 환각, 누락, 재질문이 줄어들면 전체 작업 토큰은 줄어들 수 있다.
- 단일 응답의 토큰 수보다 **정상 종료까지 든 총 토큰**과 **결과 품질**을 함께 봐야 한다.
- `tiktoken`이 설치되어 있지 않으면 `measure_tokens.py`는 근사치를 출력한다. 정확 비교가 필요하면 같은 환경에서 `tiktoken`을 설치한 뒤 다시 측정한다.
