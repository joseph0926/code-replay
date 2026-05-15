# concept identity: 같은 개념을 묶는 entity resolution 설계 (P1/P2)

리서치 일자: 2026-05-15
스코프: 여러 target에서 등장하는 개념을 같은 `concept`로 묶는 entity resolution 설계. **`docs/product-direction.md`에서 P0의 `concept`은 복잡한 graph가 아니라 단순 label/id**이므로, 이 문서의 자동 판정/embedding/active learning 설계는 P1 이후의 후속 단계로 둔다.

## 현재 결정 기준

- P0의 `concept`은 단순 label/id다. 안정된 이름을 부여하는 것까지만 P0 책임.
- 자동 entity resolution(embedding, threshold, active learning, alias graph)은 P1+ 후속 설계. transfer 자동 감지(P2)의 전제이기 때문에 transfer가 활성화되기 전까지는 부분 구현으로도 가능.
- full code knowledge graph는 P0의 비-목표(`docs/product-direction.md` non-goals). 아래 설계는 그 위에서 보존한다.

## 한 줄 결론

- **3-tier threshold 자동 판정**: cosine **≥ 0.92 auto-merge** / **0.75-0.92 사용자 확인 (active learning)** / **< 0.75 새 label**. Neo4j Agent Memory가 검증한 컨벤션 그대로 차용. **P1 이후 도입.**
- **Embedding은 hybrid**: 개념 이름(자연어) + 대표 코드 스니펫 둘 다 embed → 가중 합산. **1차 후보는 Jina Embeddings v2 또는 nomic-embed-text** (Ollama 호환).
- **Borderline (0.75-0.92)에서 LLM judge 호출은 선택지지만 default 안 함**. `docs/llm-quiz-hallucination.md` §6 bias 12종이 작은 샘플에서 결정적. **사용자 확인이 더 신뢰**.
- **Active learning 루프**: 사용자 merge/split 결정이 label data가 돼 사용자별 threshold 미세 조정. 학계 entity resolution 표준 패턴 (Saha et al. ACM CIKM 2019).
- **temporal drift 처리**: 같은 개념이 라이브러리 버전 업그레이드로 다르게 표현될 수 있음 (예: React `useMemo` vs `use(memo(...))` cache primitive). **alias 그래프**로 별칭 묶음.

## 1. 문제 정의

### 1.1 왜 어려운가

같은 패턴이 target마다 다르게 표현됨:
- target-1에서 추출: `"useMemo로 비싼 계산 캐싱"`
- target-3에서 추출: `"React useMemo 의존성 배열 활용"`
- target-7에서 추출: `"렌더 비용 줄이는 memoization 패턴"`

이 셋이 같은 개념인지 다른 개념인지 결정해야 한다. 아니면:
- **너무 합치면**: 다른 개념이 한 label로 뭉쳐 mastery 신호가 흐려짐.
- **너무 쪼개면**: transfer 측정이 0에 수렴 (같은 개념이 다른 target에서 등장해도 매칭 못 함).

### 1.2 transfer 측정의 전제

`docs/learning-metrics.md` §5 transfer 계층(P2)은 "같은 개념이 새 target에서 다시 등장했을 때 사용자가 자력 처리했는가"를 측정한다. **label 동일성 판정이 틀리면 transfer 신호 자체가 무의미해진다.** transfer 자체가 P2 후속 단계이므로 이 문서의 자동 entity resolution도 동일한 시점에 활성화한다.

### 1.3 학계 용어

- 일반: **entity resolution / record linkage / deduplication**
- KG 도메인: **ontology matching / entity matching**
- code-replay 특화: **concept identity / concept node merging**

## 2. Pipeline 개요 (5단, P1+)

```
1. Extract  : target + AST → 개념 후보 (이름 + 대표 코드 + 설명)
2. Embed    : (이름 NL embed) + (코드 embed) hybrid 벡터
3. Block    : k-NN search로 기존 label top-k 후보 추출 (n² 회피)
4. Match    : threshold 기반 분류
              ≥ 0.92  → auto-merge
              0.75-0.92 → user-flag (active learning)
              < 0.75  → new label
5. Merge    : 같은 label로 판정되면 alias 추가, card 누적, 통계 업데이트
```

