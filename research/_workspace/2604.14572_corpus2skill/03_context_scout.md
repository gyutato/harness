# 외부 맥락: Don't Retrieve, Navigate — Distilling Enterprise Knowledge into Navigable Agent Skills for QA and RAG

조사 시점: 2026-04-21
대상 논문: arXiv:2604.14572 (Yiqun Sun, Pengfei Wei, Lawrence B. Hsieh, 2026-04-16, cs.IR / cs.AI / cs.CL / cs.MA)

---

## 1. 분야 배경 (이 논문 이해에 필요한 사전 지식)

이 논문은 세 개의 흐름이 **수렴하는 지점**에 위치한다. 각 흐름의 배경을 선제적으로 이해해야 본문 해석이 쉬워진다.

### 1.1 RAG의 진화 — passive → agentic
- **Naive/Dense RAG:** 쿼리를 임베딩하여 top-k 청크를 뽑아 컨텍스트에 주입. LLM은 검색 결과의 **수동적 소비자**.
- **한계:** "모델은 corpus가 어떻게 구성됐는지 보지 못하고, 아직 못 본 부분이 무엇인지 모른다" → backtrack·조합 불가 (본 논문 motivation 인용).
- **Agentic RAG:** LLM에게 `search(query)` 도구를 주고 반복 호출하게 함. 그러나 "지도 없이 검색어를 추측"해야 하며 corpus 구조를 훑을 수 없다는 비판이 여전히 존재 ([emergentmind](https://www.emergentmind.com/topics/hierarchical-agentic-retrieval-augmented-generation-rag)).
- 실무적으로 agentic loop는 **지연·토큰비용**이 크다: 3~4회 루프에 10초 이상 소요, 토큰비 수배 증가 ([TechAhead](https://www.techaheadcorp.com/blog/agentic-rag-when-llms-decide-what-and-how-to-retrieve/), [Progress RAG docs](https://docs.rag.progress.cloud/docs/rag/advanced/consumption/)).
- "Lost-in-the-middle" 현상: 긴 컨텍스트 중간부 정보 주의도 저하 — 대량 검색 결과를 그냥 쌓아두면 성능 역전 ([관련 논문](https://arxiv.org/html/2510.10276)).

### 1.2 계층적 요약 기반 검색 — RAPTOR / GraphRAG 계열
본 논문이 **비교 기준선(baseline)으로 직접 거론**하는 계통.
- **RAPTOR** (Sarthi et al., ICLR 2024, [arXiv:2401.18059](https://arxiv.org/abs/2401.18059)): 청크를 재귀적으로 클러스터링·요약해 트리를 만들고, 쿼리 시 여러 레벨에서 검색. GMM soft clustering + UMAP 차원축소.
- **GraphRAG** (Microsoft Research, 2024, [공식 GitHub Pages](https://microsoft.github.io/graphrag/)): 엔티티·관계 그래프 → Leiden 계층 클러스터링 → 커뮤니티별 요약. Global/Local/DRIFT search 모드. Community summary 기반 글로벌 질의에서 naive RAG 대비 70~80% win rate.
- 공통 발상: **요약을 leaf가 아닌 내부 노드에 두어 추상도 다른 뷰를 동시 제공**. 본 논문은 이 발상을 유지하되 "트리를 LLM 에이전트가 직접 파일시스템처럼 탐색"하는 지점이 차별점.

### 1.3 Agent Skills 생태계 — SKILL.md 표준과 정적 파일 기반 컨텍스트
2025년 하반기~2026년 상반기 산업계에서 급부상.
- **Anthropic Agent Skills** ([공식](https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills), [docs](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview)): SKILL.md (YAML frontmatter + Markdown) 파일을 모듈로, 프론트매터만 선 로드 후 필요 시 본문 읽는 **progressive context loading** 철학.
- **서베이 논문** "Agent Skills for Large Language Models" ([arXiv:2602.12430v3](https://arxiv.org/abs/2602.12430)): architecture / acquisition / deployment / security 4축 정리. 커뮤니티 스킬 중 **26.1%에 취약점 존재**라는 경험적 분석 포함.
- HN 시리즈 토론 ([Skill.md open standard 46723183](https://news.ycombinator.com/item?id=46723183), [Agent Skills 46871173](https://news.ycombinator.com/item?id=46871173), [Agent Skills for Context Engineering 46351787](https://news.ycombinator.com/item?id=46351787)): 표준의 잦은 변경·파편화 비판과 실무 유용성 옹호가 혼재.
- 주요 논점: "skill과 CLAUDE.md/일반 문서의 본질적 차이는 **컨텍스트 내 위치·구조**뿐"이라는 회의론(vidarh) vs "스킬 수십 개를 CLAUDE.md에 다 넣으면 컨텍스트가 남지 않는다"는 실용론(smithkl42).
- 본 논문은 이 생태계의 문법(SKILL.md + progressive loading)을 **RAG로 포팅**하는 시도로 읽을 수 있다.

### 1.4 엔터프라이즈 QA/지원 벤치마크 성숙
- **WixQA** ([arXiv:2505.08643](https://arxiv.org/abs/2505.08643), [HF dataset](https://huggingface.co/datasets/Wix/WixQA), [Wix blog](https://www.wix.engineering/post/advancing-enterprise-ai-how-wix-is-democratizing-rag-evaluation)): Wix 실제 고객 지원 대화 기반. 3개 서브셋(ExpertWritten 200, Simulated 200, Synthetic 6,221) + 6,221 KB 문서 전체 공개(MIT).
- 본 논문이 **주 평가 벤치마크**로 채택. "enterprise customer-support"라는 이름이 아닌 **실재 기업 corpus를 대상 공표한 사실상 표준에 가까움**.

---

## 2. 핵심 선행 연구

### 2.1 LATTICE — LLM-guided Hierarchical Retrieval (Gupta et al., 2025-10, [arXiv:2510.13217](https://arxiv.org/abs/2510.13217))
**가장 가까운 경쟁 연구 — 반드시 대조 필요.** 저자는 Nilesh Gupta, Wei-Cheng Chang, Ngot Bui, **Cho-Jui Hsieh**, Inderjit S. Dhillon (UT Austin/Google).
- Corpus를 **semantic tree**로 오프라인 구조화(bottom-up agglomerative 또는 top-down divisive + multi-level summary) → 온라인에서 LLM이 tree를 **traverse**하여 로그 복잡도 검색.
- 본 논문과 **거의 동일한 아키텍처 철학**. 차이점은 (a) LATTICE는 일반 large-corpus retrieval, 본 논문은 enterprise QA 및 "skill file" 형식 명시, (b) 본 논문은 파일시스템/SKILL.md 비유와 Agent Skills 생태계 결합을 강조.
- 관찰: Lawrence B. Hsieh ≠ Cho-Jui Hsieh (동명이인 가능성 높음). 본 논문의 Hsieh의 학력·소속은 공개 자료로 특정하기 어렵다 ([Impactio 페이지](https://www.impactio.com/researcher/lawrence-b-hsieh) 존재하나 본 논문 연결 불확실).

### 2.2 RAPTOR (Sarthi et al., ICLR 2024, [arXiv:2401.18059](https://arxiv.org/abs/2401.18059))
- 본 논문의 **핵심 baseline**. 재귀 클러스터링·요약 트리 + embedding 기반 retrieval.
- 차이점(예상): RAPTOR는 트리 노드를 **임베딩 후 top-k**, 본 논문은 에이전트가 **파일 브라우징으로 순회**.
- 코드: [parthsarthi03/raptor](https://github.com/parthsarthi03/raptor) (official).

### 2.3 A-RAG — Agentic RAG with Hierarchical Retrieval Interfaces (2026-02, [arXiv:2602.03442](https://arxiv.org/abs/2602.03442))
- 본 논문의 "agentic RAG baseline" 카테고리 대표. `keyword_search` / `semantic_search` / `chunk_read` 3개 도구를 에이전트에게 노출.
- 차이점: A-RAG는 **도구 기반 검색 인터페이스 다양화**, 본 논문은 **corpus 구조 자체를 파일 트리로 materialize**. "도구의 다양성" vs "환경의 가시성" 대비.
- 코드: [Ayanami0730/arag](https://github.com/Ayanami0730/arag).

### 2.4 GraphRAG (Microsoft Research, 2024)
- 엔티티·관계 그래프 + Leiden 계층 클러스터링. 본 논문 자체 요약 인용: "hierarchical approaches such as RAPTOR and GraphRAG organize documents into trees or graphs".
- 본 논문의 approach는 GraphRAG의 **그래프 추출 부담을 생략**하고 순수 문서 클러스터링에 기반한다는 점에서 단순화된 변형으로 볼 수 있다.
- 공식: [microsoft.github.io/graphrag](https://microsoft.github.io/graphrag/).

### 2.5 Agent Skills 서베이 (2026-02, [arXiv:2602.12430v3](https://arxiv.org/abs/2602.12430))
- 본 논문의 **개념적 토대**를 제공. SKILL.md / progressive loading / skill acquisition / security. "skill을 문서 corpus 위에 올리는" 본 논문의 발상은 이 서베이의 (i) 아키텍처 축과 (iv) 보안 축(distilled summary의 hallucination risk)에 직접 닿는다.

---

## 3. 후속·경쟁 연구

**현재 시점 (2026-04-21) 기준 후속 인용 거의 없음** — 논문 공개 5일차. 다만 동일 문제 공간에서 병행 진행 중인 경쟁 접근이 다수 관측된다.

- **A-RAG** ([arXiv:2602.03442](https://arxiv.org/abs/2602.03442), 2026-02) — 도구 기반 에이전틱 RAG.
- **Knowledge Graph RAG for Enterprise Documents** ([arXiv:2604.14220](https://arxiv.org/html/2604.14220), 2026-04) — 본 논문과 같은 주에 cs.IR에 올라온 엔터프라이즈 대상 KG-RAG.
- **Reasoning RAG via System 1/2 서베이** ([arXiv:2506.10408](https://arxiv.org/html/2506.10408v1)) — 에이전틱 RAG의 reasoning 축 정리.
- **SkillClaw** ([HF 페이퍼 2604.08377](https://huggingface.co/papers/2604.08377)) — 에이전틱 스킬 진화(collectively with Agentic Evolver). 본 논문과 같이 "스킬"을 중심에 두는 2026-04 신작.
- **SkillX** ([HF 페이퍼 2604.04804](https://huggingface.co/papers/2604.04804)) — agent용 스킬 지식베이스 자동 구축. 본 논문과 직접 비교 가능한 자매 연구.

비교 요약 — 본 논문 vs 경쟁:

| 접근 | 오프라인 구조화 | 온라인 메커니즘 | corpus 가시성 |
|---|---|---|---|
| Dense RAG | embedding 인덱스 | top-k 임베딩 유사도 | 없음 |
| RAPTOR | 재귀 클러스터 요약 트리 | 레벨별 임베딩 검색 | 없음 |
| GraphRAG | 엔티티 그래프 + community 요약 | global/local community 검색 | 간접 (community 요약 통해) |
| A-RAG | 다중 granularity 인덱스 | 3종 도구 호출 ReAct 루프 | 부분적 |
| LATTICE | semantic tree | LLM traverse (log 복잡도) | 있음 (트리 구조 자체) |
| **Corpus2Skill (본 논문)** | **skill 파일 트리 (filesystem)** | **에이전트가 파일 탐색** | **완전 가시 (파일시스템으로)** |

---

## 4. 공식 자산

- **논문 PDF/HTML:** [arxiv.org/abs/2604.14572](https://arxiv.org/abs/2604.14572) · [HTML v1](https://arxiv.org/html/2604.14572v1) · [PDF](https://arxiv.org/pdf/2604.14572)
- **공식 코드 저장소:** **확인되지 않음.** arXiv abstract 페이지의 comments 필드에 코드 링크 없음. 저자 중 Yiqun Sun의 GitHub(`dukesun99`)·개인 홈페이지([comp.nus.edu.sg/~sunyq](https://www.comp.nus.edu.sg/~sunyq/), [dukesun99.link](https://dukesun99.link/))를 뒤져봤으나 본 논문과 연결된 repo 미발견 (현재 pinned: neovis.js, Graphin, DPoSFormalVerification 등 그래프 시각화 및 블록체인 프로젝트).
- **저자 발표/데모 페이지:** 미확인 (2026-04-21 현재).
- **저자 정보:**
  - **Yiqun Sun (dukesun99):** NUS School of Computing 박사(졸업). 지도교수 Anthony Tung. 관심 분야: Knowledge Graphs, LLM, diversified news search, 텍스트 임베딩. 이전 소속은 Magellan Technology Research Institute ([LinkedIn](https://www.linkedin.com/in/dukesun99/)). ACL 2025 발표 이력.
  - **Pengfei Wei:** ByteDance AI Lab (Singapore) Research Scientist로 공개된 동명 연구자 있음 ([pengfei-wei.com](http://pengfei-wei.com/)). 동일 인물 여부는 본 논문에서 명시되지 않음.
  - **Lawrence B. Hsieh:** 공개 프로필 접근 제한([Impactio](https://www.impactio.com/researcher/lawrence-b-hsieh)). LATTICE 저자 Cho-Jui Hsieh와는 별개(동명이인)로 추정.

---

## 5. 커뮤니티 반응

### 5.1 학계 반응 (인용·후속 논의)
- **인용:** 2026-04-21 기준 공식 인용 지표 반영 전 (공개 5일차). Semantic Scholar 인덱싱 전으로 추정.
- **후속 논의:** 현재 없음. OpenReview 리뷰 페이지 미확인(arXiv 단독 공개).

### 5.2 실무/오픈소스 커뮤니티
- **Hacker News:** **본 논문 단독 스레드 미확인.** "Agent Skills" 상위 주제로는 활발한 토론이 존재. 아래 요약은 **원출처 재검증(2026-04-21) 후 정정된 판본**:
  - [item 46871173 Agent Skills](https://news.ycombinator.com/item?id=46871173) (260 댓글): **실제 합의는 skills 정당화 쪽**. 최상위 설득력 댓글은 ashdksnndck(~180 upvote)·smithkl42(~200 upvote)·mhalle(~150 upvote)로, progressive loading의 실효성·컨텍스트 윈도우 유한성·skills가 문서와 달리 코드/데이터/메타데이터 패키지라는 점을 옹호. vidarh(46874807, ~340 upvote)는 **"본질적 차이 없음" 환원주의가 아니라 "현재 Claude Code의 skills 구현이 suboptimal"이라는 구체 진단**. 순수 환원주의는 iainmerrick 1인 소수 의견. → 본 논문 비판으로서의 무게는 **낮음**, "파일시스템 은유의 실효성에 대한 실증 부족"이라는 현장 실무자 관찰 수준.
  - [item 46723183 Skill.md open standard](https://news.ycombinator.com/item?id=46723183) (원 링크 [mintlify.com/blog/skill-md](https://www.mintlify.com/blog/skill-md), submitter skeptrune): **Mintlify는 실제로 install.md → skill.md로 전환**하며 공식 블로그에서 "install.md는 채택 부진해서 skill.md로 이동"이라 자인 ([GitHub mintlify/install-md](https://github.com/mintlify/install-md) 실존). 한 검색 출처에 따르면 **install.md deprecate는 2026-01-21**. 스레드 내 lovich의 "6일 만에 폐기"는 1인 추정 수치이므로 정확 일수는 **"단기간 내 전환"으로 완화 기술**. ClassAndBurn의 "채택 부진→빠른 폐기" 비판은 **사실 기반**, RadiozRadioz의 "rugpull 우려"는 체감 해석. → 본 논문 비판 무게 **중간**: Serve Phase가 의존하는 SKILL.md 생태계의 검증된 외부 리스크.
  - [item 46351787 Context Engineering](https://news.ycombinator.com/item?id=46351787), [item 46315414 Skills for organizations](https://news.ycombinator.com/item?id=46315414) 등 파편화된 토론이 다수.
- **Reddit (r/MachineLearning, r/LocalLLaMA):** 본 논문 관련 게시물 미확인.
- **X/Twitter:** 검색 권한 제한으로 수집 한계 (아래 §7 참조).
- **Hugging Face Papers:** 본 논문 전용 discussion 페이지 미확인.

### 5.3 한국어 커뮤니티
- **본 논문 직접 언급 한국어 자료 미확인.**
- 배경 개념 한국어 정리:
  - Agentic RAG 개념 정리 — [velog @lyj_0316](https://velog.io/@lyj_0316/Agentic-RAG), [velog @corca_llm](https://velog.io/@corca_llm/Agentic-RAG-System-자율성을-더한-AI의-미래).
  - Agent Skills(SKILL.md) 한국어 소개 — [Junhyunny's Devlogs](https://junhyunny.github.io/ai/ai-agent/skills/agent-skills/skills-md/): "개별 파일 방식에서 벗어나 SKILL.md 파일을 중심으로 한 디렉토리 구조를 새롭게 채택".
  - 한국어 스킬 모음 — [NomaDamas/k-skill](https://github.com/NomaDamas/k-skill), [heilcheng/awesome-agent-skills(한국어)](https://github.com/heilcheng/awesome-agent-skills/blob/main/README.ko.md).
  - news.hada.io 상에서 본 논문 관련 포스팅 미확인.

### 5.4 시사점 요약 (2026-04-21 신뢰도 재평가 후 정정판)
현시점 본 논문에 대한 **직접 비판·지지는 관측되지 않음**. 넓은 Agent Skills 담론에서 본 논문이 맞닥뜨릴 예상 비판은 **신뢰도 등급을 명시**해 정리:

| # | 비판 | 신뢰도 | 본 논문 비판 무게 | 권장 프레이밍 |
|---|---|---|---|---|
| 1 | "파일시스템 은유가 실제로 기여하는 것이 vector DB 대비 무엇인가?" | HN 1인 의견 + 반박 다수 | 낮음 | "파일시스템 은유의 실효성 실증 부족" (환원주의 비판은 HN 합의 아님) |
| 2 | "SKILL.md 표준이 흔들리는데 그 위에 RAG를 얹어도 괜찮은가?" | 사실 기반 사건(install.md→skill.md 공식 전환) | 중간 | "Serve Phase 의존 생태계의 검증된 외부 리스크" |
| 3 | "agentic loop 토큰비·지연 비용을 skill file 탐색이 줄이는가 늘리는가?" | context engineering 일반 비판 | 중간 | 본 논문이 token cost·latency 수치 공개했는지 확인 필요 |

---

## 6. 재현성·논란

- **재현 실패 보고:** 없음 (공개 직후·코드 미공개 상태로 재현 시도 자체가 없음).
- **결과 이의:** 없음.
- **코드/모델 공개 여부:** 코드·체크포인트 공개 미확인. WixQA는 공개 데이터셋이므로 데이터 재현성은 확보되어 있으나, Corpus2Skill 파이프라인(클러스터링 알고리즘·요약 프롬프트·skill 파일 포맷·에이전트 시스템 프롬프트) 재현은 현재 불가능 추정.
- **유사 문제의 경험적 기준선:** Agent Skills 서베이([arXiv:2602.12430](https://arxiv.org/abs/2602.12430)) §6.2가 **재인용**한 수치 "커뮤니티 스킬 26.1%에 취약점"의 **원출처는 Liu et al., "Agent skills in the wild", [arXiv:2601.10338](https://arxiv.org/abs/2601.10338), 참고문헌 [14]**. 원 실험은 two major marketplaces에서 42,447개 수집→31,132개 분석, **SkillScan**(정적 분석 + LLM 의미 분류) 사용, 4 카테고리(프롬프트 인젝션, 데이터 유출 13.3%, 권한 상승 11.8%, 공급망 위험). 실행 스크립트 번들 skills는 명령어 전용 대비 취약점 확률 2.12배(p<0.001).
  - **본 논문 파이프라인과의 논리적 거리:** Liu et al.은 **인간 기여 skill**, 본 논문은 **LLM 자동 생성 summary**로 직접 동일 대상 아님. 26.1%를 그대로 Corpus2Skill에 투사는 **과추론**. 완화된 활용: "skill artifact는 경험적으로 검증된 신뢰성 위험 기준선을 가지며, LLM-생성 summary의 factuality 검증 부재는 유사 위험을 재생산할 수 있다". 본 논문이 summary hallucination 검증 절차를 본문에 명시하는지가 실질 쟁점 → (deep-reader §재현성/검증 섹션 참조 요청).

---

## 7. 수집 한계 (접근 불가 영역)

- **X/Twitter 직접 검색:** WebFetch 권한 도메인에 포함되나 X 검색 페이지는 로그인 벽으로 실질적 수집 불가. "저자 트윗·ML 인플루언서 반응"이 이 섹션에서 누락됨.
- **OpenReview:** 본 논문은 conference 제출 흔적 확인되지 않음. 리뷰어 코멘트 없음.
- **저자 개인 홈페이지 크롤:** Yiqun Sun 홈페이지 2건 모두 403 또는 부분 콘텐츠만 반환. 최신 publication 리스트의 본 논문 포함 여부 직접 확인 실패.
- **Semantic Scholar/Google Scholar 인덱스:** 공개 5일차로 인용 지표 반영 전.
- **GitHub 비공개 저장소:** 저자가 `dukesun99` 계정의 private repo로 코드를 보유할 가능성은 배제 불가.
- **한국어 Slack/Discord 커뮤니티:** 로그인 기반으로 외부 검색 불가.

---

## 부록 A. deep-reader 참조 권장 포인트

아래 사항은 deep-reader가 본문에서 특히 교차확인할 가치가 있다 (본 섹션은 본문 해석이 아니라 교차검증 요청):

1. **LATTICE(2510.13217)와의 차별점 본문 명시 여부** — 가장 가까운 경쟁 연구이므로 related work에서 어떻게 구분하는지.
2. **"skill file" 포맷이 Anthropic SKILL.md와 실제로 같은지** — 마케팅 비유 수준인지, 실제로 SKILL.md 사양을 따르는지.
3. **WixQA 평가 시 사용한 retrieval/agent 모델명**과 RAPTOR/A-RAG baseline 재실행 여부.
4. **Summary hallucination 검증 절차**: 오프라인 단계에서 LLM이 만든 계층 요약의 정확성을 어떻게 보장하는지 (§6 재현성 리스크와 연결).
5. **Navigation loop의 토큰비·지연 수치** 공개 여부 — 품질은 올라도 비용이 올랐다면 실무 채택성에 치명적.

---

## 부록 B. deep-reader 요청 핵심 항목 심층 조사 (RAPTOR/Skills API/WixQA/관련 RAG/고전 선행/Agent 스킬)

조사 시점: 2026-04-21 (후속 보강). deep-reader의 요청 우선순위에 따라 정리.

### B.1 RAPTOR — 실제 메커니즘 정확 확인 (최우선)
출처: [arXiv:2401.18059](https://arxiv.org/abs/2401.18059), Sarthi et al., ICLR 2024.

- **클러스터링 파이프라인:** UMAP 차원축소 → GMM soft clustering → BIC로 클러스터 수 선택. 문서 하나가 **복수 클러스터에 속할 수 있는 soft assignment**. UMAP의 `n_neighbors`를 변경해 계층(상위 코스-그레인 vs 하위 파인-그레인) 구축. 구체 차원·n_neighbors 기본값은 공식 초록·기본 HTML에 미명시.
- **두 retrieval 전략:**
  - **Tree Traversal:** 루트에서 시작해 각 레벨에서 코사인 유사도 top-k만 선택하며 leaf까지 내려감. 레벨마다 추상-구체 비율이 **고정**됨.
  - **Collapsed Tree:** 트리 전체를 평평하게 만든 뒤 **모든 노드에 동시 cosine 검색**, token 예산(예: 2000 tokens ≈ top-20 nodes)까지 채움.
- **저자 최종 선택:** **Collapsed Tree**. 이유: 질문 유형에 따라 필요한 granularity가 다르므로 고정 비율보다 유연. 저자의 모든 주요 실험에서 collapsed tree를 사용.
- **→ 본 논문의 "RAPTOR는 flat vector store + collapsed retrieval" 특성화는 정확.** 저자의 실제 선호 전략을 인용한 것.
- **공정성 소폭 이슈(deep-reader §4.1/4.2 확증):** 본 논문은 RAPTOR를 **collapsed-tree 모드로만** baseline으로 돌림. tree-traversal 모드는 baseline 미포함. RAPTOR 원저자가 collapsed를 선호했다는 점에서 그 자체는 공정하나, **tree-traversal이야말로 Corpus2Skill의 "LLM agent가 레벨별로 내려가며 탐색"과 구조적으로 더 비슷한 운영 모드** — 가장 근접한 비교군을 빠뜨린 셈. "계층 탐색 vs 계층 탐색"의 직접 대조가 아닌 "계층 탐색 vs 계층 vector search" 비교만 남음. 공정성 문제로서는 **약함(최소)** 수준의 관찰이나, "에이전트 탐색이 임베딩 기반 계층 검색보다 본질적으로 낫다"는 주장의 증거 강도는 제한됨.
- **평가 데이터셋:** 공식은 QuALITY, QASPER, NarrativeQA 중심. **WixQA 또는 enterprise customer-support 계열 벤치마크 평가는 RAPTOR 원 논문에 없음** → 본 논문의 WixQA 기준 RAPTOR 수치는 본 논문 저자가 **자체 재실행**한 결과.
- **공식 성능 요약:** QuALITY에서 GPT-4 + RAPTOR로 절대 정확도 +20%p. QASPER·NarrativeQA에서도 SOTA 보고(구체 수치는 ICLR 논문 표 참조).
- **코드:** [parthsarthi03/raptor](https://github.com/parthsarthi03/raptor) 공식.

### B.2 Anthropic Skills API — 제약 수치 출처 (고우선)
출처: [platform.claude.com/docs/.../agent-skills/overview](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview), [skills-guide](https://platform.claude.com/docs/en/build-with-claude/skills-guide), [Anthropic engineering blog](https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills).

- **Progressive disclosure 구조 확인됨:** 3단계.
  - Level 1: YAML frontmatter(name ≤ 64자, description ≤ 1024자) — **startup에서 항상 로드**. ~100 tokens/skill.
  - Level 2: SKILL.md 본문 — skill이 트리거되면 bash로 `read SKILL.md`. < 5k tokens.
  - Level 3: 번들 파일·스크립트 — 참조되었을 때만 읽힘, 스크립트는 output만 컨텍스트 진입.
- **본 논문이 인용한 수치 "8 skills/request, 200 files/skill, 30 MB/skill" 공식 문서 교차검증 (deep-reader 본문 재확인 후 확증):**
  - **"8 skills per request"** — Skills API 컨테이너 파라미터 설명에 명시. 공식 문서 확인 완료.
  - **"30 MB per skill"** — Skills 생성 가이드에 "Total upload size must be under 30 MB" 명시. 공식 문서 확인 완료.
  - **"200 files per skill"** — **현재 공개 Anthropic 공식 문서에서 확인 불가 확증.** deep-reader 본문 재확인 결과 §3.2 footnote 222에 인용되었으나 **HTML fetch에서 footnote 222 내용 자체가 비가시** — URL·출처가 본문에 attach되지 않음. 가능한 해석: (a) 저자 작성 시점 Skills API 문서 스냅샷(HN 토론에서도 Skills API 잦은 변경이 지적됨), (b) 내부 beta 문서, (c) "30 MB total with max N files" 파생 제약의 오독.
  - **자기모순 관찰(deep-reader §3.2 확인):** 본 논문이 §3.2 footnote에서 이 API 제약들이 *"informed our default parameters but are not fundamental to the framework design"*이라 기술하면서도, **실제 모든 주요 하이퍼파라미터(K=7, p=10, compact mode 도입 등)가 이 제약에 맞춰 튜닝됨**. "framework에 근본적이지 않다" 주장과 설계 현실의 괴리가 존재.
  - **재현성 비판 무게 (강):** 출처 확인 불가능한 단일 API 수치에 Serve Phase 핵심 설계가 묶여 있고, 저자들 스스로 "근본적이지 않다"라고 하면서도 실제로는 전 파이프라인이 그 제약에 종속. 본 논문 코드도 미공개이므로 이 부분 재현성은 사실상 불가.
- **벤더 종속성:** Skills API는 **Anthropic Claude 제품군 전용**. 외부 개발자가 "동일 사양"을 타 프로바이더에서 구현할 수 없음. 단, SKILL.md 포맷 자체는 평문 YAML+Markdown이므로 파일 포맷은 이식 가능. **bash 기반 filesystem navigation + progressive loading 런타임**이 재현하기 어려운 부분.
- **대안 구현:** 커뮤니티 대안 [agentskills/agentskills (사양)](https://github.com/agentskills/agentskills), [VoltAgent/awesome-agent-skills](https://github.com/VoltAgent/awesome-agent-skills), GitHub CLI `gh skill` ([2026-04-16 changelog](https://github.blog/changelog/2026-04-16-manage-agent-skills-with-github-cli/))이 존재하나 런타임 일치 여부 불명.

### B.3 WixQA — 다른 논문 평가 여부 및 편향 (중우선)
출처: [arXiv:2505.08643](https://arxiv.org/abs/2505.08643), 2025-05-13 공개. [HF dataset](https://huggingface.co/datasets/Wix/WixQA) MIT 라이선스.

- **서브셋:** ExpertWritten 200 / Simulated 200 / Synthetic **6,222** (KB 당 1개).
- **공식 baseline:** BM25 + **E5 dense (e5-large-v2, 335M)**. Context Recall에서 E5가 BM25 능가, 특히 multi-article synthesis가 필요한 쿼리에서 격차 큼. Simulated 서브셋이 가장 어려움 (사용자 실제 대화 기반).
- **공식 메트릭:** F1 / BLEU / ROUGE-1 / ROUGE-2 / Factuality(LLM judge) / Context Recall(LLM judge).
- **다른 논문의 채택:** WixQA 공개가 2025-05이므로 채택 논문은 아직 초기 단계. 본 논문(Corpus2Skill) 외에 WixQA를 채택한 동류 RAG 평가 논문은 검색에서 미확인 → **WixQA 자체가 아직 사실상 표준이 되기 전 단계**. 단일 벤치마크 평가 비판이 가능하나, 저자가 WixQA 선택 자체는 합리적(enterprise customer support corpus로 공개된 거의 유일한 대형 KB).
- **편향/일반화 한계:** 저자들이 "domain-specific customer support issues 기반"이라고 **스스로 인정**. 일반 도메인(뉴스, 법률, 의료 등) 일반화는 **저자 논문에서 명시적 논의 없음**. → 본 논문의 WixQA-only 평가가 Corpus2Skill의 범용 주장을 얼마나 뒷받침하는지는 **insight-analyst가 비판 포인트로 삼을 수 있음**.

### B.4 Structure-aware / Agentic RAG 경쟁군 (중우선)
본 논문 §6 Related Work에서 **"structure-aware and agentic RAG"**로 한데 묶은 시스템들. 본문이 이들과 어떻게 차별화하는지가 novelty 판단 핵심.

| 시스템 | arXiv | 게재 | 핵심 | 본 논문과의 주장된 차이 |
|---|---|---|---|---|
| **RAPTOR** | 2401.18059 | 2024-01 | UMAP+GMM+BIC 재귀 클러스터 요약 트리, collapsed tree retrieval | 본 논문이 "파일로 materialize + 에이전트 탐색"으로 대체. **baseline으로 재실행.** |
| **GraphRAG** | (Microsoft) | 2024 | 엔티티 그래프 + Leiden community + community summary | "serve 시 graph DB 필요" vs "정적 파일시스템" |
| **StructRAG** | 2410.08815 | 2024-10 | inference-time hybrid structurization (table/graph/algorithm) | 구조 생성이 **온라인**(본 논문은 오프라인) |
| **BookRAG** | 2512.03413 | 2025-12 | BookIndex (hierarchical tree + entity graph) + Information Foraging Theory 기반 agent planner, 3단계 쿼리 분류(single-hop/multi-hop/global) | 본 논문 §6에서 인용됨. "retrieval/search infrastructure 유지" vs "정적 파일시스템"으로 구분. 차별점이 "파일시스템 은유"에 전적 의존. |
| **NaviRAG** | 2604.12766 | **2026-04-14** | 계층적 지식 + LLM agent active navigation, "passive→active" 모토 | **concurrent submission (본 논문 2일 전).** 본 논문 §6 한 문장만 할당: *"Meanwhile, NaviRAG (Dai et al., 2026) builds a hierarchical view of documents and uses an LLM agent to actively navigate it, detecting information gaps and retrieving at the appropriate granularity."* **"concurrent"·"independent work" 같은 동시기 인정 표현은 본문 전체에서 전무(deep-reader grep 확인).** motivation 슬로건이 토씨까지 같은 수준인데 "Meanwhile"로 툭 던지고 넘어간 것은 학문적 예의 결여 소지. |
| **HCAG** | 2603.20299 | 2026-03 | 계층적 code-knowledge base + multi-agent discussion | 본 논문 §6에서 NaviRAG와 함께 "동일 motivation" 카테고리로 묶여 차별점 논증. code generation 대상이라 직접 경쟁은 아니나, 동일 motivation 인지 대상으로는 포함됨. |
| **A-RAG** | 2602.03442 | 2026-02 | 3개 도구(keyword/semantic/chunk_read) + ReAct 루프 | 본 논문 §6 Related Work + 경험적 baseline("Agentic") 모두 포함. "검색 도구 다양화" vs "환경 가시화" 차별점 유효. |
| **HiRAG** | (arXiv:2503.10150 = prep판) / **Findings of EMNLP 2025, pp.6044–6060** | 2025-03 prep / EMNLP'25 게재 | 엔티티·커뮤니티 레벨을 interleave한 계층적 지식 그래프 기반 RAG. Huang 외 8인. [공식 repo](https://github.com/hhy-huang/HiRAG) | 본 논문 §6 원문: *"HiRAG ... augments RAG with a hierarchical knowledge graph that interleaves entity and community levels; **it still relies on graph retrieval at query time rather than agent-driven file navigation.**"* 차별점 주장 명시적. |
| **SPD-RAG** | 2603.08329 | 2026-03-09 | Sub-Agent Per Document RAG. Akay, Kartal, Alparslan, Ortakoyluoglu, Akpinar (TOBB ETU·OSTIM). [공식 repo](https://github.com/NebulAICompany/SPD-RAG) | 본 논문 §6 원문: *"SPD-RAG (Akay et al., 2026) assigns a dedicated sub-agent per document in a multi-agent pipeline."* 문서 축 multi-agent vs 본 논문 단일 에이전트 파일 탐색 — 차별점 유효. |
| **LATTICE** | 2510.13217 | 2025-10 | semantic tree + LLM traverse, 로그 복잡도 | **본 논문 §Related Work·References·실험 baseline 전체 누락 확정(deep-reader 본문 grep 확인).** 선행연구 크레딧 누락. |

**중대한 관측 (deep-reader 본문 재확인 후 업데이트):**
- **LATTICE 누락 확정:** deep-reader가 본문 full-text grep으로 "LATTICE", "2510.13217", Gupta/Chang/Bui/Dhillon 모두 0 hit 확인. 6개월 앞선 직접 선행연구(동일 motivation: "LLM이 계층 트리를 traverse") 인용 부재. **"몰랐다" 변호는 약함** — 본 논문이 NaviRAG(2026)·A-RAG(2026)·HCAG(2026) 등 **더 최근 preprint는 인용**하고 있음. 저자가 구조-aware RAG와 embedding-based retrieval 경계를 긋는 방식에서 LATTICE를 후자로 잘못 분류했거나 탐색에서 누락한 구조적 실수 가능성.
- **NaviRAG는 인용됐으나 concurrent work acknowledgment 결여 (deep-reader 재확증):** §6에 한 문장만 할애하고 "concurrent·independent·simultaneous" 같은 동시기 인정 표현이 본문 전체에서 0 hit. 2일 차 concurrent임에도 "Meanwhile, NaviRAG ..."로 처리. 본 논문 motivation("passive retrieval → active navigation")이 NaviRAG 것과 토씨까지 동일한데 이 정도 언급은 **학문적 예의 결여** 소지가 명확. 본 논문 차별점(파일시스템 + 에이전트 위임) 주장도 NaviRAG의 구현 세부를 공정히 짚지 못한 채 카테고리 기반 선긋기에 그침.
- **BookRAG는 본 논문이 인용함(deep-reader 확인).** 이전 §B에서 "뭉뚱그림"으로 기록한 부분 수정. 다만 BookIndex도 트리 인덱스이므로 본 논문 차별점이 여전히 "파일시스템 은유"에 전적 의존하는 점은 유효.
- **인용 구조적 균형 실패의 해석 (deep-reader 합의):** Agent Skills 계열(Voyager/SkillX/SkillClaw/EvoSkill) + 구조-aware + agentic RAG 계열(RAPTOR/GraphRAG/HiRAG/StructRAG/BookRAG/NaviRAG/A-RAG/HCAG/SPD-RAG)을 **모두** 커버하면서 LATTICE만 빠진 것은 우연보다는 **서브도메인 경계 설정의 구조적 실수**. 저자들은 **"구조-aware + agentic RAG"** 범주로는 경쟁군을 서치했으나 LATTICE가 속한 **"LLM-guided tree retrieval"** 각도로는 서치하지 않음 — retrieval algorithm 관점의 탐색 부재. 즉 단순 "한 건 누락"이 아니라 **문헌 지형도 인식의 사각지대**. reviewer가 novelty 평가의 구조적 결함으로 지적할 근거 충분.

### B.5 Scatter/Gather — 고전 선조 (Cutting/Karger/Pedersen/Tukey, SIGIR 1992)
출처: [dblp](https://dblp.org/rec/conf/sigir/CuttingPKT92.html), 1,740+ 인용 고전.

- **핵심 발상:** Document clustering을 **IR 도구가 아닌 브라우징 패러다임**으로 재규정. 매 iteration에서 시스템이 문서를 클러스터(=scatter)하고 사용자가 관심 클러스터 선택(=gather) → 반복적 드릴다운. Cluster hypothesis("유사 문서가 같은 질의에 관련") 기반.
- **알고리즘 공헌:** Buckshot, Fractionation 두 선형 시간 클러스터링. O(kn) 복잡도.
- **본 논문 인용 맥락:** 본 논문 §6은 Scatter/Gather를 "cluster hypothesis 기반 navigable clusters"의 선조로 명시 인용, 차이는 **"Corpus2Skill은 navigation을 LLM 에이전트에 완전히 위임"** — 1992년 사용자(사람)의 클릭 기반 네비게이션을 2026년 LLM 에이전트로 대체.
- **평가:** Scatter/Gather 인용 자체는 정직하고 정확. 다만 본 논문이 LLM 에이전트 자동화의 **실질적 이득**(사용자보다 낫다는 근거)을 실험에서 보이는지 본문 확인 필요 — 1992년 아이디어의 LLM 버전이라는 프레이밍이 성립하려면.

### B.6 Voyager (Wang et al., 2023), SkillX, SkillClaw — "skill" 선행연구 (저우선)
- **Voyager** ([arXiv:2305.16291](https://arxiv.org/abs/2305.16291), Wang/Xie/Jiang/Mandlekar/Xiao/Zhu/Fan/Anandkumar, Caltech·Stanford·UT·NVIDIA): Minecraft 환경에서 GPT-4 agent가 **실행 가능 코드 조각**을 skill library에 축적. 3x more unique items, 15x faster tech-tree. 본 논문이 Voyager를 명시 인용(§6) — "skill as executable code" 전통. 본 논문과 차이: Voyager는 **action trajectory → skill**(bottom-up from experience), 본 논문은 **static corpus → skill**(top-down from documents). 같은 "skill" 단어를 다른 입력에 적용.
- **SkillX** ([arXiv:2604.04804](https://arxiv.org/abs/2604.04804), [zjunlp/SkillX](https://github.com/zjunlp/SkillX), 2026-04-07): Agent experience를 **3계층 skill hierarchy**(strategic plan / functional / atomic)로 자동 구조화 + iterative refinement + exploratory expansion. AppWorld/BFCL-v3/τ²-Bench 평가. 본 논문과 같은 "agent skill" 패러다임이지만 **소스가 agent trajectory**.
- **SkillClaw** ([arXiv:2604.08377](https://huggingface.co/papers/2604.08377), 2026-04): 다중 사용자 에이전트 생태계의 집합적 skill 진화. 6라운드 진화로 +88% 개선 주장.
- **학술적 의미:** "action sequence → skill"(Voyager/SkillX/SkillClaw) vs **"corpus → skill"(본 논문)**은 **입력 모달리티**는 다르지만 **산출물(파일시스템 기반 skill library)**에서 수렴. 본 논문이 이 interface 측면의 통합을 명시적으로 주장하면 positioning 강화 가능.

### B.7 Deep-reader 회신 요약 (본 맥락 문서 §B 요지 — 2026-04-21 deep-reader 본문 재확인 후 확증판)

1) **RAPTOR 특성화 정확함. 공정성 소폭 이슈 확인.** §4.1/4.2에서 RAPTOR는 **collapsed-tree only**로 baseline 실행, tree-traversal 모드는 미포함. 가장 구조적으로 유사한 비교군을 빠뜨린 셈이나, RAPTOR 원저자 선호 모드를 채택했다는 점에서는 공정. 약한 공정성 관찰 수준.

2) **"200 files/skill" 출처 확인 불가 확증.** 본 논문 §3.2 footnote 222에 인용되었으나 HTML fetch에서 footnote 222 자체가 비가시 — URL·출처 attach 없음. 자기모순: 저자들이 *"not fundamental to framework design"*이라 기술하면서도 모든 하이퍼파라미터(K=7, p=10, compact mode)가 이 제약에 맞춰 튜닝. **재현성·벤더 종속 비판 무게 강**.

3) **WixQA 단일 벤치 + 공개 5개월차 표준 전 단계 + 저자 자인 domain bias.** 일반 도메인 전이 주장 본문에 없으면 약점.

4) **선행연구·concurrent work 처리 문제 확증 (최강 비판 무게):**
   - **LATTICE(2025-10) 완전 누락** (deep-reader grep 0 hit).
   - **NaviRAG(2026-04-14) 인용 존재하나 한 문장("Meanwhile, NaviRAG ...")만 할애, "concurrent·independent·simultaneous" 동시기 인정 표현은 본문 전체에서 0 hit.** 2일 차 concurrent임에도 처리 미흡.
   - BookRAG(2025-12)는 인용됐으나 차별점 주장이 "파일시스템 은유"에 전적 의존.
   - 종합: NaviRAG 2일 전 + LATTICE 6개월 전 모두 동일 아이디어. **본 논문 novelty 주장의 코어가 흔들림**.

5) **Scatter/Gather 인용 정직**. 차이("LLM에 navigation 위임")의 경험적 근거는 본문 실험 확인 필요.

6) **Voyager/SkillX 인용 정직**. 입력 모달리티 차이 명확.
