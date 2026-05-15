# Walkthrough 001 — 세션 만료 계산에 grace period 추가

작성 일자: 2026-05-15
스코프: `docs/mvp-spec.md` P0 흐름을 가상의 작은 agent-assisted 변경 하나에 손으로 적용해, 스펙의 빈 곳을 코드 작성 전에 찾는다. 코드 구현은 하지 않는다.

## 사용 규칙

- 이 문서는 `docs/mvp-spec.md`와 `docs/product-direction.md`를 기준으로 한 검증 artifact다. 스펙 자체는 수정하지 않고, 발견한 빈 곳을 본 문서 끝 §발견된 스펙 빈 곳에 누적한다.
- 모든 식별자, 파일 경로, 시각은 가상이다. 실제 repo에 적용하지 않는다.
- `replay task`와 `review card`는 별도 절로 분리해 작성한다. verified 여부는 명시한다.

## 1. 가상 변경 정의

대상 함수: `src/auth/session.ts`의 `isExpired(session, now)`.

원래 구현(`base` 상태):

```ts
export function isExpired(session: Session, now: number): boolean {
  return session.expiresAt <= now;
}
```

변경 후 구현(`head` 상태):

```ts
const GRACE_MS = 30_000;

export function isExpired(session: Session, now: number): boolean {
  if (session.revokedAt !== null && session.revokedAt <= now) {
    return true;
  }
  return session.expiresAt + GRACE_MS <= now;
}
```

핵심 변화는 두 가지다.

- 세션이 명시적으로 `revokedAt`으로 폐기된 경우는 grace 없이 즉시 만료로 본다.
- 자연 만료(`expiresAt`)는 30초 grace period를 더해 만료로 본다.

이 변경은 외부 에이전트가 만들었고, 사용자는 본인이 다시 회수해야 한다고 판단해 Code Replay에 명시적으로 전달한다고 가정한다.

## 2. Target 입력

다음 한 줄을 `.codereplay/input/targets.jsonl`에 append한다고 가정한다.

```jsonl
{"type":"target_added","targetId":"target-001","baseCommit":"abc123","headCommit":"def456","file":"src/auth/session.ts","lineRange":{"start":3,"end":11},"provenance":{"source":"user","label":"agent-assisted"},"priority":"normal","createdAt":"2026-05-15T10:00:00Z"}
```

스펙 §`target_added` 필수 필드 매핑.

- `type` → `target_added`
- `targetId` → `target-001`
- `baseCommit` → `abc123`
- `headCommit` → `def456`
- `file` → `src/auth/session.ts`
- `lineRange.start` → `3`
- `lineRange.end` → `11`
- `provenance.source` → `user`
- `provenance.label` → `agent-assisted`
- `createdAt` → `2026-05-15T10:00:00Z`

선택 필드 중 `priority`만 채웠다. `note`, `diffFingerprint`는 비웠다.

## 3. Active target cache rebuild

`.codereplay/cache/active-targets.json`을 위 event log에서 rebuild한다고 가정한다.

```json
{
  "version": 1,
  "rebuiltAt": "2026-05-15T10:00:01Z",
  "source": ".codereplay/input/targets.jsonl",
  "targets": [
    {
      "targetId": "target-001",
      "baseCommit": "abc123",
      "headCommit": "def456",
      "file": "src/auth/session.ts",
      "lineRange": { "start": 3, "end": 11 },
      "provenance": { "source": "user", "label": "agent-assisted" },
      "priority": "normal",
      "createdAt": "2026-05-15T10:00:00Z"
    }
  ]
}
```

캐시 규칙(스펙 §Active target cache) 중 본 walkthrough에서 만지는 항목.

- 캐시가 없는 첫 실행을 가정 → rebuild 정상 동작.
- 캐시 손상 시 rebuild 시도 → 본 walkthrough에서는 검증하지 않음.

## 4. Target 검증

스펙 §Target 검증 5개 항목을 손으로 통과시킨다.

- 현재 위치가 git repo인지 확인 → walkthrough 가정상 OK.
- `baseCommit`/`headCommit` 존재 확인 → 가상이지만 OK로 본다.
- `file`이 `baseCommit..headCommit` 변경 범위에 포함되는지 → §1 변경이 동일 파일이므로 OK.
- `lineRange`가 변경 범위와 겹치는지 → 함수 본문이 `lineRange.start=3` ~ `end=11`에 들어간다고 가정.
- 동일 `targetId`가 active 상태로 중복되지 않는지 → 첫 event라 OK.

