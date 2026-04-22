---
name: research-orchestrator
description: "논문 심층 리서치 오케스트레이터. 논문(arXiv URL/PDF/DOI/제목)을 주거나 '이 논문 분석해줘', '이 분야 서베이', '~논문 깊이 읽어줘', '~에 대한 인사이트' 등 논문 기반 리서치를 요청하면 반드시 이 스킬을 사용할 것. 단일 논문 정독, 다중 논문 비교, 주제 서베이 모두 지원. 후속 작업(재실행, 부분 업데이트, 이전 결과 보완, 특정 섹션 다시, 결과 개선)도 이 스킬로 처리한다. 단순 질의응답이나 코드 작성은 트리거하지 말 것."
---

# Research Orchestrator — 논문 심층 리서치 통합 스킬

논문 기반 리서치를 위한 **에이전트 팀**을 조율하여 **상세 설명 + 비판적 인사이트**가 담긴 보고서를 생성한다.

## 실행 모드: 에이전트 팀

`TeamCreate`로 팀을 구성하고 공유 작업 목록과 `SendMessage`로 조율한다. 이 모드를 선택한 두 가지 핵심 이유:

1. **교차 검증** — `deep-reader`가 본문에서 본 주장과 `context-scout`가 커뮤니티에서 본 반응이 실시간으로 교차된다. 한쪽이 중요한 발견을 하면 즉시 다른 쪽에 SendMessage로 전달되어 조사 방향이 조정된다. 사후 파일 대조로는 잡을 수 없는 상호 보정이 가능.
2. **도구 권한 실시간 피드백 루프** — 팀원이 도구 권한 차단(예: `WebFetch(domain:X)` 미허용)을 감지하면 리더에게 즉시 `SendMessage`로 보고한다. 리더는 사용자에게 실시간 통보하여 `settings.local.json`을 업데이트한 뒤 재시도를 지시한다. 서브 에이전트 모드에서는 49회 연속 실패해도 최종 보고서에서나 인지되는 문제를 구조적으로 차단.

**세션당 팀 1개 제약이 있으므로** 다중 논문 모드는 논문별로 팀을 순차 구성한다(`TeamCreate` → 실행 → `TeamDelete` → 다음 논문). 중간 산출물은 파일로 남아 다음 팀이 접근 가능.

**팀원 정체성 주의 (2604.14572 실전 검증 경험):** 팀원은 공식 문서상 "full, independent Claude Code session"이며 subagent의 하위 계층이 아니다. 구현 레이어에서 `Agent` 도구가 `subagent_type` 파라미터를 공유하는 것은 역할 정의 재사용의 편의이지, 팀원을 subagent로 대하라는 의미가 아니다. 팀원 spawn 프롬프트에 "너는 subagent다" 뉘앙스를 넣지 말 것. 각 팀원 정의 파일(`.claude/agents/*.md`)에 "실행 컨텍스트와 정체성" 섹션이 포함되어 있어 spawn 시 자기 정체성을 자동으로 선언하도록 되어 있다. 2604.14572 실전 검증에서 insight-analyst가 자기 자신을 subagent로 오인해 Write 도구를 스스로 억제해 team-lead 개입이 필요했던 사례가 있다.

## 작업 공간 구조

논문별로 격리된 서브디렉토리를 사용한다. 이전 논문 산출물은 **절대 삭제/이동하지 않고 누적**한다.

```
/Users/user/harness/research/_workspace/
├── {arxiv_id}_{slug}/                 ← 논문 1편 = 폴더 1개
│   ├── 00_plan.md
│   ├── 02_deep_reader.md
│   ├── 03_context_scout.md
│   └── report.md                      ← 최종 보고서
├── {다른_논문}/...
└── comparison_{YYYYMMDD}_{slug}/      ← 다중 논문 비교 시에만
    ├── 00_plan.md
    ├── 01_candidates.md               ← 서베이 모드일 때 paper-hunter 산출물
    └── report.md
```

