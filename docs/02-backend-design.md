# 02. Backend Design

> LARO Backend는 Warehouse와 Simulation의 기준 데이터를 관리하고,  
> AI Planning 결과를 실제 서비스 실행 상태와 연결하는 역할을 담당합니다.

---

## 1. Backend의 역할

LARO는 Frontend, Spring Boot Backend, AI Planning Server가 분리된 구조로 구성되어 있습니다.

Spring Boot는 단순 API Gateway가 아니라  
**서비스 데이터와 실제 Simulation 실행 상태를 관리하는 중심 영역**으로 두었습니다.

주요 책임은 다음과 같습니다.

- Warehouse / Robot / Inventory / Task 데이터 관리
- Simulation 및 실행 상태 관리
- USER / GUEST 요청자 식별과 Resource 검증
- AI Planning 요청을 위한 서비스 데이터 구성
- AI Plan 검증 및 Backend Task 연결
- Runtime State와 Frontend 실시간 관제 연동
- Warehouse 변경에 따른 Graph Sync Trigger

```text
Frontend
    │
    │ REST / STOMP
    ▼
Spring Boot
    │
    ├── Service Data
    │   Warehouse · Robot · Task · Scenario
    │
    ├── Simulation Execution
    │
    ├── Runtime State
    │
    └── AI Integration
            │
            ▼
      FastAPI Planning
```

---

## 2. 서비스 간 책임 분리

프로젝트가 확장되면서 하나의 서비스에서 모든 판단과 실행을 처리하지 않고
각 영역의 책임을 분리했습니다.

| 영역 | 책임 |
|---|---|
| **Frontend** | 사용자 입력, Warehouse 편집, Simulation 및 Robot 상태 시각화 |
| **Spring Boot** | 서비스 데이터 관리, 요청 검증, Simulation 상태 관리, AI Plan 적용 |
| **AI Planning** | 자연어 요청 해석, Mission·Constraint 구성, 최적화 및 재계획 |
| **Optimization / MAPF** | 작업 배정·순서 최적화 및 다중 로봇 이동 계획 계산 |

전체 흐름은 다음과 같습니다.

```text
사용자 요청
    ↓
Frontend
    ↓
Spring Boot
현재 Warehouse / Robot / Task 상태 구성
    ↓
AI Planning
Mission · Constraints · Robot Plan 생성
    ↓
Spring Boot
Plan 검증 및 서비스 데이터 연결
    ↓
Simulation 실행
    ↓
Frontend 관제
```

AI가 Planning을 담당하더라도  
**실제 서비스 데이터와 Simulation 상태의 기준은 Backend가 관리**하도록 역할을 구분했습니다.

---

## 3. Planning과 Execution 분리

AI가 반환한 결과를 곧바로 Robot 실행 상태에 반영하지 않습니다.

Backend에서 AI Plan의 상태를 확인하고,
실제 Backend Task와 연결한 뒤 Simulation 실행 구조에 적용합니다.

```text
AI Planning Result
        ↓
Backend Validation
        ↓
Task Mapping
        ↓
Simulation Playback
        ↓
Runtime State
```

이를 통해 다음 두 영역을 분리했습니다.

```text
Planning
"무엇을 어떻게 수행할 것인가?"

        ↓

Execution
"현재 서비스에서 이 계획을 어떻게 실행할 것인가?"
```

AI 연동과 Plan 적용 과정은  
[05. AI Integration](./05-ai-integration.md)에서 자세히 설명합니다.

---

## 4. 데이터 저장소 역할 분리

LARO에서는 모든 데이터를 하나의 저장소에 넣지 않고
데이터의 성격에 따라 역할을 나눴습니다.

| 저장소 | 역할 |
|---|---|
| **PostgreSQL** | Warehouse, Robot, Inventory, Task, Scenario, Simulation 등 영속 데이터 |
| **Redis** | Robot 위치·배터리·실행 상태 등 빠르게 변경되는 Runtime State |
| **Neo4j** | Node·Edge 및 창고 객체의 이동·접근 관계 |

```text
              Spring Boot
                   │
        ┌──────────┼──────────┐
        │          │          │
        ▼          ▼          ▼
 PostgreSQL      Redis      Neo4j
 Persistent     Runtime      Graph
   Data          State     Projection
```

정합성이 필요한 서비스 데이터는 PostgreSQL을 기준으로 관리하고,
빠르게 변하는 실행 상태와 공간 관계 탐색은
Redis와 Neo4j로 분리했습니다.

Neo4j 동기화 방식은  
[04. Digital Twin Graph](./04-digital-twin-graph.md)에서 설명합니다.

---

## 5. 사용자별 실행 환경

여러 사용자가 하나의 Shared Warehouse를 동시에 수정하거나 실행하면
Inventory와 Robot 상태가 서로 영향을 줄 수 있습니다.

이를 방지하기 위해 Shared Warehouse는 Template으로 사용하고,
실제 Simulation은 USER / GUEST별 Personal Warehouse에서 실행합니다.

```text
Shared Warehouse
     Template
        │
   ┌────┼────┐
   ▼    ▼    ▼
User A User B Guest
   │    │    │
Personal Warehouse
```

Backend에서는 Personal Warehouse 생성과
Simulation 실행 시의 Resource 검증을 담당합니다.

세부적인 Deep Clone 및 사용자 격리 구조는  
[06. Multi-user Isolation](./06-multi-user-isolation.md)에서 설명합니다.

---

## 6. 주요 Backend Component

Backend의 주요 서비스는 다음과 같이 역할을 나눴습니다.

| Component | 역할 |
|---|---|
| `WarehouseService` | Warehouse CRUD 및 기본 정책 처리 |
| `WarehouseLayoutService` | Warehouse 구성 데이터 통합 조회·수정 |
| `WarehouseGraphService` | AI 및 외부 연동용 Node·Edge Graph 구성 |
| `WarehouseTemplateCloneService` | Shared Warehouse 기반 Personal Warehouse 생성 |
| `GraphSyncService` | PostgreSQL 기준 Neo4j Warehouse Graph 동기화 |
| `SimulationRunService` | Simulation 생성·실행 및 AI Plan 연동 |
| `SimulationPlaybackService` | Robot Plan을 Runtime 실행 상태에 적용 |
| `AuthenticatedRequesterResolver` | USER / GUEST 요청자 식별 |

각 Service가 Warehouse 관리, Graph 동기화,
Planning 연동, Simulation 실행을 나누어 담당하도록 구성했습니다.

---

## 7. Backend 설계 정리

LARO Backend에서는 다음 세 가지 경계를 명확하게 두는 데 중점을 뒀습니다.

```text
서비스 데이터
PostgreSQL / Spring Boot
        │
        ▼
AI Planning
FastAPI / Solver / MAPF
        │
        ▼
실행 상태
Simulation / Redis / Frontend
```

Backend는 이 사이에서

- 서비스 데이터의 기준 유지
- 사용자 Resource 검증
- AI Planning에 필요한 현재 상태 제공
- AI 결과와 실제 Task 연결
- Simulation 실행 상태 관리

를 담당합니다.

세부 구현은 각 문서에서 분리해 설명합니다.

---

## 관련 문서

- [01. Project Overview](./01-project-overview.md)
- [03. Warehouse Domain](./03-warehouse-domain.md)
- [04. Digital Twin Graph](./04-digital-twin-graph.md)
- [05. AI Integration](./05-ai-integration.md)
- [06. Multi-user Isolation](./06-multi-user-isolation.md)

---

[← 01. Project Overview](./01-project-overview.md) · [README](../README.md)
