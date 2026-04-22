# Don't Retrieve, Navigate — 파일시스템 은유로 RAG를 재설계한 Corpus2Skill 비판적 리뷰
작성일: 2026-04-21 | 조사 범위: 단일 논문 (arXiv:2604.14572)

## 개요 (3줄 이내)

- 문서 코퍼스를 오프라인에서 계층적 스킬 디렉토리(SKILL.md / INDEX.md)로 컴파일해두고 LLM 에이전트가 파일시스템처럼 탐색하게 만드는 RAG 대안 **Corpus2Skill**이 WixQA 단일 벤치에서 Dense/RAPTOR/Agentic-RAG 대비 전 지표 최고 성능을 기록했다.
- 그러나 **LATTICE(6개월 선행) 누락**과 **NaviRAG(2일 차 concurrent)를 단 한 문장으로 처리**한 사실은 단순 한 건 인용 실수가 아니라 "LLM-guided tree retrieval" 서브도메인을 놓친 **문헌 지형도의 사각지대**를 드러내며, 결과표가 지지하는 결론은 "파일시스템 materialization의 우월성"이 아니라 "계층 구조 + 에이전트의 우월성"에 가깝다.
- 따라서 이 논문은 method paper가 아니라 **Anthropic Skills API 생태계 위에서 RAG를 구성한 system/engineering 사례**로 재포지셔닝할 때만 그 기여(progressive disclosure의 RAG 포팅, 제약 하의 설계 결정)가 온전히 평가된다.

## 조사 대상

- **제목:** Don't Retrieve, Navigate: Distilling Enterprise Knowledge into Navigable Agent Skills for QA and RAG
- **저자:** Yiqun Sun, Pengfei Wei, Lawrence B. Hsieh
- **게재:** arXiv 프리프린트, 2026-04-16 (cs.IR / cs.AI / cs.CL / cs.MA)
- **URL:** https://arxiv.org/abs/2604.14572
- **라이선스:** CC BY 4.0
- **공식 코드 저장소:** 미확인

## 상세 설명

### 문제 정의와 해결 접근

저자는 표준 RAG에서 LLM이 "검색 결과의 수동적 소비자(passive consumers of search results)"에 불과하다고 지적한다. Top-k 패시지만 주어지기 때문에 에이전트가 코퍼스 조직 구조에 대한 가시성을 갖지 못하고, 따라서 (i) 어디를 볼지 추론, (ii) 막다른 경로에서 backtrack, (iii) 여러 branch 증거 결합을 체계적으로 못 한다는 구조적 한계가 있다고 주장한다.

해결 슬로건: **"Invest compilation-time computation making corpora navigable, rather than query-time computation making corpora searchable."**

### 방법론 (Compile → Serve 2-페이즈)

**Compile Phase (6.5분 / \$5-\$10):**
1. 문서 임베딩 + content-hash ID.
2. Iterative K-Means clustering: cluster → LLM_summarize → re-embed 반복. 하이퍼파라미터 `p`(branching), `K`(루트).
3. 2-5단어 filesystem-safe 라벨.
4. Skill tree materialization: root → SKILL.md, sub → INDEX.md, 문서 본문은 documents.json 외부. 네비 파일 <2 KB.

RAPTOR와 차이: hard K-Means(정확히 하나의 path), 파일시스템 저장, 에이전트 browsing.

**Serve Phase:** Anthropic Skills API의 progressive disclosure. 도구 2종: code_execution(view), get_document(doc_id). 워크플로 2-3 turn.

### 복잡도

`L = ⌈log_p N⌉`, `O(p · log_p N)`. WixQA(N=6,221, p=10) → L=3, ≈30개 요약으로 단일 문서 도달.

### 실험 결과 (WixQA 200 질의)

| Method | F1 | Factuality | CtxRecall | InTokens | \$/query |
|---|---|---|---|---|---|
| BM25 | 0.342 | 0.470 | 0.386 | 708 | 0.007 |
| Dense | 0.363 | 0.536 | 0.450 | 656 | 0.008 |
| Hybrid | 0.360 | 0.524 | 0.410 | 698 | 0.008 |
| RAPTOR | 0.389 | 0.675 | 0.616 | 1,807 | 0.012 |
| Agentic | 0.388 | 0.724 | 0.481 | 25,807 | 0.098 |
| **Corpus2Skill** | **0.460** | **0.729** | **0.652** | 53,487 | 0.172 |