**폴더명 규칙 `{arxiv_id}_{slug}`:**
- `arxiv_id`: arXiv 번호 그대로 (예: `2604.14228`). arXiv가 아니면 DOI의 마지막 토큰이나 해시화된 식별자
- `slug`: 논문 제목에서 의미 있는 키워드 2~3개를 소문자 하이픈으로 연결 (예: `claude-code`, `chain-of-thought`, `rag-survey`)
- 예시: `2604.14228_claude-code`, `2310.11511_self-rag`

## 에이전트 구성

| 팀원/에이전트 | subagent_type | 역할 | 실행 모드 | 출력 |
|-------------|--------------|------|----------|------|
| research-lead | general-purpose | 팀 리더 (= 오케스트레이터 본인) | 팀 리더 | 계획 수립, 진행 모니터링, 최종 사용자 보고 |
| paper-hunter | general-purpose | 주제→논문 탐색 | **서브 에이전트** (Phase 1.5, 서베이 모드 전용) | `comparison_*/01_candidates.md` |
| deep-reader | general-purpose | 논문 정독 | **팀원** | `{paper_dir}/02_deep_reader.md` |
| context-scout | general-purpose | 외부 맥락 수집 | **팀원** | `{paper_dir}/03_context_scout.md` |
| insight-analyst | general-purpose | 비판적 통합·보고서 | **팀원** (같은 팀에서 실시간 질의 가능) | `{paper_dir}/report.md` 또는 `comparison_*/report.md` |

모든 호출에 `model: "opus"` 명시. 에이전트 파일은 `.claude/agents/{name}.md` 참조.

**하이브리드 근거:** paper-hunter는 단발 탐색 작업이라 팀 통신이 불필요 → Phase 1.5에 서브 에이전트로 단독 호출. **수집 Phase(deep-reader + context-scout)**와 **통합 Phase(insight-analyst)**는 팀 모드를 유지 — 이 두 Phase의 교차 검증이 본 스킬의 핵심 가치.

## 워크플로우

### Phase 0: 컨텍스트 확인 (후속 작업 지원)

스킬 진입 시 가장 먼저 수행한다.

1. 사용자 입력에서 논문 식별자를 추출 (URL, arXiv ID, 제목 등)
2. 논문 식별자가 있으면 해당 `_workspace/{paper_dir}/` 존재 여부 확인
3. 실행 모드 결정:

| 상황 | 모드 | 동작 |
|------|------|------|
| 논문 폴더 미존재 | **초기 실행** | Phase 1로 — 새 폴더 생성 |
| 논문 폴더 존재 + 사용자가 부분 수정 요청 ("X 섹션 다시", "커뮤니티 반응 보완") | **부분 재실행** | 해당 팀원만 활성화되는 축소 팀 구성, 기존 산출물 재활용 |
| 논문 폴더 존재 + 전체 재생성 요청 ("처음부터 다시") | **강제 재실행** | 폴더를 `{paper_dir}_{YYYYMMDD_HHMMSS}/`로 rename 후 Phase 1 |
| 새 논문 요청 | **초기 실행** | 다른 논문 폴더는 건드리지 않음. 새 폴더만 생성 |

**부분 재실행 규칙:**

| 수정 요청 유형 | 재호출 대상 |
|-------------|----------|
| "이 논문 본문 다시 정리" | deep-reader + insight-analyst 2인 팀 구성 |
| "커뮤니티 반응 보완" | context-scout + insight-analyst 2인 팀 구성 |
| "보고서 톤/구조만 수정" | insight-analyst만 **서브 에이전트**로 단독 호출 (팀 통신 불필요) |

부분 재실행 시 팀원 초기 프롬프트에 **기존 산출물 경로**와 **사용자 피드백**을 함께 전달한다.

### Phase 1: 준비 (초기 실행)

1. 사용자 입력 분석:
   - 입력 유형: `single-paper` / `multi-paper` / `survey`
   - 단일/다중 논문 모드: 각 논문마다 `{arxiv_id}_{slug}` 결정
   - 서베이 모드: 주제 slug 결정 (예: `rag-recent-trends`) → `comparison_{YYYYMMDD}_{slug}/` 폴더 생성