## 3. Embedding 모델 선택

### 3.1 후보

| 모델 | 차원 | 도메인 | Ollama | code-replay 적합 |
|---|---|---|---|---|
| **Jina Embeddings v2** | 768 | 일반 + 큰 코드베이스 최적화 | △ (HF) | **1차 후보** |
| **nomic-embed-text** | 768 | 범용 | ✅ | **MVP 1순위 (로컬 stack 일관)** |
| **CodeBERT (Microsoft)** | 768 | 코드 (클래식) | △ | 보조 |
| **GraphCodeBERT** | 768 | 코드 + 데이터 흐름 | △ | 보조 (구조 인식 강) |
| **CodeCSE (arxiv 2407.06360)** | — | 코드+주석 다국어 | △ | 보조 |
| **bge-large-en** | 1024 | 일반 SOTA | ✅ | 자연어 비교 강 |
| **LoRA-MME ensemble** | — | 코드 search F1 0.79 | ❌ | 후순위 (복잡) |

### 3.2 권장

- **활성화 1차 (P1+)**: `nomic-embed-text` (Ollama 호환, 로컬 stack 일관). 자연어 label 이름 비교에 충분.
- **2차 (정확도 필요 시)**: 자연어 부분은 nomic, **코드 스니펫 부분은 CodeBERT/GraphCodeBERT** 별도 embed. 두 벡터 결합.

### 3.3 Hybrid embedding 전략

```python
v_concept = α * embed_nl(concept.name + " " + concept.description)
          + β * embed_code(concept.representative_snippet)
# α=0.7, β=0.3 default (개념 이름 가중치 높게)
```

- α/β는 사용자 데이터 누적 후 active learning으로 조정.
- 정규화 후 cosine 비교.

## 4. Threshold 설계 — 3-tier

### 4.1 컨벤션 (Neo4j Agent Memory 기준)

- **auto_merge_threshold = 0.92** — 신뢰도 매우 높음. 자동 merge.
- **flag_threshold = 0.75** — 모호. 사용자 확인 큐로 보냄.
- **0.75 미만** — 새 노드 생성.

### 4.2 왜 이 값들인가

- 0.92는 거의 동일 텍스트/의미. cosine 0.92 이상에서 false positive 매우 낮음.
- 0.75는 "관련은 있지만 같진 않을 수도" — 인간 판단이 가장 가치 있는 구간.
- 0.75 미만 자동 분리는 false negative 회피 (다른 개념을 합치는 것보다 새 노드로 두는 게 회복 가능).

### 4.3 사용자별 보정

- 초반에는 default. 사용자가 active learning 큐에서 merge/split 결정 누적 → **개인별 threshold 자동 조정**.
- 예: 사용자가 0.78에서 자주 split 결정 → flag_threshold를 0.80으로 상향.

## 5. LLM Judge — Borderline에서 신중히

### 5.1 default 사용 안 함

`docs/llm-quiz-hallucination.md` §6 12 bias 적용:
- borderline 샘플 = 작은 N. self-enhancement, position, format bias가 결정적.
- LLM judge agreement는 **80% human agreement** 수준이지만 작은 N에서는 분산이 크다.
- → **LLM judge 단독 결정 금지**, 사용자 확인 우선.

### 5.2 사용 위치 (선택)

- 사용자가 active learning 큐를 쌓아두고 한 번에 처리하기 부담스러울 때 **LLM 1차 판정 + 사용자 spot check** hybrid 옵션.
- 이때도 generator-judge 분리, position shuffle, version pin 적용.

### 5.3 retrieve-then-prompt 패턴 (학계, 2024-2026)

LLM ontology matching에서 표준화된 패턴:
1. embedding으로 후보 entity 검색 (retrieve)
2. LLM에게 후보 + 새 entity 제시 → 매칭 판단 (prompt)

