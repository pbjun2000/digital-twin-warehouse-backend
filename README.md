# LARO

> **Digital Twin 기반 자율 창고 운영 및 다중 로봇 작업 최적화 시스템**

LARO(LLM Autonomous Robot Orchestration)는  
창고·재고·로봇의 현재 상태를 기반으로 AI가 작업 계획을 생성하고,
다중 로봇의 작업 배정·경로 최적화·실시간 실행·재계획까지 연결하는 창고 운영 시스템입니다.

단순한 로봇 경로 탐색이 아니라,

**Digital Twin → AI Planning → 작업 최적화 → 다중 로봇 이동 계획 → 실시간 관제 → Dynamic Replanning**

의 전체 운영 흐름을 구현했습니다.

> 본 저장소는 팀 프로젝트에서 제가 담당한 **Backend 설계·구현 및 문제 해결 과정**을 중심으로 정리한 개인 포트폴리오입니다.

---

## 프로젝트 개요

물류센터에 다수의 AGV·AMR이 도입되면서
개별 로봇의 이동보다 **여러 로봇의 작업과 이동을 동시에 조율하는 문제**가 중요해지고 있습니다.

실제 운영 중에는 신규 작업, 배터리 변화, 통로 차단 등으로
초기에 생성한 계획의 유효성이 계속 달라질 수 있습니다.

LARO는 현재 창고 상태를 기준으로 AI가 실행 계획을 구성하고,
최적화 Solver와 MAPF를 통해 작업 배정과 이동 계획을 계산한 뒤
Simulation에서 이를 실행·관제하고 상태 변화 발생 시 재계획합니다.

```text
Warehouse State
      ↓
AI Planning
      ↓
Task Assignment / Optimization
      ↓
MAPF
      ↓
Simulation
      ↓
Real-time Monitoring
      ↓
Dynamic Replanning
```

---

## 주요 기능

### Digital Twin 창고 구성

- `Warehouse · Zone · Node · Edge` 기반 창고 구조 관리
- Storage, Charging Station, Robot, Inventory 데이터 연계
- 구성한 Warehouse Graph를 AI Planning과 Simulation에서 동일하게 활용

### AI 기반 작업 계획

- 자연어 요청과 실제 창고 상태를 기반으로 Mission 구성
- 요청 특성에 따른 `Rule / Agent` 처리 경로 분리
- LLM이 직접 경로를 생성하지 않고 Solver 입력에 필요한 조건을 구조화

### 다중 로봇 최적화

```text
LLM / Agent
Mission · Constraints
        ↓
Optimization Solver
작업 배정 · 방문 순서
        ↓
MAPF
충돌을 고려한 이동 계획
        ↓
MOVE · WAIT · SERVICE
```

> **LLM은 판단하고, Solver는 계산하도록 역할을 분리했습니다.**

### 실시간 Simulation

- AI Plan 검증 후 Backend 실행 데이터로 변환
- Robot별 Plan / Step 기반 Simulation Playback
- Redis 기반 Runtime State 관리
- WebSocket / STOMP 기반 실시간 로봇 상태 전달

### Dynamic Replanning

실행 중 신규 작업이나 상태 변화가 발생하면
현재 실행 상태를 유지하면서 새로운 계획을 생성하고 전환합니다.

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

---

## 시스템 아키텍처

```text
┌─────────────────────────────┐
│        React Frontend       │
└──────────────┬──────────────┘
               │ REST / STOMP
               ▼
┌─────────────────────────────┐
│         Spring Boot         │
│                             │
│ Auth · Warehouse · Robot    │
│ Task · Simulation · AI 연동 │
└───────┬───────────┬─────────┘
        │           │
        │           └──────→ Redis
        │                   Runtime State
        ▼
     FastAPI
        │
        ▼
 GPT-5-mini / LangGraph
        │
        ▼
 Optimization Solver
 cuOpt / OR-Tools
        │
        ▼
       MAPF


PostgreSQL
    │
    │ AFTER_COMMIT
    ▼
  Neo4j
Warehouse Graph
```

### Data Store 역할

| 저장소 | 역할 |
|---|---|
| **PostgreSQL** | Warehouse, Robot, Inventory, Task, Scenario 등 영속 데이터 |
| **Redis** | 로봇 위치·배터리·실행 상태 등 Runtime State |
| **Neo4j** | Node·Edge와 창고 객체 간 이동·접근 관계 |

---

# 담당 역할 및 기여

팀 내 Backend 개발을 담당하며  
**창고·로봇·그래프 및 AI 실행 환경 연동 영역**을 구현했습니다.

