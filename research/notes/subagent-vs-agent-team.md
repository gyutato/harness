# 서브 에이전트(Sub-agent) vs 에이전트 팀(Agent Team) — 무엇을 언제 쓰는가

Claude Code에서 멀티 에이전트 워크플로우를 설계할 때 두 가지 실행 모드를 선택할 수 있다. 이 둘은 기능의 차이가 아니라 **구조의 차이**이고, 잘못 고르면 기능 자체가 성립하지 않는 경우가 생긴다. 이 문서는 구체적 실패 사례 하나를 바탕으로 차이와 선택 기준을 정리한다.

---

## 1. 기본 개념 — 한 줄 정의

**서브 에이전트 (Sub-agent):** 메인 에이전트가 `Agent` 도구로 서브 에이전트를 생성한다. 서브 에이전트는 작업을 수행한 뒤 결과를 **메인에게만 반환**하고 종료된다. 서브끼리는 서로를 모른다.

```
[메인] → [서브 A] → 결과 반환
      → [서브 B] → 결과 반환
      → [서브 C] → 결과 반환
```

**에이전트 팀 (Agent Team):** 리더가 `TeamCreate`로 팀원을 구성한다. 팀원들은 독립 Claude Code 인스턴스로 실행되면서 `SendMessage`로 서로 직접 통신하고, 공유 작업 목록(`TaskCreate`)에서 자체적으로 작업을 요청(claim)한다.

```
[리더] ←→ [팀원 A] ←→ [팀원 B]
  ↕          ↕          ↕
  └──── 공유 작업 목록 ────┘
```

---

## 2. 구조적 비교

| 차원 | 서브 에이전트 | 에이전트 팀 |
|------|--------------|------------|
| **핵심 도구** | `Agent(run_in_background=true)` | `TeamCreate` + `SendMessage` + `TaskCreate` |
| **에이전트 간 통신** | **불가능** — 파일/반환값으로만 간접 전달 | `SendMessage`로 직접 통신 |
| **결과 수집 시점** | 각자 완료 후 메인이 순차 조립 | 실행 중에도 팀원 간 부분 결과 교환 가능 |
| **작업 분배** | 호출 시점에 사전 고정 | 공유 작업 목록에서 동적 요청(claim)·재할당 |
| **유휴 알림** | 없음 (호출자가 직접 체크) | 팀원 대기 상태가 리더에게 자동 통보 |
| **사용자-에이전트 채널** | 메인만 대화 가능. 서브는 고립 | 리더가 팀원 보고를 받아 사용자와 실시간 중계 가능 |
| **세션 제약** | 없음 (병렬 N개 자유) | 세션당 1팀만 활성. Phase 간 `TeamDelete` → 새 팀 구성은 가능 |
| **토큰 비용** | 낮음 | 높음 (팀원 간 통신 오버헤드) |
| **설계 복잡도** | 낮음 | 높음 (통신 프로토콜 정의 필요) |

---

## 3. 실제 사례 — 서브 모드가 만든 두 층의 손실

논문 리서치 하네스를 구축하는 작업이었다. 사용자 요구는 다음과 같았다.

> "여러 각도에서 조사할 수 있는 **에이전트 팀** — 웹 검색, 학술 자료, 커뮤니티 반응 — **교차 검증** 후 종합 보고서"

오케스트레이터는 "에이전트마다 소스가 독립적이니 실시간 토론이 굳이 필요 없다"는 판단으로 서브 에이전트 모드를 선택했다. 에이전트 4명을 정의하고 병렬 호출하는 구조였다.

첫 실행에서 두 종류의 실패가 발생했다.

### 손실 1 — 교차 검증 기능이 구조적으로 성립하지 않음

사용자가 요구한 "교차 검증"은 여러 정보원의 주장을 비교해 상충점을 찾고 조정하는 행위다. 서브 모드에서 이게 작동하려면 각 에이전트가 **수집 중**에 다른 에이전트의 발견을 인지해야 한다. 그러나 서브 에이전트끼리는 통신 수단이 없다. 결과적으로 모든 에이전트가 끝난 뒤 단일 통합자(보고서 작성자)가 **파일만 보고 혼자 판단**하는 구조가 되었다. 이는 "사후 대조"이지 "교차 검증"이 아니다.

