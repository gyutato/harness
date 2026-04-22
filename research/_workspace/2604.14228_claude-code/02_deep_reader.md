# Dive into Claude Code: The Design Space of Today's and Future AI Agent Systems

- **저자:** Jiacheng Liu¹, Xiaohan Zhao¹, Xinyi Shang¹,², Zhiqiang Shen¹ (†corresponding)
  - ¹ VILA Lab, Mohamed bin Zayed University of Artificial Intelligence (MBZUAI)
  - ² University College London
- **게재:** arXiv 프리프린트 (cs.SE 주, cs.AI / cs.CL / cs.LG 부) — 2026-04-14 제출
- **식별자:** arXiv:2604.14228v1 / DOI 10.48550/arXiv.2604.14228
- **공개 자산:** https://github.com/VILA-Lab/Dive-into-Claude-Code
- **분석 대상:** Claude Code v2.1.88 (Anthropic npm 패키지에서 추출된 TypeScript 소스)
- **저작권 disclaimer:** "All materials obtained from publicly available online sources... original intellectual property rights to the source code belong to Anthropic."
- **분량:** 본문 37쪽 + 참고문헌 + 부록 A/B = 총 46쪽

---

## 1. 한 줄 요약

Claude Code를 *"의사결정은 모델, 집행은 하네스(harness)"*라는 철학을 실제 코드로 구현한 하나의 "design point"로 읽고, 512K LOC·1,884개 TS 파일의 **Tier B(코드 검증) 증거**를 기반으로 7 컴포넌트 / 5 계층 / 5 가치 / 13 원칙 / 9단계 턴 루프 / 5층 컨텍스트 샤퍼 / 7층 권한 / 4개 확장 메커니즘의 상호 일관성을 해부하고, OpenClaw와의 **비교를 통해 "같은 설계 질문, 다른 답"임을 보여 설계 공간을 드러낸 뒤, 6개 미래 개방 문제로 끝맺는 source-grounded 설계 공간 논문**이다.

---

## 2. 문제 정의 (Motivation)

### 2.1 논증의 출발점
AI 코딩 도구는 **autocomplete → IDE 통합 → agentic** 3단계로 진화해 왔으며 (Copilot → Cursor → Claude Code), Claude Code는 마지막 단계의 대표 사례다. 단순 제안(suggestion)에서 **자율 실행(autonomous action)**으로 넘어가면서 "완성형 도구"에는 존재하지 않던 설계 과제 — 안전, 컨텍스트, 확장, 위임, 지속성 — 이 전부 새로 등장했고, 이들이 하나의 **설계 공간(design space)**을 이룬다는 것이 출발점이다.

### 2.2 왜 굳이 Claude Code 소스 분석인가
저자들이 명시한 논리 사슬:

1. **사용자 문서는 공개되어 있으나 아키텍처 문서는 없다** — 따라서 외부 관찰자는 소스를 봐야만 한다.
2. Anthropic 내부 서베이(Huang et al., 2025; 엔지니어·연구자 132명)에 따르면 "Claude Code를 사용한 태스크 중 **약 27%는 도구가 없었다면 시도조차 하지 않았을 작업**"이다. 즉 도구가 단지 기존 워크플로를 빠르게 한 것이 아니라 **질적으로 새로운 워크플로**를 가능하게 했다. 따라서 이 아키텍처가 **무엇 때문에 그게 가능한지**를 해부할 실증적 가치가 있다.
3. 다만 Claude Code는 하나의 "점"에 불과하므로, **OpenClaw**(WhatsApp·Telegram 등 ~20여 개 메시징 채널을 통합하는 로컬 퍼스트 WebSocket 게이트웨이) 와 비교해 "같은 질문, 다른 답"의 축을 드러낸다.

### 2.3 논증 구조 (3부 구성)
1. **Design-space analysis (§3–§9)**: Claude Code 내부를 설계 질문별로 해부
2. **Architectural contrast with OpenClaw (§10)**: 6차원 비교
3. **Open directions (§12)**: 6개 개방 문제

### 2.4 "Evaluative lens" — 은근히 배치된 비판 축
저자들이 본문 §2.4에서 명시하는 중요한 장치: 5가지 가치 외에 **"장기 인간 역량 보존(long-term human capability preservation)"**을 **공동 가치**가 아닌 **평가 렌즈(evaluative lens)**로 설정했다.

근거:
- Anthropic 자체 서베이(Huang et al., 2025)에서 "paradox of supervision" 명시 — AI에 의존하면 AI를 감독할 기술이 위축된다.
- **Shen & Tamkin (2026)**: AI 지원 조건에서 이해도 테스트 점수 **17% 하락**.

→ 이 렌즈는 §11.2 표 4의 value tension, §11.4 empirical predictions, §12.6 개방 문제로 이어지는 **비판적 척추** 역할을 한다. 논문의 가장 날카로운 부분이 여기에 숨어 있다.

**독자 해석:** Shen & Tamkin (2026)의 "Shen"이 본 논문의 corresponding author Zhiqiang Shen과 동일 인물일 가능성이 높다. 저자 본인의 선행 연구를 자기 논문의 비판적 렌즈 근거로 쓰는 것은 순환 인용(self-serving citation)에 가까운 구조 — context-scout의 추가 확인 필요.

---

## 3. 핵심 아이디어

### 3.1 중심 주장 다섯 가지

**C1. 코어 루프는 얇다, 하네스가 두껍다.**
"An estimated **1.6% of the codebase constitutes AI decision logic**, with the remaining 98.4% being operational infrastructure" (§3.1, §11.1, §11.7). 커뮤니티 분석에서 빌린 수치로, 논문의 중심 주장을 뒷받침한다.

**C2. 모델 자율성 × 결정론적 하네스 = 설계 철학.**
"the model reasons and the harness enforces" (§11.1). 컴파일된 그래프(LangGraph)나 계획 스캐폴드(Devin)에 투자하는 대안 설계와 구분되는 포지션.

**C3. 5 가치 → 13 원칙 → 구현의 추적 가능성(traceability).**
표 1(Table 1)에 13개 원칙이 열거되고, 각 원칙은 (a) 어떤 가치를 섬기는지 (b) 어떤 설계 질문에 답하는지 (c) 어느 섹션에서 구현되는지로 매핑된다. 이것이 논문의 **아키텍처 지도**의 역할을 한다.

**C4. 7개 컴포넌트 ↔ 5 계층 서브시스템의 이중 서술.**
§3.2는 고수준 7-컴포넌트 모델(그림 1), §3.3은 세밀한 5-레이어 모델(그림 3). 두 모델이 소스 디렉토리와 어떻게 대응되는지 구체적 파일명으로 고정된다.

**C5. 같은 질문 → 다른 답 (OpenClaw 비교).**
6차원 비교 매트릭스(표 3)로 "deployment context가 architectural answer를 결정한다"는 메타 주장을 실증.

### 3.2 논증의 수사적 장치
- **Running example:** "Fix the failing test in auth.test.ts" 하나로 §3 ~ §9를 관통. 추상적 아키텍처를 구체 태스크에 고정.
- **Alternative-always-present:** 각 설계 선택을 제시할 때 반드시 대안 설계 가족(SWE-Agent/OpenHands, Aider, LangGraph, Devin)을 비교 언급 — "왜 이 선택인가"를 명시.
- **Evidence tier marking:** 부록 B.1에서 Tier A(product doc)/B(code-verified)/C(reconstructed)를 구분. 본문 대부분은 Tier B 주장.

---

## 4. 방법론 상세

### 4.1 개괄 — 소스 기반 설계 공간 분석(source-grounded design-space analysis)

방법론 자체는 논문의 핵심 기여 중 하나다. 절차(부록 B.2):

1. **설계 질문 식별(design-space analytic procedure)**: 각 서브시스템에서 다른 프로덕션 agent가 대안 설계를 가진 **choice point**를 찾는다.
2. **코드로 추적(trace to source)**: Claude Code의 답을 특정 파일·함수까지 고정한다(Tier B).
3. **가치 → 원칙 → 구현 매핑**: Anthropic 공식 문서·창업자 발언에서 5가치를 먼저 추출(Tier A), 13 원칙으로 전개, 구현과 연결.
4. **OpenClaw 비교**: ground truth가 아닌 **calibration point**로 사용.

**암묵적 가정:**
- v2.1.88 스냅샷이 설계 의도를 대표한다 (static snapshot 한계는 §B.3에서 인정).
- Feature flag가 가려진 코드도 분석에 포함 (TRANSCRIPT_CLASSIFIER, CONTEXT_COLLAPSE 등) — 즉 "만약 켜진다면 이렇게 동작한다"는 잠재적 behavior까지 포함.
- "Community analysis"를 독립적 증거로 인정 (Tier C).

### 4.2 핵심 수식·개념 설명

논문에 수식은 거의 없다. 유일하게 계산적인 주장은:

**compound error 계산** (§13.2, Huyen 2025 인용):

$$
P(\text{100-step task 성공}) = 0.95^{100} \approx 0.006 = 0.6\%
$$

의미: per-step 95% 정확도 모델도 100스텝 연쇄 과제에선 0.6% 성공률밖에 안 된다. → **per-step 검증(verification)** 디자인 원칙의 수학적 정당화. §4·§5의 permission gate, post-tool verification이 이 식의 결과다.

**그 외 수치 주장:**
- 1.6% / 98.4% 코드 비율 (C1 참조)
- 93% 승인율 (approval fatigue)
- 27% 새로운 태스크 (capability amplification)
- 20%→40% auto-approve (trust evolution, 세션 50개 → 750개)
- 84% 프롬프트 감소 (sandboxing 효과, Dworken & Weller-Davies 2025)
- 40.7% 복잡도 증가 (Cursor 기반, He et al. 2025) + velocity 첫 달 +281% → 3개월째 baseline
- 19% 느려짐 (AI tool, Becker et al. 2025, 16명 RCT, 246 태스크)
- 17% 이해도 점수 하락 (Shen & Tamkin 2026)
- 25% 주니어 기술직 채용 감소 (Rak 2025, 2023→2024)
- 78% AI 실패는 보이지 않음 (Bessemer 2026)
- 50개 이상 subcommand → per-subcommand deny check 무력화 (Adversa.ai 2026)

