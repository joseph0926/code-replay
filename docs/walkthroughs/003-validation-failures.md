# Walkthrough 003 — 검증 실패 케이스 묶음

작성 일자: 2026-05-15
스코프: 새로 명문화된 P0 검증 규칙(rename 미지원, 자동 fetch 없음, lineRange/diff 정합)이 실패하는 세 케이스를 손으로 통과시킨다. `target-validation.json`의 모양과 사용자 흐름을 검증한다. 코드 구현은 하지 않는다.

## 출발 상태

워크스루 001/002와 같은 가상 변경(`src/auth/session.ts` `isExpired`에 grace period 추가)이 base→head 사이에 존재한다고 가정. 본 문서는 그 위에서 "**검증을 통과 못 하는** target 입력 세 줄"을 시뮬레이션한다.

세 케이스는 같은 실행에서 한꺼번에 들어왔다고 가정해, 하나의 `target-validation.json`이 세 결과를 동시에 담는 흐름까지 본다.

## 1. 케이스 A — rename된 파일

### 1.1 입력

base에서 파일 경로는 `src/auth/session.ts`였고, head에서 `src/auth/sessions.ts`로 rename됐다고 가정한다.

외부 도구가 base 기준 경로로 target을 만들어 append.

```jsonl
{"type":"target_added","targetId":"target_01J01ABCDEFGHJKMNPQRSTVWXY","baseCommit":"abc123","headCommit":"def456","file":"src/auth/session.ts","lineRange":{"start":3,"end":11},"provenance":{"source":"codex","label":"agent-assisted"},"createdAt":"2026-05-15T12:00:00Z"}
```

### 1.2 검증

- git repo 확인 → OK.
- `baseCommit`/`headCommit` local 존재 → OK 가정.
- `file`이 `git diff <baseCommit> <headCommit>` 변경 범위에 포함 → `src/auth/session.ts`는 head에 없다. diff에서는 rename으로 표현된다(`src/auth/session.ts → src/auth/sessions.ts`).

스펙은 명시한다: "P0에서 rename 추적은 지원하지 않는다. `file`은 `headCommit` 기준 경로여야 하며, rename 때문에 target을 로컬 diff와 안정적으로 대조할 수 없으면 검증 실패로 처리한다."

따라서 검증 실패. 사용자가 head 경로(`src/auth/sessions.ts`)로 다시 target을 추가해야 한다.

### 1.3 `target-validation.json` 결과 entry

```json
{
  "targetId": "target_01J01ABCDEFGHJKMNPQRSTVWXY",
  "status": "invalid",
  "reason": "rename-not-supported"
}
```

`reason` 값은 본 walkthrough가 추정한 슬러그다. 스펙은 `missing-head-commit` 한 예시만 보여줬다 → §발견된 스펙 빈 곳 1.

## 2. 케이스 B — local에 없는 headCommit

### 2.1 입력

다른 머신에서 만든 manifest를 import했는데, 그 머신의 head SHA가 본 머신에는 아직 fetch 안 됐다고 가정한다.

```jsonl
{"type":"target_added","targetId":"target_01J01BCDFGHJKMNPQRSTVWXYZ2","baseCommit":"abc123","headCommit":"ffffff9999999999999999999999999999999999","file":"src/auth/sessions.ts","lineRange":{"start":3,"end":11},"provenance":{"source":"codex","label":"agent-assisted"},"createdAt":"2026-05-15T12:00:01Z"}
```

### 2.2 검증

- git repo 확인 → OK.
- `baseCommit`/`headCommit` local 존재 → `headCommit`이 local object database에 없다.

스펙: "P0는 자동 fetch를 수행하지 않는다. `baseCommit` 또는 `headCommit`이 local git object database에 없으면 검증 실패로 처리하고 사용자가 직접 fetch하도록 안내한다."

따라서 검증 실패.

### 2.3 결과 entry

```json
{
  "targetId": "target_01J01BCDFGHJKMNPQRSTVWXYZ2",
  "status": "invalid",
  "reason": "missing-head-commit"
}
```

이건 스펙 예시 그대로다.

### 2.4 사용자 흐름

스펙은 "사용자가 직접 fetch하도록 안내한다"고 적었다. 안내 후 사용자가 `git fetch origin ffffff9...`로 받아 오면 같은 target이 다음 실행에서 다시 검증을 통과해야 한다. 하지만 `targets.jsonl`은 append-only라 첫 검증 실패가 기록되지 않는다(검증 결과는 cache로만 존재).

→ 같은 ULID로 두 번 검증해도 문제 없는가? `target-validation.json`이 마지막 결과로 덮어쓰면 OK다. 그러나 스펙은 cache 안의 results 배열이 "마지막 검증 1회분 전체"인지 "이력 누적"인지 명시 안 했다 → §발견된 스펙 빈 곳 2.

## 3. 케이스 C — lineRange가 diff 변경 범위와 겹치지 않음

### 3.1 입력

