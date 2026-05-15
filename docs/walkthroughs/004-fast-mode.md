# Walkthrough 004 — fast mode 흐름

작성 일자: 2026-05-15
스코프: 사용자가 즉석 연습용으로 fast mode를 트리거하고, 결과가 `.codereplay/cards/`에 저장되지 않으며, 나중에 정식 카드로 만들려면 표준 생성 흐름을 다시 통과해야 한다는 P0 규칙을 손으로 따라간다. 코드 구현은 하지 않는다.

## 출발 상태

Walkthrough 002 종료 시점을 그대로 이어받는다.

`active-targets.json`에 단일 active target.

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

`target-validation.json`에 동일 target이 `valid`로 기록돼 있다.

`.codereplay/cards/`는 비어 있다고 가정한다(이 walkthrough는 standard mode 결과와 fast mode 결과를 같은 출발점에서 비교한다).

사용자는 시간이 부족해 "정식 카드까지 만들기 전에 한 번 즉석으로 풀어보고 싶다"고 결정한다.

## 1. Fast mode 트리거

스펙 §최소 CLI 흐름은 명령 이름을 명시하지 않았다. 본 walkthrough는 가장 자연스러운 추정으로 `make --fast`를 사용한다고 가정한다.

```sh
$ codereplay make --fast --target target_01J00A1H7E5T8N2K4Q6R8M3W2V
```

여기서 첫 결정점이 생긴다 → §발견된 스펙 빈 곳 1.

가상 CLI 동작:

1. active-targets.json에서 해당 target을 찾는다.
2. validation status가 `valid`인지 다시 확인한다(스펙 §`make` 흐름은 생성 전 validation을 다시 실행한다고 명시. fast mode도 같은 규칙을 따른다고 가정 — 빈 곳 2).
3. LLM을 빠른 모드(round-trip k=1, distractor 적은 수)로 호출해 fill_blank 후보 1개를 메모리에서 생성한다.
4. 결과를 stdout에 출력한다.
5. 종료. 파일 시스템에 아무것도 쓰지 않는다.

## 2. Fast mode 결과의 표시

가상 CLI 출력 형태(추정).

```text
[fast mode] target_01J00A1H7E5T8N2K4Q6R8M3W2V — 결과는 저장되지 않습니다.

# fill_blank — grace period 비교식

const GRACE_MS = 30_000;

export function isExpired(session: Session, now: number): boolean {
  if (session.revokedAt !== null && session.revokedAt <= now) {
    return true;
  }
  return session.expiresAt ____ <= now;
}

답을 입력하세요 (Ctrl+D로 종료):
```

사용자가 `+ GRACE_MS`를 입력한다.

```text
> + GRACE_MS

정답입니다.

[fast mode] 1문항 완료. 저장된 파일 없음. review-log에 기록되지 않음.
```

여기서 결정점이 두 개 더 생긴다.

- fast mode가 사용자 입력을 받는지(인터랙티브) 단순히 prompt + answer를 한꺼번에 보여주는지 → 빈 곳 3.
- 정답 판정을 어떻게 하는지(AST normalize / 문자열 일치 / LLM judge) → 빈 곳 4.

## 3. 휘발 확인

세션 종료 후 파일 시스템 상태.

```sh
$ ls .codereplay/cards/
# (비어 있음)

$ cat .codereplay/reviews/review-log.jsonl
# (변화 없음)

$ cat .codereplay/cache/target-validation.json
# validatedAt만 갱신됐을 수 있음 (§1의 가정 2에 따라)
```

기대:
- `.codereplay/cards/`에 아무 파일도 안 생긴다.
- `review-log.jsonl`에 fast mode review 이벤트가 append되지 않는다.
- `target-validation.json`은 make 실행에 따라 validatedAt만 갱신될 수 있다.

스펙 §Fast mode "P0에서는 CLI 출력과 메모리 안에서만 유지하고 `.codereplay/cards/`에 저장하지 않는다" + "spaced review 일정에 넣지 않는다"와 일관.

## 4. 다음 날 — 사용자가 정식 승격을 원함

사용자는 fast mode에서 본 문제가 마음에 들어 "정식 카드로 저장하고 spaced review에 넣고 싶다"고 결정한다.

스펙 §Fast mode: "사용자가 저장 가능한 card로 만들려면 표준 생성 흐름을 다시 통과해야 한다."

즉, fast mode에서 본 카드 텍스트를 직접 가져와 frontmatter를 붙이는 단축 경로는 없다(추정). 사용자는 standard mode로 다시 generate해야 한다.

```sh
$ codereplay make --target target_01J00A1H7E5T8N2K4Q6R8M3W2V
```

가상 동작:

