# AI 사용이 개발자 이해/학습에 미치는 영향

리서치 일자: 2026-05-15
스코프: docs/topic.md의 전제(AI가 코드를 대신 쓰면 결과는 빨리 나오지만 본인 실력으로는 쌓이지 않는다)를 뒷받침하거나 반박하는 2025-2026 실증 연구를 정리. code-replay의 시장 정당화와 MVP 필요성을 결정한다.

## 한 줄 결론

- **"AI를 많이 쓰면 결과는 빨라도 개념 retention은 떨어진다"는 가설은 2026년 시점에서 RCT/quasi-experiment로 여러 번 재현됐다.** Psychology Today (Feb 2026) 보고: AI 그룹이 사후 quiz 점수 평균 -17% (대조군 대비). Anthropic 자체 연구도 같은 방향.
- **METR 자체는 입장이 흔들렸다.** 2025년 7월 -19% (AI 사용 시 19% 느려짐) 결과가 유명해진 뒤, 2026년 2월 redesign 발표하며 "지속 노출 시 +19% 상승 가능성"을 시사. 단 selection effect caveat 강하게 명시. **단일 결과로 결론 내리기엔 시그널이 진동 중**.
- **결정적인 mediator는 "어떻게 AI를 쓰는가"다.** 순수 delegate(코드만 받기) → 점수 최저. "코드 + 설명 요청" → 중간. AI 미사용 → 최고. **= "설명을 다시 본인 머리로 통과시키는 단계"가 결정적**. 이게 code-replay가 메우려는 정확한 틈이다.
- **처방 방향은 "structured unassisted practice"로 수렴.** Augmentation Trap 논문(arxiv 2604.03501)의 비유: 자동조종 시대에도 조종사는 정기적으로 hand-flying 시간을 유지해야 면허가 유지되는 구조. **code-replay의 "재구현 과제"가 정확히 이 역할**.

## 1. METR 시리즈: 시그널 진동

### 1.1 METR 2025-07 (원전, arxiv 2507.09089)

- 디자인: 16명의 숙련 OSS 개발자 × 246개 실제 task. mature project, 평균 5년 경력.
- 결과: AI 사용 조건에서 task 완료 시간 **19% 증가** (= AI가 더 느리게 만듦).
- 자가 보고 vs 실측 격차: 사전 예측 -24%, 사후 회고 -20%, 실측 +19%. **개발자는 자기가 빨라졌다고 강하게 착각**.
- 이게 docs/alternative.md에 인용된 원래 신호다.

### 1.2 METR 2026-02 redesign 발표

- 2025년 8월 시작한 후속 실험은 "최신 AI를 쓰는 더 큰 풀"이었지만 noisy signal 때문에 **신뢰 부족 → redesign**.
- 참여자 대화 기반 비공식 추정: 2025 초기보다 2026 초기에 "AI로 인해 빨라진 정도"가 더 클 것이라 본다.
- 후속 데이터에서 **지속 노출자는 +19% 생산성** 신호. **단, selection effect 강조**(잘 적응하는 사람만 남음).
- 해석: METR 자체가 "AI = 느리다"라는 단순 결론을 회수하는 중. 그러나 "스피드"와 "이해/유지보수"는 다른 축.

### 1.3 METR 2026-05-11 self-reported survey

- self-report 기반 추가 데이터. 측정 한계가 명시돼 있어 RCT 결론과 분리해 읽어야 함.

### 1.4 함의

- "AI = 무조건 느리다"로 알려진 -19%는 2026년 시점에서 단순화된 narrative.
- 그러나 **속도 지표가 회복돼도 "이해/장기 retention"은 별개 측정 대상이다.** code-replay의 정당화는 속도 축이 아니라 retention 축에서 와야 한다 → §3, §4 결과가 더 결정적.

## 2. GitHub Copilot — 학습/이해 측면 실증

### 2.1 ICER 2025 (Brownfield Coding)

- 컴퓨팅 학생 대상, 기존 코드 이해가 필요한 brownfield task에서 Copilot 사용 시 task 완료 시간 **-34.9%**.
- 동시에 보고된 우려: AI assistant가 **인지 부하 자체를 떠안고**, 학생은 **cognitively disengaged** 상태가 될 수 있음.
- 긍정 측면: extraneous cognitive load(syntax, boilerplate) 감소 → 본질 복잡성에 working memory 할당 가능. **"무엇을 offload하는가"가 결정적**.