사용자가 손으로 추가했는데, `lineRange`를 함수 본문 아래쪽 빈 영역(파일에서 실제 변경되지 않은 부분)으로 잘못 지정했다고 가정한다.

```jsonl
{"type":"target_added","targetId":"target_01J01CDEFHJKMNPQRSTVWXYZ23","baseCommit":"abc123","headCommit":"def456","file":"src/auth/sessions.ts","lineRange":{"start":20,"end":25},"provenance":{"source":"user","label":"agent-assisted"},"createdAt":"2026-05-15T12:00:02Z"}
```

base→head diff에서 `src/auth/sessions.ts`는 3-11줄만 변경됐다고 가정. 20-25는 변경되지 않은 영역.

### 3.2 검증

- git repo 확인 → OK.
- commits local 존재 → OK 가정.
- `file`이 변경 범위에 포함 → 파일 자체는 diff에 등장하므로 OK.
- `lineRange`가 `headCommit` 기준 1-based, inclusive로 유효 → 파일이 11줄짜리라면 20-25는 파일 범위 밖이다. 만약 head에서 파일이 더 길어져 25줄까지 있다면 좌표는 유효하지만 변경 hunk와는 안 겹친다.

본 walkthrough는 head 파일이 30줄로 늘어났다고 가정해, 좌표는 유효하지만 변경 hunk와는 겹치지 않는 케이스를 본다.

- `lineRange`가 tree diff 변경 범위와 겹침 → 변경 hunk는 3-11. 20-25와 안 겹친다.

따라서 검증 실패.

### 3.3 결과 entry

```json
{
  "targetId": "target_01J01CDEFHJKMNPQRSTVWXYZ23",
  "status": "invalid",
  "reason": "line-range-outside-diff"
}
```

`reason` 슬러그는 추정.

### 3.4 좌표는 유효하지만 hunk와 안 겹치는 경계 케이스

만약 사용자가 `lineRange: {start: 200, end: 210}`을 적었다면 좌표 자체가 파일 범위 밖이다. 이때 `reason`은 `line-range-outside-diff`로 같이 묶을지, `line-range-invalid`로 따로 분류할지 모호 → §발견된 스펙 빈 곳 1과 연결.

## 4. 통합 `target-validation.json`

세 케이스를 같은 실행에서 처리했다고 가정. 추가로 walkthrough 002의 정상 target을 active 상태로 갖고 있다.

```json
{
  "version": 1,
  "validatedAt": "2026-05-15T12:00:10Z",
  "results": [
    {
      "targetId": "target_01J00A1H7E5T8N2K4Q6R8M3W2V",
      "status": "valid"
    },
    {
      "targetId": "target_01J01ABCDEFGHJKMNPQRSTVWXY",
      "status": "invalid",
      "reason": "rename-not-supported"
    },
    {
      "targetId": "target_01J01BCDFGHJKMNPQRSTVWXYZ2",
      "status": "invalid",
      "reason": "missing-head-commit"
    },
    {
      "targetId": "target_01J01CDEFHJKMNPQRSTVWXYZ23",
      "status": "invalid",
      "reason": "line-range-outside-diff"
    }
  ]
}
```

## 5. Active target cache 영향

`active-targets.json`은 검증 결과와 독립이다(스펙). active 여부는 `target_added` − `target_dismissed`/`target_superseded`에서 정해지지, 검증 실패한 target도 형식상 active 상태다. 다만 CLI `make`는 검증 실패한 target에 대해서는 replay/card 생성을 하지 않아야 한다.

`active-targets.json` 추정 상태:

```json
{
  "version": 1,
  "rebuiltAt": "2026-05-15T12:00:10Z",
  "source": ".codereplay/input/targets.jsonl",
  "targets": [
    { "targetId": "target_01J00A1H7E5T8N2K4Q6R8M3W2V", ... },
    { "targetId": "target_01J01ABCDEFGHJKMNPQRSTVWXY", ... },
    { "targetId": "target_01J01BCDFGHJKMNPQRSTVWXYZ2", ... },
    { "targetId": "target_01J01CDEFHJKMNPQRSTVWXYZ23", ... }
  ]
}
```

→ 사용자 입장에서 "왜 4개 active인데 3개는 결과물이 안 나왔지?"가 됨. 두 cache(`active-targets.json`과 `target-validation.json`)를 함께 봐야 상태가 설명된다.

CLI가 두 cache의 결합 상태를 어떻게 보여주는지 스펙은 명시 안 했다 → §발견된 스펙 빈 곳 3.

## 6. 후속 실행에서의 흐름

사용자가 케이스 A를 정정하기 위해 다음을 한다.

1. 머신을 보고 head 경로가 `src/auth/sessions.ts`임을 확인.
2. 새 target_added를 append (정상 경로로).
3. 첫 target은 그대로 두면 매 실행마다 검증을 다시 시도하고 매번 `invalid`로 남는다.

