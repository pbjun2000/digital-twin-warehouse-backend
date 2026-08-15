# 02. Backend Design

> LARO Backend는 단순 API 서버가 아니라  
> **사용자 요청과 Warehouse 상태를 AI Planning으로 연결하고, 생성된 Plan을 실제 Simulation 실행 상태로 변환·관리하는 서비스 계층**으로 설계했습니다.

---

## 1. Backend의 책임

LARO는 Frontend, Backend, AI Planning Server의 역할을 분리했습니다.

```text
Frontend
사용자 입력 · Simulation 화면 · 실시간 상태 시각화
        ↓
Spring Boot Backend
권한 검증 · 데이터 관리 · AI 연동 · 실행 상태 관리
        ↓
FastAPI AI Server
Mission 구성 · 최적화 · 경로 계획 · 재계획
```

Backend가 AI의 계획 생성이나 경로 최적화까지 직접 수행하지 않고,
서비스 실행에 필요한 데이터와 상태를 관리하도록 책임을 구분했습니다.

### Backend 주요 책임

- 사용자 인증 및 Resource 소유권 검증
- Warehouse / Robot / Inventory / Task 데이터 관리
- 사용자별 Simulation 실행 환경 구성
- AI Planning 요청 및 결과 수신
- AI Plan 실행 가능 상태 검증
- AI Task와 Backend Task Mapping
- Simulation Playback 적용
- Runtime State 관리 및 실시간 전달
- PostgreSQL과 Neo4j 데이터 동기화

---

## 2. Backend 중심 실행 흐름

사용자가 Simulation을 실행하면 Backend가 현재 Warehouse 상태와 Task 정보를 기준으로
AI Planning에 필요한 실행 환경을 구성합니다.

```text
Frontend
창고 · 시나리오 선택
Simulation 실행 요청
        ↓
Spring Boot
권한 및 Warehouse 소유권 검증
        ↓
Warehouse / Robot / Task 상태 조회
        ↓
AI Planning Request 구성
        ↓
FastAPI
Mission · Optimization · MAPF
        ↓
SimulationPlan 반환
        ↓
Spring Boot
Plan 상태 및 Task 관계 검증
        ↓
Simulation Playback 적용
        ↓
Redis Runtime State
        ↓
WebSocket / STOMP
        ↓
Frontend 실시간 관제
```

AI가 반환한 결과를 Frontend가 직접 사용하지 않고,
**Backend가 AI 결과와 실제 서비스 데이터를 연결한 뒤 실행하도록 구성**했습니다.

---

## 3. Backend와 AI의 책임 분리

프로젝트 초기에는 Backend에도 재계획과 경로 처리 책임이 일부 존재했습니다.

AI Planning 기능이 확장되면서 동일한 판단 로직이 Backend와 AI에 중복될 가능성이 생겼고,
최종적으로 다음과 같이 책임을 다시 정의했습니다.

| 영역 | 책임 |
|---|---|
| **Frontend** | 사용자 입력, Simulation 제어, 상태 시각화 |
| **Backend** | 요청 검증, 데이터 관리, 실행 상태 관리, AI Plan 적용 |
| **AI Server** | Mission 구성, 제약조건 정식화, 작업 최적화, 경로 계획, 재계획 |
| **Solver / MAPF** | 작업 배정, 수행 순서 계산, 다중 로봇 이동 계획 |

```text
AI
"어떤 계획을 실행할 것인가?"

Backend
"이 계획을 현재 서비스에서 실행할 수 있는가?"

Frontend
"현재 무엇이 실행되고 있는가?"
```

이를 통해 Backend가 AI의 판단 로직까지 가지지 않고
**서비스 데이터와 실행 상태의 일관성을 관리하는 역할**에 집중하도록 했습니다.

---

## 4. AI Plan과 Backend 실행 데이터 연결

AI Server와 Backend는 Task를 서로 다른 식별 체계로 관리할 수 있기 때문에,
AI가 반환한 Plan을 바로 Simulation에 적용할 수 없습니다.

Backend에서는 먼저 AI Plan의 실행 가능 상태를 확인한 뒤
AI Task와 실제 Backend Task를 연결합니다.

