# MVP Spec

## 목적

이 문서는 Code Replay P0가 실제로 동작하기 위해 지켜야 할 최소 제품 계약을 정의한다.

상위 방향성은 `docs/product-direction.md`를 따른다. 이 문서는 철학을 반복하지 않고, 입력, 저장, 검증, 산출물, CLI 흐름의 경계를 고정한다.

## P0 성공 기준

P0는 하나의 로컬 git repo에서 다음 흐름을 완주하면 성공이다.

1. 사용자가 replay 대상 target을 명시적으로 전달한다.
2. Code Replay가 target이 로컬 diff와 맞는지 검증한다.
3. 유효한 target에서 replay task와 review card 후보를 만든다.
4. 검증된 card만 spaced review 큐에 저장한다.
5. 사용자는 이후 원본 코드를 보기 전에 review card를 다시 풀 수 있다.

P0는 target을 자동으로 찾지 않는다. 중요 변경을 전체 diff에서 자동 선별하지도 않는다.

## Repo-local 저장소

P0의 기본 산출물은 대상 프로젝트 repo 안의 `.codereplay/` 아래에 저장한다.

```txt
.codereplay/
  input/
    targets.jsonl
  cache/
    active-targets.json
  replays/
    *.md
  cards/
    *.md
  reviews/
    review-log.jsonl
  config.json
```

기본 정책은 repo-local이다. 전역 저장소는 P0 구현 범위에 포함하지 않는다.

`.codereplay/`를 git에 커밋할지는 사용자가 결정한다. P0의 기본 가이드는 로컬 untracked 사용이다.

## Target 입력 계약

Code Replay는 replay 대상을 발견하지 않는다. 사용자가 CLI로 추가하거나 외부 연동이 manifest를 생성해 전달한 target만 처리한다.

Target event의 source of truth는 다음 파일이다.

```txt
.codereplay/input/targets.jsonl
```

이 파일은 append-only JSONL 이벤트 로그다. 전체 replace를 기본 동작으로 삼지 않는다.

### 이벤트 타입

P0에서 지원하는 이벤트 타입은 세 가지다.

- `target_added`: 새 target을 추가한다.
- `target_dismissed`: 기존 target을 비활성화한다.
- `target_superseded`: 기존 target이 새 target으로 대체됐음을 기록한다.

### target_added

`target_added`는 replay 대상으로 사용할 변경 범위를 추가한다.

```jsonl
{"type":"target_added","targetId":"target-001","baseCommit":"abc123","headCommit":"def456","file":"src/auth/session.ts","lineRange":{"start":42,"end":57},"provenance":{"source":"user","label":"agent-assisted"},"priority":"normal","createdAt":"2026-05-15T10:00:00Z"}
```

필수 필드:

- `type`
- `targetId`
- `baseCommit`
- `headCommit`
- `file`
- `lineRange.start`
- `lineRange.end`
- `provenance.source`
- `provenance.label`
- `createdAt`

선택 필드:

- `priority`
- `note`
- `diffFingerprint`

`baseCommit`과 `headCommit`은 branch name보다 commit SHA를 우선한다.

### target_dismissed

`target_dismissed`는 target을 현재 replay 대상에서 제외한다.

```jsonl
{"type":"target_dismissed","targetId":"target-001","reason":"not-worth-reviewing","createdAt":"2026-05-15T10:20:00Z"}
```

삭제는 파일 rewrite가 아니라 event append로 처리한다.

### target_superseded

`target_superseded`는 기존 target이 다른 target으로 대체됐음을 기록한다.

```jsonl
{"type":"target_superseded","targetId":"target-001","supersededBy":"target-002","createdAt":"2026-05-15T10:30:00Z"}
```

수정은 기존 event를 바꾸지 않는다. 새 `target_added`를 append하고 기존 target을 `target_superseded`로 연결한다.

## Active target cache

현재 활성 target 상태는 다음 파일에 저장한다.

```txt
.codereplay/cache/active-targets.json
```

이 파일은 source of truth가 아니다. `targets.jsonl`에서 언제든지 재생성 가능한 read model이다.

예상 형태:

```json
{
  "version": 1,
  "rebuiltAt": "2026-05-15T10:31:00Z",
  "source": ".codereplay/input/targets.jsonl",
  "targets": [
    {
      "targetId": "target-002",
      "baseCommit": "abc123",
      "headCommit": "def456",
      "file": "src/auth/session.ts",
      "lineRange": {
        "start": 42,
        "end": 61
      },
      "provenance": {
        "source": "user",
        "label": "agent-assisted"
      },
      "priority": "normal",
      "createdAt": "2026-05-15T10:30:00Z"
    }
  ]
}
```

캐시 규칙:

- `active-targets.json`은 직접 수정 대상이 아니다.
- 캐시가 없으면 `targets.jsonl`에서 rebuild한다.
- 캐시가 `targets.jsonl`보다 오래됐으면 rebuild한다.
- 캐시가 깨졌으면 실패하지 말고 rebuild를 시도한다.
- rebuild할 수 없으면 target 입력 오류로 중단한다.

## Target 검증

Code Replay가 검증하는 것은 AI 관여 여부가 아니다. 검증 범위는 target이 로컬 repo의 실제 diff와 맞는지에 한정한다.

