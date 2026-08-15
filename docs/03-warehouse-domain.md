# 03. Warehouse Domain

> LARO의 Warehouse Domain은 단순한 창고 정보 CRUD가 아니라  
> **Frontend Editor, AI Planning, Simulation이 동일한 Digital Twin 지도를 사용하도록 공간 구조와 이동 관계를 관리하는 영역**입니다.

---

## 1. 설계 목표

창고 운영에서는 단순히 창고의 이름과 크기만 저장하는 것으로는 부족합니다.

로봇이 실제로 이동하고 작업을 수행하려면 다음 정보가 함께 필요합니다.

- 창고의 구역
- 이동 가능한 지점
- 지점 간 이동 경로
- 입·출고 설비
- Storage
- Charging Station
- Robot

이를 위해 Warehouse를 중심으로 공간 구조를 다음과 같이 모델링했습니다.

```text
Warehouse
 ├─ Zone
 ├─ Node
 │   ├─ Route
 │   ├─ Rack
 │   ├─ Inbound / Outbound
 │   └─ Charging Slot
 │
 ├─ Edge
 │   └─ Node ↔ Node
 │
 ├─ Storage Location
 ├─ Charging Station
 └─ Robot
```

Frontend에서 구성한 창고와 AI가 사용하는 지도,
Simulation에서 로봇이 움직이는 지도를 각각 따로 관리하지 않고
**Backend의 Warehouse 데이터를 공통 기준으로 사용하도록 설계했습니다.**

---

## 2. Warehouse

`Warehouse`는 창고 전체의 Root Entity입니다.

주요 정보는 다음과 같습니다.

```text
Warehouse
├─ id
├─ name
├─ width / height
├─ location
├─ description
├─ status
├─ shared
├─ user
├─ guestSessionId
└─ sourceTemplate
```

### Warehouse Status

창고 운영 상태를 구분합니다.

```text
ACTIVE
MAINTENANCE
INACTIVE
```

### Shared / Personal Warehouse

공유 창고와 실제 사용자 실행 환경을 구분하기 위해
Warehouse 자체에 소유 관계를 포함했습니다.

```text
Shared Warehouse
      ↓
sourceTemplate
      ↓
Personal Warehouse
      ├─ user_id
      └─ guest_session_id
```

공유 Warehouse는 기본 Template 역할을 하며,
직접 수정하거나 Simulation 실행에 사용하는 것이 아니라
사용자별 Personal Warehouse 생성의 기준으로 활용됩니다.

> USER / GUEST별 실행 환경 격리 구조는  
> [06. Multi-user Isolation](./06-guest-access.md)에서 상세히 설명합니다.

---

## 3. Zone

`WarehouseZone`은 창고 내부 영역의 의미를 구분합니다.

```text
STORAGE
MOVING
INBOUND
OUTBOUND
CHARGING
RESTRICTED
```

각 Zone은 다음 좌표 범위를 갖습니다.

```text
minX
maxX
minY
maxY
```

이를 통해 단순한 화면상의 영역 표시가 아니라
Warehouse 내부 공간의 역할을 데이터로 구분할 수 있도록 했습니다.

```text
Warehouse
   ↓
Zone
   ├─ STORAGE
   ├─ MOVING
   ├─ INBOUND
   ├─ OUTBOUND
   └─ CHARGING
```

---

## 4. Node

`WarehouseNode`는 창고에서 로봇의 이동과 작업의 기준이 되는 지점입니다.

단순 좌표만 저장하지 않고
각 Node가 어떤 역할을 갖는지 명시적으로 구분했습니다.

### Node Type

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
→ 로봇이 Rack에 접근하는 지점

CHARGING_SLOT
→ Robot 충전 위치
```

처럼 역할이 다릅니다.

---

## 5. Routing 속성 분리

Node는 좌표와 Type 외에도
경로 계획에 필요한 속성을 별도 Column으로 관리합니다.

```text
serviceOnly
transitAllowed
holdingAllowed
nodeCapacity
resourceType
resourceCode
side
```

예를 들어,

```text
transitAllowed
→ 다른 목적 없이 통과 가능한가?