2. 입력이 단순 단일 논문으로 명확하면 오케스트레이터가 직접 판별하고 `00_plan.md` 작성
3. 입력이 모호하거나 다중 논문/서베이이면 아래 Phase 1.5 또는 Phase 2로 진입
4. 작업 폴더 생성 (`mkdir -p`)

### Phase 1.5: 주제 서베이 분기 (서브 에이전트, 서베이 모드 전용)

`입력 유형: survey`일 때만 수행한다. 이 단계는 **팀 구성 전**에 진행하며 팀 통신이 불필요하다.

```
Agent(
  subagent_type: "general-purpose",
  name: "paper-hunter",
  model: "opus",
  prompt: "paper-hunter 역할로 주제 '{topic}'에 대한 후보 논문을 탐색하라.\n\n에이전트 정의: .claude/agents/paper-hunter.md 준수.\n출력: _workspace/comparison_{YYYYMMDD}_{slug}/01_candidates.md"
)
```

paper-hunter의 후보 목록이 나오면 Phase 2의 다중 논문 모드로 진입 (각 후보 논문별로 Phase 2~5 순차 반복).

### Phase 2: 팀 구성

논문 1편을 처리하는 팀을 생성한다. **다중 논문 모드에선 논문별로 Phase 2~5를 반복한다** (세션당 팀 1개 제약).

```
TeamCreate(
  team_name: "paper-{arxiv_id_short}",   // 예: paper-2604-14572 (dot은 dash로)
  agent_type: "research-lead",
  description: "논문 {제목} 심층 리서치",
  members: [
    {
      name: "deep-reader",
      agent_type: "general-purpose",
      model: "opus",
      prompt: "deep-reader 역할로 논문 {식별자}을 정독하라.\n논문 폴더: _workspace/{paper_dir}/\n에이전트 정의: .claude/agents/deep-reader.md 준수 (팀 통신 프로토콜 포함).\n출력: {paper_dir}/02_deep_reader.md"
    },
    {
      name: "context-scout",
      agent_type: "general-purpose",
      model: "opus",
      prompt: "context-scout 역할로 논문 {식별자}의 외부 맥락을 수집하라.\n논문 폴더: _workspace/{paper_dir}/\n에이전트 정의: .claude/agents/context-scout.md 준수 (팀 통신 프로토콜 포함).\n출력: {paper_dir}/03_context_scout.md\n\n**도구 권한 차단이 감지되면 리더(research-lead)에게 즉시 SendMessage로 보고할 것.**"
    },
    {
      name: "insight-analyst",
      agent_type: "general-purpose",
      model: "opus",
      prompt: "insight-analyst 역할로 deep-reader와 context-scout의 산출물을 통합하여 최종 보고서를 작성하라.\n논문 폴더: _workspace/{paper_dir}/\n에이전트 정의: .claude/agents/insight-analyst.md 준수 (팀 통신 프로토콜 포함).\n출력: {paper_dir}/report.md\n\n수집 팀원의 산출물 생성이 완료되기 전엔 대기. 통합 중 불명확한 지점은 해당 팀원에게 직접 SendMessage로 질의할 수 있다."
    }
  ]
)
```

이어 공유 작업 목록에 작업 등록:

```
TaskCreate(subject: "본문 정독 — {paper_dir}", description: "...", owner: "deep-reader")
TaskCreate(subject: "외부 맥락 수집 — {paper_dir}", description: "...", owner: "context-scout")
TaskCreate(subject: "통합 보고서 작성 — {paper_dir}", description: "...", owner: "insight-analyst")
   → TaskUpdate(addBlockedBy: ["본문 정독", "외부 맥락 수집"])
```

insight-analyst 작업은 다른 두 작업을 `blockedBy`로 걸어 자동으로 대기 상태가 되도록 한다.

### Phase 3: 수집 (팀원 자체 조율)

deep-reader와 context-scout가 독립적으로 작업한다. 리더는 개입하지 않고 모니터링만 한다.

