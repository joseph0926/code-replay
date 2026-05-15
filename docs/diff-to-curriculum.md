# Diff/PR → 학습 자료 변환: 기술 가용성과 빈 자리

리서치 일자: 2026-05-15
스코프: 초기 6개 산출물 후보(변경 요약 / 개념 목록 / 빈칸 / 재구현 / 개념 맵 / 복습 질문)를 어떤 빌딩 블록으로 만들 수 있는지, code-replay가 새로 메우는 조합이 무엇인지 정리. 현재 결정 기준은 `docs/product-direction.md`, P0 실행 계약은 `docs/mvp-spec.md`이며, 이 문서의 "모든 diff 자동 변환" 표현은 그 결정 위에서 다시 읽는다.

## 현재 결정 기준

- 입력은 사용자나 외부 연동이 명시적으로 전달한 `target`이다. 전체 diff를 도구가 자동으로 학습 단위로 만들거나 우선순위로 선별하지 않는다.
- 산출물은 `replay task`와 `review card`로 분리되고, P0에서 review card는 `fill_blank` / `explain_decision` 두 타입만 사용한다.
- verified card만 spaced review due 계산에 들어간다. fast mode 결과는 즉석 연습용으로, 검증 전에는 review card로 저장하지 않고 spaced review에도 들어가지 않는다.
- 개념 맵, reconstruction 자동 채점, dashboard 같은 산출물은 P0의 비-목표이며 후순위(P1/P2)로 둔다.

따라서 아래 6개 산출물 매핑은 "P0가 한 번에 다 만든다"가 아니라 "각 산출물이 P0/P1/P2 중 어디에 위치하는지를 정리하는 historical 노트"로 읽는다.

## 한 줄 결론

- **빌딩 블록은 거의 다 있다.** PR 요약 / 코드 분석 / AST 기반 concept extraction / Code Knowledge Graph / 자동 quiz 생성 / spaced repetition 저장소까지 각각 OSS 또는 학술 prototype 수준에서 존재.
- **그러나 "사용자가 명시적으로 전달한 target → verified review card 큐"의 end-to-end 조합은 비어 있다.** Anki, Cursor Learn, BugBot, llmgraph, AST-T5 등은 인접하지만 어느 것도 이 정확한 파이프라인을 구현하지 않는다.
- **arxiv 2603.11103 "Understanding by Reconstruction"이 `docs/topic.md`의 "처음부터 다시 구현" 가설을 LLM pretraining 맥락에서 정확히 같은 프레임으로 재확인했다.** 이 논문은 LLM 학습 측면이지 인간 학습 측면은 아니다. **인간 학습 측면에서 이 프레임을 product화한 도구는 아직 없다는 게 code-replay의 빈 자리**이며, P0에서는 reconstruction을 초기 `replay task`로 다룬다.
- **P0 구현은 "기존 빌딩 블록 조합"으로 가능하다.** 새 모델 학습 불필요. 명시적 target 입력 → AST 검증 → LLM prompt로 P0 산출물(`replay task` + `fill_blank`/`explain_decision` review card) 생성 → repo-local `.codereplay/` 저장 → verified card만 spaced 스케줄 재출제. 모든 단계가 기성 도구로 가능.

## 1. PR/diff → 자연어 요약·리뷰 (성숙)

가장 성숙한 영역. code-replay에서 "변경 요약"과 컨텍스트 추출 기반 컴포넌트.

- **PR description / commit message 자동 생성**: T5 기반 PR description 생성(arxiv 2408.00921), git-llm, pr-auto, samuelliedtke 블로그(chain-of-thought + structured output)까지 패턴 정착.
- **PR 자동 리뷰**: BurnyCoder/llm-pr-review-gh-action(요약 + 이슈 + 보안 + 품질), LLM Code Review marketplace action, CodeRabbit 등.
- **로컬 LLM 기반 diff 분석**: udiedrichsen gist의 `analyze-changes`처럼 로컬 LLM으로 diff 인사이트 생성하는 패턴 다수.

→ code-replay 함의: **변경 요약 자체는 P0의 단독 산출물이 아니다.** `docs/topic.md`에서 정리한 대로 요약은 replay task와 review card 생성을 돕는 보조 맥락으로만 둔다. 기존 prompt 패턴은 재사용 가능하지만, 사용자에게 단독 출력으로 노출하지는 않는다.

## 2. PR/diff → 테스트 자동 생성 (성숙 중)

