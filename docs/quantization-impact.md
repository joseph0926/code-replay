# 로컬 LLM 양자화 품질 영향: 로컬 백엔드 운영 참고

리서치 일자: 2026-05-15
스코프: 로컬 LLM 사용 시 양자화 단계가 review card 생성과 채점 품질에 미치는 영향 정리. 양자화 tuning 자체는 P0 표준 작업이 아니다. P0 default는 합리적인 단일 quantization을 한 가지 골라 두고 그대로 사용하는 것이며, 본 문서의 단계별 권장은 P1+ 로컬 백엔드 최적화나 사용자 환경별 가이드로 사용한다.

## 한 줄 결론

- **P0 default**: 학습 포인트 추출과 review card 생성에 **GGUF Q5_K_M** 하나만 잡아 둔다. 양자화 tuning 자체는 P0 표준 흐름이 아니며, P1+ 로컬 백엔드 최적화에서 다시 본다.
- **LLM-as-judge / reconstruction 채점**: P0 비-목표(`docs/learning-metrics.md` §4). 도입할 때는 **Q8_0 또는 cloud API**. judge는 hallucination 영향이 카드 품질에 직접 전이되므로 정밀도 양보 안 함.
- **structured output (카드 JSON): 양자화 단계 ≤ 4bit는 회피.** 4bit는 pre-emptive decoding mechanism이 깨지는 사례 있음 (Cleanlab 보고). schema 강제(grammar-constrained generation)로 보강한다.
- **format 선택**: Apple Silicon은 **GGUF Q5_K_M** (ollama default). CUDA GPU 보유자는 **Marlin-AWQ**가 속도(741 tok/s) + 품질(95%) 동시 우위.
- **Qwen2.5-Coder 가족이 양자화 tolerance 가장 높다는 게 학계 컨센서스** → 로컬 default 모델로 합리적.

## 1. 양자화 형식 비교 (4-bit 기준)

| Format | 품질 보존 | 속도 (4-bit) | 환경 적합 | code-replay 위치 |
|---|---|---|---|---|
| **AWQ** | **~95%** | 빠름 (Marlin-AWQ 741 tok/s) | NVIDIA GPU | CUDA 보유자 1순위 |
| **GGUF Q4_K_M** | ~92% | 중간 | CPU + Apple Silicon + GPU offload | **Mac default 1순위** |
| **GPTQ** | ~90% | 매우 빠름 (Marlin 5x GGUF) | NVIDIA GPU 전용 | 코드 토큰 fidelity 우위 — 정밀 필요 시 |
| **EXL2** | 가변 (bpw 조정) | 빠름 | NVIDIA GPU | 후순위 |
| **NVFP4** | 신형 (FP4) | 매우 빠름 | NVIDIA Blackwell+ | 신규 하드웨어만 |
| **bitsandbytes** | ~92% | 중간 | 범용 fallback | dev 빠른 시작용 |

**해석**:
- 동일 4-bit에서 AWQ가 GGUF보다 3pp 높은 품질 retention. 단 GGUF는 Apple Silicon 호환이라 Mac 사용자에겐 사실상 유일 선택.
- **Marlin-AWQ가 production sweet spot** — Pass@1 51.8% 유지하면서 가장 빠름.

## 2. Bit precision별 품질 손실 (일반적 합의)

| Precision | Perplexity / 정확도 영향 | 비고 |
|---|---|---|
| FP16 / BF16 | baseline | 정확도 손실 0, 메모리 가장 큼 |
| Q8_0 (8-bit) | **<2% perplexity 증가**, 8-bit 레시피 정확도 ≤0.8% drop | "체감 무손실" 구간 |
| Q6_K | ~1-2% loss | Q8과 Q5 사이의 균형점 |
| Q5_K_M (5-bit) | ~3-5% loss | **16GB 시스템 sweet spot** |
| Q4_K_M (4-bit) | **2-8% degradation**, MMLU 1-3pt 손실 | 메모리 제약 시 표준 |
| Q3_K | 8-15% loss | 안 권장 (특히 코드) |
| Q2_K | **15-30% loss** | 거의 사용 X |

