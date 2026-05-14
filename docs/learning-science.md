# 학습 과학 근거: code-replay MVP 산출물 우선순위 결정용

리서치 일자: 2026-05-15
스코프: docs/topic.md MVP의 6개 산출물(변경 요약 / 개념 목록 / 빈칸 채우기 / 처음부터 재구현 / 개념 맵 / 복습 질문) 중 어떤 것이 학습 과학 근거가 강한지, 어떻게 설계해야 효과가 살아나는지 정리.

## 한 줄 결론

- **재구현 + 빈칸 + 복습 질문은 근거가 매우 강하다.** Generation effect, retrieval practice, Parsons + fill-in-the-blank, spaced repetition은 모두 독립적 RCT/메타분석에서 효과가 검증됐다.
- **변경 요약 + 개념 맵은 그 자체로는 약하다.** "읽기"에 가깝기 때문. self-explanation prompt나 Socratic 가이드를 결합해야 효과가 살아난다.
- **따라서 MVP는 "재구현 과제 + 빈칸 채우기 + spaced 복습 질문" 세 산출물을 중심에 두고, 변경 요약/개념 맵은 그 산출물의 입력 컨텍스트로 위치시키는 게 근거상 합리적이다.**

## 1. Generation effect — "재구현하기"의 근거

- **요지**: 같은 정보라도 직접 생성/완성한 학습자가 단순 읽은 학습자보다 훨씬 잘 기억한다. Slamecka & Graf (1978) 이후 50년간 반복 검증된 효과.
- **효과 크기**: 자기 생성 정보 retention의 효과 크기 약 **d = 0.40** (중간 크기). 단순 읽기 대비 통계적으로 강하다.
- **신경 근거**: 생성 활동은 인코딩 단계에서 더 넓은 신경 회로를 활성화시킨다는 fMRI 근거가 있다.
- **활용 형태**: cloze(빈칸) 완성, self-explanation, problem posing, 자기 정의 작성. "전체 노트 제공 X, 개념적으로 중요한 곳에만 전략적 빈칸 O".
- **code-replay 함의**: "AI가 만든 diff를 보고 이해하기"보다 **"AI가 만든 diff를 빈칸 두고 다시 채우기 / 처음부터 다시 짜기"가 generation effect 측면에서 직접적으로 정당화된다.** 이게 MVP의 핵심 차별점이어야 한다.

## 2. Parsons problems + fill-in-the-blank — 코드 도메인 특화 근거

- **요지**: Parsons problem(코드 조각을 올바른 순서로 배열)과 그 변형인 micro Parsons + fill-in-the-blank는 코드 도메인에서 검증된 generation 형식이다.
- **2024 연구**: micro Parsons problem(빈칸 포함 변형)을 시험 문항으로 평가했을 때, 학생 다수가 양쪽 형식 모두 학습에 도움이 됐다고 보고. fill-in-the-blank 포함 variant가 perceived difficulty도 높였다 = **active generation을 강제한다는 신호**.
- **자동 생성 가능성**: Java 학습 보조 시스템에서 element fill-in-the-blank 문제를 LLM으로 자동 생성하는 도구가 2025년 발표됨(MDPI Electronics 14권 11호, 2261). diff 입력 → 빈칸 문제 자동 생성이 기술적으로 가능 영역에 들어왔다.
- **code-replay 함의**: 빈칸 문제는 "단순 토큰 마스킹"이 아니라 **개념적으로 중요한 위치(invariant, boundary condition, naming, type contract)에 전략적으로 둬야** generation effect가 살아난다. 이게 단순 "랜덤 토큰 가리기"와의 차이.

## 3. Retrieval practice — "복습 질문"의 근거

