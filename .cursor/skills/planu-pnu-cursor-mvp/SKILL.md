---
name: planu-pnu-cursor-mvp
description: PlanU PNU v1 — user supplies ideal timetable + enrollable course catalog; AI builds HTML with validated ideal and Plan B/C fallbacks for specific registration failures. Trigger with PlanU, seed.yaml, 부산대, 수강신청, etc.
---

# PlanU (PNU) — Cursor MVP v1

## 제품 초점 (설계 재설정)

- **목표 아님**: 시스템이 처음부터 “이상적” 시간표를 새로 짜는 것(학생은 보통 이미 본인 안이 있음).
- **목표**: **`available_courses_catalog`**(수강 가능·개설 목록) + **`user_ideal_timetable`**(사용자 이상안)을 받은 뒤, **특정 과목 수강신청 실패**를 가정한 **플랜 B·플랜 C**(대체 시간표·행동 힌트)를 생성한다.
- **원칙**: 근거만 사용하고, 사용자 이상안을 **추정으로 덮어쓰지 않는다**.

## Product focus (design reset) — same as above

- **Not the goal**: Generating a student’s “ideal” timetable from scratch — students usually **already have** one.
- **The goal**: After **`available_courses_catalog`** and **`user_ideal_timetable`** are in place, produce **Plan B** and **Plan C** for **specific registration failures**, plus conflict checks and action hints.
- **Truth**: Evidence-only; do not overwrite the user’s ideal with guesses.

## When to apply

Use when the user invokes **PlanU v1** (e.g. message starts with **`PlanU`**, or mentions 부산대 / PNU / 수강신청 / `seed.yaml`). Read **`seed.yaml`** for constraints and exit rules.

## Non-negotiable constraints

- **Evidence**: Catalog, times, sections, instructors from **user files + explicit chat** only; no fabrication.
- **Disclaimer**: Non-official; school rules win.
- **Scope**: Sequential pipeline + organizer HTML only (no peer-evaluation loops, no competition prediction, no professor-style analytics).
- **Extraction failure**: Clear error, suggest another format, stop (no auto-OCR loop).
- **Required inputs before Plan B·C**:
  1. **수강 가능한 강의 목록** — normalized from syllabus/catalog files into `available_courses_catalog` (or equivalent).
  2. **사용자 이상적 시간표** — `user_ideal_timetable`; **do not replace** with an invented “better” schedule.
  If either is missing or ambiguous, **ask** or request files before generating B/C.
- **Plan B & C**: Two **separate `<section>`**s; each must state **which course(s) failed** (assumption), show **weekday tables** (교시/시간, 과목명, 분반, 담당 교수 or “미확인”), and **schedule_validation** (overlap + 75min rule if used). **B and C must use different failure assumptions** (no duplicate scenario).
- **User ideal section**: Own `<section>`; render the user’s timetable; run **overlap validation** — if conflicts, **warn** and ask for correction; do not silently “fix” the user’s ideal.
- **Catalog section**: Own `<section>` (or clear appendix) with list summary + **evidence paths**.
- **Time overlap**: Check each of ideal, B, C internally. **75-minute block** per period when times are inferred; prefer explicit times from evidence; state the assumption in HTML.
- **Sections for success**: **Four** blocks — (1) catalog (2) user ideal (3) Plan B (4) Plan C — all in `html_report`; omitting B or C is **not** a successful run.
- **MCP**: Notion / maps optional; paste or files if absent.
- **Streamlit**: Only if the user asks.

## Input handling

1. Collect **catalog files** (PDF/HWP/XLSX…) and **user’s ideal timetable** (file and/or structured paste).
2. Normalize with appropriate tools; record `evidence_sources` / `extracted_corpus`.
3. If **available_courses_catalog** or **user_ideal_timetable** is incomplete → **gate**: questions / file requests before strategy.
4. Clarify **which failure scenarios** Plan B vs Plan C should assume (tie to `must_take_courses` or user-stated bottlenecks).
5. Soft prefs (공강, max classes/day, etc.) — ask if needed to choose **replacements** for B/C.

## Pipeline order (strict)

1. **Core**: Parse catalog → `available_courses_catalog`; parse and preserve **user ideal** → `user_ideal_timetable`; Hard/Soft; **failure scenarios** for B vs C; `must_take` / bottlenecks.
2. **PNU route** (optional): Only if building/location data exists.
3. **Strategy**: Validate ideal vs catalog; **`schedule_validation`** for ideal; generate **Plan B** and **Plan C** only (alternatives from catalog, no invented sections); **`schedule_validation`** per plan.
4. **Organizer**: One **`html_report`** — recommended order: **catalog → user ideal → Plan B → Plan C**; save e.g. `planu_report.html` when possible.

## Final HTML — required content

- Disclaimer + assumptions (incl. **75분 교시** if used).
- **§ 수강 가능 강의 목록** — table or summary + evidence references.
- **§ 이상적 시간표 (사용자 입력)** — weekday `<table>`s; 분반·담당 교수 columns; **overlap check** summary.
- **§ 플랜 B** — failure assumption text; tables; overlap summary; registration order hints (no fake probabilities).
- **§ 플랜 C** — **different** failure assumption; same table/validation standards.
- Risk, open questions, file paths.
- If evidence is insufficient for a cell → **미확인**, never invent.

## Ontology (working objects)

| Key | Role |
|-----|------|
| `user_profile` | 학과, 학년, 학점 등 |
| `available_courses_catalog` | 개설·수강 가능 목록(근거 기반) |
| `user_ideal_timetable` | 사용자 이상안(불변 입력으로 취급) |
| `must_take_courses` | 꼭 넣을 과목·병목; 실패 시나리오와 연결 |
| `preferences` | 대체 선택용 Soft |
| `preference_clarification_rounds` | 질문-답변 로그 |
| `evidence_sources` | 인용·경로 |
| `extracted_corpus` | 추출 결과 |
| `graduation_audit` | 참고용 졸업 판단 |
| `course_candidates` | 목록 필터·후보 |
| `schedule_validation` | **이상안 + 플랜 B + 플랜 C** 각각 겹침 검사 |
| `route_assessment` | 동선(선택) |
| `timetable_plans` | **플랜 B·C** 생성 결과만(이상안은 `user_ideal_timetable`) |
| `html_report` | 최종 HTML |
| `optional_code_bundle` | 요청 시 |

## Quality bar (self-check)

- Catalog and ideal are **sourced**, not invented.
- **Four sections** present; B ≠ C failure story.
- Ideal unchanged except **user-approved** fixes.
- Overlap checks + 75min note where relevant.
- No single “merged” plan instead of B and C.

## Exit conditions

- **Success**: `html_report` delivered with **four** distinct parts: **enrollable list**, **user ideal**, **Plan B**, **Plan C**; B and C each have failure assumption + tables + validation; **different** assumptions for B vs C.
- **Error exit**: Extraction failure → message + stop.
- **Defer**: v2 features per `seed.yaml`.

## Additional resources

- **`seed.yaml`** is authoritative (currently **v2.0.0** design).