- arxiv 2501.11086 "Can LLM Generate Regression Tests for Software Commits?" — diff를 입력으로 회귀 테스트를 LLM이 생성하는 접근. 학습 도메인이 아닌 검증 도메인이지만, **"diff → 의도 추출 → 테스트 케이스 생성"이라는 파이프라인 자체가 검증됐다.**

→ code-replay 함의: 같은 파이프라인의 출력을 P0의 `replay task` 또는 `explain_decision` review card 시드로 변환할 수 있다. spec 단위 reconstruction 자동 채점은 P0 비-목표이므로, 이 빌딩 블록은 P1 이후 채점 보강 후보로 둔다.

## 3. AST 기반 Concept/Pattern Extraction (학계 성숙)

- **AST 패턴으로 학생 코드 디버깅**(SpringerLink ITS 2017): AST 패턴을 features로 학습해 옳은/틀린 프로그램 구분 → 교육 도메인에서 AST 활용은 8년 이상 검증.
- **AST-T5**(arxiv 2401.03003): AST를 pretraining에 통합한 코드 모델. structure-aware embedding이 가능하다는 학계 합의.
- **AST-based chunking**(VXRL Medium): RAG에서 AST 경계로 코드 청크를 자르는 게 token-naive split보다 우수. **개념 단위 추출의 표준 기법으로 굳어지는 중.**
- **자동 question generation for self-explanation**(SpringerLink AIED 2022, "Automatic Question Generation for Scaffolding Self-explanations for Code Comprehension") — **AST + LLM으로 코드 self-explanation 질문을 자동 생성하는 직접 prior art.**

→ code-replay 함의: P0에서는 복잡한 concept graph 대신 안정적인 `concept` label/id에 한정한다. AST 노드 + LLM 질문 조합은 review card 생성을 보조하는 데 쓰고, 노드 그래프 자체를 산출물로 노출하지 않는다. AIED 2022 패턴은 `fill_blank` / `explain_decision` 시드로 재해석한다.

## 4. Code Knowledge Graph (2025-2026 활발)

- **CGM (Code Graph Model)**: 코드 repo의 semantic + structural info를 LLM attention에 통합. node attribute → LLM input space mapping 어댑터. **repo-level 작업의 새 표준 아키텍처.**
- **KG-Code-Generation**(arxiv 2505.14394): 클래스/함수/모듈의 hierarchy/dependency/usage를 KG로 표현 → context-rich retrieval로 코드 생성 정확도 향상.
- **OSS 빌딩 블록**:
  - **Neo4j LLM Knowledge Graph Builder** — 문서/코드 chunk → embedding → 그래프. 즉시 사용 가능.
  - **Cognee** — graph + vector embedding 통합 메모리 레이어. LLM app용.
  - **llmgraph** — GraphML/GEXF/HTML로 KG 출력. ChatGPT/LiteLLM 백엔드.
- **2510.20345** — LLM-empowered KG construction survey가 발행될 정도로 분야 성숙.

→ code-replay 함의: "개념 맵" 또는 full code knowledge graph는 P0의 비-목표다. `docs/product-direction.md`에서 P0는 안정적인 `concept` label/id에 한정한다. KG 빌딩 블록은 후순위 설계로 보존하되, P0 산출물에 포함하지 않는다. KG가 들어온다면 학습의 "인덱스"이지 그 자체가 학습 산출물은 아니라는 위치 그대로 유지.

## 5. 자동 Question/Exercise Generation (직접 prior art)

- **AIED 2022 — Automatic Question Generation for Scaffolding Self-explanations for Code Comprehension** (SpringerLink): AST 기반 코드 자기설명 질문 자동 생성. **§3에서 언급했지만 다시 강조** — code-replay의 "개념 quiz / 빈칸 / 재구현 안내 질문"의 학술적 reference architecture.
- **Java element fill-in-the-blank LLM 자동 생성**(MDPI Electronics 14권 11호, 2025): docs/learning-science.md §2에서도 인용. **fill-in-the-blank 자동 생성이 production-grade 수준에 도달**했다는 직접 근거.
- **FLAIRS — Generating Distractors for Code Completion**: 객관식 distractor 자동 생성. quiz 품질 향상 핵심.

→ code-replay 함의: P0의 `fill_blank` review card 자동 생성은 **선례가 명확한 영역**. 위험은 LLM hallucination이며, 검증되지 않은 card는 spaced review 큐에 넣지 않는다. 정답 판정 로직(AST diff로 token-level grading)을 prompt 외부에 두고, fast mode 결과는 review card로 저장하지 않는다(`docs/llm-quiz-hallucination.md` 참고).

