# 05. Warehouse-AI Integration

> 이 문서는 LARO의 AI Planning 전체 구현이 아니라,  
> 제가 담당한 **Warehouse 데이터를 AI Planning에서 활용할 수 있도록 제공한 Backend 연동 영역**을 정리합니다.

---

## 1. 연동 목적

AI Planning이 창고의 실제 이동 구조를 기준으로 작업을 계획하려면
현재 Warehouse의 Node · Edge와 Robot 등 서비스 데이터가 필요합니다.

Warehouse 구조를 AI 영역에서 별도로 관리하면
Backend의 실제 창고 데이터와 서로 다른 Map을 사용할 가능성이 있습니다.

따라서 Warehouse 데이터의 기준은 Backend에 두고,
AI Planning에서 필요한 형태로 조회할 수 있도록 구성했습니다.

```text
PostgreSQL
Warehouse Data
      ↓
Spring Boot
Warehouse API / Graph API
      ↓
AI Planning
```

AI가 Warehouse 구조 자체를 생성하는 것이 아니라,
Backend에서 관리하는 데이터를 Planning에 활용할 수 있도록
데이터 제공 경계를 두었습니다.

---

## 2. Warehouse Graph API

AI Planning에서 Robot의 이동 구조를 사용할 수 있도록
Node · Edge 기반 Graph 조회 API를 구현했습니다.

```http
GET /api/warehouses/{warehouseId}/graph
```

전체 흐름은 다음과 같습니다.

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

Graph Response에는
현재 Warehouse의 이동 네트워크를 구성하는 Node와 Edge 정보가 포함됩니다.

```text
WarehouseGraphResponse
 ├─ warehouseId
 ├─ warehouseName
 ├─ nodeCount
 ├─ edgeCount
 ├─ nodes[]
 └─ edges[]
```

이를 통해 AI 영역에서
DB Entity를 직접 조회하지 않고 Backend API를 통해
Warehouse의 이동 구조를 사용할 수 있도록 했습니다.

---

## 3. Node · Edge 데이터 제공

Graph API에서는 단순 좌표뿐 아니라
Robot 이동에 필요한 Node · Edge 속성도 함께 제공합니다.

### Node

예를 들어 Node에는 다음과 같은 정보가 포함됩니다.

```text
nodeCode
nodeType
x / y

serviceOnly
transitAllowed
holdingAllowed
nodeCapacity

resourceType
resourceCode
side
```

### Edge

Edge에서는 연결 관계와 이동 조건을 제공합니다.

```text
edgeCode

fromNode
toNode

distance
direction

speedLimit
travelTime
cost

serviceOnly
mobileRobotTraversable
```

이 구조를 통해 AI Planning에서
어떤 Node가 연결되어 있고,
각 이동 구간에 어떤 조건이 있는지를 사용할 수 있도록 했습니다.

---

## 4. DB 식별자와 Map 식별자 분리

Backend Entity에서는 Numeric PK를 사용하지만,
AI Planning에 제공하는 Warehouse Map에서는
`nodeCode`와 `edgeCode`를 사용합니다.

```text
PostgreSQL
Node PK = 351
Edge PK = 824

        ↓

Spring Boot

        ↓

AI Planning
Node Code = R0_0
Edge Code = H0_0
```

즉,

```text
DB 내부
Numeric PK

        ↓

외부 Warehouse Map
nodeCode / edgeCode
```

로 식별 체계를 분리했습니다.

이를 통해 AI Planning이
PostgreSQL의 내부 PK에 직접 의존하지 않고
Warehouse Map 자체의 식별자를 사용할 수 있도록 했습니다.

이 구조는 Frontend의 Warehouse Editor에서도
동일한 Map 식별자를 사용할 수 있다는 장점이 있습니다.

---

## 5. Warehouse Scope

LARO에서는 여러 Warehouse가 존재할 수 있기 때문에
Graph 데이터 역시 Warehouse 단위로 구분합니다.

```text
Warehouse A
 ├─ Node A1
 ├─ Node A2
 └─ Edge A1-A2


Warehouse B
 ├─ Node B1
 ├─ Node B2
 └─ Edge B1-B2
```

AI Planning 요청에서도
대상 `warehouseId`를 기준으로 해당 Warehouse의 Graph를 조회합니다.

```text
warehouseId
    ↓
Warehouse 조회
    ↓
해당 Warehouse의 Node / Edge
    ↓
WarehouseGraphResponse
```

이를 통해 다른 Warehouse의 이동 구조가
Planning 데이터에 섞이지 않도록 범위를 분리했습니다.

USER / GUEST별 Personal Warehouse 역시
각각 별도의 Warehouse Scope를 사용합니다.

사용자별 실행 환경은  
[06. Multi-user Isolation](./06-multi-user-isolation.md)에서 설명합니다.

---

## 6. PostgreSQL과 Graph 데이터 일관성

AI에서 사용하는 Warehouse 구조가
Backend의 현재 데이터와 달라지지 않도록
Warehouse 변경 시 Graph Sync도 함께 관리했습니다.

```text
Warehouse 변경
      ↓
PostgreSQL COMMIT
      ↓
WarehouseGraphChangedEvent
      ↓
Neo4j Graph Sync
```

Warehouse의 기준 데이터는 PostgreSQL에 두고,
Neo4j는 해당 데이터를 기반으로 동기화되는 Graph Projection으로 사용합니다.

따라서 Warehouse 구조를 변경한 이후에도
Backend와 Graph 데이터가 같은 Warehouse 상태를 기준으로
유지될 수 있도록 구성했습니다.

Graph Sync 구조는  
[04. Digital Twin Graph](./04-digital-twin-graph.md)에서 자세히 설명합니다.

---

## 7. 전체 AI Planning 흐름과의 관계

LARO 전체 시스템에서는
Backend에서 제공한 Warehouse 데이터가
AI Planning 과정의 입력 중 하나로 사용됩니다.

```text
Backend
Warehouse / Robot Data
        ↓
Warehouse Graph
        ↓
AI Planning
        ↓
Optimization / MAPF
        ↓
Simulation Plan
```

이 중 제가 담당한 범위는

```text
Warehouse 데이터 관리
        ↓
Graph API 구성
        ↓
AI에서 사용할 Warehouse 데이터 제공
```

까지입니다.

AI 내부의 Rule / Agent 판단,
Optimization Solver,
MAPF,
AI Plan의 Simulation 적용 로직은
팀의 다른 영역에서 구현했습니다.

---

## 8. 주요 Component

| Component | 역할 |
|---|---|
| `WarehouseGraphService` | Warehouse의 Node · Edge Graph 데이터 구성 |
| `WarehouseService` | Warehouse 기준 데이터 관리 |
| `WarehouseNodeService` | Node 데이터 관리 |
| `WarehouseEdgeService` | Edge 및 이동 관계 관리 |
| `GraphSyncService` | PostgreSQL 기준 Neo4j Graph 동기화 |

Warehouse-AI Integration에서는

**AI 실행 로직 자체를 구현하는 것이 아니라,  
Backend가 관리하는 Digital Twin Warehouse 데이터를  
AI Planning에서 활용할 수 있는 형태로 제공하는 것**

을 담당했습니다.

---

## 관련 문서

- [01. Project Overview](./01-project-overview.md)
- [02. Backend Design](./02-backend-design.md)
- [03. Warehouse Domain](./03-warehouse-domain.md)
- [04. Digital Twin Graph](./04-digital-twin-graph.md)
- [06. Multi-user Isolation](./06-multi-user-isolation.md)

---

[← 04. Digital Twin Graph](./04-digital-twin-graph.md) · [README](../README.md)
