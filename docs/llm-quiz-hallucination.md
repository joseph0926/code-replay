# LLM 생성 학습 카드 hallucination: 검증 파이프라인 설계

리서치 일자: 2026-05-15
스코프: `docs/learning-metrics.md` §8 미해결 — "카드 자동 생성 품질 검증 부재. LLM이 만든 quiz의 정답/distractor가 옳다는 보장이 없으면 retention metric 자체가 오염됨." `docs/learning-science.md` §7의 "AI가 생성한 retrieval question의 품질 측정. +16pp 결과는 인간 검수 없이도 재현되는가?"와 같은 미해결. **MVP 카드 생성 파이프라인이 어떤 검증 단계를 필수로 둘지** 결정한다.

## 한 줄 결론

- **단일 패스 LLM 카드 생성은 사용 금지**. 학계 baseline 에러율 **20-30%** (GPT-3.5 AP Stats 기준 23-29%)이라 single-pass 출력은 retention 측정 자체를 오염.
- **MVP 필수 검증 4단**: (1) **AST 결정적 검증** (code 카드, 100% 정밀도 가능, arxiv 2601.19106) → (2) **QA round-trip** (LLM이 자기 빈칸을 풀 수 있어야 통과) → (3) **다른 모델/온도로 self-verification** → (4) **사용자 1-click "잘못된 카드" 피드백** = human-in-the-loop.
- **Distractor는 overgenerate-and-rank**. 단발 4지 선다 생성 X. 8-12개 후보 → preference 모델 또는 휴리스틱으로 top-3 선택.
- **LLM-as-judge는 bias 12종 인지하고 제한적 사용**. 특히 **재구현 채점에서만 사용**, **카드 자체 생성은 결정적 검증 우선**. 같은 모델 self-judge 금지(self-enhancement 5-7% 편향).
- **목표 메트릭**: 검증 후 카드 에러율 ≤ 10%, 사용자 flag rate ≤ 5% (출고 카드 100개당 5건 이하).

## 1. Baseline 위험 — 검증 없을 때 무엇이 일어나는가

### 1.1 학계 보고 에러율

- **GPT-3.5 AP Statistics 자동 생성** (IntechOpen): basic prompting **23% 에러**, higher-level prompting **29% 에러** (오히려 정교한 prompt가 에러율 증가 — 자가 추론에서 hallucination 누적).
- ChatGPT 자동 생성 question: 의도한 question type과 불일치 사례 다수, **시퀀스 후반으로 갈수록 정확도 저하** (긴 컨텍스트 누적 오류).
- 학계 합의: **단순 fact 기반 quiz도 LLM 단발 출력은 신뢰 부족**.

### 1.2 hallucination 유형 (교육 도메인 특화)

- **Source 일탈** — 입력 코드/PR에 없는 API, 함수, 동작 가정. 가장 위험 (정답이 코드에 없는 것).
- **Type mismatch** — "이 변수의 타입은?" 질문에 잘못된 답.
- **Distractor가 사실은 정답** — distractor가 다른 해석에서 옳을 수 있음.
- **모호한 질문** — 정답이 1개로 일의적이지 않음. 정답률 측정 무의미해짐.
- **Tautology** — 문제 자체에 정답이 들어 있음 (학습 효과 0).

### 1.3 결과

검증 없는 카드 + spaced repetition = **잘못된 정답을 강화 학습**시키는 anti-pattern. retention metric 오염뿐 아니라 **사용자에게 잘못된 코드 지식 주입**. code-replay의 핵심 risk.

## 2. 검증 패턴 카탈로그

