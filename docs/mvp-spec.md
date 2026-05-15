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
    target-validation.json
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

Target의 `file`과 `lineRange`는 `headCommit` 기준이다. `lineRange`는 1-based, inclusive 좌표계를 사용한다. 예를 들어 `{"start":3,"end":11}`은 headCommit의 해당 파일에서 3번째 줄부터 11번째 줄까지를 포함한다.

`baseCommit`과 `headCommit` 사이의 변경 범위는 revision range가 아니라 tree diff다. P0에서 `baseCommit`과 `headCommit`의 비교는 `git diff <baseCommit> <headCommit>` 의미로 정의한다. merge-base 기준 diff는 P0 범위에 포함하지 않는다.

P0에서 rename 추적은 지원하지 않는다. `file`은 `headCommit` 기준 경로여야 하며, rename 때문에 target을 로컬 diff와 안정적으로 대조할 수 없으면 검증 실패로 처리한다.

P0는 자동 fetch를 수행하지 않는다. `baseCommit` 또는 `headCommit`이 local git object database에 없으면 검증 실패로 처리하고 사용자가 직접 fetch하도록 안내한다.

Target, replay, card ID는 충돌을 피하기 위해 ULID 또는 그에 준하는 정렬 가능한 고유 ID를 사용한다.

- `target_<ULID>`
- `replay_<ULID>`
- `card_<ULID>`

사용자가 직접 파일을 만들더라도 같은 ID 규칙을 따른다. 파일명은 사람이 읽기 쉽게 만들 수 있지만, 참조 안정성은 파일명이 아니라 frontmatter 또는 event에 기록된 ID가 담당한다.

### 이벤트 타입

P0에서 지원하는 이벤트 타입은 세 가지다.

- `target_added`: 새 target을 추가한다.
- `target_dismissed`: 기존 target을 비활성화한다.
- `target_superseded`: 기존 target이 새 target으로 대체됐음을 기록한다.

### target_added

`target_added`는 replay 대상으로 사용할 변경 범위를 추가한다.

