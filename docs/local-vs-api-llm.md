# 로컬 LLM vs Cloud API: code-replay의 모델 백엔드 선택

리서치 일자: 2026-05-15
스코프: `docs/diff-to-curriculum.md` §11 미해결 중 두 번째. **입력이 사용자 본인의 개인 PR/diff**라는 특수성 때문에 일반 코딩 어시스턴트의 모델 선택과 트레이드오프가 다르다. 어떤 백엔드를 default로 둘지 결정한다.

## 한 줄 결론

- **Default = 로컬 LLM (Ollama + Qwen2.5-Coder 32B 또는 Kimi K2.6).** 본인 PR이 입력이라 privacy가 압도적으로 중요한 축이고, 학습 자료 생성은 latency 민감하지 않아 로컬의 약점이 거의 노출되지 않는다. 2026-05 시점 로컬 코딩 모델이 GPT-4o급 성능에 도달.
- **Cloud fallback = Anthropic Claude API**. 2025-09부터 API 입출력 **7일 자동 삭제 / 학습 사용 안 함**이 default. 동급 cloud 중 privacy posture 가장 강함. OpenAI는 30일 default / ZDR opt-in 필요라 한 단계 약함.
- **Hybrid 권장**: 사용자에게 "로컬 / Anthropic / OpenAI" 중 선택 + **민감 부분은 무조건 로컬로 라우팅**(예: secret 패턴 감지된 hunk). 단일 백엔드로 잠그지 않음.
- **Batch endpoint 활용**: cloud 선택 시 **50% 할인 batch endpoint**가 학습 자료 생성에 적합(비실시간이라).

## 1. Privacy 축이 왜 결정적인가

- 입력 = 사용자 repo 전체 또는 PR 패치. **proprietary 코드 / 비공개 secret / 내부 API URL** 노출 위험.
- 일반 코딩 어시스턴트(코드 한 줄 자동완성)와 달리 **PR 전체를 LLM에 보낸다**. 노출 surface 훨씬 큼.
- 따라서 "model이 강한가" 이전에 "model 공급자가 입력을 어떻게 다루나"가 1순위 결정 기준.

## 2. Cloud API 비교 (privacy posture)

| 항목 | Anthropic Claude API | OpenAI API |
|---|---|---|
| 기본 retention | **7일** (2025-09부터) | 30일 |
| 기본 학습 사용 | **안 함** (모든 API 호출) | 안 함 (Team/Enterprise, 계약상) |
| Zero Data Retention | Enterprise 자격으로 가능 | 가능, 단 **사전 승인 필요** |
| 30일 retention 옵션 | DPA opt-in으로 연장 가능 | 기본값(반대 방향) |
| Abuse monitoring | 적용되지만 학습 분리 | 30일 default 보관 |
| Commercial Terms 분리 | Consumer terms와 명확 분리 | Enterprise/Team과 Consumer 분리 |

**해석**:
- Anthropic이 "default 가장 안전"한 posture. **별도 설정 없이도 7일 + no-training**.
- OpenAI는 동등하게 만들려면 **ZDR 사전 승인 절차 필요**. 일반 사용자는 30일 retention 위에 있음.
- code-replay 사용자가 별도 enterprise 계약 없이 쓰는 게 default 시나리오라, Anthropic이 cloud fallback 1순위.

## 3. 로컬 LLM 옵션 (2026-05 시점)

### 3.1 모델 라인업

| 모델 | 크기 | 코드 성능 | 권장 RAM | 비고 |
|---|---|---|---|---|
| Kimi K2.6 | MoE 1T total / 42B active | **87/100 real-world coding (Tier A)** | 20GB+ | 2026-05 기준 최고 로컬 코딩 모델. 비서구권 최초 Tier A |
| Qwen2.5-Coder 32B Instruct | 32B | EvalPlus/LiveCodeBench/BigCodeBench 다수 SOTA, GPT-4o 경쟁 | 20GB+ | OSS 코드 모델 표준 |
| Qwen 3 7B | 7B | HumanEval 76.0 (under-8B 최고) | 8GB | 가벼운 라인 |
| DeepSeek Coder V2 | 16B | Python/JS 강세 | 16GB | |
| Llama 3.3 70B | 70B | 범용 로컬 최강 | 40GB+ | code 특화는 아님 |
| Devstral Small | 24B | 균형 | 16GB | |