→ 적용한다면: 새 개념의 top-3 후보 노드를 LLM에 제시 → "어느 것과 같은가, 또는 새 노드인가" 단일 호출. 여러 LLM 호출보다 비용/일관성 우위.

## 6. Human-in-the-Loop / Active Learning

### 6.1 표준 framework (entity resolution 학계)

- **Active learning**: 시스템이 control. 가장 informative한 샘플만 사용자에게 라벨 요청.
- **Interactive ML**: 사용자-시스템 상호작용 더 강함.
- **Machine teaching**: 도메인 전문가가 control.
- → code-replay는 **active learning** 모드 default. N=1 사용자에게 부담 최소.

### 6.2 active learning 루프

```
1. 새 target 처리 → 신규 개념 K개 추출
2. 각 개념을 기존 label과 매칭 (§4 threshold)
3. flag (0.75-0.92) 큐에 쌓임 → 사용자에게 batch UI:
   "이 개념과 같은가요?"
   [기존 label A | 기존 label B | 기존 label C | 새 label]
4. 사용자 결정 → label merge 또는 split → 결정을 학습 데이터로 저장
5. 누적된 결정으로 사용자별 threshold + α/β 가중치 미세 조정
```

### 6.3 UX 원칙

- **batch 처리**: 매 target마다 묻지 않고, N개 쌓이면 한 번에. 작업 흐름 방해 최소.
- **explanation**: 왜 이 후보들이 떴는지 (cosine 점수, 공통 키워드) 표시. modern entity resolution 요구사항 (explainability).
- **취소 가능**: 잘못 merge한 결정 되돌릴 수 있어야 함.
- **bypass**: "전부 새 노드로" 옵션도 항상 노출 (uncertain일 때 안전 default).

### 6.4 학습 데이터의 가치

- 같은 사용자의 historical decisions는 그 사용자의 도메인/언어/관심사 신호.
- 30-50개 결정 후 threshold 조정이 의미 있어지는 경험적 시점 (entity resolution 학계 일반).

## 7. Temporal Drift / Alias 처리

### 7.1 문제

- React 18 `useMemo` 사용자가 React 19로 업그레이드 후 같은 패턴을 `use(memo(...))` 새 primitive로 표현.
- 같은 개념인지, 새 개념인지?

### 7.2 처리 방안

- **alias 그래프**: 한 개념 노드에 여러 표현(alias)을 list로 저장. 새 alias가 추가될 때 사용자에게 확인.
- **버전 메타데이터**: 각 카드에 origin PR commit hash + 라이브러리 버전 정보. 마이그레이션 추적 가능.
- **deprecation 플래그**: alias가 더 이상 안 쓰이면 비활성화. 새 표현으로 마스터 alias 변경.

### 7.3 target 간 비교

같은 사용자가 같은 코드베이스 다른 target에서 같은 개념 등장 → 거의 항상 alias 일치. drift는 주로 **시간 / 라이브러리 버전 / 새 패러다임 도입** 때 발생.

## 8. code-replay 적용 (구체)

### 8.1 데이터 구조

```
.codereplay/concepts/<concept-id>.md (frontmatter + body, P1+ 저장 형태 후보)

---
id: cpt_useMemo_caching
canonical_name: useMemo 비싼 계산 캐싱
aliases:
  - React useMemo 의존성 배열
  - 렌더 비용 줄이는 memoization 패턴
embedding_v: 1
created: 2026-05-15
updated: 2026-05-20
mastery: 0.78
card_ids: [card_001, card_005, card_022]
source_targets: [target-001, target-014, target-031]
library_version: react@19.2.0
deprecated_aliases: []
---

(optional 자유 본문 / 사용자 메모)
```

### 8.2 매칭 알고리즘 (의사코드)

```python
def resolve_concept(new_concept, db):
    v = hybrid_embed(new_concept)  # nomic + (CodeBERT) hybrid
    candidates = db.knn(v, k=10)  # blocking
    
    if not candidates: 
        return create_new_node(new_concept)
    
    top = candidates[0]
    sim = cosine(v, top.embedding)
    
    if sim >= 0.92:
        return merge(top, new_concept)  # auto
    elif sim >= 0.75:
        return queue_for_user_review(new_concept, candidates[:3])
    else:
        return create_new_node(new_concept)
```

