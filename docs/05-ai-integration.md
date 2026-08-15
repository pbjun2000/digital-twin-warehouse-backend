# 05. AI Integration

> LARO Backend는 AI가 생성한 계획을 그대로 실행하지 않고,  
> **현재 Warehouse 상태를 기반으로 Planning 요청을 구성하고 AI 결과를 검증·매핑한 뒤 실제 Simulation 실행 데이터로 연결**합니다.

---

## 1. AI Integration의 역할

LARO에서 Spring Boot와 FastAPI는 서로 다른 책임을 갖습니다.

```text
Spring Boot
서비스 데이터 · 권한 · 실행 상태
        ↓
FastAPI AI Planning
Mission · Constraints · Optimization
        ↓
Spring Boot
검증 · Mapping · Simulation 적용
```

AI Server는 어떤 작업을 어떤 조건으로 수행할지 계획하고,
Backend는 해당 계획이 현재 서비스 상태에서 실제로 실행 가능한지 확인합니다.

### Spring Boot

- Warehouse / Robot / Task 상태 관리
- 사용자 및 Warehouse 권한 검증
- AI Planning 요청 데이터 구성
- AI Plan 상태 검증
- AI Task와 Backend Task Mapping
- Simulation Playback 적용
- 실행·재계획 상태 관리

### AI Planning Server

- 사용자 요청 해석
- Rule / Agent 처리 경로 결정
- 현재 Warehouse Context 활용
- Mission 및 Constraint 구성
- 작업 배정 및 수행 순서 최적화
- MAPF 기반 Robot 이동 계획 생성
- Replanning Plan 생성

즉,

```text
AI
→ 계획 생성

Backend
→ 실제 서비스에서의 실행 가능성 검증 및 적용
```

으로 책임을 분리했습니다.

---

## 2. 전체 연동 흐름

Simulation 실행 요청 이후 AI Plan이 실제 로봇 실행 데이터로 적용되기까지의 흐름은 다음과 같습니다.

```text
Frontend
Simulation 실행 요청
        ↓
Spring Boot
Warehouse / Scenario / Task 확인
        ↓
Runtime State 조회
        ↓
AI Planning Request 구성
        ↓
FastAPI
Planning 수행
        ↓
SimulationPlan 반환
        ↓
Spring Boot
READY 상태 검증
        ↓
AI Task ID ↔ Backend Task ID Mapping
        ↓
Prepared Execution 구성
        ↓
SimulationPlaybackService
        ↓
Robot Plan / Step 실행
        ↓
Redis Runtime State
        ↓
WebSocket / STOMP
        ↓
Frontend
```

Frontend가 AI Server를 직접 호출하지 않고,
**Spring Boot가 AI와 서비스 사이의 Integration Boundary 역할**을 담당합니다.

---

## 3. AI Planning 요청 데이터 구성

AI에게 자연어 명령만 전달하면
현재 실제 Warehouse와 관계없는 계획이 생성될 수 있습니다.

따라서 Backend에서는 현재 서비스 데이터를 기반으로
Planning 요청에 필요한 Context를 구성합니다.

```text
User Command
      +
Structured Input
      +
Runtime Snapshot
      +
Warehouse Context
      ↓
AI Planning Request
```

### Structured Input

이번 Planning에서 실제로 처리해야 할 업무를 구조화합니다.

예를 들어 다음과 같은 정보가 Planning의 기준이 됩니다.

```text
Operation
Task
Item
Quantity
Source
Destination
Priority
```

AI는 전달된 실제 Operation을 기준으로 계획을 구성하며,
존재하지 않는 업무를 임의로 추가하는 방식으로 실행하지 않도록 제한합니다.

### Runtime Snapshot

현재 실행 중인 Robot과 Warehouse 상태를 전달합니다.

```text
Robot 위치
Robot 상태
Battery
현재 실행 Task
Simulation 상태
```

특히 Replanning에서는 초기 Warehouse 상태가 아니라
**현재 실행 시점의 상태**를 다시 기준으로 사용합니다.

---

## 4. Data-grounded Planning