- Warehouse / Zone / Node / Edge API 설계 및 구현
- Robot 및 Warehouse Layout 데이터 연동
- USER / GUEST별 독립적인 Simulation 실행 환경 설계
- Shared Warehouse → Personal Warehouse Deep Clone 구현
- Warehouse Resource 소유권 검증 및 접근 제어
- PostgreSQL → Neo4j Warehouse Graph Sync 구현
- AI Plan과 Backend 실행 데이터 연동
- Backend / AI / Frontend 통합 작업 참여

특히 단순 CRUD보다

**다중 사용자 실행 상태 격리**,  
**PostgreSQL·Neo4j 데이터 일관성**,  
**AI 결과를 실제 Simulation 실행 데이터로 연결하는 과정**

을 주요 Backend 문제로 다뤘습니다.

---

# 핵심 구현

## 1. 사용자별 Digital Twin 실행 환경 격리

### 문제

초기 구조에서는 여러 사용자가 하나의 Shared Warehouse를 사용했습니다.

```text
            Shared Warehouse
          ↙       ↓       ↘
      User A   User B    Guest

      동일 Inventory / Robot State
```

동시에 Simulation을 실행하면 한 사용자의 재고·Robot 상태 변화가
다른 사용자의 실행 환경에 영향을 줄 수 있었습니다.

### 해결

Shared Warehouse를 직접 실행하지 않고
사용자별 Personal Warehouse를 생성하도록 구조를 변경했습니다.

```text
Shared Template
      │
      ├──→ User A → Personal Warehouse A
      │
      ├──→ User B → Personal Warehouse B
      │
      └──→ Guest  → Personal Warehouse C
```

Personal Warehouse 생성 시 다음 실행 데이터를 함께 Deep Clone합니다.

```text
Warehouse
Zone
Node
Edge
ChargingStation
StorageLocation
WarehouseItem
Robot
Scenario
```

USER는 `user_id`,
GUEST는 `guest_session_id`를 기준으로 소유권을 검증합니다.

이를 통해 사용자별 Inventory·Robot·Simulation 상태를 독립적으로 유지하고,
다른 사용자의 `warehouseId`를 직접 요청하는 접근도 차단했습니다.

---

## 2. PostgreSQL과 Neo4j 데이터 일관성

### 문제

Warehouse 데이터를 PostgreSQL에 저장한 뒤
Neo4j Warehouse Graph를 별도로 생성하는 구조에서는

PostgreSQL Transaction과 Graph Sync 처리 시점에 따라
두 저장소의 데이터가 서로 달라질 가능성이 있었습니다.

### 해결

PostgreSQL을 기준 데이터로 두고,
Transaction이 정상적으로 Commit된 이후에만 Graph Sync를 수행했습니다.

```text
Warehouse 데이터 변경
        ↓
PostgreSQL Transaction
        ↓
COMMIT
        ↓
WarehouseGraphChangedEvent
        ↓
AFTER_COMMIT Listener
        ↓
GraphSyncService
        ↓
Neo4j Warehouse Graph Sync
```

```java
@TransactionalEventListener(
    phase = TransactionPhase.AFTER_COMMIT
)
```

이를 통해 **PostgreSQL을 Source of Truth로 유지하면서
Warehouse 단위 Graph 데이터의 일관성을 관리**했습니다.

---

## 3. AI Plan과 Backend 실행 데이터 연결

AI가 반환한 계획을 그대로 Simulation에 적용하지 않고,
Backend에서 실행 가능한 상태인지 검증한 뒤 서비스 데이터와 연결합니다.

```text
FastAPI AI Plan
       ↓
READY 상태 확인
       ↓
AI Task ID
       ↓
Backend Task ID Mapping
       ↓
Robot Plan / Step
       ↓
Simulation Playback
```

AI 영역과 서비스 영역이 서로 다른 Task 식별자를 사용하기 때문에
AI Task와 실제 Backend Task 사이의 Mapping을 구성했습니다.

이를 통해 AI가 생성한 계획을
Backend의 Task·SimulationRun·Robot 실행 상태와 연결하여
Simulation Playback에 적용했습니다.

---

# Troubleshooting

## 다중 사용자 Simulation 상태 충돌

**원인**

Shared Warehouse의 Inventory와 Robot 상태를 여러 사용자가 함께 사용했습니다.

**해결**

- Shared Warehouse를 Template으로 변경
- USER / GUEST별 Personal Warehouse Deep Clone
- Resource Ownership Validation 적용

**결과**

각 사용자가 독립적인 Warehouse 상태에서 Simulation을 실행하도록 개선했습니다.

---

## DB와 Graph 상태 불일치

**원인**

