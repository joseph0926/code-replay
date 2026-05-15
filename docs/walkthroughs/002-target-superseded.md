# Walkthrough 002 — target을 superseded로 대체하는 흐름

작성 일자: 2026-05-15
스코프: 한 번 추가한 target이 적합하지 않아 더 넓은 범위로 다시 잡는 케이스를 mvp-spec.md에 손으로 통과시킨다. append-only event log, `active-targets.json` rebuild, 이전 replay/card 처리 방식을 검증한다. 코드 구현은 하지 않는다.

## 1. 출발 상태

Walkthrough 001과 같은 가상 변경(`src/auth/session.ts`의 `isExpired`에 grace period 추가)을 그대로 사용한다.

`head` 상태 파일 구조(추정):

```ts
// line 1
import type { Session } from "./types";
// line 2 (빈 줄)
const GRACE_MS = 30_000;
// line 4 (빈 줄)
export function isExpired(session: Session, now: number): boolean {
  if (session.revokedAt !== null && session.revokedAt <= now) {
    return true;
  }
  return session.expiresAt + GRACE_MS <= now;
}
// line 11에서 함수 끝
```

사용자는 처음 walkthrough 001에서 `lineRange: {start: 5, end: 11}`로 함수 본문만 target으로 잡았다(원본 001 문서의 3-11은 함수 위 import까지 포함된 값이었지만, 본 walkthrough는 함수 본문만으로 좁힌 케이스를 가정한다).

복습을 만들어 본 뒤 사용자는 "상수 선언인 `GRACE_MS = 30_000` 줄도 학습 대상이어야 한다"고 판단한다. 함수 안 `+ GRACE_MS` 식과 상수 자체가 한 묶음이어야 `fill_blank`에서 상수 값을 묻는 카드가 자연스러워진다.

## 2. 첫 번째 target

`targets.jsonl`에 이미 들어가 있다고 가정.

```jsonl
{"type":"target_added","targetId":"target_01HZY70Z6P6Q2T7Y9M4N8R1K3A","baseCommit":"abc123","headCommit":"def456","file":"src/auth/session.ts","lineRange":{"start":5,"end":11},"provenance":{"source":"user","label":"agent-assisted"},"priority":"normal","createdAt":"2026-05-15T10:00:00Z"}
```

이 target에서 `replay_01HZY8Y2W9C4S6N7D3M5K2R1Q0`와 `card_01HZY9D63M5E7Q8T2R4N6P1K0A` (fill_blank, `verified: true`, `status: active`)가 이미 만들어졌다고 가정한다.

## 3. 두 번째 target과 supersede 이벤트

사용자가 더 넓은 범위로 다시 추가한다. `target_added` 한 줄을 새로 append한다.

```jsonl
{"type":"target_added","targetId":"target_01J00A1H7E5T8N2K4Q6R8M3W2V","baseCommit":"abc123","headCommit":"def456","file":"src/auth/session.ts","lineRange":{"start":3,"end":11},"provenance":{"source":"user","label":"agent-assisted"},"priority":"normal","note":"include GRACE_MS constant","createdAt":"2026-05-15T11:00:00Z"}
```

이어서 첫 target을 superseded로 연결한다.

```jsonl
{"type":"target_superseded","targetId":"target_01HZY70Z6P6Q2T7Y9M4N8R1K3A","supersededBy":"target_01J00A1H7E5T8N2K4Q6R8M3W2V","createdAt":"2026-05-15T11:00:01Z"}
```

`targets.jsonl`은 다음 세 줄을 누적 보유한다.

```jsonl
{"type":"target_added","targetId":"target_01HZY70Z6P6Q2T7Y9M4N8R1K3A", ...}
{"type":"target_added","targetId":"target_01J00A1H7E5T8N2K4Q6R8M3W2V", ...}
{"type":"target_superseded","targetId":"target_01HZY70Z6P6Q2T7Y9M4N8R1K3A","supersededBy":"target_01J00A1H7E5T8N2K4Q6R8M3W2V", ...}
```

스펙 §`target_superseded`의 규칙("수정은 기존 event를 바꾸지 않는다. 새 `target_added`를 append하고 기존 target을 `target_superseded`로 연결한다")을 그대로 적용했다.

## 4. Active target cache rebuild

`targets.jsonl`을 처음부터 다시 읽어 `active-targets.json`을 만든다고 가정.

기대 결과:

```json
{
  "version": 1,
  "rebuiltAt": "2026-05-15T11:00:05Z",
  "source": ".codereplay/input/targets.jsonl",
  "targets": [
    {
      "targetId": "target_01J00A1H7E5T8N2K4Q6R8M3W2V",
      "baseCommit": "abc123",
      "headCommit": "def456",
      "file": "src/auth/session.ts",
      "lineRange": { "start": 3, "end": 11 },
      "provenance": { "source": "user", "label": "agent-assisted" },
      "priority": "normal",
      "createdAt": "2026-05-15T11:00:00Z"
    }
  ]
}
```

첫 target은 superseded 처리되어 active 목록에서 빠진다. 단일 active target만 남는다.

