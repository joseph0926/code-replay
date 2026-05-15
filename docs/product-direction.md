# Product Direction

## 한 줄 정의

Code Replay는 명시적으로 전달받은 `agent-assisted` 변경 `target`을 `replay task`와 `review card`로 변환하는 `local-first` 학습 도구다.

## 문제

AI agent는 코드 변경을 빠르게 만들 수 있지만, 개발자가 그 변경을 자기 머리로 충분히 통과시키는 속도는 그보다 느릴 수 있다.

Code Replay가 다루는 문제는 "코드 diff를 설명하기 어렵다"가 아니다. 핵심 문제는 AI agent의 도움으로 만들어진 내 코드 변경이 아직 내 장기 기억과 구현 감각으로 소화되지 않았을 수 있다는 점이다.

## 제품 명제

가장 중요한 개입은 요약, 해설, 리뷰가 아니라 원본 코드를 보기 전에 중요한 변경을 다시 회수하게 만드는 것이다.

Code Replay는 사용자가 명시적으로 전달한 변경을 작은 재구현 문제와 복습 카드로 바꾼다. 목표는 더 많은 설명을 제공하는 것이 아니라, 사용자가 자신의 변경을 나중에 다시 구현하거나 설명할 수 있게 만드는 것이다.

## 핵심 원칙

### Local-first

Code Replay의 기본 산출물은 각 프로젝트 repo의 `.codereplay/` 아래에 저장한다. 전역 저장소는 사용자 설정, 캐시, 여러 repo를 가로지르는 인덱스처럼 재생성 가능하거나 민감도가 낮은 데이터에 한정한다.

Cloud/API 모델 사용은 명시적 opt-in이어야 한다. 기본 전제는 사용자의 코드, diff, 학습 기록이 로컬에 남는 것이다.

### Provenance-first

Code Replay는 어떤 변경이 AI agent의 도움을 받았는지 탐지하거나 추론하지 않는다. `agent-assisted`는 사용자의 명시적 표시 또는 외부 연동이 남긴 `provenance`로만 주어지는 입력 속성이다.

Code Replay의 역할은 항목을 찾는 것이 아니라, 전달받은 항목을 검증하고 학습 산출물로 변환하는 것이다.

### Replay-first

Code Replay의 중심 경험은 설명을 읽는 것이 아니라 원본을 숨긴 상태에서 다시 구현하거나 핵심 결정을 다시 설명하는 것이다.

`replay task`는 처음에 정답 코드를 숨기고 변경을 다시 구현하게 하는 큰 문제다. `review card`는 이후 spaced review 큐에서 다시 푸는 작은 복습 단위다.

### Quality-first

많은 카드를 만드는 것보다 검증된 카드를 적게 만드는 것이 중요하다. 검증되지 않은 card는 spaced review 큐에 저장하지 않는다.

`fast mode`를 제공한다면 즉석 연습 모드로 취급한다. 빠르게 문제를 만들어 바로 풀 수는 있지만, 검증이 끝나기 전에는 review card로 저장하거나 spaced review 일정에 넣지 않는다. 장기 복습 큐에는 표준 검증을 통과한 card만 들어간다.

### Small queue-first

한 diff에서 모든 변경을 학습 대상으로 만들지 않는다. P0는 한 실행에서 사용자가 선택했거나 입력 manifest가 우선순위로 표시한 target을 중심으로 1-3개 수준의 replay/card를 만드는 방향을 따른다.

여기서 "핵심"은 Code Replay가 전체 diff에서 중요 변경을 스스로 찾는다는 뜻이 아니다. 하나의 target 안에서 원본을 보지 않고 다시 회수해야 하는 구현 결정, 경계 조건, 불변식처럼 작은 학습 포인트를 뜻한다.

### Minimal product-first

P0는 CLI와 repo-local 파일 저장소로 시작한다. Dashboard, IDE extension, code knowledge graph, 자동 채점, cross-repo 분석은 제품 신호가 충분해진 뒤에 확장한다.

## 핵심 객체

### Target

`target`은 사용자나 외부 연동이 Code Replay에 명시적으로 전달한 replay 대상 변경이다. Code Replay는 target이 로컬 diff와 일치하는지 검증하지만, 그 target이 agent-assisted인지 판단하지 않는다.

### Agent-assisted Change

