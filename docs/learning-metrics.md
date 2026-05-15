# 학습 효과 metric 방법론: code-replay에서 무엇을, 어떻게 측정할 것인가

리서치 일자: 2026-05-15
스코프: code-replay의 학습 효과를 product 안에서 어떻게 정량 측정할지 결정. 사용자에게 보일 metric과 내부 분석용 metric을 분리한다. 4계층 metric 구조는 장기 설계이며, 각 계층의 P0/P1/P2 위치는 `docs/product-direction.md` non-goals와 `docs/mvp-spec.md` non-goals를 기준으로 본다.

## 현재 결정 기준

- P0는 verified review card의 spaced review를 중심으로 한 단순 카드 계층 metric에 한정한다.
- reconstruction 자동 채점, transfer 자동 감지, longitudinal dashboard는 P0의 비-목표이며 P1/P2에서 검토한다.
- concept graph 자체는 P0에서 만들지 않는다. P0의 `concept`은 단순 label/id이며, 그 위에 mastery 합산을 얹는다.
- 사용자 노출은 P0에서 최소 형태(예: due card 수, 검증된 카드 수)로 시작하고, dashboard/시각화는 후속 단계의 선택지로 둔다.

## 한 줄 결론

- **장기 4계층 metric 구조**: card retention (SRS 정확도) → concept mastery (개념 단위 합산) → reconstruction accuracy (재구현 결과 정합도) → transfer (사용자가 같은 개념을 다른 변경에서 다시 자력 처리했는가). P0는 card 계층, P1은 concept mastery + reconstruction 채점, P2는 transfer 감지와 longitudinal trend다.
- **카드 계층은 학계 표준 그대로 사용**: FSRS의 Log Loss / RMSE / RMSE(bins). 9,999 collection / 3.5억 review 벤치마크로 검증된 metric.
- **재구현 계층(P1)은 AST 동등성 + 테스트 통과율 + LLM-as-judge fallback** 3단 합산.
- **Transfer 계층(P2)이 product 차원에서 가장 강력한 신호**지만 자동 감지는 P0 비-목표다.
- **속도(타이핑 시간 등)는 측정하지 않음**: `docs/ai-learning-impact.md` §1의 METR 시그널 진동에서 본 것처럼 속도 narrative와 거리 둠.
- **사용자 노출은 P0에서 최소화**한다. dashboard 형태는 P2 후보이며 P0에서는 검증된 카드 수, due card 수처럼 단순 CLI 요약에 한정한다.

## 1. 측정 4계층

| 계층 | 단위 | 무엇을 측정 | 측정 시점 | 신뢰도 | P0/P1/P2 |
|---|---|---|---|---|---|
| Card | 단일 verified review card | 다음 review 시 정답 확률 vs 실제 결과 | 매 review | 매우 높음 (FSRS 표준) | P0 |
| Concept | 개념 label/id (e.g., "useMemo의 stale closure 회피") | 해당 label의 모든 verified card 평균 mastery | 카드 review 시 자동 갱신 | 높음 (카드 합산) | P1 |
| Reconstruction | replay task 1회 | AST 동등성 + 테스트 통과율 + LLM judge | 사용자가 replay 제출 시 | 중 (AST 동등 ≠ 의미 동등 케이스 존재) | P1 |
| Transfer | 다른 target에서 같은 concept 등장 | 사용자가 hint/AI 의존 없이 처리했는가 | 후속 target 분석 시 자동 감지 | 가장 높음 (real-world 신호) | P2 |

## 2. Card 계층 — FSRS 표준 그대로

### 2.1 metric

- **Log Loss** — FSRS 옵티마이저 내부 사용. binary classification (정답/오답)에서 예측 확률과 실제 결과의 거리.
- **RMSE** — 예측 recall 확률과 실제 recall의 평균 제곱근 오차.
- **RMSE(bins)** — review를 bin으로 묶고 bin 내 평균 예측 vs 평균 실제 차이. 0-1 범위, 낮을수록 좋음. **사람이 가장 직관적으로 해석 가능한 metric**.

### 2.2 벤치마크 기준

- **9,999 Anki collection / 349,923,850 review** 데이터셋이 공개돼 있음 (open-spaced-repetition/srs-benchmark).
- FSRS default parameter가 99.5% 사용자에서 SM-2보다 정확. **자체 SRS 알고리즘 만들 필요 X**.
- code-replay에서는 FSRS 파라미터 그대로 쓰고, **로그를 같은 형식으로 저장**해서 사용자가 자기 데이터를 다른 SRS 도구로 export할 수 있게.

### 2.3 적용

