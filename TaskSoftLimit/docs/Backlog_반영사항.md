# Backlog 반영사항 — Soft Guide 개입 및 실험 인프라

## 목적

ActiveTaskCount ≥ 6 진입 시 **Soft Guide 개입**을 수행하고, 사용자의 **자발적 목표 조정 행동**을 관찰 가능하게 한다.

**가설 (Hypothesis Backlog):** ActiveTaskCount ≥ 6 진입 시 경고/가이드만 노출하면, 자발적 조정을 통한 실패율 감소 효과는 약하지만 **즉시 종료 확률은 감소**할 것이다.

---

## 1. Hypothesis Backlog 반영

### Success criteria (분석/측정)

| # | 기준 | 설명 |
|---|------|------|
| 1 | P(task_miss \| 6번째 목표 생성 시도) 개입 전 대비 **10%p 이상 감소** | task_miss 정의: 24시간 내 미완료. 분석 시점: guide_exposed payload의 `next_task_ordinal=6` 또는 첫 guide_exposed 발생. |
| 2 | P(retention_7d \| 6번째 목표 생성 시도(첫 발생)) 개입 전 대비 **3%p 초과 감소하지 않음** | 7일 유지율이 크게 떨어지지 않아야 함. |
| 3 | P(app_close ≤ 60s \| guide_exposed) 개입 전 대비 **3%p 초과 증가하지 않음** | 가이드 노출 후 즉시 이탈이 크게 늘지 않음. |
| 4 | P(task_modify \| guide_exposed) **≥ 20%** | 가이드 노출 후 목표 수정 행동이 20% 이상 발생. |
| 5 | P(ActiveTaskCount ≤5 \| guide_exposed) **≥ 50%** | 가이드 노출 이후 기간 내 활성 목표가 5 이하로 줄는 비율. |

### To-do (분석/정의)

- 6번째 목표 생성 시도 **분석 시점 고정**  
  → 구현: `guide_exposed` payload에 `next_task_ordinal` 기록. `next_task_ordinal = active_task_count + 1` (생성 시도 순서). 분석 시 `next_task_ordinal = 6` 또는 “첫 guide_exposed”로 코호트 정의.
- P(task_miss \| 6번째 목표 생성 시도) **계산 로직 점검**
- retention_7d **기준 정의 및 계산 로직 검증**
- **guide_exposed 이벤트 로그 정확성 검증**  
  → 구현: `GoalEventLog` 저장, payload에 `active_task_count`, `guide_exposure_threshold`, `next_task_ordinal` 포함.
- P(app_close ≤ 60s \| guide_exposed) **계산 로직 점검**
- P(ActiveTaskCount ≤5 \| guide_exposed) **계산 로직 점검**
- **OverloadIndex 변화량 측정 기준 정의**

---

## 2. Infrastructure Backlog 반영

### Success criteria (구현 완료)

| # | 기준 | 반영 위치 |
|---|------|-----------|
| 1 | task_create 요청 시점의 **ActiveTaskCount 계산 가능** | `ActiveTaskCountService.get_active_task_count(user_id)` |
| 2 | **ActiveTaskCount ≥ 6 실시간 판별 가능** | `OverloadCheckService.is_overload(active_task_count)` + 설정값 `GUIDE_EXPOSURE_THRESHOLD=6` |
| 3 | **guide_exposed 이후에도 task_create 가능** | 목표 생성 차단 없음. `GoalCreateService.execute()` 항상 생성 허용. |
| 4 | **guide_exposed 이후 목표 수정·앱 종료 시점 기록** | 이벤트 타입 `task_modify`, `app_close` 정의. 호출 측에서 해당 시점에 `GoalEventLogger.log()` 호출하여 기록. |

### To-do (구현/정의)

- 사용자별 목표 수 **실시간 집계 쿼리** 구현 → `ActiveGoalCountProvider` 구현체에서 DB 쿼리.
- 목표 생성 요청 시 **계산 로직에 추가** → `GoalCreateService.execute()` 1단계에서 `ActiveTaskCountService` 호출.
- 계산 결과로 **guide_exposed 여부 판단** → 2단계 `OverloadCheckService.is_overload(active_task_count)` 사용.
- **guide_exposed 이후 이벤트 로깅 정의** → `EventType`: `task_modify`, `app_close`, `task_complete`. 각 행동 발생 시점에 로그 기록.

---

## 3. Parameter Backlog 반영

### Success criteria (구현 완료)

| # | 기준 | 반영 위치 |
|---|------|-----------|
| 1 | **ActiveTaskCount Cap을 5로 설정** | `get_active_task_count_cap()` 기본값 5, 환경 변수 `ACTIVE_TASK_COUNT_CAP` |
| 2 | **guideExposureThreshold가 6일 때 목표 가이드 응답 반환** | `get_guide_exposure_threshold()` 기본값 6. `GoalCreateFlowResult.guide_message`에 `get_guide_message()` 반환값 설정 (예: "오늘 목표는 최대 5개가 적절해요") |

### To-do (구현 완료)

- ActiveTaskCount Cap **설정 파라미터 정의** → `settings.get_active_task_count_cap()`, `ACTIVE_TASK_COUNT_CAP`
- **guideExposureThreshold 설정값 로딩/적용** → `settings.get_guide_exposure_threshold()`, `GUIDE_EXPOSURE_THRESHOLD`
- **가이드 메시지/응답 정의** → `settings.get_guide_message()` (Cap 기반 문구). API 응답 시 `GoalCreateFlowResult.guide_message` 전달.

---

## 4. 코드 반영 요약

| 항목 | 파일/위치 |
|------|------------|
| Cap / threshold | `task_soft_limit/settings.py` |
| 가이드 메시지 | `get_guide_message()`, `GoalCreateFlowResult.guide_message` |
| guide_exposed payload | `next_task_ordinal` 추가 (`events/logging.py`) |
| 생성 흐름 | `GoalCreateService.execute()`, `execute_goal_create_flow()` |
| 이벤트 로그 | `GoalEventLog`, `EventType`, `GoalEventLogRepository` |

분석·측정 관련 To-do는 데이터 분석 파이프라인에서 이벤트 로그와 위 success criteria를 사용해 검증하면 된다.