이 수치들은 **수식이라기보다 "설계 선택을 정당화하는 경험적 anchor"**로 기능한다.

### 4.3 알고리즘/아키텍처

#### 4.3.1 7 컴포넌트 고수준 구조 (그림 1 → §3.2)

| # | 컴포넌트 | 책임 | 주요 소스 |
|---|---------|------|----------|
| 1 | User | 프롬프트 제출, 권한 승인, 출력 검토 | — |
| 2 | Interfaces | 인터랙티브 CLI / 헤드리스 CLI / Agent SDK / IDE·Desktop·Browser | `src/entrypoints/`, `src/screens/`, `src/components/` |
| 3 | Agent Loop | 모델 호출 → 툴 디스패치 → 결과 수집의 iterative cycle | `query.ts` (`queryLoop()` AsyncGenerator) |
| 4 | Permission System | Deny-first 룰 평가 + ML 분류기 + hook 인터셉션 | `permissions.ts`, `yoloClassifier.ts`, `types/hooks.ts` |
| 5 | Tools | 최대 54개 빌트인(19 무조건 + 35 조건부) + MCP 툴 통합 | `tools.ts` `assembleToolPool()` |
| 6 | State & Persistence | JSONL 세션 transcript, prompt history, subagent sidechain | `sessionStorage.ts`, `history.ts` |
| 7 | Execution Environment | Shell, 파일시스템, 웹, MCP, 원격 | `BashTool.tsx`, `services/mcp/client.ts`, `src/remote/` |

좌→우 "spine": 사용자 → 인터페이스 → 루프 → 권한 → 툴 → 실행환경 / 그 옆에 state & persistence.

**애플리케이션 진입점:** `main.tsx`의 `main()`. 보안 설정(NoDefaultCurrentDirectoryInExePath로 Windows PATH hijacking 방지), 시그널 핸들러 등록 후 실행 모드별 분기.

#### 4.3.2 5 계층 서브시스템 분해 (그림 3 → §3.3)

| 계층 | 역할 | 주요 소스 |
|------|------|----------|
| **Surface** | entry & rendering: 인터랙티브 CLI, 헤드리스, SDK, IDE | `entrypoints/coreTypes.ts`, `controlSchemas.ts`, `coreSchemas.ts`, `screens/`, `components/` (ink framework) |
| **Core** | agent loop, compaction pipeline | `query.ts` (`queryLoop()`, 365–453행의 5 shapers) |
| **Safety/Action** | 7-layer permission, hook pipeline, extensibility, tools, sandbox, subagents | `permissions.ts`, `yoloClassifier.ts`, `types/hooks.ts`, `tools.ts` `assembleToolPool()`, `shouldUseSandbox.ts`, `AgentTool.tsx`, `runAgent.ts` |
| **State** | context assembly, runtime state, session persistence, memory, sidechains | `context.ts` `getSystemContext()`/`getUserContext()`, `src/state/`, `sessionStorage.ts`, `claudemd.ts`, `conversationRecovery.ts` |
| **Backend** | shell, remote, MCP, concrete tool 42종 | `BashTool.tsx`, `PowerShellTool.tsx`, `src/remote/`, `services/mcp/client.ts`, `src/tools/*/` |

**7-컴포넌트 ↔ 5-계층 대응 관계:**
- 7의 1(User)은 5계층 외부
- 7의 2(Interfaces) = 5의 Surface
- 7의 3(Agent loop) = 5의 Core
- 7의 4(Permission)·5(Tools)의 일부 = 5의 Safety/Action
- 7의 5(Tools)의 실제 구현·6(State)·7(Execution) = 5의 State·Backend
- 즉 5계층 모델은 Tools를 "schema 등록(Safety/Action의 tool pool)" vs "실행(Backend)"으로 나눈 점이 7-컴포넌트보다 세밀.

#### 4.3.3 QueryEngine vs queryLoop 구분 (§3.4)

저자가 명시적으로 **혼동 주의점**으로 기술:
- `QueryEngine.ts`는 "headless/SDK 경로를 위한 conversation wrapper"로 SDK·헤드리스 대상.
- 실제 공유되는 경로는 `query()` 함수(내부적으로 `queryLoop()` 호출).
- 인터랙티브 CLI도 `QueryEngine`을 거치지 않고 `query()`를 직접 호출 → **"공유 코드 경로는 engine 클래스가 아니라 loop 함수"**.

이 정정은 클래스 이름만 보고 아키텍처를 추정하면 틀릴 수 있음을 보여주는, 저자의 Tier B 방법론적 세심함의 사례다.

#### 4.3.4 queryLoop() 턴 실행 9단계 (§4.1)

Running example("Fix the failing test in auth.test.ts")이 한 turn 안에서 거치는 순서:

1. **Settings resolution.** `queryLoop()`이 immutable 파라미터(시스템 프롬프트, user context, 권한 콜백, 모델 설정)를 destructure.
2. **Mutable state initialization.** 하나의 `State` 객체에 모든 변경 가능한 상태(messages, tool context, compaction tracking, recovery counter). 루프의 **7개 continue site**는 전부 **whole-object reassignment**로 갱신 — 필드 개별 뮤테이션 대신 한 번에 교체하는 설계.
3. **Context assembly.** `getMessagesAfterCompactBoundary()`로 마지막 compact 경계 이후 메시지를 가져온다. 즉 이미 요약된 구간은 원본이 아니라 요약으로 대체되어 있다.
4. **Pre-model context shapers.** 5 샤퍼 순차 실행(§4.3, §7.3).
5. **Model call.** `for await (... of deps.callModel())`으로 스트리밍 — 메시지(user context prepend), 시스템 프롬프트 전체, thinking config, 가능한 툴셋, abort signal, 모델 스펙, fast-mode·effort·fallback model 옵션 전달.
6. **Tool-use dispatch.** 응답에 `tool_use` 블록이 있으면 툴 오케스트레이션 계층으로 흐른다(§4.2).
7. **Permission gate.** 각 툴 요청이 permission system을 통과(§5).
8. **Tool execution & result collection.** `tool_result` 메시지로 대화에 추가되고 루프가 이어진다.
9. **Stop condition.** text-only 응답(no tool_use)이면 턴 종료.

**이 루프가 AsyncGenerator인 이유:** UI 계층으로 StreamEvent, RequestStartEvent, Message, TombstoneMessage, ToolUseSummaryMessage를 yield하면서도 루프 내부에선 단일한 synchronous control flow를 유지하는 **generator-based streaming design**.

**패턴 위치:** ReAct(Yao et al., 2022)의 표준 reactive 루프. 대안 — LangGraph의 명시적 state graph, LATS의 트리 탐색(Zhou et al., 2023). Claude Code는 subagent 위임(§8)에선 orchestrator-workers 패턴(Schluntz & Zhang 2024 의 5패턴 중 하나)을 쓰지만 **core loop는 reactive**를 유지. **단일 action sequence에 커밋, 백트래킹 없음** — 탐색 완전성(search completeness)을 simplicity·latency와 맞바꿈.

#### 4.3.5 Tool dispatch의 이중 경로 (§4.2)

두 실행 경로:

- **Primary: `StreamingToolExecutor`** — 모델 응답이 스트리밍되는 대로 툴 실행 시작. 다중 툴 응답 latency 감소.
- **Fallback: `runTools()` (toolOrchestration.ts)** — `partitionToolCalls()` partition 순회.

공통: 툴을 **concurrent-safe vs exclusive**로 분류. Read-only는 병렬, state-modifying(shell 등)은 직렬.

`StreamingToolExecutor.ts`의 두 조정(coordination) 메커니즘:
- **Sibling abort controller**: 어떤 Bash 툴이 에러 나면 다른 in-flight subprocess들 즉시 종료.
- **Progress-available signal**: 새 output 준비 시 `getRemainingResults()` consumer 깨움.

**Result ordering:** 버퍼링하여 툴 **요청된 순서**대로 emit. 모델이 tool_use 순서와 동일한 tool_result 순서를 기대하므로 중요. → **concurrent-read, serial-write execution model**. PASTE(Sui et al., 2026)의 speculative 선행 실행 대비 절충적 중간 지점.

**특이 사항:** `hook_stopped_continuation` attachment 감지 시 `shouldPreventContinuation` 플래그 설정 → PostToolUse hook이 루프 중단 가능.

#### 4.3.6 5층 컨텍스트 샤퍼 (§4.3, §7.3) — 논문의 핵심 분석 대상

`query.ts`에서 모델 호출 **직전**에 순차 실행되는 5 shapers. `messagesForQuery` 배열을 점진적으로 압축. 설계 근거는 **"no single compaction strategy addresses all types of context pressure"** — 각 층은 다른 비용-효과 트레이드오프에 대응.

| # | 샤퍼 | 게이트 플래그 | 대상 | 메커니즘 | 비용 |
|---|------|--------------|------|---------|------|
| 1 | **Budget reduction** | 항상 활성 | 개별 tool result | `applyToolResultBudget()`: per-message size limit, 초과분을 content reference로 대체. Exempt 툴(`maxResultSizeChars` non-finite)은 유지. Content replacement는 resume 시 재구성 위해 지속화. | 개별 output만 건드림, 가장 가벼움 |
| 2 | **Snip** | `HISTORY_SNIP` | 오래된 history 세그먼트 | `snipCompactIfNeeded()`: 경량 trim. `{messages, tokensFreed, boundaryMessage}` 반환. 중요 기술 디테일: 토큰 카운터는 "most recent assistant message의 usage 필드"에서 파생되므로 그 메시지가 pre-snip input_tokens를 유지한 채 살아남으면 **snip 절감이 카운터에 보이지 않는다** → `snipTokensFreed`를 auto-compact로 명시적으로 plumb해야 함. | 낮음 (시간 기반 trim) |
| 3 | **Microcompact** | 시간 기반은 항상, 캐시 기반은 `CACHED_MICROCOMPACT` | 툴 결과 + 메시지 | 세밀 압축. Cache-aware 경로: boundary message를 API 응답 후로 defer해서 실제 `cache_deleted_input_tokens`를 쓰게 함 (추정값 대신). `{messages, compactionInfo}` 반환, `pendingCacheEdits` 포함 가능. **tool_use_id로만 동작하고 content를 inspect하지 않으므로 budget reduction 이후에 깨끗하게 조합됨**. | 중간, 캐시 영향 고려 |
| 4 | **Context collapse** | `CONTEXT_COLLAPSE` | 전체 history | **read-time projection**(소스 주석: "Nothing is yielded; the collapsed view is a read-time projection over the REPL's full history. Summary messages live in the collapse store, not the REPL array."). REPL 저장소를 변경하지 않음; `applyCollapsesIfNeeded()`가 `messagesForQuery`만 교체 — 모델은 collapsed 버전을 보지만 full history는 재구성용으로 보존. | 중간 |
| 5 | **Auto-compact** | 기본 활성, 사용자 비활성화 가능 | 전체 대화 | `compactConversation()`(compact.ts)가 **모델 기반 요약 생성**. PreCompact hooks 실행 → `getCompactPrompt()`로 summary request 생성 → 모델 호출 → `buildPostCompactMessages()`로 출력. | 높음 (추가 모델 호출) |