- 모든 verified review card를 FSRS 카드로 모델링한다. `verified: false` card는 due 계산에 포함하지 않는다.
- review 결과를 1(정답)/0(오답) + 응답 시간으로 기록한다 (`.codereplay/reviews/review-log.jsonl`).
- FSRS가 다음 review 일자 + 예측 retention 출력.
- **이 카드 계층 metric은 product 내부 운영용** (스케줄러 자체 평가). 사용자에게 직접 노출은 §7에서.

## 3. Concept 계층 — 카드 합산 (P1)

### 3.1 정의

- 한 target에서 추출된 concept label K개에 대해, 각 label은 1개 이상의 verified card와 연결.
- label mastery = 그 label에 묶인 verified card들의 **predicted retention 가중 평균** (카드 수가 많은 label에 가중치).

### 3.2 의미

- label 단위로 "이 개념은 안정 상태인가" 판정.
- mastery > 0.9 = 안정 / 0.7-0.9 = 유지 review 필요 / < 0.7 = 다시 학습 필요.
- 임계값은 사용자별/도메인별 보정 필요(초기엔 default).

### 3.3 활용

- 사용자 노출(P2 후보): "개념 N개 학습 중, M개 안정". dashboard 형태는 P0의 비-목표.
- 새 target 처리 시 "이 변경에는 안정 안 된 concept X개가 들어 있음 → 우선순위 review" 알림(P1 이후).

## 4. Reconstruction 계층 — 재구현 정합도 (P1)

`replay task` 결과의 자동 채점은 **P0 비-목표**다. P0에서 reconstruction은 사용자가 직접 비교·자가 채점하는 형태로 두고, 자동 채점은 P1에서 도입한다.

### 4.1 측정 방식 (3단 hybrid)

- **(a) AST 동등성** (`docs/ast-diff-tools.md` 활용)
  - 사용자 솔루션 AST vs 원본 AST → 노드 normalize 후 동등 비율.
  - 변수명/공백 무시, 구조만 비교.
  - **장점**: 자동, 빠름. **단점**: 동작 같지만 구조 다른 케이스에 false negative.
- **(b) 테스트 통과율** (테스트 있는 PR 한정)
  - PR 안에 테스트가 있으면(또는 LLM이 spec에서 생성), 사용자 솔루션을 테스트 돌려 pass rate.
  - **장점**: 의미 동등성 직접 측정. **단점**: 테스트 부재 PR 다수.
- **(c) LLM-as-judge** (G-Eval 패턴)
  - 원본 + 사용자 솔루션 + 채점 rubric을 LLM에 제시. JSON score 출력.
  - **장점**: 의미적 평가 가능. **단점**: hallucination, 일관성. **인간 검증과 함께 써야 신뢰**.

### 4.2 합산 점수

```
reconstruction_score = w_a * AST_equivalence + w_b * test_pass_rate + w_c * llm_judge_score
```

- 가중치 default: a=0.3, b=0.5(테스트 있을 때), c=0.2.
- 테스트 없으면: a=0.5, c=0.5.
- 사용자가 자기 채점 가능 (override) — false negative 보정 루프.

### 4.3 외부 reference

- **GitLab Copilot eval 패턴**: 일부러 실패 도입 → 모델이 통과 코드 생성하는지. **테스트 통과 = ground truth** 모델.
- **G-Eval (NLG eval with GPT-4)**: 메트릭 design 패턴, task-specific rubric LLM 평가.
- **HumanEval/MBPP** 한계: 자동 채점만 → context-sensitive 오류 놓침. **전문 환경에선 인간 검토 병행 필요** → 사용자 자기 검증 루프 정당화.

## 5. Transfer 계층 — 가장 결정적인 신호 (P2)

### 5.1 정의

- 한 target에서 학습한 concept X가, **시간이 지나 다른 target에서 다시 등장**했을 때, 사용자가 외부 도움 없이 처리했는가.
- 자동 감지는 P0 비-목표이며 P2에서 검토한다.
- "외부 도움 없음"의 신호:
  - 사용자가 새 target을 만들지 않고 직접 처리한 흐름
  - 같은 concept 카드의 review 점수 안정
  - 코드 리뷰에서 같은 종류 코멘트 받지 않음

### 5.2 분류 — 학계 정의 매핑

- **Near transfer**: 같은 라이브러리/같은 패턴, 다른 파일/feature. (예: `useMemo` 한 컴포넌트에서 학습 → 다른 컴포넌트에서 같은 패턴 적용)
- **Far transfer**: 같은 개념의 추상화, 다른 domain. (예: React `useMemo` memoization 학습 → 백엔드 caching 결정에 동일 mental model 적용) — 자동 측정 어려움. 사용자 자기보고에 의존.

