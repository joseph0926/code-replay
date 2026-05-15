# Agent 연동: provenance 입력과 replay 실행 경계

리서치 일자: 2026-05-15
스코프: 외부 코딩 에이전트 흐름(Codex, Claude Code 등)이 Code Replay에 **agent-assisted `target`을 어떻게 명시적으로 전달**하고, replay 실행 시 원본 solution을 어떻게 분리하는지 정한다. `docs/product-direction.md`의 입력 경계와 `docs/mvp-spec.md` Target 입력 계약을 외부 에이전트 작업 흐름 관점에서 보강한다.

## 현재 결정 기준

이 문서는 초기에 "AI가 짠 부분 vs 내가 짠 부분"을 도구가 식별한다는 프레임으로 시작했다. 현재 product-direction은 다음 경계를 따른다.

- Code Replay는 어떤 변경이 AI agent의 도움을 받았는지 탐지하거나 추론하지 않는다.
- `agent-assisted`는 사용자나 외부 연동이 명시적으로 전달한 `provenance` 속성이다.
- 표준 입력은 `.codereplay/input/targets.jsonl` append-only event log이며, 활성 상태는 `.codereplay/cache/active-targets.json` 재생성 가능한 read model이다.

따라서 이 문서의 "신호 우선순위", "agent-hunks.json 스키마" 같은 절은 현재 결정 위에서 **외부 에이전트가 target event를 어떻게 만들어 전달하는가**에 대한 보조 설계 노트로 읽는다.

## 한 줄 결론

- agent 연동의 핵심은 "에이전트가 만든 변경을 도구가 알아내는 것"이 아니라 **에이전트(또는 사용자)가 만든 변경을 명시적인 target event로 Code Replay에 전달하고, 사용자가 그 변경을 unassisted replay하게 만드는 것**이다.
- 표준 입력은 `.codereplay/input/targets.jsonl` 이벤트 로그다. P0에서 외부 에이전트가 남길 수 있는 신호는 commit trailer, 사용자 명시 표시, 외부 도구가 직접 append한 target event 정도다. transcript/log 직접 파싱이나 AI authorship classifier는 비-목표다.
- Codex 쪽은 `AGENTS.md`를 repo 공통 규칙 source of truth로 두고, 반복 replay 워크플로는 Skill, 공유/설치 단위는 Plugin으로 승격한다. replay 실행은 `workspace-write + on-request` 같은 낮은 위험 permission profile에서 시작한다.
- Claude Code 쪽은 `CLAUDE.md`가 `@AGENTS.md`를 참조하는 현재 구조를 유지하고, 빠른 실험은 `.claude/skills`, 공유형은 plugin으로 둔다. replay 중 원본 solution 노출 차단은 permissions + hook으로 보조하되, 학습 ground truth는 사용자 제출과 로컬 결과 파일이 맡는다.
- 모든 provenance, 카드 품질 피드백, concept label은 **로컬 `.codereplay/`에만 저장**한다. code-replay 자체 서버나 외부 analytics로 보내지 않는다. `.codereplay/`의 git tracking 여부는 사용자가 결정하며, P0 기본 가이드는 local untracked 사용이다.

## 1. 왜 agent 연동이 필요한가

code-replay의 차별점은 단순히 PR diff를 학습 자료로 바꾸는 것이 아니라, **사용자가 명시적으로 전달한 agent-assisted 변경을 사후에 unassisted replay하게 만드는 것**이다.

이미 문서화된 핵심 전제:

- `docs/product-direction.md` — Code Replay는 target을 발견하지 않고, 사용자나 외부 연동이 명시적으로 전달한 target만 받는다.
- `docs/mvp-spec.md` — 표준 입력은 `.codereplay/input/targets.jsonl` append-only event log다.
- `docs/ast-diff-tools.md` — AST diff는 누가 썼는지 알 수 없고, provenance는 외부에서 주어진다.
- `docs/local-vs-api-llm.md` — diff 전체가 입력이 될 수 있으므로 privacy가 모델 성능보다 우선이다.
- `docs/llm-quiz-hallucination.md` — 생성 카드 품질은 검증과 사용자 피드백 루프 없이는 retention metric을 오염시킨다.

따라서 agent 연동은 세 가지 책임을 가진다.