- **요지**: 학습 후 일정 간격으로 능동 인출(시험/퀴즈)을 시키면 단순 재학습보다 장기 retention이 훨씬 높다. testing effect로도 알려짐.
- **STEM 메타분석 (2024, IJ STEM Education)**: 9개 STEM 입문 코스에서 spaced retrieval practice를 bi-weekly quiz로 embed. 단일 quiz 집중 vs 여러 quiz 분산 비교에서 **분산 retrieval이 일관되게 우위**.
- **LLM 생성 retrieval practice (2025 RCT)**: 대학 데이터 사이언스 코스에서 LLM이 생성한 객관식 retrieval practice를 받은 그룹이 **89% vs 73%, +16pp** quiz 정확도 우위. AI 자동 생성 retrieval practice가 실제 교육 효과를 낸다는 직접 근거.
- **모범 형식**: 다지선다/빈칸/짧은 답변. AI 지원이 "구조"(retrieval practice 등)와 metacognitive prompt를 추가했을 때 효과가 가장 컸다는 systematic review 결과.
- **code-replay 함의**: "다음 복습 질문" 산출물은 단순 flashcard가 아니라 **diff에서 추출한 개념을 객관식/빈칸 형태로, 일정을 두고 다시 묻는 형식**이어야 한다. 단발성으로 한 번 보여주고 끝나면 효과 거의 없다 → 큐 시스템 필요.

## 4. Spaced repetition — 복습 큐의 스케줄링 근거

- **요지**: 동일한 retrieval을 시간 간격을 두고 반복하면 같은 학습 시간 대비 장기 retention이 극적으로 높다. spacing effect.
- **메커니즘**: 간격이 늘어날수록 인출 난이도가 올라가고, 그 어려운 인출이 장기 기억으로의 deeper processing을 만든다(desirable difficulty).
- **알고리즘 최적화**: PNAS 2019 (Tabibian et al.) — 학습자별 망각 곡선을 모델링해 review 시점을 최적화하면 동일 시간 투입 대비 retention이 유의하게 높음. SuperMemo SM-2, Anki, FSRS 같은 알고리즘의 학문적 뒷받침.
- **STEM 적용 사례**: 1년차 물리 / 학부 생물 / pediatric 의학 교육 등에서 spaced repetition 도입 후 long-term retention 유의 향상이 반복 보고됨. STEM 도메인 일반화 가능성 높음.
- **체감 vs 실제**: 학습자 64-66%가 "기억에 도움됐다"고 자가 보고. 객관 측정과 자가 보고가 같은 방향.
- **code-replay 함의**: 복습 질문은 한 번 만들고 버리지 말고 **개념 단위 큐에 적재 → SM-2/FSRS류 스케줄로 재출제**. "어제 짠 PR의 개념을 3일 / 7일 / 21일 후 다시 묻기"가 정당화된다. Anki integration 또는 자체 스케줄러 둘 중 하나가 MVP scope에 들어와야 retention 주장이 성립한다.

## 5. 코드 읽기 vs 쓰기 — "변경 요약" 산출물의 한계

- **요지**: 단순 코드 읽기는 학습 효과가 약하고, **self-explanation prompt나 Socratic 가이드를 결합할 때 효과가 살아난다.**
- **계층 가설 (Lopez et al.)**: 프로그래밍 학습은 (1) 언어 construct 지식 → (2) 영어로 설명하기 / Parsons / iteration tracing → (3) 코드 작성 의 위계. 중간 계층(설명/tracing/Parsons)을 건너뛴 채 읽기만 하면 작성 능력으로 연결되지 않음.
- **Worked example + self-explanation**: worked example 단독보다 self-explanation prompt를 함께 주면 performance, problem-solving, self-efficacy 모두 유의 향상. **무엇을 읽는가보다 "읽으면서 무엇을 묻는가"가 결정적**.
- **Socratic > free self-explanation**: 자유롭게 설명하라고 두는 것보다, AI가 가이드 질문을 내는 Socratic 방식이 novice에게 더 효과적.
- **code-replay 함의**: "변경 요약" 산출물을 그냥 보여주는 건 읽기에 가까워서 학습 효과가 약하다. **요약을 보여주는 동시에 "왜 이렇게 했나?", "이 줄이 없으면 무엇이 깨지는가?" 같은 Socratic prompt를 함께 내야** 한다. 개념 맵도 단독 노출은 약함 → 맵의 노드를 클릭하면 self-explanation 질문이 뜨는 식의 결합 설계가 필요.

## 6. MVP 6개 산출물 재배치 (근거 기반)

