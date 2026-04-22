# 리서치 계획 — 2604.14572 (corpus2skill)

## 대상 논문
- **제목:** Don't Retrieve, Navigate: Distilling Enterprise Knowledge into Navigable Agent Skills for QA and RAG
- **저자:** Yiqun Sun, Pengfei Wei, Lawrence B. Hsieh
- **게재:** arXiv, 2026-04-16 (cs.IR)
- **URL:** https://arxiv.org/abs/2604.14572

## 입력 유형
`single-paper` — 단일 논문 심층 분석

## 실행 모드
에이전트 팀 (TeamCreate 기반). 이번 세션이 **팀 모드 전환 후 첫 실전 검증**이므로 팀원 간 SendMessage 발생·권한 피드백 루프 동작 여부를 명시적으로 검증.

## 팀 구성 (Phase 2)
- 팀명: `paper-2604-14572`
- 팀원: deep-reader, context-scout, insight-analyst
- 작업: 본문 정독 / 외부 맥락 수집 / 통합 보고서 (통합은 앞 둘의 `blockedBy`)

## 검증 체크포인트
1. TeamCreate 정상 실행
2. 팀원 간 SendMessage 최소 1회 이상 실제 발생
3. 권한 차단 발생 시 실시간 피드백 동작 (없으면 "미발생" 명시)
4. 최종 report.md에 팀원 간 조정 흔적 (예: context-scout 비판 발견 → deep-reader §X 해석 수정)

## 예상 산출물
- `02_deep_reader.md`: 논문 본문 섹션별 정독
- `03_context_scout.md`: 저자 공식 자산, 선행/후속 연구, 커뮤니티 반응
- `report.md`: 비판적 통합 보고서 + 팀원 간 조정 흔적 기록