1. agent 또는 사용자가 만든 변경을 **target event**로 Code Replay에 전달하고, provenance를 함께 남긴다.
2. replay 중 사용자가 원본 solution을 너무 일찍 보지 않도록 실행 경계를 만든다.
3. 카드와 재구현 결과에 대한 사용자 피드백을 다음 생성 품질에 반영한다.

## 2. 항상-온 규칙과 on-demand workflow 분리

### 2.1 현재 repo 구조

현재 구조는 좋다.

- `AGENTS.md`: repo 공통 규칙과 문서 지도.
- `CLAUDE.md`: `@AGENTS.md`만 참조.
- `docs/`: 제품 리서치 source of truth.
- `research/`: 로컬 지식베이스, 수정 금지.

이 구조는 Codex와 Claude Code 양쪽에서 중복 규칙을 줄인다. 특히 Claude Code 쪽에서 `CLAUDE.md`가 별도 긴 규칙을 갖지 않고 `@AGENTS.md`를 참조하므로, 공통 repo 규칙의 source of truth가 하나다.

### 2.2 분리 원칙

| 정보 | 위치 | 이유 |
|---|---|---|
| 모든 작업에서 필요한 repo 규칙 | `AGENTS.md` | 항상 로드돼도 부담이 작고, 도구 공통 규칙이다 |
| Claude Code용 import bridge | `CLAUDE.md` | `@AGENTS.md`만 두면 중복을 피한다 |
| 긴 제품/기술 판단 | `docs/*.md` | 항상-온 context에 넣기에는 길고, source of truth로 링크하는 편이 낫다 |
| 반복 실행 절차 | Codex Skill / Claude Skill | progressive disclosure로 필요할 때만 로드한다 |
| 팀 공유 설치 단위 | Plugin | skill, hook, MCP, 설정을 묶어 배포할 수 있다 |

Codex 공식 문서는 Skills를 반복 workflow용 authoring format으로, Plugins를 installable distribution unit으로 본다. Claude Code 공식 문서도 개인/프로젝트 실험은 standalone `.claude/`, 공유와 버전 관리는 plugin을 권장한다.

## 3. Provenance 입력 신호

### 3.1 신호 우선순위

표준 입력은 `.codereplay/input/targets.jsonl`의 `target_added` 이벤트다. 어떤 source가 그 이벤트를 만드는지는 다음 우선순위로 본다.

| Source | 신뢰도 | 비용 | MVP 적용 |
|---|---:|---:|---|
| 사용자가 CLI로 직접 추가한 target | 높음 | 중 | P0 |
| 외부 에이전트가 target event를 직접 append (`target import`) | 높음 | 중 | P0 |
| commit trailer / commit message marker → 사용자가 import 명령으로 변환 | 중 | 낮음 | P0 |
| Codex / Claude Code session id pointer를 provenance에 첨부 | 중 | 중 | P1 |
| transcript/log 직접 파싱 → target 변환 | 중 | 높음 | P2 |
| LLM이 "AI가 쓴 코드 같다"로 추론한 hunk | — | — | 비-목표 |

LLM에게 "이 코드는 AI가 쓴 것 같은가"를 묻는 방식이나 코드 스타일 기반 authorship classifier는 사용하지 않는다. `docs/local-vs-api-llm.md` §6과 `docs/ast-diff-tools.md` §5의 결론처럼, agent-assisted 여부는 모델 인상비평이 아니라 사용자 표시, trailer, 외부 도구가 명시적으로 남긴 provenance로만 처리한다.

### 3.2 Target event provenance 필드

P0 표준 입력은 `docs/mvp-spec.md`의 `target_added` 스키마다. agent 연동이 만들 때 채워야 할 핵심 필드는 다음과 같다.

- `provenance.source`: `user`, `codex`, `claude-code`, `cursor`, `copilot` 등 어떤 주체가 이 target을 만들었는지.
- `provenance.label`: `agent-assisted` 등 학습 가중치 입력. Code Replay는 이 라벨을 추론하지 않고 입력 그대로 받는다.
- `note` (선택): 사용자가 왜 이 target을 골랐는지에 대한 메모.
- `priority` (선택): 같은 실행에서 우선순위 정렬에 사용한다.

P0 표준 입력은 단일 hunk 단위 JSON이 아니라 `targets.jsonl` event log다. 과거 리서치에서 제안했던 `.codereplay/agent-hunks.json` snapshot 포맷은 P0 표준 입력에서 빠진다. 비슷한 정보가 필요하면 외부 도구가 그 데이터로 `target_added` 이벤트들을 만들어 `targets.jsonl`에 append하고, 활성 상태는 `.codereplay/cache/active-targets.json`에서 본다.