holdingAllowed
→ 해당 Node에서 대기할 수 있는가?

nodeCapacity
→ 동시에 몇 개의 Robot이 점유 가능한가?

resourceCode
→ 어떤 Rack / Station 등의 설비와 연결되는가?
```

와 같은 정보를 표현합니다.

추가적인 타입별 Metadata는 `route_attributes` JSONB에 저장합니다.

```java
@JdbcTypeCode(SqlTypes.JSON)
@Column(name = "route_attributes", columnDefinition = "jsonb")
private Map<String, Object> routeAttributes;
```

핵심 Routing 속성은 Typed Column으로 관리하고,
확장 속성만 JSON으로 분리함으로써
**필수 데이터의 구조는 유지하면서 Map Schema의 확장성을 확보**했습니다.

---

## 6. Edge

`WarehouseEdge`는 두 Node 사이에서
Robot이 이동할 수 있는 연결 관계를 표현합니다.

```text
Node A
  │
  │ Edge
  ▼
Node B
```

Edge에는 단순 연결 관계 외에도
실제 이동 계획에 필요한 속성을 포함합니다.

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

### Direction

```text
BOTH
A_TO_B
B_TO_A
```

양방향·단방향 이동을 데이터 자체에서 구분할 수 있도록 했습니다.

### 이동 Cost

기본적으로 거리 정보를 사용하지만
Edge별 Cost와 이동 시간을 별도로 관리할 수 있습니다.

```text
Distance
Speed Limit
Travel Time
Cost
```

이 정보는 이후 AI Planning과 경로 계산에서
Warehouse의 이동 조건을 표현하는 근거 데이터로 활용됩니다.

---

## 7. 숫자 PK와 Graph Code 분리

DB 내부에서는 JPA Entity의 숫자 PK를 사용하지만,
Frontend와 AI에 Graph를 전달할 때는 숫자 ID를 직접 사용하지 않습니다.

Node와 Edge에 각각 별도의 Code를 두었습니다.

```text
DB PK

Node ID = 351
Edge ID = 824
```

외부에서는:

```text
Node Code = R0_0
Edge Code = H0_0
```

와 같이 사용합니다.

```text
PostgreSQL
Numeric PK
    ↓
Backend
nodeCode / edgeCode
    ↓
Frontend / AI
```

이를 통해 DB의 내부 ID와
Digital Twin 지도에서 사용하는 식별자를 분리했습니다.

또한 창고 레이아웃 수정에서도 `nodeCode`와 `edgeCode`를 기준으로
기존 Entity를 찾아 갱신합니다.

따라서 단순히 화면에서 Node 위치를 이동했다고 해서
기존 DB PK와 연결 데이터가 새로 생성되지 않도록 구성했습니다.

---

## 8. Warehouse JSON Import

Frontend에서는 창고를 직접 구성하는 것뿐 아니라
JSON 형식의 Warehouse Map을 Import할 수 있습니다.

API:

```http
POST /api/warehouses/import
```

Import 요청에는 다음 정보가 포함됩니다.

```text
Warehouse 기본 정보
        +
Map
 ├─ Nodes
 └─ Edges
```

Backend에서는 Map을 받아
Warehouse 실행 환경에 필요한 여러 데이터를 함께 구성합니다.

```text
Warehouse JSON
        ↓
Warehouse 생성
        ↓
Node 생성
        ↓
Zone 생성
        ↓
Edge 생성
        ↓
Charging Station
        ↓
Storage Location
        ↓
Initial Inventory
        ↓
Robot
        ↓