| 패턴 | 비용 | 효과 | code-replay 위치 |
|---|---|---|---|
| **AST 결정적 검증** | 매우 낮음 | code 카드에 한해 **100% 정밀도** | code 카드 1차 필수 |
| **QA round-trip** (정답 vs LLM 풀이) | 중간 (1 추가 LLM 호출) | 모호 질문 / tautology 걸러냄 | 모든 카드 2차 필수 |
| **Self-verification (다른 온도/모델)** | 중간-높음 | 일관성 검증 | 3차 권장 |
| **Overgenerate-and-rank** (distractor) | 중간 | distractor 품질 향상 | distractor 생성 시 항상 |
| **Human-in-the-loop (사용자 flag)** | 0 (UX만) | 누락된 에러 회수 | UX 필수 |
| **Question classifier (validator)** | 낮음 (한번 학습) | type 일치 확인 | 후순위 (학습 데이터 부족) |
| **Item-Writing Flaws (IWF) 루브릭** | 인간 검토 | 교육적 품질 | 인간 spot check |

## 3. AST 기반 결정적 검증 (code 카드 한정)

### 3.1 핵심 근거

- **arxiv 2601.19106 "Detecting and Correcting Hallucinations in LLM-Generated Code via Deterministic AST Analysis"**: 200 Python snippet에서 **100% 정밀도 / 87.6% 재현율 / 77% 자동 수정** — code 도메인은 결정적 검증이 가능한 드문 영역.
- 핵심 통찰: 코드는 **자연어와 달리 결정적 syntax**라 AST가 ground truth 역할 가능.

### 3.2 적용 방식

- 빈칸 카드 정답이 코드 토큰/표현일 때, **AST 노드 단위로 정답 여부 판정**.
- 사용자 입력을 같은 컨텍스트에 삽입 → AST 파싱 → 원본 AST와 normalize 비교.
- 변수명 alpha-rename, 공백 무시, 동등 표현 인정.

### 3.3 한계

- **AST 동등하지만 동작 다른 케이스** 있음 (`++i` vs `i++` 같은 부작용).
- **AST 다르지만 동작 같은 케이스** 더 많음 (`for` vs `forEach`).
- 따라서 **AST 동등 = 정답 확정 / AST 다름 = LLM judge fallback**의 2단 구조.

## 4. QA round-trip 검증

가장 비용 대비 효과 높은 패턴. **LLM이 자기 빈칸을 풀 수 있어야 통과**.

```
1. 카드 생성기: 코드 + 빈칸 위치 → 정답 = X
2. 검증기: 같은 코드 + 빈칸을 다른 LLM 호출로 풀이 → 답 = Y
3. X == Y (AST normalize)인가?
   - 같으면 통과
   - 다르면 카드 폐기 또는 재생성
4. 반복적으로 X != Y면 카드를 모호 질문으로 분류
```

### 4.1 효과

- **모호 질문 / tautology / source 일탈** 대부분 자동 검출.
- 검증기 모델이 생성기와 다르면 self-enhancement bias 회피.
- 비용: 카드 1개당 LLM 호출 2회 (생성 + 검증).

### 4.2 변형

- **k-회 sampling round-trip**: 검증기를 k=3-5번 호출, 다수결로 통과 판정. tautology 강하게 잡힘.
- **AST round-trip**: 코드 도메인이라 답이 코드 표현일 때 AST 동등 판정.

## 5. Distractor 품질 — Overgenerate-and-Rank

### 5.1 단발 생성의 문제

- LLM이 한 번에 4지 선다를 만들면 distractor가 **너무 명백히 틀림** (학습 효과 없음) 또는 **사실 정답** (정답률 무너짐).
- Self-BLEU 같은 lexical diversity는 **품질을 보장 못 함**.

### 5.2 권장 패턴

- **Overgenerate**: 8-12개 distractor 후보 생성.
- **Rank**: 다음 기준으로 점수화 → top-3 선택.
  - 정답과의 lexical/semantic 거리 (너무 가깝거나 너무 멀면 감점)
  - Item-Writing Flaws 휴리스틱 (negative wording 회피, all-of-the-above 회피)
  - **direct preference optimization (DPO)** 기반 ranker — 실제 학습자 선택 likelihood 모델 (학술 prior art 다수)
  - Code 도메인 한정: AST 거리 (너무 비슷하면 trivial)