LARO의 AI Planning은 LLM의 일반 지식만을 기반으로 계획하지 않습니다.

```text
PostgreSQL
영속 업무 데이터

Redis
현재 Robot Runtime State

Neo4j
Warehouse 이동 관계
        ↓
Current Warehouse Context
        ↓
AI Planning
```

실제 Warehouse 데이터와 Runtime State를 AI 판단의 Context로 사용하여

```text
존재하지 않는 Robot
존재하지 않는 Task
존재하지 않는 Node
연결되지 않은 이동 관계
```

등을 임의로 가정하는 것을 줄이는 방향으로 구성했습니다.

> AI가 업무 데이터의 Source of Truth가 되는 것이 아니라  
> **Backend가 제공한 실제 서비스 데이터를 Planning의 기준으로 사용합니다.**

---

## 5. Rule / Agent Routing

모든 요청을 동일하게 Agent로 처리하지 않습니다.

요청 특성에 따라 Rule 또는 Agent 경로를 선택할 수 있도록 구성했습니다.

```text
사용자 요청
      ↓
Routing
   ↙       ↘
Rule      Agent
```

### Rule Path

식별자와 조건이 명확한 정형 요청을 처리합니다.

```text
명확한 Task
명확한 Resource
구조화된 Operation
```

### Agent Path

추가적인 Context 조회와 해석이 필요한 요청을 처리합니다.

```text
자연어 조건
복수 조건
운영 우선순위
현재 상태를 함께 판단해야 하는 요청
```

Backend 입장에서는 어떤 경로를 사용했는지와 관계없이
최종적으로 실행 가능한 Planning 결과를 동일한 서비스 실행 흐름에 연결합니다.

---

## 6. LLM과 Solver의 책임 분리

AI Integration에서 중요하게 둔 원칙 중 하나는
LLM에게 작업 배정과 로봇 경로 계산까지 직접 맡기지 않는 것입니다.

```text
LLM / Agent
요청 해석
Mission · Constraints 구조화
        ↓
Optimization Solver
작업 배정 · 수행 순서 계산
        ↓
MAPF
다중 로봇 이동 계획
```

### LLM / Agent

```text
무엇을 수행해야 하는가?
어떤 조건을 고려해야 하는가?
현재 어떤 Resource를 사용할 수 있는가?
```

를 판단합니다.

### Optimization Solver

```text
어떤 Robot에 Task를 배정할 것인가?
어떤 순서로 작업을 수행할 것인가?
```

를 계산합니다.

Optimization Backend는 실행 환경에 따라
OR-Tools 또는 NVIDIA cuOpt를 사용할 수 있는 구조로 구성되어 있습니다.

### MAPF

Solver 결과를 기반으로
다수 Robot의 시간축 이동 계획을 구성합니다.

최종 Robot Plan은 다음 Step으로 표현됩니다.

```text
MOVE
WAIT
SERVICE
```

> **LLM은 판단하고, Solver는 계산하도록 역할을 분리했습니다.**

---

## 7. Constraint 구조

AI는 자유로운 텍스트 결과를 바로 Solver에 넘기지 않고,
최적화에 필요한 조건을 구조화합니다.

대표적으로 다음과 같은 범주의 Constraint를 다룹니다.

```text
Task Constraint
Fleet Constraint
Map Constraint
Objective
```

### Task Constraint

작업 자체의 조건을 정의합니다.

```text
Operation Type
Item
Quantity
Pickup Node
Delivery Node
Priority
Mandatory 여부
Robot 지정 조건
```

### Fleet Constraint

계획에서 사용할 Robot 범위를 정의합니다.

```text
사용 가능한 Robot
제외 Robot
Reserve Robot
Robot 상태
```

### Map Constraint

현재 Warehouse에서 이동 가능한 조건을 반영합니다.

```text
Blocked Edge
Avoid Edge
이동 가능 여부
대기 관련 조건
```

이러한 Constraint를 통해
LLM 결과를 바로 실행하는 대신 **Solver가 계산할 수 있는 명시적인 문제 구조**로 변환합니다.

