# TaskSoftLimit — ActiveTaskCount 기반 Soft 제한 실험 인프라

목표를 5개 초과하여 생성하려는 상황에서 **생성은 허용하되 가이드만 노출**하고, 이후 행동 변화를 추적할 수 있는 도메인 로직 및 이벤트 구조를 제공합니다.

## 문서

| 문서 | 설명 |
|------|------|
| [docs/문제정의서.md](docs/문제정의서.md) | 문제 정의, 지표, 제약 조건 |
| [docs/해결방안탐색문서.md](docs/해결방안탐색문서.md) | Soft/Hard 제한, 이벤트 정의, 실험 구조 |
| [docs/PRD.md](docs/PRD.md) | 제품 요구사항 정의 |
| [docs/MVP.md](docs/MVP.md) | MVP 범위 및 인수 조건 |
| [docs/설계서.md](docs/설계서.md) | 도메인 모델, 이벤트, ActiveTaskCount, 흐름, 패키지 구조 |

## 핵심 도메인 규칙

- 목표 생성 요청 시점에 **ActiveTaskCount** 계산 (당일·활성 목표 수)
- **ActiveTaskCount ≥ 6**이면 과부하 → **guide_exposed** 이벤트 기록
- 목표 생성은 **차단하지 않음**
- 실험 단위: **"6번째 목표 생성 시도 첫 발생"**

## 패키지 구조

```
TaskSoftLimit/
├── docs/                    # 문제정의서, 해결방안, PRD, MVP, 설계서
├── task_soft_limit/
│   ├── domain/              # GoalCreateFlowResult 등
│   ├── events/              # EventType, GoalEventLogger, log_guide_exposed
│   ├── policy/              # is_overload (과부하 판별)
│   ├── service/             # execute_goal_create_flow
│   └── settings.py          # threshold, cap 설정
└── tests/
```

## 사용 예시

```python
from task_soft_limit import execute_goal_create_flow, GoalCreateFlowResult

# 호출 측에서 active_task_count 계산 (당일·활성 목표 수)
active_task_count = goal_repository.count_active_today(user_id)

# 이벤트 로거 구현체 (본 패키지는 인터페이스만 정의)
event_logger = MyGoalEventLogRepository()

result = execute_goal_create_flow(
    user_id=user_id,
    active_task_count=active_task_count,
    event_logger=event_logger,
)

# 생성은 항상 진행. guide_exposed 여부만 반환 (헤더/메타 등에 활용)
if result.guide_exposed:
    # 가이드 노출됨 (UI는 별도, 본 패키지에서는 UX 미구현)
    pass
# 목표 저장 후 task_create 이벤트 기록은 호출 측에서 수행
```

## 설정

| 환경 변수 | 기본값 | 설명 |
|-----------|--------|------|
| `GUIDE_EXPOSURE_THRESHOLD` | 6 | 이 값 이상이면 guide_exposed 기록 |
| `ACTIVE_TASK_COUNT_CAP` | 5 | 권장 활성 목표 수 상한 |

## 테스트

```bash
# 프로젝트 루트에서
pip install pytest
$env:PYTHONPATH = "TaskSoftLimit"
pytest TaskSoftLimit/tests/ -v
```