## 6. UX 측면 prior art — Cursor Learn / BugBot / Agent Review

- **Cursor Learn**: Cursor 공식 튜토리얼. "Reviewing AI-generated code" 모듈, BugBot, cloud agent 병렬 테스트 등 **AI 결과물을 사후 검증/학습 흐름으로 풀어내는 UX**의 reference.
- **BugBot**: 매니지드 PR 자동 리뷰. `.cursor/rules/bugbot.md`로 팀 룰 주입. **"PR 단위 인텔리전스"의 UX 표준화 사례.**
- **Agent Review**: 로컬 diff에 대해 push 전 agent가 분석. **로컬 우선 diff 분석 흐름** — code-replay의 1차 페르소나(개인 repo)와 동일.
- **"Self-Learning Code Review"** (Elementor Engineers 블로그): "AI agent가 PR 리뷰 코멘트로부터 인간처럼 학습할 수 있다면?"이라는 질문. **방향은 반대**(code-replay는 AI → 인간 학습, 이 글은 인간 피드백 → AI 학습)지만, "PR을 학습 신호로 본다"는 프레임은 같음.

→ code-replay 함의: P0 UX는 **CLI + repo-local `.codereplay/` 파일 저장**이며 dashboard나 IDE extension은 P0에서 만들지 않는다(`docs/product-direction.md` non-goals). Cursor의 BugBot UX는 "PR 단위 인텔리전스"의 인접 참고로 두되, P0는 그보다 단순한 형태로 출발한다. 차이 frame은 동일: BugBot이 "이 PR의 버그"를 본다면, code-replay는 "사용자가 명시적으로 전달한 변경의 학습 빈자리"를 본다.

## 7. Spaced Repetition 저장 계층 (성숙)

- **Anki**: open-source SRS의 사실상 표준. add-on ecosystem 풍부. external script로 카드 주입 가능 → **MVP의 복습 큐 백엔드 후보 1순위.**
- **Hashcards**: plain-text SRS. **Markdown + git-tracked**. flashcard collection을 git repo로 관리. **code-replay 철학(로컬 우선, 개인 repo 단위)과 정렬 완벽.**
- **Memorex**: Phoenix LiveView 기반 SRS, Markdown + image/text 저장.
- **FSRS**(Free Spaced Repetition Scheduler): SM-2를 대체하는 현대 알고리즘. Anki에 통합됨.

→ code-replay 함의: **자체 SRS 알고리즘 구현 X.** Anki integration 또는 Hashcards 스타일 plain-text 어느 쪽이든 P0 범위에서 가능하고, P0는 Hashcards 스타일(repo 안 `.codereplay/cards/*.md`)을 따른다. `.codereplay/`의 git tracking 여부는 사용자가 결정하며, P0 기본 가이드는 local untracked 사용이다. scheduling 알고리즘은 검증된 FSRS/SM-2를 사용한다.

## 8. "Understanding by Reconstruction" — 가장 결정적 prior art

- **arxiv 2603.11103 "Understanding by Reconstruction: Reversing the Software Development Process for LLM Pretraining"**.
- 핵심: **LLM이 코드를 더 깊이 이해하게 하려면, 완성된 코드만 보여주지 말고 "역으로 개발 과정을 재구성하게 시켜라"** — 즉, requirements → spec → 구현 단계를 reverse-engineer하게 학습시키는 접근.
- docs/topic.md의 "처음부터 다시 구현하기 단계별 과제"와 **개념적으로 동일한 프레임**.
- **차이**: 이 논문은 LLM 학습용. code-replay는 **인간 학습용으로 같은 프레임을 적용**. 인간 학습 측면 product는 아직 없다.

→ code-replay 함의: 이 논문이 강력한 narrative 카드. "AI도 reconstruction으로 더 잘 배운다는데, 인간은?" 식으로 `docs/topic.md`의 직관에 학술 백킹을 붙일 수 있다. **P0의 `replay task` 산출물은 이 논문의 인간 학습 버전 implementation으로 포지셔닝 가능**하며, reconstruction 자동 채점은 P0 비-목표로 후순위(P1) 둔다.

## 9. Gap Analysis — code-replay가 새로 만드는 조합