**팀원 간 통신 규칙 (실행 중 실제 발생 예상):**

- **deep-reader → context-scout**: 본문에서 발견한 중요 외부 참조(예: OpenClaw 같은 시스템명)를 "이거 즉시 조사 부탁" SendMessage
- **context-scout → deep-reader**: 커뮤니티에서 논문 주장에 대한 강한 비판 발견 → "§X 섹션 재확인 요청" SendMessage
- **context-scout → insight-analyst**: "논문 주장과 커뮤니티 반응 상충 발견" 조기 경보 (insight-analyst는 아직 대기 중이지만 메시지는 큐에 쌓임)
- **모든 팀원 → research-lead**: 도구 권한 차단 감지 시 **즉시 보고** (가장 중요한 실시간 피드백 경로)

**리더(= 오케스트레이터) 모니터링:**

- 팀원이 유휴 상태가 되면 자동 알림 수신 → 정상 완료인지 중단인지 판별
- 권한 차단 보고 수신 시 즉시 사용자에게 전달:
  - "팀원 `{name}`이 `{도구}` 사용 중 권한 차단. 현재 막힌 도메인: `{X}`. `.claude/settings.local.json`에 다음 권한 추가 필요: `\"WebFetch(domain:X)\"`. 수정 후 '재시도'라고 답해주세요."
- 사용자가 권한 업데이트 후 재시도 지시 시 해당 팀원에게 SendMessage로 재시작 요청
- 전체 진행률은 `TaskList`/`TaskGet`으로 확인

### Phase 4: 통합 (insight-analyst + 실시간 질의)

deep-reader와 context-scout의 작업이 완료되면 insight-analyst의 `blockedBy`가 해제되어 자동 활성화된다.

insight-analyst는 `{paper_dir}/02_deep_reader.md`와 `{paper_dir}/03_context_scout.md`를 Read한 뒤 통합 보고서 작성. **핵심 가치:** 같은 팀 안에서 다른 팀원이 아직 살아있으므로, 불명확한 지점에 대해 직접 SendMessage 질의 가능:

- insight-analyst → deep-reader: "§4.2의 수식 해석 확신 수준은? 내가 보기엔 변수 정의가 암묵적이라…"
- insight-analyst → context-scout: "비판 스레드 A의 신뢰도 어떻게 판단했나? 1명 스레드 vs 재현 연구?"

수신 팀원은 실시간 답변을 보내 insight-analyst의 교차 검증 품질을 올린다.

완료 시 `TaskUpdate(status: "completed")`로 작업 마감 → 리더에게 자동 idle 알림.

### Phase 5: 정리

1. 리더가 모든 작업 완료 확인 (`TaskList`)
2. 팀원들에게 종료 요청:
   ```
   SendMessage(to: "*", message: {type: "shutdown_request", reason: "팀 작업 완료"})
   ```
3. 모든 팀원 shutdown 완료 대기 후 `TeamDelete`
4. 사용자에게 결과 요약:
   - 최종 보고서 경로 (예: `_workspace/2604.14572_corpus2skill/report.md`)
   - 팀원 간 통신 요약 (몇 회 SendMessage, 권한 차단 발생 여부)
   - 수집 성공·실패 영역
5. 후속 작업 안내 — "특정 섹션 보완, 커뮤니티 반응 보강 등 요청 가능"

`_workspace/` 디렉토리 및 하위 폴더는 **절대 삭제하지 않는다** (누적 아카이브).

### 다중 논문 모드: Phase 2~5 순차 반복

세션당 팀 1개 제약 때문에 동시에 여러 논문 팀을 띄울 수 없다.

```
논문 A: TeamCreate → Phase 3/4 → TeamDelete
논문 B: TeamCreate → Phase 3/4 → TeamDelete
...
마지막 단계: insight-analyst를 **단일 서브 에이전트로** 호출하여 모든 논문의 02/03 파일을 Read → `comparison_{YYYYMMDD}_{slug}/report.md` 비교 매트릭스 작성
```

각 논문 폴더의 개별 `report.md`와 비교 `report.md` 모두 산출물로 남는다.