**왜 이 5층 순서인가 — 저자의 설계 근거 4가지**

1. **Lazy degradation**: "least disruptive 먼저, 비싼 전략은 나중에". 각 층은 실패 시에만 다음 층으로 escalate.
2. **Layer composition cleanness**: budget reduction은 content 변경, microcompact는 ID 기반 → 의미 간섭 없음.
3. **Different pressure sources**: tool output 비대(budget), 시간 깊이(snip), 캐시 오버헤드(microcompact), 긴 history(collapse), 의미 압축(auto-compact).
4. **Auto-compact는 최후 수단**: 4층이 전부 실행된 뒤에도 pressure threshold를 초과할 때만 발동.

**비용:** "Five interacting compression layers, several gated by feature flags, create behavior that is difficult for users to fully predict." 특히 context collapse는 **user-visible output 없이** 동작한다 (auto-compact는 transcript에 summary 가시, microcompact는 boundary marker만).

**buildPostCompactMessages() 출력 구조** (compact.ts):
```
[boundaryMarker, ...summaryMessages, ...messagesToKeep, ...attachments, ...hookResults]
```
Boundary marker는 `annotateBoundaryWithPreservedSegment()`로 **headUuid, anchorUuid, tailUuid**를 기록 → read-time chain patching 가능. **mostly-append 설계**: compaction은 기존 transcript를 수정·삭제하지 않고 새 boundary·summary만 append.

**compactConversation()의 세부 디자인 포인트들:**
- PreCompact hook이 먼저 발동 → 사용자 정의 instruction 주입 가능.
- GrowthBook feature flag로 compaction 경로가 main conversation의 prompt cache를 재사용할지 결정. 소스 주석 인용: "false path is 98% cache miss, costs ~0.76% of fleet cache_creation" (2026-01 실험 기록).
- Compaction 후 attachment builder가 plans/skills/async agents 상태를 live app state에서 다시 announce (기존 attachment는 버려지므로).

#### 4.3.7 7 권한 레이어 — Defense in Depth (§3.5, §5)

"A request must pass through all applicable layers, and any single layer can block it." — 즉 AND-composition이 아닌 **any-block-denies** 로직.

| 층 | 메커니즘 | 소스 | 차단력 |
|----|---------|------|--------|
| 1 | **Tool pre-filtering** | 모델 뷰에서 blanket-deny 툴 제거 (assembleToolPool 이전) | `tools.ts` `filterToolsByDenyRules()` | 모델이 아예 호출 시도 불가. MCP 서버-prefix 룰(`mcp__server`)로 서버 전체 strip 가능 |
| 2 | **Deny-first rule evaluation** | `toolMatchesRule()`: deny가 allow보다 항상 우선 (구체성 무관) | `permissions.ts` | "allow npm test"가 있어도 "deny all Bash"가 이김 |
| 3 | **Permission mode constraints** | 활성 모드가 룰 매칭 없는 요청의 baseline 결정 | `types/permissions.ts` | 7 모드 |
| 4 | **Auto-mode classifier** | ML 기반, 룰 시스템이 allow한 것도 deny 가능 | `yoloClassifier.ts` | TRANSCRIPT_CLASSIFIER 게이트 |
| 5 | **Shell sandboxing** | 승인된 shell 명령도 sandbox 내 실행 가능 | `shouldUseSandbox.ts` | 파일시스템·네트워크 격리, 권한 시스템과 **독립축** |
| 6 | **Not restoring permissions on resume** | 세션-scoped 권한이 resume/fork 시 복원되지 않음 | `conversationRecovery.ts` | 세션을 "isolated trust domain"으로 간주 |
| 7 | **Hook-based interception** | PreToolUse hook으로 permission 결정 수정, PermissionRequest hook으로 비동기 해결 | `types/hooks.ts` | coordinator 모드에선 dialog 전 실행 가능 |

**7 권한 모드 상세 (§5.1):**

외부 5 모드(`EXTERNAL_PERMISSION_MODES` 배열) + auto(조건부) + bubble(type union 전용):

1. **plan**: 모델은 plan을 만들어야 하며 사용자 승인 후에만 실행.
2. **default**: 표준 인터랙티브. 대부분 승인 필요.
3. **acceptEdits**: working dir 내 edit + 특정 fs shell 명령(mkdir, rmdir, touch, rm, mv, cp, sed) 자동 승인, 나머지 shell은 승인 필요.
4. **auto**: ML 분류기가 fast-path check 통과 못한 요청 평가 (TRANSCRIPT_CLASSIFIER 게이트).
5. **dontAsk**: 프롬프트 없음, 하지만 deny rule은 여전히 적용.
6. **bypassPermissions**: 대부분 스킵, 하지만 safety-critical check와 bypass-immune rule은 유지.
7. **bubble**: 내부 전용, subagent 권한을 parent terminal로 escalate.

**Graduated autonomy spectrum:** plan(최대 감독) → default → acceptEdits → auto → dontAsk → bypassPermissions(최소 감독). "autonomy가 증가할수록 interactive approval → automated safety check로 이동."

**Approval fatigue → 디자인 반응 (중요 논증):**
"when Anthropic found that users approve **93% of permission prompts** (Hughes, 2026), the response was not to add more warnings but to restructure the problem: defined boundaries (sandboxing, auto-mode classifiers) within which the agent can work freely, rather than per-action approvals that users stop reviewing once habituated."

→ **Human Decision Authority**와 **Safety**가 "사람이 결정하는 힘" vs "시스템이 보호할 의무"로 분리되는 이유: 승인 피로 때문에 사람의 vigilance가 신뢰할 수 없다면, 안전은 사람의 주의와 **독립적**으로 작동해야 함.

**Auto-mode classifier (§5.3):**
TRANSCRIPT_CLASSIFIER 활성 시 `yoloClassifier.ts`가 세 프롬프트 리소스 로드: base system prompt, external permissions template, Anthropic-internal 전용 template. Transcript와 permission template 대비 툴 invocation 평가 → allow/deny/manual. `isUsingExternalPermissions()`가 USER_TYPE + `forceExternalPermissions` 플래그로 템플릿 선택.

**Hook 기반 세부 제어 — 27개 이벤트 중 5개가 권한 직접 관여:**
- `PreToolUse`: permissionDecision (deny/ask, allow는 후속 체크 bypass 안 함), permissionDecisionReason, updatedInput 반환 가능
- `PostToolUse`: additionalContext 주입, MCP 툴에 대해 updatedMCPToolOutput
- `PostToolUseFailure`: error-specific guidance
- `PermissionDenied`: auto-mode 거부 후 retry guidance
- `PermissionRequest`: allow/deny 결정 — coordinator 경로에선 dialog 전 해결, 인터랙티브에선 dialog와 병행 가능

**Permission handler 4분기 (§5.2 useCanUseTool.tsx):**
1. **Coordinator**: multi-agent coordination 모드. 자동 해결(classifier, hooks, rules) 먼저, 실패 시 사용자 개입.
2. **Swarm worker**: swarm 워커 에이전트 자체 해결 로직.
3. **Speculative classifier**: BASH_CLASSIFIER 활성 + BashTool 시, 미리 시작된 classification 결과를 timeout과 경주. high confidence면 사용자 인터랙션 없이 즉시 승인.
4. **Interactive (fallback)**: 표준 사용자 승인 dialog.

**Denial as routing signal (§5.2 핵심 디자인 철학):**
"When the classifier or a deny rule blocks an action, the system treats the denial as a routing signal rather than a hard stop: the model receives the denial reason, revises its approach, and attempts a safer alternative." → permission 거부는 **에이전트 행동을 halt하는 게 아니라 shape**한다. PermissionDenied hook으로 외부에서 관찰·개입 가능.

**>50 subcommand fallback (§5.4, §11.3) — defense-in-depth의 균열:**
Adversa.ai 2026이 보고. 50개 초과 subcommand를 가진 명령은 per-subcommand deny check를 돌리지 않고 **단일 generic approval prompt로 fallback**. 이유: per-subcommand 파싱이 UI freeze 유발. → **"defense-in-depth can degrade when its layers share failure modes"** — 공식적으로 다층 방어지만 성능 제약을 공유하면 동시 실패.

#### 4.3.8 확장 메커니즘 4종 — 비용·격리 분리 (§6)

저자의 중심 설계 명제: "different kinds of extensibility impose different costs on the context window, and a single mechanism cannot span the full range from zero-context lifecycle hooks to schema-heavy tool servers without forcing unnecessary trade-offs."

즉 4종은 **context cost에 따라 의도적으로 계층화된 graduated stack** (Table 2):

| 메커니즘 | 고유 기능 | Context Cost | Insertion Point | 관련 소스 |
|---------|---------|-------------|----------------|----------|
| **MCP servers** | 외부 서비스 통합(multi-transport: stdio/SSE/HTTP/WS/SDK/IDE 변형) | **High (tool schemas)** | `model():` tool pool | `services/mcp/config.ts`, `services/mcp/client.ts`, `MCPTool` objects |
| **Plugins** | 다중 컴포넌트 패키징 + 배포 (10가지 component type) | Medium | All three points | `utils/plugins/schemas.ts` `PluginManifestSchema`, `utils/plugins/pluginLoader.ts` |
| **Skills** | 도메인 instruction + meta-tool 호출 | **Low (descriptions only)** | `assemble():` context injection | `loadSkillsDir.ts` `parseSkillFrontmatterFields()` (15+ fields) |
| **Hooks** | lifecycle interception + event-driven automation | **Zero by default** | `execute():` pre/post tool | `coreTypes.ts` 27 events, `types/hooks.ts` 15 schemas, `schemas/hooks.ts` 4 command types |