```text
SimulationPlan
      ↓
READY 상태 확인
      ↓
AI Task ID
      ↓
AI Task ↔ BE Task Mapping
      ↓
Robot Plan / Plan Step
      ↓
Prepared Execution
      ↓
SimulationPlaybackService
      ↓
Simulation 실행
```

대표적인 실행 흐름에서는 다음 단계를 수행합니다.

1. AI Plan이 `READY` 상태인지 확인
2. AI Task ID와 Backend Task ID Mapping
3. Task와 SimulationRun 관계 검증
4. Robot별 Plan / Step 실행 구조 구성
5. `SimulationPlaybackService`에 Plan 적용

AI 결과와 Backend 데이터를 명시적으로 연결함으로써
**AI가 생성한 식별자와 실제 서비스 데이터가 불일치한 상태로 실행되는 것을 방지**했습니다.

---

## 5. Runtime State와 영속 데이터 분리

Simulation에서는 Robot 위치와 배터리처럼 빠르게 변경되는 데이터와
Warehouse·Task처럼 정합성이 중요한 데이터의 특성이 다릅니다.

따라서 저장소별 책임을 분리했습니다.

```text
PostgreSQL
영속 데이터
Warehouse · Robot · Inventory · Task · Simulation 이력

Redis
Runtime State
Robot 위치 · 배터리 · 실행 상태

Neo4j
Warehouse Graph
Node · Edge · 접근 관계
```

### PostgreSQL

서비스의 기준이 되는 영속 데이터를 관리합니다.

- Warehouse
- Robot
- Inventory
- Task
- Scenario
- SimulationRun
- 실행 이력

### Redis

Simulation 실행 중 빠르게 변경되는 Runtime State를 관리합니다.

- Robot 현재 위치
- 배터리
- 실행 상태
- 빠른 상태 조회

### Neo4j

Warehouse의 공간 구조를 관계 중심으로 표현합니다.

- Node
- Edge
- Storage 접근 관계
- Charging Station 연결 관계

Backend에서는 각 저장소의 역할을 분리하되,
**PostgreSQL을 Warehouse 데이터의 Source of Truth로 유지**했습니다.

---

## 6. Transaction 이후 Graph 동기화

Personal Warehouse 생성이나 Warehouse 구조 변경 이후에는
PostgreSQL 데이터와 Neo4j Warehouse Graph가 함께 갱신되어야 합니다.

두 저장소를 동일한 Transaction으로 묶을 수 없기 때문에
PostgreSQL 저장이 완료된 이후 Graph Sync가 실행되도록 구성했습니다.

```text
Warehouse 데이터 변경
        ↓
PostgreSQL Transaction
        ↓
COMMIT
        ↓
WarehouseGraphChangedEvent(warehouseId)
        ↓
AFTER_COMMIT
        ↓
GraphSyncService
        ↓
Neo4j Warehouse Graph Sync
```

이벤트는 변경된 Warehouse의 ID를 전달하고,
Commit 이후 해당 Warehouse Scope의 Graph를 다시 동기화합니다.

```java
@TransactionalEventListener(
    phase = TransactionPhase.AFTER_COMMIT
)
```

이 구조를 통해 PostgreSQL Transaction이 실패한 상태에서
Neo4j만 먼저 변경되는 상황을 방지했습니다.

> 상세 구현은 [04. Digital Twin Graph](./04-digital-twin-graph.md)에서 설명합니다.

---

## 7. 사용자별 실행 환경

Simulation의 상태는 실행 중 계속 변경되기 때문에
여러 사용자가 하나의 Shared Warehouse를 직접 사용하도록 할 수 없었습니다.

Backend에서는 Shared Warehouse를 Template으로 사용하고,
실제 실행 시 USER / GUEST별 Personal Warehouse를 생성합니다.

```text
Shared Warehouse Template
          ↓
WarehouseTemplateCloneService
          ↓
Personal Warehouse
          ↓
Ownership Validation
          ↓
SimulationRun
```

Personal Warehouse에는 실행에 영향을 주는 종속 데이터를 함께 복제합니다.

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
GUEST는 `guest_session_id`를 기준으로 실행 환경을 분리합니다.