| 단계 | 기성 도구 | 부족한 부분 | code-replay 추가 | P0/P1/P2 |
|---|---|---|---|---|
| target 입력 | git, gh CLI | — | 사용자/외부 연동이 명시적으로 전달하는 `target_added` event log | P0 |
| 변경 요약·자연어화 | LLM PR review action 다수 | — | replay/card 생성을 돕는 보조 맥락으로만 사용 | P0 |
| 학습 대상 추출 | AST-T5, AIED 2022 question gen | **명시적 target + 개인 작업 컨텍스트 결합 부재** | target 내부에서 작은 학습 포인트 추출 | P0 |
| `fill_blank` card | MDPI 2025 Java fill-in-blank, FLAIRS distractor | **AST + LLM hybrid 정답 검증 부재** | AST diff 기반 grading + 검증 게이트 | P0 |
| `explain_decision` card | self-explanation 패턴 (학계) | code 도메인 product화 부재 | 조건/경계/불변식/설계 선택 설명 카드 | P0 |
| `replay task` | (없음) | **인간 학습용 reconstruction product 부재** | arxiv 2603.11103을 product화 (초기 문제 형태) | P0 |
| reconstruction 자동 채점 | HumanEval/Copilot eval pattern | code-replay 도메인 적용 부재 | AST 동등성 + 테스트 통과율 + LLM judge | P1 |
| concept graph | Neo4j KG Builder, llmgraph, CGM | 학습 큐와 연결 부재 | P0에서는 단순 `concept` label/id에 한정 | P2 |
| 복습 큐 / 스케줄 | Anki, Hashcards, FSRS | verified card 게이트 부재 | verified card만 spaced review due에 포함 | P0 |
| UX shell | Cursor Learn / BugBot 패턴 | 학습 도메인 transposition 부재 | CLI + repo-local 파일 (dashboard/IDE extension은 후순위) | P0 (CLI) / P2 (dashboard·IDE) |

**결론**: 모든 컴포넌트가 인접 도구로 존재하지만, "**사용자가 명시적으로 전달한 agent-assisted target → AST + LLM hybrid로 학습 포인트 구성 → `replay task` + verified `review card` → 검증된 카드만 spaced review 큐 → repo-local `.codereplay/` 저장**" 이 정확한 조합은 비어 있다. **code-replay의 venture는 "신기술 발명"이 아니라 "검증된 빌딩 블록 조합"**이다 — 이건 좋은 신호다 (실행 위험 낮음).

## 10. P0 기술 스택 후보 (근거 기반)

| 컴포넌트 | 선택지 | 근거 | P0/P1/P2 |
|---|---|---|---|
| target 입력 / 검증 | git CLI + tree-sitter로 baseCommit/headCommit/lineRange가 diff와 맞는지 검증 | `docs/mvp-spec.md` target 검증 | P0 |
| diff 파싱 | git CLI + tree-sitter (언어별 AST) | AST-T5 / AST-based chunking에서 검증 | P0 |
| LLM 호출 | 로컬 Ollama default + 명시적 opt-in cloud (Anthropic 우선) | `docs/local-vs-api-llm.md` privacy 원칙 | P0 |
| 학습 포인트 추출 | tree-sitter + LLM prompt (AIED 2022 패턴) | 직접 prior art | P0 |
| `fill_blank` 생성 | LLM + AST diff 기반 정답 검증 + verified gate | MDPI 2025 패턴 | P0 |
| `explain_decision` 생성 | LLM + rubric 기반 검증 | self-explanation 패턴 | P0 |
| reconstruction 자동 채점 | AST 동등성 + 테스트 통과율 + LLM judge fallback | `docs/learning-metrics.md` §4 | P1 |
| concept graph | Neo4j LLM KG Builder 또는 llmgraph | OSS 성숙도 | P2 |
| SRS 백엔드 | Hashcards 스타일 .md (`.codereplay/cards/*.md`) + FSRS 라이브러리 | repo-local / 자체 알고리즘 회피 | P0 |
| UX shell | CLI 우선 (P0) → 후속 단계에서 dashboard/IDE extension 재검토 | `docs/product-direction.md` non-goals | P0 (CLI) / P2 (dashboard·IDE) |
| 결과 저장 | repo-local `.codereplay/` (P0 기본 가이드: local untracked, git tracking 여부는 사용자가 결정) | `docs/mvp-spec.md` repo-local 저장소 | P0 |

## 11. 미해결 질문 / 추가 리서치 후보

- **AST diff 라이브러리 선택**: difftastic, tree-sitter, gumtree 등 비교 필요. → 별도 spike.
- **로컬 LLM vs API**: privacy(개인 PR이 입력) 측면에서 로컬 LLM 강한 후보. ollama + Llama/Qwen-Coder 로컬 모델 성능 검증 필요.
- **AST 기반 정답 grading의 false positive율**: 같은 동작을 다르게 짜는 케이스. → MVP에서는 LLM judge fallback으로 처리하고, 나중에 측정.
- **학습 효과 측정 metric**: docs/learning-science.md §7과 docs/ai-learning-impact.md §6에서도 미해결로 남긴 것. **product 차원 핵심 미해결.**