### 5.3 MVP 단순화

- DPO ranker 없이도 **(a) 정답 AST 거리 1-3 토큰, (b) 정답 type 일치, (c) 동일 코드 컨텍스트에 컴파일/parse 가능** 휴리스틱만으로 baseline 가능.
- 사용자 피드백 누적 후 ranker 학습 도입.

## 6. LLM-as-Judge — 제한적 사용

### 6.1 알려진 bias (CALM framework, 12종)

| Bias | 영향 | 완화 |
|---|---|---|
| **Position bias** | GPT-4 40% 일관성 손실 (선택지 순서 바꾸면 답 바뀜) | 답 순서 random shuffle 후 재호출 |
| **Verbosity bias** | 긴 답을 선호 (~15% 인플레이션) | 응답 길이 정규화 또는 제한 |
| **Self-enhancement** | 같은 모델이 만든 답을 선호 (5-7%) | **judge 모델 ≠ generator 모델** |
| **Authority bias** | 권위 표현(논문 인용 등)에 가산 | 메타데이터 제거 |
| **Domain gap** | 전문 도메인에서 10-15% agreement 하락 | 인간 검수 hybrid |
| **Judge drift** | API 버전 업데이트로 결과 변동 | 모델 version pin |
| **Format sensitivity** | 단순 formatting 변경으로 일관성 깨짐 | input format 통일 |

### 6.2 그럼에도 가치

- 인간 평가 대비 **500-5000x 비용 절감**.
- 인간-인간 일관성과 동등한 **80% human agreement**.
- "비평가들 주장보다는 강하고, 사용자 가정보다는 약함" — 무비판적 사용 X.

### 6.3 code-replay에서의 위치

- **카드 자체 정답성 판정엔 안 씀** (AST + QA round-trip이 우선).
- **재구현 채점**의 (c) LLM judge fallback에서만 사용 (`docs/learning-metrics.md` §4.1c).
- generator와 judge 모델 분리 (예: generator = Qwen2.5-Coder, judge = Claude Sonnet 4 또는 다른 model).
- 채점 결과는 **사용자에게 confidence 표시** ("AI 채점 — 본인 검토 권장"), self-grade override 허용.

## 7. code-replay 카드 생성 파이프라인 (구체)

```
INPUT: PR diff + 학습 단위 (개념 노드)

1. Generation (Q5_K_M Qwen2.5-Coder)
   - 빈칸 위치 후보 생성 (AST 노드 단위)
   - 각 후보에 카드 (질문, 정답, distractor 8-12개) 생성

2. AST 결정적 검증 (code 카드)
   - 정답 토큰을 다시 코드에 삽입 → AST 파싱 → 원본 AST와 동등?
   - 불일치 시 카드 폐기

3. QA round-trip (모든 카드)
   - 다른 LLM 호출 또는 다른 온도/seed로 빈칸 풀이
   - k=3 sampling, 다수결 답이 정답과 AST 동등?
   - 불일치 시 카드 폐기 (모호 / tautology)

4. Distractor overgenerate-and-rank
   - 8-12 후보 → 휴리스틱 ranker로 top-3 선택
   - top-3 모두 AST 컴파일 통과 + 정답 type 일치 확인

5. 사용자 노출
   - 카드에 "AI 생성" 배지 + 1-click "잘못된 카드" 버튼
   - flag 시 즉시 큐에서 제외 + 분석용 로그

OUTPUT: .codereplay/cards/<id>.md (FSRS 호환 frontmatter)
```

### 7.1 비용