---

## 8. AI Planning Result

FastAPI에서 반환되는 계획은
Robot별 실행 계획을 포함합니다.

개념적으로 다음과 같은 구조로 Backend에 전달됩니다.

```text
SimulationPlan
 ├─ Status
 ├─ Tasks
 └─ RobotPlans
       └─ PlanSteps
            ├─ MOVE
            ├─ WAIT
            └─ SERVICE
```

Backend는 이 결과를 수신했다고 바로 Simulation을 시작하지 않습니다.

먼저 Plan 상태와 서비스 데이터의 관계를 확인합니다.

---

## 9. READY Plan 검증

실행 대상은 AI Planning이 정상적으로 완료된 Plan으로 제한합니다.

```text
AI Plan 수신
      ↓
Status 확인
      ↓
READY ?
 ├─ YES → 실행 준비
 └─ NO  → 실행하지 않음
```

Backend 실행 경계에서 상태를 다시 확인함으로써
Planning이 완료되지 않은 Plan이나 검토가 필요한 결과가
Simulation에 바로 적용되지 않도록 합니다.

```text
AI의 Planning 완료
≠
서비스 실행 완료
```

두 상태를 별도로 관리합니다.

---

## 10. AI Task ID와 Backend Task ID Mapping

AI와 Backend는 Task를 서로 다른 목적으로 사용하기 때문에
AI Planning 결과의 Task ID를 그대로 서비스 실행 데이터로 사용할 수 없습니다.

따라서 Backend에서는 명시적인 Mapping을 생성합니다.

```text
AI Task ID
      ↓
aiTaskToBeTask
      ↓
Backend Task
      ↓
SimulationRun
```

이를 통해 Robot Plan에 포함된 AI Task가
실제 Backend의 어떤 Task를 의미하는지 연결합니다.

```text
AI Planning Domain
          ↓
       Mapping
          ↓
Backend Service Domain
```

이 Mapping이 완료된 이후에야
Robot Plan을 Simulation 실행 구조로 변환합니다.

---

## 11. Prepared Execution

READY 상태 확인과 Task Mapping 이후에는
Simulation Playback에 전달할 실행 데이터를 준비합니다.

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
```

이 단계에서 AI의 Planning 결과와
Backend의 실제 Task / SimulationRun 관계가 연결됩니다.

AI 결과를 서비스 계층에서 한 번 더 변환하는 이유는

```text
AI의 데이터 모델
```

과

```text
실제 서비스 실행 모델
```

을 직접 결합하지 않기 위해서입니다.

---

## 12. Simulation Playback 적용

일반 실행에서는 준비된 AI Plan을
`SimulationPlaybackService`에 전달하여 실행합니다.

```text
Prepared Execution
        ↓
SimulationPlaybackService.installAiPlan(...)
        ↓
Robot Plan
        ↓
Plan Step
        ↓
Runtime State 갱신
```

Playback 계층은 AI가 어떻게 계획을 생성했는지 알 필요 없이
최종적으로 준비된 Robot Plan과 Step을 실행하는 역할에 집중합니다.

이를 통해

```text
Planning
```

과

```text
Execution
```

의 책임을 분리했습니다.

---

## 13. Dynamic Replanning 연동

Replanning에서는 새로운 AI Plan을 받았다고
현재 Plan을 즉시 덮어쓰지 않습니다.

```text
현재 Plan 실행
      ↓
상태 변화 / 추가 요청
      ↓
REPLANNING
      ↓
현재 Runtime State 기반 AI 요청
      ↓
새로운 Plan 생성
      ↓
Plan Stage
      ↓
Activation
      ↓
새 Plan 실행
```

Backend에서는 일반 실행과 Replanning을 구분하여
새로운 AI Plan을 먼저 준비합니다.

재계획 결과는 기존 실행과 분리된 상태에서 Stage한 뒤
전환 가능한 시점에 적용합니다.

대표적인 실행 상태는 다음과 같습니다.

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

이를 통해 실행 중인 Robot의 상태와
새롭게 생성된 Plan의 상태가 뒤섞이지 않도록 관리합니다.

---

## 14. 오래된 Plan 적용 방지

AI Planning은 Backend 내부 처리보다 시간이 오래 걸릴 수 있기 때문에
요청 이후 Simulation 상태가 다시 변경될 가능성이 있습니다.

예를 들어,

```text
Execution Version 10
        ↓