여기서 잘못된 target을 사용자가 어떻게 정리해야 자연스러운지 결정점이 있다.

- `target_dismissed`로 명시적 종료
- `target_superseded`로 새 정상 target에 연결
- 그냥 두기 (검증은 매번 실패하지만 active로 남음)

스펙은 어느 것을 권하는지 명시 없음 → §발견된 스펙 빈 곳 4.

비슷하게 케이스 B는 사용자가 `git fetch` 후 다시 실행하면 같은 ULID로 같은 target이 자동 통과한다. 검증 결과만 cache에 덮어쓰면 정상이다. 단 §발견된 스펙 빈 곳 2와 연결.

## 7. 수용 기준 점검 (실패 케이스 위주)

- 명시적 target 없이 replay/card를 생성하지 않음 → 실패한 세 target 모두 replay/card 미생성. OK.
- 유효하지 않은 target은 active workflow에서 제외 → §5에서 본 것처럼 `active-targets.json`엔 남고 `make` 단계에서 제외. "제외"의 의미가 active 목록에서 빼는 건지, make에서 skip하는 건지 모호 → §발견된 스펙 빈 곳 5.
- 좌표/diff 검증 규칙 → 세 실패 모두 규칙대로 실패. OK.
- 자동 fetch 없음 → 케이스 B에서 실패로 끝남. OK.
- ID 형식 → 모두 `target_<ULID>`. OK.
- `targets.jsonl` append-only → 실패해도 이벤트는 그대로 남음. OK.
- `target-validation.json` rebuildable → §4에서 재구성 가능. OK.
- agent-assisted 탐지 표현 없음 → OK.

## 발견된 스펙 빈 곳

1. **`reason` 슬러그 집합 미정.** 스펙은 `missing-head-commit` 하나만 예시로 보여준다. 본 walkthrough는 `rename-not-supported`, `line-range-outside-diff` 등을 추정해 적었다. CLI 출력과 외부 도구가 이 값에 분기를 거는 경우가 생기므로, P0가 어떤 슬러그를 보장하는지 정의가 필요하다. 최소 후보: `missing-base-commit`, `missing-head-commit`, `file-not-in-diff`, `rename-not-supported`, `line-range-invalid`, `line-range-outside-diff`, `duplicate-target-id`, `not-a-git-repo`.
2. **`target-validation.json` `results`가 "마지막 1회분"인지 "이력 누적"인지 미정.** §2.4와 §6에서 같은 ULID가 fetch 전후로 두 번 검증된다. cache 내용이 이전 결과를 덮어쓰는지, 두 결과를 누적해 가지는지 결정 필요. cache 정의상 rebuildable이라면 "마지막 1회분"이 자연스럽다.
3. **CLI가 두 cache(active + validation)의 결합 상태를 어떻게 보여주는지 미정.** 사용자 입장에서 "active 4개 중 1개만 진짜 처리됨"을 알 수 없다. `target ls`나 `make` 출력에서 두 cache를 어떻게 결합해 보여주는지 결정 필요.
4. **검증 실패가 누적된 target을 사용자가 정리하는 권장 흐름 부재.** dismiss vs supersede vs 그대로 두기 중 어느 것이 default인지 명시 없음. 케이스별로 권장이 다를 수 있다(rename 정정은 supersede가 자연스럽고, 잘못된 lineRange는 dismiss 후 새로 만드는 게 자연스럽다).
5. **"active workflow에서 제외"의 의미 미정.** `active-targets.json`에서 빼는 것인지, `make` 단계에서 skip하는 것인지 정의 필요. 본 walkthrough는 후자(active list엔 남고 make에서 skip)로 추정. 두 의미가 CLI 출력 결정에 직접 영향.
6. **재시도/캐시 무효화 트리거 미정.** 케이스 B에서 사용자가 `git fetch` 후 같은 target을 재검증하려면, CLI가 자동으로 stale cache를 재계산하는지(`targets.jsonl` mtime 비교?), 사용자가 명시적 명령(`target validate` 또는 `target rebuild-cache`)을 실행해야 하는지 결정 필요. 스펙의 `target rebuild-cache`는 `active-targets.json` 재생성에 한정돼 있어 validation cache는 다른 trigger가 필요할 수 있다.
7. **diff hunk 단위 교차 검증의 정의 부재.** lineRange가 hunk와 "겹친다"의 정확한 정의가 없다. 예를 들어 hunk가 5-9이고 lineRange가 4-6이면 부분 겹침이다. 부분 겹침을 통과시킬지, 완전 포함만 통과시킬지 결정 필요. 학습 단위 추출에 직접 영향.
8. **`provenance.source` 값 자유도 미정.** mvp-spec.md는 `source`를 필수 필드로 적었지만 값 집합은 product-direction의 `user`, `codex`, `claude-code`, `cursor`, `copilot` 예시뿐이다. 외부 에이전트가 새 source 라벨을 임의로 만들 수 있는지, allowed set인지 결정 필요.
