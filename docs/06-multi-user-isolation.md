# 06. Multi-user Isolation

> 여러 사용자가 동일한 Warehouse Template을 사용하더라도  
> Inventory · Robot · Scenario · Simulation 상태가 서로 영향을 주지 않도록 사용자별 실행 환경을 분리했습니다.

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
Robot 위치 / 상태
Task
Scenario
Simulation 실행 상태
```

따라서 동일한 Warehouse를 여러 사용자가 실행하면
한 사용자의 상태 변화가 다른 사용자의 Simulation에 영향을 줄 수 있습니다.

이를 해결하기 위해 Shared Warehouse를
직접 실행하는 공간이 아닌 **Template**으로 사용하도록 구조를 변경했습니다.

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

`Warehouse`에는 이를 구분하기 위한 정보를 함께 관리합니다.

```text
shared
user
guestSessionId
sourceTemplate
```

### USER

```text
user_id
+
source_template_id
```

### GUEST

```text
guest_session_id
+
source_template_id
```

를 기준으로 Personal Warehouse를 구분합니다.

동일한 사용자가 같은 Template을 반복 선택할 경우를 고려해
해당 조합에는 Unique Constraint도 적용했습니다.

---

## 3. USER / GUEST 식별

회원과 Guest는 서로 다른 식별 방식을 사용합니다.

```text
Authenticated Request
        ↓
AuthenticatedRequesterResolver
        ↓
 ┌────────────────┬──────────────────┐
 │ USER           │ GUEST            │
 │ userId         │ guestSessionId   │
 └────────────────┴──────────────────┘
```

USER는 실제 사용자 계정을 기준으로,
GUEST는 Guest 인증 정보의 식별자를 기준으로 Warehouse 소유자를 결정합니다.

Personal Warehouse 생성 API도 요청자 유형에 따라 분리했습니다.

```http
POST /api/warehouses/{templateWarehouseId}/personal-copy
```

```http
POST /api/warehouses/{templateWarehouseId}/guest-personal-copy
```

클라이언트가 직접 소유자 ID를 지정하는 대신
Backend에서 현재 인증된 요청자를 기준으로 Personal Warehouse를 연결합니다.

---

## 4. Warehouse Deep Clone

Warehouse Entity만 새로 생성하면
실제 Simulation 상태는 분리되지 않습니다.

예를 들어 Robot이나 Inventory를 원본 Template과 계속 공유한다면
Personal Warehouse를 만들어도 사용자 간 상태 충돌이 발생합니다.

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
Node를 참조하는 Edge나 Storage 등의 관계도
새 Personal Warehouse 내부 Entity를 바라보도록 다시 연결합니다.

```text
Template Node
      ↓
New Personal Node
      ↓
Edge / Storage / ChargingStation
참조 관계 재구성
```

이를 통해 원본 Template과 실행 데이터를 공유하지 않는
독립적인 Warehouse 환경을 구성했습니다.

---

## 5. Shared Warehouse 실행 차단과 소유권 검증

Personal Warehouse를 제공하더라도
클라이언트가 Shared Warehouse ID나 다른 사용자의 Warehouse ID를
직접 요청할 가능성을 고려해야 합니다.

따라서 Simulation 실행 시 Backend에서 다시 검증합니다.

```text
Simulation 실행 요청
        ↓
Warehouse 조회
        ↓
Shared Template?
   ├─ YES → 실행 차단
   └─ NO
        ↓
현재 요청자 소유?
   ├─ YES → 실행
   └─ NO  → 접근 차단
```

즉,

```text
User A → User B Warehouse
Guest A → Guest B Warehouse
Guest  → User Warehouse
```

와 같은 요청이 Simulation 실행으로 이어지지 않도록
실제 Resource의 소유 관계를 확인합니다.

단순히 로그인 여부만 확인하는 것이 아니라
**요청자가 해당 Warehouse를 사용할 수 있는지 실행 경계에서 다시 검증**하도록 구성했습니다.

---

## 6. Personal Warehouse와 Scenario

Scenario도 Warehouse에 종속된 실행 데이터이기 때문에
Personal Warehouse 생성 과정에서 함께 복제됩니다.

```text
Shared Warehouse
 └─ Scenario

        ↓ Deep Clone

Personal Warehouse
 └─ Personal Scenario
```

따라서 Personal Warehouse의 Simulation에서는
Template Scenario가 아니라
복제된 Warehouse에 속한 Scenario를 사용합니다.

이를 통해 Warehouse만 분리되고
실행 시나리오는 다시 Shared Resource를 참조하는 상태를 방지했습니다.

---

## 7. Neo4j Graph 격리

PostgreSQL 데이터만 사용자별로 복제하고
Neo4j에서 동일한 Graph를 공유하면
Planning 단계에서는 여전히 같은 Digital Twin을 사용하게 됩니다.

Personal Warehouse 생성이 완료되면
새 Warehouse ID를 기준으로 Graph Sync Event를 발생시킵니다.

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

결과적으로 사용자마다 서로 다른 Warehouse Scope를 갖습니다.

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

Graph 동기화 방식은  
[04. Digital Twin Graph](./04-digital-twin-graph.md)에서 설명합니다.

---

## 8. 최종 실행 구조

사용자가 Shared Warehouse를 선택한 이후의 실행 흐름은 다음과 같습니다.

```text
로그인 / Guest 인증
        ↓
Shared Warehouse 선택
        ↓
요청자 식별
        ↓
Personal Warehouse 생성 또는 조회
        ↓
Warehouse 종속 데이터 분리
        ↓
Personal Graph 구성
        ↓
소유권 검증
        ↓
Simulation 실행
```

최종적으로 Shared Warehouse는
여러 사용자가 사용할 수 있는 **초기 Template** 역할만 담당하고,

```text
Inventory
Robot
Scenario
Graph
Simulation
```

등 실제 실행에 영향을 받는 상태는
Personal Warehouse 단위로 분리됩니다.

---

## 9. 주요 Component

| Component | 역할 |
|---|---|
| `WarehouseTemplateCloneService` | Shared Template 기반 Personal Warehouse Deep Clone |
| `AuthenticatedRequesterResolver` | USER / GUEST 요청자 식별 |
| `Warehouse` | Shared 여부와 USER / GUEST 소유 관계 관리 |
| `SimulationRunService` | Simulation 실행 시 Warehouse 검증 |
| `WarehouseGraphChangedEvent` | Personal Warehouse Graph Sync Trigger |
| `GraphSyncService` | Warehouse Scope 기준 Neo4j Graph 동기화 |

Multi-user Isolation의 핵심은
Guest 접속 기능 자체가 아니라,

**사용자가 동일한 Warehouse Template에서 시작하더라도  
실제 Digital Twin 실행 상태는 서로 공유하지 않도록 Resource 경계를 분리한 것**입니다.

---

## 관련 문서

- [02. Backend Design](./02-backend-design.md)
- [03. Warehouse Domain](./03-warehouse-domain.md)
- [04. Digital Twin Graph](./04-digital-twin-graph.md)
- [05. AI Integration](./05-ai-integration.md)

---

[← 05. AI Integration](./05-ai-integration.md) · [README](../README.md)
