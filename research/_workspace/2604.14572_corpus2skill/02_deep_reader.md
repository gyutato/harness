# Don't Retrieve, Navigate: Distilling Enterprise Knowledge into Navigable Agent Skills for QA and RAG

- 저자: Yiqun Sun, Pengfei Wei, Lawrence B. Hsieh
- 게재: arXiv 프리프린트, 2026-04-16 제출
- 식별자: arXiv:2604.14572
- 분류: cs.IR, cs.AI, cs.CL, cs.MA
- 라이선스: CC BY 4.0

> **독해 경로:** arXiv abs 페이지 → arxiv.org/html/2604.14572 (HTML 실험 버전) 본문 전체 정독. PDF는 접근 미필요 (HTML이 섹션 본문·표·수식 포함).

---

## 1. 한 줄 요약

"검색하지 말고 **탐색**하라" — 문서 코퍼스를 오프라인에서 **계층적 스킬 디렉토리(summary tree)** 로 미리 컴파일해두고, 런타임에는 LLM 에이전트가 파일 시스템을 browsing하듯 요약을 따라 내려가며 증거를 수집하게 만드는 RAG 대안 프레임워크 **Corpus2Skill** 을 제안. WixQA(Wix 고객지원 벤치) 단일 실험에서 Token F1·Factuality·Context Recall 등 모든 품질 지표에서 Dense/Hybrid/RAPTOR/Agentic-RAG 베이스라인 대비 최고 성능.

---

## 2. 문제 정의 (Motivation)

### 저자가 지목한 기존 RAG의 한계

표준 RAG는 LLM을 "검색 결과의 수동적 소비자(passive consumers of search results)" 로 다룬다. Top-k 패시지가 주어지지만 **코퍼스 조직 구조에 대한 가시성이 없다(lacks visibility into how the corpus is organized or what alternative evidence exists).** 저자의 핵심 주장:

- 복합 질문(multi-topic)은 여러 문서 클러스터에 산재한 정보를 결합해야 하는데, flat retrieval은 증거 간 관계를 모른다.
- 검색 함수 `f(q, D) → D_q`는 에이전트에게 **black box**. 에이전트가 "다른 곳은 더 없나?" 확인하거나 backtrack할 방법이 없다.
- 결과적으로 best evidence를 체계적으로 찾거나 scattered information을 조합하지 못한다는 "structural limitation".

### 제안하는 대안의 직관

문서 시스템을 **잘 정리된 파일 시스템**처럼 에이전트에게 노출시키면, 에이전트는:
1. 어디를 볼지 추론하고 (reason about where to look)
2. 막다른 경로에서 backtrack하고 (backtrack from unproductive paths)
3. 여러 branch의 증거를 결합할 수 있다 (combine evidence across branches).

> 본문 인용: "Rather than searching over a document corpus, agents **navigate** through it."

### 이 motivation이 성립하는 숨은 전제 (독자 관점)

- **전제 A:** LLM 에이전트가 요약 기반 라우팅에 충분히 능숙하다. (모델 능력 가정)
- **전제 B:** 코퍼스가 hierarchical decomposition이 가능한 구조(분리 가능한 topic cluster)를 가진다. WixQA 같은 엔터프라이즈 FAQ성 코퍼스에는 자연스럽지만, 개방형/논증형 코퍼스에는 불분명.
- **전제 C:** 요약 트리가 실제 질의 분포를 잘 커버한다 (컴파일 시점에 서비스 질의 패턴을 알지 못해도).

저자는 이 전제들을 명시하지 않는다. 특히 B는 한계 섹션의 "hard single-path clustering" 논의에서 부분적으로 드러난다 (후술).

---

## 3. 핵심 아이디어

### 세 가지 기여 (저자 명시)

1. **Corpus navigability** 를 RAG의 under-explored dimension으로 개념화.
2. **Compile-then-navigate** 프레임워크: embedding-based retrieval을 agent-driven file exploration으로 교체.
3. **WixQA에서 경험적 검증**: Dense, RAPTOR, Agentic RAG 대비 전 지표 개선.