팀 모드였다면 가능했던 상호작용 예시:

- 본문 독해 에이전트가 논문에서 경쟁 시스템 이름만 발견 → 외부 조사 에이전트에게 "해당 시스템 실체 우선 조사 부탁" SendMessage → 조사 방향 즉시 전환
- 외부 조사 에이전트가 커뮤니티에서 "이 논문의 실험 방법에 논리 결함 있다"는 반응 발견 → 본문 독해 에이전트에게 "해당 섹션 재확인 요청" SendMessage → 재독해 후 해석 보정
- 두 에이전트 사이에 상충 해석 발견 → 실시간 토론으로 합의 도출 또는 양측 병기

이 모든 게 서브 모드에서는 **구조적으로 발생할 수 없다**.

### 손실 2 — 사용자-에이전트 실시간 피드백 루프 부재

외부 조사 에이전트가 실행 중 WebFetch 도메인 접근 권한이 차단되어 **5분 37초 동안 49회의 도구 호출 시도가 모두 실패**하는 사태가 벌어졌다. 차단 대상은 Hacker News, Reddit, Semantic Scholar, 주요 블로그 플랫폼 등 조사의 핵심 경로였다.

사용자는 이 실패를 **언제** 알게 되었는가? 실행이 완료되고, 최종 보고서의 "수집 한계" 섹션을 읽은 뒤였다. 즉 5분 37초는 그대로 버려졌고, 사용자가 개입할 기회 자체가 없었다.

서브 에이전트는 사용자와 대화 채널이 없다. 권한 프롬프트를 띄울 수 없고, 막혀도 누구에게 보고할 길이 없다. 자동 실패 후 "수집 한계" 같은 기록을 남기는 것이 최선이다.

팀 모드였다면 다음 흐름이 가능했다:

```
팀원: "HN 도메인 WebFetch 막힘"
  ↓ SendMessage
리더: 알림 즉시 수신
  ↓ 사용자에게 보고
사용자: "권한 추가할게" → settings 수정
  ↓
팀원: 도구 재시도 → 성공
```

**에이전트-사용자 간 실시간 피드백 루프는 팀 모드의 핵심 가치다.** 외부 도구 의존도가 높은 워크플로우에서는 이 루프가 있느냐 없느냐가 실패율을 크게 바꾼다.

### 정리

원래 요구는 "에이전트 팀 + 교차 검증"이었고, 구현은 "서브 에이전트 + 단일 통합자"였다. 기술적 경량화 이점만 보고 선택했지만, 결과는 두 가지 상실이었다:

1. 요구된 기능(교차 검증)이 구조적으로 불가능
2. 도구 의존 워크플로우에서 실시간 권한 피드백 차단

두 손실 모두 **사용자 통보 지연 + 토큰 낭비 + 결과 품질 저하**로 이어졌다.

---

## 4. 언제 서브 모드가 맞고, 언제 팀 모드가 맞나

### 서브 에이전트를 선택하는 경우

- **작업이 완전히 독립적이고 병렬화만 필요**할 때 — 예: 100개 파일에 대한 lint 수정, N개 페이지 스크래핑
- **단발 전문가 호출** — 특정 도메인 전문가를 한 번 부르고 결과만 받으면 끝
- **토큰 예산이 빡빡**한 대규모 반복 작업
- **중첩 구조** — 서브가 또 서브를 호출해야 할 때 (팀은 중첩 불가)
- 결과를 받는 쪽이 오로지 메인 하나일 때

### 에이전트 팀을 선택하는 경우

- **교차 검증·상호 비판**이 결과 품질의 핵심일 때 — 리서치, 코드 리뷰, 다관점 분석
- **외부 도구 의존도가 높아** 실패 시 사용자 개입이 필요할 때 — WebFetch/WebSearch/MCP 조합
- **작업 의존성이 런타임에 결정**될 때 — "A의 발견에 따라 B의 조사 방향이 바뀐다"
- **감독자 패턴** — 공유 작업 목록에서 팀원이 자체 요청으로 동적 분배
- **실시간 피드백**이 가치를 만들 때 — 생성-검증, 설계-구현-검증 루프