### 8.3 active learning 큐 UX (의사 마크업)

```
[code-replay] 새 개념 3개 — 같은 거 묶어 줄래?

1. "Drizzle inner join 활용"
   ┌ 가장 가까움 0.83  ▸ "Drizzle relational query"  [같음] [다름]
   ├ 두번째    0.79  ▸ "Drizzle leftJoin 패턴"      [같음] [다름]
   └ 세번째    0.76  ▸ "SQL JOIN 기본"              [같음] [다름]
                                                    [전부 새 노드]

(공통 토큰: drizzle, join | 공통 코드 패턴: db.select().from().innerJoin())
```

### 8.4 평가

- false merge rate (사용자가 "다름"으로 split 결정) 추적 → threshold 자동 조정 신호.
- false split rate (다른 후보를 사용자가 같다고 결정) 추적.
- 목표: flag queue 사용자 결정이 일관되게 한 방향(merge/split) 쏠리면 threshold 조정.

## 9. 위험 / 미해결

- **Embedding 모델 도메인 적합성**: nomic-embed-text는 범용. 한국어 주석/이름 + 코드 hybrid에서 정확도 미측정. → 활성화 시점에 자체 측정.
- **Cold start**: 첫 target 10-20개는 비교 대상이 부족해 거의 모든 개념이 새 label. flag queue 의미 약함. → 명시적 "데이터 수집 중" 상태 표시.
- **Threshold default 의존성**: 0.92/0.75는 Neo4j 컨벤션이지만 도메인마다 최적값 다름. 사용자별 보정 데이터 30개 미만에서는 default 신뢰 낮을 수 있음.
- **alias 폭발**: 한 label에 alias가 너무 많아지면 의미가 흐려짐. **상한 (e.g., 10 aliases) + 분할 제안**.
- **LLM judge 사용 시 비용**: borderline마다 호출하면 target 처리 시간 증가. batch + 사용자 직접 결정이 더 빠를 수 있음.
- **개인 데이터로 학습한 threshold/가중치 export 가능성**: 사용자가 자기 데이터를 다른 도구로 옮길 때 portability 보장 (`docs/local-vs-api-llm.md` §5.4 로컬 우선 원칙).
- **multi-language 코드베이스**: 한 사용자가 TS/Python/Go 섞어 작업하면 embedding 도메인이 복잡해짐. 언어별 별도 sub-index 가능성 검토.
- **추후 자동 split 결정 추가 여부**: alias 누적 후 의미 분기가 명확해지면 자동 split 제안. P0 scope 밖.

## 10. 단계별 적용

| 시점 | 만든다 | 만들지 않는다 |
|---|---|---|
| P0 | 단순 `concept` label/id만 (안정된 이름 부여까지) | 자동 entity resolution, embedding, active learning, alias graph |
| P1 | nomic-embed-text 기반 3-tier threshold + active learning batch UI, hybrid embedding(NL+code), alias graph 초기 | retrieve-then-prompt LLM judge |
| P2 | retrieve-then-prompt LLM judge (선택), 사용자별 threshold 자동 조정, automatic split 제안, transfer 자동 감지와 연동 | DPO ranker, federated 학습 |

## Sources

### Code embedding 모델
- Best Open-Source Embedding Models Benchmarked (Supermemory): <https://supermemory.ai/blog/best-open-source-embedding-models-benchmarked-and-ranked/>
- 10 Best Embedding Models 2026 (Openxcell): <https://www.openxcell.com/blog/best-embedding-models/>
- Benchmark of 16 Open Source Embedding Models for RAG (AIMultiple): <https://aimultiple.com/open-source-embedding-models>
- Embedding and Searching Millions of Code Lines (dasroot 2026-04): <https://dasroot.net/posts/2026/04/embedding-searching-millions-code-lines-efficiently/>
- CodeCSE Multilingual Code+Comment Embeddings (arxiv 2407.06360): <https://arxiv.org/html/2407.06360v1>
- CodeBERT (Microsoft, GitHub): <https://github.com/microsoft/codebert>
- CodeBERT clone detection sentence-transformers (HF): <https://huggingface.co/mchochlov/codebert-base-cd-ft>
- CodeBERT Embeddings overview (Emergent Mind): <https://www.emergentmind.com/topics/codebert-embeddings>
- MTEB benchmark (GitHub): <https://github.com/embeddings-benchmark/mteb/>
- sentence-transformers (HF org): <https://huggingface.co/sentence-transformers>