- 카드 1개당 LLM 호출: 생성 1 + 검증 round-trip 3-5 + distractor 8-12 = **약 12-18 호출**.
- Q5_K_M 로컬 호출 1회 ~1-3초 (M4 Mac, 32B 모델 기준 추정) → **카드 1개 ~30-60초**.
- PR 1개에서 카드 5-15개 생성 → **PR 처리 시간 5-15분**. 백그라운드 처리 적합 (즉시 결과 X).

### 7.2 cost-quality 단계

- **Fast mode** (default): 검증 round-trip k=1, distractor 5 후보. 빠르지만 에러율 ~15-20%.
- **Standard mode**: round-trip k=3, distractor 8-12. 에러율 ~10%.
- **Strict mode**: round-trip k=5 + cloud judge cross-check. 에러율 ~5% 목표.

## 8. Human-in-the-Loop (사용자 피드백 루프)

### 8.1 UX

- 카드 옆 1-click 버튼:
  - **잘못된 카드** (정답 틀림 / 모호 / tautology) → 즉시 제거
  - **개선 가능** (정답은 맞지만 distractor 약함) → 큐 유지, 분석 로그
  - **좋은 카드** (옵션) → 선호 모델 학습용 데이터

### 8.2 학습 신호로 활용

- flag 누적 → 어떤 모델 / 어떤 prompt / 어떤 PR 패턴에서 에러 잦은지 추적.
- 사용자별 ranker 미세조정 (DPO) — 충분한 flag 누적 후 가능.
- **사용자 flag 데이터는 로컬에만**. cloud로 보내지 않음 (privacy 일관성).

### 8.3 목표 metric

- 출고 카드 100개당 사용자 flag ≤ 5건 (5%).
- flag된 카드는 같은 PR에서 재생성 시 **다른 패턴**으로 시도 (단순 retry 금지).

## 9. 위험 / 미해결

- **검증 비용 / 사용자 인내심 trade-off**: PR마다 5-15분 백그라운드 = UX 부담. fast/standard/strict mode 사용자 선택권 필수.
- **AST 결정적 검증 언어 커버리지**: arxiv 2601.19106은 Python 200개. **TypeScript/JS/Go/Rust 등에서 같은 정밀도 재현 미검증**. → MVP에서 자체 측정 필요.
- **QA round-trip의 generator-judge 같은 모델 의존**: 자원 제약 시 같은 모델로 round-trip하면 self-enhancement bias로 모호 질문 못 잡을 가능. → 검증기는 다른 모델 강력 권장 (예: Qwen + DeepSeek-Coder hybrid).
- **검증을 통과한 카드도 사용자 도메인에서 부적절 가능**: 사용자 코드베이스 convention과 anti-pattern인 정답을 LLM이 줄 수 있음. 사용자 flag 의존도 높음.
- **fast mode 데이터 오염**: 검증 약한 카드가 spaced 큐에 들어가면 retention metric 신뢰 떨어짐. fast mode는 "저장 X, drill만" 모드로 분리 검토.
- **DPO ranker 학습용 사용자 데이터 보호**: 향후 ranker 학습 시 사용자 flag 데이터를 어떻게 다룰지 (로컬 학습 vs federated). MVP scope에서는 로컬 휴리스틱 ranker만.
- **judge model API 약관 변동**: cloud judge 사용 시 약관 변경에 따른 재평가 필요 (`docs/local-vs-api-llm.md` §6 참조).

## Sources