### 하이브리드가 맞는 경우

전체를 한 모드로 통일하지 않아도 된다. Phase별로 섞을 수 있다:

- **단발 전문가 호출(서브) → 본격 수집(팀) → 독립 검증(서브)** 같은 구성이 흔히 효과적
- Phase 전환 지점에서 이전 팀은 `TeamDelete`로 정리하고, 산출물은 파일로 넘긴다
- 세션당 팀 1개 제약을 우회하면서 팀 모드의 이점을 필요한 구간에만 적용

---

## 5. 선택 전 체크리스트

다음 중 하나라도 "yes"면 **팀 모드를 기본**으로 검토한다:

- [ ] 사용자 요구에 "교차 검증", "상호 확인", "토론", "다관점 합의" 같은 표현이 있다
- [ ] 에이전트들이 **동일 대상**을 다른 각도에서 본다 (같은 논문을 본문/맥락으로 나눠 분석 등)
- [ ] 외부 도구(WebFetch/WebSearch/MCP/DB)가 **워크플로우 성공의 핵심 경로**에 있다
- [ ] 실패했을 때 사용자 개입으로 복구 가능한 유형의 실패가 예상된다 (권한, 도메인, 인증 등)
- [ ] 한 에이전트의 발견이 다른 에이전트의 작업 범위를 바꿀 수 있다
- [ ] 작업 수가 런타임에 결정되거나 동적 재할당이 필요하다

다음 중 **모두 "yes"면** 서브 모드가 더 맞을 가능성이 높다:

- [ ] 각 에이전트의 입력과 산출물이 완전히 결정되어 있다
- [ ] 에이전트 간 중간 발견 공유가 결과에 영향을 주지 않는다
- [ ] 외부 도구 의존이 없거나 실패 시 복구 없이 누락 처리로 충분하다
- [ ] 결과를 받는 주체가 오로지 메인 하나다
- [ ] 팀 통신 오버헤드를 감당할 예산이 없다

---

## 6. 설계 시 흔한 함정

### 함정 1: "소스가 독립적이니 서브로 충분하다"

입력 소스가 독립적이라는 것과 **결과 해석이 독립적**이라는 것은 다른 문제다. 서로 다른 소스에서 모은 정보를 **통합 해석하려면** 수집 중의 교차 참조가 거의 항상 품질을 높인다. 수집 중 교차 참조 없이 사후에 혼자 통합하는 사람은 "각자가 수집한 걸 이어 붙인 보고서"만 만든다.

### 함정 2: "사용자 명시 요구"를 기술 판단으로 대체

사용자가 "팀"이라는 단어를 쓴 것은 구현 세부를 모르고 한 말이 아닐 수 있다. 용어 선택 자체에 구조적 기대가 담겨 있을 확률이 높다. 기술적으로 서브가 "더 효율적"이어도, 그게 사용자 요구를 **충족하지 못하면** 효율이 아니다.

### 함정 3: 서브 에이전트의 "권한 침묵"을 과소평가

서브 에이전트가 도구 권한 차단을 만나면 보통 **조용히 실패하거나 "수집 한계"로만 기록**한다. 사용자는 이 침묵을 실행 종료 후에야 발견한다. 외부 도구 의존이 있는 워크플로우에서 이 지연은 치명적이다.

### 함정 4: 팀 모드의 오버헤드를 과대평가

팀 모드가 토큰이 더 든다는 건 맞지만, "실패 후 재실행"의 비용과 비교해야 공정하다. 권한 막힘으로 5분 낭비한 뒤 settings 고치고 재실행하는 것보다, 첫 실행 중에 리더가 1분 만에 알려주고 사용자가 바로 조치하는 편이 **총 비용은 낮다**.

### 함정 5: 팀원의 자기 인식 오류 — "나는 subagent인가 teammate인가"

팀 모드 구성이 올바르게 되어도, **팀원 에이전트 본인이 자기 정체성을 "subagent"로 잘못 인식**하면 subagent 관례("findings를 텍스트로 반환하고 파일을 직접 쓰지 않는다")를 자발적으로 적용해 출력 프로토콜을 위반할 수 있다. 공식 문서는 팀원을 명확히 **"full, independent Claude Code session"**으로 규정하지만, `Agent` 도구의 `subagent_type` 파라미터가 공유되는 구현 세부 때문에 에이전트가 학습된 subagent 행동 규약을 끌어쓸 위험이 있다.

