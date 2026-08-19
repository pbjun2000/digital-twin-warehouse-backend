# 02. Backend Design

> 이 문서는 LARO의 전체 Backend 구조와 서비스 간 책임을 설명합니다.  
> 이 중 제가 직접 담당한 영역은 **Warehouse Domain, Warehouse Graph, Multi-user 실행 환경 및 관련 데이터 연동**입니다.

---

## 1. Backend의 역할

LARO는 Frontend, Spring Boot Backend, AI Planning Server가 분리된 구조로 구성되어 있습니다.

Spring Boot는 서비스의 기준 데이터를 관리하고,
Frontend와 AI Planning 사이에서 Warehouse · Robot · Task · Simulation 데이터를 연결합니다.

```text
Frontend
    │
    │ REST / STOMP
    ▼
Spring Boot
    │
    ├── Warehouse / Robot
    ├── Inventory / Task
    ├── Scenario / Simulation
    ├── User Resource Validation
    └── AI Planning 연동
            │
            ▼
      FastAPI Planning
```

Backend 전체에서는 다음 역할을 담당합니다.

- Warehouse / Robot / Inventory / Task 데이터 관리
- Scenario 및 Simulation 실행 상태 관리
- 사용자 요청 및 Resource 접근 검증
- AI Planning에 필요한 서비스 데이터 제공
- AI Planning 결과의 Simulation 적용
- Runtime State와 Frontend 실시간 관제 연동
- Warehouse 변경에 따른 Neo4j Graph 동기화

이 중 저는 **Warehouse · Robot · Graph와 사용자별 Warehouse 실행 환경**을 중심으로 구현했습니다.

---

## 2. 서비스 간 책임 분리

프로젝트에서는 하나의 서비스가 모든 판단과 실행을 처리하지 않고
Frontend, Backend, AI 영역의 책임을 나눴습니다.

| 영역 | 주요 책임 |
|---|---|
| **Frontend** | 사용자 입력, Warehouse 편집, Simulation 및 Robot 상태 시각화 |
| **Spring Boot** | 서비스 데이터 관리, Resource 검증, Simulation 실행 및 AI 연동 |
| **AI Planning** | 자연어 요청 해석, Mission·Constraint 구성, 계획 및 재계획 |
| **Optimization / MAPF** | 작업 배정·수행 순서 최적화 및 다중 로봇 이동 계획 계산 |

전체 서비스 흐름은 다음과 같습니다.

```text
사용자 요청
    ↓
Frontend
    ↓
Spring Boot
현재 서비스 데이터 제공
    ↓
AI Planning
Mission · Constraints · Robot Plan 생성
    ↓
Spring Boot
Simulation 실행 구조에 적용
    ↓
Frontend 관제
```

AI가 Planning을 담당하더라도
Warehouse, Robot, Inventory 등 서비스 기준 데이터는
Backend에서 관리하도록 역할을 구분했습니다.

---

## 3. Warehouse 데이터를 중심으로 한 연동 구조

제가 담당한 Backend 영역에서는
Frontend Editor, AI Planning, Simulation이
동일한 Warehouse 데이터를 사용할 수 있도록 구성하는 것이 중요했습니다.

```text
                PostgreSQL
                    │
             Warehouse Data
                    │
        ┌───────────┼───────────┐
        ▼           ▼           ▼
    Frontend    AI Planning   Simulation
     Editor
```

Warehouse 구조는 Backend에서 관리하고,
외부 영역에서 필요한 형태에 따라 각각 제공합니다.

```text
Warehouse / Zone / Node / Edge
              │
              ├── Warehouse Layout API
              │       → Frontend
              │
              ├── Warehouse Graph API
              │       → AI Planning
              │
              └── Personal Warehouse
                      → Simulation
```

이를 통해 Frontend, AI, Simulation이
서로 다른 Warehouse 구조를 별도로 관리하지 않도록 했습니다.

---

## 4. 데이터 저장소 역할 분리

LARO에서는 데이터 특성에 따라 저장소의 역할을 구분했습니다.

| 저장소 | 역할 |
|---|---|
| **PostgreSQL** | Warehouse, Robot, Inventory, Task, Scenario 등 영속 데이터 |
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

Warehouse 데이터에서는 PostgreSQL을 기준 데이터로 사용하고,
Neo4j는 PostgreSQL의 현재 상태를 기반으로 생성되는
Warehouse Graph Projection으로 구성했습니다.

Graph 동기화 방식은  
[04. Digital Twin Graph](./04-digital-twin-graph.md)에서 자세히 설명합니다.