검증 통과 → 다음 단계 진행.

## 5. Replay task

`.codereplay/replays/replay-001.md`에 저장된다고 가정한다.

```markdown
---
replayId: replay-001
targetId: target-001
sourceFile: src/auth/session.ts
baseCommit: abc123
headCommit: def456
createdAt: 2026-05-15T10:00:05Z
referencePointer: src/auth/session.ts@def456:3-11
---

# Replay task — 세션 만료 계산에 grace period 추가

## 문제

`src/auth/session.ts`의 `isExpired(session, now)` 함수를 base 상태에서 head 상태로 다시 구현하라.

base 상태에서 이 함수는 `session.expiresAt`만 보고 즉시 만료를 판정했다. head 상태에서는 두 가지 조건이 추가된다. 어떤 조건이 어떤 순서로 어떤 값과 비교돼야 하는지 본인 머리로 다시 작성한다.

## 요구사항

- 함수 signature는 그대로 유지한다.
- `revokedAt`이 명시적으로 설정된 세션은 자연 만료와 다른 경로로 처리한다.
- 자연 만료에는 30초 grace period를 적용한다.
- `null`과 비교 누락으로 인한 false negative가 발생하지 않아야 한다.

## 힌트 (제출 후 공개)

- 두 경로의 결합 순서가 중요하다.
- grace period 상수는 모듈 상수로 둔다.

## 정답 참조

`reference` pointer를 따라간다. 제출 전까지는 열지 않는다.
```

스펙 P0 replay task 필수 정보 매핑.

- `replayId` → OK
- 연결된 `targetId` → OK
- source file → OK
- base/head commit → OK
- prompt → 본문 §문제·요구사항으로 충족
- hidden reference 또는 reference pointer → frontmatter `referencePointer`로 충족
- 생성 시각 → OK

## 6. Review card

`replay task`에서 파생한 작은 retrieval 단위 두 장을 만든다.

### 6.1 `fill_blank`

`.codereplay/cards/card-001.md`.

```markdown
---
cardId: card-001
targetId: target-001
replayId: replay-001
type: fill_blank
concept: session-expiry-with-grace
verified: true
fsrs:
  state: new
  due: 2026-05-15T10:00:05Z
createdAt: 2026-05-15T10:00:05Z
---

# fill_blank — grace period 비교식

다음 코드의 빈칸을 채워라.

```ts
const GRACE_MS = 30_000;

export function isExpired(session: Session, now: number): boolean {
  if (session.revokedAt !== null && session.revokedAt <= now) {
    return true;
  }
  return session.expiresAt ____ <= now;
}
```

## expected

`+ GRACE_MS`
```

verified 판정 근거.

- 연결된 target이 유효(§4 통과).
- prompt가 target 밖 맥락 없이 풀 수 있다(함수 본문만으로 답 결정 가능).
- expected answer가 target과 충돌하지 않는다(head 상태와 일치).
- 카드 타입이 P0 지원 타입(`fill_blank`).

→ `verified: true`. spaced review due 계산 대상.

### 6.2 `explain_decision`

`.codereplay/cards/card-002.md`.

```markdown
---
cardId: card-002
targetId: target-001
replayId: replay-001
type: explain_decision
concept: session-expiry-with-grace
verified: true
fsrs:
  state: new
  due: 2026-05-15T10:00:05Z
createdAt: 2026-05-15T10:00:05Z
---

# explain_decision — revoked 경로를 먼저 두는 이유

## 질문

`isExpired` 안에서 `revokedAt` 분기가 grace period를 적용하는 분기보다 앞에 있다. 이 순서를 뒤집으면 어떤 행동 변화가 생기는가? 한두 문장으로 답하라.

## rubric

- 명시적으로 폐기된 세션이 grace period 동안 계속 valid로 보일 수 있다는 점을 짚으면 정답.
- grace는 자연 만료에만 적용되며 revoke 의도와 충돌한다는 점을 짚으면 정답.
- 단순히 "순서가 중요하다"만 답하면 부분 정답.
```

verified 판정 근거.

- 연결된 target이 유효.
- prompt가 target 밖 맥락 없이 풀 수 있다(코드 두 분기의 의미 차이).
- rubric이 target과 충돌하지 않는다.
- 카드 타입이 P0 지원 타입(`explain_decision`).

→ `verified: true`.

## 7. Review log

