# 05. AI Integration

> LARO Backend는 현재 Warehouse 상태를 AI Planning Server에 전달하고,  
> 반환된 Plan을 검증한 뒤 실제 Task와 Simulation 실행 데이터로 연결합니다.

---

## 1. Backend와 AI의 역할

LARO에서는 Planning과 실제 서비스 실행을 분리했습니다.

```text
Spring Boot
Warehouse · Robot · Task · 실행 상태
        ↓
FastAPI AI Planning
Mission · Constraints · Robot Plan
        ↓
Spring Boot
Plan 검증 · Task 연결 · Simulation 적용
```

AI Server는 현재 상황에서 **어떤 작업을 어떻게 수행할지 계획**하고,
Backend는 해당 결과를 **실제 서비스 데이터와 연결하여 실행**합니다.

| 영역 | 역할 |
|---|---|
| **Spring Boot** | 서비스 데이터 관리, Planning 요청 구성, 결과 검증, Simulation 적용 |
| **AI Planning** | 요청 해석, Mission·Constraint 구성, 계획 및 재계획 |
| **Solver / MAPF** | 작업 배정·수행 순서 및 다중 로봇 이동 계획 계산 |

Frontend가 AI Server를 직접 호출하지 않고
Spring Boot가 서비스와 AI 사이의 연동 지점 역할을 담당합니다.

---

## 2. Planning 요청 구성

AI에게 자연어 요청만 전달하면
현재 Warehouse 상태와 맞지 않는 계획이 만들어질 수 있습니다.

따라서 Backend에서는 Planning에 필요한 실제 서비스 데이터를 함께 구성합니다.

```text
사용자 요청
     +
Warehouse / Task 정보
     +
현재 Robot 상태
     ↓
AI Planning Request
```

요청에는 이번 실행에서 처리해야 하는 Operation과
현재 Warehouse 및 Robot 상태가 포함됩니다.

특히 Replanning에서는 초기 상태가 아니라
**현재 실행 시점의 상태를 다시 기준으로 사용**합니다.

```text
PostgreSQL
Warehouse · Task 등 영속 데이터

Redis
Robot Runtime State

Warehouse Graph
Node · Edge 이동 구조
        ↓
Planning Context
```

AI가 서비스 데이터의 원본이 되는 것이 아니라,
Backend가 관리하는 현재 상태를 Planning의 기준으로 제공합니다.

---

## 3. AI Planning 내부 처리

Planning Server에서는 요청의 형태에 따라
Rule 또는 Agent 경로를 선택할 수 있습니다.

```text
Planning Request
       ↓
 Rule / Agent
       ↓
Mission · Constraints
       ↓
Optimization Solver
       ↓
MAPF
       ↓
Simulation Plan
```

LLM / Agent는 자연어 요청과 운영 조건을 구조화하고,
실제 작업 배정과 수행 순서는 Optimization Solver가 계산합니다.

다중 로봇의 이동 계획은 MAPF를 통해 구성하며,
최종 실행 Step은 다음 형태로 Backend에 전달됩니다.

```text
MOVE
WAIT
SERVICE
```

즉 LLM이 직접 Robot의 전체 이동 경로를 결정하기보다
**판단 영역과 계산 영역을 분리한 결과를 Backend가 전달받는 구조**입니다.

---

## 4. AI Plan 검증 및 Task Mapping

AI Plan을 수신했다고 바로 Simulation에 적용하지 않습니다.

Backend에서 먼저 실행 가능한 Plan인지 확인합니다.

```text
AI Plan
   ↓
READY 상태 확인
   ↓
AI Task ↔ Backend Task Mapping
   ↓
실행 데이터 준비
```

### READY 상태 확인

실행 대상은 Planning이 정상적으로 완료된 Plan으로 제한합니다.

```text
Plan Status
   ↓
READY ?
 ├─ YES → 다음 단계
 └─ NO  → 실행하지 않음
```

이를 통해 Planning 완료 상태와
실제 서비스 실행 상태를 별도로 관리합니다.

### Task ID Mapping

AI와 Backend는 Task를 서로 다른 실행 모델에서 사용하기 때문에
AI가 반환한 Task 식별자를 그대로 사용할 수 없습니다.