## Sources

### PR/diff → 자연어
- T5 PR description (arxiv 2408.00921): <https://arxiv.org/html/2408.00921v1>
- LLM PR review GitHub Action: <https://github.com/BurnyCoder/llm-pr-review-gh-action>
- PR auto: <https://github.com/vblagoje/pr-auto>
- LLM Code Review marketplace: <https://github.com/marketplace/actions/llm-code-review>
- git-llm: <https://github.com/rsaryev/git-llm>
- analyze-changes (local LLM diff): <https://gist.github.com/udiedrichsen/979ae7ee3aaaae00cf3e15046ee5bba0>
- Anyscale LLM PR bot: <https://www.anyscale.com/blog/building-an-llm-powered-github-bot-to-improve-your-pull-requests>
- Commit message LLM (Samuel Liedtke): <https://www.samuelliedtke.com/blog/automatic-git-commit-message-llm-chain-of-thought-structured-output/>

### PR/diff → 테스트
- LLM 회귀 테스트 생성 (arxiv 2501.11086): <https://arxiv.org/html/2501.11086v1>

### AST 기반 학습/추출
- AST-T5 (arxiv 2401.03003): <https://arxiv.org/html/2401.03003v1>
- AST patterns for debugging student programs (SpringerLink): <https://link.springer.com/chapter/10.1007/978-3-319-61425-0_14>
- AST + LLM 자기설명 질문 생성 (AIED 2022, SpringerLink): <https://link.springer.com/chapter/10.1007/978-3-031-11644-5_77>
- AST for programming language survey (arxiv 2312.00413): <https://arxiv.org/pdf/2312.00413>
- AST-based RAG chunking (VXRL): <https://vxrl.medium.com/enhancing-llm-code-generation-with-rag-and-ast-based-chunking-5b81902ae9fc>
- ASTSDL functionality prediction: <https://link.springer.com/article/10.1007/s11432-021-3665-1>

### Code Knowledge Graph
- CGM (OpenReview): <https://openreview.net/forum?id=b98ODdeYq5>
- KG-based repo-level code gen (arxiv 2505.14394): <https://arxiv.org/html/2505.14394v1>
- LLM-empowered KG construction survey (arxiv 2510.20345): <https://arxiv.org/html/2510.20345v1>
- Neo4j LLM Knowledge Graph Builder: <https://neo4j.com/blog/developer/llm-knowledge-graph-builder-release/>
- llmgraph: <https://github.com/dylanhogg/llmgraph>
- Awesome-LLM-KG: <https://github.com/RManLuo/Awesome-LLM-KG>
- KG + AST 코딩 어시스턴트 (Cyril Sadovsky): <https://medium.com/@cyrilsadovsky/advanced-coding-chatbot-knowledge-graphs-and-asts-0c18c90373be>

### Question/Exercise 자동 생성
- Java element fill-in-the-blank LLM (MDPI Electronics 14:11, 2025): <https://www.mdpi.com/2079-9292/14/11/2261>
- FLAIRS distractors for code completion: <https://journals.flvc.org/FLAIRS/article/download/138995/144092/276184>

### Cursor / BugBot / Agent Review
- Cursor Learn (공식): <https://cursor.com/learn>
- Cursor Learn — Reviewing & Testing: <https://cursor.com/learn/reviewing-testing>
- Cursor agent best practices: <https://cursor.com/blog/agent-best-practices>
- Self-Learning Code Review (Elementor Engineers): <https://medium.com/elementor-engineers/the-self-learning-code-review-teaching-ai-cursor-to-learn-from-human-feedback-454df64c98cc>

### Spaced Repetition 저장 계층
- Anki: <https://github.com/ankitects/anki>
- Hashcards (plain-text + git): <https://borretti.me/article/hashcards-plain-text-spaced-repetition>
- spaced-repetition-flashcards GitHub topic: <https://github.com/topics/spaced-repetition-flashcards>
- Anki manual background: <https://docs.ankiweb.net/background.html>

### Reconstruction 프레임
- Understanding by Reconstruction (arxiv 2603.11103): <https://arxiv.org/html/2603.11103v1>
- LLM4Decompile (참고): <https://github.com/albertan017/LLM4Decompile>