### 5.3 자동 감지 알고리즘 (P2 초안)

1. 새 target 처리 → 등장 concept label 추출.
2. 기존 학습 큐에 같은 label 있는지 매칭.
3. 매칭 시:
   - 사용자가 그 변경을 새 target으로 등록하지 않고 통과 + 같은 concept 카드 mastery > 0.7 → **transfer success**.
   - 같은 변경이 다시 target으로 등록되거나 mastery 낮음 → 미전이 → 학습 큐 우선순위 상향.

전제: Code Replay는 commit 자체의 agent-assisted 여부를 추론하지 않는다. 위 "외부 도움 없음" 신호는 사용자나 외부 연동이 명시적으로 표시한 provenance에 의존한다.

### 5.4 의미

- product 차원에서 **가장 진짜에 가까운 학습 신호**. 카드 review는 인공 setting, 재구현은 단발 성능, 그러나 transfer는 일상 작업에서 자동 측정.
- **단점**: 같은 개념이 시간 안에 다시 안 나오면 측정 불가. coverage 낮을 수 있음.
- 따라서 카드/재구현은 1차 측정, transfer는 2차 보강.

## 6. Longitudinal — 본인 cohort

### 6.1 학계 사례

- **debugging skill 8주 longitudinal** (arxiv 2509.22420, 2025): 4가지 instruction 비교, 8주 retention.
- **소프트웨어 tutor 71% retention** (2020): 첫 세션 후 시간 경과 retention 측정.
- **TDD 5개월 cohort** (ACM ESEM 2018): 30명, TDD 실천 5개월 retention.
- **CS1 12년 longitudinal** (ICER 2020): 코호트 데이터 12년.
- 일반 design 원칙: **skill training은 4-8주, long-term은 6-12개월**.

### 6.2 code-replay 적용

- N=1 cohort (사용자 본인) → 사용자 자신의 historical baseline과 비교.
- 분기별 trend (지난 90일 mastery curve, transfer success rate, reconstruction score 평균).
- **인구 cohort 비교는 product 단계에서 안 함** (privacy / 사용자 동의 문제).

## 7. 사용자에게 보일 metric vs 내부

### 7.1 P0 사용자 노출

- CLI 기반 단순 요약: 검증된 카드 수, due 카드 수, 최근 review 결과 정도.
- dashboard 형태나 longitudinal trend 시각화는 P0의 비-목표(`docs/product-direction.md` non-goals). P2 후보.

### 7.2 P1+ 사용자 노출

- **"이번 주 학습 안정 concept / 전체 학습 중 concept"** — 73 / 120 같은 단순 분수.
- (선택) **"7일 transfer 성공률"** — "지난 7일 새 target의 학습 concept 중 X%를 도움 없이 처리". 자동 감지가 갖춰진 P2 이후.
- 그 이상은 정보 과부하. **행동 변화 만들 수 있는 단순 score 1개**가 우선.

### 7.3 내부 / power user

- 카드 RMSE, concept mastery 분포, reconstruction 점수 trend, transfer 매트릭스 등을 별도 detail view로 분리.
- power user 옵션이지 default 아님.

### 7.4 안 보이는 것

- 속도 (타이핑/완료 시간) — `docs/ai-learning-impact.md` §1 narrative 일관성.
- 다른 사용자와 비교 — privacy + 동기 측면 부정적.

## 8. 위험 / 미해결

- **카드 자동 생성 품질 검증 부재**: LLM이 만든 quiz의 정답/distractor가 옳다는 보장이 없으면 retention metric 자체가 오염됨. → 사용자 "이 카드 잘못됨" 피드백 루프 필수.
- **AST 동등성의 false negative**: 동작은 같지만 구조 다른 솔루션을 틀렸다고 채점. → LLM judge fallback + 사용자 override.
- **Transfer 측정의 noise**: 사용자가 일부러 AI 안 쓴 게 아니라 그냥 그 개념이 안 나온 경우 vs 진짜 transfer. → 충분한 sample size 모이기 전엔 신뢰 낮게 표시.
- **Cold start**: 첫 1-2주는 데이터 부족으로 어떤 metric도 무의미. → 명시적 "데이터 모으는 중" 상태 표시.
- **사용자 자기 평가 편향**: reconstruction 자기 채점은 self-serving bias. → AST/test 자동 점수와 self-report를 분리해 둠.
- **개념 노드 동일성 판정**: 같은 개념인지 판정이 틀리면 transfer 측정 망가짐. → embedding similarity + 인간 라벨 hybrid.
- **Privacy**: 모든 metric이 로컬에 머물러야 함. cloud에 retention 데이터 보내지 않음 (`docs/local-vs-api-llm.md` §5.4).