```jsonl
{"type":"target_added","targetId":"target_01HZY6XQJ7N4M2P8K9S3V5T6W1","baseCommit":"abc123","headCommit":"def456","file":"src/auth/session.ts","lineRange":{"start":42,"end":57},"provenance":{"source":"user","label":"agent-assisted"},"priority":"normal","createdAt":"2026-05-15T10:00:00Z"}
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

`priority` 값은 P0에서 다음 세 가지를 사용한다. 생략하면 `normal`로 처리한다.

- `high`
- `normal`
- `low`

`provenance.source`는 non-empty source slug다. P0는 특정 도구 목록을 allowed set으로 고정하지 않고, `user` 또는 외부 연동이 제공한 안정적인 lower-case slug를 허용한다. Code Replay는 이 값을 보고 agent-assisted 여부를 추론하지 않는다.

### target_dismissed

`target_dismissed`는 target을 현재 replay 대상에서 제외한다.

```jsonl
{"type":"target_dismissed","targetId":"target_01HZY6XQJ7N4M2P8K9S3V5T6W1","reason":"not-worth-reviewing","createdAt":"2026-05-15T10:20:00Z"}
```

삭제는 파일 rewrite가 아니라 event append로 처리한다.

### target_superseded

`target_superseded`는 기존 target이 다른 target으로 대체됐음을 기록한다.

```jsonl
{"type":"target_superseded","targetId":"target_01HZY6XQJ7N4M2P8K9S3V5T6W1","supersededBy":"target_01HZY70Z6P6Q2T7Y9M4N8R1K3A","reason":"replace-with-wider-range","createdAt":"2026-05-15T10:30:00Z"}
```

수정은 기존 event를 바꾸지 않는다. 새 `target_added`를 append하고 기존 target을 `target_superseded`로 연결한다.

`target_added.note`는 target 자체에 대한 사용자 메모리이고, `target_superseded.reason`은 대체 이벤트의 사유다.

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
      "targetId": "target_01HZY70Z6P6Q2T7Y9M4N8R1K3A",
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

`active-targets.json`은 lifecycle 상태만 반영한다. `target_dismissed` 또는 `target_superseded`가 적용된 target은 빠지고, 검증 실패한 target은 lifecycle상 active라면 남는다.

정렬은 deterministic해야 한다. P0 정렬 순서는 다음과 같다.

1. `priority`: `high`, `normal`, `low`
2. `createdAt`: 오래된 순
3. `targetId`: 오름차순

동일 파일에서 line range가 겹치는 active target은 P0에서 허용한다. Code Replay는 이를 자동 dedupe하거나 실패 처리하지 않는다.

## Target 검증

Code Replay가 검증하는 것은 AI 관여 여부가 아니다. 검증 범위는 target이 로컬 repo의 실제 diff와 맞는지에 한정한다.

P0 검증 항목:

- 현재 위치가 git repo인지 확인한다.
- `baseCommit`과 `headCommit`이 local git object database에 존재하는지 확인한다.
- `file`이 `git diff <baseCommit> <headCommit>` 변경 범위에 포함되는지 확인한다.
- `lineRange`가 `headCommit` 기준 1-based, inclusive 좌표계로 유효한지 확인한다.
- `lineRange`가 해당 tree diff의 변경 범위와 겹치는지 확인한다.
- 동일한 `targetId`가 active 상태로 중복되지 않는지 확인한다.

검증 실패 시 해당 target은 replay/card 생성 대상에서 제외한다. CLI는 실패 요약을 출력하고, 마지막 검증 결과를 다음 파일에 기록한다.

```txt
.codereplay/cache/target-validation.json
```

이 파일은 source of truth가 아니다. `targets.jsonl`과 현재 git 상태에서 다시 만들 수 있는 진단용 cache다. 이 파일은 마지막 검증 실행의 snapshot이며 이력 로그가 아니다. 같은 target이 fetch 전에는 invalid였다가 fetch 후 valid가 되면 다음 검증 snapshot에서 최신 결과로 덮어쓴다.

Validation 대상은 lifecycle상 active인 target이다. `dismissed` 또는 `superseded` target은 `target-validation.json`에 `superseded` 같은 status로 남기지 않는다. Lifecycle 상태는 `targets.jsonl`을 fold해 얻고, validation cache의 `status`는 `valid` 또는 `invalid`만 사용한다.

예상 형태:

```json
{
  "version": 1,
  "validatedAt": "2026-05-15T10:40:00Z",
  "results": [
    {
      "targetId": "target_01HZY70Z6P6Q2T7Y9M4N8R1K3A",
      "status": "valid"
    },
    {
      "targetId": "target_01HZY8C4D8R6P5K2J7N3M9Q1T0",
      "status": "invalid",
      "reason": "missing-head-commit"
    }
  ]
}
```

P0에서 보장하는 invalid `reason` slug는 다음과 같다.

- `not-a-git-repo`
- `missing-base-commit`
- `missing-head-commit`
- `file-not-in-diff`
- `rename-not-supported`
- `line-range-invalid`
- `line-range-outside-diff`
- `duplicate-target-id`

`lineRange`와 변경 범위의 겹침은 head-side changed line과 하나 이상 교차하면 통과한다. P0는 line range가 diff hunk를 완전히 포함해야 한다고 요구하지 않는다.

검증 실패 target은 lifecycle상 active로 남는다. 다만 `make`는 validation status가 `invalid`인 target을 skip한다. 사용자는 같은 학습 의도의 정정 target을 추가할 때 `target_superseded`를 권장하고, 더 이상 처리하지 않을 잘못된 target은 `target_dismissed`를 권장한다. Code Replay는 invalid target을 자동 dismiss하지 않는다.

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
- reference pointer
- answer snapshot
- 생성 시각

Replay task는 spaced review 큐에 직접 들어가지 않는다. 이후 작은 review card의 근거가 된다.

Reference pointer는 다음 정보를 포함한다.

- `headCommit`
- `file`
- `lineRange`

Pointer는 원래 정답 위치를 설명하기 위한 참조다. rebase, squash, git gc, 후속 코드 변경으로 pointer가 깨질 수 있으므로 P0 replay task는 target의 정답에 해당하는 최소 `answerSnapshot`을 함께 저장한다.

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

P0 card는 Markdown 파일이며, 필수 metadata는 YAML frontmatter에 둔다.

```md
---
cardId: card_01HZY9D63M5E7Q8T2R4N6P1K0A
targetId: target_01HZY70Z6P6Q2T7Y9M4N8R1K3A
replayId: replay_01HZY8Y2W9C4S6N7D3M5K2R1Q0
type: fill_blank
conceptId: concept_session_expiry_boundary
verified: false
status: draft
createdAt: 2026-05-15T10:50:00Z
fsrs:
  state: new
  due: null
  stability: null
  difficulty: null
  reps: 0
  lapses: 0