**Figure 5의 3 injection point 관점(pseudocode 형태 — 섹션 6에 포함):**
- `a assemble()`: 모델이 **보는** 것 — CLAUDE.md, skill descriptions, MCP resources/prompts, output style, UserPromptSubmit·SessionStart hook
- `b model()`: 모델이 **닿을 수 있는** 것 — Built-in tools, MCP tools, SkillTool(meta), AgentTool(meta)
- `c execute()`: 액션이 **어떻게 실행되는지** — Permission rules, PreToolUse/PostToolUse/Stop/SubagentStop/Notification hooks

**Hook 27개 이벤트 카테고리 (§6):**
- Tool authorization (5): PreToolUse, PostToolUse, PostToolUseFailure, PermissionRequest, PermissionDenied
- Session lifecycle (5): SessionStart, SessionEnd, Setup, Stop, StopFailure
- User interaction (3): UserPromptSubmit, Elicitation, ElicitationResult
- Subagent coordination (5): SubagentStart, SubagentStop, TeammateIdle, TaskCreated, TaskCompleted
- Context management (4): PreCompact, PostCompact, InstructionsLoaded, ConfigChange
- Workspace (4): CwdChanged, FileChanged, WorktreeCreate, WorktreeRemove
- Notifications (1)

15개는 event-specific output schema 보유. 4가지 persistable command type: `command`(shell), `prompt`(LLM), `http`, `agent`(agentic verifier). 추가로 SDK·internal instrumentation용 non-persistable `callback` 타입.

**Tool pool 조립 5단계 (§6.2 assembleToolPool()):**
1. Base tool enumeration (`getAllBaseTools()`: 19 always + 35 conditional = up to 54, internal user·worktree·swarm·Bun embedded search에 따라 달라짐)
2. Mode filtering (`getTools()`: `CLAUDE_CODE_SIMPLE`은 Bash/Read/Edit만, 각 툴의 `isEnabled()`)
3. Deny rule pre-filtering
4. MCP tool integration (deny rule 필터 후 빌트인과 merge)
5. Deduplication (이름 기준, 빌트인 우선)

**왜 한 개 API가 아닌가 (§6.3):**
Zero-context hook ↔ schema-heavy MCP server의 **비용 범위**를 단일 메커니즘으로 커버하면 extension author에게 불필요한 trade-off 강제. 대신 "cheap extension은 널리 scale, expensive는 진짜 필요할 때만".

**개인 소감(독자 해석):** 4 메커니즘의 비용 ordering이 실제로 graduated인지는 구현자에게만 명확하고, **사용자는 여전히 "어떤 확장을 언제 써야 하는가"를 배워야** 한다는 학습 곡선 비용(§6.3 말미에 저자도 명시)이 발생한다. 이 graduation은 시스템 설계자의 이익이지 사용자의 이익이 아닐 수 있다.

#### 4.3.9 Subagent 위임·격리 (§8)

**Agent tool** (`AgentTool.tsx`, `Task` legacy alias 유지)이 delegation 진입점. Input schema는 **feature-gated**: isolation 필드는 internal user에겐 `['worktree', 'remote']`, external에겐 `['worktree']`; cwd는 플래그 게이트; run_in_background는 조건부 생략.

**6개 빌트인 subagent type (§8.1):**
- **Explore**: read/search 중심, write·edit은 deny-list
- **Plan**: 구조화된 plan 생성, 실행은 표준 permission model
- **General-purpose**: 광범위 capability, 명시 요청 시에만 (type 생략 시 fork-subagent로 routing 가능)
- **Claude Code Guide**: 온보딩·문서, 고유 permissionMode override
- **Verification**: 테스트 suite, lint 등 검증
- **Statusline-setup**: 터미널 statusline 설정 전용

**Custom agents** (`.claude/agents/*.md`, plugin contrib via `loadPluginAgents.ts`): markdown body = system prompt, YAML frontmatter = description/tools allowlist/disallowedTools/model/effort/permissionMode/mcpServers/hooks/maxTurns/skills/memory scope/background/isolation mode. JSON 포맷도 지원(`prompt` 명시 필드 추가, `loadAgentsDir.ts`).

**AgentTool vs SkillTool 핵심 차이:**
- `SkillTool`: instruction을 **현재 context window에 주입**
- `AgentTool`: **새로운 isolated context window 생성** (default 경로는 parent history 상속 안 함, fork-subagent 경로는 예외) → subagent invocation은 대부분 **self-contained prompt** 필요

**Isolation 3 modes (§8.2):**
- **Worktree**: 임시 git worktree 생성, subagent가 repo copy 소유. 부모 working tree에 영향 없음.
- **Remote** (internal-only): Claude Code Remote 환경에서 백그라운드 실행.
- **In-process (default)**: 파일시스템은 공유하나 conversation context 격리.

**Permission override logic** (runAgent.ts, §8.2):
- Subagent `permissionMode` 정의 시 override 적용 — **단** parent가 bypassPermissions/acceptEdits/auto 중이면 parent 우선 (사용자의 명시적 safety/autonomy 선택 존중).
- Async agent의 prompt 회피: `canShowPermissionPrompts` → bubble 모드 → default (sync=show, async=hide) cascade.
- Background + prompt 가능 agent는 `awaitAutomatedChecksBeforeDialog: true`로 사용자 인터럽트 최소화.

**Two-tier permission scoping (§8.2):**
- `allowedTools`가 runAgent()에 명시될 때: SDK-level (`--allowedTools`)는 보존, session-level rule은 subagent 선언으로 **대체**.
- 명시 안 될 때(일반 AgentTool 경로): parent session-level rule 상속.

**Sidechain transcripts (§8.3):**
각 subagent가 자신의 `.jsonl` + `.meta.json`을 별도 기록. "Parent session file을 부풀리지 않되 subagent history는 감사·디버깅용으로 보존." 부모에게 돌아가는 것은 **subagent의 final response text와 metadata뿐** — "context as bottleneck" 원칙 준수.

**runAgent() — 21개 파라미터:** agent definition, prompts, permissions, tools, model settings, isolation, callbacks 등.

**Multi-agent 코스트 (§8.3):**
"Claude Code's agent teams consume approximately **7× the tokens** of a standard session in plan mode (Anthropic, 2025b)." → summary-only return은 비용 억제를 위해 더 critical.

**Team 코디네이션:** File locking 기반(메시지 브로커·분산 coordination 서비스 아님). Task는 lock-file mutex로 claim. Trade-off: 처리량을 희생하고 **zero-dependency deployment + 완전한 debuggability** 획득(plain-text JSON).

#### 4.3.10 컨텍스트 조립·메모리 (§7)

**Context window 조립 순서 (§7.1, Figure 6):**
1. System prompt (+ output style mod, `--append-system-prompt`)
2. Environment info (`getSystemContext()`: git status, BREAK_CACHE_COMMAND gated cache-break). Memoized.
3. CLAUDE.md hierarchy (`getUserContext()`: 4-level). Memoized.
4. Path-scoped rules (conditional, directory-matched, lazy on read)
5. Auto memory (비동기 prefetch)
6. Tool metadata (skill description, MCP tool name, deferred tool via ToolSearch)
7. Conversation history (compaction 대상)
8. Tool results (file reads, command output, subagent summary)
9. Compact summary

**구조적 배치 결정 (§7.1 미묘한 포인트):**
- `asSystemPrompt(appendSystemContext(systemPrompt, systemContext))()` — system context는 system prompt에 append
- `prependUserContext()` — CLAUDE.md + date는 **user message로 prepend**
→ CLAUDE.md 내용이 API request에서 system prompt와 다른 structural position 점유 → **모델 attention pattern에 영향 가능**.

**Late injection sources:** relevant-memory prefetch, MCP instruction delta (new/changed만), agent listing delta, background agent task notification. → context window는 assembly 후에도 **turn 중 동적 성장 가능**.

**CLAUDE.md 4 level (§7.2 claudemd.ts):**
1. **Managed memory** (예: `/etc/claude-code/CLAUDE.md`): OS-level policy.
2. **User memory** (`~/.claude/CLAUDE.md`): private global.
3. **Project memory** (`CLAUDE.md`, `.claude/CLAUDE.md`, `.claude/rules/*.md`): codebase committed.
4. **Local memory** (`CLAUDE.local.md`): gitignored, private project-specific.

File discovery는 CWD에서 root로 traverse. "**Later-loaded files receive more model attention**" — 로딩 우선순위 reverse. Root→CWD까진 `.claude/rules/*.md`의 unconditional rule이 startup에 eager load. CWD 이하 nested dir는 **lazy** — "the model's instruction set can evolve during a conversation as new parts of the codebase are explored."

**중요한 설계 분리 (§7.2):**
> "CLAUDE.md content is delivered as **user context (a user message)**, not as system prompt content (context.ts). This architectural choice has a significant implication: because CLAUDE.md content is delivered as conversational context rather than system-level instructions, **model compliance with these instructions is probabilistic rather than guaranteed**. Permission rules evaluated in deny-first order (Section 5) provide the deterministic enforcement layer. This creates a deliberate separation between guidance (CLAUDE.md, probabilistic) and enforcement (permission rules, deterministic)."

→ **CLAUDE.md 는 suggestion, permission 은 enforcement** 로 두 책임을 분리. 보안·정책은 CLAUDE.md에 적지 말라는 의미.

**@include 디렉티브** (processMemoryFile at claudemd.ts):
- `@path`, `@./relative`, `@~/home`, `@/absolute` 지원
- leaf text node에서만 동작 (code block 내 무시)
- including file 먼저 push → included file append
- 순환 참조는 processed path tracking으로 방지
- 없는 파일은 silently ignore

