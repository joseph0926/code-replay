# 시장 대안과 빈 자리

리서치 일자: 2026-05-15 (`docs/learning-science.md`, `docs/ai-learning-impact.md`, `docs/diff-to-curriculum.md` 결과 반영)

## 기존 대안

- **GitHub Copilot 학습 가이드** — inline suggestion을 끄고 Copilot을 튜터처럼 쓰도록 설정하는 방식을 안내한다. **코드 자체가 학습 자료로 변환되는 흐름은 없음.**
- **Cursor Learn / BugBot / Agent Review** — AI 결과물을 사후 검증·리뷰하는 UX(`.cursor/rules/`, 로컬 diff 분석, 매니지드 PR 리뷰)를 표준화. **버그 검출이 목적이지, 본인 학습 자료 변환은 아님.** code-replay의 UX shell 참고 모델로는 매우 가깝다(`docs/diff-to-curriculum.md` §6).
- **CodeTeach** — 코드 스니펫이나 GitHub repo를 입력하면 interactive course, 빈칸 채우기, checkpoint, spaced repetition을 생성한다. **가장 가까운 인접 도구.** 단 입력이 "코드 스니펫/repo" 단위지 "내 PR/diff" 단위가 아니고, "내가 안 짠 부분에 가중치"라는 차별 컨텍스트가 없다.
- **Exercism** — 언어별 연습과 멘토링. 입력은 미리 만들어진 exercise. 본인 작업물 단위 학습 변환 없음.
- **CodeCrafters / build-your-own-x** — 실제 시스템을 직접 재구현하며 학습. **"reconstruction" 프레임은 동일**하지만 입력이 "유명 시스템(Redis, git, ...)" 단위지 "내 PR" 단위가 아님.
- **Anki / Hashcards / Memorex / FSRS** — spaced repetition 저장 계층은 성숙. **카드는 인간이 수기로 만든다는 게 한계.** code-replay의 SRS 백엔드로는 후보(특히 Hashcards의 git-tracked `.md` 모델, 자세히는 `docs/diff-to-curriculum.md` §7).
- **AIED 2022 self-explanation question gen / MDPI 2025 Java fill-in-blank LLM 자동 생성** — 학술 prototype 수준에서 AST+LLM 자동 학습 자료 생성은 존재. **단, end-to-end product와 PR/diff 입력 컨텍스트는 부재.**

## 연구/시장 신호

### 생산성 지표 — 시그널 진동

- **METR 2025-07 RCT** (16명 / 246 task / 평균 5년 경력): AI 사용 시 task 시간 +19%(=느려짐). 사전 -24%, 사후 -20% 자가 예측과 정반대.
- **METR 2026-02 redesign 발표**: 후속 실험 noise로 redesign. 비공식 추정으로 지속 노출 시 +19% 생산성 가능성, 단 selection effect caveat 강조.
- **METR 2026-05-11 self-reported survey** 추가 발표.
- 해석: "AI = 무조건 느리다" narrative는 단순화됐다. 그러나 **속도 지표는 본 프로젝트의 정당화 축이 아니다**(`docs/ai-learning-impact.md` §1).

### 학습/이해 지표 — 일관된 신호 (정당화의 본축)

- **AI 사용 그룹 사후 quiz 점수 대조군 대비 -17%** (Psychology Today, Feb 2026). 순수 delegation 최저 / "코드+설명 요청" 중간 / AI 미사용 최고. → **mediator는 "설명을 본인 머리로 다시 통과시키느냐"**.
- **Anthropic 자체 연구** — AI assistance가 coding skill formation에 부정적 영향을 미친다는 동일 방향 신호. **vendor가 risk를 인정**.
- **The Augmentation Trap** (arxiv 2604.03501): "performance-extracting" vs "skill-preserving" deployment 분류. 처방으로 **structured unassisted practice**(조종사 hand-flying 비유). code-replay 정확히 이 처방의 product 형태.
- **ICER 2025 brownfield 실험**: Copilot으로 task -34.9% 시간이지만 **cognitive disengagement 위험** 동시 보고.

### Vibe coding 자체

- **ACM TechBrief on Vibe Coding** 발행 — policy 차원에서 mainstream화.
- **arxiv 2601.02410 The Vibe-Check Protocol** — cognitive offloading을 metric화하는 프로토콜.
- vibe coder 약 14%는 비전공자 / 접근성 동기. **새로운 vulnerable developer 계층** 형성 → code-replay 2차 페르소나 후보.

### 학습 자료 자동 생성의 기술 성숙

- AST 기반 학습 자료 생성: AIED 2022 self-explanation question gen, AST-T5, AST-based RAG chunking — **학계에서 8년+ 검증된 영역**.
- LLM 자동 retrieval practice 효과: 2025 데이터 사이언스 코스 RCT에서 **quiz 정확도 +16pp**(89% vs 73%).
- Code Knowledge Graph: CGM, KG-based code generation, Neo4j LLM KG Builder, llmgraph 등 **2025-2026 폭증 영역**.
- LLM 학습 측면 reconstruction: arxiv 2603.11103 "Understanding by Reconstruction" — **인간 학습 product화는 비어 있음** = code-replay의 정확한 빈자리.

## 비어 있는 틈 (재정리)

기존 대안과 인접 OSS를 합치면 빌딩 블록은 거의 다 있다(`docs/diff-to-curriculum.md` §9 gap analysis). 그러나 다음 정확한 조합은 비어 있다:

| 차원 | 기존 도구의 한계 | code-replay의 채움 |
|---|---|---|
| 입력 | 코드 스니펫(CodeTeach), 유명 시스템(CodeCrafters), 미리 만든 exercise(Exercism) | **개인 git repo에서 사용자가 명시적으로 전달한 target** |
| 가중치 | 모든 코드를 동등하게 다룸 | **사용자나 외부 연동이 표시한 agent-assisted target에 집중** (도구가 추론하지 않음) |
| 추출 | 인간 수기(Anki) 또는 단일 단위(AIED 2022는 함수 단위) | **전달받은 target에서 AST + LLM hybrid로 학습 단위 구성** |
| 변환 | 각각 따로 존재 | **replay task + verified review card + spaced 복습 큐로 묶음** |
| 처방 | Augmentation Trap이 처방으로 제시만 함 | **AI 사용 후 사후 unassisted replay를 product화** |
| 저장 | 별도 앱(Anki) 또는 cloud(CodeTeach) | **repo-local `.codereplay/`** (P0 기본 가이드: local untracked, git tracking 여부는 사용자가 결정) |
| UX | PR 리뷰(BugBot)는 버그 탐지가 목적 | **명시적으로 전달된 변경의 학습 빈자리** 패널 |

핵심: code-replay는 **"신기술"이 아니라 "검증된 빌딩 블록의 새 조합"**이다. 실행 위험은 낮고 narrative는 학술 백킹이 강하다.