### LLM 생성 quiz 품질 / hallucination
- LLM Hallucination Comprehensive Survey (arxiv 2510.06265): <https://arxiv.org/html/2510.06265v2>
- LLMs for educational question classification/generation (ScienceDirect): <https://www.sciencedirect.com/science/article/pii/S2666920X24001012>
- Hallucination detection via semantic entropy (Nature 2024): <https://www.nature.com/articles/s41586-024-07421-0>
- Survey on Hallucination in LLMs (ACM TOIS): <https://dl.acm.org/doi/10.1145/3703155>
- Awesome hallucination detection (EdinburghNLP): <https://github.com/EdinburghNLP/awesome-hallucination-detection>
- Why Language Models Hallucinate (OpenAI): <https://openai.com/index/why-language-models-hallucinate/>
- LLM hallucinations in CI (CircleCI): <https://circleci.com/blog/llm-hallucinations-ci/>
- LLM evaluation hallucinations & faithfulness (F22 Labs): <https://www.f22labs.com/blogs/how-to-evaluate-llm-hallucinations-and-faithfulness/>
- LLM for Statistics Education Q&A (IntechOpen): <https://www.intechopen.com/chapters/1230865>
- Hallucination AI Wikipedia: <https://en.wikipedia.org/wiki/Hallucination_(artificial_intelligence)>

### Distractor generation
- Distractor Generation in MCQ Survey (arxiv 2402.01512): <https://arxiv.org/html/2402.01512v2>
- Automatic distractor generation MCQ (PMC): <https://pmc.ncbi.nlm.nih.gov/articles/PMC11623049/>
- Math MCQ distractor with LLMs (arxiv 2404.02124): <https://arxiv.org/html/2404.02124v1>
- Improving distractor with overgenerate-and-rank (arxiv 2405.05144): <https://arxiv.org/html/2405.05144>
- Enhancing distractor with RAG + KG (arxiv 2406.13578): <https://arxiv.org/html/2406.13578v1>
- Vocabulary distractor generation (Springer TELRP): <https://link.springer.com/article/10.1186/s41039-018-0082-z>
- Vision Language Model MCQ (arxiv 2501.03225): <https://arxiv.org/html/2501.03225v1>
- Distractor systematic literature review (ResearchGate): <https://www.researchgate.net/publication/385809302_Automatic_distractor_generation_in_multiple-choice_questions_a_systematic_literature_review>

### LLM-as-Judge bias / reliability
- Survey on LLM-as-a-Judge (arxiv 2411.15594): <https://arxiv.org/abs/2411.15594>
- Survey on LLM-as-a-Judge ScienceDirect: <https://www.sciencedirect.com/science/article/pii/S2666675825004564>
- Justice or Prejudice — CALM bias framework: <https://llm-judge-bias.github.io/>
- LLM as Judge 2026 Guide (Label Your Data): <https://labelyourdata.com/articles/llm-as-a-judge>
- Empirical Study of LLM-as-Judge (arxiv 2506.13639): <https://arxiv.org/html/2506.13639v1>
- Judge Reliability Harness (arxiv 2603.05399): <https://arxiv.org/html/2603.05399v1>
- Bias and Uncertainty in LLM Judge (arxiv 2605.06939): <https://arxiv.org/html/2605.06939>
- LLM-As-A-Judge Reliability Bias (Adaline): <https://www.adaline.ai/blog/llm-as-a-judge-reliability-bias>

### AST 결정적 검증 / Self-verification
- Detecting Hallucinations via Deterministic AST Analysis (arxiv 2601.19106): <https://arxiv.org/html/2601.19106v1>
- Self-Verification Prompting (LearnPrompting): <https://learnprompting.org/docs/advanced/self_criticism/self_verification>
- Chain of Targeted Verification Questions (arxiv 2405.13932): <https://arxiv.org/pdf/2405.13932>
- Incentivizing LLMs to Self-Verify (arxiv 2506.01369): <https://arxiv.org/html/2506.01369v1>

### Automatic short answer grading
- LLM ASAG medical education (Springer BMC): <https://link.springer.com/article/10.1186/s12909-024-06026-5>
- I understand why I got this grade — ASAG with feedback (arxiv 2407.12818): <https://arxiv.org/html/2407.12818v1>
- LLM-based Autograding for Short Textual Answers (arxiv 2309.11508): <https://arxiv.org/pdf/2309.11508>
- LLMs for Automated Scoring in Formative Assessment (MDPI): <https://www.mdpi.com/2076-3417/15/5/2787>