`agent-assisted change`는 Code Replay가 발견한 속성이 아니다. 사용자의 명시적 표시 또는 도구 연동이 남긴 provenance로 전달된 변경이다.

### Replay Task

`replay task`는 정답 코드를 숨긴 상태에서 사용자가 변경의 핵심 구현을 다시 작성하거나 재구성하는 문제다.

### Review Card

`review card`는 spaced review 큐에 들어가는 검증된 복습 문제다. P0의 card는 `fill_blank`와 `explain_decision`처럼 작은 retrieval 단위에 집중한다.

### Concept

`concept`는 여러 replay task와 card를 묶을 수 있는 학습 단위 이름이다. P0에서는 복잡한 concept graph보다 안정적인 label/id를 우선한다.

## 입력 경계

Code Replay는 replay 대상을 발견하지 않는다. 사용자나 외부 연동이 명시적으로 전달한 target만 입력으로 받는다.

모든 연동은 동일한 입력 계약으로 수렴해야 한다. Core는 target이 로컬 diff와 일치하는지 검증하지만, 해당 변경이 agent-assisted인지 판단하지 않는다.

Target 이력과 현재 활성 상태는 분리한다.

- `.codereplay/input/targets.jsonl`은 append-only source of truth다.
- `.codereplay/cache/active-targets.json`은 core workflow가 빠르게 읽기 위한 재생성 가능한 read model이다.

`targets.jsonl`은 전체 replace 대상이 아니다. target 삭제는 `dismissed`, 수정은 `superseded` 계열 이벤트로 기록한다.

간단한 target 입력 예시는 다음과 같다.

```jsonl
{"type":"target_added","targetId":"target-001","baseCommit":"abc123","headCommit":"def456","file":"src/auth/session.ts","lineRange":{"start":42,"end":57},"provenance":{"source":"user","label":"agent-assisted"},"createdAt":"2026-05-15T10:00:00Z"}
```

이 입력은 Code Replay에게 AI 관여 여부를 판단해 달라는 요청이 아니다. 사용자나 외부 연동이 이 변경 범위를 replay 대상으로 전달했다는 기록이며, Core는 commit, file, line range가 로컬 diff와 맞는지만 검증한다.

## P0 제품 형태

P0는 하나의 로컬 repo에서 명시적으로 전달받은 agent-assisted target을 replay 학습 큐로 바꾸는 최소 흐름이다.

- 입력은 git diff 또는 commit range와 명시적 target manifest다.
- 산출물은 repo-local `.codereplay/`에 저장한다.
- target history는 append-only JSONL로 남긴다.
- active target state는 재생성 가능한 cache로 둔다.
- reconstruction은 초기 replay task로 다룬다.
- `fill_blank`와 `explain_decision`은 review card의 기본 타입이다.
- verified card만 spaced review 큐에 들어간다.
- full agent transcript, prompt, raw cloud response는 기본 저장 대상이 아니다.

P0의 성공 기준은 하나의 명시적 agent-assisted target에서 검증된 replay/card 큐를 만들고, 사용자가 나중에 원본 코드를 보기 전에 핵심 구현 또는 결정을 다시 회수하는 것이다.

## Non-goals

Code Replay는 다음을 P0 목표로 삼지 않는다.

- AI가 작성한 코드를 자동으로 찾아내는 도구
- 일반 git diff 학습 도구
- PR review bot
- 팀 온보딩 도구
- 오픈소스 diff 해설기
- 코드 스타일 기반 authorship classifier
- full code knowledge graph
- dashboard-first 학습 제품
- VSCode-first 제품
- cloud-first 제품
- agent transcript 저장 시스템
- 모든 변경을 자동으로 커리큘럼화하는 도구

## 금지 표현

제품 문서와 CLI 표현에서 Code Replay가 AI 관여 여부를 스스로 찾는 것처럼 보이는 표현은 피한다.

피해야 할 표현:

- detect agent-written code
- discover AI changes
- infer authorship
- classify AI code
- find agent-assisted hunks

선호 표현:

- receive provided targets
- import target manifest
- validate target against local diff
- transform targets into replay tasks
- generate verified review cards

## 제품 베팅

사용자의 실제 agent-assisted diff에서 나온 소수의 고품질 replay prompt는 넓은 요약, 일반 개념 지도, 범용 퀴즈보다 더 오래 남는 학습을 만들 가능성이 높다.

Code Replay는 AI coding assistant가 아니다. Post-AI coding learning system이다.