Backend에서는 다음 Mapping을 구성합니다.

```text
AI Task ID
     ↓
aiTaskToBeTask
     ↓
Backend Task
```

Robot Plan에 포함된 AI Task가
실제 Backend의 어떤 Task인지 연결한 뒤 Simulation에 적용합니다.

---

## 5. Simulation Playback 연결

Plan 검증과 Task Mapping이 끝나면
AI 결과를 Backend의 실행 구조로 변환합니다.

```text
SimulationPlan
      ↓
READY Validation
      ↓
Task Mapping
      ↓
Prepared Execution
      ↓
SimulationPlaybackService
      ↓
Robot Plan / Step 실행
```

`SimulationPlaybackService`는
AI가 어떤 방식으로 계획을 생성했는지 알 필요 없이
준비된 Robot Plan을 실행 상태에 반영하는 역할을 담당합니다.

이 구조를 통해

```text
Planning
AI가 실행 계획을 생성

Execution
Backend가 실제 서비스 상태에 적용
```

두 영역의 책임을 분리했습니다.

---

## 6. Dynamic Replanning

실행 중 새로운 운영 조건이 발생하면
현재 Simulation을 기준으로 다시 Planning을 요청합니다.

새로운 Plan이 만들어졌다고 기존 실행을 즉시 덮어쓰지 않고,
Backend에서 재계획 상태를 별도로 관리합니다.

```text
RUNNING
   ↓
QUIESCING
   ↓
REPLANNING
   ↓
PENDING_ACTIVATION
   ↓
RUNNING
```

전체 흐름은 다음과 같습니다.

```text
현재 Simulation 실행
        ↓
재계획 요청
        ↓
현재 Runtime State 구성
        ↓
AI Replanning
        ↓
새 Plan 수신
        ↓
검증 및 Stage
        ↓
Activation
        ↓
새 Plan 실행
```

현재 Plan과 새 Plan 사이에 전환 단계를 두어
재계획 중인 결과가 기존 실행 상태와 바로 섞이지 않도록 했습니다.

---

## 7. 오래된 Planning 결과 방지

AI Planning은 즉시 완료되지 않을 수 있기 때문에
요청을 보낸 이후 Simulation 상태가 다시 변경될 가능성이 있습니다.

예를 들어,

```text
Execution Version 10
        ↓
Replanning 요청
        ↓
실행 상태 변경
        ↓
Execution Version 11
        ↓
Version 10 기준 AI 응답 도착
```

과 같은 상황이 발생할 수 있습니다.

이때 이전 상태를 기준으로 생성된 Plan을 그대로 적용하면
최신 실행 상태를 과거 결과가 덮어쓸 수 있습니다.

Backend에서는 실행 Version을 기준으로
Planning 요청 시점과 현재 실행 상태를 구분하여
오래된 결과가 그대로 적용되지 않도록 관리합니다.

---

## 8. 주요 Component

| Component | 역할 |
|---|---|
| `SimulationRunService` | Simulation 실행 및 AI Planning 연동 |
| `SimulationPlaybackService` | AI Plan을 Robot 실행 상태로 적용 |
| `PreparedExecution` | AI 결과를 Backend 실행 데이터로 구성 |
| `aiTaskToBeTask` | AI Task와 Backend Task Mapping |

전체 연동 흐름은 다음과 같습니다.

```text
Backend State
      ↓
AI Planning
      ↓
SimulationPlan
      ↓
READY Validation
      ↓
Task Mapping
      ↓
Prepared Execution
      ↓
Simulation Playback
      ↓
Runtime State
```

AI API를 단순 호출하는 데 그치지 않고,
**AI의 Planning 결과와 실제 서비스 실행 사이에 검증과 Mapping 단계를 둔 것**을
Backend 연동의 핵심으로 두었습니다.

---

## 관련 문서

- [01. Project Overview](./01-project-overview.md)
- [02. Backend Design](./02-backend-design.md)
- [03. Warehouse Domain](./03-warehouse-domain.md)
- [04. Digital Twin Graph](./04-digital-twin-graph.md)
- [06. Multi-user Isolation](./06-multi-user-isolation.md)

---

[← 04. Digital Twin Graph](./04-digital-twin-graph.md) · [README](../README.md)