**Auto memory retrieval:** **no embeddings**. LLM-based scan of memory-file headers, up to 5 files on demand, file granularity. Embedding 기반 대안 대비 **inspectability와 infrastructure simplicity** 우선. (Trade-off: entry granularity 불가)

#### 4.3.11 Session 지속성 (§9)

**Transcript model (§9.1):**
- "Mostly append-only JSONL files at project-specific path" — 명시적 cleanup rewrite만 예외
- `getTranscriptPath()`: `join(projectDir, ${getSessionId()}.jsonl)` 
- `projectDir` 결정: `getSessionProjectDir()` (resume/branch 시 switchSession이 설정) → fallback to `getProjectDir(getOriginalCwd())`

**3 persistence channels (독립적):**
1. **Session transcript**: user/assistant/attachment/system message + compaction·metadata event (project-scoped, 1 file/session)
2. **Global prompt history**: user prompt only, `history.jsonl` at Claude config home. `makeHistoryReader()` generator는 `readLinesReverse()`로 역순 — Up-arrow·Ctrl+R 네비게이션.
3. **Subagent sidechain**: `.jsonl` + `.meta.json` per subagent

**Transcript 이벤트 종류:** compaction marker, file-history snapshot, attribution snapshot, content-replacement record 등 단순 메시지 외 여러 종류.

**Append-only JSONL의 이유 (§9.1):**
- Auditability·simplicity 우선
- Human-readable, version-controllable, 특수 도구 없이 재구성 가능
- DB 대안: query power 좋지만 deploy dependency·불투명성

**Session identity:** `sessionId` + `sessionProjectDir` 쌍, resume/branch 시 함께 설정 — 메시지 작성 당시의 project directory를 사용해야 hook이 올바른 디렉토리에서 작동.

**Resume/fork — permission 비복원 (§9.2, §5의 layer 6):**
`--resume`은 transcript replay로 대화 복원(`conversationRecovery.ts`), fork는 기존 세션에서 새 session 생성(`commands/branch/branch.ts`). 그러나 **session-scoped permission은 복원하지 않는다**. 근거:
- "Sessions are treated as isolated trust domains"
- 복원하면 stale trust가 changed context에 carry over 위험
- 재부여의 user friction 비용을 **trust invariant 유지**로 보상

**compact_boundary 마커 설계:** `annotateBoundaryWithPreservedSegment()`가 headUuid/anchorUuid/tailUuid 기록 → 세션 로더가 **read-time**에 message chain 패치. Preserved message는 원본 parentUuid를 디스크에 유지, loader가 metadata로 link.

**"Checkpoint"의 오해:** Claude Code의 checkpoint는 **file-history checkpoint for --rewind-files** (`~/.claude/file-history/<sessionId>/`에 저장). **파일 레벨 스냅샷으로 filesystem 되돌리기용이지 generic checkpoint store 아님.**

---

## 5. 실험 설계

### 5.1 데이터셋 / 베이스라인 / 지표

**이 논문은 실험 논문이 아니라 source-level 아키텍처 분석 논문이므로 통상적 ML 실험 구성은 없다.**

대신 **방법론적 대응물**:

- **데이터셋** ≈ Claude Code v2.1.88 소스 코퍼스
  - TypeScript 1,884 파일
  - 약 512K lines of code
  - 공개 npm 패키지에서 추출 (구체 추출 도구·URL은 논문 본문에 미명시)
- **베이스라인** ≈ OpenClaw (Steinberger and OpenClaw Contributors, 2026) — 6차원 비교용 **calibration point**
  - 주의: "ground truth가 아닌 calibration rather than ground truth" (§B.1)
- **보조 증거** — alternative agent systems (SWE-Agent, OpenHands, Aider, Devin, LangGraph, AutoGen, Codex CLI, Cursor, Copilot)
- **지표** ≈ **Evidence tier system** (§B.1):
  - **Tier A** — product-documented: 공식 문서·엔지니어링 publication
  - **Tier B** — code-verified: 특정 파일·함수 인용 (**가장 강한 tier**)
  - **Tier C** — reconstructed: community analysis, OpenClaw 비교, code pattern 추론 — hedging 언어 사용

### 5.2 구현 세부

- **분석 범위 명시 제한 (§B.3):**
  - **Static snapshot** — v2.1.88 한 버전
  - **Feature flag variability** — TRANSCRIPT_CLASSIFIER, CONTEXT_COLLAPSE 등이 build-time에 상이한 실효 아키텍처 생성
  - **Reverse-engineering epistemology** — 구조·제어 흐름·의존성·flag는 밝힐 수 있으나 **design intent, enabled production flag, runtime prevalence, unshipped behavior는 확인 불가**
  - **Single-system analysis** — Claude Code 외 일반화 bounded
  - **OpenClaw snapshot** — 특정 개발 시점, 현재 capability 대표 못 할 수 있음

---

## 6. 결과 해석

### 6.1 주요 결과

실험 수치가 없으므로 결과는 **아키텍처 발견(findings)**으로 해석:

**F1. 7-컴포넌트 + 5-계층 + 5-value + 13-principle의 일관성이 실증됨 (§3 전체).**
각 원칙이 구체 소스 파일까지 추적되고, value → 원칙 → 구현의 맵(§2.3)이 누수 없이 성립. **"absences also revealing"** — "it does not impose explicit planning graphs on the model's reasoning, does not provide a single unified extension mechanism, and does not restore all session-scoped trust-related state across resume" (§2.3) — 없는 것도 원칙에서 도출.

**F2. queryLoop()의 9단계가 ReAct 표준과 일치하되, "continue-sites + AsyncGenerator + whole-object state reassignment"라는 구현 관용이 추가 (§4.1).**

**F3. 5 shaper의 비용-효과 graduation이 합리적이되 사용자 예측 불가능성이 단점 (§7.3).**

**F4. 7 permission layer의 defense in depth는 >50 subcommand 사례에서 **독립성 가정 붕괴** 확인됨 (§5.4, §11.3 Adversa.ai 2026).**

**F5. 4 확장 메커니즘의 context cost ordering (Hook 0 → Skill low → Plugin medium → MCP high)은 graduated extensibility 설계 의도와 구현의 일치를 보여주는 가장 깔끔한 mapping (§6.3 Table 2).**

**F6. Subagent는 isolated context + summary-only return으로 context bottleneck을 존중하나, **7× 토큰 비용**과 cross-agent consistency 공백이라는 cost를 동반 (§8.3).**

**F7. Session transcript의 append-only + read-time chain patching은 auditability·resume/fork를 동시에 달성하나, session-scoped permission의 deliberate 비복원은 "trust invariant > user convenience"의 설계 선택 (§9.2).**

**F8. OpenClaw 비교 6차원에서 "대립적" 패턴 (§10):**
- 시스템 범위: ephemeral CLI ↔ persistent daemon
- 신뢰 모델: per-action deny ↔ perimeter-level access
- 런타임: loop = center ↔ loop = component in gateway
- 확장: single agent context ↔ gateway-wide capability registry
- 메모리: 5-layer compaction ↔ long-term memory promotion (dreaming, daily notes)
- 멀티에이전트: task delegation ↔ multi-agent routing + sub-agent delegation

**F9. 결정적 compositional 관찰:** "OpenClaw can host Claude Code... via ACP, meaning the two systems are composable rather than exclusive alternatives" — 설계 공간은 **flat taxonomy 가 아니라 layered**.

### 6.2 Ablation / 분석

ablation은 없으나 **value tension table (Table 4, §11.2)**이 ablation의 수사적 대응물:

| Value pair | Tension | Evidence |
|-----------|---------|----------|
| Authority × Safety | 승인 피로 vs 보호 | 93% 승인율 |
| Safety × Capability | 성능 vs defense depth | >50 subcommand fallback |
| Adaptability × Safety | 확장성 vs attack surface | CVE 4건 — pre-trust initialization |
| Capability × Adaptability | proactivity vs disruption | 12–18% task 증가, 고빈도 시 선호도 하락 |
| Capability × Reliability | velocity vs coherence | bounded context, subagent isolation, 복잡도 증가 |

**Pre-trust initialization ordering 발견 (§11.3):**
두 취약점(CVE-2025-59536 CVSS 8.7, CVE-2026-21852 CVSS 5.3, Donenfeld & Vanunu 2026, Check Point Research)이 공통 root cause 공유: **초기화 중 실행되는 코드(hook, MCP 서버 connection, setting 파일 해결)가 interactive trust dialog 제시 전에 돌아감**. 이 **pre-trust execution window**는 deny-first 파이프라인 바깥에 있어 §5의 안전 보장이 **temporal하게 미적용**.

→ 저자의 핵심 관찰: "Figure 4가 safety check의 spatial ordering을 보여주지만 **temporal dimension은 포착하지 않는다**." 초기화 시퀀스(extension loading → trust dialog → permission enforcement)가 extensibility와 safety 사이의 window를 연다.

**추가 CVE:** CVE-2025-54794, CVE-2025-54795 (Beber 2025) — path validation + command parsing flaw, 위 두 개와 별도. 네 건 모두 공개 후 수 주 내 패치됨.

**Empirical predictions (§11.4):**
저자는 아키텍처 속성으로부터 **검증 가능한 예측**을 끌어낸다:
1. Bounded context → pattern duplication·convention violation 예측
2. Subagent isolation → parallel agent가 독립적으로 동일 solution 재구현 예측
3. "good local decision → poor global outcome when bounded context prevents global awareness"

보조 증거(인접 시스템):
- He et al. 2025: Cursor 채택 807 repo causal analysis — 복잡도 +40.7% (p < 0.001), velocity 첫 달 +281% → 3개월째 baseline, 복잡도 상승이 future velocity 감소와 연결. "self-cancelling".
- Liu et al. 2026: 304,000 AI-authored commit / 6,275 repo audit — AI 유발 이슈 약 1/4 latest revision까지 잔존, security-related는 substantially higher.

Claude Code 고유 대응:
- Graduated compression로 최신·관련 context 보존
- Cache-aware compaction으로 prompt cache invalidation 회피
- Read-time projection으로 full history 재구성 가능
- Subagent summary isolation

→ **"이 메커니즘들이 bounded context의 structural limitation을 극복하기에 충분한지는 source-level analysis로 해결 못 함"** (§11.4 말미의 epistemic 솔직함).

