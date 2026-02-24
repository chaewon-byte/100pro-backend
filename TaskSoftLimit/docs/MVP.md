# MVP — ActiveTaskCount 기반 Soft 제한 실험 인프라

## 1. MVP 범위

본 MVP는 **Soft 제한 실험 인프라**의 최소 구현 범위를 정의한다.  
UI/팝업 없이 **백엔드 도메인·이벤트·API**만 구현하며, 이후 행동 추적 및 실험 분석이 가능하도록 한다.

---

## 2. MVP 목표

- 목표 생성 시 **ActiveTaskCount 계산** 및 **과부하(≥6) 시 guide_exposed 기록**
- 목표 **생성은 항상 허용**
- **이벤트 로그** 저장 구조 확보 (guide_exposed, task_create, task_modify, app_close, task_complete)
- 실험 단위(**6번째 목표 생성 시도 첫 발생**) 식별 가능한 데이터 구조

---

## 3. MVP 기능 목록

| # | 기능 | 설명 | 우선순위 |
|---|------|------|----------|
| 1 | **ActiveTaskCount 계산** | 당일·활성 상태인 목표 수 계산 (설정 기반 threshold=6) | P0 |
| 2 | **과부하 판별** | ActiveTaskCount ≥ guide exposure threshold 여부 판별 | P0 |
| 3 | **guide_exposed 이벤트 기록** | 과부하 시 GoalEventLog에 guide_exposed 기록 | P0 |
| 4 | **목표 생성 처리 흐름** | 계산 → 과부하 판별 → 이벤트 기록 → 생성 허용 | P0 |
| 5 | **task_create 이벤트 기록** | 목표 생성 시 task_create 로그 | P0 |
| 6 | **task_modify / app_close / task_complete** | 각 행동 시 이벤트 로깅 (추적 가능 구조) | P0 |
| 7 | **설정 기반 threshold** | guide exposure threshold, ActiveTaskCount Cap 설정값으로 관리 | P1 |
| 8 | **목표 생성 API** | 목표 생성 요청 수신 → 서비스 흐름 수행 → 생성 허용 응답 | P0 |

---

## 4. MVP에서 제외

- 팝업·배너·가이드 문구 UI (UX 요소 구현 금지)
- Hard Limit(생성 차단) 로직 (실험 Treatment는 별도)
- 실험 그룹 배정 로직 (분석 단계에서 이벤트 기반 코호트 구성 가능하면 충분)
- OverloadIndex·추천 목표 수 계산 (추후 확장)

---

## 5. 인수 조건 (Acceptance Criteria)

1. **목표 생성 요청** 시:
   - ActiveTaskCount가 계산되고,
   - ActiveTaskCount ≥ 6이면 **guide_exposed** 이벤트가 저장되며,
   - 목표는 **항상 생성**된다 (에러로 차단되지 않음).
2. **GoalEventLog**에 다음 이벤트 타입이 기록 가능하다: guide_exposed, task_create, task_modify, app_close, task_complete.
3. **설정**으로 guide exposure threshold(기본 6), ActiveTaskCount Cap(기본 5)을 변경할 수 있다.
4. **실험 단위** 식별을 위해, 6번째 목표 생성 시도 시점(또는 guide_exposed 최초 발생 시점)을 이벤트 로그로 복원할 수 있다.

---

## 6. 참조

- [PRD](./PRD.md)
- [설계서](./설계서.md)
- [문제정의서](./문제정의서.md), [해결방안탐색문서](./해결방안탐색문서.md)