### 3.3 Commit trailer 권장

agent가 commit을 만들거나 사용자가 agent-assisted commit을 표시하고 싶을 때 trailer를 쓴다.

```text
Code-Replay-Agent-Assisted: true
Code-Replay-Provider: codex
Code-Replay-Session: local-session-id
```

이 trailer는 표준 Git trailer 형식이라 CLI에서 읽기 쉽고, 개인 repo 밖으로 push돼도 코드 본문보다 민감도가 낮다. trailer는 그 자체로 Code Replay의 입력이 아니라, **사용자나 외부 도구가 `target import` 흐름으로 변환해 `targets.jsonl`에 append할 때 사용하는 보조 신호**다. private repo에서는 session id가 외부 시스템 식별자로 이어질 수 있으므로 export 시 제거 옵션이 필요하다.

## 4. Replay mode 실행 경계

### 4.1 Replay mode의 목표

Replay mode는 agent가 정답을 다시 써주는 시간이 아니다. 사용자가 먼저 구현하고, 이후 원본 diff와 비교해야 한다.

필수 동작:

1. 원본 solution hunk를 숨긴다.
2. 요구사항, 테스트 의도, public API, 실패 조건만 보여준다.
3. 사용자가 직접 답을 제출한다.
4. 제출 후에만 원본 hunk, AST diff, 테스트 결과, 카드 피드백을 보여준다.

### 4.2 Codex 실행 경계

Codex에서는 다음 운영을 권장한다.

| 목적 | 권장 surface |
|---|---|
| repo 규칙 | `AGENTS.md` |
| 반복 replay 생성 | `.agents/skills/code-replay/SKILL.md` |
| 팀 배포 | Codex plugin |
| 실행 권한 | `workspace-write + on-request` 또는 docs-only profile |
| 외부 네트워크 | 기본 off, cloud 모델 opt-in 때만 명시 |
| secret hunk | 무조건 로컬 처리 |

Codex 공식 문서 기준으로 `workspace-write`는 로컬 작업에 적합한 낮은 마찰 모드이고, `on-request`는 sandbox 밖 작업이 필요할 때 사용자 승인을 받는다. code-replay는 private PR을 다루므로 `danger-full-access + never`를 기본값으로 두면 안 된다.

Codex Skill의 역할은 다음 정도로 제한한다.

- 사용자가 명시적으로 전달한 변경에 대해 `target_added` event 초안을 만들어 `.codereplay/input/targets.jsonl`에 append한다.
- 유효한 target에서 P0 replay task와 review card 후보를 생성한다.
- 생성 결과를 `docs/llm-quiz-hallucination.md` 검증 단계에 따라 검증한다.
- `fast mode`는 즉석 연습용으로만 사용한다. 검증 전 결과는 review card로 저장하지 않고 spaced review 일정에도 넣지 않는다.

### 4.3 Claude Code 실행 경계

Claude Code에서는 다음 운영을 권장한다.

| 목적 | 권장 surface |
|---|---|
| repo 공통 규칙 | `CLAUDE.md`에서 `@AGENTS.md` import |
| 빠른 실험 | `.claude/skills/code-replay/SKILL.md` |
| 팀 배포 | Claude Code plugin |
| 자동 차단/검증 | permissions + hooks |
| 배경 수집 | plugin monitor 또는 hook, 단 민감 정보 저장 금지 |

Claude Code 공식 문서는 `CLAUDE.md`를 항상 로드되는 project context로, Skills를 reusable knowledge/workflow로, Hooks를 lifecycle event 자동화로, Plugins를 배포 단위로 설명한다. code-replay에서는 이 계층을 그대로 쓴다.

Hook은 유용하지만 학습 ground truth가 아니다. 예를 들어 `PreToolUse` hook으로 replay 중 원본 solution 파일을 읽으려는 동작을 막을 수는 있지만, 사용자가 실제로 이해했는지는 hook이 아니라 제출 답안, 테스트 결과, review 결과, 카드 결과가 판단해야 한다.

## 5. Privacy boundary

### 5.1 저장하는 것

- hunk 단위 provenance metadata
- card id, concept id, source commit hash
- 사용자 flag: 잘못된 카드 / 개선 가능 / 좋은 카드
- concept merge/split label
- replay 제출 결과와 채점 결과