---

## 5. 사용자별 실행 환경

여러 사용자가 하나의 Shared Warehouse를 그대로 실행하면
Inventory와 Robot 상태가 서로 영향을 줄 수 있습니다.

이를 방지하기 위해 Shared Warehouse는 Template으로 사용하고,
실제 Simulation은 USER / GUEST별 Personal Warehouse에서 실행하도록 구성했습니다.

```text
Shared Warehouse
     Template
        │
   ┌────┼────┐
   ▼    ▼    ▼
User A User B Guest
   │    │    │
   ▼    ▼    ▼
Personal Warehouse
```

Personal Warehouse에는 Warehouse뿐 아니라
실행에 영향을 주는 종속 데이터도 함께 분리합니다.

```text
Zone
Node
Edge
StorageLocation
ChargingStation
Inventory
Robot
Scenario
```

또한 Simulation 실행 시
현재 요청자가 해당 Warehouse를 사용할 수 있는지 소유권을 검증합니다.

세부 구조는  
[06. Multi-user Isolation](./06-multi-user-isolation.md)에서 설명합니다.

---

## 6. Warehouse Graph와 AI Planning

AI Planning에서 현재 Warehouse의 이동 구조를 사용할 수 있도록
Node · Edge 기반 Graph 조회 API를 제공합니다.

```text
PostgreSQL
Warehouse / Node / Edge
        ↓
WarehouseGraphService
        ↓
WarehouseGraphResponse
        ↓
AI Planning
```

DB 내부에서는 Numeric PK를 사용하지만
외부 Warehouse Map에서는 `nodeCode`와 `edgeCode`를 사용합니다.

```text
Database
Numeric PK
    ↓
Backend
nodeCode / edgeCode
    ↓
Frontend / AI
```

이를 통해 외부 시스템이 DB PK에 직접 의존하지 않고
Warehouse Map을 사용할 수 있도록 했습니다.

상세 내용은  
[05. Warehouse-AI Integration](./05-warehouse-ai-integration.md)에서 설명합니다.

---

## 7. 주요 Component

### 직접 담당한 주요 Component

| Component | 역할 |
|---|---|
| `WarehouseService` | Warehouse 기본 정보 및 정책 관리 |
| `WarehouseLayoutService` | Warehouse 구성 데이터 통합 조회·수정 |
| `WarehouseGraphService` | Node·Edge 기반 Warehouse Graph 제공 |
| `WarehouseTemplateCloneService` | Shared Warehouse 기반 Personal Warehouse 생성 |
| `GraphSyncService` | PostgreSQL 기준 Neo4j Warehouse Graph 동기화 |
| `WarehouseNodeService` | Node 관리 및 참조 관계 검증 |
| `WarehouseEdgeService` | Edge와 이동 관계 관리 |

### Backend 전체 구조의 주요 영역

팀 Backend에는 이외에도
Simulation 실행, Runtime State, AI Planning 결과 적용,
실시간 통신을 담당하는 영역이 별도로 구성되어 있습니다.

해당 영역은 팀 전체 Backend 구조의 일부이며,
본 포트폴리오에서는 제가 담당한 Warehouse · Graph 영역을 중심으로 설명합니다.

---

## 8. Backend 설계 정리

제가 담당한 영역에서는 다음 세 가지를 중심으로 Backend 구조를 설계했습니다.

```text
Warehouse Domain
      │
      ├── Frontend Layout
      │
      ├── AI Planning Graph
      │
      └── Personal Simulation
              │
              ▼
       Warehouse Scope
```

주요 설계 과제는 다음과 같습니다.

- 하나의 Warehouse 데이터를 Frontend · AI · Simulation에서 공통 사용
- PostgreSQL과 Neo4j의 기준 데이터 및 동기화 시점 분리
- USER / GUEST별 실행 데이터와 Warehouse Resource 격리
- DB 내부 식별자와 외부 Warehouse Map 식별자 분리

각 구현 내용은 이후 문서에서 세부적으로 설명합니다.

---

## 관련 문서

- [01. Project Overview](./01-project-overview.md)
- [03. Warehouse Domain](./03-warehouse-domain.md)
- [04. Digital Twin Graph](./04-digital-twin-graph.md)
- [05. Warehouse-AI Integration](./05-warehouse-ai-integration.md)
- [06. Multi-user Isolation](./06-multi-user-isolation.md)

---

[← 01. Project Overview](./01-project-overview.md) · [README](../README.md)