AI Replanning 요청
        ↓
그 사이 다른 실행 변경
        ↓
Execution Version 11
        ↓
과거 Version 10 기준 AI 응답 도착
```

과 같은 상황을 고려해야 합니다.

Backend에서는 현재 실행 Version과
Planning 요청 당시의 실행 상태를 확인하여
오래된 결과가 최신 실행 상태를 덮어쓰지 않도록 관리합니다.

이 구조는 특히 Replanning이나 Human Review처럼
Planning과 실제 적용 사이에 시간이 발생하는 흐름에서 중요합니다.

---

## 15. Human Review

AI가 자동으로 실행하기 어려운 요청은
Human Review 흐름으로 전환할 수 있도록 구성되어 있습니다.

```text
사용자 요청
      ↓
AI Planning
      ↓
자동 판단 가능?
   ↙          ↘
 YES          NO
 ↓             ↓
Plan      Human Review
               ↓
          사용자 확인
               ↓
          Planning 재개
```

Human Review 역시 AI Server 내부에서 끝나는 것이 아니라
Backend의 현재 Simulation 상태와 연결되어야 합니다.

따라서 검토 결과가 돌아왔을 때도
현재 실행 상태와의 일치 여부를 확인한 뒤
Planning 흐름을 이어가도록 구성합니다.

---

## 16. 실패를 서비스 실행과 분리

AI 호출 실패나 Planning 실패가
현재 서비스 데이터 자체를 손상시키지 않도록
Planning과 실행을 분리했습니다.

```text
AI 요청 실패
      ↓
현재 Backend 데이터 유지

Planning 실패
      ↓
현재 Simulation Plan 유지

READY Plan
      ↓
검증 성공 시에만 실행
```

Backend가 실행의 최종 경계를 담당하기 때문에
AI 응답이 존재한다는 사실만으로
Robot 실행 상태가 변경되지 않습니다.

---

## 17. 주요 Backend Component

| Component | 역할 |
|---|---|
| `SimulationRunService` | AI Planning 요청 및 실행 가능 상태 검증 |
| `SimulationPlaybackService` | AI Plan을 실제 Robot 실행 상태로 적용 |
| `PreparedExecution` | AI 결과를 Backend 실행 데이터로 준비 |
| `aiTaskToBeTask` Mapping | AI Task와 Backend Task 연결 |

핵심은 AI Client 호출 자체가 아니라
**AI 결과를 서비스 실행 모델과 안전하게 연결하는 과정**입니다.

---

## 18. 설계 결과

최종 Integration 구조는 다음과 같습니다.

```text
Backend Data
Warehouse · Robot · Task
        ↓
AI Planning Request
        ↓
Rule / Agent
        ↓
Mission · Constraints
        ↓
Solver / MAPF
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

이를 통해 다음 기준을 유지했습니다.

- AI Planning의 기준 데이터를 Backend에서 제공
- LLM과 Solver의 책임 분리
- AI Plan 상태를 실행 경계에서 재검증
- AI Task와 실제 Backend Task를 명시적으로 Mapping
- Planning과 Simulation Execution 책임 분리
- Replanning 시 현재 실행 상태를 유지하며 새로운 Plan 준비
- 오래된 AI 결과가 최신 실행 상태를 덮어쓰지 않도록 관리

> LARO의 AI Integration에서 중요하게 둔 것은  
> **AI API를 호출하는 것 자체가 아니라, AI의 비결정적인 Planning 영역과 실제 서비스의 결정적인 실행 영역 사이에 명확한 경계를 두는 것**입니다.

---

## 다음 문서

- [06. Multi-user Isolation](./06-guest-access.md)

---

[← 04. Digital Twin Graph](./04-digital-twin-graph.md) · [README](../README.md)
