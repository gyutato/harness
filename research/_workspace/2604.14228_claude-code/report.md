# Dive into Claude Code — 설계 공간 해부 논문의 통합 리뷰

작성일: 2026-04-21 | 조사 범위: 단일 논문 (arXiv:2604.14228 v1)
작성자: insight-analyst (팀 `paper-2604-14228`)

---

## 개요 (3줄)

본 논문은 Claude Code v2.1.88의 **512K LOC / 1,884 TypeScript 파일**을 소스 수준까지 추적해 "모델은 판단하고 하네스가 집행한다(the model reasons and the harness enforces)"는 설계 철학을 7 컴포넌트 / 5 계층 / 5 가치 / 13 원칙 / 5층 컨텍스트 샤퍼 / 7층 권한 / 4개 확장 메커니즘의 상호 정합성으로 해부한 **source-grounded design-space analysis**다. 가장 실용적 기여는 **"AI 결정 로직 1.6% vs 운영 인프라 98.4%"**라는 경험적 단언과, 이를 관통하는 **Evidence Tier(A/B/C) 방법론**이며, 가장 취약한 부분은 **분석 대상 소스가 2026-03-31 Anthropic npm source map 유출 사고로 공개된 코드**라는 점을 논문이 전면적으로 다루지 않았다는 윤리·방법론적 공백이다. 하네스 엔지니어 관점에서는 "어떤 책임을 결정론 계층에 내려야 하고, 어떤 확장을 어느 비용에 배치해야 하는가"에 대한 **산업 표준 수준의 레퍼런스 카탈로그**로서 높은 실무 가치를 지닌다.

---

## 조사 대상

