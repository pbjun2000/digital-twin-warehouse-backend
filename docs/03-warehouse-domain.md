# 03. Warehouse Domain

> Warehouse Domain은 창고의 공간 구조와 이동 관계를 관리하고,  
> Frontend Editor와 AI Planning에서 동일한 Warehouse 데이터를 활용할 수 있도록 구성한 Backend 영역입니다.

---

## 1. Domain 구성

Robot이 창고에서 이동하고 작업을 수행하려면
창고 기본 정보뿐 아니라 구역, 이동 지점, 이동 경로와 설비 정보가 함께 필요합니다.

```text
Warehouse
 ├─ Zone
 ├─ Node
 │   ├─ Route
 │   ├─ Rack
 │   ├─ Inbound / Outbound
 │   └─ Charging
 │
 ├─ Edge
 │   └─ Node ↔ Node
 │
 ├─ StorageLocation
 ├─ ChargingStation
 └─ Robot
```

`Warehouse`를 기준으로
Frontend에서 편집하는 창고 구조와
AI Planning에서 사용하는 이동 데이터를 관리하도록 구성했습니다.

### Warehouse

Warehouse에는 창고의 기본 정보와
실행 환경을 구분하기 위한 정보를 함께 관리합니다.

```text
id
name
width / height
location
status

shared
user
guestSessionId
sourceTemplate
```

Shared Warehouse는 Template으로 사용하고,
USER / GUEST별 실행 환경은 Personal Warehouse로 분리합니다.

사용자별 Warehouse 구조는  
[06. Multi-user Isolation](./06-multi-user-isolation.md)에서 설명합니다.

### Zone

`WarehouseZone`은 창고 내부 공간의 역할을 구분합니다.

```text
STORAGE
MOVING
INBOUND
OUTBOUND
CHARGING
RESTRICTED
```

각 Zone은 좌표 범위를 가지며
창고 내 작업 영역을 데이터 수준에서 구분합니다.

```text
minX / maxX
minY / maxY
```

---

## 2. Node와 Edge

### Node

`WarehouseNode`는 Robot 이동과 작업의 기준이 되는 지점입니다.

Node를 단순 좌표로만 관리하지 않고
역할에 따라 Type을 구분합니다.

```text
ROUTE
ROUTE_CHARGE_JUNCTION

RACK_STORAGE
RACK_ACCESS

INBOUND_HANDOFF_ACCESS
OUTBOUND_STATION_ACCESS
EMPTY_TOTE_BUFFER_ACCESS

INBOUND
OUTBOUND

CHARGING_SLOT
PARKING_SLOT
```

예를 들어,

```text
ROUTE
→ 일반 이동 지점

RACK_STORAGE
→ 재고가 위치하는 Rack

RACK_ACCESS
→ Robot이 Rack에 접근하는 작업 지점

CHARGING_SLOT
→ Robot 충전 위치
```

처럼 동일한 좌표 데이터라도
창고에서 수행하는 역할에 따라 의미를 구분했습니다.

### Routing 속성

Node에는 위치뿐 아니라
이동 및 작업에 필요한 속성도 함께 관리합니다.

```text
serviceOnly
transitAllowed
holdingAllowed
nodeCapacity

resourceType
resourceCode
side
```

예를 들어 `transitAllowed`는
해당 Node를 이동 경로로 사용할 수 있는지를 표현합니다.

추가적인 Map Metadata는
`route_attributes` JSONB 형태로 관리할 수 있도록 구성되어 있습니다.

```java
@JdbcTypeCode(SqlTypes.JSON)
@Column(name = "route_attributes", columnDefinition = "jsonb")
private Map<String, Object> routeAttributes;
```

자주 사용하는 값은 Typed Column으로 관리하고,
추가 Metadata는 별도 구조로 분리했습니다.

---

### Edge

`WarehouseEdge`는 두 Node 사이의 이동 관계를 표현합니다.

```text
Node A
  │
  │ Edge
  ▼
Node B
```

주요 속성은 다음과 같습니다.

```text
fromNode
toNode

distance
directionType

speedLimitMps
nominalTravelTimeMs
cost

serviceOnly
mobileRobotTraversable
physicalResourceCode
```

이동 방향은 다음과 같이 구분합니다.

```text
BOTH
A_TO_B
B_TO_A
```

Node와 Edge를 분리하여
창고의 위치 정보와 이동 관계를 각각 관리할 수 있도록 구성했습니다.

---

## 3. DB PK와 Map 식별자 분리

JPA Entity 내부에서는 Numeric PK를 사용하지만,
Frontend와 AI Planning에서 사용하는 Warehouse Map에는 별도의 Code를 제공합니다.

```text
Database
Node PK = 351
Edge PK = 824

        ↓

Warehouse Map
Node Code = R0_0
Edge Code = H0_0
```

전체 흐름은 다음과 같습니다.

```text
PostgreSQL
Numeric PK
    ↓
Spring Boot
nodeCode / edgeCode
    ↓
Frontend / AI Planning
```

외부 시스템이 DB 내부 PK에 직접 의존하지 않고
Warehouse Map을 식별할 수 있도록
DB 식별자와 Map 식별자를 분리했습니다.

---

## 4. JSON 기반 Warehouse 구조 연동

Frontend에서 Warehouse를 직접 구성하는 것 외에도
JSON 형태의 Map 데이터를 Backend에 전달할 수 있도록 구성했습니다.

```http
POST /api/warehouses/import
```

입력 데이터는 크게 다음과 같은 구조를 가집니다.