## 데이터 흐름 (단일 논문, 에이전트 팀)

```
[오케스트레이터 = research-lead]
    │
    ├── Phase 1: mkdir _workspace/{paper_dir}/, 00_plan.md
    │
    ├── Phase 2: TeamCreate(paper-xxx, members=[deep-reader, context-scout, insight-analyst])
    │            TaskCreate(3개 작업, insight-analyst는 blockedBy로 대기)
    │
    ├── Phase 3 (팀원 자체 조율):
    │       deep-reader ──SendMessage──→ context-scout    (본문 발견 선행연구 즉시 조사)
    │       context-scout ──SendMessage──→ deep-reader    (커뮤니티 비판 → §X 재확인)
    │       context-scout ──SendMessage──→ insight-analyst (상충 조기 경보, 큐)
    │       모든 팀원 ──SendMessage──→ research-lead      (권한 차단 즉시 보고)
    │       리더 ──────(사용자에게 실시간 전달)────────→  사용자
    │       파일 저장: {paper_dir}/02_deep_reader.md, 03_context_scout.md
    │
    ├── Phase 4 (blockedBy 해제):
    │       insight-analyst Read 02/03 → 통합 중 불명확
    │         ──SendMessage──→ deep-reader/context-scout (실시간 질의)
    │         ←───답변───
    │       파일 저장: {paper_dir}/report.md
    │
    └── Phase 5: shutdown_request → TeamDelete → 사용자 보고
```

## 에러 핸들링

| 상황 | 전략 |
|------|------|
| arXiv ID 추출 실패 | 사용자에게 확인 요청 (URL·제목 재확인) |
| slug 결정 애매 | 논문 제목 첫 의미 단어 2개 사용. 실패 시 `unknown` |
| **팀원의 도구 권한 차단** | 팀원이 리더에게 SendMessage 보고 → 리더가 사용자에게 실시간 통보(추가 권한 JSON 제시) → 사용자가 `settings.local.json` 수정 → 리더가 팀원에게 재시도 지시 |
| 팀원 1명 실패/중단(권한 외 원인) | 리더가 유휴 알림 감지 → SendMessage로 상태 확인 → 재시작 지시 1회 → 재실패 시 해당 영역 "미수집" 명시하고 Phase 4 진행 |
| 팀원 과반 실패 | 사용자에게 알리고 진행 여부 확인 |
| deep-reader와 context-scout의 상충 발견 | insight-analyst가 양측 출처 병기, 삭제 금지 |
| 작업이 blockedBy에서 풀리지 않음 | 리더가 `TaskGet`으로 의존 작업 상태 수동 확인 후 필요 시 `TaskUpdate` |
| insight-analyst 실패 | 1회 재시도. 재실패 시 `_workspace/` 경로를 사용자에게 안내하고 수동 통합 제안 |
| 이미 존재하는 폴더 (중복 논문 요청) | Phase 0 "강제 재실행" 분기로 처리. 기존 폴더는 타임스탬프 suffix 붙여 보존 |
| 다중 논문 중 일부 논문 팀 실패 | 성공한 논문만으로 비교 진행, 실패 논문 명시 |

## 테스트 시나리오

### 정상 흐름 1: 단일 논문 (초기 실행, 권한 문제 없음)

1. 사용자: "https://arxiv.org/abs/2604.14572 분석해줘"
2. Phase 0: `_workspace/2604.14572_*/` 미존재 → 초기 실행
3. Phase 1: 폴더명 `2604.14572_corpus2skill` 결정, `mkdir` 수행
4. Phase 2: `TeamCreate(paper-2604-14572, members=[deep-reader, context-scout, insight-analyst])` + 3개 작업 등록
5. Phase 3: deep-reader와 context-scout 병렬 작업. 중간에 context-scout가 HN 스레드에서 "이 방법 재현 안 됐다" 보고 발견 → deep-reader에게 "§5.2 재확인 부탁" SendMessage → deep-reader 수정 후 파일 저장
6. Phase 4: insight-analyst가 blockedBy 해제되어 활성화. 통합 중 context-scout의 커뮤니티 비판에 대한 신뢰도를 질의 → context-scout 답변 → 보고서에 "비판 신뢰도 중간 수준" 명시
7. Phase 5: shutdown_request → TeamDelete → 사용자 보고("팀원 간 SendMessage 3회, 권한 차단 없음, 보고서: …/report.md")

