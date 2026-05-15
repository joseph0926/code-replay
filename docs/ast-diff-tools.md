# AST diff 도구 비교: target 검증과 학습 단위 추출 백엔드

리서치 일자: 2026-05-15
스코프: 사용자나 외부 연동이 전달한 `target`을 **로컬 diff와 대조해 검증**하고, target 내부에서 **학습 단위(어떤 함수/블록이 의미적으로 바뀌었는가)**를 추출할 백엔드를 결정한다. text diff(git diff)만으로는 "공백/리포맷/리팩터링"과 "실제 의미 변경"이 구분되지 않아 학습 포인트 추출이 부정확하다.

## 한 줄 결론

- **계층적 선택**: tree-sitter(파서) → difftastic(CLI 표시) → diffsitter(라이브러리 활용) 또는 **GumTree**(엄밀한 edit script가 필요할 때).
- **MVP 1차 추천: tree-sitter + difftastic CLI**. 30+ 언어 지원, OSS 성숙, 의존성 단순. **edit script가 필요한 시점에 GumTree로 격상**.
- **상용 SemanticDiff는 정확도 우위지만 라이선스/통합 비용**. MVP 단계에서는 후순위.
- **ast-grep은 diff 도구가 아님** — pattern search/rewrite. code-replay에서는 **"이런 패턴이 들어왔는가"** 판정에 보조로 쓸 수 있음(예: "이 target에서 `useMemo` 새로 추가됨" 감지).
- **agent-assisted 여부는 AST diff가 판정하지 않는다.** Code Replay는 그 속성을 추론·탐지하지 않으며, 사용자나 외부 연동이 `target_added` event의 `provenance`로 명시적으로 전달한다(§5).

## 1. 도구 카테고리

### 1.1 파서 인프라

- **tree-sitter** — incremental parser library. Rust 코어, JS 문법 정의, C/WASM 출력 파서. 30+ 언어. VSCode/Neovim/Emacs 표준. **모든 현대 구조 diff의 사실상 기반.**

### 1.2 CLI 구조 diff (사용자 표시 / 사후 분석)

- **difftastic** — tree-sitter 기반 구조 diff CLI. Wilfred Hughes 단독 OSS. 30+ 언어. 출력은 사람용 highlighted diff. **edit script API는 약함 — 프로그램적 활용보다 사람이 보는 용도**.
- **diffsitter** — tree-sitter 파싱 후 leaf 노드에 LCS 적용. Rust. **라이브러리 형태로 다른 도구에 통합하기 difftastic보다 유리**할 수 있음. 활성도는 difftastic보다 낮음.

### 1.3 학술 grade AST diff (edit script)

- **GumTree** (Falleri et al.) — 표준 학술 AST 차분. 출력이 **{insert, remove, update, move} edit operation 시퀀스**. greedy top-down isomorphic subtree matching → bottom-up container matching. **MSR/ICSE 등에서 base tool**. 단점: 큰 AST에서 비용이 크다(2024 ICSE 후속 연구가 cheap heuristic으로 scalable 개선).
- **GumTree-Spoon** — Spoon 기반 Java 특화 GumTree. Java 한정.
- **TOSEM 2024 — "Novel Refactoring and Semantic Aware AST Differencing Tool"** — refactoring/semantic-aware로 GumTree 정확도 개선. **benchmark 동시 발표**, 정확도 비교의 기준선.
- **arxiv 2011.10268** — AST differencing 하이퍼파라미터 최적화. mapping 정확도가 edit script 품질에 결정적임을 정량화.

### 1.4 구조 search / rewrite (diff 인접)

- **ast-grep** — Rust, tree-sitter 기반. 패턴 매칭/lint/rewrite CLI. **diff가 아니라 search**지만, "이 PR에 X 패턴이 새로 들어왔는가" 같은 보조 판정에 유용.
- **Comby** — 구조 패턴 기반 search/rewrite. **AST 기반이 아님**(파서 abstraction). Python/Haskell 같은 indentation-sensitive 언어에 약함. code-replay에는 부적합.

### 1.5 상용

- **SemanticDiff** — VSCode/GitHub 통합. **difftastic보다 더 깊은 의미 인식**(Python 문자열 concat no-op, 괄호 추가 무시 등). 라이선스 비용 / 통합 의존성 추가.

### 1.6 데이터 구조 diff (참고)

- **graphtage** — JSON/YAML/XML 등 generic structured data diff. 코드 도메인은 X. 설정 파일 변경 추출에 보조로 유용.
- **json-diff** — JSON 한정.