### 2.2 GitHub 자체 데이터

- 90% 개발자가 Copilot 제안 코드를 commit. 88% character retention rate. **"받은 코드를 그대로 쓰는" 빈도가 매우 높음**. = 본인이 다시 쓰는 단계가 거의 없다 → §3 cognitive offloading 위험과 직결.

### 2.3 함의

- Copilot은 속도 측면에서 검증됨. 그러나 학습/이해 측면은 **"어떤 인지 부하를 덜었는가"에 따라 효과 부호가 갈림.** boilerplate/syntax offload는 +, 핵심 설계 판단 offload는 -.

## 3. Cognitive Offloading — code-replay의 핵심 정당화

### 3.1 Psychology Today (Feb 2026) 보고

- 실험: 같은 학습 task를 AI 사용 그룹과 비사용 그룹에 배정 → 사후 quiz로 retention 측정.
- **AI 그룹 평균 점수가 대조군 대비 17% 낮음**.
- 세분화:
  - **순수 delegation** (AI에게 코드만 시킴) → 점수 최저
  - **AI에게 코드 + 설명 요청** → 중간
  - **AI 미사용** → 최고
- 결론: "AI 사용 자체"가 아니라 **"설명을 본인 머리로 다시 통과시키느냐"가 mediator**.

### 3.2 Anthropic 자체 연구 (AI assistance and coding skill formation)

- Anthropic이 AI assistance가 코딩 skill 형성에 미치는 영향을 측정한 연구를 공개. 같은 방향(skill formation 저하 신호)을 보고. **vendor 자체가 risk를 인정한다는 점이 정책적으로 중요**.

### 3.3 The Augmentation Trap (arxiv 2604.03501, 2026-04)

- 핵심 분류:
  - **performance-extracting deployment** — 노동자의 현재 skill을 뽑아내지만 유지하지 않음. 장기적으로 skill 고갈.
  - **skill-preserving deployment** — 생산성을 유지하면서도 worker의 expertise가 계속 발달.
- 처방: **structured unassisted practice**. 자동조종 시대에도 조종사는 정기적 hand-flying으로 면허 유지하듯, AI 시대 개발자도 **AI 없이 짜는 시간**을 의도적으로 확보해야 함.
- recovery rate(skill 회복률)을 끌어올리는 수단이 product 차원으로 구현 가능.

### 3.4 함의 — code-replay의 정확한 위치

- code-replay = **"AI로 만든 PR을 retro-actively unassisted re-implementation으로 변환하는 도구"**.
- §3.1의 mediator(설명을 본인 머리로 다시 통과)와 §3.3의 처방(structured unassisted practice)을 동시에 만족.
- 이게 단순 "AI tutor"나 "코드 리뷰 도우미"와 카테고리적으로 다른 점이고, MVP 산출물(재구현 과제 / 빈칸 / 복습)이 정확히 이 처방을 instantiate한다.

## 4. Vibe Coding 자체에 대한 연구

### 4.1 정의 / 확산

- Karpathy(2025-02) 명명. AI가 자연어 프롬프트로부터 코드를 만들고, 개발자는 결과 vibe만 평가하는 모드.
- 2026년 시점에서 grey literature, mixed-method case study, qualitative study 다수 등장.

### 4.2 Vibe-Check Protocol (arxiv 2601.02410, 2026-01)

- AI 프로그래밍에서 cognitive offloading을 **정량적으로 측정**하는 프로토콜 제안. 학계가 cognitive offloading을 가설이 아니라 metric화 단계로 옮기고 있다는 신호.

### 4.3 ACM TechBrief on Vibe Coding

- 정책 차원에서 ACM이 vibe coding을 다룰 정도로 토픽이 mainstream화됐다. **시장 신호로서 강함**.

### 4.4 위험군

- vibe coder의 약 14%는 **접근성/empowerment 동기**. 즉 비전공자가 AI로 코드를 짜는 케이스. 문제 발생 시 대응할 도구가 없다 → 새로운 vulnerable developer 계층 발생.

### 4.5 함의

- code-replay의 1차 페르소나는 **숙련 개발자가 AI로 짠 자기 PR을 다시 학습화**하는 케이스. 그러나 §4.4를 고려하면 **2차 페르소나(vibe coder가 자기 product 코드를 사후에 이해하기)**도 시장이 빠르게 형성 중. MVP scope에는 첫째만 두되, narrative에는 둘째를 의식.

## 5. code-replay 차원으로 환원