**실전 사례:** 논문 리서치 하네스의 2604.14572 실전 검증에서 `insight-analyst` 팀원이 `{paper_dir}/report.md` 출력 프로토콜을 명시적으로 받았음에도 Write 도구 호출을 스스로 억제했다. 본인은 `"Subagents should return findings as text, not write report files"`라는 `tool_use_error`를 받았다고 보고했으나, 전역·플러그인 hooks 전수 조사에서 해당 차단 hook은 발견되지 않았고, 메시지 톤도 `system-reminder` 형식에 가까웠다. 에이전트가 자기 인식 오류로 **의사-에러를 생성**했을 가능성이 높다. 결과적으로 team-lead가 3회 재지시 후에야 에이전트가 `Bash heredoc` 우회로 파일을 저장했다.

**방어 방법:**
- **1차 (주):** 에이전트 정의 파일(`.claude/agents/{teammate}.md`) 상단에 "실행 컨텍스트와 정체성" 섹션을 두어, 팀원 본인이 spawn 시 자기 정체성을 "full, independent Claude Code session"으로 선언하도록 하드코딩. 공식 문서 인용 포함. "Subagents should return findings as text류의 내부 추론은 이 하네스의 규약이 아니다" 명시.
- **2차 (부):** 오케스트레이터 스킬 파일(SKILL.md)의 "에이전트 팀 모드" 섹션에 리더용 운영 원칙으로 "팀원을 subagent로 대하지 말 것, spawn 프롬프트에 해당 뉘앙스를 넣지 말 것"을 명시.
- 1차가 본질적 교정 — 문제 발생 지점이 "에이전트의 자기 인식"이며 SKILL.md는 팀원이 읽지 않기 때문. 2차는 리더의 작업 일관성 보조.

**이 함정의 교훈:** 팀 모드 전환은 **구조 설계**만으로 끝나지 않는다. 에이전트 본인의 **정체성 선언**까지 교정해야 팀 모드의 "full independence" 전제가 실제로 작동한다. experimental 단계의 숨은 구조적 문제이자 Phase 7 수준의 하네스 개선 과제.

---

## 7. 팀 모드에서 쓰는 도구 — 레퍼런스

팀 모드는 여러 도구의 조합이다. `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` 환경변수로 활성화되며, 대부분 deferred tool이라 `ToolSearch`로 스키마를 먼저 로드해야 호출 가능하다.

### 7.1 팀 생성·해체

| 도구 | 역할 | 핵심 제약 |
|------|------|-----------|
| `TeamCreate` | 팀 + 팀 전용 작업 목록 동시 생성 (1:1 대응). `~/.claude/teams/{name}/config.json`과 `~/.claude/tasks/{name}/` 생성 | 세션당 1팀만 활성 |
| `TeamDelete` | 팀과 작업 목록 디렉토리 정리 | 팀원이 active면 실패. 먼저 각 팀원에게 `shutdown_request`를 보내 종료 후 호출 |

Phase마다 다른 팀 구성이 필요하면 `TeamDelete` → `TeamCreate` 반복으로 우회. 이전 팀 산출물은 파일로 저장해 다음 팀이 읽게 한다.

### 7.2 통신 — `SendMessage`

팀 모드의 **심장**. 팀원의 plain text 출력은 다른 팀원에게 **보이지 않는다**. 소통하려면 반드시 SendMessage.

- `to`: 팀원 `name` (UUID 아님, 항상 name으로 참조) 또는 `"*"` (브로드캐스트)
- `message`: plain text 또는 프로토콜 구조체
- `summary`: UI 프리뷰용 5~10단어 (plain text일 때 필수)
- **브로드캐스트 `*`는 비싸다** — 팀 크기에 비례. 정말 모두가 알아야 할 때만
- 메시지 수신은 **자동** — inbox 확인 불필요, 새 대화 턴처럼 도착

