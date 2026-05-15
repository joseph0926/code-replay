# AI 시대의 개발 학습 복구 리서치

## 문서 역할

이 문서는 Code Replay의 초기 문제의식과 리서치 맥락을 정리하는 주제 문서다.

현재 확정된 제품 방향은 `docs/product-direction.md`, P0 실행 계약은 `docs/mvp-spec.md`를 기준으로 한다. 향후 `README.md`가 생기면 외부 소개, 빠른 시작, 사용법은 README가 맡고, 이 문서는 리서치 배경과 문서 지도를 유지한다.

## 초기 문제

AI agent가 코드 변경을 빠르게 만들수록 개발자가 그 변경을 자기 머리로 충분히 통과시키지 못하는 간극이 생길 수 있다. 결과물은 빨리 나오지만, 나중에 같은 변경을 직접 구현하거나 설명하려 할 때 막히는 문제가 남는다.

이 리서치는 그 간극을 줄이기 위해, AI 도움을 받은 내 실제 코드 변경을 사후에 다시 회수하고 재구성하게 만드는 제품 가능성을 검토한다.

## 현재 수렴한 방향

Code Replay는 명시적으로 전달받은 `agent-assisted` 변경 `target`을 `replay task`와 `review card`로 변환하는 `local-first` 학습 도구로 수렴했다.

중요한 경계는 다음과 같다.

- Code Replay는 AI가 관여한 변경을 자동으로 찾지 않는다.
- `agent-assisted`는 사용자나 외부 연동이 전달한 provenance 속성이다.
- Code Replay의 역할은 전달받은 target을 로컬 diff와 대조해 검증하고 학습 산출물로 변환하는 것이다.
- 기본 산출물은 각 프로젝트 repo의 `.codereplay/` 아래에 둔다.
- `.codereplay/`의 git tracking 여부는 사용자가 결정하며, P0의 기본 가이드는 local untracked 사용이다.

## 초기 MVP 후보와 현재 해석

초기 리서치에서 검토한 MVP 후보는 다음처럼 현재 방향에 흡수됐다.

1. "처음부터 다시 구현하기"는 초기 `replay task`로 남긴다.
2. 빈칸 채우기는 P0 `review card` 타입인 `fill_blank`로 남긴다.
3. 다음 복습 질문은 spaced review 큐에 들어가는 verified card로 남긴다.
4. 이해해야 할 개념 목록은 P0에서 단순 `concept` label/id로 축소한다.
5. 변경 요약은 단독 산출물이 아니라 replay/card 생성을 돕는 보조 맥락으로 둔다.
6. 개념 간 연결 맵은 P0에서 제외하고 장기 설계로 미룬다.

현재 P0의 자세한 범위는 `docs/mvp-spec.md`를 따른다.

## 차별점

Code Replay의 차별점은 새로운 단일 기술이 아니라 검증된 빌딩 블록을 개인 repo의 agent-assisted 변경 맥락에 묶는 조합에 있다.

- 입력은 개인 git repo의 명시적 target이다.
- provenance는 Code Replay가 추론하지 않고 사용자나 외부 연동이 제공한다.
- 변환 결과는 replay task와 spaced review card다.
- 저장은 repo-local `.codereplay/`를 기본으로 한다.
- 목표는 AI 사용 후 내 코드 변경을 다시 내 구현 감각으로 회수하는 것이다.

## 문서 지도

### 방향과 스펙

- `docs/product-direction.md` — 현재 제품 방향, 핵심 원칙, non-goals.
- `docs/mvp-spec.md` — P0 입력 계약, 캐시 계약, 산출물, CLI 흐름, 수용 기준.
- `docs/topic.md` — 초기 주제, 리서치 맥락, 문서 지도.

### 핵심 근거

- `docs/learning-science.md` — generation effect, retrieval practice, spaced repetition, Parsons, 코드 읽기와 쓰기 실증 근거.
- `docs/ai-learning-impact.md` — AI 사용과 학습 간극, cognitive offloading, structured unassisted practice 근거.
- `docs/alternative.md` — 인접 도구와 비어 있는 제품 조합.

### P0 설계 근거

- `docs/agent-integration.md` — provenance 수집, replay mode 권한 경계, agent 연동 전략.
- `docs/llm-quiz-hallucination.md` — LLM 생성 카드의 hallucination 방지와 검증 파이프라인.
- `docs/diff-to-curriculum.md` — diff를 학습 단위로 변환하는 기술 빌딩 블록과 gap analysis.
- `docs/ast-diff-tools.md` — 구조 diff, line range, hunk 처리 구현 근거.
- `docs/learning-metrics.md` — card retention, concept mastery, reconstruction, transfer metric 계층.

### 후속 또는 구현 참고

- `docs/concept-identity.md` — PR 간 같은 개념을 묶는 entity resolution과 active learning 설계.
- `docs/local-vs-api-llm.md` — local-first 모델 운영과 cloud opt-in 경계.
- `docs/quantization-impact.md` — 로컬 LLM 양자화가 카드 생성, judge, structured output에 주는 영향.

## 참고

- https://docs.github.com/en/get-started/learning-to-code/setting-up-copilot-for-learning-to-code
- https://www.trycursor.ai/
- https://codeteach.codes/
- https://exercism.org/
- https://codecrafters.io/
- https://github.com/codecrafters-io/build-your-own-x
- https://arxiv.org/abs/2507.09089