### Entity resolution / Semantic dedup
- Rise of Semantic Entity Resolution (Graphlet AI Blog, Russell Jurney): <https://blog.graphlet.ai/the-rise-of-semantic-entity-resolution-45c48d5eb00a?gi=e880137a66f4>
- Rise of Semantic Entity Resolution (TDS): <https://towardsdatascience.com/the-rise-of-semantic-entity-resolution/>
- Entity-Resolved Knowledge Graphs (TDS, Mel Richey): <https://towardsdatascience.com/entity-resolved-knowledge-graphs-6b22c09a1442/>
- Entity Resolution at Scale (Shereshevsky, 2026-01): <https://medium.com/@shereshevsky/entity-resolution-at-scale-deduplication-strategies-for-knowledge-graph-construction-7499a60a97c3>
- Entity Resolution and Deduplication (Neo4j Agent Memory): <https://neo4j.com/labs/agent-memory/explanation/resolution-deduplication/>
- Entity Resolution (TigerGraph glossary): <https://www.tigergraph.com/glossary/entity-resolution/>
- Combining ER and KG (Linkurious): <https://linkurious.com/blog/entity-resolution-knowledge-graph/>
- Capturing Concept Similarity with KG (Semantic Web Journal): <https://www.semantic-web-journal.net/system/files/swj3022.pdf>

### Ontology learning / matching
- Ontology learning Wikipedia: <https://en.wikipedia.org/wiki/Ontology_learning>
- Ontology Guided Information Extraction (arxiv 1302.1335): <https://arxiv.org/pdf/1302.1335>
- Ontology-based dataset for process-structure-property (Nature Sci Data): <https://www.nature.com/articles/s41597-024-03926-5>
- ontology-matching GitHub topic: <https://github.com/topics/ontology-matching>
- NLP Techniques for Term Extraction & Ontology Population (GATE book): <https://gate.ac.uk/sale/olp-book/main.pdf>
- Ontology-Based Information Extraction (Wimalasuriya): <https://www.cs.uoregon.edu/Reports/AREA-200903-Wimalasuriya.pdf>
- Agent-OM — LLM Agents for Ontology Matching (arxiv 2312.00326): <https://arxiv.org/html/2312.00326v21>
- Ontology Matching with LLMs and Prioritized DFS (arxiv 2501.11441): <https://arxiv.org/html/2501.11441v1>

### Human-in-the-loop / Active learning
- Deep Indexed Active Learning for ER (arxiv 2104.03986): <https://arxiv.org/pdf/2104.03986>
- Human-in-the-Loop Challenges for Entity Matching (ACM HILDA 2017): <https://dl.acm.org/doi/abs/10.1145/3077257.3077268>
- Neural Networks for Entity Matching Survey (ACM TKDD): <https://dl.acm.org/doi/abs/10.1145/3442200>
- Learning-Based Methods with HITL for ER (ACM CIKM 2019): <https://dl.acm.org/doi/10.1145/3357384.3360316>
- Human-in-the-loop machine learning state of the art (Springer AI Review 2022): <https://link.springer.com/article/10.1007/s10462-022-10246-w>
- Pre-trained data dedup with active learning (ScienceDirect): <https://www.sciencedirect.com/science/article/abs/pii/S095741742502247X>
- Robust Active Learning of Linkage Rules (ResearchGate): <https://www.researchgate.net/publication/333227207_Robust_Active_Learning_of_Expressive_Linkage_Rules>