PostgreSQL 데이터 변경과 Neo4j Graph Sync의 처리 시점이 분리되어 있었습니다.

**해결**

`WarehouseGraphChangedEvent`와 `AFTER_COMMIT Listener`를 적용하여
PostgreSQL Commit 이후에만 Graph Sync를 수행하도록 변경했습니다.

---

## Backend와 AI의 Replanning 책임 중복

**원인**

초기에는 Backend에도 재계획 로직이 존재했지만,
AI Planning 기능이 확장되면서 Backend와 AI의 책임이 중복되었습니다.

**해결**

```text
AI
→ 계획 생성 · 최적화 · 재계획

Backend
→ 요청 검증 · 상태 관리 · Plan 적용

Frontend
→ 실행 상태 시각화
```

서비스별 책임을 다시 정의하고
중복된 Backend 재계획 로직을 제거했습니다.

---

# Tech Stack

### Backend

`Java 17` `Spring Boot` `Spring Data JPA`  
`Spring Security` `WebSocket / STOMP`

### Database

`PostgreSQL` `Redis` `Neo4j`

### AI / Optimization

`FastAPI` `GPT-5-mini` `LangGraph`  
`NVIDIA cuOpt` `OR-Tools` `MAPF`

### Frontend

`React` `Vite` `JavaScript`

### Infra

`AWS` `Docker` `ECR` `ECS` `S3` `CloudFront`

---

# Documentation

상세 설계와 개발 과정은 별도 문서로 정리했습니다.

| 문서 | 내용 |
|---|---|
| [01. 프로젝트 개요](./docs/01-project-overview.md) | 프로젝트 목표 및 역할 |
| [02. Backend 설계](./docs/02-backend-design.md) | Backend 구조 및 설계 |
| [03. Warehouse Domain](./docs/03-warehouse-domain.md) | Warehouse·Zone·Node·Edge |
| [04. Digital Twin Graph](./docs/04-digital-twin-graph.md) | PostgreSQL·Neo4j 연동 |
| [05. AI Integration](./docs/05-ai-integration.md) | AI Planning 연동 |
| [06. Multi-user Isolation](./docs/06-guest-access.md) | USER/GUEST 실행 환경 격리 |

---

## Repository

**Portfolio**  
https://github.com/pbjun2000/digital-twin-warehouse-backend

**Team Backend**  
https://github.com/kt-aivle-big-project/BE# LARO

> **Digital Twin 기반 자율 창고 운영 및 다중 로봇 작업 최적화 시스템**

LARO(LLM Autonomous Robot Orchestration)는  
창고·재고·로봇의 현재 상태를 기반으로 AI가 작업 계획을 생성하고,
다중 로봇의 작업 배정·경로 최적화·실시간 실행·재계획까지 연결하는 창고 운영 시스템입니다.

단순한 로봇 경로 탐색이 아니라,

**Digital Twin → AI Planning → 작업 최적화 → 다중 로봇 이동 계획 → 실시간 관제 → Dynamic Replanning**

의 전체 운영 흐름을 구현했습니다.

> 본 저장소는 팀 프로젝트에서 제가 담당한 **Backend 설계·구현 및 문제 해결 과정**을 중심으로 정리한 개인 포트폴리오입니다.

---

## 프로젝트 개요

물류센터에 다수의 AGV·AMR이 도입되면서
개별 로봇의 이동보다 **여러 로봇의 작업과 이동을 동시에 조율하는 문제**가 중요해지고 있습니다.

실제 운영 중에는 신규 작업, 배터리 변화, 통로 차단 등으로
초기에 생성한 계획의 유효성이 계속 달라질 수 있습니다.

LARO는 현재 창고 상태를 기준으로 AI가 실행 계획을 구성하고,
최적화 Solver와 MAPF를 통해 작업 배정과 이동 계획을 계산한 뒤
Simulation에서 이를 실행·관제하고 상태 변화 발생 시 재계획합니다.

```text
Warehouse State
      ↓
AI Planning
      ↓
Task Assignment / Optimization
      ↓
MAPF
      ↓
Simulation
      ↓
Real-time Monitoring
      ↓
Dynamic Replanning
```

---

## 주요 기능

### Digital Twin 창고 구성

- `Warehouse · Zone · Node · Edge` 기반 창고 구조 관리
- Storage, Charging Station, Robot, Inventory 데이터 연계
- 구성한 Warehouse Graph를 AI Planning과 Simulation에서 동일하게 활용

### AI 기반 작업 계획

- 자연어 요청과 실제 창고 상태를 기반으로 Mission 구성
- 요청 특성에 따른 `Rule / Agent` 처리 경로 분리
- LLM이 직접 경로를 생성하지 않고 Solver 입력에 필요한 조건을 구조화