### 핵심 문구들 (원문 발췌)

- "**Invest compilation-time computation making corpora navigable, rather than query-time computation making corpora searchable.**" — 이 한 문장이 논문의 설계 철학을 압축한다.
- 에이전트 스킬 4-튜플 formalism: `(C, π, T, R)` (Applicability Condition / Execution Policy / Termination / Reusable Interface). 이는 기존 agentic skills 문헌에서 차용한 것으로 명시.

---

## 4. 방법론 상세

### 4.1 개괄: 2-페이즈 구조

```
                  [ Compile Phase — offline ]
  Raw Docs D ──▶ (1) Load&Embed ──▶ (2) Iterative Clustering ──▶
                 (3) Labeling ──▶ (4) Skill Tree Materialization
                             │
                             ▼
                  Hierarchical Skill Forest  S = {s_1,...,s_k}
                  (SKILL.md / INDEX.md / documents.json)
                             │
                             ▼
                  [ Serve Phase — online ]
  Query q ──▶ Agent (code_execution tool) ──▶ 요약 따라 트리 하강 ──▶
              get_document(doc_id) ──▶ grounded answer a
```

각 skill `s_k`는 튜플 `(C, π, T, R)`:
- **C (Applicability Condition):** 스킬 요약. 에이전트가 읽고 해당 질의에 적합한지 판단.
- **π (Execution Policy):** 네비게이션 워크플로 (요약 하강 → 문서 fetch).
- **T (Termination):** 증거 충분성에 대한 에이전트의 자체 판단.
- **R (Reusable Interface):** 표준화된 파일 구조(SKILL.md, INDEX.md) + `get_document` 도구.

### 4.2 Compile Phase 4단계 상세

**(1) Document loading & embedding**
- 입력: `.md/.txt/.json` 표준 포맷 문서.
- 각 문서에 content-hash 기반 unique ID 할당 (결정성 확보).
- 최대 문자 길이로 truncation 후 Sentence-BERT류 임베딩 모델로 dense vector화.

**(2) Iterative hierarchical clustering (핵심 기술)**

하이퍼파라미터 2개로 통제:
- **p (branching ratio):** 클러스터당 목표 자식 수
- **K (max top-level clusters):** 루트 클러스터 최대 수

알고리즘:
```
embeddings = embed(documents)
while len(current_level) > K:
    n = len(current_level)
    clusters = KMeans(n_clusters=ceil(n/p))(L2_normalized(current_level))
    summaries = [LLM_summarize(representative_texts(c)) for c in clusters]
    current_level = embed(summaries)
root_clusters = current_level
```

- **K-Means on L2-normalized vectors** 로 cosine 유사도 기반 hard clustering.
- 각 클러스터 요약은 **LLM이 대표 멤버 텍스트를 읽어** 생성: topical coverage, question types, key terms 포함.
- 생성된 요약은 **다시 임베딩되어 다음 라운드의 입력**이 된다(bottom-up). 이는 RAPTOR의 재귀적 구조와 유사하지만 할당 방식이 다름.

**RAPTOR와의 차별 (저자 주장):**
| 측면 | RAPTOR | Corpus2Skill |
|---|---|---|
| 클러스터링 | UMAP + GMM + BIC (soft) | K-Means (hard) |
| 할당 | 문서가 여러 path에 속할 수 있음 | 문서는 **정확히 하나의 path**에만 |
| 저장 | flat vector index | 파일 시스템(browsable) |
| 검색 | collapsed-tree vector search | agent의 파일 browsing |

**(3) Labeling**

각 non-leaf 노드에 대해 LLM이 2-5단어짜리 filesystem-safe 라벨 생성. 사람 가독성과 에이전트의 topic routing을 동시에 지원. (예: `wix-commerce-monetization`, `wix-payments-ecosystem`)

**(4) Skill tree construction (materialization)**

