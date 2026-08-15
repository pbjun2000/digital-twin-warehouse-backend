# 04. Digital Twin Graph

> LARO에서는 PostgreSQL의 Warehouse 데이터를 기준으로  
> **Node · Edge · Storage · Charging Station의 연결 관계를 Neo4j Graph로 동기화**하여 Digital Twin의 공간 관계를 표현합니다.

---

## 1. Graph DB를 사용한 이유

Warehouse 데이터는 단순한 개별 Entity 정보뿐 아니라  
**공간 객체 사이의 연결 관계**가 중요합니다.

예를 들어 AI Planning에서는 다음과 같은 관계가 필요합니다.

```text
현재 위치에서 목적지까지 어떤 Node들이 연결되어 있는가?

특정 Storage에는 어떤 이동 Node를 통해 접근할 수 있는가?

Charging Station은 어떤 Route와 연결되어 있는가?

특정 Edge가 차단되면 어떤 이동 관계가 영향을 받는가?
```

관계형 데이터베이스에서도 이러한 데이터를 저장할 수 있지만,
LARO에서는 Warehouse의 공간 구조를 관계 중심으로 탐색하기 위해
Neo4j Graph를 별도로 구성했습니다.

```text
Warehouse
   │
   ├── Node ── CONNECTED_TO ── Node
   │
   ├── Storage
   │      └── 접근 가능한 Node
   │
   └── Charging Station
          └── 연결된 Node
```

Neo4j는 영속 데이터의 원본이 아니라  
**PostgreSQL Warehouse 데이터를 기반으로 생성되는 Graph Projection**으로 사용합니다.

---

## 2. Source of Truth

LARO에서는 Warehouse 데이터의 기준을 PostgreSQL로 정의했습니다.

```text
              Source of Truth
                    │
                    ▼
              PostgreSQL
                    │
           Graph Projection
                    ▼
                 Neo4j
```

PostgreSQL에서는 다음과 같은 기준 데이터를 관리합니다.

```text
Warehouse
Zone
Node
Edge
StorageLocation
ChargingStation
Robot
Inventory
Scenario
```

Neo4j는 이 중 공간 탐색에 필요한 데이터를 Graph 형태로 표현합니다.

따라서 데이터가 서로 다를 경우  
**PostgreSQL 데이터를 기준으로 Neo4j Graph를 다시 생성할 수 있도록 설계**했습니다.

---

## 3. 단순 Dual Write의 문제

초기 구조에서 다음과 같이 두 저장소를 순차적으로 직접 수정하면 문제가 발생할 수 있습니다.

```text
Spring Boot
    │
    ├── PostgreSQL 저장
    │
    └── Neo4j 저장
```

예를 들어,

```text
1. PostgreSQL 저장 성공
2. Neo4j 저장 성공
3. PostgreSQL Transaction Rollback
```

또는

```text
1. PostgreSQL 저장 성공
2. Neo4j 동기화 실패
```

와 같은 상황에서는 두 저장소의 상태가 달라질 수 있습니다.

PostgreSQL과 Neo4j를 하나의 일반적인 JPA Transaction으로 묶을 수 없기 때문에,
**Graph Sync를 실행하는 시점**을 명확하게 정의할 필요가 있었습니다.

---

## 4. AFTER_COMMIT 기반 Graph Sync

이를 해결하기 위해 PostgreSQL Transaction이 정상적으로 완료된 이후에만
Neo4j Graph Sync가 시작되도록 구성했습니다.

```text
Warehouse 변경
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
Neo4j Graph Sync
```

Listener는 Spring의 `TransactionalEventListener`를 사용합니다.

```java
@TransactionalEventListener(
    phase = TransactionPhase.AFTER_COMMIT
)
```

이를 통해 PostgreSQL Transaction이 Rollback된 경우에는
Graph Sync 자체가 실행되지 않도록 했습니다.

---

## 5. WarehouseGraphChangedEvent

Graph 변경 이벤트는 세부 Node나 Edge의 변경 정보를 직접 전달하지 않습니다.

실제 Event의 책임은 단순합니다.

```java
public record WarehouseGraphChangedEvent(
    Long warehouseId
) {
}
```

즉 다음과 같은 형태가 아닙니다.

```text
changeType
changedNode
changedEdge
changedData
```

대신,

