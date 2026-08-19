# 03. Warehouse Domain

> Warehouse Domain은 Frontend Editor, AI Planning, Simulation이  
> 동일한 Digital Twin Warehouse 데이터를 활용할 수 있도록 창고의 공간 구조와 이동 관계를 관리합니다.

---

## 1. Domain 구성

창고에서 Robot이 이동하고 작업을 수행하려면
창고의 기본 정보뿐 아니라 구역, 이동 지점, 이동 경로와 설비 정보가 함께 필요합니다.

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

`Warehouse`를 Root로 두고
Frontend에서 편집하는 지도와 AI Planning·Simulation에서 활용하는 지도가
같은 Backend 데이터를 기준으로 동작하도록 구성했습니다.

### Warehouse

Warehouse에는 기본 정보와 함께
실행 환경을 구분하기 위한 정보를 관리합니다.

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

Shared Warehouse는 기본 Template으로 사용하고,
실제 사용자 Simulation은 Personal Warehouse에서 실행합니다.

사용자별 Warehouse 분리 구조는  
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

각 Zone은 `minX / maxX / minY / maxY` 좌표 범위를 가지며
창고 내 작업 영역을 데이터 수준에서 구분합니다.

---

## 2. Node와 Edge

### Node

`WarehouseNode`는 Robot 이동과 작업의 기준이 되는 지점입니다.

Node를 단순 좌표로만 관리하지 않고
역할에 따라 Type을 구분했습니다.

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

예를 들면,

```text
ROUTE
→ 일반 이동 지점

RACK_STORAGE
→ 재고가 위치하는 Rack

RACK_ACCESS
→ Robot이 Rack 작업을 수행하기 위한 접근 지점

CHARGING_SLOT
→ Robot 충전 위치
```

처럼 같은 좌표 데이터라도
역할에 따라 서로 다른 의미를 갖도록 구성했습니다.

### Routing 속성

경로 계획에서 자주 사용하는 값은 별도의 Column으로 관리합니다.

```text
serviceOnly
transitAllowed
holdingAllowed
nodeCapacity

resourceType
resourceCode
side
```

예를 들어 `transitAllowed`는 해당 Node를 통과 경로로 사용할 수 있는지,
`holdingAllowed`는 Robot이 해당 위치에서 대기할 수 있는지를 표현합니다.

추가적인 Map Metadata는 `route_attributes` JSONB에 저장합니다.

```java
@JdbcTypeCode(SqlTypes.JSON)
@Column(name = "route_attributes", columnDefinition = "jsonb")
private Map<String, Object> routeAttributes;
```

경로 계산에 필요한 핵심 값은 Typed Column으로 유지하고,
확장 가능성이 높은 Metadata만 JSON으로 분리했습니다.

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

거리뿐 아니라 속도 제한, 예상 이동 시간, Cost를 함께 관리하여
Warehouse 이동 구조에서 사용할 수 있는 이동 조건을 표현했습니다.

---

## 3. DB PK와 Map 식별자 분리

JPA Entity 내부에서는 Numeric PK를 사용하지만
Frontend와 AI Planning에 제공하는 Warehouse Map에서는 별도의 Code를 사용합니다.

```text
Database
Node PK = 351
Edge PK = 824

        ↓

Digital Twin Map
Node Code = R0_0
Edge Code = H0_0
```

전체 구조는 다음과 같습니다.

```text
PostgreSQL
Numeric PK
    ↓
Spring Boot
nodeCode / edgeCode
    ↓
Frontend / AI Planning
```

DB 내부 식별자와 외부 Map 식별자를 분리하여
Frontend나 AI가 DB PK에 직접 의존하지 않고
Warehouse Map을 사용할 수 있도록 했습니다.

Warehouse Editor에서 Layout을 수정할 때도
`nodeCode`와 `edgeCode`를 기준으로 기존 데이터를 식별합니다.

---

## 4. JSON 기반 Warehouse Import

Frontend에서 직접 Warehouse를 편집하는 것 외에도
JSON Map을 한 번에 Import할 수 있도록 구성했습니다.