### 3.2 인프라

- **Ollama** — 사실상 표준 로컬 추론 런타임. 2026 Q1 monthly downloads **52M**. CLI / HTTP API / 모델 라이브러리 통합.
- **LM Studio, MLX, vLLM, llama.cpp** — 대안.
- **Hugging Face** — 모델 배포.

### 3.3 하드웨어 옵션

| 하드웨어 | 가격 | 적합 모델 |
|---|---|---|
| RTX 5090 | ~$1,999 | Qwen2.5-Coder 32B (양자화) / Kimi K2.6 partial |
| M4 Ultra Mac Studio | $3,999+ | Kimi K2.6 / Qwen3.6 27B 풀스펙 |
| 16GB Mac (M3) | (사용자 보유) | Qwen 7B / Devstral 24B (양자화) |
| GPU 없는 노트북 | 0 | 사실상 cloud 필요 |

→ code-replay 1차 페르소나가 "본인 repo 가진 숙련 dev"라 **Apple Silicon Mac 보유 비율 높을 것으로 가정**. M3/M4 16-20GB+ 대상으로 시작.

## 4. Cost 비교

### 4.1 Cloud (2026 시세)

- **개발자당 $50-200/월**, 모델·사용량에 따라. GPT-4.1 / Claude Sonnet 4 / Gemini 2.5 Pro 기준.
- 10인 팀 → **연 $6,000-24,000**.
- **Batch endpoint 50% 할인** — 비실시간 처리(학습 자료 생성)에 적합.

### 4.2 Local

- 1회 하드웨어 투자 후 한계비용 ≈ 전기료.
- 미디엄 유즈(일 3M-5M 토큰) 기준 **18-24개월 break-even**.
- code-replay는 PR마다 generate라 토큰량 미디엄 ~ 라이트. break-even 더 길 수 있음.

### 4.3 해석

- 개인 사용자 단독은 **API가 단기적으로 저렴**. PR당 학습 자료 생성 토큰량이 작아 월 $5-20 수준 가능.
- 팀/조직 단위로 보면 로컬이 시간 누적시 유리.
- **그러나 code-replay에서 cost는 2순위**. privacy 축이 우선이라 비용 동등하면 로컬.

## 5. code-replay 추천 디플로이먼트

### 5.1 MVP 1차 — 로컬 default + cloud opt-in

```
사용자 선택:
  [default] Local: Ollama + Qwen2.5-Coder 32B  (또는 사용자 지정 모델)
  [opt-in]  Cloud: Anthropic Claude API (sonnet 4)
  [opt-in]  Cloud: OpenAI API (gpt-5 mini/full)
```