Simulation 실행 단계에서도 Warehouse 소유권을 다시 검증하여
다른 사용자의 `warehouseId`를 직접 전달하는 방식의 접근을 차단했습니다.

> 상세 구현은 [06. Multi-user Isolation](./06-guest-access.md)에서 설명합니다.

---

## 8. Simulation 실행 상태 관리

AI Planning과 Simulation 실행은 한 번의 API 요청으로 끝나는 작업이 아닙니다.

특히 재계획에서는 기존 Plan을 실행하는 도중 새로운 Plan이 생성되기 때문에
계획 생성 상태와 실제 실행 상태를 분리해서 관리해야 합니다.

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

새로운 Plan을 수신했다고 즉시 기존 Plan을 교체하지 않고,
현재 실행 상태를 기준으로 새로운 Plan을 준비한 뒤 실행 흐름에 적용합니다.

Backend는 이 과정에서

- 현재 SimulationRun 확인
- Plan 상태 검증
- Task Mapping
- 새로운 Plan 준비
- Playback 적용
- Runtime State 관리

를 담당합니다.

이를 통해 **AI는 새로운 계획을 생성하고,
Backend는 기존 실행과 새로운 계획 사이의 전환을 관리**하도록 역할을 분리했습니다.

---

## 9. 인증과 Resource 접근 제어

REST API에서는 인증 여부뿐 아니라
요청한 Warehouse가 현재 사용자에게 속한 Resource인지 함께 검증합니다.

```text
JWT 인증
   ↓
USER / GUEST 식별
   ↓
Warehouse 조회
   ↓
Ownership Validation
   ↓
Service 실행
```

주요 정책은 다음과 같습니다.

- Shared Template Warehouse 직접 Simulation 실행 차단
- USER는 자신의 `user_id`에 속한 Warehouse만 실행
- GUEST는 JWT Subject 기반 `guest_session_id`로 실행 환경 식별
- 다른 사용자의 Warehouse ID 직접 접근 차단

단순한 로그인 여부가 아니라
**실제 Business Resource의 소유권까지 실행 경계에서 검증**하도록 구성했습니다.

---

## 10. 주요 Backend Component

최종 구조에서 핵심 역할을 담당하는 Component는 다음과 같습니다.

| Component | 역할 |
|---|---|
| `SimulationRunService` | Simulation 생성·실행 검증 및 실행 상태 관리 |
| `SimulationPlaybackService` | AI Plan을 Robot 실행 구조로 적용 |
| `WarehouseTemplateCloneService` | Shared Warehouse를 Personal Warehouse로 Deep Clone |
| `GraphSyncService` | PostgreSQL 기반 Neo4j Warehouse Graph 동기화 |
| `WarehouseGraphChangedEvent` | Warehouse Graph 재동기화 이벤트 |
| `AuthenticatedRequesterResolver` | USER / GUEST 요청자 식별 |

각 Service가 AI 판단, 데이터 복제, Graph Sync, 실행 상태 관리 등
서로 다른 책임을 갖도록 분리했습니다.

---

## 11. Backend 설계 요약

LARO Backend에서 중요하게 둔 기준은 다음 세 가지입니다.

### 실행의 기준은 Backend 데이터

AI 결과를 그대로 신뢰하지 않고
실제 Task 및 Simulation 상태와 연결한 뒤 실행합니다.

### 데이터 특성에 따라 저장소 책임 분리

```text
PostgreSQL → 영속 데이터
Redis      → Runtime State
Neo4j      → Warehouse Graph
```

### AI와 서비스 실행 책임 분리

```text
AI
Planning · Optimization · Replanning

Backend
Validation · State · Execution

Frontend
Control · Visualization
```

이 구조를 통해 Spring Boot Backend가
**AI Planning과 실제 서비스 실행 사이의 경계에서 데이터와 상태의 일관성을 관리**하도록 설계했습니다.

---

## 다음 문서

- [03. Warehouse Domain](./03-warehouse-domain.md)
- [04. Digital Twin Graph](./04-digital-twin-graph.md)
- [05. AI Integration](./05-ai-integration.md)
- [06. Multi-user Isolation](./06-guest-access.md)

---

[← 01. Project Overview](./01-project-overview.md) · [README](../README.md)