파일시스템으로 구현:
- **Root cluster → top-level skill 디렉토리** 내부에 `SKILL.md` (클러스터 요약 + 자식 그룹 리스트)
- **Sub-cluster → 하위 디렉토리** 내부에 `INDEX.md` (추가 subgroup 또는 leaf 문서 ID + brief summary)
- **문서 본문은 documents.json 에 외부 저장**, 네비 파일에서는 ID로만 참조

"네비게이션 파일은 전형적으로 <2 KB로 작게 유지" — 요약과 ID만 담기 때문. 이 **separation of navigational vs. evidential information** 이 비용 절감의 핵심: INDEX.md 하나로 20개 문서를 "survey"하는 비용이 20개 문서 본문 읽는 비용보다 훨씬 적다.

### 4.3 핵심 수식: Navigation Complexity

저자가 제시한 복잡도 분석:

$$N \xrightarrow{\div p} \frac{N}{p} \xrightarrow{\div p} \frac{N}{p^2} \cdots \xrightarrow{\div p} \frac{N}{p^L} \approx 1$$

여기서 `L = ⌈log_p N⌉`. 깊이 `L`번 내려가면 후보 문서 수가 1 근처로 수렴.

**총 점검 요약 수:** `O(p · log_p N)` (각 레벨에서 `p`개 자식 요약을 1번씩 읽음).

**WixQA 적용:** N=6,221, p=10 → L=3 레벨, ≈30개 요약 점검으로 단일 문서 도달.

**직관적 해석 (내 풀이):** 선형 스캔은 `O(N)` 요약을 봐야 하지만, 트리 네비게이션은 각 레벨에서 한 번만 "어느 자식으로 갈지" 결정하므로 `O(log N)` 레벨 × `O(p)` 자식 = `O(p log_p N)`. `p`가 너무 크면 자식당 결정 비용(요약 읽기)이 커지고, `p`가 너무 작으면 트리가 깊어져 에이전트 turn이 늘어남 → **p가 품질/비용의 핵심 trade-off 축.** 이 직관은 ablation(4.1)에서 경험적으로 검증됨.

### 4.4 Serve Phase — 에이전트의 질의 처리

**인프라:** Anthropic **Skills API** 에 미리 컴파일된 스킬 디렉토리 업로드. **Progressive disclosure** 메커니즘 활용:
- **사전 로드:** skill **이름 + 설명**만 (~200 tokens) 에이전트 컨텍스트에 pre-load.
- **지연 로드:** 전체 파일 내용은 명시적 access 시에만 로드.

**API 제약 (§3.2 footnote 222 원문):**
> *"Anthropic's current Skills API imposes limits of 8 skills per request, 200 files and 30 MB per skill. These constraints informed our default parameters but are not fundamental to the framework design."*
>
> ⚠️ context-scout 조사 결과, footnote 222의 실제 출처 URL은 HTML 버전에 렌더링되지 않으며 platform.claude.com 현행 공식 문서에 **"200 files per skill" 수치가 부재**. "8 skills/request" 및 "30 MB/skill"은 공식 문서 확인됨. "200 files" 출처 미상은 재현성 측면에서 문제. 저자가 "not fundamental"이라 부연하면서도 하이퍼파라미터(K=7, p=10, 3-level, compact mode)를 전부 이 제약에 맞춰 튜닝한 것은 자기모순.

**도구 2종:**
- `code_execution` (view 명령 통한 SKILL.md/INDEX.md browsing)
- `get_document(doc_id)` (document store에서 full text 반환)

**전형적 워크플로 (2-3 turn):**
1. 에이전트가 사전 로드된 설명으로 관련 스킬 식별 → 해당 `SKILL.md` 읽기 → subgroup 구조 파악.
2. 관련 subgroup의 `INDEX.md` 진입 → 문서 ID + 제목 + 요약 리스트 확인.
3. `get_document(doc_id)` 호출 → 본문 획득 → grounded answer 합성.

**구조적 가시성이 활성화하는 두 가지 행동 (저자 강조):**
- **Targeted backtracking:** 유망하지 않은 branch 포기하고 다른 곳으로 이동.
- **Cross-branch synthesis:** 여러 subgroup 혹은 skill에서 증거를 결합.