### 5.2 기본 저장하지 않는 것

- 전체 agent transcript
- 원본 prompt 전문
- cloud model raw response 전문
- secret 후보 hunk 내용
- 외부 서비스 access token, 내부 URL, 고객 데이터

필요한 경우에도 transcript는 `session_ref` pointer만 저장하고, 사용자가 명시적으로 export를 선택할 때만 복사한다.

### 5.3 Cloud opt-in

`docs/local-vs-api-llm.md`의 원칙을 따른다.

- 기본은 로컬 모델.
- cloud는 사용자 opt-in.
- secret/key/internal URL/customer data 패턴이 감지된 hunk는 cloud로 보내지 않는다.
- cloud 사용 결과도 code-replay 자체 서버에는 보내지 않는다.

## 6. 생성/검증 loop와 agent 역할

agent가 잘하는 일:

- diff를 읽고 replay 과제 후보를 만든다.
- invariant/boundary/contract 위치를 찾아 빈칸 후보를 만든다.
- 개념 후보를 추출하고 `concept-identity` 매칭 큐에 넣는다.
- 카드 품질 검증을 자동화한다.

agent에게 맡기면 안 되는 일:

- 사용자가 이해했는지 단독 판정.
- agent-assisted 여부를 코드 스타일이나 LLM 인상비평으로 판정.
- 검증 안 된 fast-mode 결과를 review card로 저장하거나 spaced review 큐에 적재.
- 사용자의 private diff 데이터를 기본 cloud로 전송.

`docs/llm-quiz-hallucination.md`에 따라 카드 생성은 single-pass가 아니라 generation -> AST verification -> QA round-trip -> overgenerate-and-rank -> user flag 순서로 처리한다.

## 7. MVP 단계

| 시점 | 만든다 | 만들지 않는다 |
|---|---|---|
| MVP P0 | `targets.jsonl` append 흐름, commit trailer를 사용자 import로 변환하는 helper, Codex/Claude Skill 초안, replay mode docs-only profile | transcript parser, cloud sync, AI authorship classifier, snapshot 형태 agent-hunks.json 표준 입력 |
| MVP P1 | session id pointer를 provenance에 첨부, hook/monitor 기반 target event 보조 수집, plugin package | transcript 전문 저장, 조직 analytics |
| MVP P2 | provider별 adapter, target log export/import, local provenance eval, optional app/IDE panel | 중앙 서버 기반 학습 데이터 수집 |

## 8. 완료 조건

agent 연동이 구현됐다고 말하려면 최소한 아래가 가능해야 한다.

- 사용자나 외부 에이전트가 만든 변경이 `target_added` event로 `targets.jsonl`에 들어가고, `active-targets.json`이 그 입력에서 재생성된다.
- 사용자가 provenance를 정정하면 `target_superseded` 또는 새 `target_added`로 기록되고, 이후 흐름에서 정정된 값이 우선한다.
- replay mode에서 원본 solution을 제출 전까지 숨긴다.
- 표준 검증을 통과한 review card만 spaced review due 계산에 들어간다.
- fast mode 결과는 review card로 저장되지 않고 spaced review 일정에도 들어가지 않는다.
- secret 후보가 cloud 경로로 가지 않는다는 테스트가 있다.
- `.codereplay/` 디렉터리만으로 target log, replay/card, review log를 export할 수 있다.

## Sources

- 로컬 출처: `docs/product-direction.md`
- 로컬 출처: `docs/mvp-spec.md`
- 로컬 출처: `docs/topic.md`
- 로컬 출처: `docs/ast-diff-tools.md`
- 로컬 출처: `docs/local-vs-api-llm.md`
- 로컬 출처: `docs/llm-quiz-hallucination.md`
- 로컬 출처: `docs/concept-identity.md`
- Codex customization / Skills / Plugins: <https://developers.openai.com/codex/concepts/customization>
- Codex sandbox defaults: <https://developers.openai.com/codex/concepts/sandboxing#configure-defaults>
- Codex Skills: <https://developers.openai.com/codex/skills>
- Claude Code extension overview: <https://code.claude.com/docs/en/features-overview>
- Claude Code permissions: <https://code.claude.com/docs/en/permissions>
- Claude Code plugins: <https://code.claude.com/docs/en/plugins>