| 연구 신호 | code-replay 결정 |
|---|---|
| AI 사용 시 quiz 점수 -17% | 산출물 핵심 KPI를 "속도"가 아니라 **개념 retention** 측정으로 잡기 |
| "코드+설명 요청" 그룹이 순수 delegate보다 나음 | 단순 요약 X, **Socratic prompt + 본인 답변 입력**까지 강제 |
| structured unassisted practice 처방 | "재구현 과제"는 AI 솔루션을 가리고 본인이 짜게 한 뒤 비교. **AI 차단 모드** 필수 |
| 88% character retention(=받은 코드 그대로 commit) | 입력은 사용자가 명시적으로 전달한 agent-assisted **target**으로 한정 (Code Replay는 도구가 diff에서 작성 주체를 추론하지 않음) |
| METR 신호 진동 | 생산성 narrative와 거리 두기. retention narrative만 사용 |
| ACM TechBrief / vibe coding mainstream화 | 1차 페르소나(숙련 dev) 외에 2차(vibe coder) narrative 준비 |

## 6. 미해결 질문 / 추가 리서치 후보

- **PR 단위 retention 측정 방법**: §3.1 quiz는 인공 setting. 실제 repo 작업 흐름에서 retention을 어떻게 측정할지 확립된 방법론이 없음. → MVP의 dogfooding이 사실상 첫 번째 측정.
- **AI 그룹의 skill 회복 곡선**: structured unassisted practice를 얼마나 자주 / 얼마나 오래 해야 회복되는가? Augmentation Trap 논문은 정성적 처방, 정량 처방 부재. → 자체 데이터가 곧 자산.
- **개인 보안/privacy boundary**: AI 학습 영향 연구는 대부분 공개 task. 본인 사적 PR을 학습 자료로 변환하는 흐름의 boundary는 별도 검토 필요.

## Sources

- METR 2025-07 원전 (arxiv 2507.09089): <https://arxiv.org/abs/2507.09089>
- METR 2025-07 블로그: <https://metr.org/blog/2025-07-10-early-2025-ai-experienced-os-dev-study/>
- METR 2026-02 redesign 발표: <https://metr.org/blog/2026-02-24-uplift-update/>
- METR 2026-05-11 self-reported survey: <https://metr.org/blog/2026-05-11-ai-usage-survey/>
- METR 참여자 후기 (Domenic Denicola): <https://domenic.me/metr-ai-productivity/>
- METR 2025 결과 해설 (Simon Willison): <https://simonwillison.net/2025/Jul/12/ai-open-source-productivity/>
- Copilot brownfield study (ICER 2025): <https://arxiv.org/html/2506.10051v1>
- Copilot brownfield ACM ICER 페이지: <https://dl.acm.org/doi/10.1145/3702652.3744219>
- Copilot 초기 productivity (arxiv 2302.06590): <https://arxiv.org/abs/2302.06590>
- Cognitive offloading & skill formation (Psychology Today, Feb 2026): <https://www.psychologytoday.com/us/blog/the-asymmetric-brain/202602/cognitive-offloading-using-ai-reduces-new-skill-formation>
- Anthropic — How AI assistance impacts coding skill formation: <https://www.anthropic.com/research/AI-assistance-coding-skills>
- The Augmentation Trap (arxiv 2604.03501): <https://arxiv.org/html/2604.03501>
- Protecting Human Cognition in the Age of AI (arxiv 2502.12447): <https://arxiv.org/html/2502.12447v1>
- Avoiding Skill Atrophy in the Age of AI (Addy Osmani): <https://addyo.substack.com/p/avoiding-skill-atrophy-in-the-age>
- Distributed Cognition Today and in an AI-Enhanced Future (PMC): <https://pmc.ncbi.nlm.nih.gov/articles/PMC9329671/>
- The Vibe-Check Protocol (arxiv 2601.02410): <https://arxiv.org/pdf/2601.02410>
- Vibe Coding in Practice — grey literature review (arxiv 2510.00328): <https://arxiv.org/html/2510.00328v1>
- Good Vibrations qualitative study (arxiv 2509.12491): <https://arxiv.org/html/2509.12491v1>
- ACM TechBrief on Vibe Coding: <https://www.acm.org/public-policy/techbriefs/techbrief-vibe-coding>
- Evidence Against Vibe Coding (SoftwareSeni): <https://www.softwareseni.com/the-evidence-against-vibe-coding-what-research-reveals-about-ai-code-quality/>