---

## 5. 실험 설계

### 5.1 데이터셋 / 베이스라인 / 지표

**데이터셋: WixQA** (단일 벤치마크)
- 코퍼스: Wix 지식베이스 고객지원 아티클 **6,221개** (웹사이트 빌드, 이커머스, SEO, 마케팅, 플랫폼 기능)
- 평가셋: expert 작성 질문 **200개** + gold-standard answer + 참조 article ID

**베이스라인 5종 (3가지 retrieval paradigm):**
| 카테고리 | 시스템 | 설명 |
|---|---|---|
| Sparse | BM25 | 키워드 기반 |
| Dense | Qwen3-Embedding-0.6B + FAISS | 밀집 임베딩 |
| Hybrid | BM25 × Dense의 RRF (Reciprocal Rank Fusion) | |
| Hierarchical | RAPTOR | UMAP+GMM+BIC clustering, collapsed tree retrieval |
| Agentic | LLM 에이전트 + {BM25, Dense, Hybrid} 도구 (최대 10 rounds) | |

앞 3개는 single-shot retrieval (top-5 passages, 1 LLM call). RAPTOR는 **collapsed tree retrieval만 사용** (본문 §4.2 명시). Agentic은 확장 상호작용.

> **베이스라인 공정성 노트 (context-scout):** RAPTOR 원 논문은 "collapsed가 tree-traversal보다 consistently better"라고 보였고, 저자들이 그 설정을 채택한 것은 RAPTOR에 유리한 공정 선택. 다만 **RAPTOR의 tree-traversal 모드**가 Corpus2Skill의 "레벨별 LLM 의사결정" 구조와 더 유사한 운영 모드임에도 비교군에서 제외됨 → "계층 탐색 vs 계층 탐색"의 직접 비교는 부재, "계층 탐색 vs 계층 vector search"만 존재.

**평가 지표 6종 + 비용 2종:**
- Lexical: **Token F1, BLEU, ROUGE-1, ROUGE-2**
- LLM-judged (1-5 scale → 0-1 정규화): **Factuality, Context Recall**
- Cost: **input tokens / query, $/query**

### 5.2 구현 세부

**Compilation 기본 설정:**
- Branching ratio **p = 10**
- Max top-level clusters **K = 7**
- 결과 구조: **3-level hierarchy, 6 top-level skills, 665 navigation files, 13 MB document store**
- Compilation 시간: **6.5분** (32-CPU 서버)
- Compilation 비용: **$5–$10 LLM calls** (한계 섹션에서 언급)

**Serving 기본 설정:** Claude **Sonnet 4.6**, 최대 20 rounds.

---

## 6. 결과 해석

### 6.1 메인 결과 테이블 (WixQA 200 질의)

| Method | F1 | BLEU | R-1 | R-2 | Factuality | CtxR | InTokens | $/query |
|---|---|---|---|---|---|---|---|---|
| BM25 | 0.342 | 0.060 | 0.364 | 0.119 | 0.470 | 0.386 | 708 | 0.007 |
| Dense | 0.363 | 0.074 | 0.382 | 0.141 | 0.536 | 0.450 | 656 | 0.008 |
| Hybrid | 0.360 | 0.071 | 0.380 | 0.137 | 0.524 | 0.410 | 698 | 0.008 |
| RAPTOR | 0.389 | 0.109 | 0.406 | 0.189 | 0.675 | 0.616 | 1,807 | 0.012 |
| Agentic | 0.388 | 0.099 | 0.404 | 0.186 | 0.724 | 0.481 | 25,807 | 0.098 |
| **Corpus2Skill** | **0.460** | **0.137** | **0.476** | **0.231** | **0.729** | **0.652** | 53,487 | 0.172 |