P0 검증 항목:

- 현재 위치가 git repo인지 확인한다.
- `baseCommit`과 `headCommit`이 존재하는지 확인한다.
- `file`이 `baseCommit..headCommit` 변경 범위에 포함되는지 확인한다.
- `lineRange`가 해당 변경 범위와 겹치는지 확인한다.
- 동일한 `targetId`가 active 상태로 중복되지 않는지 확인한다.

검증 실패 시 해당 target은 replay/card 생성 대상에서 제외하고 오류를 보고한다.

## Replay task

`replay task`는 원본 정답을 숨긴 상태에서 target의 핵심 구현을 다시 작성하게 하는 초기 문제다.

P0 저장 위치:

```txt
.codereplay/replays/*.md
```

P0 replay task는 다음 정보를 포함해야 한다.

- `replayId`
- 연결된 `targetId`
- source file
- base/head commit
- prompt
- hidden reference 또는 reference pointer
- 생성 시각

Replay task는 spaced review 큐에 직접 들어가지 않는다. 이후 작은 review card의 근거가 된다.

## Review card

`review card`는 spaced review 큐에 들어가는 작은 복습 단위다.

P0 저장 위치:

```txt
.codereplay/cards/*.md
```

P0 card 타입:

- `fill_blank`: 코드 일부를 비워 두고 다시 채운다.
- `explain_decision`: 조건, 경계, 불변식, 설계 선택의 이유를 설명한다.

Card는 최소한 다음 정보를 포함해야 한다.

- `cardId`
- 연결된 `targetId`
- 연결된 `replayId`
- `type`
- `concept`
- prompt
- expected answer 또는 answer rubric
- `verified`
- spaced review metadata

`verified: false`인 card는 저장될 수 있지만 spaced review due 계산에는 포함하지 않는다.

## Review log

복습 결과는 append-only JSONL로 기록한다.

P0 저장 위치:

```txt
.codereplay/reviews/review-log.jsonl
```

Review log는 card 원문을 대체하지 않는다. 사용자가 언제 어떤 card를 풀었고 어떤 결과였는지 기록한다.

예시:

```jsonl
{"type":"card_reviewed","cardId":"card-001","rating":"good","reviewedAt":"2026-05-16T09:00:00Z"}
```

## 최소 CLI 흐름

명령 이름은 구현 시 바뀔 수 있지만, P0 흐름은 다음 기능을 포함해야 한다.

### init

`.codereplay/` 기본 디렉터리와 config를 만든다.

### target add

사용자가 file, commit range, line range를 지정해 target을 추가한다.

이 명령은 AI 관여 여부를 판단하지 않는다. 사용자가 제공한 provenance를 기록한다.

### target import

외부 연동이 만든 JSONL target event를 `.codereplay/input/targets.jsonl`에 append한다.

### target rebuild-cache

`targets.jsonl`에서 `active-targets.json`을 재생성한다.

### make

active target에서 replay task와 review card 후보를 만든다.

P0에서는 한 실행에서 1-3개 수준의 replay/card 생성을 기본으로 한다.

### review

due 상태의 verified card를 사용자에게 제시하고 결과를 `review-log.jsonl`에 append한다.

## 품질 게이트

P0는 single-pass LLM 생성 결과를 곧바로 spaced review 큐에 넣지 않는다.

Verified card가 되려면 최소한 다음 조건을 만족해야 한다.

- 연결된 target이 유효하다.
- prompt가 target 밖의 맥락 없이는 풀 수 없는 과도한 문제로 생성되지 않았다.
- expected answer 또는 rubric이 target과 충돌하지 않는다.
- card 타입이 P0 지원 타입에 속한다.

검증 방식은 구현 단계에서 결정하되, 검증 전 card는 due 계산에 포함하지 않는다.

## Fast mode

P0에서 fast mode를 제공한다면 즉석 연습용으로만 사용한다.

Fast mode 결과는 다음 제약을 가진다.

- review card로 자동 승격하지 않는다.
- spaced review 일정에 넣지 않는다.
- 사용자가 별도로 검증하거나 표준 생성 흐름을 통과해야 저장 가능한 card가 된다.

## Non-goals

P0는 다음을 하지 않는다.

- AI가 작성한 코드 자동 탐지
- 코드 스타일 기반 authorship 추론
- 전체 diff에서 중요 변경 자동 선별
- full agent transcript 저장
- dashboard 제공
- IDE extension 제공
- cross-repo due 통합
- reconstruction 자동 채점
- concept graph 구축
- cloud/API 모델 기본 사용

## 수용 기준

P0 구현은 다음을 만족해야 한다.

- 명시적 target 없이 replay/card를 생성하지 않는다.
- 유효하지 않은 target은 active workflow에서 제외한다.
- `targets.jsonl`은 append-only로 유지한다.
- `active-targets.json`은 rebuild 가능해야 한다.
- verified card만 due review 대상이 된다.
- 사용자는 원본 코드를 보기 전에 review card를 풀 수 있다.
- Code Replay는 agent-assisted 여부를 탐지했다는 표현을 UI/CLI/문서에 쓰지 않는다.
