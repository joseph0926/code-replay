# Codex / Claude Code 통합: agent provenance와 replay 실행 경계

리서치 일자: 2026-05-15
스코프: `docs/topic.md`의 핵심 차별점인 **"AI가 짠 부분 vs 내가 짠 부분"** 가중치를 실제 Codex / Claude Code 작업 흐름에서 어떻게 수집하고, code-replay의 재구현 모드를 어떻게 안전하게 실행할지 정한다. `docs/ast-diff-tools.md` §5의 "AI vs human 작성 구분"을 구현 가능한 데이터 계약으로 내리는 문서다.

## 한 줄 결론

- code-replay의 agent 통합 핵심은 "에이전트를 더 잘 코딩하게 하기"가 아니라 **에이전트가 만든 변경의 provenance를 로컬에 남기고, 사용자가 그 변경을 unassisted replay하게 만드는 것**이다.
- MVP P0는 transcript 전체 파싱이 아니라 **commit trailer + 사용자 명시 태그 + `.codereplay/agent-hunks.json`** 조합으로 시작한다. transcript/log 직접 파싱은 privacy와 도구별 포맷 변동 때문에 후순위다.
- Codex 쪽은 `AGENTS.md`를 repo 공통 규칙 source of truth로 두고, 반복 replay 워크플로는 Skill, 공유/설치 단위는 Plugin으로 승격한다. replay 실행은 `workspace-write + on-request` 같은 낮은 위험 permission profile에서 시작한다.
- Claude Code 쪽은 `CLAUDE.md`가 `@AGENTS.md`를 참조하는 현재 구조를 유지하고, 빠른 실험은 `.claude/skills`, 공유형은 plugin으로 둔다. replay 중 AI solution 노출 차단은 permissions + hook으로 보조하되, 학습 ground truth는 사용자 제출과 로컬 결과 파일이 맡는다.
- 모든 provenance, 카드 품질 피드백, concept merge label은 **로컬 `.codereplay/`에만 저장**한다. code-replay 자체 서버나 외부 analytics로 보내지 않는다.

## 1. 왜 agent integration이 필요한가

code-replay의 차별점은 단순히 PR diff를 학습 자료로 바꾸는 것이 아니라, **내가 직접 고민하지 않은 변경을 더 높은 우선순위로 replay하는 것**이다.

이미 문서화된 핵심 전제:

- `docs/topic.md` — 입력은 개인 repo의 PR/diff이고, 가중치는 "AI가 짠 부분 vs 내가 짠 부분" 구분이다.
- `docs/ast-diff-tools.md` — AST diff는 누가 썼는지를 알 수 없고, provenance는 별도 레이어가 필요하다.
- `docs/local-vs-api-llm.md` — PR 전체가 입력이므로 privacy가 모델 성능보다 우선이다.
- `docs/llm-quiz-hallucination.md` — 생성 카드 품질은 검증과 사용자 피드백 루프 없이는 retention metric을 오염시킨다.

따라서 agent integration은 세 가지 책임을 가진다.

1. 어떤 hunk가 agent assistance를 받았는지 기록한다.
2. replay 중 사용자가 원본 solution을 너무 일찍 보지 않도록 실행 경계를 만든다.
3. 카드/개념/재구현 결과에 대한 사용자 피드백을 다음 생성 품질에 반영한다.

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

| 신호 | 신뢰도 | 비용 | MVP 적용 |
|---|---:|---:|---|
| 사용자 명시 태그 | 높음 | 중 | P0 |
| commit trailer / commit message marker | 중 | 낮음 | P0 |
| `.codereplay/agent-hunks.json` | 높음 | 중 | P0 |
| Codex / Claude Code session id pointer | 중 | 중 | P1 |
| transcript/log 직접 파싱 | 중 | 높음 | P2 |
| "AI가 쓴 코드 같다" LLM 판정 | 낮음 | 중 | 사용 금지 |

LLM에게 "이 코드는 AI가 쓴 것 같은가"를 묻는 방식은 사용하지 않는다. `docs/local-vs-api-llm.md` §6과 `docs/ast-diff-tools.md` §5의 결론처럼, 작성 주체 판정은 모델 인상비평이 아니라 blame, trailer, 사용자 태그, session metadata로 처리해야 한다.

### 3.2 `.codereplay/agent-hunks.json` 스키마

MVP P0의 핵심 파일이다. 사용자가 직접 편집할 수 있어야 하며, 자동 생성된 값은 항상 override 가능해야 한다.

