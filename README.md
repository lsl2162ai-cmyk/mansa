# mansa — 개인 작업 기록 (lsl2162ai-cmyk)

> 원본 저장소: [HYUNDAI-AI-team6/mansa](https://github.com/HYUNDAI-AI-team6/mansa)
> 팀 프로젝트 README: [TEAM_README.md](TEAM_README.md)
> 이 문서는 팀 프로젝트에서 **내가 작성한 이슈·PR·커밋만** 정리한 것이다.

설비 매뉴얼·안전 문서를 근거로 답하는 기술 질의응답(L2) 서비스에서 **팀장**을 맡았다.
개발에서는 **질의응답 처리 흐름, 검색(Retriever), LLM 공통 함수, 질문 분석기**를 구현했고,
**발표 자료(PPT)와 결과보고서**를 작성했다.

## 팀장 · 산출물

**팀장**

- 일정 관리, 역할 분배, 회의 진행, PR 리뷰·머지 등 팀 운영 전반 담당

**발표 자료 (PPT)**

- 발표 자료 전체 작성 (발표는 다른 팀원이 진행)

**결과보고서**

- 프로젝트 결과보고서 작성

## 요약

| 항목 | 수 |
| --- | --- |
| 이슈 | 7 |
| PR (모두 머지) | 6 |
| 커밋 | 8 |
| 기간 | 2026-10-27 ~ 2026-10-29 |

## 주요 작업

### 1. 질의응답 핵심 처리 단계 — [#32](https://github.com/HYUNDAI-AI-team6/mansa/issues/32) / [PR #36](https://github.com/HYUNDAI-AI-team6/mansa/pull/36)

LLM·벡터 DB·MySQL 없이 동작하는 부분을 먼저 만들어 팀원이 맞출 기준을 정했다.

- `core/schemas.py`: 질의응답 공통 데이터 구조 (`Intent`, `ScoredChunk`, `Evidence`, `AnswerSection`, `AnswerDraft`, `Answer`, `Clarify`, `NoAnswer`)
- `core/exceptions.py`: `LlmError`
- `merge_evidence`: 매뉴얼·안전 검색 결과 병합, 중복 청크는 높은 점수만 유지, 상위 8개 선택
- `answer_guard.py`: `is_sufficient`(근거 충분성 판단), `build_no_answer`(사유별 답변 불가 안내)
- `attach_notice`: 안전 영역 맨 앞에 안전관리자 확인 문구를 코드로 부착
- `QAService.answer`: 질문 분석 → 범위 밖·되묻기 → 검색 → 근거 판단 → 생성·검증(최대 2회 재생성) → 안전 문구 → 기록 순서 조율. 단계별 입출력은 `Protocol`로 정의
- 단위 테스트 21개 작성·통과 (외부 서비스는 가짜 객체로 대체)

### 2. 안전 문서 하이브리드 검색 — [#41](https://github.com/HYUNDAI-AI-team6/mansa/issues/41) / [PR #43](https://github.com/HYUNDAI-AI-team6/mansa/pull/43)

- 검색 모듈을 `features/qa/` → `features/retrieval/`로 이동 (`bm25.py`, `rrf.py`, `retriever.py`)
- `search_safety` 1~4단계 구현
  1. 검색어 보강: 작업 부위·작업 내용을 질문에 덧붙임 (보강 함수는 교체 가능)
  2. 벡터 검색: `secure/safety_docs` 컬렉션 상위 20개
  3. BM25 검색: 안전 문서 전체를 500개씩 나눠 읽어 상위 20개
  4. RRF로 두 순위를 합쳐 최종 순서 결정
- 외부 호출 함수를 인자로 받아 설정·네트워크 없이 테스트 가능하게 설계

### 3. LLM 공통 함수 `core.llm`

**`embed`** — [#55](https://github.com/HYUNDAI-AI-team6/mansa/issues/55) / [PR #57](https://github.com/HYUNDAI-AI-team6/mansa/pull/57)

- 텍스트 목록 → 임베딩 벡터 목록, 모델 객체는 한 번 만들고 재사용
- `EMBEDDING_MODEL` 설정 분리 (기본 `text-embedding-3-large`, 3072차원)
- API 키가 없으면 호출 전 오류, 빈 입력은 API 호출 없이 빈 목록 반환
- 모델 변경(1536 → 3072차원)에 따른 컬렉션 재적재 필요성을 팀에 공유

**`complete`** — [#71](https://github.com/HYUNDAI-AI-team6/mansa/issues/71) / [PR #73](https://github.com/HYUNDAI-AI-team6/mansa/pull/73)

- `with_structured_output`으로 Pydantic 형식의 응답만 받아 `dict`로 반환
- 형식 오류·길이 잘림·시간 초과만 `LLM_MAX_RETRIES`번 재시도, 초과 시 원인 예외를 보존한 `LlmError`
- LLM 거부는 즉시 `LlmError`, 인증 실패 등은 그대로 전달
- SDK 자체 재시도를 꺼서 재시도 횟수를 정확히 제어
- `LLM_TIMEOUT`(30초), `LLM_MAX_RETRIES`(2회) 설정 추가

### 4. 질문 분석기 `QuestionAnalyzer`

**`validate_ids`** — [#80](https://github.com/HYUNDAI-AI-team6/mansa/issues/80) / [PR #86](https://github.com/HYUNDAI-AI-team6/mansa/pull/86)

- LLM이 뽑은 모델·부위·작업을 실제 등록된 기준정보와 대조해 없는 값 제거
- 동의어를 표준 태그로 변환 (예: "와이어로프" → "메인 로프")
- 허용되지 않은 의도는 `out_of_scope`로, 중복 제거, 입력 불변(새 `Intent` 반환)

**`analyze`** — [#88](https://github.com/HYUNDAI-AI-team6/mansa/issues/88) / [PR #94](https://github.com/HYUNDAI-AI-team6/mansa/pull/94)

- L2 질문 분석 프롬프트 작성: 의도 5종(사양·절차·안전·검사·범위 밖) 판단 기준, 등록 모델·부위·작업 목록 제공, 질문 원문은 지시가 아닌 데이터로 취급
- 대화 첫 질문일 때만 같은 호출에서 대화 제목 생성 (추가 호출 없음)
- LLM이 부위·작업을 빠뜨려도 질문 원문에서 분류표 이름을 찾아 보완
- 실제 OpenAI 결과로 발견한 `validate_ids` 문제 수정 (모델명 대괄호 차이, 표준 부위가 동의어 변환으로 지워지던 문제, 단위만 있는 수치 제거)

### 5. 태그 필터 벡터 검색 — [#104](https://github.com/HYUNDAI-AI-team6/mansa/issues/104)

- 부위·작업 태그가 유사도 상위 20개 밖에 있으면 검색에서 빠지는 문제를 찾아 이슈로 정리
- Chroma `where` 조건으로 태그 안에서 유사도 검색 후 RRF에 합치는 방향을 설계·검증
- 구현은 검색 담당 팀원의 [PR #123](https://github.com/HYUNDAI-AI-team6/mansa/pull/123)으로 반영됨

## 이슈 · PR 목록

| 이슈 | 제목 | PR | 머지일 |
| --- | --- | --- | --- |
| [#32](https://github.com/HYUNDAI-AI-team6/mansa/issues/32) | 질의응답 핵심 처리 단계 구현 | [#36](https://github.com/HYUNDAI-AI-team6/mansa/pull/36) | 2026-10-27 |
| [#41](https://github.com/HYUNDAI-AI-team6/mansa/issues/41) | 안전 문서 하이브리드 검색(search_safety) 구현 | [#43](https://github.com/HYUNDAI-AI-team6/mansa/pull/43) | 2026-10-27 |
| [#55](https://github.com/HYUNDAI-AI-team6/mansa/issues/55) | core.llm 임베딩 함수(embed) 추가 | [#57](https://github.com/HYUNDAI-AI-team6/mansa/pull/57) | 2026-10-28 |
| [#71](https://github.com/HYUNDAI-AI-team6/mansa/issues/71) | core.llm 구조화 응답 호출 함수(complete) 추가 | [#73](https://github.com/HYUNDAI-AI-team6/mansa/pull/73) | 2026-10-28 |
| [#80](https://github.com/HYUNDAI-AI-team6/mansa/issues/80) | QuestionAnalyzer.validate_ids 구현 | [#86](https://github.com/HYUNDAI-AI-team6/mansa/pull/86) | 2026-10-28 |
| [#88](https://github.com/HYUNDAI-AI-team6/mansa/issues/88) | QuestionAnalyzer.analyze 구현 (L2 질문 분석) | [#94](https://github.com/HYUNDAI-AI-team6/mansa/pull/94) | 2026-10-28 |
| [#104](https://github.com/HYUNDAI-AI-team6/mansa/issues/104) | 태그 필터를 건 벡터 유사도 검색 추가 | 팀원 [#123](https://github.com/HYUNDAI-AI-team6/mansa/pull/123) | — |

## 커밋 목록

| 날짜 | 커밋 | 내용 |
| --- | --- | --- |
| 2026-10-27 | [`9b0e438`](https://github.com/HYUNDAI-AI-team6/mansa/commit/9b0e438) | QA 서비스 구현 (근거 병합, 안전 문구 부착) |
| 2026-10-27 | [`0b5a8ae`](https://github.com/HYUNDAI-AI-team6/mansa/commit/0b5a8ae) | 로컬 전용 QA 테스트 파일 정리 |
| 2026-10-27 | [`cacf024`](https://github.com/HYUNDAI-AI-team6/mansa/commit/cacf024) | 검색 모듈 재구성 및 BM25 순위 구현 |
| 2026-10-28 | [`f5745b0`](https://github.com/HYUNDAI-AI-team6/mansa/commit/f5745b0) | 임베딩 모델 설정 및 `embed` 추가 |
| 2026-10-28 | [`eaa5e78`](https://github.com/HYUNDAI-AI-team6/mansa/commit/eaa5e78) | `EMBEDDING_MODEL` 기본값 처리 수정 |
| 2026-10-28 | [`82ba837`](https://github.com/HYUNDAI-AI-team6/mansa/commit/82ba837) | LLM 타임아웃·재시도 설정 및 `complete` 추가 |
| 2026-10-28 | [`b575a5e`](https://github.com/HYUNDAI-AI-team6/mansa/commit/b575a5e) | `QuestionAnalyzer` 의도 검증·정규화 구현 |
| 2026-10-28 | [`68a987a`](https://github.com/HYUNDAI-AI-team6/mansa/commit/68a987a) | `QuestionAnalyzer` 프롬프트·검증 보완 |

## 사용 기술

Python, uv, LangChain + OpenAI (structured output, embeddings), ChromaDB, BM25, RRF, Pydantic, unittest