사용자가 첫 review 세션에서 두 카드를 풀었다고 가정한다. `.codereplay/reviews/review-log.jsonl`에 두 줄 append.

```jsonl
{"type":"card_reviewed","cardId":"card-001","rating":"good","reviewedAt":"2026-05-16T09:00:00Z"}
{"type":"card_reviewed","cardId":"card-002","rating":"again","reviewedAt":"2026-05-16T09:00:30Z"}
```

review log는 카드 원문을 대체하지 않으며 결과만 기록한다(스펙 §Review log).

## 8. 수용 기준 점검

스펙 §수용 기준 7개 항목을 본 walkthrough가 위반하지 않는지 확인.

- 명시적 target 없이 replay/card를 생성하지 않음 → §2가 입력으로 들어와야만 §5, §6가 생성됨. OK.
- 유효하지 않은 target은 active workflow에서 제외 → §4 검증 통과를 전제로만 §5 진행. OK.
- `targets.jsonl`은 append-only → §2에서 한 줄 append만. OK.
- `active-targets.json`은 rebuild 가능 → §3에서 §2 입력으로부터 재구성. OK.
- verified card만 due review 대상 → §6 두 카드 모두 `verified: true`. OK.
- 사용자는 원본 코드를 보기 전에 review card를 풀 수 있음 → §6 카드는 함수 본문만으로 풀이 가능. OK.
- Code Replay는 agent-assisted 여부를 탐지했다는 표현을 쓰지 않음 → 본 walkthrough는 사용자 입력만으로 `provenance.label`을 받음. OK.

## 발견된 스펙 빈 곳

walkthrough 진행 중 mvp-spec.md / product-direction.md가 답을 주지 않았던 지점만 적는다. 추측으로 채운 결정은 "추정"이라고 표기.

1. **`lineRange` 좌표계 정의 부재.** 1-based vs 0-based, end-inclusive vs exclusive가 스펙에 없다. walkthrough에서는 1-based + inclusive로 추정.
2. **`baseCommit..headCommit`의 의미가 모호.** range diff인지 `git diff base head`인지, merge 베이스 자동 추론인지 명시 없음. walkthrough는 `git diff base head` 단일 호출로 추정.
3. **target 검증의 "변경 범위 포함" 판정 기준 부재.** 파일이 rename된 경우, lineRange만 보면 `--follow` 없이 못 잡는다. P0가 rename을 어떻게 다루는지 결정 필요.
4. **`replayId`/`cardId` 생성 규칙 부재.** 단순 sequential인지 ULID/UUID인지 스펙에 없음. 사용자 직접 편집 가능성에 영향.
5. **`replay task`의 reference pointer 형식 부재.** 본 walkthrough는 `path@sha:start-end`로 추정. 동일 줄 범위가 head에서 바뀌었을 때 어떻게 추적하는지 정의 필요.
6. **review card frontmatter 표준 부재.** spec은 "필수 정보"만 나열한다. FSRS metadata 구조(`state`, `due`, `stability` 등)와 `verified` 위치(frontmatter vs body)가 결정 안 됨.
7. **review log의 rating 값 집합 미정.** FSRS 표준은 `again/hard/good/easy`, SM-2는 0-5. walkthrough는 `good/again`만 사용했지만 spec이 어느 쪽인지 명시 필요.
8. **card 폐기 이벤트 부재.** `target_dismissed`, `target_superseded`는 있지만 카드 단위 dismiss/superseded는 없다. 사용자가 잘못된 카드를 한 번 flag하면 어떤 이벤트가 어디에 남는지(별도 log? cards/ 안 frontmatter?) 결정 필요.
9. **`concept` label/id 발급 규칙 부재.** walkthrough는 `session-expiry-with-grace` 같은 슬러그를 가정. 누가 어떤 시점에 만드는지(생성기 자동, 사용자 입력, 첫 등장 시 prompt?) 미정.
10. **`fast mode`의 산출물 위치 부재.** spec은 review card로 저장 안 함이라고만 명시. 그럼 fast mode 결과는 어디에 잠시 보관되는지(`/tmp`, `.codereplay/scratch/`, 휘발) 결정 필요.
11. **target 검증 실패 시 "오류 보고" 경로 부재.** stderr인지 별도 `validation-log.jsonl`인지 미정. cache rebuild와 다른 경로일 수 있음.
12. **`baseCommit`/`headCommit` 존재 확인 시 fetch가 필요한 케이스 부재.** local에 없으면 자동 fetch인지, 실패 처리인지 미정.