rebuild 알고리즘(추정):

1. 모든 `target_added` 이벤트를 ID별로 모은다.
2. 같은 ID에 `target_dismissed` 또는 `target_superseded`가 있으면 제외한다.
3. 남은 active target을 `createdAt` 순으로 정렬해 `targets` 배열에 둔다.

## 5. Target 검증

새 target의 검증 항목(스펙 §Target 검증)을 손으로 통과시킨다.

- 현재 위치가 git repo → OK.
- `baseCommit`/`headCommit`이 local git object database에 존재 → OK 가정.
- `file`이 `git diff <baseCommit> <headCommit>` 변경 범위에 포함 → 같은 파일 같은 변경이므로 OK.
- `lineRange`가 `headCommit` 기준 1-based, inclusive로 유효 → 3-11은 11줄 파일 안. OK.
- `lineRange`가 tree diff 변경 범위와 겹침 → diff hunk가 3-11을 포함. OK.
- 동일 `targetId`가 active 상태로 중복되지 않음 → 새 ULID라 중복 없음. OK.

검증 통과 → `target-validation.json`에 결과 entry를 둘 다 기록한다고 가정.

```json
{
  "version": 1,
  "validatedAt": "2026-05-15T11:00:06Z",
  "results": [
    {
      "targetId": "target_01HZY70Z6P6Q2T7Y9M4N8R1K3A",
      "status": "superseded"
    },
    {
      "targetId": "target_01J00A1H7E5T8N2K4Q6R8M3W2V",
      "status": "valid"
    }
  ]
}
```

여기서 첫 의문이 생긴다 → §발견된 스펙 빈 곳 1.

## 6. 새 replay task

`replay_01J00BR2N9M5C8K2T7Y4Q6S3P1`로 가정. frontmatter는 다음을 따른다.

```yaml
---
replayId: replay_01J00BR2N9M5C8K2T7Y4Q6S3P1
targetId: target_01J00A1H7E5T8N2K4Q6R8M3W2V
sourceFile: src/auth/session.ts
baseCommit: abc123
headCommit: def456
createdAt: 2026-05-15T11:01:00Z
referencePointer:
  headCommit: def456
  file: src/auth/session.ts
  lineRange: { start: 3, end: 11 }
answerSnapshot: |
  const GRACE_MS = 30_000;

  export function isExpired(session: Session, now: number): boolean {
    if (session.revokedAt !== null && session.revokedAt <= now) {
      return true;
    }
    return session.expiresAt + GRACE_MS <= now;
  }
---
```

`answerSnapshot`이 추가됐다(스펙 §Replay task). pointer가 깨져도 정답 본문을 복원할 수 있다.

## 7. 첫 target에 묶여 있던 card 처리

이게 본 walkthrough의 핵심 미해결 질문이다.

`card_01HZY9D63M5E7Q8T2R4N6P1K0A`는 frontmatter에 다음을 가진다.

```yaml
targetId: target_01HZY70Z6P6Q2T7Y9M4N8R1K3A
replayId: replay_01HZY8Y2W9C4S6N7D3M5K2R1Q0
type: fill_blank
status: active
verified: true
```

이제 첫 target이 superseded됐다. 이 카드는 어떻게 되어야 하는가? 후보 처리:

- **A. 그대로 둔다.** card는 여전히 `status: active`이고 due 계산에 들어간다. supersede는 target 수준 사건이라 card에 자동 전파되지 않는다.
- **B. 자동 dismiss.** target이 superseded되면 그 target에 묶인 모든 card를 `card_dismissed` 이벤트로 자동 종료한다.
- **C. 자동 reattach.** card의 `targetId`를 supersededBy로 자동 갱신한다.
- **D. 사용자 결정으로 미룬다.** CLI가 "이 target에 묶인 card 3개가 있습니다. 어떻게 할까요?"라고 물어 사용자가 선택한다.

스펙은 이 동작을 명시하지 않는다 → §발견된 스펙 빈 곳 2.

본 walkthrough는 가장 단순한 A(그대로 둔다)를 추정해서 진행한다. 단, 새 target에서 만들 새 card와 같은 `conceptId`를 가질 수 있으므로 concept mastery 합산(P1) 시점에서는 같은 그룹에 들어간다.

## 8. 새 card

새 target의 더 넓은 범위에서 한 장을 더 만든다고 가정.

```yaml
---
cardId: card_01J00C9T4K8E2P7M5N1Q3R6W0A
targetId: target_01J00A1H7E5T8N2K4Q6R8M3W2V
replayId: replay_01J00BR2N9M5C8K2T7Y4Q6S3P1
type: fill_blank
conceptId: concept_session_expiry_boundary
verified: true
status: active
createdAt: 2026-05-15T11:02:00Z
fsrs:
  state: new
  due: null
  stability: null
  difficulty: null
  reps: 0
  lapses: 0
---

# fill_blank — GRACE_MS 상수 값

```ts
const GRACE_MS = ____;
```

## expected

`30_000`
```

이전 card(`card_01HZY9D...`)와 같은 `conceptId`를 사용한다. 사용자가 의도적으로 같은 개념의 카드로 묶었다고 가정.

