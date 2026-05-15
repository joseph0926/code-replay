# AI 시대의 개발 학습 복구 리서치

## 문제

AI가 코드를 대부분 작성하면 결과물은 빨리 나오지만, 내가 직접 코드를 작성하려 할 때 막힌다. 학습을 해도 실제 코드 변경, 개념, 설계 판단이 머릿속에 체계적으로 쌓이지 않는다.

이 직관은 2026년 시점에서 실증 근거를 얻었다. AI 사용 그룹의 사후 quiz 점수가 대조군 대비 평균 **-17%**로 보고됐고, 순수 delegation이 가장 낮은 점수였다(`docs/ai-learning-impact.md` §3). The Augmentation Trap 논문은 이 현상을 "performance-extracting deployment"로 부르고, 처방으로 **structured unassisted practice**(자동조종 시대의 hand-flying 비유)를 제시한다. code-replay는 이 처방의 product 형태다.

## 가능한 MVP

로컬 repo에서 git diff나 PR patch를 읽어 다음을 생성한다. 학습 과학 근거 강도에 따른 우선순위를 함께 표시한다(상세 정당화: `docs/learning-science.md` §6).

1. **[P0] "처음부터 다시 구현하기" 단계별 과제** — generation effect(d≈0.40) + Parsons 계열 + arxiv 2603.11103 "Understanding by Reconstruction"의 인간 학습 버전. AI 솔루션은 hint로만, 본인이 먼저 짠다.
2. **[P0] 빈칸 채우기 문제** — micro Parsons + fill-in-the-blank. 자동 생성이 production-grade에 도달(MDPI Electronics 2025). 단 **전략적 위치(invariant / boundary / contract)에만 빈칸**, 랜덤 토큰 마스킹은 효과 없음.
3. **[P0] 다음 복습 질문** — LLM 생성 retrieval practice의 +16pp RCT 근거. **단발 출제 X, spaced 큐 필수.** Anki/Hashcards 스타일 SRS 백엔드 + FSRS 스케줄러.
4. **[P1] 이해해야 할 개념 목록** — 그 자체는 학습 산출물 아님. 복습 큐의 키 단위이자 §1-3의 입력.
5. **[P1] 변경 요약** — 단독 노출은 "읽기"에 가까워 효과 약함. **Socratic prompt와 결합 필수** ("왜 이렇게 했나?", "이 줄이 없으면 무엇이 깨지는가?").
6. **[P2] 개념 간 연결 맵** — 단독 시각화는 효과 미약. 노드 클릭 → self-explanation + spaced 큐 연결 시에만 의미.

## 정리

개인적 용도, "코딩 교육 플랫폼"이 아니라 **"AI가 대신 만든 내 실제 작업을 다시 내 실력으로 돌려주는 로컬 도구"**로 좁힌다.

학술 framing으로는 **"AI 시대의 skill-preserving deployment 도구"**. 사용자는 AI로 PR을 만들고, code-replay가 그 PR을 사후에 본인 머리로 한 번 더 통과시키는 spaced 학습 큐로 변환한다.

## 차별점

기존 인접 도구(Copilot 학습 가이드 / Cursor Learn / CodeTeach / Exercism / CodeCrafters / Anki)와 비교했을 때 code-replay의 새 조합은 다음과 같다(`docs/alternative.md` "비어 있는 틈" 참조):

- 입력 = **개인 git repo의 내 PR/diff** (남이 만든 exercise나 유명 시스템 X)
- 가중치 = **"AI가 짠 부분 vs 내가 짠 부분"** 구분 (다른 도구 없음)
- 변환 = **재구현 + 빈칸 + spaced 복습 큐로 묶음**
- 처방 = **AI 사용 후 사후 reconstruction 강제** (Augmentation Trap 처방의 product화)
- 저장 = **로컬 git-tracked, 본인 repo 안 `.codereplay/` 디렉터리**

신기술 발명이 아니라 **검증된 빌딩 블록의 새 조합**이라 실행 위험은 낮다(`docs/diff-to-curriculum.md` §9 gap analysis).

## 근거 문서

- `docs/learning-science.md` — generation effect / retrieval practice / spaced repetition / Parsons / 코드 읽기 vs 쓰기 실증 근거. MVP 6개 산출물의 P0/P1/P2 배치 정당화.
- `docs/ai-learning-impact.md` — METR 2025-2026 시리즈, AI 사용 시 quiz -17%, Augmentation Trap, vibe coding 연구. 시장 정당화와 narrative 카드.
- `docs/diff-to-curriculum.md` — diff → 학습 자료 변환의 기술 빌딩 블록(AST, KG, OSS), prior art, gap analysis, MVP 기술 스택 후보.
- `docs/ast-diff-tools.md` — 구조 diff 백엔드와 AI vs human 작성 구분의 입력 신호.
- `docs/local-vs-api-llm.md` — private PR/diff 입력을 고려한 로컬 LLM default와 cloud opt-in 경계.
- `docs/learning-metrics.md` — card retention, concept mastery, reconstruction, transfer 4계층 metric.
- `docs/quantization-impact.md` — 로컬 LLM 양자화가 카드 생성, judge, structured output에 주는 영향.
- `docs/llm-quiz-hallucination.md` — LLM 생성 카드의 hallucination 방지와 검증 파이프라인.
- `docs/concept-identity.md` — PR 간 같은 개념을 같은 노드로 묶는 entity resolution / active learning 설계.
- `docs/agent-integration.md` — Codex / Claude Code provenance 수집, replay mode 권한 경계, skill/plugin 통합 전략.
- `docs/alternative.md` — 시장 대안과 빈 자리.

## 참고

- https://docs.github.com/en/get-started/learning-to-code/setting-up-copilot-for-learning-to-code
- https://www.trycursor.ai/
- https://codeteach.codes/
- https://exercism.org/
- https://codecrafters.io/
- https://github.com/codecrafters-io/build-your-own-x
- https://arxiv.org/abs/2507.09089