| 항목 | 값 |
|---|---|
| 제목 | Dive into Claude Code: The Design Space of Today's and Future AI Agent Systems |
| 저자 | Jiacheng Liu¹, Xiaohan Zhao¹, Xinyi Shang¹·², Zhiqiang Shen¹ (†corresponding) |
| 소속 | ¹ VILA Lab, MBZUAI (Mohamed bin Zayed University of AI, UAE) / ² UCL (University College London) — **단, 저자 Zhiqiang Shen Group 공식 홈페이지([zhiqiangshen.com/projects/group](https://zhiqiangshen.com/projects/group/index.html))는 4명 전원 MBZUAI 기재. 논문 PDF 표기와 저자 홈페이지 간 불일치** |
| 제출일 | 2026-04-14 (v1) |
| 분야 | cs.SE (주) / cs.AI / cs.CL / cs.LG |
| arXiv | https://arxiv.org/abs/2604.14228 |
| 공식 저장소 | https://github.com/VILA-Lab/Dive-into-Claude-Code (CC BY-NC-SA 4.0, 482★/60 forks @ 2026-04-21) |
| 분량 | 본문 37쪽 + 참고문헌 + 부록 A/B = 46쪽 |
| 분석 대상 | Claude Code v2.1.88 (npm 패키지에서 추출된 TypeScript 소스) |

---

## 상세 설명

### 1. 논증 구조 (§1~§2)

저자들은 AI 코딩 도구가 **autocomplete → IDE 통합 → agentic** 3단계로 진화했고 Claude Code는 마지막 단계의 프로덕션 대표라는 관찰에서 출발한다. 핵심 질문은 **"같은 설계 질문에 다른 답을 낼 수 있는 design space를 어떻게 드러낼 것인가"**이며, 이를 위해 (1) Claude Code 내부를 설계 질문별로 해부하고, (2) OpenClaw와 6차원 대조하고, (3) 6개 개방 문제를 제시하는 3부 구성을 채택했다.

논증의 중심에는 다섯 가지 주장이 있다.

- **C1** AI 결정 로직은 약 1.6%, 나머지 98.4%는 운영 인프라다
- **C2** "model reasons, harness enforces" — 컴파일 그래프(LangGraph)나 계획 스캐폴드(Devin)와 구분되는 포지션
- **C3** 5 가치 → 13 원칙 → 구현까지의 **추적 가능성(traceability)**을 Table 1에 카탈로그화
- **C4** 7 컴포넌트 모델(Figure 1)과 5 계층 모델(Figure 3)을 **이중 서술**하여 소스 파일까지 고정
- **C5** OpenClaw 비교로 "deployment context가 architectural answer를 결정한다"는 메타 주장을 제시

저자는 추가로 **"장기 인간 역량 보존(long-term human capability preservation)"**을 6번째 가치가 아닌 **평가 렌즈(evaluative lens)**로 선언한다. 이 장치가 §11.2의 value tension 표, §11.4의 경험적 예측, §12.6의 개방 문제로 이어지며 논문의 **비판적 척추** 역할을 한다.

### 2. queryLoop() — 9단계 턴 루프 (§4.1)

`query.ts`의 `queryLoop()` AsyncGenerator가 한 턴 안에서 실행하는 9단계:

1. Settings resolution (immutable parameter destructure)
2. Mutable state initialization — 7개 continue site가 모두 **whole-object reassignment**로 상태 갱신
3. Context assembly (`getMessagesAfterCompactBoundary()`)
4. Pre-model context shapers — 5 샤퍼 순차 실행
5. Model call (streaming)
6. Tool-use dispatch
7. Permission gate
8. Tool execution & result collection
9. Stop condition

AsyncGenerator를 채택한 이유는 UI 계층으로 StreamEvent·Message·TombstoneMessage를 yield하면서도 내부는 단일 synchronous control flow를 유지하기 위함이다. 패턴 자체는 **ReAct 표준 reactive 루프**이며 탐색 완전성(search completeness)을 simplicity·latency와 맞바꿨다. 즉 **백트래킹이 없다**.

### 3. Tool Dispatch — Primary/Fallback 이중 경로 (§4.2)

- **Primary**: `StreamingToolExecutor` — 응답 스트리밍 중 툴 실행 시작, 다중 툴 지연 감소
- **Fallback**: `runTools()` — `partitionToolCalls()` 기반 순회

공통으로 툴을 **concurrent-safe vs exclusive**로 분류(read-only 병렬, shell 직렬). Sibling abort controller로 한 bash 툴 실패 시 in-flight 형제를 즉시 종료한다. Result는 모델의 `tool_use` 순서에 맞게 **요청 순서대로 emit** — **concurrent-read, serial-write** 실행 모델.

### 4. 5층 컨텍스트 샤퍼 — 논문의 핵심 분석 대상 (§4.3, §7.3)

| # | 샤퍼 | 게이트 | 대상 | 비용 |
|---|---|---|---|---|
| 1 | **Budget reduction** | 항상 | 개별 tool result | 가장 낮음 (content reference로 대체) |
| 2 | **Snip** | `HISTORY_SNIP` | 오래된 history | 낮음 (경량 trim) |
| 3 | **Microcompact** | `CACHED_MICROCOMPACT` | 툴 결과+메시지, tool_use_id 기반 | 중간 (cache-aware) |
| 4 | **Context collapse** | `CONTEXT_COLLAPSE` | 전체 history | 중간 (**read-time projection** — REPL 불변) |
| 5 | **Auto-compact** | 기본 활성 | 전체 대화 | 높음 (모델 기반 요약) |

설계 근거: **"no single compaction strategy addresses all types of context pressure"**. 4가지 원리 — (a) **lazy degradation** (값싼 것 먼저), (b) **layer composition cleanness** (content vs ID 기반 분리), (c) **different pressure sources**, (d) **auto-compact는 최후 수단**.

단, **사용자 예측 불가능성**이 단점으로 저자도 명시: "Five interacting compression layers, several gated by feature flags, create behavior that is difficult for users to fully predict." 특히 context collapse는 user-visible output 없이 동작한다.

### 5. 7 권한 레이어 — Defense in Depth (§3.5, §5)

"A request must pass through all applicable layers, and any single layer can block it" — **any-block-denies** 로직.

1. **Tool pre-filtering** — 모델 뷰에서 blanket-deny 툴 제거
2. **Deny-first rule evaluation** — deny가 allow보다 항상 우선
3. **Permission mode constraints** — 7 모드 (plan / default / acceptEdits / auto / dontAsk / bypassPermissions / bubble)
4. **Auto-mode classifier** — `yoloClassifier.ts`의 ML 기반
5. **Shell sandboxing** — 권한 시스템과 **독립축**
6. **Non-restoration of permissions on resume** — 세션을 "isolated trust domain"으로 간주
7. **Hook-based interception** — PreToolUse / PermissionRequest 훅

**Approval fatigue 대응의 설계 반응**이 특히 주목할 만하다. 사용자가 **93% 권한 프롬프트를 자동 승인**(Hughes 2026)한다는 사실에 대해, Anthropic은 "더 많은 경고"가 아닌 **"defined boundaries(sandboxing, auto-mode classifier) within which the agent can work freely"**로 응답했다. 즉 **Human Decision Authority**(사람이 결정)와 **Safety**(시스템이 보호)가 분리되는 이유는 "승인 피로로 사람의 vigilance가 신뢰 불가하다면 안전은 사람 주의와 **독립적**으로 동작해야 한다"는 것이다.

**Defense-in-depth의 균열 (§5.4):** 50개 초과 subcommand를 가진 명령은 per-subcommand deny check 대신 **단일 generic approval prompt로 fallback**(Adversa.ai 2026). 사유: per-subcommand 파싱이 UI freeze 유발. → **"다층 방어도 계층이 실패 모드를 공유하면 동시 실패한다"**는 실측 사례.

**Denial as routing signal**: permission 거부는 halt가 아니라 shape이다. 모델이 denial reason을 받아 safer alternative를 시도한다.

### 6. 4 확장 메커니즘 — 비용·격리 분리 (§6)

| 메커니즘 | Context Cost | Insertion Point |
|---|---|---|
| **Hooks** | **Zero by default** | execute() |
| **Skills** | **Low (descriptions only)** | assemble() |
| **Plugins** | Medium | all three |
| **MCP servers** | **High (tool schemas)** | model() |

저자의 중심 명제: "different kinds of extensibility impose different costs on the context window, and a single mechanism cannot span the full range from zero-context lifecycle hooks to schema-heavy tool servers without forcing unnecessary trade-offs."

3 injection points(Figure 5): **`assemble()`(보는 것) / `model()`(닿을 수 있는 것) / `execute()`(어떻게 실행되는지)**.

Hook은 27 이벤트 / 15 schemas / 4 persistable command types(command/prompt/http/agent).

### 7. Subagent 위임·격리 (§8)

`AgentTool.tsx`가 delegation의 진입점. **SkillTool은 instruction을 현재 컨텍스트에 주입**하는 반면 **AgentTool은 새 isolated context window를 생성**한다. Isolation modes: worktree / remote(internal-only) / in-process(default).

**핵심 비용 포인트:**
- Multi-agent 경로: "Claude Code's agent teams consume approximately **7× the tokens** of a standard session in plan mode" (Anthropic 2025b)
- Team coordination은 **file locking mutex** — zero-dependency + 완전한 debuggability 획득, 처리량 희생
- 부모에게 돌아가는 것은 **summary-only return** (context bottleneck 원칙)

### 8. 컨텍스트 조립 & CLAUDE.md 4 계층 (§7)

Context window 9개 소스 — system prompt → environment info → CLAUDE.md hierarchy → path-scoped rules → auto memory → tool metadata → conversation history → tool results → compact summary.

**CLAUDE.md 4 레벨:** Managed / User / Project / Local. **"Later-loaded files receive more model attention"**이라 discovery는 CWD에서 root 방향.

**결정적 설계 분리:**
> "CLAUDE.md content is delivered as user context (a user message), not as system prompt content. ... model compliance with these instructions is probabilistic rather than guaranteed. Permission rules evaluated in deny-first order provide the deterministic enforcement layer."

즉 **CLAUDE.md는 suggestion (확률적), Permission rules는 enforcement (결정적)**. 보안·정책은 CLAUDE.md에 적으면 안 된다는 강한 권고가 여기서 나온다.

**Auto memory retrieval은 embedding을 쓰지 않는다** — LLM이 memory-file header를 스캔, 최대 5 파일, 파일 단위. Inspectability + infrastructure simplicity 우선.

### 9. Session Persistence (§9)

3 independent persistence channels:
1. **Session transcript** (project-scoped, 1 file/session JSONL, mostly append-only)
2. **Global prompt history** (`history.jsonl`, 역순 reader)
3. **Subagent sidechain** (.jsonl + .meta.json per subagent)

**`compact_boundary` 마커**: `headUuid`/`anchorUuid`/`tailUuid`로 read-time에 message chain을 patch. 원본은 디스크 보존, 로더가 metadata로 link — **mostly-append 설계**.

**Resume/Fork에서 session-scoped permission 비복원 (§9.2, 권한 layer 6):** "Sessions are treated as isolated trust domains" — 재부여 friction을 **trust invariant 유지**로 교환.

### 10. OpenClaw 6차원 대조 (§10)

| 축 | Claude Code | OpenClaw |
|---|---|---|
| 시스템 범위 | ephemeral CLI | persistent daemon |
| 신뢰 모델 | per-action deny | perimeter-level access |
| 런타임 | loop = center | loop = component in gateway |
| 확장 | single agent context | gateway-wide capability registry |
| 메모리 | 5-layer compaction | long-term promotion (dreaming, daily notes) |
| 멀티에이전트 | task delegation | multi-agent routing + sub-agent delegation |

결정적 compositional 관찰: **"OpenClaw can host Claude Code... via ACP"** — 둘은 배타적 대안이 아니라 **composable layered taxonomy**다.

### 11. 주목할 실증 요소 — CVE와 Empirical Predictions (§11)

**Pre-trust initialization CVE 4건:**
- CVE-2025-59536 (CVSS 8.7), CVE-2026-21852 (CVSS 5.3), CVE-2025-54794, CVE-2025-54795
- 공통 root cause: **초기화 코드(hook, MCP 서버 connection, settings 해결)가 interactive trust dialog 이전에 실행**
- 저자의 핵심 관찰: "Figure 4가 safety check의 spatial ordering은 보여주지만 **temporal dimension은 포착하지 않는다**"
- → **defense-in-depth의 시간적 공백**

**Empirical predictions:** 아키텍처 속성에서 검증 가능한 예측 도출.
1. Bounded context → pattern duplication·convention violation
2. Subagent isolation → 병렬 에이전트가 독립적으로 동일 solution 재구현
3. "good local decision → poor global outcome when bounded context prevents global awareness"

보조 증거: Cursor 기반 807-repo causal study(He et al. 2025) — 복잡도 **+40.7% (p<0.001)**, velocity 첫 달 +281% → 3개월째 baseline ("self-cancelling").

### 12. 6 Open Directions (§12)

1. Silent failure / observability-evaluation gap
2. Cross-session persistence
3. Harness boundary evolution (4축)
4. Horizon scaling (세션 → 멀티 세션 → scientific program)
5. Governance and oversight
6. Evaluative lens revisited — long-term capability preservation

패턴: 개방 문제 6개 중 "하네스 내부에서 해결 가능"(12.1, 12.5 부분)과 "하네스를 넘어서는 문제"(12.2, 12.3, 12.4, 12.6)로 나뉜다. 저자는 **후자에 더 많은 비중**을 뒀다 — 즉 논문의 수사적 포지션은 "Claude Code는 잘 설계되었지만 **미래 문제의 locus는 하네스 layer보다 넓다**".

---

## 외부 맥락 요약

### 학계 맥락

- **공식 피인용 0건** (제출 1주차, Google Scholar 반영 전)
- **가장 가까운 선행 논문**: OPENDEV(arXiv:2603.05344, 2026-03-05 제출, 1개월 선행) — 터미널 기반 코딩 에이전트를 **Scaffolding / Harness / Context Engineering** 키워드로 분석. **Claude Code 논문과 동형 구조의 선행 연구**가 존재하는데 본 논문이 이를 인용했는지는 참고문헌 전수조사 필요. deep-reader 독해상 **명시적 인용 없음**으로 추정, 선행연구 누락 비판 가능.
- **VILA-Lab의 포지셔닝**: 주 연구 테마는 비전-언어 모델·효율화였으며, Dive-into-Claude-Code는 **에이전트 시스템 분야 첫 진출**이자 조직 내 2위 스타 수 레포 — 연구 방향 확장 시점의 시그널.
- **Zhiqiang Shen ≠ Judy Hanwen Shen** — "Shen & Tamkin 2026"의 Shen은 **Anthropic 소속 Judy Hanwen Shen**으로, 저자 Zhiqiang Shen(MBZUAI)과 **동명이인**. deep-reader A2의 순환 인용 의혹은 **해소**. 다만 함의가 있다 — Shen & Tamkin 2026(17% comprehension 하락)은 **Anthropic 자체가 내놓은 Claude Code 생산성 주장에 반하는 증거**. 본 논문의 비판적 독립성이 오히려 **강화**되는 방향.
- **외부 매체의 저자 오귀속 현상 (메타 관찰)**: 본 논문은 외부 매체에서 "UCL researchers의 연구"로 잘못 소개되는 현상이 관측된다. 예: Akshay Pachaar X 포스트, HuggingFace Papers 추출 일부. 그러나 저자 본인 운영 Zhiqiang Shen Group 공식 홈페이지(https://zhiqiangshen.com/projects/group/index.html)는 저자 4명 전원 MBZUAI 소속으로 기재한다. → 이 오귀속은 **저자의 부주의가 아니라 외부 생태계의 오류 전파** 현상이다. 후속 문헌이 원 소속을 재확인하지 않으면 오류가 고착될 위험. 단, **본 논문에 대한 비판 근거로 사용할 수는 없다**.

### 1차 자료와 원출처

논문의 주요 수치는 **Anthropic 엔지니어링 블로그 2종**으로 추적:

| 수치 | 원출처 | 성격 |
|---|---|---|
| 27% 새로운 태스크 | How AI Is Transforming Work at Anthropic (2025-09) | **132명 서베이 + 53명 인터뷰** |
| 93% 승인률 / 17% FNR / 0.4% FPR | Claude Code auto mode (Hughes, 2026-03-25) | **데이터셋 평가(서베이 아님)** |
| 20%→40% auto-approve | McCain et al. 2026 (Anthropic) | <50→750 세션 수 기준 |
| 84% prompt 감소 | Dworken & Weller-Davies 2025 (Anthropic sandbox) | 내부 usage |
| 40.7% 복잡도 증가 | He et al. 2025 (arXiv:2511.04427) | **Cursor 대상**, Claude Code 아님 ⚠️ |
| 19% 느림 | Becker et al. 2025 (METR, 16명 RCT) | Cursor Pro + Claude 3.5/3.7 (Claude Code 이전) ⚠️ |
| 17% comprehension 하락 | Shen & Tamkin 2026 (Anthropic, 52명 RCT) | passive delegation이 원인 |

> ⚠️ **40.7% / 19% 주의**: 두 수치 모두 **Cursor 대상 연구**이며, 19% (Becker) 연구는 Claude Code 정식 릴리스 이전 시기 Cursor Pro + Claude 3.5/3.7 Sonnet (n=16). 논문이 이를 Claude Code 비판의 일반 증거로 전용했다면 **일반화의 비약** (H3b 참조).

### 커뮤니티 반응

- **GitHub 저장소**: 482★ / 60 forks / 0 issues (1주차치고 수용도 양호, 이슈 0은 저장소가 PDF 호스팅+큐레이션 역할이라 기술 논쟁 공간이 아님)
- **HN/Reddit**: 논문 자체를 다룬 직접 스레드는 **미확인**. 배경 토론은 풍부(HN 1,128 pts 스레드 등)
- **가장 신뢰도 높은 독립 리뷰**: arxiviq Substack — 긍정(5-layer context compaction, isolated subagents, deny-first gates 찬사) + 비판(**approval fatigue가 safety mechanism 약화**, **supervision paradox**: aggressive compaction/subagent isolation이 global awareness 없이 optimal local decision을 강제)

### 소스 출처의 윤리적 모호성 (비판의 중심축)

**2026-03-31, Anthropic이 Claude Code v2.1.88 npm 패키지에 .map 파일(약 59.8MB)을 실수로 포함 배포.** 4,600+ TypeScript/TSX 파일이 외부 노출됐다. 이는 **의도적 오픈소스가 아닌 사고 유출**이다.

- 논문은 "All materials obtained from publicly available online sources"로 **"publicly available"이라는 중립 프레이밍** 사용
- 유출 사건 자체와 Terms of Service 관계, 리버스 엔지니어링의 윤리적 정당성은 **본문에서 명시적으로 다뤄지지 않는다**
- 재현 가능(✅) vs 학술 출판의 윤리적 정당성(❓)은 open question

### 재구현·분석 생태계

- **instructkr/claw-code** (Sigrid Jin) — 누출 당일부터 Rust+Python clean-room rewrite, **GitHub 역대 최속 성장 레포 중 하나**
- 기타 7+ 개 리버스 엔지니어링·재구현 프로젝트가 논문 공식 `related-resources.md`에 큐레이션
- 한국어권: Jonas Kim, Taeho Kim, Brunch @seawolf 등이 유출 직후 독립 분석

---

## 핵심 인사이트

### 1. 기여 재정의 — 이 논문이 실제로 한 일

저자들이 표면적으로 내세우는 "5 가치 / 13 원칙 / 7 컴포넌트" 프레임워크는 **본질이 아니다**. 한 단계 올리면 실질 기여는 세 가지다.

**(1) Evidence Tier 방법론의 제시**
Tier A(product doc) / Tier B(code-verified) / Tier C(reconstructed) 구분은 리버스 엔지니어링으로 상용 시스템을 분석하는 **후속 연구에 재사용 가능한 방법론적 장치**다. 단 본 논문은 이 장치를 **부록 B**에 숨겨두어 본문 독자가 Tier를 구분 못 한다 — 방법론 기여를 본문이 활용 못 하는 **자기 모순적 배치**.

**(2) "model reasons, harness enforces" 의 경험적 카탈로그**
이 슬로건은 Martin Fowler, Philipp Schmid 등이 2025년 후반부터 반복해 온 것이고, 본 논문이 **처음 제안한 것이 아니다**. 진짜 기여는 슬로건이 아니라 **그것이 실제 소스 어디에 어떻게 구현되는지 Tier B 수준으로 고정한 최초의 학술 문서**라는 점.

**(3) 설계 공간을 flat taxonomy가 아닌 composable layered로 재정의**
§10의 "OpenClaw can host Claude Code via ACP" 관찰은 사소해 보이지만 중요. 기존 AI 에이전트 비교 연구가 "A vs B"로 대립 프레이밍되던 것을 **"A는 B의 컴포넌트가 될 수 있다"는 계층적 조합 관점**으로 전환 — 이것이 개념적 기여. OpenClaw 비교의 공정성 문제를 감안해도 살아남는 통찰.

**(4) 보조 관찰 — 수사적 기법의 세련됨: "Anthropic의 말로 Anthropic을 비판한다"**
저자들의 인용 구조를 분석하면 한 가지 패턴이 드러난다:
- **Anthropic 공식 자료를 적극 인용** — Huang et al. 2025 (132명 서베이), Hughes 2026 (auto-mode 데이터셋), Dworken & Weller-Davies 2025 (sandboxing), McCain et al. 2026 (agent autonomy), Martin 2026 (Managed Agents)
- **비판 렌즈(evaluative lens)의 핵심 근거도 Anthropic 내부 연구** — Judy Hanwen Shen & Tamkin 2026 (Anthropic 소속 연구자의 17% comprehension 하락 RCT)
- 외부 empirical은 주변 보강으로만 사용 — He et al. 2025 (Cursor), Becker 2025 (METR), Kosmyna 2025 (EEG)

즉 이 논문의 비판적 에너지는 **Anthropic 외부에서 끌어오지 않는다**. Anthropic 자체가 내놓은 증거로 Anthropic 시스템의 한계를 지적한다.

**이것은 자기 인용이 아니지만 수사적으로 정교하다.** 장점: (a) 저자의 비판이 "반대 진영의 편향"이라는 반박을 선제 차단, (b) Anthropic 독자·리뷰어가 증거를 기각하기 어려움, (c) 저자의 "연속성 + 비판적 거리" 포지션을 구조적으로 뒷받침.

**Blind spot**: 이 구도는 **Anthropic 생태계가 생산한 증거의 범위 바깥에 있는 비판**을 시야에서 탈락시킬 수 있다. 예컨대 Anthropic이 공개하지 않은 실패 사례, 타 조직 연구자의 경험적 비판, 사용자 커뮤니티의 정성적 불만 등은 논문에 거의 반영되지 않는다. **비판의 수원(source)을 Anthropic으로 한정하면 Anthropic이 측정하지 않은 축에서의 비판은 누락**된다.

### 2. 한계와 암묵적 가정 — 비판적 축 6가지

**H1. 소스 출처의 윤리적 공백 + 재현성 규범 미달 (가장 심각)**

분석 대상은 Anthropic이 **의도하지 않은 유출로 공개된 상용 코드**다.
- **2026-03-31** Anthropic npm 패키지 .map 파일 실수 포함 사건 (4,600+ TypeScript/TSX 파일 노출)
- **2026-04-14** 본 논문 arXiv 제출 — **누출 후 정확히 2주**
- 논문 disclaimer "publicly available online sources... not unauthorized"는 "간접 공개된 유출 소스"가 authorized에 해당하는지의 **법적·윤리적 회색지대**에서 침묵

VILA-Lab 공식 저장소도 재현 아티팩트 **5가지 누락**:
1. `src/`, `data/`, `scripts/`, `analysis/` 폴더 부재 — 추출·분석 파이프라인 0 공개
2. Methodology / Data Sources / How We Obtained the Source 섹션 누락
3. "1.6% / 98.4%" 측정 스크립트·원 데이터 미공개 — 재현 불가
4. 컴포넌트 참조에 **파일 경로·라인 포인터 부재** — 개념 서술뿐
5. CITATION.cff가 제출 1주 후에도 "To be updated" 미완성

**역설(아이러니의 핵심)**: `docs/related-resources.md`가 큐레이션한 타 커뮤니티 프로젝트(ComeOnOliver/claude-code-analysis, alejandrobalderas/claude-code-from-source 등) — 아마추어 리버스 엔지니어링이 **소스 트리 구조·모듈 경계·파일 경로를 공개**한다. 즉 **학술 논문이 학술 밖 아마추어 분석보다 재현성 투명도가 낮다**는 메타 역설.

**프레이밍(명확화) — 독자가 가장 잘못 받아들일 수 있는 축**: 이 비판은 **"사기·날조"가 아니다**.
- 수치 다수가 타 독립 분석과 중첩 검증됨
- 인용 9/10이 실존 논문·블로그로 추적 가능
- CVE 4건 NVD 등록 실존

비판의 정확한 대상은 **"방법론 학술 규범 미달"**이다:
- 논문의 Tier B 주장(code-verified)이 **실질적으로 Tier C에 가깝게 운영**된다 — 파일·라인 인용이 있어도 독자가 원 소스에서 검증할 경로가 저장소에 없다
- 방법론 챕터의 신뢰 기반인 Tier A/B/C 위계 **자기 자신에게 엄밀히 적용되지 않았다** (방법론 기여의 자기 모순적 배치)
- CITATION.cff "To be updated"는 작은 단서지만 상징적 — 급조된 인상 강화

→ **후속 연구자의 의무**: Claude Code를 cite할 때 (a) 누출 사실 투명 병기, (b) 본 논문의 Tier B 주장을 **Tier C로 재분류**해 신뢰도 조정.

**H2. OpenClaw 비교의 범주 오류**
OpenClaw는 **"개인 AI 어시스턴트, 컴퓨터 사용 에이전트, Telegram/Discord/WhatsApp 멀티채널 게이트웨이"**로 **배포 컨텍스트 자체가 다른 시스템**. §10.2에서 "the questions are stable; the answers vary with deployment context"라 인정하지만, 그러면 **비교가 "대립적 설계 bet"으로 프레이밍되는 것 자체**가 과장. 더 적절한 비교 대상:
- **OPENDEV (arXiv:2603.05344, 1개월 선행)** — 터미널 코딩 에이전트, 키워드 100% 겹침
- Aider, OpenHands, SWE-Agent

OPENDEV 미인용은 **방법론적 누락**이자 **기여 일부 중복** 가능성. 다만 논문이 OpenClaw를 "multi-channel personal assistant gateway"로 **범주 차이를 명시**했다면 판정은 "대립적 대조가 아닌 범주 간 조명"으로 **완화 가능** — 공정성 비판의 강도는 "명시 정도"에 따라 달라진다.

**보조 관찰 (신뢰도 중)**: OpenClaw 개발자 Peter Steinberger는 2026-02-15 OpenAI 합류. 3개월만에 100K stars 돌파 직후. 본 논문(2026-04 제출)이 비교 대상으로 OpenClaw를 선택한 것과 이 화제성의 상관관계는 본문에서 논의되지 않음. 비교 대상의 **인지도·드라마성**이 선정 기준에 영향을 줬을 가능성은 배제되지 않는다.

**H3. "1.6% vs 98.4%" 의 정의 공백**
중심 주장(§11.1, §11.7)을 지탱하는 이 비율은 **저자가 먼저 측정한 것이 아니다**. 2026-04 시점 Akshay Pachaar(X), Odaily(중국어), ComeOnOliver 등 **커뮤니티 리버스 엔지니어링이 선행**. 논문은 "community analysis"로 익명화 인용.
- 더 심각한 문제: **"AI decision logic" vs "operational infrastructure" 경계 정의**가 본문에 없다. Prompt construction, Tool schema는 어느 쪽인가? 해석에 따라 10%p 이상 이동 가능.
- **"harness가 많으면 좋다" 논리의 약점**: 저자는 이 비율을 아키텍처 우월성 증거로 쓰지만, 더 간단한 모델 + 더 복잡한 하네스 조합이 더 높은 비율로 열등할 수도(LangGraph·Devin). 경험적 반례 검증 부재.

**H3b. 외부 수치의 "간접 전용" — 문맥별 hedge 일관성 부재**
논문은 Claude Code 아키텍처를 비판하기 위해 **다른 시스템 대상 경험 연구**를 자주 끌어온다. 문제는 **hedge의 일관성**이다:
- §11.4는 "architecturally similar tools"로 hedge (Cursor 연구가 Claude Code에 직접 적용되지 않음을 인정)
- 그러나 §11.2 Table 4 value tension 근거에는 **그런 hedge 없이** 40.7% 복잡도 (Cursor 연구), 19% 느림 (Cursor Pro + Claude 3.5/3.7, Claude Code 이전 시기 n=16)이 Claude Code의 "Capability × Reliability" 증거로 바로 제시
- 25% 주니어 채용 감소 (Rak 2025): 출처 직접 미확인, 가장 가까운 Brynjolfsson 2025-08은 22-25세 **약 20%** 감소(25% 아님)

이는 "수치가 거짓"이 아니라 **추론 단계별 hedge가 섹션 간 일관되지 않다**는 서지적 투명성 문제. 동일 수치가 방법론 섹션과 아귀먼트 섹션에서 다른 신뢰도로 사용됨.

**추가 — 93% 수치의 재프레이밍 문제**: 원출처 Hughes 2026 블로그포스트는 **auto-mode classifier의 평가 지표** (10K real traffic + 1K synthetic, 0.4% FPR·17% FNR on 52 real overeager actions). "사용자의 승인 습관 자체"를 주제로 한 사용자 연구가 **아니다**.

논문은 이 수치를 "approval fatigue — 사람은 권한 프롬프트를 신뢰할 수 없다"의 **행동 근거**로 재프레이밍. 블로그포스트가 그 결론을 직접 주장한 것은 아니다. **논리적 도약 존재**.

또한 Huang et al. 2025 ("132명 서베이")와 Hughes 2026 (데이터셋 평가)은 방법론도 대상도 다른 **독립된 두 연구**인데, 논문은 수시로 "Anthropic 내부 데이터"로 묶어 제시한다. 출처의 미세한 차이가 주장의 강도에 영향을 주는데 이 구분이 표면에 드러나지 않는다.

**H4. Evaluative lens의 수사적 격하**
저자는 "장기 인간 역량 보존"을 6번째 value가 아닌 **평가 렌즈**로 선언하며 "Anthropic이 stated value로 두지 않았으므로"를 근거로 든다. 그러나 다른 5 value도 **"Anthropic이 채택했는지"를 독립 검증하지 않고** 공식 문서 인용으로 정당화. → **6번째를 격하하는 기준과 나머지 5를 채택하는 기준이 비대칭**. 의도된 수사 전략: "Anthropic과 연속성 유지하면서 비판적 거리 확보".
- 독립 연구자라면 **6개 value 대칭 구조**로 다시 썼어야 했고, 그랬다면 §11.2 value tension과 §12.6 개방 문제가 훨씬 강해졌을 것.

**H5. 다층 방어의 Temporal 차원 (저자 인정)**
CVE 4건에서 드러난 **pre-trust initialization window** — 초기화 코드가 interactive trust dialog **이전에 실행**되어 §5 safety 보장이 temporal하게 미적용되는 공백 — 저자가 §11.3에서 솔직히 인정하는 **프레임워크의 구멍**. Figure 4의 "spatial ordering"이 안전을 완전 포착한다는 가정이 깨짐.
- 7 레이어 모델이 temporal dimension 없이 완결적이지 않다면 **"defense in depth" 레이블 자체가 부정확**.

### 3. 적용 가능성과 조건 — 무엇을 할 수 있는가

**유효한 경우 (신뢰 가능):**
- Claude Code v2.1.88 스냅샷의 **구조·제어 흐름·의존성·feature flag 존재** — Tier B 주장
- 5층 컨텍스트 샤퍼, 7층 권한, 4 확장 메커니즘의 **질적 디자인 패턴**
- OpenClaw와의 **차이점 자체** (공정성은 별개, 대조점은 관측 사실)
- CLAUDE.md suggestion vs Permission enforcement의 **분리 원리**

**유효하지 않은 경우 (주의):**
- "1.6%" 같은 **정량 수치를 정확한 benchmark로 인용** — 정의 경계 불명확
- **"harness가 두꺼울수록 좋다"는 일반화** — 반례 검증 없음
- OpenClaw 비교 표를 **설계 대안 선택 의사결정 도구**로 사용 — 범주 오류
- "long-term capability preservation"을 Claude Code 고유 문제로 읽기 — 저자도 "하네스 layer보다 넓은 locus" 인정
- Shen & Tamkin 17% 하락을 **Claude Code에 직접 귀속** — 연구 대상은 일반 AI 지원 프로그래밍 학습

**조건부 유효:**
- CVE·pre-trust window 분석은 **보안 연구에 즉시 유용**, 단 패치 상태(논문 제출 시점 모두 패치됨) 명시
- "93% 승인률" 인용 시 **서베이가 아닌 데이터셋 평가**임을 병기

### 4. 열린 질문

논문 §12 외에 본 논문의 구조적 공백에서 도출:
1. **"model reasons, harness enforces"가 실제로 정량적 우위인가?** LangGraph·Devin vs Claude Code 동일 benchmark 비교 없음
2. **5 shapers가 실제로 user-unpredictability를 trade off할 가치를 주는가?** 단일 auto-compact 대비 ablation 가능한가
3. **Pre-trust window temporal 공백을 원칙적으로 메울 수 있는가?** 초기화 전 권한 체크 강제 시 cold-start cost 증가는?
4. **유출 소스 기반 연구의 학술 윤리 가이드라인은?** 50+ 유사 연구 생태계에 적용할 기준 필요
5. **OpenClaw를 대체할 공정한 비교 대상은?** OPENDEV·OpenHands·Aider 각각과의 대조
6. **"Harness boundary evolution" 4축(where/when/what/with whom)이 직교하는가?** 구조적 대칭성을 위한 분할인지 경험적 검증

---

## 하네스 엔지니어 관점 — 자체 하네스 설계에 적용 가능한 교훈

Claude Code를 "성경"이 아닌 **"잘 문서화된 하나의 design point"**로 다룬다.

### 교훈 1: 결정론 계층과 확률 계층을 **문자 그대로** 분리하라

CLAUDE.md(user context, **probabilistic**) vs Permission rules(deterministic enforcement)의 **명시적 분리 원칙**(§7.2).
- **보안·정책**: permission rule / deny list / sandbox에만. 절대 프롬프트에 의존하지 말 것
- **Style·preference**: CLAUDE.md / system prompt / user context
- **신뢰성**: 두 계층 혼동하면 "모델이 따를 줄 알았는데 안 따랐다" 버그가 재현 불가로 발생
- **테스트**: deny rule 작성 시 **회피 시도 케이스 필수** 첨부

### 교훈 2: 컨텍스트 압축은 단일 전략이 아니라 **graduated stack**으로

5층의 진짜 교훈은 "5층이 정답"이 아니라 **"서로 다른 context pressure 소스에 서로 다른 비용의 대응책"**.
- 가장 가벼운 층부터 활성화 (budget reduction 계열)
- **content-mutating layer vs ID-referencing layer**를 분리하면 순서 강건
- **모델 기반 요약은 진짜 최후 수단** — 비용·지연·semantic drift 복합
- **User-visible 불가측성**을 인지, 디버깅용 transcript marker 필수

단 **본 논문의 5층 그대로 copy 금지**. 대부분 자체 하네스는 3층(budget-reduction, snip, auto-compact)이 충분, context collapse · microcompact 도입은 user-unpredictability 비용이 이득을 넘을 가능성 큼.

### 교훈 3: 확장 메커니즘은 **context cost × insertion point** 2축으로

4 메커니즘의 교훈은 "4가지가 있다"가 아니라 **"context cost에 따라 graduated stack을 두면 extension author에게 불필요한 trade-off를 강제하지 않는다"**.
- **Zero-context lifecycle hook** — 부작용·알림·로깅
- **Description-only text injection** (skills) — 도메인 지식
- **Tool-schema injection** (MCP 류) — 진짜 새 capability 필요할 때만

한 메커니즘에 모든 확장을 몰아넣으면 **양방향 과부하** 발생 — 저비용 확장자에 schema 등록 부담, 고비용 확장자를 프롬프트로 구현하게 만듦.

### 교훈 4: 권한 거부를 **halt이 아닌 routing signal**로

"Denial as routing signal"(§5.2). 거부 시 모델에 **reason 전달**해 safer alternative 시도.
- Safety와 productivity가 zero-sum이 아니게 됨
- 실패 recovery 로직이 하네스가 아닌 **모델 추론에 위임** — 하네스 코드 감소
- 단 **deny reason의 품질**이 reroute 품질을 결정 — 템플릿 설계 중요

`permission_denied_error`를 단순 에러가 아닌 **structured feedback**으로 설계할 것.

### 교훈 5: Session을 **isolated trust domain**으로 간주하고 permission 비복원

세션 간 권한 복원은 편의를 주지만 **stale trust carry over** 리스크를 만듦(§9.2).
- `--resume` / `--branch` 시 session-scoped 권한 **의식적으로 폐기**
- 재부여 friction은 UX 비용이 아니라 **trust invariant 비용**으로 예산에 포함
- Global scope vs session scope 명확 분리 표시

### 교훈 6: Subagent는 **summary-only return + sidechain log**로

`AgentTool`이 parent에 돌려주는 것은 final response + metadata뿐, 전체 history는 **sidechain JSONL**에 별도 저장(§8.3).
- Parent context bloat 방지(7× 토큰 비용 일부 완화)
- Subagent 디버깅·감사 가능성 유지
- File-lock coordination으로 zero-dependency

단 **multi-agent는 7× 토큰 비용**이므로 subagent 분기는 **명시적 비용 justify가 있을 때만**.

### 교훈 7: Temporal Safety Gap을 명시적으로 점검하라

CVE 4건의 **pre-trust initialization window**는 어느 하네스에나 존재 가능. 점검 질문:
- Trust dialog **이전에** 실행되는 코드가 있는가? (config 파싱, plugin load, MCP 연결, hook 초기화)
- 해당 코드가 어떤 **파일시스템/네트워크 접근**을 할 수 있는가?
- Trust dialog 실패 시 이미 실행된 초기화를 **롤백**할 수 있는가?

Claude Code조차 CVE 4건을 겪었다는 사실은 이 gap이 **어느 하네스에서나 기본값 위험**임을 시사.

### 교훈 8: Evidence Tier를 전파가 아니라 **자체 검증**에

논문의 Tier A/B/C는 자체 하네스 문서화에도 직접 적용 가능:
- **Tier A**: 내부 ADR·design doc (원저자 명시)
- **Tier B**: 코드에서 추적 가능한 제약 (테스트 커버된 동작)
- **Tier C**: 관찰 패턴·합리적 추론 (untested)

하네스 README·CLAUDE.md에 Tier label을 부여하면 **"이건 보장되는가? 추측인가?"**를 사용자와 모델 모두가 구분 가능.

---

## 교차 검증 — deep-reader 관찰의 외부 맥락 기반 결론

| ID | 관찰 | 상태 | 근거 |
|---|---|---|---|
| A1 | 1.6/98.4 출처 불명확 | **부분 입증** | 커뮤니티 선행 측정(Pachaar, Odaily) 존재, 정의 경계는 불명 |
| A2 | Shen 순환 인용 의혹 | **반증 (아이러니 있음)** | Judy Hanwen Shen(Anthropic) ≠ Zhiqiang Shen(MBZUAI), 동명이인. **추가 함의**: Shen & Tamkin 2026(17% comprehension 하락)은 Anthropic 자체가 낸 증거인데 Claude Code 생산성 주장에 반한다 — 저자의 비판적 독립성 **강화** 방향 |
| A3 | OpenClaw 비교 공정성 | **입증** | OpenClaw는 메시징 게이트웨이, OPENDEV 등 더 적절한 대상 존재 |
| A4 | harness-heavy 주장 반례 부재 | **유지** | 외부 증거로도 해소 불가 |
| A5 | Source 추출 투명성 | **강화 입증** | 2026-03-31 npm source map 유출, 논문은 "publicly available"로 흐림 |
| A6 | Memoization staleness | **유지** | 외부 데이터 없음, 관찰 독립 유효 |
| A7 | 13 원칙 간 충돌 | **유지** | 외부 논의 미확인 |
| A8 | KAIROS 존재 여부 | **부분 유지 (Tier 분류 쟁점)** | KAIROS 자체는 소스에 150+ 참조로 Tier B 확인. 다만 production 미활성 상태를 "미래 방향의 구체적 illustration"으로 반복 인용하는 것은 Tier B(구조)와 Tier C(방향성 주장)를 한 문단에 섞는 서술상 쟁점. X @niki2ai가 동일 이름·개념의 always-on 백그라운드 에이전트를 독립 구현 — KAIROS 아이디어는 업계 병렬 탐색 중이며 Claude Code 고유 혁신 아님 |
| A9 | 6 개방 문제 크기 불균형 | **유지** | 독자 관찰로 유효 |
| A10 | Evaluative lens 격하 asymmetry | **유지** | 구조적 관찰로 유효 |

**불명확 지점 U1~U10 해소:**
- **해소**: U6 (ACP는 Agent Client Protocol, OpenClaw 공식 스펙 확인)
- **부분 해소**: U1 (7× 토큰 출처 Anthropic 2025b 확인, per-session/per-task 구분 여전히 불명)
- **미해소**: U2, U3, U4, U5, U7, U8, U9, U10 (소스 직접 확인 필요)

---

## 참고자료

### 논문·공식 자산
- 논문: https://arxiv.org/abs/2604.14228 (v1, 2026-04-14, CC BY-NC-SA 4.0)
- 공식 저장소: https://github.com/VILA-Lab/Dive-into-Claude-Code
- 독립 리뷰: https://arxiviq.substack.com/p/dive-into-claude-code-the-design

### 수치·주장의 1차 자료
- How AI Is Transforming Work at Anthropic (2025-09): https://www.anthropic.com/research/how-ai-is-transforming-work-at-anthropic
- Claude Code auto mode (Hughes 2026-03-25): https://www.anthropic.com/engineering/claude-code-auto-mode
- Making Claude Code more secure and autonomous (Dworken & Weller-Davies 2025): https://www.anthropic.com/engineering/claude-code-sandboxing
- Measuring agent autonomy (McCain et al. 2026): https://www.anthropic.com/research/measuring-agent-autonomy

### 선행·경쟁 연구
- OPENDEV (Bui, arXiv:2603.05344, 2026-03-05): https://arxiv.org/abs/2603.05344
- Speed at the Cost of Quality (He et al., arXiv:2511.04427): https://arxiv.org/abs/2511.04427
- METR 2025 (Becker et al., arXiv:2507.09089): https://arxiv.org/abs/2507.09089
- Decoding the Configuration of AI Coding Agents (arXiv:2511.09268)

### 유출 사건 맥락
- Alex Kim: "The Claude Code Source Leak" (2026-03-31): https://alex000kim.com/posts/2026-03-31-claude-code-source-leak/
- Jonas Kim: "Claude Code 내부 아키텍처 분석": https://bits-bytes-nn.github.io/insights/agentic-ai/2026/03/31/claude-code-source-map-leak-analysis.html
- HN 토론: https://news.ycombinator.com/item?id=47597085

### 커뮤니티 재구현
- instructkr/claw-code (Sigrid Jin, Rust+Python clean-room rewrite)
- WaveSpeedAI 리뷰: https://wavespeed.ai/blog/posts/what-is-claw-code/

### 하네스 담론 배경
- Firecrawl: https://www.firecrawl.dev/blog/what-is-an-agent-harness
- LangChain: https://blog.langchain.com/the-anatomy-of-an-agent-harness/
- Martin Fowler: https://martinfowler.com/articles/harness-engineering.html
- Philipp Schmid: https://www.philschmid.de/agent-harness-2026

---

## 미수집 영역 (투명 기록)

1. **논문 참고문헌 전수 검증 미완료** — OPENDEV 등 선행연구 인용 여부 bibliography 전수 조사 필요. deep-reader 정독상 명시 인용 없음으로 추정.
2. **저자별 개별 소속 — 부분 해소 + 표기 불일치 확인**: 저자 Zhiqiang Shen Group 공식 홈페이지(zhiqiangshen.com/projects/group)는 4명 전원 MBZUAI 기재. 논문 PDF 본문은 Xinyi Shang을 UCL 이중 소속(¹·²)으로 표기. **양 소스 간 불일치 상태**. 어느 쪽이 시점상 최신인지는 미확인.
3. **X/Twitter 스레드** — WebFetch 제약으로 직접 확인 실패. Akshay Pachaar 1.6% 인용은 context-scout 간접 확인.
4. **HN·Reddit 직접 스레드** — "Dive into Claude Code" 키워드 관련 스레드 확증 실패.
5. **HuggingFace Papers 투표·코멘트** — 페이지 파싱 실패.
6. **Google Scholar 피인용** — 제출 1주차라 반영 전.
7. **저자 발표 영상** — YouTube "The Architecture of Claude Code"가 저자 발표인지 미확증.
8. **불명확 지점 U2~U5, U7~U10** — 본문 외부 자료로 해소 불가, 소스 직접 확인 필요.
9. **"Rak 2025 - 25%" 인용** — 원출처 정확 매치 실패. Stanford Brynjolfsson et al. 2025이 근접 후보이나 인용 오기 가능성.