**주요 관찰:**
1. **전 지표 최고.** F1 0.460 → Agentic 대비 **+19% rel**, Dense 대비 **+27% rel**.
2. **Factuality 0.729** (vs Agentic 0.724, RAPTOR 0.675) — 답변의 사실성에서도 우위.
3. **Context Recall 0.652** (vs RAPTOR 0.616, Agentic 0.481) — 관련 증거 회수율에서 Agentic보다 **+36% rel** 높다. 저자는 이를 "navigation이 더 많은 관련 문서를 surface한다"는 증거로 해석.
4. **Hierarchical advantage 확인:** RAPTOR와 Corpus2Skill 모두 flat retrieval(BM25/Dense/Hybrid)을 크게 상회. 계층화 자체가 quality에 기여.

### 6.2 비용 구조

- Corpus2Skill: **$0.172/query**, input tokens **53,487** — Agentic의 **1.75×**, RAPTOR의 **14×**.
- 병목은 **input tokens** — Skills API가 매 호출마다 네비게이션 파일 내용을 포함하기 때문.
- 흥미로운 반전: Corpus2Skill의 **output tokens는 752** (Agentic의 1,391의 **약 절반**). 저자 해석: "올바른 문서로 navigate하면 더 targeted answer 생성 → 출력 짧아진다."

저자 결론: "high-value queries(복잡한 지원 질의, 컴플라이언스-중요 조회)에 적합. high-volume low-stakes는 single-shot retrieval이나 RAPTOR가 cost-effective."

### 6.3 Ablation Studies

#### (a) 클러스터 구조 (p, K)

| Variant | F1 | Factuality | CtxR | $/q |
|---|---|---|---|---|
| Narrow (p=5) | **0.461** | **0.736** | **0.674** | 0.186 |
| Default (p=10) | 0.460 | 0.729 | 0.652 | 0.172 |
| Wide (p=20) | 0.361 | 0.410 | 0.355 | 0.242 |

- Narrow(p=5): **4-level, 3 top-level skills.** 모든 품질 지표 소폭 향상, 비용 +8%.
- Wide(p=20): **2 extremely broad skills (≈3,000 docs each).** F1 -21%, **Factuality -44%**, 비용 +41%. 저자 해석: "SKILL.md 요약이 너무 generic해져서 effective routing 불가."
- 저자가 **_compact mode_** 도 언급: 깊은 계층이나 API 파일 제한에 걸릴 때, near-leaf INDEX.md를 부모로 merge해 파일 수 최대 80% 감축.

**이것이 시사하는 바 (내 해석):** `p`는 단순 튜닝 파라미터가 아니라 **"요약 granularity가 topic discrimination에 충분한가"를 결정**. Wide는 요약이 너무 abstract해져서 에이전트가 올바른 branch 선택 불가 → **raw cluster 품질이 아니라 요약 품질이 병목**.

#### (b) 에이전트 탐색 예산 (max rounds)

| Max rounds | F1 | Factuality | CtxR | $/q |
|---|---|---|---|---|
| 5 | 0.453 | 0.721 | 0.636 | 0.170 |
| 10 | **0.461** | **0.748** | **0.667** | 0.172 |
| 20 | 0.460 | 0.729 | 0.652 | 0.172 |

- **5-round budget도 F1 0.453 (full의 -1.5%).** 저자 해석: "계층 구조가 충분히 잘 조직되어서 extended exploration이 거의 benefit 없다."
- 비용 차이 negligible.

**이것이 시사하는 바 (내 해석):** 메인 결과의 53K input tokens는 **exploration 때문이 아니라 Skills API의 navigation file pre-inclusion overhead**. 실제로는 에이전트가 몇 턴 안에 답을 찾는다. 이는 Corpus2Skill의 **turn-efficiency**가 실제로 높다는 간접 증거이면서 동시에 **"비용의 주범은 API 스펙이지 접근법 자체가 아니다"** 라는 변호가 된다.

#### (c) 서빙 LLM 선택

| Model | F1 | Factuality | CtxR | $/q |
|---|---|---|---|---|
| Claude Sonnet 4.6 | **0.460** | **0.729** | 0.652 | 0.172 |
| Claude Haiku 4.5 | 0.423 | 0.645 | **0.705** | **0.088** |