F1 Agentic 대비 +19%rel, Dense 대비 +27%rel. Context Recall +36%rel vs Agentic. 비용은 Agentic의 1.75×, RAPTOR의 14×. Output tokens 752로 Agentic의 절반.

### Ablation
- **(p, K):** Narrow(p=5) 품질 소폭↑·비용 +8%, Wide(p=20) F1 -21%·Factuality -44%.
- **Exploration budget:** 5 vs 20 rounds 차이 1.5%·비용 negligible.
- **LLM:** Sonnet 4.6 F1 0.460 vs Haiku 4.5 F1 0.423, 단 Haiku가 Context Recall에서 오히려 높음(0.705).

### Case Studies
Trace 1: 4-step direct descent. Trace 2: 7-step cross-branch synthesis (두 subgroup 증거 결합). 200 질의 전체 평균 통계는 미공개.

---

## 핵심 인사이트

### 1. 기여 재정의 — method paper가 아니라 system paper

**NaviRAG는 2일 전, LATTICE는 6개월 전, BookRAG는 4개월 전 — 모두 "계층적 LLM navigation" 아이디어를 이미 제시했다.**

**(a) LATTICE (arXiv:2510.13217, 2025-10) — 완전 누락.**
Corpus를 semantic tree로 오프라인 구조화 + LLM tree traverse + 로그 복잡도. 본 논문 compile-then-navigate 골격과 거의 동일. 저자는 RAPTOR·GraphRAG·HiRAG·StructRAG·BookRAG·NaviRAG·A-RAG·HCAG·SPD-RAG + Voyager·SkillX·SkillClaw·EvoSkill까지 **13편 이상을 나열하면서 LATTICE만 빠졌다**. deep-reader grep으로 LATTICE/2510.13217/저자명 0 hit 확정.[^02]

[^02]: context-scout §B.4 프레이밍 합의: 저자들은 "구조-aware + agentic RAG" 범주는 폭넓게 서치했으나 **"LLM-guided tree retrieval"** 각도는 놓쳤다. 서브도메인 경계를 RAG 중심으로만 긋고 retrieval algorithm 관점을 놓친 결과 — 단순 한 건 누락이 아니라 **문헌 지형도 인식의 사각지대**이며 본 논문 novelty 평가의 구조적 결함의 지표.

**(b) NaviRAG (arXiv:2604.12766, 2026-04-14) — 2일 차 concurrent, 1문장 처리.**
본 논문 §6에 `"NaviRAG and HCAG share the motivation of active hierarchical navigation, [...]"` 단 한 문장. "concurrent/independent/simultaneous" 표현은 본문 0 hit. 저자의 "retrieval/search infrastructure 유지 대비 filesystem materialization" 차별화가 아키텍처 차이인지 프레이밍 차이인지가 쟁점.

**(c) BookRAG (arXiv:2512.03413, 2025-12) — 트리 인덱스 + 에이전트 플래너.**
"파일시스템 materialization이 본 논문의 핵심"이라면 BookIndex 같은 트리 인덱스 대비 왜 더 나은가를 실험/이론으로 보여야 하는데 어느 쪽도 없다.

**(d) WixQA 단일 벤치 — 검증이 아니라 시연.**
저자도 한계에서 "future work: evaluation across additional domains"로 인정.

**→ 진짜 방어 가능한 기여는 (e) Anthropic Skills API 생태계와의 결합 — "LLM 에이전트에게 파일 탐색 UI를 노출한다"는 interface 결정** 하나로 수렴. 8 skills/request · 200 files/skill · 30 MB/skill이라는 상업 API의 벽이 K=7, p=10, compact mode 설계를 강제한 **"provider-locked production engineering 사례"**로서의 가치.[^team-A]

[^team-A]: "method → system paper 재포지셔닝" 가설은 insight-analyst가 제기, deep-reader가 본문 근거(K=7, p=10, compact mode가 Skills API 제약에 맞춰 튜닝된 자기모순)로 동의. 팀 실행 로그 A.

### 2. 한계와 암묵적 가정

