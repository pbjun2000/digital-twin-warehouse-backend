# 06. Multi-user Isolation

> 여러 사용자가 동일한 Warehouse Template을 사용하더라도  
> Inventory · Robot · Scenario · Graph 등 실행 데이터가 서로 영향을 주지 않도록 사용자별 Warehouse 환경을 분리했습니다.

---

## 1. Shared Warehouse의 문제

초기 구조에서는 여러 사용자가 하나의 Shared Warehouse를 사용할 수 있었습니다.

```text
             Shared Warehouse
          ↙        ↓        ↘
      User A    User B     Guest
```

창고 구조를 조회하는 것만으로는 문제가 없지만,
Simulation이 시작되면 Warehouse에 연결된 상태가 계속 변경됩니다.

```text
Inventory
Robot
Task
Scenario
Simulation State
```

동일한 Warehouse를 여러 사용자가 실행하면
한 사용자의 재고나 Robot 상태 변화가
다른 사용자의 실행 환경에도 영향을 줄 수 있습니다.

이를 해결하기 위해 Shared Warehouse를
직접 실행하는 환경이 아닌 **Template**으로 사용하도록 구조를 변경했습니다.

---

## 2. Personal Warehouse 구조

사용자가 Shared Warehouse를 선택하면
해당 Template을 기반으로 개인 실행 환경을 생성합니다.

```text
              Shared Template
                    │
       ┌────────────┼────────────┐
       ▼            ▼            ▼
   User A        User B        Guest
       │            │            │
Personal WH A  Personal WH B  Personal WH C
```

실제 Simulation은 Shared Warehouse가 아니라
각 사용자의 Personal Warehouse를 대상으로 실행합니다.

`Warehouse`에는 실행 환경을 구분하기 위한 정보를 함께 관리합니다.

```text
shared
user
guestSessionId
sourceTemplate
```

USER와 GUEST는 각각 다음 기준으로
Personal Warehouse를 구분합니다.

```text
USER
user_id
+
source_template_id
```

```text
GUEST
guest_session_id
+
source_template_id
```

동일한 사용자 또는 Guest가
같은 Template에 대한 Personal Warehouse를 중복 생성하지 않도록
해당 조합에는 Unique Constraint를 적용했습니다.

---

## 3. USER / GUEST 소유 관계

회원과 Guest의 인증 방식 자체는
프로젝트의 공통 인증 영역에서 처리합니다.

Warehouse 영역에서는 인증 계층에서 확인된 요청자 정보를 이용해
Personal Warehouse의 소유 관계를 구분했습니다.

```text
Authenticated Request
        ↓
요청자 정보
        ↓
 ┌────────────────┬──────────────────┐
 │ USER           │ GUEST            │
 │ userId         │ guestSessionId   │
 └────────────────┴──────────────────┘
        ↓
Warehouse Ownership
```

USER는 `userId`,
GUEST는 `guestSessionId`를 기준으로
자신의 Personal Warehouse와 연결됩니다.

Personal Warehouse 생성 API도 요청자 유형에 따라 분리했습니다.

```http
POST /api/warehouses/{templateWarehouseId}/personal-copy
```

```http
POST /api/warehouses/{templateWarehouseId}/guest-personal-copy
```

클라이언트가 소유자 ID를 직접 지정하는 대신
현재 인증된 요청자의 정보를 기준으로
Personal Warehouse가 생성되도록 구성했습니다.

---

## 4. Warehouse Deep Clone

Warehouse Entity 하나만 복제해서는
사용자별 실행 상태를 분리할 수 없습니다.

예를 들어 Robot이나 Inventory를 Template과 계속 공유하면
Personal Warehouse를 만들어도 상태 충돌이 발생합니다.

따라서 `WarehouseTemplateCloneService`에서
Warehouse에 종속된 실행 데이터를 함께 복제합니다.

```text
Shared Warehouse
       ↓
WarehouseTemplateCloneService
       ↓
Personal Warehouse

├─ Zone
├─ Node
├─ Edge
├─ ChargingStation
├─ StorageLocation
├─ WarehouseItem
├─ Robot
└─ Scenario
```

복제된 Entity에는 새로운 DB ID가 생성되므로
원본 Entity의 참조 관계를 그대로 사용할 수 없습니다.

예를 들어 Node를 먼저 복제한 뒤,
Edge · Storage · ChargingStation 등이
새 Personal Warehouse의 Node를 참조하도록 관계를 다시 연결합니다.

```text
Template Node
      ↓
Personal Node
      ↓
Edge / Storage / ChargingStation
참조 관계 재구성
```

이를 통해 Template과 실행 데이터를 공유하지 않는
독립적인 Warehouse 환경을 구성했습니다.

### Deep Clone을 선택한 이유

사용자별로 Runtime State만 별도로 관리하고
Warehouse의 기준 구조를 공유하는 방식도 고려할 수 있었습니다.

하지만 2개월 MVP에서는

- 사용자 간 실행 상태 격리를 명확하게 보장할 수 있고
- Inventory · Robot · Scenario 등 서로 연관된 실행 데이터를
  하나의 Personal Warehouse Scope 안에서 관리할 수 있으며
- 구현과 테스트 시 사용자별 데이터 경계를 확인하기 쉽다는 점

을 우선했습니다.

따라서 Shared Warehouse를 Template으로 두고,
USER / GUEST별 Personal Warehouse를 Deep Clone하는 방식을 적용했습니다.

### Trade-off

Deep Clone은 사용자별 실행 환경을 명확하게 분리할 수 있다는 장점이 있지만,
Warehouse 데이터 규모와 사용자 수가 증가할수록

- 데이터 복제 비용
- 저장 공간 사용량
- Personal Warehouse 생성 시간