- Haiku가 절반 비용에 F1 -8%, Factuality -12%.
- **의외 관찰:** Haiku의 **Context Recall이 더 높다 (0.705 vs 0.652).** 저자 해석: "cheaper models navigate more thoroughly, retrieving more relevant documents despite slightly less polished synthesis."
- 저자 주장: "**hierarchy quality, not navigator sophistication, drives retrieval performance**" — 모델 선택에 대한 robustness.

**불명확 지점:** Haiku의 over-navigation이 "더 신중한 탐색"인지 "더 주저하며 돌아다니는 비효율"인지는 데이터만으로 구분 불가. 비용은 output tokens에도 민감할 수 있으나 세부 분해 없음.

### 6.4 Case Studies (정성 trace)

저자가 두 가지 항행 패턴을 제시:

**Trace 1 — Direct descent (4 step):**
> Q: "I need to switch my business type from sole prop to LLC in order to use an EIN."

```
SKILL.md (wix-commerce-monetization, 10 subgroups)
  → INDEX.md (wix-payments-ecosystem, 8 leaf groups)
    → INDEX.md (group-05, account management)
      → get_document("d56cc797742e"): "Changing Your Wix Payments Account Type"
A: "You cannot change your account type directly. Contact Wix Customer Care."
```

**Trace 2 — Cross-branch exploration (7 step):**
> Q: "I want to change the currency for my course."

에이전트가 **SKILL.md에서 복수 subgroup을 relevant로 식별**(`wix-online-programs` + `wix-billing-documents`), 먼저 전자에서 "currency는 site-level setting"이라는 단서를 얻은 뒤, 후자의 currency management group으로 이동해 "Settings > Language & Region > Currency" 경로를 확인. 두 문서의 증거를 결합해 최종 답변 생성.

이는 저자가 강조하는 **cross-branch synthesis** 행동의 실제 instance. 수치 지표 너머 "왜 잘 되는가"의 정성적 근거로 쓰이지만, cherry-picked 예시일 가능성(선별 편향)은 저자가 언급하지 않음.

---

## 7. 한계

### 7.1 저자 명시 한계

1. **Cost.** Per-query 비용이 input tokens에 의해 지배됨 (Skills API가 매 call마다 navigation file content 포함). high-volume/low-stakes 용도엔 부적합. **Prompt caching** 이 잠재 완화책으로 언급.

2. **API constraints.** Anthropic Skills API 제약: **8 skills/request, 200 files/skill, 30 MB/skill.**
   - 8-skill 제한 → top-level cluster 수 상한.
   - 200-file 제한 → 트리 깊이 상한 (compact mode 필요 시점).
   - WixQA 코퍼스는 comfortably fit (6 skills, ≤133 files each). 더 큰 코퍼스는 compact merging 혹은 multi-level partitioning 필요.

3. **Hard single-path clustering.** 각 문서를 정확히 하나의 path에만 할당 → filesystem materialization과 deterministic navigation의 enabler. 그러나:
   - Multi-topic 문서(예: billing **AND** subscriptions)는 하나의 branch에만 등장 가능.
   - 다른 branch로 라우팅된 질의에서 bottleneck 발생.
   - **저자 실패 분석: top-level routing errors가 실패의 대부분.**
   - 대안(soft/multi-parent assignment)은 content 중복과 복잡도 trade-off.

4. **Compilation.** $5-$10 + 6.5분 one-time cost (엔터프라이즈엔 negligible). 그러나 **incremental update 미지원** — 문서 추가 시 **재컴파일 필요**.

### 7.2 독자 관점에서 발견한 추가 한계/암묵적 가정

- **단일 벤치마크(WixQA) 평가.** 저자가 conclusion에서 "future work: evaluation across additional domains"로 인정하지만, 현 시점 근거는 한 도메인(고객지원 FAQ)뿐. 다음 상황 일반화는 검증 안 됨: (a) 논증/창작성 질의, (b) 수치/표 중심 보고서 코퍼스, (c) 매우 중첩된 법률/의료 텍스트.