---
```

필수 frontmatter 필드:

- `cardId`
- `targetId`
- `replayId`
- `type`
- `conceptId`
- `verified`
- `status`
- `createdAt`
- `fsrs`

`status` 값은 P0에서 다음을 사용한다.

- `draft`: 생성됐지만 검증되지 않은 card
- `active`: 검증되어 due 계산 대상이 될 수 있는 card
- `dismissed`: 폐기된 card

`verified: true`, `status: active`, 연결된 `targetId`가 lifecycle상 active인 조건을 모두 만족하는 card만 due 계산에 포함한다.

`conceptId`는 `concept_<slug>` 형태를 사용한다. `<slug>`는 lower-case ASCII, 숫자, `_`로 구성한다. 생성기가 초안을 제안할 수 있지만, card가 `active`가 되기 전에 사용자가 수정할 수 있어야 한다. P0는 concept 자동 merge나 concept graph를 제공하지 않는다.

잘못 생성된 card는 파일 삭제로 처리하지 않는다. card frontmatter의 `status`를 `dismissed`로 바꾸고, `review-log.jsonl`에 `card_dismissed` 이벤트를 append한다.

## Review log

복습 결과는 append-only JSONL로 기록한다.

P0 저장 위치:

```txt
.codereplay/reviews/review-log.jsonl
```

Review log는 card 원문을 대체하지 않는다. 사용자가 언제 어떤 card를 풀었고 어떤 결과였는지 기록한다.

예시:

```jsonl
{"type":"card_reviewed","cardId":"card_01HZY9D63M5E7Q8T2R4N6P1K0A","rating":"good","reviewedAt":"2026-05-16T09:00:00Z"}
```

P0의 rating 값은 FSRS와 호환되는 네 가지로 고정한다.

- `again`
- `hard`
- `good`
- `easy`

Card 폐기는 같은 로그에 이벤트로 남긴다.

```jsonl
{"type":"card_dismissed","cardId":"card_01HZY9D63M5E7Q8T2R4N6P1K0A","reason":"incorrect-answer","dismissedAt":"2026-05-16T09:10:00Z"}
```

## Target lifecycle and artifacts

Target lifecycle 변경은 기존 replay/card 파일을 자동 수정하지 않는다.

`target_dismissed` 또는 `target_superseded`가 발생하면:

- 기존 replay task는 historical artifact로 남긴다.
- 기존 card 파일은 자동으로 dismiss하지 않는다.
- 기존 card를 새 target으로 자동 reattach하지 않는다.
- due 계산은 lifecycle상 active target에 연결된 card만 포함한다.

따라서 superseded target에 연결된 card는 frontmatter가 `verified: true`, `status: active`여도 due review 대상에서 제외된다. 사용자가 해당 card를 계속 쓰고 싶다면 새 target 기준으로 표준 생성 흐름을 다시 통과하거나, 별도로 card를 수정한 뒤 검증해야 한다.

P0는 supersede 시점에 사용자에게 card 처리 방식을 묻는 interactive migration을 제공하지 않는다.

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

### target validate

`active-targets.json`의 lifecycle-active target을 로컬 git 상태와 대조하고 `.codereplay/cache/target-validation.json`을 최신 snapshot으로 덮어쓴다.

### target ls

`active-targets.json`과 `target-validation.json`을 결합해 현재 target 상태를 보여준다. 최소 출력에는 `targetId`, `file`, `lineRange`, `priority`, validation `status`, invalid `reason`이 포함돼야 한다.

### make

active target에서 replay task와 review card 후보를 만든다.

P0에서는 한 실행에서 1-3개 수준의 replay/card 생성을 기본으로 한다.

`make`는 생성 전에 target validation을 다시 실행하고 `target-validation.json`을 갱신한다. Validation status가 `invalid`인 target은 생성에서 skip하고, CLI 출력에 skip 개수와 reason 요약을 보여준다.

### review

due 상태의 verified card를 사용자에게 제시하고 결과를 `review-log.jsonl`에 append한다. `review`는 target validation을 다시 실행하지 않지만, due 계산 시 연결된 target이 lifecycle상 active인지 확인한다.

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

Fast mode는 미리보기나 passive preview가 아니라 retrieval drill이다. 사용자는 답안을 입력해야 하며, 입력 없이 정답만 보여주는 흐름은 fast mode로 취급하지 않는다.

Fast mode 결과는 다음 제약을 가진다.

- 실행 전에 target validation을 거친다.
- Validation status가 `invalid`인 target에서는 fast mode를 실행하지 않는다.
- P0에서 fast mode는 `fill_blank` 타입만 다룬다.
- P0에서 fast mode는 한 번에 문항 1개만 만든다.
- review card로 자동 승격하지 않는다.
- spaced review 일정에 넣지 않는다.
- P0에서는 CLI 출력과 메모리 안에서만 유지하고 `.codereplay/cards/`에 저장하지 않는다.
- 사용자가 fast mode를 실행했다는 사실은 P0에서 `review-log.jsonl`이나 다른 `.codereplay/` 산출물에 기록하지 않는다.
- 사용자가 저장 가능한 card로 만들려면 표준 생성 흐름을 다시 통과해야 한다.

Fast mode와 standard mode는 같은 target에서 항상 같은 문항을 만든다고 보장하지 않는다. P0는 deterministic generation을 제품 계약으로 제공하지 않는다.

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
- target 좌표는 `headCommit` 기준 1-based, inclusive `lineRange`로 해석한다.
- diff 검증은 `git diff <baseCommit> <headCommit>` 기준으로 수행한다.
- local에 없는 commit을 자동 fetch하지 않는다.
- ID는 `target_<ULID>`, `replay_<ULID>`, `card_<ULID>` 형태를 따른다.
- `priority`는 `high`, `normal`, `low` 중 하나이며 기본값은 `normal`이다.
- `targets.jsonl`은 append-only로 유지한다.
- `active-targets.json`은 rebuild 가능해야 한다.
- `active-targets.json`은 lifecycle-active target을 deterministic order로 담는다.
- `target-validation.json`은 lifecycle-active target의 마지막 검증 snapshot을 기록하는 rebuildable cache다.
- validation `status`는 `valid` 또는 `invalid`만 사용한다.
- invalid target은 active cache에 남을 수 있지만 `make`에서 skip한다.
- `target ls`와 `make`는 validation reason을 사용자에게 보여준다.
- `verified: true`, `status: active`, active target 연결을 모두 만족하는 card만 due review 대상이 된다.
- `conceptId`는 `concept_<slug>` 형식을 따른다.
- review rating은 `again`, `hard`, `good`, `easy` 중 하나다.
- card 폐기는 파일 삭제가 아니라 `status: dismissed`와 `card_dismissed` 이벤트로 기록한다.
- target dismiss/supersede는 기존 replay/card를 자동 수정하거나 reattach하지 않는다.
- fast mode 결과는 P0에서 `.codereplay/cards/`에 저장하지 않는다.
- fast mode는 valid target에서만 실행하고, 사용자 답안 입력을 요구한다.
- fast mode는 P0에서 `fill_blank` 1문항만 생성한다.
- fast mode 실행 사실은 P0에서 `.codereplay/` 산출물에 기록하지 않는다.
- P0는 fast/standard generation의 deterministic output을 보장하지 않는다.
- 사용자는 원본 코드를 보기 전에 review card를 풀 수 있다.
- Code Replay는 agent-assisted 여부를 탐지했다는 표현을 UI/CLI/문서에 쓰지 않는다.