WarehouseGraphChangedEvent
```

단순히 Node와 Edge만 저장하는 것이 아니라
Simulation에서 바로 사용할 수 있는 Warehouse 데이터 구조까지
한 번에 구성하도록 구현했습니다.

---

## 9. Warehouse Editor 수정 처리

Frontend Warehouse Editor에서 기존 창고를 수정하는 경우에는

```http
PUT /api/warehouses/{warehouseId}/layout
```

API를 사용합니다.

단순히 기존 Node와 Edge를 모두 삭제한 뒤 다시 생성하지 않고,
**Node Code와 Edge Code를 기준으로 기존 데이터와 요청 데이터를 비교**합니다.

```text
기존 Warehouse Layout
        +
수정 요청 Layout
        ↓
Code 기준 비교
        ↓
 ┌───────────────┐
 │ 기존 Code 존재 │ → UPDATE
 ├───────────────┤
 │ 새로운 Code    │ → INSERT
 ├───────────────┤
 │ 요청에서 제거  │ → DELETE / RETIRE
 └───────────────┘
```

이 방식으로 Editor에서 좌표나 속성을 변경해도
기존 Resource와의 연결을 최대한 유지할 수 있도록 했습니다.

---

## 10. Node 삭제 시 참조 관계 보호

Warehouse Node는 다양한 서비스 데이터에서 참조될 수 있습니다.

예를 들어:

```text
Storage Location
Charging Station
Robot
Task
```

따라서 Node 삭제 요청이 들어왔다고 바로 물리 삭제하지 않습니다.

먼저 현재 Node를 참조하는 Resource가 있는지 확인합니다.

```text
Node 삭제 요청
      ↓
Storage 참조?
Charging Station 참조?
Robot 위치 참조?
실행 중 Task 참조?
      ↓
참조 존재 → 삭제 차단
```

특히 실행 중인 Task가 Node를 참조하는 경우
Node 제거를 제한합니다.

---

## 11. Node Retire 방식

이미 과거 Task에서 사용된 Node를 DB에서 완전히 삭제하면
기존 실행 이력의 Foreign Key 관계가 깨질 수 있습니다.

이를 위해 `WarehouseNode`에는 `active` 상태가 존재합니다.

```java
public void activate() {
    this.active = true;
}

public void retire() {
    this.active = false;
}
```

현재 지도에서 제거된 Node는

```text
DELETE
```

가 아니라

```text
active = false
```

로 Retire할 수 있도록 구성했습니다.

```text
과거 실행 이력
      ↓
Node 유지

현재 Warehouse Map
      ↓
active = true Node만 사용
```

`WarehouseLayoutService`와 `WarehouseGraphService`에서도
현재 운영 지도에는 Active Node와 연결된 Edge만 반환합니다.

이를 통해 **운영 지도 변경과 과거 실행 이력 보존을 함께 고려**했습니다.

---

## 12. Warehouse Layout 통합 조회

Frontend가 창고 화면을 구성하기 위해
Zone, Node, Edge, Robot 등을 각각 호출하도록 하지 않고
Warehouse Layout 통합 조회 API를 제공합니다.

```http
GET /api/warehouses/{warehouseId}/layout
```

응답 구조:

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
 ├─ Warehouse
 ├─ Zone
 ├─ Node
 ├─ Edge
 ├─ Charging Station
 └─ Robot
    ↓
Warehouse Editor / Simulation
```

창고 시각화에 필요한 데이터를 하나의 응답으로 제공하여
Frontend가 동일 Warehouse Scope의 데이터를 한 번에 구성할 수 있도록 했습니다.

---

## 13. AI와 공유하는 Warehouse Graph

AI Planning에서도 Frontend와 동일한 Warehouse Map을 사용할 수 있도록
Graph 조회 API를 별도로 제공합니다.

```http
GET /api/warehouses/{warehouseId}/graph
```

Graph Response에는 다음 정보가 포함됩니다.

```text
warehouseId
warehouseName

mapVersion

nodeCount
edgeCount

nodes[]
edges[]
```

Node와 Edge는 Numeric PK가 아닌
`nodeCode`, `edgeCode` 기준으로 반환합니다.