1. validation 재실행 → `valid`.
2. LLM을 standard mode(round-trip k=3, AST 검증, rubric 충돌 검사)로 호출해 review card 후보를 생성한다.
3. 검증 통과한 card를 `.codereplay/cards/card_<ULID>.md`로 저장한다.
4. `status: draft`로 시작하고 verified gate 통과 시 `active`로 승격(또는 즉시 `active`로 저장, 스펙은 어느 쪽인지 명시 없음 → 빈 곳 5).

가상 결과 파일 `.codereplay/cards/card_01J03H2X9Y4M6N8K1Q5R7T2V0B.md`:

```md
---
cardId: card_01J03H2X9Y4M6N8K1Q5R7T2V0B
targetId: target_01J00A1H7E5T8N2K4Q6R8M3W2V
replayId: replay_01J03GZQ7M2N5K8P3R6T9W4V1A
type: fill_blank
conceptId: concept_session_expiry_boundary
verified: true
status: active
createdAt: 2026-05-16T09:00:00Z
fsrs:
  state: new
  due: 2026-05-16T09:00:00Z
  stability: null
  difficulty: null
  reps: 0
  lapses: 0
---

# fill_blank — grace period 비교식

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

여기서 fast mode 결과와 standard mode 결과의 관계가 모호하다 → 빈 곳 6.

- 두 결과가 같은 문제로 보장되나? (같은 빈칸 위치, 같은 정답?)
- 같은 LLM seed로 deterministic하게 같은 카드가 나오나, 아니면 매 호출 다른 변형이 나올 수 있나?
- 사용자가 fast mode에서 풀었던 문제와 standard mode 카드가 다르면 "내가 fast mode에서 잘 풀었던 그 문제를 정식으로 받고 싶다"는 의도를 만족 못 함.

## 5. Fast mode 중 lifecycle 변화

walkthrough 시나리오 확장. fast mode 실행 도중 별도 프로세스에서 사용자가 같은 target에 `target_dismissed`를 append했다고 가정.

fast mode는 메모리에서 진행 중이라 이미 LLM 호출은 끝났을 수 있다. 종료 시점에 lifecycle이 바뀐 걸 감지해야 하는가?

- 감지하지 않는다면: 사용자에게는 이미 dismiss한 target의 문제를 풀어보게 된다. 큰 해는 없지만 혼란.
- 감지한다면: 종료 시점에 lifecycle 재확인 + 안내 메시지.

스펙은 fast mode의 lifecycle 재확인 시점을 명시 안 했다 → 빈 곳 7. P0 단일 프로세스 사용을 가정하면 무시할 수도 있다.

## 6. Fast mode와 invalid target

또 다른 확장. 사용자가 검증 실패한 target(walkthrough 003 케이스 A)에 fast mode를 시도한다.

```sh
$ codereplay make --fast --target target_01J01ABCDEFGHJKMNPQRSTVWXY
```

스펙은 `make`가 invalid target을 skip한다고 명시한다. fast mode도 같은 규칙을 따른다면 fast mode 역시 skip해야 일관.

```text
[fast mode] target_01J01ABCDEFGHJKMNPQRSTVWXY skipped: rename-not-supported
```

본 walkthrough는 이렇게 가정하지만 스펙이 명시하지 않았다 → 빈 곳 2와 같은 결정 그룹.

## 7. Fast mode와 review log의 분리 재확인

명시적으로 강조할 가치가 있다.

스펙 §Fast mode 제약 세 줄을 다시 본다.

- review card로 자동 승격하지 않는다 → 파일 시스템에 카드 파일 없음. OK.
- spaced review 일정에 넣지 않는다 → review-log.jsonl에 `card_reviewed` 이벤트 없음. OK.
- 메모리/CLI 출력만 → 종료 후 흔적 없음. OK.

다만 "사용자가 fast mode에서 어떻게 풀었는지" 데이터는 product가 어디서도 갖지 않는다. 이는 의도된 결정이다(spaced review를 오염시키지 않음). 그러나 fast mode 빈도/패턴 같은 사용 신호 자체도 안 남는다는 점을 명시한다 → 빈 곳 8.

## 8. 수용 기준 점검

스펙 §수용 기준 중 fast mode 관련 항목.

- "fast mode 결과는 P0에서 `.codereplay/cards/`에 저장하지 않는다" → §3에서 확인. OK.
- "verified: true, status: active, active target 연결을 모두 만족하는 card만 due review 대상" → §4의 standard 결과 카드만 due. fast mode 결과는 due 대상 아님. OK.
- "target dismiss/supersede는 기존 replay/card를 자동 수정하거나 reattach하지 않는다" → fast mode는 어차피 파일을 만들지 않으니 무관. OK.
- "Code Replay는 agent-assisted 여부를 탐지했다는 표현을 UI/CLI/문서에 쓰지 않는다" → fast mode CLI 출력에도 적용. §2 출력에 그런 표현 없음. OK.

기타 수용 기준 위반 없음.

## 발견된 스펙 빈 곳

1. **Fast mode를 트리거하는 CLI 형태 미정.** `make --fast`인지, 별도 `practice`/`drill` 명령인지, `review --fast`인지 스펙에 없음. CLI surface 안정성에 직접 영향이라 P0에서 결정 필요. `make --fast`가 명령 수를 안 늘려서 가장 단순.
2. **Fast mode가 validation을 거치는지 미정.** standard `make`는 생성 전 validation을 다시 실행한다고 명시. fast mode가 같은 규칙을 따르는지 명시 없음. invalid target에서 fast mode가 가능한지(연습용이라 통과시킬지) 결정 필요.
3. **Fast mode가 사용자 답안 입력을 받는지 미정.** 단순 prompt + answer 공개(passive viewing)인지, 사용자가 답을 입력해 즉시 채점되는지(interactive drill)인지 스펙 없음. UX 결정 + LLM 호출 수에 영향.
4. **Fast mode에서 정답 판정 방식 미정.** AST normalize 비교, 문자열 정확 일치, LLM judge 중 어느 것을 쓰는지. fast mode가 LLM 호출을 줄이기 위한 모드라 judge fallback이 어울리지 않는다. AST normalize 또는 문자열 일치가 자연스럽지만 명시 필요.
5. **Standard make 결과의 초기 status가 `draft`인지 `active`인지 미정.** 스펙은 "verified: true, status: active만 due"라고 적었다. make가 검증을 통과한 card를 곧바로 `active`로 저장하는지, `draft`로 저장한 뒤 별도 승인 단계가 있는지 명시 없음. 본 walkthrough §4는 즉시 `active`로 추정.
6. **Fast mode와 standard mode의 결정성/동일성 보장 부재.** 같은 target에 두 모드를 돌렸을 때 같은 빈칸 위치, 같은 정답이 나오는지 보장 없음. 사용자가 fast에서 본 카드가 standard에서 그대로 나오기를 기대할 수 있는데 LLM 호출이라 일반적으로 비결정적. seed 고정/캐싱 정책 결정 필요. 또는 명시적으로 "다를 수 있다"고 적어 사용자 기대치를 맞춰야 함.
7. **Fast mode 도중 lifecycle 변경 감지 시점 미정.** §5 시나리오는 P0 단일 프로세스에서는 거의 발생하지 않지만, 사용자가 새 터미널에서 `target dismiss`를 동시에 실행할 수 있다. fast mode 종료 시점에 재확인하는지, 무시하는지 명시 필요.
8. **Fast mode 사용 신호 저장 정책 미정.** review-log에 안 들어간다는 건 명시. 그러나 "fast mode N회 사용" 같은 단순 counter도 안 남는지(=대시보드/통계 신호 없음), 아니면 별도 telemetry-free counter가 가능한지 명시 없음. P0 default는 "아무 신호도 안 남김"이 자연스럽지만 명시는 필요.
9. **Fast mode가 다루는 card 타입 미정.** P0 review card 타입은 `fill_blank`와 `explain_decision`. fast mode가 둘 다 지원하는지, 즉석 채점이 어려운 `explain_decision`은 제외하는지 결정 필요. 본 walkthrough는 `fill_blank`만 가정.
10. **Fast mode가 한 번에 만드는 문항 수 미정.** §2는 1문항을 가정했다. 스펙은 P0 standard make가 한 실행에서 1-3개 수준이라고 적었다. fast mode도 1-3개인지, 1개로 더 좁히는지, 사용자가 지정 가능한지 명시 필요.
11. **Fast mode 결과를 정식 승격하는 단축 경로 부재.** 사용자가 fast에서 본 정확히 그 카드를 정식으로 저장하고 싶을 때, 스펙은 "표준 생성 흐름을 다시 통과해야 한다"고만 적었다. 단축 경로가 없다는 결정 자체는 일관적이지만, UX 측면에서 "방금 본 그 카드"를 다시 만나려면 §6 결정성 문제와 묶인다.
12. **CLI 출력 안 fast mode 표식 위치 미정.** 본 walkthrough §2는 `[fast mode]` 접두사를 가정. 스펙은 "AI 생성" 배지(`docs/llm-quiz-hallucination.md` §7)는 명시했지만 fast mode임을 명확히 알리는 출력 표준은 없음. 사용자가 standard make 출력과 혼동하지 않도록 명시 필요.