- **Skill API 벤더 종속.** 논문의 Serve Phase 전체가 Anthropic-specific 기능(progressive disclosure, skills format)에 의존. 다른 LLM provider에 이식성은 "원리상 가능"에 그침. 재현성 측면에서 중요한 결함 (context-scout 조사 요청 항목).

- **"Agentic" 베이스라인이 weak한가?** Agentic 베이스라인이 **BM25/Dense/Hybrid 도구만 받고 hierarchy 정보는 못 받음** — 즉 "에이전트가 hierarchical skill 없이 iterative retrieval만 하면 어떠냐"의 비교. 이는 "hierarchical indexing + agent"(유사 시스템)와의 진짜 비교가 아니다. **BookRAG / NaviRAG / HCAG 같은 agent-planning + hierarchical indexing 시스템이 실험 대상에서 빠짐.**

- **Factuality/Context Recall이 LLM-judge 기반.** 1-5 scale의 LLM 심판 사용. 심판 모델 정체·프롬프트·편향에 대한 세부 기술이 본문에 없음(HTML 버전에서 안 보임). judge model이 Claude인 경우 self-preference 우려.

- **클러스터 품질과 요약 품질의 coupling.** 요약을 re-embed해 다음 레벨 clustering에 쓰는 iterative 방식은, **저품질 요약이 상위 구조 전체를 오염**시킬 수 있다. 저자는 이 실패 모드에 대한 robustness 실험을 제공하지 않음.

- **Claim "hierarchy quality > navigator sophistication" 의 약한 증거.** Sonnet vs Haiku 한 쌍만으로 일반화. 훨씬 약한 모델(예: GPT-4o-mini, Llama 70B)에서도 성립하는지 미확인.

- **트리 깊이 vs 라우팅 오류의 비선형 관계 미탐구.** 저자는 wide(p=20)에서 폭망, narrow(p=5)에서 약간 우수라고 보여주지만, 정말 깊은 트리(p=3 등)에서는 top-level routing이 여러 번 곱해져 실패율이 누적될 수 있음. 이 failure mode는 탐구 안 됨.

- **NaviRAG concurrent work acknowledgment 누락 (context-scout 확인).** NaviRAG (Dai et al., arXiv:2604.12766, 2026-04-14 제출) 은 본 논문(2026-04-16 제출)보다 **딱 2일 먼저 공개**. motivation("passive retrieval → active knowledge navigation")이 본 논문과 사실상 동일. 본문 §6 한 문장만 언급:
  > *"Meanwhile, NaviRAG (Dai et al., 2026) builds a hierarchical view of documents and uses an LLM agent to actively navigate it, detecting information gaps and retrieving at the appropriate granularity."*
  - 본문 전체에서 "concurrent", "concurrently", "simultaneous", "independent work" 같은 동시기 인정 표현 **전무**. 학계 관례상 2일 차이는 concurrent work로 acknowledge 대상.
  - 본 논문의 차별점 주장은 "파일시스템 materialization vs retrieval/search infrastructure 유지"로 카테고리 선긋기만 할 뿐, NaviRAG의 실제 구현 세부(hierarchical view 구조, 정보 갭 탐지 로직)와의 정량·정성 비교 없음.
  - NaviRAG 역시 경험적 baseline에서 제외 → 본 논문이 거의 동일한 motivation의 concurrent work와의 직접 비교를 회피했다는 비판 가능.