## 9. 단계별 적용

| 시점 | 만든다 | 만들지 않는다 |
|---|---|---|
| P0 | verified card 계층(FSRS) + due 카드 수/검증된 카드 수 CLI 요약 | concept mastery 합산, reconstruction 자동 채점, transfer 자동 감지, dashboard |
| P1 | concept mastery 합산, reconstruction 채점(AST 동등성 + LLM judge), reconstruction trend | transfer 자동 감지, longitudinal dashboard |
| P2 | transfer 자동 감지, longitudinal trend dashboard | 인구 cohort, 외부 leaderboard |

## Sources

### Longitudinal CS/programming retention
- Debugging skill longitudinal (arxiv 2509.22420): <https://arxiv.org/pdf/2509.22420>
- Long-term retention with software tutor (Springer): <https://link.springer.com/chapter/10.1007/978-3-030-49663-0_46>
- CS1 longitudinal 12-year (ICER 2020): <https://dl.acm.org/doi/10.1145/3372782.3406274>
- TDD retention 5-month cohort (ACM ESEM 2018): <https://dl.acm.org/doi/10.1145/3239235.3240502>
- Long-Term Retention of Knowledge and Skills (DTIC): <https://apps.dtic.mil/sti/pdfs/ADA349869.pdf>
- Longitudinal study Wikipedia: <https://en.wikipedia.org/wiki/Longitudinal_study>
- Longitudinal studies overview (PMC): <https://pmc.ncbi.nlm.nih.gov/articles/PMC4669300/>

### SRS / FSRS metrics
- SRS Benchmark (open-spaced-repetition): <https://github.com/open-spaced-repetition/srs-benchmark>
- FSRS Optimal Retention wiki: <https://github.com/open-spaced-repetition/fsrs4anki/wiki/The-optimal-retention>
- Understanding retention in FSRS (Expertium): <https://expertium.github.io/Retention.html>
- FSRS Benchmark blog (Expertium): <https://expertium.github.io/Benchmark.html>
- ankitects fsrs-benchmark: <https://github.com/ankitects/fsrs-benchmark>
- FSRS vs SM-2: <https://www.mindomax.com/fsrs-vs-sm2-spaced-repetition-algorithm>
- FSRS Anki tutorial: <https://github.com/open-spaced-repetition/fsrs4anki/blob/main/docs/tutorial.md>
- Anki FSRS Explained: <https://anki-decks.com/blog/post/anki-fsrs-explained/>

### Transfer of learning
- Near vs Far Transfer (Carpentries): <https://carpentries.org/blog/2017/02/near-vs-far-transfer/>
- Transfer is Hard (Learning Scientists): <https://www.learningscientists.org/blog/2016/6/2-1>
- Effects of Near vs Far Transfer (ERIC): <https://files.eric.ed.gov/fulltext/ED504847.pdf>
- Transfer Near and Far (Learning Sci Comm): <https://www.learningscicomm.com/post/transfer-near-and-far-an-important-idea-for-all-teachers-to-understand>
- Testing Effect and Far Transfer (Frontiers): <https://www.frontiersin.org/journals/psychology/articles/10.3389/fpsyg.2016.01977/full>
- Transfer of Learning overview (ScienceDirect): <https://www.sciencedirect.com/topics/psychology/transfer-of-learning>
- Learning Transfer 2026 Guide (Thirst): <https://thirst.io/blog/what-is-learning-transfer-the-complete-2025-guide/>

### Code/LLM evaluation
- Large Language Model Evaluation 2026 (AIMultiple): <https://research.aimultiple.com/large-language-model-evaluation/>
- LLM evaluation metrics (Evidently AI): <https://www.evidentlyai.com/llm-guide/llm-evaluation-metrics>
- LLM Evaluation Metrics (Confident AI): <https://www.confident-ai.com/blog/llm-evaluation-metrics-everything-you-need-for-llm-evaluation>
- DeepEval framework: <https://github.com/confident-ai/deepeval>
- LLM evaluation guide (Braintrust): <https://www.braintrust.dev/articles/llm-evaluation-metrics-guide>
- Evaluating LLM Systems (Confident AI): <https://www.confident-ai.com/blog/evaluating-llm-systems-metrics-benchmarks-and-best-practices>
- Evaluating LLM Metrics through Real-World Capabilities (arxiv 2505.08253): <https://arxiv.org/html/2505.08253v1>