**6 Open directions (§12) — 실행가능성 평가:**

| # | 제목 | 저자가 낸 구체 단서 | 실행가능성 (독자 해석) |
|---|------|-------------------|---------------------|
| 12.1 | Silent failure / observability-evaluation gap | "hook 이벤트 27개에 generator-evaluator separation을 추가할지?" — 하네스 내부 vs 외부 | 중간. Hook 확장은 현실적, 평가 인터페이스 표준화는 어려움 |
| 12.2 | Cross-session persistence (memory + colleague relationship) | Hu 2025, Chhikara 2025(Mem0), Xu 2025, Packer 2023(MemGPT), Shinn 2023(Reflexion) 인용 | 낮음-중간. 파일 기반 transparency 유지하면서 promotion 자동화 어렵다 |
| 12.3 | Harness boundary evolution (where/when/what/with whom) | 4축 분해 — Managed Agents(Martin 2026), KAIROS(where/when), VLA(what), multi-agent debate(with whom) | 낮음. 4축 동시 확장은 fragmented stack 위험 |
| 12.4 | Horizon scaling (session → multi-session → scientific program) | Lu 2024(autonomous research), Gottweis 2025, Novikov 2025, Kwa(METR) | 매우 낮음. 주간 scale 작업에 cross-session memory + 새 coordination primitive 모두 필요 |
| 12.5 | Governance and oversight | EU AI Act (2026-08), MIT AI Agent Index, International AI Safety Report, Bartz v. Anthropic | 중간. 외부 감사 인터페이스는 현 아키텍처가 준비된 부분(session transcript) + 미비한 부분(외부 감사 포맷) |
| 12.6 | Evaluative lens revisited — long-term capability preservation | Becker 2025, Kosmyna 2025(EEG), Aiersilan 2026, Barke 2023, Perry 2023 | 매우 낮음. session-granularity comprehension signal 자체가 없음; 저자조차 "whether the harness documented here is even the right locus for that action (as opposed to the IDE, the organisation, or the human development loop)" 질문 |

**패턴:** 개방 문제들은 "하네스 내부에서 해결 가능"(12.1, 12.5 부분)과 "하네스를 넘어서는 문제"(12.2, 12.3, 12.4, 12.6)로 나뉜다. 저자는 **후자를 더 많이 잡았다** — 즉 이 논문의 수사적 포지션은 "Claude Code 아키텍처는 잘 설계되었지만 **미래 문제의 locus는 하네스 layer보다 넓다**"는 것.

---

## 7. 한계 (저자 명시 + 독자 관찰)

### 7.1 저자가 명시한 한계 (§11.5, §B.3)

1. **Static snapshot** — v2.1.88 한 버전만 분석.
2. **Feature flag build-time variability** — 다른 빌드는 기능적으로 다른 앱. `TRANSCRIPT_CLASSIFIER`가 false면 auto-mode classifier 통째로 제거됨. 일부 feature-gated 모듈은 bun:bundle tree-shaking 제약으로 `require()` 동적 사용.
3. **Memoized context assembly의 staleness** — `getSystemContext()`와 `getUserContext()`가 lodash memoize 사용 → git status, CLAUDE.md가 매 턴 recompute 되지 않음. 대화 중 동적 변경이 즉시 반영되지 않을 수 있음 (compaction이 cache clear + lazy path-scoped rule이 부분 보완).
4. **Reverse-engineering epistemology** — 소스는 구조를 밝히나 design intent, enabled production flag, runtime prevalence, unshipped behavior 확인 불가.
5. **Single-system generalization bound** — Claude Code만 보고 전체 coding agent 설계 공간을 일반화할 수 없음.
6. **OpenClaw snapshot** — 그 시점의 capability 대표 못 할 수 있음.

### 7.2 독자 관찰 (암묵적 가정 + 논리적 약점)

**A1. "1.6% decision logic vs 98.4% infrastructure" 출처 불명확.**
- 저자는 §3.1에서 "Community analysis of the extracted source estimates"라고만 언급. 구체 출처 링크·날짜 없음.
- 이 수치는 §11.1("ratio is not accidental"), §11.7("quantitatively")에서 **중심 주장의 주춧돌**로 반복 사용.
- 정의 불명확: "AI decision logic"과 "operational infrastructure"의 경계를 누가 어떻게 그었는지 명시 없음. Prompt construction은 어느 쪽인가? Tool schema 정의는?

**A2. Shen & Tamkin (2026) 자가 인용 의혹 — 철회.**
- ~~앞서 자가 인용으로 판정했으나, context-scout의 후속 검증 결과 **오판으로 확인**.~~
- 정정: Shen & Tamkin 2026의 "Shen"은 **Judy Hanwen Shen (Anthropic)**으로, 본 논문 corresponding author **Zhiqiang Shen (MBZUAI VILA-Lab)**과 **동명이인**이다. 두 사람은 성만 공유할 뿐 별개 연구자.
- 즉 §2.4 evaluative lens의 근거는 **실제로 독립적 3자 증거**이며, 자가 인용이라는 비판은 성립하지 않는다. 저자의 인용 관행에는 이 지점에서 문제 없음.
- **교훈(메타):** 성만 보고 저자 동일성을 추정하는 것은 위험하다. 향후 관찰 기록 시 fullname 대조 필수. — **이 정정 자체가 insight-analyst와 통합 보고 독자에게 "외부 검증의 가치"를 보여주는 사례**로 기능할 수 있다.

**A3. OpenClaw 비교의 공정성 — 여전히 유효한 "도메인 불일치" 의혹.**
- context-scout 확인: OpenClaw는 실존 프로젝트. **Peter Steinberger(오스트리아)가 개발, Clawdbot(2025-11) → Moltbot(2026-01-27) → OpenClaw로 진화**. 3개월만에 100K stars 돌파, 2026-02-15 Steinberger는 OpenAI 합류. 개인 AI 어시스턴트로 **WhatsApp bot에서 출발**했다.
- 논문 표현 "multi-channel personal assistant gateway"는 OpenClaw의 정체성과 **표면적으로 일치** — 즉 저자가 시스템의 성격을 잘못 묘사한 것은 아니다.
- 그러나 공정성 의혹은 **여전히 유효**: Claude Code(터미널 코딩 에이전트)와 OpenClaw(메시징 퍼스널 어시스턴트)는 **근본적으로 다른 사용자·입력·태스크 도메인**. 6차원에서 "같은 질문에 다른 답"이라는 프레이밍의 힘은 "같은 질문"이 성립할 때 나오는데, 이 두 시스템이 **정말로 같은 설계 질문에 직면했다고 볼 수 있는가?**
- 저자 본인도 §10.2에서 "the questions are stable; **the answers vary with deployment context**"라고 인정한다. 그러면 6개 차이는 **deployment context가 다르기 때문에 생긴 자연스러운 차이**이지 설계 철학의 "opposing bets"가 아닐 수 있다.
- OpenHands / SWE-Agent / Aider / Codex CLI 등 **실제 코딩 에이전트 계열**이 존재하는데 이들과의 직접 비교는 §13(Related Work)에만 짧게 놓인다. 비교의 주 무대로 OpenClaw를 선택함으로써 "극적 대립"은 쉬워지나 **같은 도메인 내 경쟁 시스템과의 더 어려운 직접 비교는 회피**된다.
- **보조 관찰:** Steinberger가 2026-02 OpenAI에 합류한 타이밍과 본 논문(2026-04 제출)의 OpenClaw 선정 사이 관계는 본문에서 논의되지 않는다. 비교 대상의 인지도·화제성이 OpenClaw의 100K stars 급성장과 상관있을 가능성.

**A4. "Model judgment within deterministic harness" 의 범용성 과잉 일반화.**
- "1.6% vs 98.4%" 비율이 아키텍처 우월성을 말한다면, 더 간단한 모델·더 복잡한 하네스 조합이 더 큰 비율로 outperform 한다는 반례(e.g., LangGraph-based 시스템)를 **반박하지 않는다**. 저자는 Devin·LangGraph를 대안으로 언급만 할 뿐, 경험적 비교는 없음.
- "trusts the model to make good local decisions, but good local decisions can produce poor global outcomes" (§11.7) — 이 한계는 저자 본인이 인정하지만, 그러면 "harness가 많으면 좋다"는 1.6/98.4 비율 해석이 스스로 weak해진다.

**A5. Source extraction의 투명성 부족 — "재현성 구조적 부재 + 저장소 메타데이터 미완성" (2차 보강).**

**(a) 누출 경위 vs 논문 disclaimer 괴리 (기존 근거, 유지):**
- 이 "공개 TypeScript 소스"는 **Anthropic이 2026-03-31 경 npm 패키지에 .map (source map) 파일을 실수로 포함하면서 소스 전체가 사실상 누출된 사건**의 결과물. Claude Code는 **의도적으로 오픈소스화된 적이 없다**.
- 본 논문 제출일 2026-04-14는 **누출 2주 후**. 타이밍 일치.
- 저자 disclaimer: "All materials used in this work are obtained from publicly available online sources. We have not used any private, confidential, or unauthorized materials" — **.map을 통해 간접 공개된 소스가 "publicly available, authorized"에 해당하는지 법적·윤리적 회색지대**. Anthropic 의도 여부는 본문에 명시 안 됨.

**(b) context-scout의 저장소 직접 조사로 추가 확증된 "5중 투명성 부재":**

VILA-Lab/Dive-into-Claude-Code 저장소 실제 구성:
```
.github/workflows/ (CI)
assets/            (다이어그램 6종)
docs/
  ├─ architecture.md         ← 개념 서술만. 파일경로·라인·코드 스니펫 부재
  ├─ build-your-own-agent.md
  └─ related-resources.md    ← 50+ 커뮤니티 링크 큐레이션
paper/Dive_into_Claude_Code.pdf
CITATION.cff       ← authors: "To be updated" (플레이스홀더)
LICENSE (CC BY-NC-SA 4.0)
README.md
```