#### (1) [최강 비판] 53K input tokens — method 비용이 아니라 인프라 결함 비용

Ablation (b): 5 rounds(\$0.170)와 20 rounds(\$0.172) 차이 negligible. **에이전트가 몇 턴 안에 답을 찾는데도** input 53,487/query 소모 → Skills API가 매 call마다 **스킬 파일 내용을 반복 포함**한다는 간접 증거. 저자도 한계에서 "prompt caching이 완화책"임을 인정.

→ baseline 비교 공정성 균열(다른 baseline도 동일 조건이었는가 불명), "high-value queries 적합" 주장 약화(caching 켜면 RAPTOR 근접, quality 우위 유지 미검증). **본문 내부 증거만으로 도출 가능한 가장 강한 비판 카드** (deep-reader 1순위 지목).[^team-B]

[^team-B]: deep-reader가 §8 불명확 6종 중 비판적 인사이트 전환 1순위로 이 문제를 지목, Ablation (b)를 결정적 증거로 제시. 팀 실행 로그 B.

#### (2) [강 비판] Agentic baseline 공정성 — hierarchy 정보 비대칭

Agentic baseline은 BM25/Dense/Hybrid 3도구만, hierarchy 정보 없음. 즉 **"계층 구조 + 에이전트" vs "flat retrieval + 에이전트"** 비교일 뿐 "filesystem vs vector-based hierarchical navigation" 비교가 아니다. RAPTOR도 collapsed-tree only 비교로 tree-traversal 변형 제외. 진짜 공정 대상(LATTICE/BookRAG/NaviRAG/HCAG) 전부 누락.

→ 결과표가 지지하는 결론은 "계층 구조를 주면 에이전트가 더 잘한다"이지 "파일시스템 materialization이 더 낫다"가 아니다.

#### (3) [중강 비판] "O(log_p N) scalable" 허구 — API 제약이 먼저 터진다

200 files/skill 제약이 log 복잡도보다 먼저 터진다. 우회 수단 compact mode(파일 수 80% 감축)는 대규모에서 default가 되어야 하나 **품질 영향 미측정**. 더 나아가 **"200 files/skill" 수치 자체가 공식 platform.claude.com 문서에서 확인되지 않는다** (footnote 222에 URL·출처 attach 없음). 저자가 "not fundamental to framework design"이라면서도 모든 하이퍼파라미터(K=7, p=10, compact mode)가 이 제약에 맞춰 튜닝된 **자기모순**.[^team-D]

[^team-D]: context-scout §B.1·§B.4 확증, deep-reader §3.2/§5 재검증. 팀 실행 로그 D.

#### (4) [암묵적 가정] 벤더 종속과 표준 불안정성

가정 α(API 스펙 안정), β(수치 유효), γ(SKILL.md 포맷 유지).

**가정 γ 외부 리스크(사실 기반):** Mintlify가 2026-01경 install.md → skill.md 전환. 공식 블로그에서 "install.md didn't see much adoption" 자인. HN "6일 deprecate"는 1인 추정이나 단기 전환은 공식 확증. SKILL.md 생태계가 **현재 표준화 불안정 국면**에 있다는 검증 가능한 지표.