```text
PostgreSQL Warehouse Map
          ↓
WarehouseGraphService
          ↓
WarehouseGraphResponse
          ├─ Frontend
          └─ AI Planning
```

이를 통해 Frontend와 AI가 서로 다른 지도 복사본을 관리하면서
구조가 달라지는 문제를 줄였습니다.

---

## 14. Map Version

Warehouse Graph에는 `mapVersion`을 함께 제공합니다.

형식:

```text
MAP-{warehouseId}-{nodeCount}-{edgeCount}-{checksum}
```

별도의 Version Column을 수동으로 증가시키는 방식이 아니라
현재 Node와 Edge의 내용을 기반으로 Version을 계산합니다.

```text
Node / Edge 변경
        ↓
Graph Signature 변경
        ↓
Checksum 변경
        ↓
mapVersion 변경
```

AI가 계획을 생성한 시점의 Map과
현재 Warehouse Map이 같은지 확인할 수 있는 기준으로 활용할 수 있도록 설계했습니다.

---

## 15. 주요 API

### Warehouse

| Method | Endpoint | 설명 |
|---|---|---|
| `GET` | `/api/warehouses` | 접근 가능한 Warehouse 목록 |
| `GET` | `/api/warehouses/{id}` | Warehouse 상세 조회 |
| `POST` | `/api/warehouses` | 빈 Warehouse 생성 |
| `PATCH` | `/api/warehouses/{id}` | Warehouse 정보 수정 |
| `DELETE` | `/api/warehouses/{id}` | Warehouse 삭제 |
| `POST` | `/api/warehouses/import` | JSON 기반 Warehouse 생성 |
| `GET` | `/api/warehouses/{id}/layout` | Warehouse Layout 통합 조회 |
| `PUT` | `/api/warehouses/{id}/layout` | Warehouse Layout 갱신 |
| `GET` | `/api/warehouses/{id}/graph` | AI / Graph용 Warehouse Map 조회 |

### Warehouse Graph

```text
/api/warehouse-zones
/api/warehouse-nodes
/api/warehouse-edges
```

각 Domain에 대해 생성·조회·수정·삭제 API를 제공합니다.

---

## 16. 주요 Component

| Component | 역할 |
|---|---|
| `WarehouseService` | Warehouse 기본 정보 관리 |
| `WarehouseImportService` | JSON Map Import 및 Layout 갱신 |
| `WarehouseLayoutService` | Frontend용 Warehouse 통합 조회 |
| `WarehouseGraphService` | AI/Graph용 Node·Edge Map 제공 |
| `WarehouseNodeService` | Node 관리 및 삭제 참조 검증 |
| `WarehouseEdgeService` | Edge 및 이동 속성 관리 |
| `WarehouseZoneService` | 창고 영역 관리 |
| `WarehouseTemplateCloneService` | Personal Warehouse 복제 |

---

## 17. 설계 요약

Warehouse Domain에서 중점적으로 고려한 부분은 다음과 같습니다.

### 하나의 Warehouse Map을 서비스 전체가 공유

```text
Frontend
     ↘
    Backend Warehouse Map
     ↗
AI Planning
```

Frontend와 AI가 서로 다른 지도를 갖지 않도록
Backend를 공통 데이터 경계로 사용했습니다.

### DB ID와 지도 식별자 분리

```text
Database
Numeric PK

Digital Twin
Node Code / Edge Code
```

Editor 수정과 AI 연동에서 안정적인 식별자를 유지합니다.

### 현재 지도와 과거 실행 데이터 분리

```text
Current Map
→ Active Node

Historical Data
→ Retired Node 유지
```

지도 변경 때문에 기존 Task와 Simulation 이력이 깨지지 않도록 고려했습니다.

---

## 다음 문서

- [04. Digital Twin Graph](./04-digital-twin-graph.md)
- [05. AI Integration](./05-ai-integration.md)
- [06. Multi-user Isolation](./06-guest-access.md)

---

[← 02. Backend Design](./02-backend-design.md) · [README](../README.md)