### 다중 로봇 최적화

```text
LLM / Agent
Mission · Constraints
        ↓
Optimization Solver
작업 배정 · 방문 순서
        ↓
MAPF
충돌을 고려한 이동 계획
        ↓
MOVE · WAIT · SERVICE
```

> **LLM은 판단하고, Solver는 계산하도록 역할을 분리했습니다.**

### 실시간 Simulation

- AI Plan 검증 후 Backend 실행 데이터로 변환
- Robot별 Plan / Step 기반 Simulation Playback
- Redis 기반 Runtime State 관리
- WebSocket / STOMP 기반 실시간 로봇 상태 전달

### Dynamic Replanning

실행 중 신규 작업이나 상태 변화가 발생하면
현재 실행 상태를 유지하면서 새로운 계획을 생성하고 전환합니다.

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

---

## 시스템 아키텍처

```text
┌─────────────────────────────┐
│        React Frontend       │
└──────────────┬──────────────┘
               │ REST / STOMP
               ▼
┌─────────────────────────────┐
│         Spring Boot         │
│                             │
│ Auth · Warehouse · Robot    │
│ Task · Simulation · AI 연동 │
└───────┬───────────┬─────────┘
        │           │
        │           └──────→ Redis
        │                   Runtime State
        ▼
     FastAPI
        │
        ▼
 GPT-5-mini / LangGraph
        │
        ▼
 Optimization Solver
 cuOpt / OR-Tools
        │
        ▼
       MAPF


PostgreSQL
    │
    │ AFTER_COMMIT
    ▼
  Neo4j
Warehouse Graph
```

### Data Store 역할

| 저장소 | 역할 |
|---|---|
| **PostgreSQL** | Warehouse, Robot, Inventory, Task, Scenario 등 영속 데이터 |
| **Redis** | 로봇 위치·배터리·실행 상태 등 Runtime State |
| **Neo4j** | Node·Edge와 창고 객체 간 이동·접근 관계 |

---

# 담당 역할 및 기여

팀 내 Backend 개발을 담당하며  
**창고·로봇·그래프 및 AI 실행 환경 연동 영역**을 구현했습니다.

- Warehouse / Zone / Node / Edge API 설계 및 구현
- Robot 및 Warehouse Layout 데이터 연동
- USER / GUEST별 독립적인 Simulation 실행 환경 설계
- Shared Warehouse → Personal Warehouse Deep Clone 구현
- Warehouse Resource 소유권 검증 및 접근 제어
- PostgreSQL → Neo4j Warehouse Graph Sync 구현
- AI Plan과 Backend 실행 데이터 연동
- Backend / AI / Frontend 통합 작업 참여

특히 단순 CRUD보다

**다중 사용자 실행 상태 격리**,  
**PostgreSQL·Neo4j 데이터 일관성**,  
**AI 결과를 실제 Simulation 실행 데이터로 연결하는 과정**

을 주요 Backend 문제로 다뤘습니다.

---

# 핵심 구현

## 1. 사용자별 Digital Twin 실행 환경 격리

### 문제

초기 구조에서는 여러 사용자가 하나의 Shared Warehouse를 사용했습니다.

```text
            Shared Warehouse
          ↙       ↓       ↘
      User A   User B    Guest

      동일 Inventory / Robot State
```

동시에 Simulation을 실행하면 한 사용자의 재고·Robot 상태 변화가
다른 사용자의 실행 환경에 영향을 줄 수 있었습니다.

### 해결

Shared Warehouse를 직접 실행하지 않고
사용자별 Personal Warehouse를 생성하도록 구조를 변경했습니다.

```text
Shared Template
      │
      ├──→ User A → Personal Warehouse A
      │
      ├──→ User B → Personal Warehouse B
      │
      └──→ Guest  → Personal Warehouse C
```

Personal Warehouse 생성 시 다음 실행 데이터를 함께 Deep Clone합니다.

```text
Warehouse
Zone
Node
Edge
ChargingStation
StorageLocation
WarehouseItem
Robot
Scenario
```

USER는 `user_id`,
GUEST는 `guest_session_id`를 기준으로 소유권을 검증합니다.

이를 통해 사용자별 Inventory·Robot·Simulation 상태를 독립적으로 유지하고,
다른 사용자의 `warehouseId`를 직접 요청하는 접근도 차단했습니다.

---

## 2. PostgreSQL과 Neo4j 데이터 일관성

### 문제

Warehouse 데이터를 PostgreSQL에 저장한 뒤
Neo4j Warehouse Graph를 별도로 생성하는 구조에서는