| 산출물 | 학습 과학 근거 | 우선순위 | 설계 메모 |
|---|---|---|---|
| 처음부터 재구현 과제 | Generation effect (d≈0.40), Parsons 계열 | **P0** | diff에서 핵심 부분 마스킹 + step-by-step 재현. AI 솔루션은 hint로만 |
| 빈칸 채우기 | Generation effect, micro Parsons + fill-in-the-blank, 자동 생성 OSS 존재 | **P0** | 전략적 위치(invariant/boundary/contract)에만 빈칸. 랜덤 마스킹 금지 |
| 다음 복습 질문 | Retrieval practice (LLM 생성 +16pp), STEM 메타분석 | **P0** | spaced 큐 필수. 단발 출제는 거의 무효 |
| 개념 목록 추출 | (간접) retrieval/spacing의 입력 단위 역할 | P1 | 그 자체는 학습 산출물 아님. 복습 큐의 키 |
| 변경 요약 | (단독으론 약함) self-explanation prompt 결합 시만 효과 | P1 | 요약 옆에 Socratic 질문 동반 필수 |
| 개념 맵 | (단독으론 약함) generation/retrieval 결합 시만 효과 | P2 | 노드 클릭 → self-explanation. 단독 시각화는 효과 미약 |

## 7. 미해결 질문 / 추가 리서치 후보

- **diff 단위에 generation/retrieval을 적용한 직접 RCT는 아직 본 적 없음.** 일반 STEM/CS 입문 결과를 PR 단위 학습으로 확장하는 외삽이 어디까지 정당한가? → axis 2(AI 학습 영향)와 교차 검증 필요.
- **AI가 생성한 retrieval question의 품질 측정**. +16pp 결과는 인간 검수 없이도 재현되는가? hallucination된 정답 보기가 학습을 망가뜨릴 위험은? → 별도 리서치 필요.
- **개인 repo 단위 long-term retention 측정 방법론**. 복습 효과를 어떻게 metric화할지가 product 차원 미해결.

## Sources

- Slamecka & Graf (1978), generation effect 원전: <https://www.scribd.com/document/267986618/Generation-Effect-Slamecka-Graf-1978-JEP-HLM>
- Generation effect, neural circuit fMRI 근거: <https://pmc.ncbi.nlm.nih.gov/articles/PMC3556209/>
- Generation effect 개관: <https://en.wikipedia.org/wiki/Generation_effect>
- Generation effect 활용 가이드: <https://www.structural-learning.com/post/generation-effect-active-learning>
- Micro Parsons problems as exam questions (2024, arXiv): <https://arxiv.org/html/2405.19460v1>
- Generating distractors for code completion (FLAIRS): <https://journals.flvc.org/FLAIRS/article/download/138995/144092/276184>
- LLM 자동 생성 fill-in-the-blank for Java (2025, MDPI Electronics): <https://www.mdpi.com/2079-9292/14/11/2261>
- LLM-Generated Retrieval Practice in Data Science courses (2025 RCT, +16pp): <https://arxiv.org/pdf/2507.05629>
- Retrieval practice on retention/application of complex concepts (2025, ScienceDirect): <https://www.sciencedirect.com/science/article/pii/S0959475225001434>
- Spaced retrieval practice meta-analysis in 9 STEM courses (2024, IJ STEM Education): <https://link.springer.com/article/10.1186/s40594-024-00468-5>
- Retrieval Practice in Stepwise Worked Examples (2025, ScienceDirect): <https://www.sciencedirect.com/science/article/pii/S0959475225001203>
- Spacing repetitions over long timescales review (PMC): <https://pmc.ncbi.nlm.nih.gov/articles/PMC5476736/>
- Effect of Spaced Repetition on Learning (2024 meta, PubMed): <https://pubmed.ncbi.nlm.nih.gov/39250798/>
- Enhancing human learning via spaced repetition optimization (PNAS, Tabibian et al.): <https://www.pnas.org/doi/10.1073/pnas.1815156116>
- Spaced repetition for STEM (ERIC): <https://files.eric.ed.gov/fulltext/EJ1241511.pdf>
- Spaced repetition pediatrics 적용 (PMC): <https://pmc.ncbi.nlm.nih.gov/articles/PMC12343689/>
- Reading vs tutoring for code comprehension (ACM SAC 2024): <https://dl.acm.org/doi/pdf/10.1145/3605098.3636007>
- Reading/tracing/writing skill 관계 (ICER 2008, Lopez et al.): <https://dl.acm.org/doi/abs/10.1145/1404520.1404531>
- Coding & computational thinking outcomes review (2025, Review of Educational Research): <https://journals.sagepub.com/doi/10.3102/00346543241241327>