```text
"이 Warehouse의 Graph를 다시 동기화해야 한다"
```

라는 사실만 전달합니다.

```text
Warehouse 변경
      ↓
WarehouseGraphChangedEvent
      │
      └── warehouseId
              ↓
      GraphSyncService
              ↓
      해당 Warehouse Scope 재동기화
```

이 방식은 Event 자체가 Graph 구조를 알지 않아도 되기 때문에
Warehouse Domain과 Neo4j Sync 로직의 책임을 분리할 수 있습니다.

---

## 6. Warehouse Scope 기반 Graph Sync

LARO는 하나의 공용 Graph만 사용하는 것이 아니라
Warehouse별 Graph Scope를 구분합니다.

```text
Warehouse A
 ├─ Node
 ├─ Edge
 └─ Storage

Warehouse B
 ├─ Node
 ├─ Edge
 └─ Storage
```

Graph Node에도 어떤 Warehouse에 속하는지 식별할 수 있는 Scope를 포함합니다.

예를 들어 Graph의 식별자는 다음과 같이 Warehouse 범위를 포함할 수 있습니다.

```text
WH-002::R3_9
```

이를 통해 동일한 Node Code가 존재하더라도
Warehouse가 다르면 서로 다른 Digital Twin 객체로 구분할 수 있습니다.

```text
WH-002::R3_9
≠
WH-003::R3_9
```

사용자별 Personal Warehouse를 생성하는 구조에서도
각 Warehouse별 Graph를 독립적으로 유지할 수 있습니다.

---

## 7. Graph Sync 처리 방식

`WarehouseGraphChangedEvent`는 변경된 Edge 하나만 수정하는
Delta Event가 아닙니다.

```text
Edge A 변경
        ↓
"Edge A만 수정"
```

방식보다,

```text
Warehouse 데이터 변경
        ↓
warehouseId 전달
        ↓
PostgreSQL의 현재 Warehouse 상태 조회
        ↓
해당 Warehouse Graph 동기화
```

방식을 사용합니다.

즉 Neo4j의 상태를 이벤트 데이터 자체에 의존시키지 않고
**PostgreSQL의 현재 데이터를 다시 기준으로 삼아 Graph를 구성**합니다.

이 구조에서는 Graph가 잘못된 상태가 되더라도
Source of Truth인 PostgreSQL을 기준으로 복구하기 쉽다는 장점이 있습니다.

---

## 8. Personal Warehouse 생성과 Graph Sync

USER / GUEST별 Personal Warehouse를 생성할 때는
Warehouse 하나만 복제하지 않습니다.

```text
Shared Warehouse
       ↓
Personal Warehouse 생성
       ↓
Zone
Node
Edge
ChargingStation
StorageLocation
WarehouseItem
Robot
Scenario
       ↓
PostgreSQL COMMIT
       ↓
WarehouseGraphChangedEvent
       ↓
Neo4j Graph Sync
```

`WarehouseTemplateCloneService`에서는
Personal Warehouse와 종속 데이터를 생성한 뒤
새롭게 생성된 Warehouse ID를 기준으로 Graph Sync Event를 발생시킵니다.

```text
Template Warehouse
        ↓
Deep Clone
        ↓
Personal Warehouse
        ↓
COMMIT
        ↓
Personal Warehouse 전용 Graph 생성
```

따라서 사용자별 Simulation에서는
PostgreSQL의 실행 데이터뿐 아니라
Neo4j의 Digital Twin Graph도 Warehouse Scope 단위로 분리됩니다.

---

## 9. PostgreSQL과 Neo4j의 역할 분리

두 데이터베이스에는 동일한 데이터를 무조건 중복 저장하지 않고
각 저장소의 역할을 구분했습니다.

### PostgreSQL

서비스의 영속 데이터와 정합성을 관리합니다.

```text
Warehouse
Robot
Inventory
Task
Scenario
SimulationRun
```

### Neo4j

공간 객체 사이의 관계 탐색을 담당합니다.

```text
Node
Edge
Storage 접근 관계
Charging Station 연결 관계
```

전체 구조는 다음과 같습니다.

```text
                  Spring Boot
                       │
          ┌────────────┴────────────┐
          │                         │
          ▼                         ▼
     PostgreSQL                  Neo4j
   Persistent Data          Graph Projection
          │                         ▲
          │                         │
          └── AFTER_COMMIT Sync ────┘
```