- **핵심 선행연구 LATTICE 인용 누락 (context-scout 확인, §2.1 참조).** Gupta, Chang, Bui, **Cho-Jui Hsieh**, Dhillon, "LATTICE: LLM-guided Hierarchical Retrieval" (arXiv:2510.13217, 2025-10, UT Austin/Google).
  - LATTICE의 아키텍처: **corpus를 semantic tree로 offline 구조화 (bottom-up agglomerative / top-down divisive + multi-level summary) → online에서 LLM이 tree를 traverse → log 복잡도.** 본 논문 §Method의 compile-then-navigate 골격과 거의 동일.
  - 본 논문 Related Work · baseline · references 전부에서 누락. "LATTICE", "2510.13217", 해당 저자명 어떤 토큰으로도 매칭 없음.
  - 시점 비교: 본 논문 2026-04-16, LATTICE 2025-10 → **6개월 선행**. 본 논문이 NaviRAG(2026)·A-RAG(2026-02)·HCAG(2026) 등 **더 최근 혹은 동시기** preprint는 인용하고 있어 "인지 불가" 변호 어려움.
  - 저자가 제공하지 않은 (하지만 변호 가능한) 차별점 후보: (a) enterprise QA 특화, (b) Anthropic Agent Skills 생태계와의 결합(SKILL.md/INDEX.md 포맷), (c) 파일시스템 은유 기반 progressive disclosure. 그러나 저자가 직접 대조하지 않았으므로 "독립 발견"/"근친연구 누락" 판단 모호.
  - 동명이인 주의: 본 논문 공저자 **Lawrence B. Hsieh ≠ LATTICE 저자 Cho-Jui Hsieh** — 성씨 충돌이 실제 관계를 암시하지 않음 (context-scout 확인).
  - 이는 **novelty 평가에서 중대한 결격 사유**로, insight-analyst 분석 시 핵심 논점이 될 것.

---

## 8. 불명확 지점 (확신 없는 부분)

1. **Skills API "progressive disclosure" 정확한 스펙.** 본문은 "skill names/descriptions pre-load, full content on access"라고만 말함. 각 call에 navigation file 내용이 왜 반복 포함되는지(prompt caching이 왜 켜져있지 않은지) 세부가 부족. "Skills API includes navigation file content in each call"이라는 서술이 정확한가는 context-scout의 조사 필요. 추가 확증: **§3.2 footnote 222의 "200 files per skill"은 현행 공식 문서에 부재** (context-scout 확증). "8 skills/request"와 "30 MB/skill"은 공식 확인되지만 200 files 수치는 출처 미상. 설계 전체가 이 수치에 묶여 있으므로 재현성 critical point.

2. **Compact mode 의 실제 성능 영향.** 본문에서 "파일 수 80% 감축"만 언급, 품질에 대한 실측 없음. Default와 동등 유지 주장이 검증되지 않음.

3. **Compilation 비용 $5-$10의 내역.** 6,221 문서 + 요약 생성 → 몇 번의 LLM call, 어떤 모델, 어떤 프롬프트 길이인지 본문 상에서 불명. 더 큰 코퍼스로의 확장 시 선형 가정이 유효한가도 검증 필요.

4. **"Context Recall" 지표의 정확한 정의.** LLM-judge라는 것만 명시. "retrieved context가 gold evidence를 얼마나 포함하는가"로 해석했지만, 저자의 실제 프롬프트/정의는 세부 미공개 (본문 HTML 기준).

5. **Agentic baseline의 도구 호출 전략.** "최대 10 rounds, BM25/Dense/Hybrid 접근 허용"이라는 것만 알고, system prompt·planning 구조가 Corpus2Skill만큼 최적화되었는지 불분명. 비교 공정성 의심지점.

6. **"코퍼스 n=6,221 → 30 summaries로 docs 도달" 계산의 실제 분포.** Trace 1은 4 step, Trace 2는 7 step이었음. 200 질의 전체에 대한 평균 turn 수·평균 파일 수 등 통계 공개 안 됨 (case study 2개만 제시).

---

## 참고: 본문에서 발견한 외부 참조 중 context-scout에게 이미 조사 요청한 항목들

- RAPTOR (비교군 #1)
- WixQA (유일 벤치)
- Anthropic Skills API (인프라 종속)
- Scatter/Gather (cluster hypothesis 선조)
- GraphRAG / StructRAG / BookRAG / NaviRAG / HCAG (구조-aware RAG 경쟁군, 특히 BookRAG)
- Voyager / SkillX (agent skills 선행)

---

**산출물 위치:** `/Users/user/harness/research/_workspace/2604.14572_corpus2skill/02_deep_reader.md`