부재한 재현 아티팩트:
1. `src/`, `data/`, `scripts/`, `analysis/` 폴더 없음 — 추출·분석 파이프라인 0
2. Methodology / Data Sources / How We Obtained the Source 섹션 없음 (README·architecture.md 모두)
3. "1.6% / 98.4%" 측정 방법·스크립트·원 데이터 미공개 — 논문 핵심 주장 재현 불가
4. `queryLoop` in `query.ts` 같은 컴포넌트 참조에 **실제 파일 경로·라인 포인터 없음** — 개념 서술만 있고 TypeScript 스니펫 부재
5. CITATION.cff가 논문 제출 1주 경과 후에도 **"To be updated" 미완성**

**(c) 아이러니:** `related-resources.md`가 큐레이션한 타 리버스 엔지니어링 프로젝트들(ComeOnOliver 등)은 오히려 **소스 트리 구체 참조를 제공**해 더 투명하다. 즉 저자들은 커뮤니티의 좋은 관행을 알고 있으면서 본인 저장소에선 그 수준을 갖추지 않았다.

**(d) 종합 판정:**
- 사기·날조로 볼 근거 **없음** (수치 다수가 타 분석과 일치, 인용 9/10 실존, CVE 실존).
- 그러나 "source code as evidence" 학술 규범에는 **미달**.
- 논문의 **Tier B 주장(code-verified)은 실질적으로 Tier C(reconstructed, 검증 없음)에 가깝게 운영된다** — 저자가 파일·라인을 인용하더라도 독자가 그것을 원 소스에서 검증할 경로가 저장소에 없다.
- 논문의 Tier 구분(§B.1) 자체가 수사적 장치일 뿐, **재현 가능한 Tier B와 단순 주장 Tier B를 구분하는 기준**은 없다. 이는 방법론 챕터의 핵심 신뢰 기반인 Tier A/B/C 위계 자체에 균열을 낸다.

**A6. Memoized cache의 dynamic staleness 한계 — 저자 명시보다 더 심각할 수 있음.**
- `CLAUDE.md` 내용이 대화 중 편집되어도 memoization 때문에 즉시 반영 안 된다는 점은 **User 권한**에도 영향. 사용자는 자신이 수정한 규칙이 모델에게 전달됐다고 가정할 수 있으나 실제로는 아닐 수 있다.
- 보안 관련 instruction을 CLAUDE.md에 쓰면 안 된다는 점(§7.2)은 저자도 명시했지만, "보안 외 다른 중요 규칙은 OK"라는 해석은 위험.

**A7. 13 원칙의 경계 — 원칙 간 중복·충돌 가능성.**
- Table 1에서 각 원칙이 여러 value를 동시에 섬긴다 ("Principles map to multiple values"). 좋다. 그러나 **원칙 간 충돌**은 다루지 않음.
- 예: "Values over rules" (원칙) vs "Deny-first defaults" (원칙) — "values"가 "rules"보다 우선인데 deny-first는 rule의 한 형태. §5에서 deny rule이 allow를 이긴다는 원칙과 충돌.
- 저자는 Value tension은 §11.2에서 다루지만 **원칙 tension**은 다루지 않음.