## 2. 도구 비교

| 도구 | 언어 지원 | 출력 형식 | 라이브러리 활용 | 성숙도 | 라이선스 | code-replay 적합성 |
|---|---|---|---|---|---|---|
| tree-sitter | 30+ | 파싱 트리 | 직접 라이브러리 | 매우 높음 | MIT | **파서 기반, 항상 사용** |
| difftastic | 30+ | 사람용 colored diff | 약함 | 높음 | MIT | **MVP CLI 1차 후보** |
| diffsitter | 다수 | LCS 기반 diff | 라이브러리 | 중 | MIT | difftastic 대안 후보 |
| GumTree | Java/JS/Python 등 | **edit script (insert/remove/update/move)** | API 가능 | 학계 표준 | LGPL | **edit script 필요 시 격상** |
| GumTree-Spoon | Java only | edit script | API 가능 | 높음(Java) | LGPL | Java 단독 시 |
| ast-grep | 다수 | 패턴 매치 | API/CLI | 높음 | MIT | **보조** (패턴 감지) |
| SemanticDiff | 다수 | 의미 diff | VSCode/GitHub | 상용 | 유료 | MVP 후순위 |
| Comby | 다수 | 패턴 rewrite | CLI | 중 | Apache | 부적합 |

## 3. code-replay 요구사항 매핑

| 요구사항 | 필요 도구 |
|---|---|
| target이 가리키는 file/line range가 실제 diff와 맞는지 검증 | tree-sitter (AST) → difftastic 또는 diffsitter (구조 diff) |
| 변경된 함수/블록 단위 식별 | tree-sitter (AST) → difftastic 또는 diffsitter (구조 diff) |
| **edit operation 분류** (insert/remove/update/move) → 학습 포인트 우선순위화 | **GumTree** (이게 핵심 차별 — text diff와 difftastic은 move를 잘 못 구분) |
| 공백/리포맷 변경 무시 | difftastic, GumTree 모두 OK (tree-based) |
| 새 패턴 도입 감지 ("처음으로 useMemo 들어옴") | ast-grep 보조 |
| target의 agent-assisted 여부 판정 | **AST diff가 하지 않는다.** 사용자나 외부 연동이 명시적 provenance로 입력(§5) |
| 학습 단위 추출 | tree-sitter 노드 → AST-T5 / AIED 2022 패턴(`docs/diff-to-curriculum.md` §3, §5) |

## 4. 추천 스택 (MVP 단계)

**1차 (MVP P0/P1):**
- tree-sitter — 파서 기반 (Node.js binding 또는 Rust)
- difftastic — CLI 출력 + 사람이 보는 용도
- ast-grep — 패턴 감지 보조

**2차 (학습 가중치 정밀화 단계):**
- GumTree — edit script가 필요해지면 추가. 출력의 move/update/insert/remove 분류가 "내가 새로 만든 부분 vs 옮긴 부분" 판정에 결정적.

**보류:**
- SemanticDiff — 라이선스 / 통합 비용 / API surface 폐쇄도 검토 필요. MVP 후 재평가.
- diffsitter — difftastic이 라이브러리 통합에 한계 보일 때만 격상.

## 5. agent-assisted 판정 — AST diff 외부 입력

AST diff 도구는 **누가 썼는지** 모른다. Code Replay 역시 이 속성을 추론·탐지하지 않는다. `agent-assisted`는 `target_added` event의 `provenance` 입력으로 받는 속성이다(`docs/product-direction.md`, `docs/mvp-spec.md`).

외부 입력으로 들어올 수 있는 보조 신호:
- **사용자 명시** — CLI `target add` 또는 외부 도구가 `target_added` event에 `provenance.label: "agent-assisted"`로 표시. 가장 확실한 신호.
- **commit trailer** — `Co-Authored-By:` trailer, `Code-Replay-Agent-Assisted: true` 등. 사용자나 외부 도구가 이 trailer를 읽어 target event로 변환할 때 보조 신호로 쓴다.
- **에디터/에이전트 활동 로그** — Cursor, Claude Code, Codex 등의 session log. P0에서는 자동 파싱하지 않는다. 외부 도구가 사용자 의사로 변환해 target event를 만들 때만 쓴다.
- **`git blame`** — 누가 commit했는지 단순 정보. 단일 author 여부 확인 정도.
- **commit 시간 분포** — 휴리스틱이 약하고 Code Replay에서 직접 사용하지 않는다.