- **8-bit = 체감 무손실 구간**, 4-bit = "허용 가능" 구간, 3-bit 이하 = 위험.
- multi-step 추론 / instruction following / 다국어가 양자화에 가장 취약. **수학 GSM8K 같은 일부 벤치는 4bit에서도 robust**.

## 3. 코드 도메인 특이사항

- **token-level fidelity가 결정적**: 코드는 한 글자(괄호, 콜론, 쉼표) 차이로 결과가 달라짐. 자연어 generation보다 양자화 영향 더 직접적.
- **GPTQ-INT4 / INT8이 GGUF보다 token 정확도 우위** (정성적 보고 다수). AWQ와 lower GGUF는 completion 일부 destabilize 사례 있음.
- **Qwen2.5-Coder-32B GPTQ-INT4 → HumanEval/BBH 마진 회귀만**. **Qwen2.5 family = 양자화 tolerance 최고**라는 게 2026 시점 컨센서스.
- HumanEval Pass@1 비교 (한 벤치): AWQ / Marlin-AWQ / GGUF Q4_K_M / BitsandBytes 모두 **51.8%** (FP16과 큰 차이 없음). **단순 코드 생성은 4bit에서 robust**.
- 그러나 **multi-step refactoring / repo-level 추론**처럼 token sequence가 길고 의존성이 깊은 task는 4bit에서 누적 오류 위험.

## 4. Structured output (JSON) 특이사항

code-replay는 review card 산출을 일정한 구조(질문, 정답, rubric, target/replay 연결, verified flag 등)로 만들어야 하므로 structured output 안정성이 중요하다.

- **smaller model (8B 이하)이 양자화에 더 민감** — 작은 모델 + 4bit + structured output은 위험 조합.
- Cleanlab 분석: **양자화된 모델에서 pre-emptive decoding이 작동 안 함** → structured output 신뢰도 떨어짐. FP16에서만 안정적.
- **8-bit 레시피 (FP8 / GPTQ-INT8): ≤0.8% accuracy drop** → JSON output 안정성 거의 영향 없음.
- 4bit + JSON 조합에서는 **grammar-constrained generation** (llama.cpp의 GBNF, vLLM의 outlines, ollama의 json mode)을 schema 강제로 사용하는 것이 거의 필수. 단순 prompt JSON 요청은 깨짐 사례 다수.

## 5. Qwen2.5-Coder 가족 — 양자화 tolerance

`docs/local-vs-api-llm.md` §3.1에서 default 후보로 정한 모델. 양자화 측면에서 selection 보강:

- **공식 GGUF 옵션 다양**: q2_K / q3_K_M / q4_0 / **q4_K_M** / q5_0 / **q5_K_M** / q6_K / **q8_0**.
- Coder variants가 일반 Qwen variants보다 **HumanEval/BBH에서 양자화에 더 견고**.
- 32B 기준 GPTQ-INT4에서 marginal regression만. INT8은 사실상 무손실.
- 7B 기준 Q5_K_M이 16GB 환경 대부분에서 메모리 안에 들어옴.

## 6. code-replay 컴포넌트별 권장 quantization

| 컴포넌트 | 추천 quantization | 이유 | P0/P1/P2 |
|---|---|---|---|
| 학습 포인트 추출 (AST + LLM) | Q5_K_M default | structured 기반이라 schema 강제 + grammar 사용 | P0 |
| `fill_blank` / `explain_decision` review card 생성 | **Q5_K_M minimum, Q8 권장** | structured output + 정답 정확도 결정적, 카드는 사용자에게 직접 노출 | P0 |
| 변경 요약 / 자연어 설명 (보조 맥락) | Q5_K_M (GGUF) 또는 Marlin-AWQ | 자연어 토큰 fidelity 중요도 중간. 4bit에서 충분, 5bit 마진 | P0 |
| reconstruction 채점 LLM-as-judge | **Q8_0 또는 cloud API** | judge hallucination이 채점 품질로 직결 — 정밀도 양보 X | P1 |
| concept 동일성 embedding | (별도) embedding model 자체 양자화는 영향 적음 | `docs/concept-identity.md` 참조 | P1+ |
| privacy 민감 영역 처리 | 무조건 로컬, Q5_K_M+ | cloud 격리 필수 (`local-vs-api-llm.md` §5.2) | P0 |