**가정 β 지표:** 위 (#3)의 수치 미확인.

**지표 약한 비판 참고:** vidarh의 HN 환원주의는 재검증 결과 1인 의견·반박 다수. "현재 구현 실효성 실증 부족" 수준으로 수위 하향.[^team-C]

[^team-C]: context-scout가 HN 3건 회신에서 초기 경보 (1)을 "과장이었다"고 명시 정정. vidarh 환원주의 무게 낮음 / Mintlify 중간(사실) / 26.1% 중간(논리거리). 팀 실행 로그 C.

#### (5) [방법론 가정] 요약 품질의 재귀적 오염

LLM 요약 re-embed → 다음 레벨 입력 구조는 저품질 요약 하나가 상위 구조 오염 가능. 저자는 이 실패 모드 robustness 실험 미제공.

관련 경험 데이터: Liu et al. (arXiv:2601.10338) "Agent skills in the wild" — 42,447 skills 분석, 취약점 26.1%(프롬프트 인젝션·데이터 유출 13.3%·권한 상승 11.8%·공급망). 단 Liu et al. 대상은 "커뮤니티 인간 기여 skill"이고 본 논문은 "LLM 자동 생성 summary"이므로 **수치 직접 투사 금지**. "skill artifact는 경험적 신뢰성 위험 기준선을 갖는다" 수준의 논거로만 활용. Corpus2Skill의 summary hallucination·factuality 검증 부재가 실질 쟁점.

#### (6) [평가 방법] LLM-as-judge의 self-preference 우려

Judge model 정체·프롬프트 세부 본문 미공개. Judge가 Claude라면 Sonnet 4.6 서빙 Corpus2Skill에 self-preference 가능성.

### 3. 적용 가능성과 조건

**도입 유효 조건 (모두 충족):**
1. 코퍼스가 명확한 topic cluster로 분해 가능 (FAQ·KB·제품 도큐·정책).
2. 엔터프라이즈 내부 사용 또는 high-value query.
3. 월 단위 이하 업데이트 (incremental 미지원).
4. Anthropic 인프라 lock-in 수용.

**주의:** Prompt caching 최우선, p=5~10 범위, 요약 결과 오프라인 QA, compact mode A/B 선행.

**대안 선호:** N<500 → flat Dense, 빈번 업데이트 → incremental 지원 엔진, 법률·의료 → GraphRAG류, provider-neutral → LATTICE/RAPTOR OSS.

### 4. 열린 질문
- **Q1.** Caching 켠 상태 Corpus2Skill vs RAPTOR 비용 곡선.
- **Q2.** LATTICE/NaviRAG/BookRAG 대비 WixQA head-to-head.
- **Q3.** Multi-topic 문서 지배 코퍼스에서 hard single-path 라우팅 실패율.
- **Q4.** Compact mode 품질 영향 scale별(N=10⁴·10⁵·10⁶).
- **Q5.** 컴파일러 LLM 교체 시 클러스터 품질 열화.
- **Q6.** 네비 파일에 요약 대신 원문 일부 사용 시 routing 품질.

---

## 외부 맥락 요약

### 학계 반응
공개 5일차 공식 인용 없음, Semantic Scholar 인덱싱 전, OpenReview 없음.

### 경쟁·선행 연구 맥락

| 연구 | 시점 | 핵심 아이디어 | 본 논문과의 거리 |
|---|---|---|---|
| LATTICE (2510.13217) | 2025-10 | Semantic tree + LLM traverse | **거의 동일**, 6개월 선행, **완전 누락** |
| BookRAG (2512.03413) | 2025-12 | BookIndex + IFT planner | 트리 인덱스, 직접 비교 부재 |
| A-RAG (2602.03442) | 2026-02 | 3종 도구 ReAct | 다른 축 |
| Agent Skills 서베이 (2602.12430) | 2026-02 | SKILL.md 4축 | 개념적 토대 |
| NaviRAG (2604.12766) | 2026-04-14 | hierarchical + agent navigation | **2일 차 concurrent**, 1문장 처리 |
| HiRAG | — | query-time graph retrieval | 본 논문이 "agent-driven file navigation과 다름"으로 명시 선긋기 |
| SPD-RAG | — | 문서당 sub-agent multi-agent | 대비 안정 |
| **Corpus2Skill** | 2026-04-16 | 파일시스템 + Anthropic Skills API | — |
| KG-RAG Enterprise (2604.14220) | 2026-04 | 엔터프라이즈 KG-RAG | 같은 주 공개 |

저자들이 구조-aware + agentic RAG 범주(13편)는 폭넓게 서치했지만 LLM-guided tree retrieval 각도는 놓쳤다. HiRAG에는 "query-time graph retrieval이라 본 논문과 다르다"며 선명히 선긋기 하면서 LATTICE에는 같은 논리를 적용할 기회조차 없었다는 대조가 사각지대를 선명히 한다.

### 실무 커뮤니티 반응

- **HN 46871173 "Agent Skills":** vidarh 환원주의 초기 발언 있었으나 ashdksnndck·smithkl42·mhalle 등 **반박 다수 합의**. 파일시스템 은유 부정 담론은 HN 주류 아님.
- **HN 46723183 + Mintlify 공식 블로그:** install.md → skill.md 전환 사실. SKILL.md 표준 불안정 검증 리스크.
- **Liu et al. (2601.10338) 26.1%:** LLM-dependent skill artifact 경험적 위험 기준선. 단 본 논문 직접 대입은 과추론.

### 재현성
- 공식 코드 미확인 (저자 Yiqun Sun GitHub dukesun99에 본 논문 연결 repo 없음).
- WixQA 공개 데이터(MIT)로 데이터 재현성만 확보.
- 클러스터링 세부·요약 프롬프트·skill 포맷·시스템 프롬프트·judge 설정 부분 공개로 **파이프라인 재현 불가능**.
- **"벤더 종속 + 단일 도메인 + 코드 미공개" 3중 재현성 리스크**.

---

## 참고자료

- 논문: https://arxiv.org/abs/2604.14572
- 미인용 선행: LATTICE https://arxiv.org/abs/2510.13217
- 2일 차 concurrent: NaviRAG https://arxiv.org/abs/2604.12766
- Baseline: RAPTOR https://arxiv.org/abs/2401.18059 · https://github.com/parthsarthi03/raptor
- 구조 경쟁: GraphRAG https://microsoft.github.io/graphrag/ · BookRAG https://arxiv.org/abs/2512.03413
- 에이전틱: A-RAG https://arxiv.org/abs/2602.03442
- 벤치: WixQA https://arxiv.org/abs/2505.08643 · https://huggingface.co/datasets/Wix/WixQA
- 생태계: Agent Skills 서베이 https://arxiv.org/abs/2602.12430 · Anthropic Agent Skills https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills
- SKILL.md 사건: Mintlify skill.md 블로그 https://www.mintlify.com/blog/skill-md · docs https://www.mintlify.com/docs/ai/skillmd · https://github.com/mintlify/install-md
- Skill 취약점 실증: Liu et al. https://arxiv.org/abs/2601.10338
- 커뮤니티: HN 46871173, 46723183, 46351787
- 저자: Yiqun Sun https://www.linkedin.com/in/dukesun99/ · https://www.comp.nus.edu.sg/~sunyq/

---

## 미수집 영역

- 저자 공식 코드/데모 (2026-04-21 기준 미확인)
- OpenReview / conference 제출 흔적
- Semantic Scholar / Google Scholar 인용 지표 (공개 5일차)
- X/Twitter 반응 (로그인 벽)
- 한국어 커뮤니티 직접 반응
- Judge model 세부 (프롬프트·모델명)
- "200 files/skill" 수치 공식 문서 출처
- Mintlify install.md → skill.md 전환 정확한 일수
- Concurrent NaviRAG/동시기 KG-RAG WixQA head-to-head

---

## 팀 실행 로그

본 보고서는 **논문 2604.14572 리서치가 에이전트 팀 모드(TeamCreate)로 전환된 첫 실전 검증**에서 작성되었다.

### A. SendMessage 발생 내역 (총 8건)

| # | 방향 | 성격 | 해석 영향 |
|---|---|---|---|
| 1 | deep-reader → context-scout | 조기 조사 요청 (LATTICE·RAPTOR·Skills API 발견) | context-scout §2.1에서 LATTICE 승격. 없었으면 누락이 "미관측"으로 남았을 것 |
| 2 | context-scout → insight-analyst | 상충 경보 3건(skill 회의론·SKILL.md 불안정·LATTICE) | idle 큐 보관 → §4 핵심 논거 편입 |
| 3 | insight-analyst → deep-reader | LATTICE novelty 순위 + 불명확 전환 후보 질의 | "(b)>(c)>>(a)" 확정, system paper 재포지셔닝 확정, §1 뼈대 |
| 4 | insight-analyst → context-scout | HN 비판 3축 신뢰도 등급 질의 | (6·7)을 유도한 경로 |
| 5 | context-scout → insight-analyst | 추가 상충 3건(200 files 수치·NaviRAG concurrent·BookRAG) | §1 novelty 분해 표 NaviRAG·BookRAG 편입, §2 #3 수치 불일치 증거 |
| 6 | context-scout → insight-analyst | 3축 비판 재정리 (LATTICE grep 확정, NaviRAG 1문장 확인) | §1 (b)(c) 논증 재작성, concurrent 간과 프레임 폐기 |
| 7 | context-scout → insight-analyst | HN 3건 회신 + **자기 정정** (vidarh 과장 인정) | §4 프레이밍 수위 하향, Mintlify 공식 링크 편입, 26.1% 직접 투사 금지 |
| 8 | context-scout → insight-analyst | "문헌 지형도 사각지대" 최종 문구 + HiRAG/SPD-RAG 인용 | §1 footnote에 그대로 인용, 비교표 행 추가 |

### B. 팀원 간 조정 흔적

1. **LATTICE 누락 발견 4단 협력:** deep-reader 이상감지(#1) → context-scout arXiv 확정(#2) → deep-reader grep 0 hit 재확정 → context-scout "13편 중 LATTICE만 누락" + 사각지대 프레이밍 격상(#6·#8). 단독 에이전트로는 도달 어려운 결론.

2. **method → system paper 재포지셔닝:** insight-analyst 가설(#3) → deep-reader 본문 근거로 확정. 가설 제기와 본문 확인 분리.

3. **context-scout 명시적 자기 정정(#7):** vidarh 환원주의 과장을 스스로 인정. 단방향 handoff였다면 과장 프레이밍 잔존. **팀 모드에서만 가능한 자기 수정 루프**.

4. **주 비판축 자율적 이동:** 커뮤니티 시그널 → 본문·공식문서·실증연구로 이동. 정보 원천 신뢰도 자동 조율.

### C. 권한 차단 발생 여부

- 외부 도구 차단: 없음.
- **insight-analyst Write 차단: 실제 발생.** tool_use_error "Subagents should return findings as text, not write report files"가 2회 반환. insight-analyst.md의 `{paper_dir}/report.md` 출력 프로토콜과 충돌하는 하네스-레벨 규약 존재 의심. team-lead의 정정 지시에도 도구 차단 유지. **Bash heredoc 우회로 최종 저장.** Phase 7 하네스 개선 대상.
- context-scout 사전 선언 자원 접근 한계(X 로그인 벽·OpenReview 부재·저자 홈페이지 403·SS 미인덱싱)는 미수집 영역에 반영.

### D. 팀 모드 검증 결과

| 체크포인트 | 결과 | 비고 |
|---|---|---|
| 1. TeamCreate 정상 실행 | PASS | 3명 config 등록, 역할별 prompt 주입 |
| 2. SendMessage ≥1건 발생 | PASS (8건) | 6개 pair가 해석에 실제 영향 |
| 3. 권한 차단 실시간 피드백 | PASS (부분) | insight-analyst Write 차단 시 team-lead 즉시 정정, 단 에이전트 반응 지연 관찰 |
| 4. report.md에 조정 흔적 | PASS | §1 `[^02]`, §1 `[^team-A]`, §2-#1 `[^team-B]`, §2-#3 `[^team-D]`, §2-#4 `[^team-C]`, 본 섹션 A·B |

### 서브에이전트 모드 대비 팀 모드의 실측 가치

- (i) 실시간 상호 조사 요청(#1)이 context-scout §2 구조 변경.
- (ii) 가설 검증 협력(#3)이 단일 에이전트 자기 확증 회피.
- (iii) 정보 원천 자율 업그레이드(#5·#6)가 주 비판축을 커뮤니티 시그널에서 본문·공식문서로 이동.
- (iv) 명시적 자기 정정 루프(#7)가 과장 프레이밍을 실시간 수정.
- (v) 단계적 프레이밍 합의(#6 → #8)로 "concurrent 간과" 부정확 프레임 폐기, "문헌 지형도 사각지대"로 수렴.

다섯 효과 모두 순차 파이프라인에서는 불가능. 단방향 handoff였다면 LATTICE/NaviRAG/200-files/HN 정정 중 어느 하나도 최종 보고서까지 정확한 프레임으로 도달하지 못했을 가능성이 높다.

**동시에 팀 모드 운영상 문제점 1건:** 에이전트가 시스템 안내(tool_use_error)를 하네스 규약으로 착오 해석해 최종 저장을 지연. team-lead의 재지시에도 Write 도구 차단이 유지되어 Bash 우회 필요. Phase 7 하네스 개선 대상.