→ P0에서는 **사용자가 직접 target을 추가하거나 외부 연동이 target event를 만들어 전달**하는 흐름만 표준이다. 에디터 로그 자동 파싱과 시간 휴리스틱은 비-목표. agent-assisted 여부를 LLM이나 코드 스타일 classifier로 추론하는 것 역시 사용하지 않는다.

## 6. 위험 / 미해결

- **GumTree LGPL**: code-replay이 어떤 라이선스로 배포될지에 따라 GumTree를 직접 임포트할지 CLI 호출할지 결정 필요. CLI로 격리하면 LGPL 전염 회피 가능.
- **GumTree 비용**: 큰 모노레포 PR에서 비용 이슈. 2024 ICSE 후속이 cheap heuristic 추가했지만 실측 필요.
- **tree-sitter 문법 버전**: 같은 언어도 grammar 버전에 따라 결과 다름. 재현성 위해 grammar 버전 pin 필요.
- **JSX/TSX 같은 hybrid**: tree-sitter 문법 품질이 언어마다 편차 큼. JSX/TSX, Svelte, Astro 등 합성 언어는 별도 검증.
- **agent-assisted 입력의 정확도는 외부 책임**: Code Replay는 provenance를 추론하지 않으므로, 외부 도구나 사용자가 잘못 표시하면 잘못된 라벨이 그대로 흐른다. P0는 사용자 명시 표시(`target_added` / `target_superseded`)로 정정하는 흐름만 둔다.

## Sources

### tree-sitter
- tree-sitter repo: <https://github.com/tree-sitter/tree-sitter>
- tree-sitter 공식 사이트: <https://tree-sitter.github.io/>
- tree-sitter Wikipedia: <https://en.wikipedia.org/wiki/Tree-sitter_(parser_generator)>
- Strumenta — Incremental parsing using tree-sitter: <https://tomassetti.me/incremental-parsing-using-tree-sitter/>
- tree-sitter 도구 모음 (Shae Erisson): <https://www.scannedinavian.com/tools-built-on-tree-sitters-concrete-syntax-trees.html>

### difftastic
- difftastic repo: <https://github.com/Wilfred/difftastic>
- difftastic 공식 사이트: <https://difftastic.wilfred.me.uk/>
- Structural Diffs wiki: <https://github.com/Wilfred/difftastic/wiki/Structural-Diffs>
- SemanticDiff vs Difftastic: <https://semanticdiff.com/blog/semanticdiff-vs-difftastic/>
- HN difftastic 토론: <https://news.ycombinator.com/item?id=39778412>
- difftastic alternatives (Slant 2025): <https://www.slant.co/options/45362/alternatives/~difftastic-alternatives>

### diffsitter
- diffsitter repo: <https://github.com/afnanenayet/diffsitter>

### GumTree
- GumTree repo: <https://github.com/GumTreeDiff/gumtree>
- GumTree-Spoon (Java): <https://github.com/SpoonLabs/gumtree-spoon-ast-diff>
- Falleri et al. 원논문 (HAL): <https://hal.science/hal-01054552/document>
- GumTree slides (Virginia Tech CS6704): <https://courses.cs.vt.edu/cs6704/spring17/slides_by_students/CS6704_gumtree_Kijin_AN_Feb15.pdf>
- AST differencing 도구 개관 (Monperrus): <https://www.monperrus.net/martin/tree-differencing>
- Hyperparameter optimization for AST differencing (arxiv 2011.10268): <https://arxiv.org/pdf/2011.10268>
- Differential testing for AST tools (Xia et al., ICSE 2021): <https://xin-xia.github.io/publication/icse212.pdf>
- Refactoring and Semantic Aware AST Diff (TOSEM 2024): <https://users.encs.concordia.ca/~nikolaos/publications/TOSEM_2024.pdf>
- Refactoring and Semantic Aware AST Diff ACM 페이지: <https://dl.acm.org/doi/10.1145/3696002>
- Fine-grained, accurate, scalable source differencing (ICSE 2024): <https://dl.acm.org/doi/10.1145/3597503.3639148>

### ast-grep
- ast-grep 공식: <https://ast-grep.github.io/>
- ast-grep repo: <https://github.com/ast-grep/ast-grep>
- ast-grep 도구 비교: <https://ast-grep.github.io/advanced/tool-comparison.html>
- Structural vs Textual: Why AI Agents Need AST Tools (2026-02): <https://themiloway.github.io/milo-blog/agentic-coding/typescript/2026/02/03/structural-vs-textual-code-manipulation.html>

### 상용 / 인접
- SemanticDiff: <https://semanticdiff.com/blog/semanticdiff-vs-difftastic/>