```text
Warehouse 기본 정보
        +
Map
 ├─ Nodes
 └─ Edges
```

Warehouse 구조를 구성할 때는
Node와 Edge뿐 아니라 관련 Warehouse Resource도 함께 연결됩니다.

```text
Warehouse
    ↓
Zone / Node / Edge
    ↓
ChargingStation
StorageLocation
    ↓
Inventory / Robot
```

이를 통해 JSON 기반으로 전달된 창고 구조를
Backend의 Warehouse Domain 데이터로 사용할 수 있도록 연동했습니다.

---

## 5. Warehouse Layout

Frontend에서 Warehouse Editor와 Simulation 화면을 구성할 때
Zone, Node, Edge, Robot 데이터를 각각 호출하지 않도록
Layout 단위의 통합 조회 구조를 제공합니다.

```http
GET /api/warehouses/{warehouseId}/layout
```

응답은 다음과 같은 Warehouse 데이터를 포함합니다.

```text
WarehouseLayoutResponse
 ├─ Warehouse
 ├─ Zones
 ├─ Nodes
 ├─ Edges
 ├─ ChargingStations
 └─ Robots
```

```text
Frontend
    ↓
GET /layout
    ↓
Spring Boot
    ↓
Warehouse
Zone
Node / Edge
ChargingStation
Robot
```

동일한 Warehouse Scope에 속하는 데이터를
한 번에 조회할 수 있도록 구성했습니다.

---

## 6. Warehouse Layout 수정

Warehouse Editor에서 변경된 Layout은
다음 API를 통해 Backend에 반영합니다.

```http
PUT /api/warehouses/{warehouseId}/layout
```

Node와 Edge는 외부 Map에서 사용하는 Code를 기준으로
기존 데이터와 요청 데이터를 구분합니다.

```text
현재 Warehouse 데이터
        +
Editor 요청 데이터
        ↓
nodeCode / edgeCode 기준 식별
        ↓
Warehouse Layout 반영
```

이를 통해 Frontend Editor에서 구성한
Node · Edge 기반 창고 구조를 Backend 데이터와 연결했습니다.

---

## 7. Warehouse Graph API

AI Planning에서 현재 Warehouse의 이동 구조를 활용할 수 있도록
Node · Edge 기반 Graph 조회 API를 제공합니다.

```http
GET /api/warehouses/{warehouseId}/graph
```

응답은 Warehouse와
현재 이동 구조를 구성하는 Node · Edge 정보를 포함합니다.

```text
WarehouseGraphResponse
 ├─ warehouseId
 ├─ warehouseName
 ├─ nodeCount
 ├─ edgeCount
 ├─ nodes[]
 └─ edges[]
```

외부에는 DB Numeric PK 대신
`nodeCode`와 `edgeCode`를 기준으로 데이터를 제공합니다.

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

이를 통해 AI Planning이 DB 내부 구조에 직접 의존하지 않고
Backend가 관리하는 Warehouse 이동 데이터를 활용할 수 있도록 했습니다.

상세 연동 구조는  
[05. Warehouse-AI Integration](./05-warehouse-ai-integration.md)에서 설명합니다.

---

## 8. 주요 API

### Warehouse

| Method | Endpoint | 역할 |
|---|---|---|
| `GET` | `/api/warehouses` | Warehouse 목록 조회 |
| `GET` | `/api/warehouses/{id}` | Warehouse 상세 조회 |
| `POST` | `/api/warehouses` | Warehouse 생성 |
| `PATCH` | `/api/warehouses/{id}` | Warehouse 정보 수정 |
| `DELETE` | `/api/warehouses/{id}` | Warehouse 삭제 |
| `POST` | `/api/warehouses/import` | JSON 기반 Warehouse Import |
| `GET` | `/api/warehouses/{id}/layout` | Warehouse Layout 통합 조회 |
| `PUT` | `/api/warehouses/{id}/layout` | Warehouse Layout 수정 |
| `GET` | `/api/warehouses/{id}/graph` | Node · Edge Graph 조회 |

### Warehouse Resource

```text
/api/warehouse-zones
/api/warehouse-nodes
/api/warehouse-edges
```

Zone, Node, Edge는 각각
생성·조회·수정·삭제 API를 제공합니다.

---

## 9. 주요 Component

| Component | 역할 |
|---|---|
| `WarehouseService` | Warehouse 기본 정보 관리 |
| `WarehouseImportService` | JSON 기반 Warehouse 구조 연동 |
| `WarehouseLayoutService` | Warehouse Layout 조회 및 수정 |
| `WarehouseGraphService` | Node · Edge 기반 Graph 데이터 제공 |
| `WarehouseNodeService` | Warehouse Node 관리 |
| `WarehouseEdgeService` | Warehouse Edge 및 이동 관계 관리 |
| `WarehouseZoneService` | Warehouse Zone 관리 |
| `WarehouseTemplateCloneService` | Personal Warehouse 생성 |

Warehouse Domain에서는

**Warehouse · Zone · Node · Edge 데이터를 하나의 기준으로 관리하고,  
Frontend Editor와 AI Planning에서 활용할 수 있도록 제공하는 구조**

를 중심으로 구현했습니다.

---

## 관련 문서

- [01. Project Overview](./01-project-overview.md)
- [02. Backend Design](./02-backend-design.md)
- [04. Digital Twin Graph](./04-digital-twin-graph.md)
- [05. Warehouse-AI Integration](./05-warehouse-ai-integration.md)
- [06. Multi-user Isolation](./06-multi-user-isolation.md)

---

[← 02. Backend Design](./02-backend-design.md) · [README](../README.md)