```http
POST /api/warehouses/import
```

요청은 크게 다음 구조를 가집니다.

```text
Warehouse 기본 정보
        +
Map
 ├─ Nodes
 └─ Edges
```

Backend에서는 Node와 Edge만 저장하지 않고
Warehouse에서 사용하는 관련 Resource도 함께 구성합니다.

```text
Warehouse JSON
        ↓
Warehouse 생성
        ↓
Node / Zone / Edge 생성
        ↓
ChargingStation
StorageLocation
        ↓
Initial Inventory
Robot
        ↓
WarehouseGraphChangedEvent
```

Import 이후 생성된 Warehouse 구조를
Editor와 Simulation에서 사용할 수 있도록 구성했습니다.

---

## 5. Warehouse Editor 수정 처리

기존 Warehouse Layout 수정은 다음 API를 사용합니다.

```http
PUT /api/warehouses/{warehouseId}/layout
```

수정할 때 기존 Node와 Edge를 모두 삭제한 뒤 다시 만들지 않고,
요청 데이터와 현재 데이터를 Code 기준으로 비교합니다.

```text
현재 Layout
      +
수정 Layout
      ↓
nodeCode / edgeCode 비교
      ↓
┌────────────────────────┐
│ 기존 Code     → UPDATE │
│ 새로운 Code   → INSERT │
│ 제거된 Code   → REMOVE │
└────────────────────────┘
```

예를 들어 Node의 좌표만 수정된 경우
새로운 Node를 만드는 대신 기존 Entity를 갱신합니다.

이 방식으로 Layout 변경 과정에서도
기존 Resource와 연결된 식별 관계를 최대한 유지했습니다.

---

## 6. Node 삭제와 실행 이력 보존

Node는 단독 Resource가 아니라
여러 Domain에서 참조됩니다.

```text
Node
 ├─ StorageLocation
 ├─ ChargingStation
 ├─ Robot
 └─ Task
```

따라서 삭제 요청이 들어왔다고 바로 물리 삭제하지 않습니다.

현재 Storage, Charging Station, Robot에서 사용하는 Node이거나
실행 중인 Task가 참조하고 있다면 삭제를 제한합니다.

```text
Node 삭제 요청
      ↓
참조 Resource 확인
      ↓
참조 중?
 ├─ YES → 삭제 차단
 └─ NO  → 제거 처리
```

### Node Retire

과거 Task에서 사용한 Node를 물리적으로 삭제하면
Simulation 이력과 연결된 Foreign Key 관계가 깨질 수 있습니다.

이를 위해 `WarehouseNode`에는 Active 상태를 두었습니다.

```java
public void activate() {
    this.active = true;
}

public void retire() {
    this.active = false;
}
```

현재 Map에서 제거된 Node는 필요한 경우

```text
DELETE
```

대신

```text
active = false
```

상태로 유지합니다.

```text
과거 Task / Simulation
        ↓
Retired Node 유지

현재 Warehouse Map
        ↓
Active Node 사용
```

현재 Warehouse Map에서의 사용 여부와
과거 실행 데이터의 참조 관계를 분리하여 관리했습니다.

---

## 7. Warehouse 조회 구조

### Layout API

Frontend가 Warehouse 화면을 구성할 때
Zone, Node, Edge, Robot 등을 각각 호출하지 않도록
통합 조회 API를 제공합니다.

```http
GET /api/warehouses/{warehouseId}/layout
```

응답은 다음 데이터를 포함합니다.

```text
WarehouseLayoutResponse
 ├─ Warehouse
 ├─ Zones
 ├─ Nodes
 ├─ Edges
 ├─ ChargingStations
 └─ Robots
```

Warehouse Editor와 Simulation 화면에서 필요한
동일 Warehouse Scope의 데이터를 한 번에 조회할 수 있도록 구성했습니다.

---

### Graph API

AI Planning에서 현재 Warehouse의 이동 구조를 활용할 수 있도록
별도의 Graph 조회 API를 제공합니다.

```http
GET /api/warehouses/{warehouseId}/graph
```