PostgreSQL Transaction과 Graph Sync 처리 시점에 따라
두 저장소의 데이터가 서로 달라질 가능성이 있었습니다.

### 해결

PostgreSQL을 기준 데이터로 두고,
Transaction이 정상적으로 Commit된 이후에만 Graph Sync를 수행했습니다.

```text
Warehouse 데이터 변경
        ↓
PostgreSQL Transaction
        ↓
COMMIT
        ↓
WarehouseGraphChangedEvent
        ↓
AFTER_COMMIT Listener
        ↓
GraphSyncService
        ↓
Neo4j Warehouse Graph Sync
```

```java
@TransactionalEventListener(
    phase = TransactionPhase.AFTER_COMMIT
)
```

이를 통해 **PostgreSQL을 Source of Truth로 유지하면서
Warehouse 단위 Graph 데이터의 일관성을 관리**했습니다.

---

## 3. AI Plan과 Backend 실행 데이터 연결

AI가 반환한 계획을 그대로 Simulation에 적용하지 않고,
Backend에서 실행 가능한 상태인지 검증한 뒤 서비스 데이터와 연결합니다.

```text
FastAPI AI Plan
       ↓
READY 상태 확인
       ↓
AI Task ID
       ↓
Backend Task ID Mapping
       ↓
Robot Plan / Step
       ↓
Simulation Playback
```

AI 영역과 서비스 영역이 서로 다른 Task 식별자를 사용하기 때문에
AI Task와 실제 Backend Task 사이의 Mapping을 구성했습니다.

이를 통해 AI가 생성한 계획을
Backend의 Task·SimulationRun·Robot 실행 상태와 연결하여
Simulation Playback에 적용했습니다.

---

# Troubleshooting

## 다중 사용자 Simulation 상태 충돌

**원인**

Shared Warehouse의 Inventory와 Robot 상태를 여러 사용자가 함께 사용했습니다.

**해결**

- Shared Warehouse를 Template으로 변경
- USER / GUEST별 Personal Warehouse Deep Clone
- Resource Ownership Validation 적용

**결과**

각 사용자가 독립적인 Warehouse 상태에서 Simulation을 실행하도록 개선했습니다.

---

## DB와 Graph 상태 불일치

**원인**

PostgreSQL 데이터 변경과 Neo4j Graph Sync의 처리 시점이 분리되어 있었습니다.

**해결**

`WarehouseGraphChangedEvent`와 `AFTER_COMMIT Listener`를 적용하여
PostgreSQL Commit 이후에만 Graph Sync를 수행하도록 변경했습니다.

---

## Backend와 AI의 Replanning 책임 중복

**원인**

초기에는 Backend에도 재계획 로직이 존재했지만,
AI Planning 기능이 확장되면서 Backend와 AI의 책임이 중복되었습니다.

**해결**

```text
AI
→ 계획 생성 · 최적화 · 재계획

Backend
→ 요청 검증 · 상태 관리 · Plan 적용

Frontend
→ 실행 상태 시각화
```

서비스별 책임을 다시 정의하고
중복된 Backend 재계획 로직을 제거했습니다.

---

# Tech Stack

### Backend

`Java 17` `Spring Boot` `Spring Data JPA`  
`Spring Security` `WebSocket / STOMP`

### Database

`PostgreSQL` `Redis` `Neo4j`

### AI / Optimization

`FastAPI` `GPT-5-mini` `LangGraph`  
`NVIDIA cuOpt` `OR-Tools` `MAPF`

### Frontend

`React` `Vite` `JavaScript`

### Infra

`AWS` `Docker` `ECR` `ECS` `S3` `CloudFront`

---

# Documentation

상세 설계와 개발 과정은 별도 문서로 정리했습니다.

| 문서 | 내용 |
|---|---|
| [01. 프로젝트 개요](./docs/01-project-overview.md) | 프로젝트 목표 및 역할 |
| [02. Backend 설계](./docs/02-backend-design.md) | Backend 구조 및 설계 |
| [03. Warehouse Domain](./docs/03-warehouse-domain.md) | Warehouse·Zone·Node·Edge |
| [04. Digital Twin Graph](./docs/04-digital-twin-graph.md) | PostgreSQL·Neo4j 연동 |
| [05. AI Integration](./docs/05-ai-integration.md) | AI Planning 연동 |
| [06. Multi-user Isolation](./docs/06-guest-access.md) | USER/GUEST 실행 환경 격리 |

---

## Repository

**Portfolio Repository**  
https://github.com/pbjun2000/digital-twin-warehouse-backend

**Team Project**  
https://github.com/kt-aivle-big-project