### 6.1 디플로이먼트 default (참고용)

`docs/mvp-spec.md`의 표준 구성은 `.codereplay/config.json`이며, 아래 예시는 로컬 백엔드 운영을 가정한 참고 형태다. P0에서는 generation_model 하나만 고정해 두는 것이 최소 구성이며, judge/embedding 항목은 P1+ 도입 시 추가한다.

```jsonc
// .codereplay/config.json (참고 예시)
{
  "model_backend": "ollama",
  "generation_model": "qwen2.5-coder:32b-instruct-q5_K_M",
  // 아래는 P1+ 단계에서 도입 후보
  // "judge_model": "qwen2.5-coder:32b-instruct-q8_0",
  // "embedding_model": "nomic-embed-text",
  "structured_output_grammar": true
}
```

### 6.2 메모리 부족 시 단계적 강등

| RAM | 권장 generation | 권장 judge |
|---|---|---|
| 32GB+ | Qwen2.5-Coder-32B Q5_K_M | Q8_0 |
| 16-24GB | Qwen2.5-Coder-14B Q5_K_M | Q8_0 (14B) 또는 cloud judge |
| 8-12GB | Qwen2.5-Coder-7B Q5_K_M | cloud judge 강제 (로컬 7B Q4 judge는 신뢰 낮음) |
| <8GB | cloud generation + judge | 로컬 단념 |

## 7. 위험 / 미해결

- **자체 eval은 후속 단계**: 위 수치들은 일반 벤치(HumanEval, MMLU). **fill_blank 정답 정확도, distractor 적절성, judge 일관성**은 code-replay 자체 eval set으로 측정해야 한다. P0에서는 default quantization 한 가지만 고정하고, 자체 eval/fixture 누적은 P1+ 로컬 백엔드 최적화 단계에서 다룬다.
- **다국어/한국어 영향**: 양자화는 비영어 컨텍스트에서 더 큰 손실 사례. 사용자 코드 안의 한글 주석 / 한국어 prompt 영향 별도 측정 필요.
- **hardware accelerator별 차이**: M-series Neural Engine, NVIDIA Marlin, AMD ROCm에서 같은 양자화도 성능/품질 다름. 첫 실행 자동 벤치 + 권장 모델 제안하는 selector 도구 필요.
- **양자화 + tool use**: 양자화된 모델은 tool/function calling 정확도가 더 떨어진다는 보고 다수. code-replay가 tool use 패턴 채택하면 별도 검증.
- **모델 업데이트 주기**: Qwen3 / Qwen3-Coder 출시 시 같은 quantization 권장이 유지되는지 재측정 필요.

## Sources

### 양자화 형식 비교
- The Complete Guide to LLM Quantization with vLLM (Jarvis Labs): <https://jarvislabs.ai/blog/vllm-quantization-complete-guide-benchmarks>
- Quantization Methods Compared — GGUF/AWQ/GPTQ/EXL2/NVFP4 (ai.rs): <https://ai.rs/ai-developer/quantization-methods-compared>
- GGUF vs GPTQ vs AWQ Best Quantization 2026 (Local AI Master): <https://localaimaster.com/blog/quantization-explained>
- Which Quantization Method Is Best (E2E Networks): <https://www.e2enetworks.com/blog/which-quantization-method-is-best-for-you-gguf-gptq-or-awq>
- Which Quantization Method (Maarten Grootendorst newsletter): <https://newsletter.maartengrootendorst.com/p/which-quantization-method-is-right>
- LLMs on CPU GGUF/AWQ/GPTQ (Ionio): <https://www.ionio.ai/blog/llms-on-cpu-the-power-of-quantization-with-gguf-awq-gptq>
- AWQ vs GGUF vs GPTQ 2026 (index.dev): <https://www.index.dev/skill-vs-skill/ai-gptq-vs-awq-vs-gguf>
- GGUF Quantization Quality vs Speed on Consumer GPUs (dasroot 2026-02): <https://dasroot.net/posts/2026/02/gguf-quantization-quality-speed-consumer-gpus/>
- Tradeoffs Explained (BestAIWeb): <https://www.bestaiweb.ai/gptq-vs-awq-vs-gguf-vs-bitsandbytes-quantization-formats-and-their-tradeoffs-explained/>