```text
WarehouseGraphResponse
 ├─ warehouseId
 ├─ warehouseName
 ├─ mapVersion
 ├─ nodeCount
 ├─ edgeCount
 ├─ nodes[]
 └─ edges[]
```

Node와 Edge는 DB Numeric PK 대신
`nodeCode`, `edgeCode` 기준으로 반환합니다.

```text
PostgreSQL Warehouse
        ↓
WarehouseGraphService
        ↓
WarehouseGraphResponse
        ├─ Frontend
        └─ AI Planning
```

Backend에서 관리하는 Warehouse 구조를
Frontend와 AI Planning에서 공통으로 활용할 수 있도록 제공했습니다.

---

## 8. Map Version

Graph Response에는 현재 Warehouse Map을 식별하기 위한
`mapVersion`을 함께 제공합니다.

```text
MAP-{warehouseId}-{nodeCount}-{edgeCount}-{checksum}
```

Version 값을 직접 증가시키는 방식이 아니라
현재 Active Node와 Edge 내용을 기반으로 Signature를 생성합니다.

```text
Node / Edge 변경
        ↓
Graph Signature 변경
        ↓
Checksum 변경
        ↓
mapVersion 변경
```

따라서 Map 내용이 변경되면 Version도 함께 달라집니다.

현재 Backend의 Warehouse Map과
외부에서 사용 중인 Map을 구분할 수 있는 식별 정보로 활용할 수 있습니다.

AI Planning에서의 Warehouse Graph 활용 방식은  
[05. Warehouse-AI Integration](./05-warehouse-ai-integration.md)에서 설명합니다.

---

## 9. 주요 API

### Warehouse

| Method | Endpoint | 역할 |
|---|---|---|
| `GET` | `/api/warehouses` | 접근 가능한 Warehouse 조회 |
| `GET` | `/api/warehouses/{id}` | Warehouse 상세 조회 |
| `POST` | `/api/warehouses` | Warehouse 생성 |
| `PATCH` | `/api/warehouses/{id}` | Warehouse 정보 수정 |
| `DELETE` | `/api/warehouses/{id}` | Warehouse 삭제 |
| `POST` | `/api/warehouses/import` | JSON Map Import |
| `GET` | `/api/warehouses/{id}/layout` | Layout 통합 조회 |
| `PUT` | `/api/warehouses/{id}/layout` | Layout 수정 |
| `GET` | `/api/warehouses/{id}/graph` | Node·Edge Graph 조회 |

### Warehouse Resource

```text
/api/warehouse-zones
/api/warehouse-nodes
/api/warehouse-edges
```

Zone, Node, Edge는 각각
생성·조회·수정·삭제 API를 제공합니다.

---

## 10. 주요 Component

| Component | 역할 |
|---|---|
| `WarehouseService` | Warehouse 기본 정보 및 정책 관리 |
| `WarehouseImportService` | JSON Map 기반 Warehouse 생성 |
| `WarehouseLayoutService` | Layout 통합 조회 및 수정 |
| `WarehouseGraphService` | Node·Edge Graph 생성 및 조회 |
| `WarehouseNodeService` | Node 관리 및 참조 관계 검증 |
| `WarehouseEdgeService` | Edge 및 이동 속성 관리 |
| `WarehouseZoneService` | Warehouse Zone 관리 |
| `WarehouseTemplateCloneService` | Personal Warehouse 생성 |

Warehouse Domain에서는 단순 CRUD뿐 아니라
**하나의 Warehouse 데이터를 Editor · AI Planning · Simulation에서 공통으로 사용할 수 있는 구조를 만드는 것**을 중심으로 구현했습니다.

---

## 관련 문서

- [01. Project Overview](./01-project-overview.md)
- [02. Backend Design](./02-backend-design.md)
- [04. Digital Twin Graph](./04-digital-twin-graph.md)
- [05. Warehouse-AI Integration](./05-warehouse-ai-integration.md)
- [06. Multi-user Isolation](./06-multi-user-isolation.md)

---

[← 02. Backend Design](./02-backend-design.md) · [README](../README.md)