---

## 10. AI Planning에서의 활용

AI Planning은 임의의 Warehouse 구조를 생성하지 않고
Backend가 관리하는 실제 Warehouse 데이터를 기준으로 동작합니다.

Graph는 다음과 같은 공간 관계 탐색의 근거로 활용됩니다.

```text
Warehouse
    ↓
현재 요청과 관련된 Node 탐색
    ↓
연결된 Edge 확인
    ↓
Storage / Station 접근 관계 확인
    ↓
AI Context 구성
    ↓
Optimization / MAPF
```

이를 통해 AI가 존재하지 않는 위치나
연결되지 않은 이동 경로를 임의로 가정하는 것을 줄이고,
실제 Warehouse 구조를 계획의 근거로 사용할 수 있도록 구성했습니다.

---

## 11. Graph Sync 실패에 대한 설계 기준

`AFTER_COMMIT` 구조는 PostgreSQL의 Commit 이전에
Neo4j가 먼저 변경되는 문제를 방지합니다.

다만 다음 상황은 여전히 발생할 수 있습니다.

```text
PostgreSQL COMMIT 성공
        ↓
Neo4j Graph Sync 실패
```

현재 구조에서는 PostgreSQL이 Source of Truth이므로
이 경우에도 영속 데이터 자체는 유지됩니다.

```text
PostgreSQL
현재 정상 데이터 유지
        ↓
Graph Sync 재수행 가능
        ↓
Neo4j 복구
```

즉 두 저장소를 완전히 동일한 Transaction으로 처리하는 것이 아니라,

> **PostgreSQL의 정합성을 우선 보장하고,
> Neo4j를 재생성 가능한 Projection으로 관리**

하는 방향을 선택했습니다.

대규모 운영 환경으로 확장할 경우에는
Graph Sync 실패 이벤트의 재시도·보상 처리와
Dead Letter Queue 등의 구조를 추가할 수 있습니다.

---

## 12. 주요 Component

| Component | 역할 |
|---|---|
| `WarehouseGraphChangedEvent` | 특정 Warehouse의 Graph Sync 필요 여부 전달 |
| `GraphSyncService` | PostgreSQL 데이터를 기반으로 Warehouse Graph 동기화 |
| `WarehouseTemplateCloneService` | Personal Warehouse 생성 후 Graph Sync Trigger |
| `WarehouseGraphService` | Warehouse Node·Edge Graph 조회 |
| `WarehouseNode` | Warehouse 이동 및 작업 지점 |
| `WarehouseEdge` | Node 사이의 이동 관계 |

---

## 13. 설계 결과

최종적으로 데이터 흐름을 다음과 같이 구성했습니다.

```text
         PostgreSQL
       Source of Truth
             │
             │ COMMIT
             ▼
WarehouseGraphChangedEvent
             │
             │ AFTER_COMMIT
             ▼
      GraphSyncService
             │
             ▼
           Neo4j
      Graph Projection
```

이 구조를 통해 다음 기준을 유지했습니다.

- PostgreSQL을 Warehouse 데이터의 단일 기준으로 사용
- Transaction 성공 이후에만 Graph Sync 수행
- Event와 Graph 동기화 로직의 책임 분리
- USER / GUEST별 Warehouse Graph Scope 분리
- Neo4j 데이터를 PostgreSQL 기준으로 재생성 가능하도록 구성

---

## 핵심 코드 흐름

```text
Warehouse 변경
    ↓
Service Transaction
    ↓
PostgreSQL 저장
    ↓
WarehouseGraphChangedEvent(warehouseId)
    ↓
Transaction COMMIT
    ↓
@TransactionalEventListener(AFTER_COMMIT)
    ↓
GraphSyncService.syncWarehouseGraph(warehouseId)
    ↓
Neo4j Warehouse Graph 갱신
```

> 핵심은 **두 DB에 동시에 Write하는 것보다,
> 어떤 DB를 기준으로 삼고 어느 시점에 동기화할 것인지를 명확하게 정의한 것**입니다.

---

## 다음 문서

- [05. AI Integration](./05-ai-integration.md)
- [06. Multi-user Isolation](./06-guest-access.md)

---

[← 03. Warehouse Domain](./03-warehouse-domain.md) · [README](../README.md)