**프로토콜 메시지 (legacy):**
```json
{"to": "researcher", "message": {"type": "shutdown_request", "reason": "work complete"}}
{"to": "team-lead",  "message": {"type": "plan_approval_response", "request_id": "...", "approve": true, "feedback": "..."}}
```
- `shutdown_request` 수락 시 해당 팀원 프로세스 종료
- `plan_approval_request`는 팀원이 위험한 작업 전에 리더 승인받을 때 씀. 리더가 `_response`로 approve/reject

### 7.3 공유 작업 목록 — Task 도구

**팀 1개 = 작업 목록 1개 (1:1)**. 팀원 모두 같은 목록을 본다. 실체는 `~/.claude/tasks/{team-name}/`.

| 도구 | 용도 |
|------|------|
| `TaskCreate` | 새 작업 추가 (초기 분배, 작업 중 발견한 파생 작업) |
| `TaskList` | 전체 작업 조회. **작업 완료 직후 바로 호출해 다음 일 찾기** |
| `TaskGet` | 특정 작업의 상세(description, blockedBy/blocks) 조회 — 시작 전에 |
| `TaskUpdate` | 상태 변경, `owner` 설정(claim), 의존성 추가 |

> `TaskOutput`/`TaskStop`은 팀 작업 목록과 무관 — 백그라운드 프로세스(Bash `run_in_background`, Agent 등)용이다. 헷갈리지 말 것.

**표준 claim 패턴:**
```
TaskList → pending + owner 없는 작업 발견
       → TaskUpdate(owner: 자기 name)            # 선점
       → TaskGet(taskId)                          # 상세 확인
       → 작업 수행 (필요 시 SendMessage로 협업)
       → TaskUpdate(status: "completed")
       → TaskList → 다음 작업 또는 idle
```

**작업 ID 순 선호**: 여러 pending 작업 중 선택할 때는 가장 낮은 ID 우선 — 먼저 만든 작업이 후속 작업의 컨텍스트를 설정하는 경우가 많다.

### 7.4 팀원 스폰 — `Agent` + 팀 파라미터

팀원도 `Agent` 도구로 스폰하지만 `team_name`과 `name` 파라미터를 추가해 팀에 합류시킨다.

- `subagent_type`: `general-purpose` / `Explore`(읽기 전용) / `Plan`(읽기 전용) / `.claude/agents/`의 커스텀 타입
- **읽기 전용 타입에게 구현 작업 금지** — Edit/Write 권한 없음. 연구·계획 전용
- `name`: 팀 내 고유 식별자. 다른 팀원이 `SendMessage(to: "...")`와 `TaskUpdate(owner: "...")`로 이 이름을 참조

### 7.5 Idle 알림 — 팀 모드 고유 피드백 채널

팀원은 **매 턴 끝에 자동으로 idle 상태**가 되고 리더에게 알림을 보낸다. 팀 모드의 감시 비용을 0으로 만드는 핵심 메커니즘이다.

- Idle ≠ 완료. 팀원이 메시지 보낸 뒤 idle 되는 것은 **정상 흐름** (응답 대기 중)
- 리더가 즉시 개입할 필요는 없음 — 실제 막혔을 때만
- Idle 상태 팀원에게 `SendMessage` → 깨어나서 응답
- 팀원끼리 DM을 주고받으면 그 **요약이 리더의 idle 알림에 포함**됨 → 리더가 팀원 간 협업 상황을 가시화할 수 있음 (별도 폴링 없이)

### 7.6 팀 구성원 탐색

팀원이 서로의 존재를 알려면 팀 config 파일을 읽는다:

```
~/.claude/teams/{team-name}/config.json
```

`members` 배열에 각 팀원의 `name`, `agentId`, `agentType`이 기록돼 있다. **항상 `name`으로 호칭** (agentId/UUID 사용 금지).

**왜 프로젝트 로컬(`.claude/`)이 아니라 전역 경로(`~/.claude/`)인가 — 공식 근거:**

공식 문서는 팀 config이 **런타임 상태**(session IDs, tmux pane IDs 등)를 담고 있으며 팀원 join/idle/leave 시마다 Claude Code가 **자동으로 덮어쓴다**고 명시한다. 그래서 손으로 편집하거나 사전 작성(pre-author)하지 말라는 경고가 딸려 있다. 프로젝트 repo는 소스용이지 세션 특화 휘발성 상태용이 아니다.