이 증가할 수 있습니다.

현재 프로젝트는 2개월 MVP이기 때문에
확실한 상태 격리를 우선했지만,

서비스 규모가 커질 경우에는
공통 Warehouse Template은 공유하면서

**Robot 위치·배터리·재고·Simulation 상태와 같은 Runtime State만 사용자별로 분리하거나,
변경이 발생한 데이터만 복제하는 Copy-on-write 방식**을 검토할 수 있습니다.

---

## 5. Shared Template 실행 차단과 소유권 검증

Personal Warehouse를 제공하더라도
클라이언트가 Shared Warehouse ID나
다른 사용자의 Warehouse ID를 직접 요청할 수 있습니다.

따라서 Simulation 실행 경계에서는
Warehouse의 상태와 소유 관계를 다시 확인합니다.

```text
Simulation 실행 요청
        ↓
Warehouse 조회
        ↓
Shared Template?
   ├─ YES → 실행 차단
   └─ NO
        ↓
현재 요청자의 Warehouse?
   ├─ YES → 실행
   └─ NO  → 접근 차단
```

이를 통해 다음과 같은 접근을 제한합니다.

```text
User A → User B Warehouse

Guest A → Guest B Warehouse

USER / GUEST
→ 자신이 소유하지 않은 Personal Warehouse
```

단순히 인증된 사용자라는 이유만으로 실행을 허용하지 않고,
**요청자가 실제로 해당 Warehouse를 사용할 수 있는지 Resource 단위로 검증**하도록 구성했습니다.

---

## 6. Scenario도 Personal Warehouse 단위로 분리

Scenario 역시 Warehouse에 연결된 실행 데이터이기 때문에
Personal Warehouse 생성 과정에서 함께 복제합니다.

```text
Shared Warehouse
 └─ Scenario

        ↓ Deep Clone

Personal Warehouse
 └─ Personal Scenario
```

복제 과정에서 새로운 Scenario ID가 생성되므로
Personal Warehouse에서는
원본 Template Scenario가 아닌 복제된 Scenario를 사용해야 합니다.

```text
Template Scenario
        ↓
Personal Warehouse Clone
        ↓
Personal Scenario
        ↓
Simulation
```

Simulation 실행 시 Personal Warehouse에 복제된 Scenario를 사용하도록 구성했습니다.

---

## 7. Personal Warehouse별 Neo4j Graph

PostgreSQL 데이터만 사용자별로 복제하고
Neo4j에서 동일한 Graph를 사용하면
Digital Twin의 이동 관계는 여전히 공유됩니다.

따라서 Personal Warehouse 생성이 완료되면
새로운 Warehouse ID를 기준으로 Graph Sync를 수행합니다.

```text
Personal Warehouse 생성
        ↓
PostgreSQL COMMIT
        ↓
WarehouseGraphChangedEvent
        ↓
GraphSyncService
        ↓
Personal Warehouse Graph
```

결과적으로 사용자별 Warehouse와 Graph가
각각 독립적인 Scope를 갖습니다.

```text
User A
Personal Warehouse A
        ↓
Neo4j Graph A


User B
Personal Warehouse B
        ↓
Neo4j Graph B
```

Graph 동기화 구조는  
[04. Digital Twin Graph](./04-digital-twin-graph.md)에서 설명합니다.

---

## 8. 전체 구조

Shared Warehouse를 선택한 뒤
실행 환경이 구성되는 흐름은 다음과 같습니다.

```text
USER / GUEST 인증
        ↓
Shared Template 선택
        ↓
요청자 식별
        ↓
Personal Warehouse 생성 또는 조회
        ↓
Warehouse 종속 데이터 Deep Clone
        ↓
Personal Graph Sync
        ↓
Warehouse Ownership 검증
        ↓
Simulation 실행
```

최종적으로 Shared Warehouse는
공통으로 사용할 수 있는 **초기 Template** 역할을 담당하고,

```text
Inventory
Robot
Scenario
Graph
Simulation State
```

등 실행에 영향을 받는 데이터는
Personal Warehouse를 기준으로 분리합니다.

---

## 9. 주요 Component

| Component | 역할 |
|---|---|
| `WarehouseTemplateCloneService` | Shared Template 기반 Personal Warehouse Deep Clone |
| `Warehouse` | Shared 여부와 USER / GUEST 소유 관계 관리 |
| `WarehouseService` | Personal Warehouse 생성 및 Warehouse 정책 처리 |
| `SimulationRunService` | Simulation 실행 시 Shared Warehouse 차단 및 소유권 검증 |
| `WarehouseGraphChangedEvent` | Personal Warehouse 생성 후 Graph Sync Trigger |
| `GraphSyncService` | Warehouse Scope 기준 Neo4j Graph 동기화 |

인증 영역에서 제공되는 요청자 정보를 활용해
Personal Warehouse의 소유 관계를 구분하고,
Simulation 실행 시 실제 Warehouse 소유권을 다시 검증하도록 구성했습니다.

Multi-user Isolation에서 중점을 둔 부분은
Guest 로그인 기능 자체가 아니라,

**동일한 Warehouse Template에서 시작하더라도  
실제 Digital Twin 실행 데이터는 사용자별로 공유하지 않도록  
Warehouse Resource의 경계를 분리하는 것**이었습니다.
---

## 관련 문서

- [01. Project Overview](./01-project-overview.md)
- [02. Backend Design](./02-backend-design.md)
- [03. Warehouse Domain](./03-warehouse-domain.md)
- [04. Digital Twin Graph](./04-digital-twin-graph.md)
- [05. Warehouse-AI Integration](./05-warehouse-ai-integration.md)

---

[← 05. Warehouse-AI Integration](./05-warehouse-ai-integration.md) · [README](../README.md)