### 정상 흐름 2: 권한 차단 실시간 피드백

1. Phase 3 중 context-scout가 `WebFetch(domain:reddit.com)` 호출 실패
2. context-scout가 research-lead에게 SendMessage: "reddit.com 도메인 차단 감지. 커뮤니티 반응 수집 중단"
3. 리더가 사용자에게 전달: "context-scout가 reddit.com 권한 차단. settings.local.json에 `WebFetch(domain:reddit.com)` 추가 필요. 수정 후 '재시도' 답해주세요"
4. 사용자 수정 후 "재시도" 응답
5. 리더가 context-scout에게 SendMessage: "권한 업데이트됨. 재시도"
6. 정상 진행 재개

### 정상 흐름 3: 두 번째 논문 (기존 유지)

1. 사용자: "이 논문도 분석: arXiv:2502.YYYYY"
2. Phase 0: 새 논문 → 새 폴더 생성. **기존 `2604.14572_*` 폴더는 그대로 보존**
3. Phase 2~5: 새 팀 구성 후 단일 논문 흐름 반복 (이전 팀은 Phase 5에서 이미 TeamDelete됨)

### 정상 흐름 4: 부분 재실행

1. 사용자: "2604.14572 논문의 커뮤니티 반응 더 찾아줘"
2. Phase 0: 해당 폴더 존재 → 부분 재실행 분기 → context-scout + insight-analyst 2인 팀 구성
3. deep-reader는 팀에 포함시키지 않음 (기존 `02_deep_reader.md` 재활용)
4. context-scout 작업 완료 후 insight-analyst가 `02_deep_reader.md`(기존) + `03_context_scout.md`(갱신됨) 읽어 `report.md` 재생성
5. Phase 5: 정리

### 정상 흐름 5: 다중 논문 비교

1. 사용자: "이 세 논문 비교: [URL1, URL2, URL3]"
2. Phase 1: 세 논문 폴더 + `comparison_{YYYYMMDD}_{slug}/` 생성
3. 각 논문에 대해 Phase 2~5 순차 반복 (각 팀은 생성 → 실행 → 삭제)
4. 모든 논문 개별 `report.md` 완료 후, insight-analyst를 **단일 서브 에이전트**로 호출 → 세 폴더의 02/03 파일을 Read하여 `comparison_*/report.md` 비교 매트릭스 작성
5. 사용자에게 개별 3개 + 비교 1개 총 4개 보고서 경로 전달

### 에러 흐름 1: deep-reader 중단

1. Phase 3에서 deep-reader가 PDF 접근 불가로 실패
2. 1회 재시도 지시 → 여전히 실패
3. context-scout는 정상 완료
4. insight-analyst에게 SendMessage로 "deep-reader 영역 미수집" 통보
5. insight-analyst가 context-scout 산출물만으로 제한적 보고서 작성 + "미수집 영역" 섹션 명시

### 에러 흐름 2: 중복 논문 요청

1. 사용자가 이미 분석한 논문을 "다시 분석해줘" (부분 수정 지시 없이)
2. Phase 0: 강제 재실행 분기 → 기존 폴더를 `{paper_dir}_20260421_183000/`로 rename
3. 새 폴더로 초기 실행 (Phase 2~5)

## 참조 파일

- 에이전트 정의: `.claude/agents/{research-lead,deep-reader,context-scout,insight-analyst,paper-hunter}.md`
- 하네스 템플릿: `~/.claude/plugins/cache/harness-marketplace/harness/1.2.0/skills/harness/references/orchestrator-template.md` (템플릿 A)
- 팀 예시: 동 디렉토리 `team-examples.md` (예시 1: 리서치 팀)