또한 프로젝트 로컬 equivalent는 **명시적으로 금지**된다 — `.claude/teams/teams.json` 같은 파일을 프로젝트에 두어도 Claude는 configuration으로 인식하지 않고 ordinary file로 처리한다. 실수로 repo에 커밋되는 걸 구조적으로 차단하는 설계.

> "The team config holds runtime state such as session IDs and tmux pane IDs, so don't edit it by hand or pre-author it: your changes are overwritten on the next state update."
>
> "There is no project-level equivalent of the team config. A file like `.claude/teams/teams.json` in your project directory is not recognized as configuration; Claude treats it as an ordinary file."
>
> — [Claude Code Docs: Orchestrate teams of Claude Code sessions](https://code.claude.com/docs/en/agent-teams)

**실무 함의:** 산출물은 프로젝트에, 런타임 상태는 전역에 — 분리 설계. 팀 운영 흔적을 git에 남기고 싶다면 오케스트레이터나 팀원이 `_workspace/` 같은 프로젝트 내부 파일에 "팀 실행 로그"를 **명시적으로 기록**해야 한다. 자동으로 남지 않는다.

### 7.7 환경변수

| Env | 용도 |
|-----|------|
| `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` | 팀 도구(`TeamCreate`, `SendMessage`, `TeamDelete`) 활성화. 미설정 시 deferred tools 리스트에 아예 안 나타남 |

`~/.claude/settings.json`의 `env` 블록에 넣어 세션 시작 시 로드. 프로젝트 범위로만 원하면 `.claude/settings.json`의 `env`.

### 7.8 전체 흐름 예시

**리더 관점:**
```
1. TeamCreate(team_name: "research-team", description: "논문 X 분석")
2. TaskCreate × N — 초기 작업 목록 구성 (blockedBy로 의존성 명시)
3. Agent(team_name: "research-team", name: "reader",  subagent_type: "general-purpose", prompt: ...)
   Agent(team_name: "research-team", name: "scout",   ...)
   Agent(team_name: "research-team", name: "analyst", ...)
4. TaskUpdate(taskId_A, owner: "reader")   # 명시 할당, 또는 팀원이 자체 claim 허용
5. [대기 — 팀원들이 SendMessage / TaskUpdate로 자체 조율]
   idle 알림 수신 → 필요 시 개입
6. 모든 작업 completed 확인 → 각 팀원에게 SendMessage(shutdown_request)
7. TeamDelete
```

**팀원 관점:**
```
1. TaskList → 자기 owner 작업 or pending unassigned 발견
2. TaskGet(taskId)으로 상세 확인
3. 작업 수행 — 다른 팀원 도움 필요 시 SendMessage
4. 완료 → TaskUpdate(status: "completed")
5. TaskList → 다음 작업 또는 자동 idle
```

### 7.9 하지 말 것

- **plain text로 팀원에게 말하기** — 안 보임. 반드시 SendMessage
- **`{"type":"idle"}`, `{"type":"task_completed"}` 같은 구조화된 상태 메시지 직접 전송** — 이건 시스템이 자동 처리. 일반 소통은 plain text로, 작업 상태는 TaskUpdate로
- **UUID로 팀원 호칭** — 항상 `name`
- **TeamDelete를 팀원이 active일 때 호출** — 실패함. shutdown 먼저
- **terminal 툴로 팀 활동 엿보기** — config.json Read는 구성원 탐색용으로 허용이지만, 작업 상태·메시지는 반드시 `TaskList`/`SendMessage`로

---

## 8. 실무 팁

- **Deferred tool 선행 로드**: `TeamCreate`, `SendMessage`, `TaskCreate` 등은 세션에서 기본 로드되지 않는다. 팀 모드 진입 전에 `ToolSearch(query: "select:TeamCreate,SendMessage,TaskCreate,TeamDelete,TaskGet,TaskUpdate,TaskList")`로 스키마를 먼저 확보한다.

- **에이전트 정의 파일에 "팀 통신 프로토콜" 섹션을 두라**: 누가 누구에게 어떤 메시지를 언제 보내는지 명시하지 않으면 팀 모드의 이점이 살지 않는다. "SendMessage 쓰세요"만으론 팀원이 어떤 상황에서 써야 할지 판단하지 못한다.

- **리더의 역할은 "조율 + 사용자 중계"** — 팀원들이 막혔을 때 리더가 사용자와 대화 채널이라는 점을 명심. 리더가 도구 실행에 매몰되면 팀 모드의 핵심 이점이 사라진다.

- **세션당 팀 1개 제약**은 Phase 간 `TeamDelete` → `TeamCreate` 반복으로 우회 가능. 각 Phase의 산출물은 파일로 저장해 다음 팀이 읽게 한다.

- **생성-검증 쌍은 팀 모드가 특히 빛난다** — 생성자와 검증자가 `SendMessage`로 실시간 피드백 교환하면 재작업 횟수가 크게 줄어든다. 예: 이미지 생성자 ↔ 품질 검수자, 코드 작성자 ↔ 리뷰어.

---

## 9. Experimental 상태 — 공식 제약과 출처

팀 모드는 2026년 4월 기준 experimental 기능이며 다음 제약이 공식 문서에 명시되어 있다.

### 9.1 최소 버전

**Claude Code v2.1.32+** (`claude --version`). 이전 버전에선 `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`을 설정해도 팀 도구가 활성화되지 않는다.

### 9.2 Known Limitations (공식)

| 제약 | 실무 영향 |
|------|----------|
| **No session resumption with in-process teammates** | `/resume`, `/rewind` 후 팀원 복원 안 됨 → 리더가 존재하지 않는 팀원에게 메시지 보내려다 실패. 복귀 후엔 팀을 새로 구성해야 함 |
| **Task status lag** | 팀원이 completed 마감을 빠뜨리면 의존 작업이 블록됨 → `TaskList` 주기 점검 + 수동 마감 필요 |
| **Shutdown 지연** | 팀원은 현재 tool call이 끝나야 shutdown. 급히 정리하려 해도 기다려야 함 |
| **세션당 팀 1개** | 동시에 두 팀 불가. Phase 간 `TeamDelete` → `TeamCreate` 반복으로 우회 |
| **팀 중첩 불가** | 팀원은 다시 팀을 만들 수 없음. 리더만 팀 관리자 |
| **리더 고정** | 팀을 만든 세션이 평생 리더. 승격/이양 불가 |
| **Permissions는 spawn 시점에 균일** | 모든 팀원이 리더의 permission mode 상속. spawn 이후 개별 변경은 가능하나, spawn 시 per-teammate 다르게 설정 불가 |
| **Split panes는 tmux 또는 iTerm2 필요** | VS Code 통합 터미널, Windows Terminal, Ghostty에서 split 모드 불가. in-process는 어디서나 동작 |

### 9.3 Subagent definition을 팀원으로 재사용할 때의 비대칭

팀원 spawn 시 `subagent_type`으로 `.claude/agents/` 커스텀 정의를 지정하면 해당 정의의 `tools` allowlist와 `model`은 존중되고 body는 시스템 프롬프트에 append된다. 그러나 **`skills`와 `mcpServers` frontmatter는 팀원으로 실행될 때 적용되지 않는다** — 팀원은 프로젝트/사용자 settings에서 스킬과 MCP를 로드한다. 이 비대칭은 트러블슈팅 시 놓치기 쉬운 포인트.

### 9.4 공식 출처

- **[Orchestrate teams of Claude Code sessions](https://code.claude.com/docs/en/agent-teams)** — 팀 모드 전체 문서 (이 섹션의 인용문·제약 근거)
- [Subagents](https://code.claude.com/docs/en/sub-agents) — 비교 대상 공식 문서
- [Settings](https://code.claude.com/docs/en/settings) — 환경변수, `teammateMode` 설정
- [Hooks: TeammateIdle / TaskCreated / TaskCompleted](https://code.claude.com/docs/en/hooks) — 팀원 품질 게이트 강제에 사용

---

## 10. 한 줄 결론

서브 에이전트는 **"병렬 실행"**의 도구이고, 에이전트 팀은 **"협업"**의 도구다. 병렬로 돌릴 수 있다고 협업이 되는 건 아니다. 요구가 "협업"이면 팀 모드로 간다.