```json
{
  "version": 1,
  "repo": "owner/project",
  "base_ref": "main",
  "head_ref": "feature/replay",
  "commit_sha": "abc123",
  "generated_at": "2026-05-15T09:00:00+09:00",
  "hunks": [
    {
      "id": "hunk_001",
      "file": "src/example.ts",
      "old_start": 10,
      "old_lines": 4,
      "new_start": 10,
      "new_lines": 12,
      "agent_assisted": true,
      "provider": "codex",
      "source_signal": ["user_tag", "commit_trailer"],
      "confidence": 0.9,
      "session_ref": {
        "tool": "codex",
        "session_id": "local-session-id",
        "transcript_path": null
      },
      "user_override": null,
      "notes": "사용자가 replay 대상으로 표시"
    }
  ]
}
```

필드 규칙:

- `agent_assisted`: 학습 가중치의 직접 입력이다.
- `provider`: `codex`, `claude-code`, `cursor`, `copilot`, `unknown`, `human` 중 하나로 시작한다.
- `source_signal`: 왜 그렇게 판정했는지 남긴다. 예: `user_tag`, `commit_trailer`, `session_ref`, `blame_time_heuristic`.
- `confidence`: 자동 판정 신뢰도다. 사용자 명시 태그는 0.9 이상, 휴리스틱은 0.6 이하로 둔다.
- `session_ref.transcript_path`: 기본값은 `null`이다. transcript 본문을 자동 저장하지 않는다.
- `user_override`: 사용자가 `agent_assisted` 판정을 바꾸면 원래 판정 대신 여기를 source of truth로 본다.

### 3.3 Commit trailer 권장

agent가 commit을 만들거나 사용자가 agent-assisted commit을 표시하고 싶을 때 trailer를 쓴다.

```text
Code-Replay-Agent-Assisted: true
Code-Replay-Provider: codex
Code-Replay-Session: local-session-id
```

이 trailer는 표준 Git trailer 형식이므로 CLI에서 읽기 쉽고, 개인 repo 밖으로 push돼도 코드 본문보다 민감도가 낮다. 그래도 private repo에서는 session id가 외부 시스템 식별자로 이어질 수 있으므로 export 시 제거 옵션이 필요하다.

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

- diff/PR 입력을 받아 `.codereplay/agent-hunks.json` 초안을 만든다.
- P0 카드와 replay 과제를 생성한다.
- 생성 결과를 `docs/llm-quiz-hallucination.md` 검증 단계에 따라 검증한다.
- `Fast mode` 결과는 저장하지 않고 drill only로 둔다.

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
- AI 작성 여부를 코드 스타일만 보고 판정.
- 검증 약한 fast-mode 카드를 FSRS 큐에 저장.
- 사용자의 private PR 데이터를 기본 cloud로 전송.

`docs/llm-quiz-hallucination.md`에 따라 카드 생성은 single-pass가 아니라 generation -> AST verification -> QA round-trip -> overgenerate-and-rank -> user flag 순서로 처리한다.

## 7. MVP 단계

| 시점 | 만든다 | 만들지 않는다 |
|---|---|---|
| MVP P0 | `.codereplay/agent-hunks.json`, commit trailer parser, user override, Codex/Claude Skill 초안, replay mode docs-only profile | transcript parser, cloud sync, automatic AI authorship classifier |
| MVP P1 | session id pointer, hook/monitor 기반 provenance 보조 수집, plugin package, concept merge batch UI | transcript 전문 저장, 조직 analytics |
| MVP P2 | provider별 adapter, export/import, local provenance eval, optional app/IDE panel | 중앙 서버 기반 학습 데이터 수집 |

## 8. 완료 조건

agent integration이 구현됐다고 말하려면 최소한 아래가 가능해야 한다.

- 같은 PR diff에서 agent-assisted hunk와 human hunk를 분리해 표시한다.
- 사용자가 hunk 판정을 수정하면 다음 생성부터 그 판정이 우선한다.
- replay mode에서 원본 solution을 제출 전까지 숨긴다.
- Standard 이상 검증을 통과한 카드만 FSRS 큐에 저장한다.
- secret 후보 hunk가 cloud 경로로 가지 않는다는 테스트가 있다.
- `.codereplay/` 디렉터리만으로 provenance, 카드, concept, review log를 export할 수 있다.

## Sources

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