- 설정 파일 (`.codereplay/config.toml`):
  - `model_backend = "ollama"` (default)
  - `ollama_model = "qwen2.5-coder:32b-instruct"` (사용자 환경 따라 7b/14b/32b)
  - cloud 사용 시 `api_key` env 변수 (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY`)

### 5.2 Hybrid 라우팅 (1.5차)

- **민감 패턴 감지된 hunk는 무조건 로컬**: secret/key/internal URL/customer data 패턴 → 정규식 + entropy heuristic → 강제 로컬 처리.
- **나머지는 사용자 선택**: cloud로 보낼 수 있음.

### 5.3 Cloud 사용 시 권장

- **Anthropic Claude Sonnet 4 default** (7일 retention, no-training default).
- Batch endpoint 사용 (50% 할인, 비실시간 학습 자료 생성).
- API 키 ZDR enterprise 사용자는 사전 승인 신청 권장.

### 5.4 안 하는 것

- **사용자 PR 데이터를 code-replay 자체 서버로 보내지 않음.** product가 로컬 CLI/extension으로만 동작. 사용자 → 사용자 선택 LLM(로컬 또는 직접 cloud)으로 직접. **중계 서버 없음** = privacy story 단순.

## 6. 위험 / 미해결

- **GPU 없는 사용자 UX**: 사양 부족자는 cloud opt-in 강제됨. **API key onboarding 마찰**이 큰 적용 장벽 가능. → 첫 실행 가이드 필요.
- **양자화 품질 손실**: Q4_K_M / Q5_K_M 양자화에서 fill-in-the-blank 정답 판정 정확도 손실 측정 필요. → 자체 eval 필요.
- **로컬 모델 학습 컷오프**: 2025년 학습된 모델이 2026년 새 라이브러리(React 19+, Next 15+ 등)를 모를 수 있음. retrieval / docs context 보강 필요.
- **AI vs human 작성 구분 신호로서 모델 출력은 못 씀**: 로컬/cloud 어느 쪽이든 "AI가 짠 코드 같다"는 판정은 AST/blame 기반(`docs/ast-diff-tools.md` §5)이지 LLM 자체 판정 아님.
- **OpenAI ZDR 사전 승인 절차**가 개인 사용자에게 사실상 불가. OpenAI를 default cloud로 쓰면 30일 retention 노출 → narrative 약함.
- **API 약관 변경 리스크**: 7일 / no-training은 정책이라 향후 변경 가능. 정기 검토 필요.

## Sources

### Local LLM 모델 / 인프라
- Open-Source LLMs 2026 (Hugging Face): <https://huggingface.co/blog/daya-shankar/open-source-llms>
- Best LLM for Coding 2026 (WhatLLM): <https://whatllm.org/best-llm-for-coding>
- Best Local LLM Models 2026 (SitePoint): <https://www.sitepoint.com/best-local-llm-models-2026/>
- Best Local LLMs for Coding 2026 — Kimi K2.6 vs Qwen: <https://www.promptquorum.com/local-llms/best-local-llms-for-coding>
- Best Open Source Self-Hosted LLMs (Pinggy): <https://pinggy.io/blog/best_open_source_self_hosted_llms_for_coding/>
- Qwen2.5-Coder Ollama: <https://ollama.com/library/qwen2.5-coder>
- Best Local Coding Models Ranked (InsiderLLM): <https://insiderllm.com/guides/best-local-coding-models-2026/>
- Best Local AI Coding Models for Ollama 2026: <https://localaimaster.com/models/best-local-ai-coding-models>

### Anthropic privacy
- Anthropic Claude Data Retention Policy 2026 (Char Blog): <https://char.com/blog/anthropic-data-retention-policy/>
- Anthropic Privacy Center — model training: <https://privacy.claude.com/en/articles/7996868-is-my-data-used-for-model-training>
- Updates to consumer terms: <https://www.anthropic.com/news/updates-to-our-consumer-terms>
- Updates to Privacy Policy: <https://privacy.claude.com/en/articles/10301952-updates-to-our-privacy-policy>
- Anthropic Privacy Audit 2026 (terms.law): <https://terms.law/Privacy-Watchdog/ai-services/anthropic/>
- Claude AI Updates impact on Privacy (AMST Legal): <https://amstlegal.com/anthropics-claude-ai-updated-terms-explained/>

### OpenAI privacy
- Data controls in OpenAI platform: <https://developers.openai.com/api/docs/guides/your-data>
- OpenAI Privacy Policy: <https://openai.com/policies/row-privacy-policy/>
- OpenAI Zero Data Retention (Medium): <https://medium.com/@jeffkessie50/openais-zero-data-retention-policy-916ff04a3599>
- ZDR Information (OpenAI community): <https://community.openai.com/t/zero-data-retention-information/702540>
- OpenAI Business data privacy: <https://openai.com/business-data/>
- OpenAI Enterprise privacy: <https://openai.com/enterprise-privacy/>
- GPT-5 Training Data Opt-Out (Secure Privacy): <https://secureprivacy.ai/blog/gpt-5-training-data-opt-out>

### Local vs Cloud cost
- Local AI Coding vs Cloud Performance 2026 (SitePoint): <https://www.sitepoint.com/local-vs-cloud-ai-coding-performance-analysis-2026/>
- Local LLMs vs Cloud APIs TCO 2026 (SitePoint): <https://www.sitepoint.com/local-llms-vs-cloud-api-cost-analysis-2026/>
- Running Local AI Models for Coding 2026 (DEV.to): <https://dev.to/alexcloudstar/running-local-ai-models-for-coding-in-2026-when-cloud-tools-are-not-the-answer-gfb>
- AI Coding Tools Pricing 2026 (IJONIS): <https://ijonis.com/en/ai-coding-tools-pricing>
- Local vs Cloud AI Coding (InnerZero): <https://innerzero.com/blog/local-ai-vs-cloud-ai>
- AI Coding Tools Pricing Comparison (Developers Digest): <https://www.developersdigest.tech/blog/ai-coding-tools-pricing-comparison>
