# PRD — ActiveTaskCount 기반 Soft 제한 실험 인프라

## 1. 개요

| 항목 | 내용 |
|------|------|
| **제목** | 목표 수 Soft 제한 실험 인프라 (ActiveTaskCount 기반) |
| **목적** | 사용자가 목표를 5개 초과하여 생성하려는 상황에서 **생성은 허용하되 가이드만 노출**하고, 이후 행동 변화를 추적할 수 있는 시스템 구축 |
| **범위** | 백엔드 도메인 규칙, 이벤트 로깅, ActiveTaskCount 계산, 목표 생성 처리 흐름 (UX/UI 미포함) |

## 2. 배경 및 문제

- **문제:** ActiveTaskCount ≥ 6 구간에서 task_miss 비율 증가, 7일 유지율 감소 (상세: [문제정의서](./문제정의서.md), [해결방안탐색문서](./해결방안탐색문서.md))
- **해결 방향:** Soft Limit — 생성 차단 없이 가이드만 노출하여 과다 설정 구간 진입 시 사용자 행동을 관찰·분석

## 3. 목표 및 성공 지표

### 목표

- 목표 생성 요청 시점에 **ActiveTaskCount 계산** 후 **과부하(≥6) 시 guide_exposed 이벤트 기록**
- 목표 생성은 **차단하지 않음**
- **task_modify, app_close, task_complete** 등 이후 사용자 행동 추적 가능
- 실험 단위: **"6번째 목표 생성 시도 첫 발생"**

### 성공 지표 (실험 KPI)

- 실패율: `P(task_miss | 6번째 목표 생성 시도, group)`
- 6개 이상 목표 첫 진입 기준 7일 유지율: `P(retention_7d | 6번째 목표 생성 시도(첫 발생), group)`
- 목표 조정률: `P(task_modify | guide_exposed, Soft)` 등
- 즉시 반발: `P(app_close <= 60s | guide_exposed, Soft)` 등

## 4. 도메인 규칙 (핵심)

- 목표 생성 요청 시점에 **ActiveTaskCount 계산**
- **ActiveTaskCount ≥ 6**이면 과부하 상태로 판단
- 목표 생성은 **차단하지 않는다**
- **guide_exposed** 이벤트 기록
- 이후 사용자 행동 추적: task_modify, app_close, task_complete
- 실험 단위: **"6번째 목표 생성 시도 첫 발생"**

## 5. 파라미터 정책

| 파라미터 | 값 | 설명 |
|----------|-----|------|
| **ActiveTaskCount Cap** | 5 | 권장 활성 목표 수 상한 |
| **guide exposure threshold** | 6 | 이 값 이상이면 가이드 노출(이벤트 기록) |
| **threshold 관리** | 설정값 기반 | 환경/설정으로 변경 가능하도록 관리 |

## 6. 기능 요구사항

### 필수

- [ ] 목표 생성 시 ActiveTaskCount 계산 (당일·활성 상태 기준)
- [ ] ActiveTaskCount ≥ 6일 때 **과부하 판별** 후 **guide_exposed** 이벤트 기록
- [ ] 목표 생성 **허용** (생성 차단 로직 없음)
- [ ] 이벤트 로그 저장: guide_exposed, task_create, task_modify, app_close, task_complete
- [ ] Controller → Service → Repository 구조, 이벤트 로그 중심 설계
- [ ] 실험 분석 가능 구조 (실험 단위·그룹·이벤트 시점 보존)

### 비기능

- [ ] UX 요소 구현 금지 (팝업, UI 코드 금지)
- [ ] threshold는 설정(환경/DB 등) 기반으로 관리

## 7. 제약 사항

- 부채형 UI 금지, 강제적 의사결정 요구 금지 (문제정의서 제약 조건 준수)
- 정답 지향·평가 뉘앙스 금지

## 8. 참조 문서

- [문제정의서](./문제정의서.md)
- [해결방안탐색문서](./해결방안탐색문서.md)
- [MVP](./MVP.md)
- [설계서](./설계서.md)