### Bit precision별 영향
- Qwen 3.6 Quantization Deep Dive Q4_K_M vs Q8_0 (dasroot 2026-05): <https://dasroot.net/posts/2026/05/qwen-36-quantization-bf16-gguf-q4-k-m-q8-0/>
- LLM Quantization Explained Q4_K_M Q5 Q8 (Mustafa.net 2026): <https://mustafa.net/llm-quantization-explained/>
- How to Choose LLM Quantization Q4_K_M for 6GB (PromptQuorum): <https://www.promptquorum.com/local-llms/llm-quantization-explained>
- Demystifying LLM Quantization Suffixes (Paul Ilvez Medium): <https://medium.com/@paul.ilvez/demystifying-llm-quantization-suffixes-what-q4-k-m-q8-0-and-q6-k-really-mean-0ec2770f17d3>
- Which Quantization Should I Use? Unified Evaluation (arxiv 2601.14277): <https://arxiv.org/pdf/2601.14277>
- Benchmarking Quantized LLMs (Ionio): <https://www.ionio.ai/blog/llm-quantize-analysis>
- Quantization benchmarks (AWS samples): <https://github.com/aws-samples/optimizing-llm-inference-with-quantization/blob/main/Quantization%20benchmarks.md>
- Quantized LLMs Cost & Performance (Latitude): <https://latitude.so/blog/quantized-llms-cost-performance-results>

### Qwen2.5-Coder GGUF
- Qwen2.5-Coder-7B-Instruct-GGUF (공식): <https://huggingface.co/Qwen/Qwen2.5-Coder-7B-Instruct-GGUF>
- Qwen2.5-Coder-14B-Instruct-GGUF (공식): <https://huggingface.co/Qwen/Qwen2.5-Coder-14B-Instruct-GGUF>
- Qwen2.5-Coder-7B-Instruct-GGUF (bartowski): <https://huggingface.co/bartowski/Qwen2.5-Coder-7B-Instruct-GGUF>
- Qwen2.5-Coder-32B-Instruct-GGUF (bartowski): <https://huggingface.co/bartowski/Qwen2.5-Coder-32B-Instruct-GGUF>
- Qwen2.5-Coder Technical Report (arxiv 2409.12186): <https://arxiv.org/html/2409.12186v1>
- Qwen 2.5 Models Overview (Emergent Mind): <https://www.emergentmind.com/topics/qwen-2-5-models>
- Speculative decoding for big LLMs on consumer GPUs (llama.cpp #10466): <https://github.com/ggml-org/llama.cpp/discussions/10466>

### Structured output 영향
- Half a million evaluations on quantized LLMs (Red Hat Developer): <https://developers.redhat.com/articles/2024/10/17/we-ran-over-half-million-evaluations-quantized-llms>
- Comprehensive Evaluation of Quantization Strategies (arxiv 2402.16775): <https://arxiv.org/html/2402.16775v1>
- LLM Structured Output Benchmarks Mistakes (Cleanlab): <https://cleanlab.ai/blog/structured-output-benchmark/>
- Generating Structured Outputs from LMs Benchmark (arxiv 2501.10868): <https://arxiv.org/html/2501.10868v1>
- LLM Inference Handbook — Quantization (BentoML): <https://bentoml.com/llm/getting-started/llm-quantization>
- Assessing Accuracy Loss in Quantized LLMs (apxml): <https://apxml.com/courses/quantized-llm-deployment/chapter-3-performance-evaluation-quantized-llms/evaluating-accuracy-degradation>
- Practical Guide to LLM Quantization (Hivenet): <https://www.hivenet.com/post/llm-quantization-guide>
- Guide to Quantization in LLMs (Symbl.ai): <https://symbl.ai/developers/blog/a-guide-to-quantization-in-llms/>