## 9. Review log

이전 card와 새 card를 차례로 풀었다고 가정.

```jsonl
{"type":"card_reviewed","cardId":"card_01HZY9D63M5E7Q8T2R4N6P1K0A","rating":"good","reviewedAt":"2026-05-16T09:00:00Z"}
{"type":"card_reviewed","cardId":"card_01J00C9T4K8E2P7M5N1Q3R6W0A","rating":"again","reviewedAt":"2026-05-16T09:01:00Z"}
```

rating은 새 스펙의 FSRS 4단계(`again/hard/good/easy`)를 따른다.

## 10. 수용 기준 점검

스펙 §수용 기준 확장본을 본 walkthrough가 위반하지 않는지 확인.

- 명시적 target 없이 생성 안 함 → OK.
- 유효하지 않은 target은 active에서 제외 → 첫 target은 superseded로 active에서 빠짐. OK.
- `headCommit` 기준 1-based, inclusive → OK.
- diff 검증은 `git diff <baseCommit> <headCommit>` → OK.
- local에 없는 commit 자동 fetch 안 함 → 본 walkthrough는 fetch 시도 없음. OK.
- ID는 `target_<ULID>`/`replay_<ULID>`/`card_<ULID>` → OK.
- `targets.jsonl` append-only → 세 줄 모두 append. 기존 줄 수정 없음. OK.
- `active-targets.json` rebuild 가능 → §4에서 재구성. OK.
- `target-validation.json`은 rebuildable cache → §5에서 재구성. OK.
- `verified: true` + `status: active`만 due 대상 → 두 card 모두 만족. OK.
- `conceptId`는 `concept_<slug>` 형식 → `concept_session_expiry_boundary`. OK.
- rating은 4단계 중 하나 → OK.
- card 폐기는 file 삭제 아님 → 본 walkthrough는 card 폐기 없음. 해당 사항 없음.
- fast mode 결과는 `.codereplay/cards/`에 저장 안 함 → 본 walkthrough는 fast mode 없음. 해당 사항 없음.
- 원본 보기 전 review card 풀 수 있음 → OK.
- agent-assisted 탐지 표현 없음 → OK.

## 발견된 스펙 빈 곳

1. **`target-validation.json`의 `status` 값 집합 미정.** 본 walkthrough는 `valid`, `invalid`, `superseded`를 가정해 적었다. 그런데 스펙 예시는 `valid` / `invalid`만 보여준다. superseded는 검증 통과 여부 자체가 아닌 라이프사이클 상태라 같은 필드를 공유해야 하는지 모호하다. `superseded`/`dismissed`는 validation 대상 자체에서 빠져야 하나, 진단용으로 결과에 같이 보여줄지 결정 필요.
2. **target supersede가 발생할 때 묶여 있던 card/replay 처리 규칙 부재.** 위 §7에서 A/B/C/D 네 후보를 적었다. 어느 동작이 P0 default인지 스펙에 없다. 사용자가 같은 변경의 더 넓은 범위로 supersede하는 경우와 완전히 다른 변경으로 갈아끼우는 경우는 다르게 처리해야 할 수도 있다.
3. **`active-targets.json` rebuild 순서/정렬 규칙 부재.** 스펙은 "재생성 가능"이라고만 적었다. 사용자가 priority 정렬을 기대할 수도 있고, createdAt 역순을 기대할 수도 있다. CLI `make` 명령이 어떤 순서로 target을 처리하는지에 직접 영향. 본 walkthrough는 createdAt 순 추정.
4. **`note` 필드의 사용 범위 부재.** `target_added`의 선택 필드 `note`를 본 walkthrough에서 supersede 사유로 활용했다(`"note":"include GRACE_MS constant"`). 그런데 supersede 사유는 `target_superseded` 이벤트에 별도 `reason` 필드로 둘 수도 있다. 둘 중 어디에 사유를 남기는 게 권장인지 명시 필요.
5. **이미 active한 동일 `lineRange` 중복 처리 부재.** 두 active target이 같은 file의 겹치는 lineRange를 가리킬 수 있다. supersede 흐름이 아니라 사용자가 실수로 또 추가했을 때, 검증이 이를 잡아야 하는지 허용해야 하는지 명시 없음.
6. **replay task의 `answerSnapshot` 크기 상한 부재.** 매우 큰 함수가 target일 경우 snapshot이 카드 한 장보다 훨씬 커진다. cache가 아니라 source라서 truncate해서는 안 되지만, 너무 큰 경우 사용자 UX(`replays/*.md`를 열면 코드 본문이 화면을 가득 채움)에 영향. 상한 정책 결정 필요.
7. **여러 card가 같은 `conceptId`를 공유할 때 `verified` 게이트 위에 추가 정책이 있는지 부재.** 같은 concept을 다른 각도로 묻는 두 카드가 모두 active일 때, FSRS due 계산은 카드 단위지만 사용자 노출 시점에서 "concept 단위로 묶어 한 번에 보여줄지 / 각각 따로 보여줄지" 명시 없음. P0 review 흐름의 단위 결정 필요.
