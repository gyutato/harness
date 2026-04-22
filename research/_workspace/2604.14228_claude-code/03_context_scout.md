# 외부 맥락: Dive into Claude Code (arXiv:2604.14228)

조사 시점: 2026-04-21
조사자: context-scout (에이전트 팀 `paper-2604-14228`)

본 문서는 논문 **본문 내용을 해석하지 않는다**. 본문 독해는 `02_deep_reader.md`를 참조. 여기서는 논문을 둘러싼 외부 생태계(배경·선행/후속 연구·공식 자산·커뮤니티 반응·재현성)만 다룬다.

---

## 1. 분야 배경 (이 논문 이해에 필요한 사전 지식)

### 1.1 "Agent Harness" 개념의 등장

2026년 현재, AI 에이전트 업계에서 **"Agent = Model + Harness"** 라는 분리 관점이 사실상 표준이 됐다. 하네스는 모델 외부의 모든 것 — 도구 실행, 상태 관리, 컨텍스트 관리, 권한 체크, 메모리, 피드백 루프 — 을 가리킨다.

- **대표 정의**: "A harness is every piece of code, configuration, and execution logic that isn't the model itself" — Firecrawl, ["What Is an Agent Harness?"](https://www.firecrawl.dev/blog/what-is-an-agent-harness)
- **LangChain의 해부**: ["The Anatomy of an Agent Harness"](https://blog.langchain.com/the-anatomy-of-an-agent-harness/) — 컴포넌트 도식화
- **Martin Fowler 글**: ["Harness engineering for coding agent users"](https://martinfowler.com/articles/harness-engineering.html) — 엔지니어링 관점의 정착
- **Philipp Schmid**: ["The importance of Agent Harness in 2026"](https://www.philschmid.de/agent-harness-2026) — 프로덕션 관점

이 용어의 확산 시점은 2025년 후반~2026년 초로 보이며, 본 논문의 주제(Claude Code 하네스 해부)는 이 담론의 정점에 위치한다.

### 1.2 AI 코딩 에이전트 경쟁 구도 (2026-04 기준)

| 시스템 | 제공자 | 특성 | 비고 |
|---|---|---|---|
| Claude Code | Anthropic | 터미널 네이티브, 상용 프로덕션 | 본 논문 주제 |
| Cursor | Anysphere | IDE 통합 에이전트 | 점유율 최상위 |
| Codex | OpenAI | API + CLI (oh-my-codex) | |
| Aider | 오픈소스 | 터미널 기반, Git 인지 | 초창기 |
| OpenHands | 오픈소스(AI Software Developer) | ICLR 2025 논문 존재 | 연구 기반 |
| Devin | Cognition | 원격 자율 에이전트 | |
| OPENDEV | 오픈소스 (arXiv 2603.05344) | 터미널 네이티브, Rust | Claude Code와 가장 유사한 연구 대상 |
| OpenClaw | openclaw/openclaw (Peter Steinberger 창립) | 개인 AI 어시스턴트, Telegram/Discord/WhatsApp 연동, WhatsApp bot으로 시작 | 본 논문이 비교 대상으로 선정. **100K GitHub stars 3개월 달성** |
| Claude Managed Agents | Anthropic | 클라우드 관리형 에이전트 (2026-04-08 출시, public beta) | 논문이 "Martin et al. 2026"으로 인용할 가능성 |

**OpenClaw 추가 정보**: 창립자 [Peter Steinberger](https://steipete.me/posts/2026/openclaw)(오스트리아 개발자). 2025-11 Clawdbot로 시작 → 2026-01-27 "Moltbot"으로 개명(Anthropic 상표권 이의) → 3일 후 "OpenClaw"로 재개명. 2026-02-15 Steinberger OpenAI 합류 발표 (Sam Altman "genius" 호칭). 3개월만에 100K stars 달성 (GitHub 역대 최속 성장 에이전트 프레임워크 중 하나).

**Claude Managed Agents 추가 정보**: [Anthropic "Scaling Managed Agents"](https://www.anthropic.com/engineering/managed-agents) (2026-04-08). 샌드박스 코드 실행, 체크포인팅, 자격 증명 관리, 스코프 권한, end-to-end 추적 기능. Claude Code와 차별화된 프로덕션 워크로드 대상. **논문이 "§11.6 / §12.3"에서 다룬다면 공식 출시 후 불과 6일 만에 논문에 반영**한 것

출처: [OpenClaw vs Claude Code Channels vs Managed Agents](https://www.mindstudio.ai/blog/openclaw-vs-claude-code-channels-vs-managed-agents-2026), [OpenClaw vs Claude Code vs Hermes vs MCPlato](https://mcplato.com/en/blog/ai-agent-harness-comparison-2026/)

### 1.3 Claude Code 소스 누출 사건 (연구 배경의 핵심 맥락)

**2026-03-31**, Anthropic이 Claude Code v2.1.88 npm 패키지에 `.map` 파일(약 59.8MB)을 실수로 포함해 배포했다. 이 소스 맵을 통해 읽을 수 있는 4,600+ TypeScript/TSX 파일이 외부에 노출됐다.

- **사건 분석 (영문)**: Alex Kim, ["The Claude Code Source Leak: fake tools, frustration regexes, undercover mode, and more"](https://alex000kim.com/posts/2026-03-31-claude-code-source-leak/) (2026-03-31)
- **사건 분석 (한국어)**: Jonas Kim, ["Claude Code 내부 아키텍처 분석"](https://bits-bytes-nn.github.io/insights/agentic-ai/2026/03/31/claude-code-source-map-leak-analysis.html)
- **HN 토론**: [Claude Code Unpacked: A visual guide](https://news.ycombinator.com/item?id=47597085) (1,128 points, 401 comments) — 소스 누출 후 커뮤니티 대응 대표 스레드

**이 사건의 연구적 함의**: 본 논문이 주장하는 "publicly available TypeScript source code"는 **의도된 오픈소스가 아니라 사고로 유출된 소스**를 포함한다. 논문 본문이 이를 명시적으로 다뤘는지는 `02_deep_reader.md`에서 확인 필요. 미공개 시 **연구 윤리/저작권 논란의 소지**가 존재한다.

---

## 2. 핵심 선행·관련 연구

### 2.1 Building Effective AI Coding Agents for the Terminal (arXiv:2603.05344) — **가장 가까운 경쟁 논문**
- 저자: Nghi D. Q. Bui (소속 미확인)
- 제출: 2026-03-05 (v1) → 2026-03-13 (v3). **본 논문보다 약 1개월 선행**
- 대상 시스템: OPENDEV (Rust 기반 자체 구현 터미널 코딩 에이전트)
- 구조: ReAct 6단계 루프(pre-check+compaction, thinking, self-critique, action, tool execution, post-processing) + 7개 지원 서브시스템
- 키워드 100% 겹침: **Scaffolding / Harness / Context Engineering**
- 관계: Dive into Claude Code의 "7 컴포넌트 + 5 서브시스템" 모델과 사실상 동형. 본 논문이 이를 인용·대조했는지 `deep-reader`가 확인 필요. 미인용 시 선행연구 누락 비판 가능
- URL: https://arxiv.org/abs/2603.05344

### 2.2 OpenHands: An Open Platform for AI Software Developers (ICLR 2025)
- Dive-into-Claude-Code 공식 저장소의 `related-resources.md`에 "Academic Papers" 섹션에 수록
- 관계: 오픈소스 코딩 에이전트 프레임워크의 대표 주자. Claude Code의 프로덕션 대비판 비교 대상
- 출처: [VILA-Lab/Dive-into-Claude-Code/docs/related-resources.md](https://github.com/VILA-Lab/Dive-into-Claude-Code/blob/main/docs/related-resources.md)

### 2.3 SWE-Agent: Agent-Computer Interfaces (NeurIPS 2024)
- 본 논문 공식 관련 자료 목록에 수록
- 관계: 도구 인터페이스 설계 관점의 선행. Claude Code의 "40+ discrete tools" 설계 사상의 학술적 뿌리

### 2.4 Decoding the Configuration of AI Coding Agents (arXiv:2511.09268)
- 저자: Helio Victor F. Santos, Vitor Costa, Joao Eduardo Montandon, Marco Tulio Valente
- 제출: 2025-11-12
- 내용: 328개 공개 Claude Code 프로젝트의 구성 파일(CLAUDE.md 등)을 분석. 아키텍처 명세의 중요성 실증
- 관계: Dive into Claude Code가 "내부 아키텍처"를 본다면, 이 논문은 "사용자 구성"을 본 보완적 연구
- URL: https://arxiv.org/abs/2511.09268

### 2.5 On the Use of Agentic Coding Manifests (arXiv, 연도 미확인)
- 253개 CLAUDE.md 구조적 패턴 분석
- 본 논문 공식 관련 자료 목록 수록
- 관계: Claude Code 주변 "메모리·스킬" 생태계의 정량 분석

### 2.6 Anthropic 공식 문서 (선행이자 1차 자료)
- **Claude Code auto mode (2026-03-25)**: https://www.anthropic.com/engineering/claude-code-auto-mode
  - 본 논문의 주요 수치 다수의 원출처: 93% 승인률, 0.4% FPR (end-to-end), 17% FNR (on 52 real overeager actions), 10K real traffic + 1K synthetic exfiltration
  - **중요**: 이 블로그는 "서베이"가 아니라 **데이터셋 평가**. 서베이(132명)와 혼용하지 말 것
  - 저자: **John Hughes** (논문에서 "Hughes 2026"으로 인용된 것과 일치)
- **How AI Is Transforming Work at Anthropic (2025-09)**: https://www.anthropic.com/research/how-ai-is-transforming-work-at-anthropic
  - 132명 엔지니어/연구자 서베이 + 53명 심층 인터뷰 (2025-08 조사)
  - **27% 수치의 원출처**: "Staff estimate that 27% of Claude assisted work would not have been done before" (논문이 "capability amplification"의 경험적 근거로 반복 인용)
  - 저자: **Saffron Huang** 주도 (논문에서 "Huang et al. 2025"로 인용)

### 2.7 기타 논문 내 핵심 인용 (검증 결과)
- **Shen & Tamkin 2026** — [How AI Impacts Skill Formation (arXiv:2601.20245, 2026-01-29)](https://r.jordan.im/download/language-models/shen2026.pdf)
  - 저자: **Judy Hanwen Shen** (Anthropic), Alex Tamkin (Anthropic)
  - **중요**: 이 Shen은 **Judy Hanwen Shen (Anthropic)**이며 **논문 공저자인 Zhiqiang Shen(MBZUAI VILA-Lab)과 동명이인**. 본 논문의 "17% comprehension 하락" 인용은 **자기 인용이 아니다**. 순환 인용 의혹 해소됨
  - 연구 설계: 52명 소프트웨어 엔지니어 RCT, 비동기 프로그래밍 라이브러리 학습, AI 그룹 vs 통제 그룹. AI 보조 참가자는 동일 시간에 태스크 완료했으나 후속 comprehension 퀴즈에서 17% 하락(50% vs 67%). 디버깅 능력에서 가장 큰 낙폭. **수동 위임(passive delegation)**이 원인이며 AI 사용 자체가 문제는 아님
  - **논문이 Claude Code 비판으로 전용한 것의 아이러니**: Anthropic이 자체적으로 경험적 반증을 낸 연구
- **McCain et al. 2026**: "measuring agent autonomy" 연구 — https://www.anthropic.com/research/measuring-agent-autonomy
  - 저자: Miles McCain 포함 Anthropic 팀
  - **20%→40% auto-approve 수치 출처**: "Among new users, roughly 20% of sessions use full auto-approve, which increases to over 40% as users gain experience (<50 → 750 세션)"
  - 논문의 "graduated trust spectrum" 실증 근거
- **Dworken & Weller-Davies 2025**: ["Making Claude Code more secure and autonomous" Anthropic 엔지니어링 블로그](https://www.anthropic.com/engineering/claude-code-sandboxing)
  - 저자: **David Dworken, Oliver Weller-Davies** (Anthropic)
  - **84% 수치 출처**: "In Anthropic's internal usage, sandboxing safely reduces permission prompts by 84%"
  - GitHub: https://github.com/anthropic-experimental/sandbox-runtime
- **He et al. 2025 — Cursor 40.7% 복잡도 증가**: [arXiv:2511.04427 "Speed at the Cost of Quality: How Cursor AI Increases Short-Term Velocity and Long-Term Complexity in Open-Source Projects"](https://arxiv.org/abs/2511.04427)
  - 실제 저자명은 검색상 "He et al."로 인용되며 논문은 CMU 연구진으로 확인됨
  - 설계: **807개 GitHub 저장소 + 1,380개 대조군(propensity score matching)** 분석. 2024-01 ~ 2025-08 관측
  - 주 결과: 복잡도 40% 이상 증가, velocity는 일시적. **정확 40.7% 수치와 부합**
  - **주의**: 이 연구는 **Cursor** 대상이지 **Claude Code** 대상이 아니다. 논문이 이를 Claude Code 비판으로 전용한다면 일반화의 비약 가능성
- **Becker et al. 2025 — AI tools 19% 느림 RCT**: [arXiv:2507.09089 "Measuring the Impact of Early-2025 AI on Experienced Open-Source Developer Productivity"](https://arxiv.org/abs/2507.09089)
  - 저자: **Joel Becker 외** (METR)
  - 설계: **16명 경험 많은 개발자, 246 태스크, 5년 이상 프로젝트 경험**, randomized. AI 도구는 Cursor Pro + Claude 3.5/3.7 Sonnet
  - 주 결과: AI 허용 시 **19% 더 오래 걸림**(체감은 20% 빨라졌다고 응답). 예상과 정반대
  - **논문 맥락 주의**: 피실험자 16명 표본은 작음. Claude Code 정식 릴리스 이전(2025년 초) AI 도구 기반
- **Rak 2025 — 25% 주니어 채용 감소**: 정확 매치 저자명 "Rak" 미확인
  - 가장 가까운 후보: [Stanford Brynjolfsson, Chandar, Chen, "Canaries in the Coal Mine? Six Facts about the Recent Employment Effects of Artificial Intelligence" (2025-08)](https://digitaleconomy.stanford.edu/wp-content/uploads/2025/08/Canaries_BrynjolfssonChandarChen.pdf)
  - 이 연구: 22-25세 소프트웨어 개발자 고용 2022-late peak 대비 **거의 20% 감소**(정확히 25%는 아님)
  - **의혹**: 논문의 "Rak 2025 — 25%" 인용은 잘못된 인용 또는 별도 출처일 가능성. `deep-reader`가 원문 각주 재확인 필요
- **1.6% / 98.4% 수치의 출처**: **커뮤니티 리버스 엔지니어링이 먼저**
  - 주 관찰치: 512,000 lines 중 약 8,000 lines만 AI 모델 호출 로직 (≈1.6%). Query Engine(46K lines), Tool System(29K lines)이 최대 모듈
  - **초기 독립 분석**: Akshay Pachaar (2026-04, X/Twitter): "Researchers from UCL reverse-engineered the leaked Claude source... Only 1.6% of the codebase is AI decision logic. The other 98.4% is operational infrastructure" → https://x.com/akshay_pachaar/status/2045404494641733962
  - **중요 단서**: 이 X 포스트는 논문을 "**UCL researchers**"로 소개. 이는 `02_deep_reader` 영역에서 확인해야 할 핵심 — **논문에 UCL 저자가 실제로 있는지**
  - 중국어 분석: [Odaily: "What Exactly Are AI Agents Doing?"](https://www.odaily.news/en/post/5210047) — 동일 수치 정리
  - 본 논문이 "community analysis of the extracted source"로 출처를 **익명화**했으나, 이는 관측 가능한 표준 통계라 재현 쉬움. 원출처 명시가 아쉽지만 수치 자체는 중첩 검증됨

---

## 3. 후속·경쟁 연구

### 3.1 현재 시점 인용 수: 미확인
- 제출 1주 경과 (2026-04-14 → 2026-04-21). 공식 피인용 집계 표시 없음 (Google Scholar에서도 반영 미적용)
- "citing"/follow-up paper 검색 결과 0건

### 3.2 비공식 후속 기여 (동일 문제 영역)
- **ComeOnOliver/claude-code-analysis** — 소스 트리/모듈 경계 리버스 엔지니어링 (GitHub)
- **alejandrobalderas/claude-code-from-source** — 18-chapter, 400쪽 기술 매뉴얼
- **liuup/claude-code-analysis** — 중국어 분석 (startup flow, query loops, MCP)
- **sanbuphy/claude-code-source-code** — 다국어 (EN/JA/KO/ZH) 10 도메인, 75 리포트
- **cablate/claude-code-research** — Agent SDK 내부 조사
- **Yuyz0112/claude-code-reverse** — LLM 상호작용 시각화 도구
- **AgiFlow/claude-code-prompt-analysis** — 5세션 API 로그 실증
- 모두 출처: [VILA-Lab/Dive-into-Claude-Code/docs/related-resources.md](https://github.com/VILA-Lab/Dive-into-Claude-Code/blob/main/docs/related-resources.md)

### 3.3 오픈소스 재구현체 (Claude Code Harness 수준의 복제)
- **instructkr/claw-code** (Sigrid Jin, 한국 개발자) — **Rust(72.9%)+Python(27.1%) clean-room 재작성**. 누출 당일(2026-03-31) 4시부터 구현 시작 → 2시간만에 5만 스타, 역대 최속 성장 리포 중 하나. 출처: [WaveSpeedAI](https://wavespeed.ai/blog/posts/what-is-claw-code/), [Cybernews](https://cybernews.com/tech/claude-code-leak-spawns-fastest-github-repo/)
- **ultraworkers/claw-code** — 위 fork. 512K TS → 20K Rust로 축약 주장
- **777genius/claude-code-working** — 리버스 엔지니어 CLI, 450+ chunk files
- **T-Lab-CUHKSZ/claude-code** — 중국 고등학교-대학 연합 연구 빌드 포크
- **ruvnet/open-claude-code** — 자동 디컴파일 리빌드, 903+ 테스트, 25 도구
- **Enderfga/openclaw-claude-code** — OpenClaw 플러그인으로 Claude/Codex/Gemini/Cursor 지원
- **memaxo/claude_code_re** — minified bundle 디오브퍼스케이션

### 3.4 경쟁 관점의 시스템 분석 (학계 밖 자료지만 비중 있음)
- Han HELOIR YAN: "The moat is the harness, not the model" — 본 논문의 주제문과 거의 동일 주장을 먼저 한 독립 분석
- Haseeb Qureshi: 모듈 분석 + cross-agent 아키텍처 비교
- Engineer's Codex: 46K 라인 query engine, modular system prompt 실증

---

## 4. 공식 자산

| 자산 | URL | 상태 (2026-04-21) |
|---|---|---|
| arXiv 논문 | https://arxiv.org/abs/2604.14228 | v1, 2026-04-14 제출. cs.SE/cs.AI/cs.CL/cs.LG |
| HTML 렌더링 | https://arxiv.org/html/2604.14228v1 | 정상 |
| PDF (arXiv) | https://arxiv.org/pdf/2604.14228 | 정상 |
| PDF (GitHub) | https://github.com/VILA-Lab/Dive-into-Claude-Code/blob/main/paper/Dive_into_Claude_Code.pdf | 저장소 내 동봉 |
| GitHub 저장소 | https://github.com/VILA-Lab/Dive-into-Claude-Code | **482 stars / 60 forks / 0 open issues** (1주차치고 활발). 라이선스 CC BY-NC-SA 4.0 |
| HuggingFace Papers | https://huggingface.co/papers/2604.14228 | 페이지 존재 (수치 확인 실패 — 페이지 본문 파싱 불가) |
| 독립 리뷰 | https://arxiviq.substack.com/p/dive-into-claude-code-the-design | Substack "arxiviq" — 외부 독립 리뷰 |

### 저자 정보 (전원 MBZUAI 확정, 2026-04-21 재조사)

**4명 저자 전원 MBZUAI 소속 확정**. 1차 소스: [Zhiqiang Shen Group 공식 페이지](https://zhiqiangshen.com/projects/group/index.html):

| 저자 | 역할 | 시작 | 비고 |
|---|---|---|---|
| Jiacheng Liu | ML PhD student | 2024- | Shen 그룹 박사과정 |
| Xiaohan Zhao | ML PhD student | 2024- | Shen 그룹 박사과정. 연구 관심사: efficient DL + adversarial attacks. 기존 VILA-Lab 출판물 "Open CaptchaWorld" 공저 |
| Xinyi Shang | Visiting student / Research Assistant | 2025- | 방문 연구원 |
| Zhiqiang Shen | Assistant Professor (지도교수) | — | MBZUAI ML Dept, VILA-Lab 운영 |

- **Zhiqiang Shen 상세**: MBZUAI Assistant Professor of ML. Ph.D. Fudan / UIUC joint, postdoc CMU CyLab. 연구 분야 공식 프로필은 "efficient deep learning, CV, quantization, distillation" 중심이며 **AI 에이전트/Claude Code는 프로필에 명시되지 않음** (=최근 주제 확장). 출처: [MBZUAI 공식 프로필](https://mbzuai.ac.ae/study/faculty/zhiqiang-shen/), X: [@szq0214](https://x.com/szq0214)
- **VILA-Lab (Vision/Language, learning and acceleration group)**: MBZUAI 기반. 128 followers. GitHub org: https://github.com/VILA-Lab. 주요 레포: ATLAS(982 stars), Awesome-DLMs(965), OD3(ICLR 2026), Elastic-Cache(ICLR 2026), SRe2L(NeurIPS 2023). **주 연구 테마는 비전-언어 모델·효율화**였으며, Dive-into-Claude-Code는 현재 org 내 2위 스타 수이자 **에이전트 시스템 분야 첫 진출**에 해당

### "UCL researchers" 단서 — **외부 소개자의 오류**로 판정
- [Akshay Pachaar X 포스트](https://x.com/akshay_pachaar/status/2045404494641733962)가 본 논문을 "Researchers from UCL reverse-engineered the leaked Claude source"로 소개했지만, **4명 저자 전원 MBZUAI 소속임이 Shen 그룹 공식 페이지에서 확인됨**. UCL 관련 저자 **없음**
- 판정: 외부 소개자의 **오류**이며 논문 자체의 귀속 문제 아님. HF Papers 추출도 같은 출처를 인용해 오류가 전파된 것으로 추정
- **공정성 비판 접근선 소멸**: 논문·저장소 어디에도 UCL 표기가 없으므로 "부정확 표기" 지적은 성립 안 함 — deep-reader가 제안한 비판 각도는 사용 불가

### 공식 저장소 구성 (상세 재조사 2026-04-21)
```
.github/workflows/   (CI)
assets/              (아키텍처 다이어그램 6종: 메인 아키텍처, 계층 분해, 권한 흐름, 컨텍스트 조립, 서브에이전트, 세션 지속성·컴팩션)
docs/
  ├─ architecture.md        (아키텍처 딥다이브 — **개념적 설명만; 소스 파일 경로·라인 번호·코드 스니펫 부재**)
  ├─ build-your-own-agent.md  (설계 가이드)
  └─ related-resources.md    (50+ 커뮤니티 큐레이션)
paper/
  └─ Dive_into_Claude_Code.pdf  (arXiv PDF 동봉)
CITATION.cff         (**authors 필드: "To be updated" 플레이스홀더 — 미완성**)
LICENSE              (CC BY-NC-SA 4.0; 비상업·동일조건변경허락)
README.md            (종합 가이드; "no proprietary source", "original pseudocode")
```

**재현성·투명성 부재 요소** (A5 쟁점 직접 응답 증거):
- 루트에 `src/`, `data/`, `scripts/`, `analysis/` 등 **재현 아티팩트 폴더 없음**
- README와 `docs/architecture.md` 어디에도 **"Methodology / Data Sources / How We Obtained the Source / Reproducibility" 섹션 없음**
- `architecture.md`는 주요 수치("~1.6% AI decision logic, 98.4% infrastructure") 측정 방법·스크립트·원 데이터 제시 안 함
- `queryLoop` in `query.ts`, `assembleToolPool` 같은 컴포넌트 참조도 **실제 구현 파일 경로/라인 포인터 없음**
- 개념적 서술("model call → tool dispatch → result collection → repeat")만 있고 **실제 TypeScript 스니펫·의사 코드 부재**
- CITATION.cff가 **"To be updated" 플레이스홀더** — 논문 제출 1주 경과에도 미완성

**A5 강화 판정 (deep-reader §7.2 A5 응답)**:
- 저장소는 "재현 가능한 연구 산출물"이 아니라 **"정리된 아키텍처 문서집 + 논문 PDF + 50+ 커뮤니티 큐레이션"**
- 상업 시스템(Claude Code v2.1.88)의 비공개 소스 분석에 대해 **5중 투명성 부족** 관찰: (a) 소스 획득 경로 불투명, (b) 추출 스크립트 비공개, (c) 원 데이터·로그 비공개, (d) 소스 파일 경로 인용 부재, (e) CITATION.cff 미완성
- 결과: **제3자 독립 검증 불가능**. "512K 라인", "4,600+ 파일", "1.6%/98.4%", "27 hooks", "40+ tools" 같은 숫자를 재계산할 수 없는 구조
- 누출 사건(2026-03-31)과 논문 제출(2026-04-14) 간격 **2주** + 저장소의 "no proprietary source" 문구를 교차하면, **분석 대상이 유출본일 가능성이 높은데 공식 저장소는 이 사실을 명시하지 않음**
- **정상적 리버스 엔지니어링 학술 연구가 갖춰야 할 것** 중 부재한 요소: 소스 획득 출처 명시 (npm 공식 vs 유출본), 추출 스크립트(소스맵 파싱/AST 분석), 원시 카운트 데이터(파일별 LoC 집계), 파일 경로↔논문 주장 매핑, 분석 일자·정확한 버전 커밋 해시
- **단**: 커뮤니티의 다른 리버스 엔지니어링 프로젝트(ComeOnOliver/claude-code-analysis, alejandrobalderas/claude-code-from-source 등)는 **소스 트리·모듈 경계 구체 참조 제공**. 논문 저장소가 오히려 덜 투명
- 종합: **사기·날조로 볼 근거는 없음** (수치 다수가 타 분석과 일치, 인용 9/10 실존, CVE 실존) — **그러나 "source code as evidence" 학술 규범 미달**. A5는 "방법론 기재 미흡" 수준에서 "재현성 구조적 부재 + 메타데이터 미완성"으로 강화 가능

### 저자 발표
- 공식 발표 영상: **미확인** (YouTube "The Architecture of Claude Code: Kernel and the Harness"가 검색됐으나 논문 저자 발표인지 제3자 해설인지 메타데이터 확인 불가)

---

## 5. 커뮤니티 반응

### 5.1 학계 반응 (인용·후속 논의)
- **공식 피인용**: 2026-04-21 시점 0건 (제출 1주차). Google Scholar/Semantic Scholar 반영 미적용
- **관련 학술 문헌과의 위치**: 동일 주제의 선행 논문(OPENDEV, 2603.05344)가 존재하지만 본 논문은 Claude Code라는 **상용 시스템 개별 해부**에 차별점이 있다
- **재현 연구**: 없음 (논문 자체가 리버스 엔지니어링이므로 "재현"의 대상 애매)
- **UCL 소속 단서**: [Akshay Pachaar X 포스트](https://x.com/akshay_pachaar/status/2045404494641733962) (2026-04 초)가 본 논문을 "**UCL researchers**"로 소개함. HuggingFace Papers 추출 결과에서도 UCL 공동 저자 가능성이 언급됨. Zhiqiang Shen은 MBZUAI이지만, 제1저자 Jiacheng Liu 등이 UCL 소속일 수 있음. **PDF 본문에서 affiliation 확인 필요 (deep-reader 영역)**

### 5.2 실무/오픈소스 커뮤니티

**Hacker News 직접 언급 스레드: 미확인**
- 2026-04-21 시점, `arxiv.org/abs/2604.14228` URL을 직접 제출한 HN 스레드가 검색되지 않음. "Dive into Claude Code" 키워드 검색 결과 0건
- 단, 배경 토론 활발: [Claude Code Unpacked: A visual guide](https://news.ycombinator.com/item?id=47597085) (1,128 pts), [Claude Code Source Leak](https://news.ycombinator.com/item?id=47586778), [The Claude Code Leak](https://news.ycombinator.com/item?id=47609294), [What Claude Code's Source Revealed About AI Engineering Culture](https://news.ycombinator.com/item?id=47772282) (76 pts, 55 comments — 1주일 전)

**GitHub 저장소 지표 (2026-04-21)**
- 482 stars / 60 forks / 0 open issues
- **해석**: 1주차치고 수용도 양호 (AI 분야 arXiv 논문 대비 중상위권). 이슈 0은 분쟁이 없다기보다 "공식 저장소의 역할이 주로 PDF 호스팅 + 큐레이션 목록"이라 기술 논쟁이 이 공간에서 일어나지 않는다는 뜻
- `docs/related-resources.md`에 **50+ 개 독립 분석/재구현 큐레이션** (본 논문을 중심 노드로 하는 생태계 지도 역할)

**Reddit**: 본 논문 자체 토론 스레드 미확인 (검색 한계). Claude Code 일반 토론은 r/ClaudeCode가 4,200+ weekly contributors로 활발 — 출처: [Claude Code Reddit: What Developers Actually Use It For in 2026](https://www.aitooldiscovery.com/guides/claude-code-reddit)

**독립 리뷰**
- **arxiviq Substack** ([링크](https://arxiviq.substack.com/p/dive-into-claude-code-the-design)) — **가장 신뢰도 높은 독립 리뷰**. 긍정/비판 균형:
  - 긍정: "5-layer context compaction", "isolated subagents in git worktrees", "deny-first gates" 설계 찬사
  - 비판: **"approval fatigue (93% 승인률이 safety mechanism을 약화)"**, **"code complexity 40.7% 증가"**, **"aggressive compaction/subagent isolation이 optimal local decisions without full global codebase awareness를 강제하는 supervision paradox"**

### 5.3 한국어 커뮤니티

한국어권은 **논문 자체를 다룬 글은 미확인**, 그러나 Claude Code 소스 누출 사건을 계기로 한 기술 분석은 풍부:

| 자료 | 저자 | 날짜 | 성격 |
|---|---|---|---|
| [Claude Code 내부 아키텍처 분석](https://bits-bytes-nn.github.io/insights/agentic-ai/2026/03/31/claude-code-source-map-leak-analysis.html) | Jonas Kim | 2026-03-31 | 소스 누출 직후 리버스 엔지니어링. **4600+ 파일 / async generator 루프 / 4-layer compaction / 8-layer security** 명확히 기술. **논문의 주장과 상당 부분 겹침** |
| [Claude Code 내부 아키텍처 분석](https://taeho.io/reading/claude-code-%EB%82%B4%EB%B6%80-%EC%95%84%ED%82%A4%ED%85%8D%EC%B2%98-%EB%B6%84%EC%84%9D_20264) | Taeho Kim | 2026-04-01 | 위 글 리뷰·확장. 같은 구조 검증 |
| [클로드 코드 유출 소스코드 아키텍처 정밀 분석](https://brunch.co.kr/@seawolf/121) | (Brunch @seawolf) | 2026-03~04 | 2,041 파일, 512,664 라인 정량 집계 |
| [WaveSpeedAI 한국어판: Claude Code 소스 유출 숨겨진 기능](https://wavespeed.ai/blog/ko/posts/claude-code-leaked-source-hidden-features/) | WaveSpeedAI | 2026-03~04 | BUDDY, KAIROS, Undercover Mode 등 |
| [별첨 91. 클로드 코드 소스 코드 분석서 — 클로드 코드 가이드](https://wikidocs.net/338204) | (위키독스 커뮤니티) | 2026 | 루프·권한·도구 시스템 |
| [seilk/claude-code-docs](https://github.com/seilk/claude-code-docs) | seilk | — | 한국어 문서 저장소 |
| [SK Devocean: 진짜 AI 에이전트형 코딩 툴 Claude-Code](https://devocean.sk.com/blog/techBoardDetail.do?id=167718) | broccoli | 2025-08-25 | 사용법 중심 소개 (누출 이전) |

**한국 측 특이 기여**: **Sigrid Jin(@instructkr)** — 논문보다 2주 앞서 claw-code(Rust+Python clean-room rewrite)를 공개. GitHub 역대 최속 스타 성장 리포 중 하나. 출처: [WaveSpeedAI](https://wavespeed.ai/blog/posts/what-is-claw-code/)

한국어 네이버/카카오 공식 기술 블로그에서 본 논문을 다룬 글은 **미확인**.

---

## 6. 재현성·논란

### 6.1 방법론상 이슈
- **소스 출처의 법적·윤리적 모호성**: 논문이 분석 대상으로 삼는 Claude Code v2.1.88 소스는 **Anthropic이 의도적으로 공개한 오픈소스가 아닌, 2026-03-31 npm 소스맵 실수로 유출된 상용 코드**. 논문 본문이 이 사실과 Terms of Service 관계를 명시적으로 다루었는지가 중요. 미공개 시:
  - 재현성: ✅ (누구나 누출된 소스로 동일 분석 가능)
  - 윤리: ❓ (상업 시스템 리버스 엔지니어링 결과의 학술 출판 정당성 논란 가능)
- **OpenClaw 비교 공정성 의문**: OpenClaw(github.com/openclaw/openclaw)는 실존하지만 성격 상이. 개인 AI 어시스턴트(Telegram/Discord/WhatsApp 연동, computer-use 에이전트)이며 Claude Code는 터미널 코딩 에이전트. **동일 범주 내 대조가 아니라 범주 간 대조**일 수 있음. 비교 대상으로 더 적합했을 후보: OPENDEV(arXiv 2603.05344), OpenHands, Aider, SWE-Agent, Cursor. 출처: [OpenClaw vs Eigent](https://nerdbot.com/2026/04/07/openclaw-vs-eigent-choosing-the-right-open-source-ai-agent-in-2026/), [OpenClaw 공식](https://openclaw.ai/)

### 6.2 수치 출처 의문 (검증 업데이트)

deep-reader가 요청한 10개 수치의 원출처 검증 결과 (2026-04-21 재조사):

| # | 본문 수치 | 본문 인용 표시 | 실제 원출처 | 판정 |
|---|---|---|---|---|
| 1 | 27% task | Huang et al. 2025 | [Anthropic "How AI Is Transforming Work" (2025-09, 132 survey)](https://www.anthropic.com/research/how-ai-is-transforming-work-at-anthropic) | ✅ 확인 |
| 2 | 93% 승인률 | Hughes 2026 | [Anthropic "Claude Code auto mode" (2026-03-25, John Hughes)](https://www.anthropic.com/engineering/claude-code-auto-mode) | ✅ 확인 |
| 3 | 20%→40% auto-approve | McCain et al. 2026 | [Anthropic "Measuring Agent Autonomy" (Miles McCain 외)](https://www.anthropic.com/research/measuring-agent-autonomy) | ✅ 확인 |
| 4 | 17% comprehension 하락 | Shen & Tamkin 2026 | [arXiv:2601.20245 (Judy Hanwen Shen & Alex Tamkin)](https://r.jordan.im/download/language-models/shen2026.pdf) | ✅ 확인. **Zhiqiang Shen과 동명이인, 자기인용 아님** |
| 5 | 1.6% / 98.4% | community analysis (익명) | 커뮤니티 공통 관측치 (512K lines 중 ≈8K lines AI 로직). X/블로그 다수 | ⚠️ 출처 익명화되어 있으나 수치 자체는 재현 가능 |
| 6 | CVE-2025-59536 (8.7) | 각주3 | [NVD, Check Point Research](https://research.checkpoint.com/2026/rce-and-api-token-exfiltration-through-claude-code-project-files-cve-2025-59536/) | ✅ 실존, hooks/MCP 원격 코드 실행 |
| 6b | CVE-2026-21852 (5.3) | 각주3 | [NVD, GitLab Advisories](https://advisories.gitlab.com/pkg/npm/@anthropic-ai/claude-code/CVE-2026-21852/) | ✅ 실존, ANTHROPIC_BASE_URL via env → API 키 유출 |
| 6c | CVE-2025-54794 | 각주3 | [Cymulate "InversePrompt"](https://cymulate.com/blog/cve-2025-547954-54795-claude-inverseprompt/) | ✅ 실존, path restriction bypass, <v0.2.111에서 수정 |
| 6d | CVE-2025-54795 | 각주3 | 상동 | ✅ 실존, command injection, <v1.0.20에서 수정 |
| 7 | 84% 프롬프트 감소 (sandboxing) | Dworken & Weller-Davies 2025 | [Anthropic "Claude Code sandboxing" 엔지니어링 블로그](https://www.anthropic.com/engineering/claude-code-sandboxing) | ✅ 확인 |
| 8 | 40.7% 복잡도 증가 (Cursor, 807 repo) | He et al. 2025 | [arXiv:2511.04427 CMU 연구](https://arxiv.org/abs/2511.04427) | ✅ 실존 (807+1380 repo DiD 연구). **단 Cursor 대상이지 Claude Code 대상 아님 → Claude Code 비판으로 전용하면 일반화의 비약** |
| 9 | 19% 느림 (16 devs) | Becker et al. 2025 | [arXiv:2507.09089 METR Joel Becker](https://arxiv.org/abs/2507.09089) | ✅ 실존 (RCT 16명). **단 n=16은 작음, Claude Code 이전 AI 도구 시기** |
| 10 | 25% 주니어 채용 감소 | Rak 2025 | **"Rak 2025" 직접 출처 미확인**. 가장 근접한 값은 [Stanford Brynjolfsson et al. 2025-08 "Canaries"](https://digitaleconomy.stanford.edu/wp-content/uploads/2025/08/Canaries_BrynjolfssonChandarChen.pdf)의 22-25세 약 20% 감소 | ⚠️ **의심 인용** — `deep-reader`가 각주 원문 재확인 필요 |

**종합 평가**:
- 10건 중 **9건이 실존 출처로 확인**. 심각한 fabrication 없음
- **유일 불확실**: "Rak 2025 — 25%" 건 (가장 가까운 Stanford 연구는 약 20%이고 저자명 불일치)
- "Shen & Tamkin 2026" 자기인용 의혹은 **무효** — Zhiqiang Shen(MBZUAI)과 Judy Hanwen Shen(Anthropic)는 **동명이인**. 오히려 이 연구는 Anthropic 자체가 Claude Code 생산성 주장에 반하는 경험적 증거를 낸 것으로 논문의 독립성 확보
- **전용의 비약**:
  - He et al.(Cursor) → Claude Code 비판에 전용: Cursor와 Claude Code는 서로 다른 시스템. 결론 일반화 주의
  - Becker et al.(n=16, Cursor+Claude 3.5/3.7) → Claude Code 현재 버전 비판에 전용: 표본 작고 대상 버전 상이

### 6.3 재현 실패 보고
- 미확인 — 논문 제출 7일차이고, 본 논문 실험은 정성적 아키텍처 해부라 "재현 실패" 개념 자체가 희석됨

### 6.4 직접적 기술 논쟁
- **본 논문 대상 기술 논쟁**: 미확인
- **Claude Code 아키텍처 전반에 대한 기술 비판**: HN [Claude Code Unpacked](https://news.ycombinator.com/item?id=47597085) 상위 댓글 위주로 존재:
  - "500K 라인은 TUI wrapper치고 과도" (troupo) vs "production polish는 폭증한다" (hombre_fatal)
  - "probabilistic 모델을 deterministic하게 만드는 것은 state-management nightmare" (amangsingh)
  - Unicode/scrolling 버그 실제 보고 다수
  - 논문이 이런 실무 관점 비판을 다룬지 여부 확인 필요 — 다루지 않았다면 **"설계 의도와 실제 실행 품질의 괴리"**에 대한 인사이트 누락

---

## 7. 수집 한계 (접근 불가 영역)

- **X/Twitter**: 본 논문을 다룬 저자/핵심 연구자 스레드를 직접 검색·확인하지 못했다. WebSearch 결과에 스레드 URL이 없었고 X 플랫폼 자체는 WebFetch로 안정적 접근 불가
- **HuggingFace Papers 페이지의 추천수/댓글**: 페이지 본문 파싱 실패 (조회 시 arXiv로 리디렉트된 것으로 추정). 업보트·댓글 수치 미확인
- **Reddit 스레드 직접 조회**: 본 논문을 다룬 r/MachineLearning·r/ClaudeAI·r/LocalLLaMA 스레드 존재 여부 확증 불가. 키워드 검색 결과 일반 Claude Code 스레드만 노출
- **YouTube 메타데이터**: ["The Architecture of Claude Code: Kernel and the Harness"](https://www.youtube.com/watch?v=QAXPfAbUM6U) 영상이 논문 관련인지, 누가 업로드했고 조회수가 얼마인지 파싱 실패
- **저자 개별 소속**: Jiacheng Liu / Xiaohan Zhao / Xinyi Shang은 arXiv 페이지에 affiliation 없음. 논문 PDF 본문 확인 필요 (`deep-reader` 영역)
- **Google Scholar 피인용 수**: 제출 1주차라 반영 전
- **피인용 추적 도구(Semantic Scholar 등)**: 동일 사유로 반영 전

수집 도구 권한 차단은 **감지되지 않았다**. WebFetch/WebSearch/mcp__github-oss 모두 정상 동작.

---

## 부록: deep-reader에게 전달한 교차검증 요청

- §5 수치 원출처 검증: 93%/17%/0.4%는 Anthropic auto mode 블로그(2026-03-25) 출처. 132명은 별도 Claude at Work Survey 출처. 두 출처가 본문에서 혼용되지 않았는지 확인
- §비교 분석: OpenClaw 비교의 범주 타당성, OPENDEV(2603.05344) 등 더 유사 시스템과의 대조 여부 확인
- §서론: "publicly available TypeScript source code" 표현이 누출 사건(2026-03-31)을 어떻게 다루는지 확인