**A8. KAIROS 언급의 Tier 불일치 — 부분 정정.**
- context-scout 확인: KAIROS는 유출된 소스에 **150회 이상 참조**되어 **실재 확인**. autoDream 서브시스템, GitHub webhook, 5분 cron 등 구체 구조가 코드에 존재. 외부 빌드에서는 `false`로 컴파일되어 논문의 "cannot be confirmed as active in production builds" 단서는 정확.
- 즉 **KAIROS 자체는 Tier B 증거**로 읽을 수 있음 (소스 코드 검증 가능). 내 앞선 "존재하지 않을 수도 있는 시스템" 우려는 과도했다 — 정정.
- 다만 **수사적 쟁점은 남음**: production에서 활성화된 적이 없는 기능을 "미래 방향의 구체적 illustration"으로 §11.6·§12.3에서 반복 인용하는 것은, 디폴트로 꺼진 feature flag 기반 코드를 "Claude Code가 이미 이 방향으로 가고 있다"는 증거처럼 제시한 것. 저자는 단서를 달았으나 독자가 flag 상태를 놓치면 오독 가능.
- **Tier 분류 관점:** KAIROS 자체의 구조 서술은 Tier B이나, "이 방향은 실현 가능하다"는 아키텍처적 주장은 Tier C에 가깝다. 둘이 한 문단에 섞여 있음.
- **병렬 수렴 첨언(Niki 사례, 신뢰도 중):** X 사용자 @niki2ai가 Claude Code 소스 누출 후 "**same name, same concept**"의 always-on 백그라운드 에이전트를 **독립적으로 구현**했다고 공개 포스팅(URL: https://x.com/niki2ai/status/2038955636298592626, WebFetch 402로 본문 직접 검증 실패). 의미 두 가지: (i) KAIROS 아이디어 자체는 **희소하지 않으며** 외부 커뮤니티에서도 같은 방향 탐색 중 → 저자가 제시한 "미래 방향"은 Claude Code 고유 혁신이 아니라 업계 병렬 아이디어. (ii) Claude Code가 KAIROS를 내부에 가진 채 flag off로 잠재우는 것은 "**흔한 아이디어를 feature flag로 지연**"하는 전략적 선택 — 기술적 한계가 아닌 제품·정책 선택일 가능성. 이는 §11.6의 "emerging directions" 프레임이 과도하게 Anthropic 내부 관점을 반영한다는 비판 접근선이 된다.

**A9. 6 개방 문제의 크기 불균형.**
- §12.1(observability-eval gap)과 §12.5(governance)는 harness 내부 문제로 상대적 구체적.
- §12.3(harness boundary evolution)과 §12.6(long-term capability)은 **학문 분야 하나 이상**의 영역 (VLA 전체, HCI 전체, 정책 전체). "open direction"이라는 레이블은 균등 중요도를 암시하지만 실제로 크기가 매우 다르다.
- 저자는 §12 서두에서 "six span the paper's five-value framework and evaluative lens"라고 framing을 정당화하지만, 이는 구조적 대칭성 추구이지 해결 가능성 이슈가 아니다.

**A10. 외부 수치의 "간접 전용(generalization by proxy)" 문제 — 신규, context-scout 검증 기반.**
- 논문은 Claude Code 아키텍처를 비판하기 위해 **다른 시스템 대상 경험 연구**를 자주 끌어온다. context-scout 검증으로 드러난 구체 문제:
  - **40.7% 복잡도 증가 (He et al. 2025, arXiv:2511.04427 CMU)**: 대상은 **Cursor 채택 807 repository**. Claude Code가 아닌 Cursor에 대한 인과 분석. 저자는 §11.4에서 "architecturally similar tools"라고 hedge하지만, §11.2 Table 4 value tension 근거로는 그 조건을 완화해 쓴다.
  - **19% 느려짐 (Becker et al. 2025, METR arXiv:2507.09089)**: **n=16 RCT, 2025년 초**. Claude Code 정식 릴리스 **이전** 시기 AI 도구 대상. 표본 크기도 작다. 저자는 §11.2 Table 4에서 이 수치를 "long-term sustainability" 근거로 쓰는데, 시기·표본을 생각하면 **generalization by proxy**.
  - **25% 주니어 채용 감소 (Rak 2025)**: context-scout가 **직접 출처 미확인**. 가장 가까운 자료는 Stanford Brynjolfsson 2025-08 "Canaries"의 **22-25세 약 20% 감소**(25% 아님). 논문 각주의 정확한 "Rak 2025" 확인 필요.
- 이 수치들 자체가 "거짓"인 것은 아니지만, **그들을 Claude Code 설계에 귀속시키는 추론의 단계별 hedge가 본문에서 일관되지 않다**. §11.4는 "adjacent system"을 인정하는데 §11.2 Table 4는 그런 hedge 없이 "Capability × Reliability" 같은 Claude Code의 value tension의 evidence로 바로 제시.

**A11. "93% 승인율" 인용의 출처 맥락 누락.**
- context-scout 확인: 93% 수치의 1차 출처는 Anthropic 엔지니어링 블로그 "Claude Code auto mode: a safer way to skip permissions" (2026-03-25). **서베이가 아니라 10K real traffic + 1K synthetic exfiltration 에 대한 데이터셋 평가**. 함께 보고된 수치는 0.4% FPR, 17% FNR (on 52 real overeager actions).
- 논문은 이 수치를 "approval fatigue" 주장의 행동적 근거로 쓰는데, **원출처는 auto-mode classifier의 평가 지표 블로그포스트**였지 "사람의 승인 습관" 자체를 주제로 한 사용자 연구가 아니다.
- 즉 "사용자가 93% 승인한다 → 사람은 권한 프롬프트를 신뢰할 수 없다"의 **논리적 도약**이 존재. 저자는 Hughes 2026의 데이터를 그 프레임으로 재해석한 것이지, 해당 블로그포스트가 그 결론을 직접 주장한 것은 아닐 수 있다.
- Huang et al. 2025 ("132명 엔지니어·연구자 서베이")와 Hughes 2026 (데이터셋 평가)은 **독립된 두 출처**지만 논문은 두 곳을 수시로 엮어 "Anthropic 내부 데이터"로 제시한다. 엄밀하게는 방법론도 대상도 다른 연구다.

**A12. "Long-term capability preservation" 를 6번째 value가 아닌 "evaluative lens"로 **격하시킨** 이유.**
- 저자 본인: "it is not prominently reflected as a design driver in the architecture or in Anthropic's stated values" (§2.4) → Anthropic이 그렇게 안 했으니 평가 렌즈로만 쓴다.
- 그러나 §2.1의 다른 5 value도 "Anthropic이 설계 driver로 보고 있는가"를 직접 검증하지 않은 상태에서 공식 문서 인용으로 정당화되었다. 즉 **6번째 value를 "격하"하는 근거가 다른 5개를 "채택"하는 근거와 비대칭**.
- 이 asymmetry는 저자의 수사적 전략으로 볼 수 있다 — 비판을 "기본 축 바깥"으로 두어 Anthropic과의 연속성을 유지하면서도 비판적 거리를 확보.

---

## 8. 불명확 지점 (확신 없는 부분)

**U1. "7×" 토큰 소비 (§8.3) 의 정확한 정의·출처.**
"Claude Code's agent teams consume approximately 7× the tokens of a standard session in plan mode (Anthropic, 2025b)" — Anthropic 2025b는 참고문헌에 있는지 전체 검토 필요. 7×가 무엇의 몇 배인지(per-task? per-session? total?)도 불명.

**U2. `bubble` 모드의 실제 behavior.**
Type union에만 있고 mode array에 없으며 "internal-only for subagent permission escalation"으로만 기술. "bubble"이라는 이름의 의미(terminal로 escalate 버블 올라간다?), 실제 dialog가 parent에 어떻게 나타나는지, 언제 발동되는지 — 소스 수준 추적 없이는 불명확. 저자도 이를 의도적으로 얕게 다뤘을 가능성.

**U3. `yoloClassifier` 의 모델·학습 데이터.**
ML 기반이라 했지만 어떤 모델(같은 Claude? 별도 소형?), 어떤 데이터로 학습, false positive/negative rate 등은 본문에 없음.

**U4. "fork-subagent path"의 정확한 의미.**
§8.1에서 "type을 생략하면 fork-subagent path로 route될 수 있다"고만 언급. 이 경로가 parent context를 **상속한다**는 것 외 상세 미기술.

**U5. `CACHED_MICROCOMPACT` gated cache-aware path의 구체적 cost-benefit.**
소스 주석 인용으로 "false path is 98% cache miss, costs ~0.76% of fleet cache_creation"은 제시되나, 이게 cached path와 non-cached path 중 어느 쪽인지, 2026년 1월 실험 이후 결론은 어디서 확인 가능한지 불명.

**U6. OpenClaw의 "pi-agent core"와 "ACP (Agent Client Protocol)" 의 정확한 스펙.**
§10 비교에서 여러 번 등장하지만 ACP가 공식 프로토콜인지(MCP처럼 표준인지), pi-agent가 무엇의 약자인지 — 본문만으로 판단 불가. context-scout 조사 대상.

**U7. "evaluative lens"가 후속 섹션에서 정확히 어떻게 작동하는가.**
§2.4는 "cross-cutting concern applied across all five values in §11"이라고 함. §11.2의 table은 5 value pair × 2 추가 tension을 보여주는데, **"applied across all five values"**가 구체적으로 어떤 인식론적 작업인지 — §11 전체를 꼼꼼히 봐도 "모든 5 value에 렌즈가 씌워졌다"보다는 "일부 value tension에서 long-term capability가 언급된다"에 가까움. **저자의 메타 주장과 본문의 실제 분석 사이에 gap 존재 가능**.

**U8. Sidechain `.meta.json` 의 스키마·필드.**
Subagent 격리 아키텍처의 핵심 기술 아티팩트인데 본문에 상세 없음. 재현하려면 실제 소스·출력 샘플 확인 필요.

**U9. §10의 "agent teams ~7×" 수치와 §8의 file locking 기반 team coordination 설명 간의 경제성 연결.**
File locking으로 throughput를 희생하고 zero-dependency를 얻는다 했는데, 그것이 7× 토큰 비용과 별도 축인지 동일 축인지 — throughput(time) 비용과 token(money) 비용의 관계가 명시적이지 않음.

**U10. 논문의 "Section A(Package Structure)"와 "Section B(Evidence Methodology)" 순서.**
"Sections 13 and 14 cover related work and conclusions. Section B describes the evidence base and methodology" (§1 paper organization). 즉 방법론을 **부록**에 둔다. 이는 논문 형식상 이례적 (통상 §2-3에 methodology). 이 배치가 주장의 Tier A/B/C 혼재를 **제한적으로 가시화**했을 가능성. 독자가 본문만 읽고 "이건 Tier B인가 C인가" 구분 어려움.

---

## 부록: 핵심 Figure·Table의 역할

| 요소 | 역할 | 위치 |
|------|------|------|
| Figure 1 | 7-컴포넌트 고수준 구조 (spine 데이터 흐름) | §3.2 |
| Figure 2 | Turn flow — user prompt → iter N → assistant response | §3.1 |
| Figure 3 | 5-layer 서브시스템 분해 | §3.3 |
| Figure 4 | Permission gate overview + principle (Table 형태 병기) | §5 |
| Figure 5 | Pseudocode (agent loop 3 injection point) + table (assemble/model/execute) | §6 |
| Figure 6 | Context window 6 source + mutability·access gradient | §7.1 |
| Figure 7 | Subagent isolation 3축 (routing/isolation/lifecycle) + pipeline | §8 |
| Figure 8 | Session persistence — live state vs durable storage | §9 |
| Figure 9 | Package structure ↔ runtime responsibility mapping (부록 A) | A.1 |
| Table 1 | **13 원칙 × value × 설계 질문 × 섹션** — 아키텍처의 인덱스 | §2.2 |
| Table 2 | 4 확장 메커니즘 × 고유 capability × context cost × insertion point | §6.3 |
| Table 3 | Claude Code ↔ OpenClaw 6차원 비교 | §10.1 |
| Table 4 | 5 value tension + evidence | §11.2 |
| Table 5 | AI coding tool 4 범주 | §13.1 |
| Table 6 | Context management 5 접근법 (taxonomy) | §13.2 |
| Table 7 | Key source file × size × responsibility | A.1 |
| Table 8 | Conditional tool availability category | A.2 |

**저자의 구성 전략:** 섹션 초반에 **figure 로 지도 제시** → 텍스트로 소스 파일 수준까지 줌인 → 대안 설계로 줌아웃. Running example("auth.test.ts 수정")이 §3부터 §9까지 일관되게 참조되어 9장 교과서 스타일을 유지.

---

## 메모: context-scout/insight-analyst 을 위한 포인터

- **context-scout에게 이미 전달한 외부 조사 항목**: 27%, 93%, 20→40%, 17%, 1.6/98.4, CVE 4건, 84%, 40.7%, 19%, 25%, OpenClaw, KAIROS, Managed Agents, VILA-Lab/Dive-into-Claude-Code 저장소 실재성, Shen&Tamkin 순환 인용 가능성.
- **context-scout가 확인해준 핵심 사실 (2차 정정 포함)**:
  - **A2 정정**: Shen & Tamkin 2026의 Shen = **Judy Hanwen Shen (Anthropic)**으로 공저자 Zhiqiang Shen과 **동명이인**. **자가 인용 아님**. 내 앞선 판단은 오판으로 철회.
  - **A3 유지**: OpenClaw는 실존 (Peter Steinberger, 2025-11 시작, 3개월만에 100K stars, 2026-02 OpenAI 합류). WhatsApp bot 기반 개인 AI 어시스턴트로 **코딩 에이전트 아님** — 도메인 불일치 의혹은 유효.
  - **A8 부분 정정**: KAIROS는 유출 소스에서 **150+회 참조 확인**. 단 외부 빌드에서 false로 컴파일되므로 "production에서 실활성화 안 됨" 단서는 정확. Tier 분류만 정정.
  - **A11 유지**: 93% 수치는 Anthropic 엔지니어링 블로그(2026-03-25, Hughes)의 데이터셋 평가(10K real + 1K synthetic + 52 real overeager actions). Huang 2025의 132명 서베이(Saffron Huang 주도, Anthropic 2025-09)와는 별도.
  - **A5 강화**: (i) npm .map 누출(2026-03-31, 논문 제출 2주 전) (ii) VILA-Lab 저장소 실조사 — `src/scripts/data/analysis` 폴더 0, Methodology 섹션 0, 1.6%/98.4% 측정 스크립트 0, 파일경로·라인 인용 0, CITATION.cff "To be updated" 미완성. **Tier B 주장이 실질적으로 재현 불가**.
  - **A10 신규**: He et al. 2025(Cursor arXiv:2511.04427)·Becker et al. 2025(METR n=16 arXiv:2507.09089)·Rak 2025 등 외부 수치 인용은 Claude Code가 아닌 **인접 시스템 대상**이거나(generalization by proxy), 출처 확인 불완전(Rak).
  - **1.6/98.4**: 출처 "community analysis"는 익명이되 수치 재현 가능.
  - 6개 CVE 모두 NVD 실존 확인.
  - **저자 소속 전원 MBZUAI 확정** (Zhiqiang Shen Group 공식 페이지 직접 조회): Jiacheng Liu, Xiaohan Zhao, Xinyi Shang, Zhiqiang Shen 네 명 모두 Shen 그룹 소속. 커뮤니티에서 회자되는 "UCL researchers" 소개는 외부 소개자 오류 → 저자 귀속 자체는 정확, **"소속 부정확 표기" 비판 접근선은 폐기**.
  - **KAIROS 병렬 수렴 사례(Niki, @niki2ai)**: X 포스트 (WebFetch 402로 직접 미확인). 동일 이름·동일 개념의 always-on 백그라운드 에이전트 독립 구현 보고. A8에 반영.
  - **수집 한계**: X/Twitter 본문은 WebFetch 402 Payment Required로 직접 파싱 불가. 검색 결과 인용문 수준까지만 가능.
- **insight-analyst의 비판 잠재지(독자 관찰 기반)**:
  - **유효 비판**: A1(1.6/98.4 출처), A3(OpenClaw 도메인 불일치), A4(harness-heavy 주장 반례 부재), A5(소스 누출 윤리), A6(memoization staleness), A7(원칙 충돌 미논의), A9(개방 문제 크기 비대칭), A10(외부 수치 proxy 전용), A11(93% 프레이밍 도약), A12(evaluative lens 수사 격하)
  - **철회된 비판**: A2(자가 인용 오판 — 동명이인 확인)
  - **부분 유지**: A8(KAIROS 존재는 확인, 수사적 Tier mix만 쟁점)
- **실무자(하네스 엔지니어)에게 직접 가치 있는 발견**:
  - 5 shaper의 lazy degradation 순서 + 각 층의 failure mode (특히 snip tokensFreed의 visibility 문제)
  - 7 permission layer의 AND-분리(temporal도 고려해야 함 — pre-trust window)
  - 4 확장 메커니즘 선택 기준은 **context cost × insertion point** 2축
  - Subagent는 summary-only return + sidechain transcript 패턴이 비용 억제와 auditability를 동시 달성
  - CLAUDE.md guidance vs permission enforcement의 분리 — 규칙을 어디에 둘지의 실무 기준
  - Session resume의 permission 비복원 원칙 — "trust invariant > user convenience"
