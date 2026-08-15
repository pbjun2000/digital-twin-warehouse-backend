# 04. Digital Twin Graph

> LARO에서는 Warehouse의 영속 데이터는 PostgreSQL에서 관리하고,  
> Node·Edge와 공간 객체의 이동·접근 관계는 Neo4j Graph로 구성합니다.

---

## 1. Neo4j를 사용한 이유

Warehouse에서는 개별 객체의 정보뿐 아니라
객체 사이의 **연결 관계**가 중요합니다.

예를 들어 Robot의 이동 계획을 구성하려면 다음과 같은 정보가 필요합니다.

```text
어떤 Node가 서로 연결되어 있는가?

특정 Storage에는 어떤 Node를 통해 접근할 수 있는가?

Charging Station은 어느 이동 경로와 연결되어 있는가?

특정 Edge를 사용할 수 없을 때 이동 관계가 어떻게 달라지는가?
```

PostgreSQL은 Warehouse, Robot, Inventory 등 서비스 데이터를 관리하고,
Neo4j는 이러한 공간 관계를 Graph 형태로 표현하도록 역할을 나눴습니다.

```text
PostgreSQL
Warehouse · Node · Edge · Resource
        ↓
Graph Sync
        ↓
Neo4j
Warehouse Graph
```

Neo4j를 원본 데이터베이스로 사용하지 않고
**PostgreSQL 데이터를 기반으로 생성되는 Graph Projection**으로 구성했습니다.

---

## 2. PostgreSQL을 Source of Truth로 사용

두 저장소에 같은 Warehouse 정보가 존재하면
어느 데이터를 기준으로 판단할지 명확해야 합니다.

LARO에서는 PostgreSQL을 Warehouse 데이터의 기준으로 사용합니다.

```text
PostgreSQL
Source of Truth
     │
     └──→ Neo4j
          Graph Projection
```

Warehouse 구조가 변경되면 먼저 PostgreSQL에 저장하고,
정상적으로 Commit된 데이터를 기준으로 Neo4j를 갱신합니다.

따라서 두 저장소의 상태가 일치하지 않는 경우에도
PostgreSQL의 현재 데이터를 기준으로 Graph를 다시 구성할 수 있습니다.

---

## 3. AFTER_COMMIT 기반 Graph Sync

PostgreSQL과 Neo4j를 Service에서 순차적으로 직접 수정하면
Transaction 처리 시점에 따라 데이터가 달라질 수 있습니다.

예를 들어,

```text
PostgreSQL 저장
      ↓
Neo4j 저장
      ↓
PostgreSQL Rollback
```

과 같은 상황이 발생하면
Neo4j에는 변경 사항이 남고 PostgreSQL에는 남지 않을 수 있습니다.

이를 방지하기 위해 Graph Sync는
PostgreSQL Transaction이 정상적으로 Commit된 이후에 실행합니다.

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

Listener에는 Spring의 `TransactionalEventListener`를 사용합니다.

```java
@TransactionalEventListener(
    phase = TransactionPhase.AFTER_COMMIT
)
```

Transaction이 Rollback되면 Event Listener의 Graph Sync 단계까지 진행되지 않습니다.

---

## 4. WarehouseGraphChangedEvent

Graph Sync Event에는 변경된 Node나 Edge 데이터를 직접 담지 않습니다.

실제 Event는 동기화가 필요한 Warehouse만 전달합니다.

```java
public record WarehouseGraphChangedEvent(
    Long warehouseId
) {
}
```

즉 Event의 의미는 다음과 같습니다.

```text
"warehouseId에 해당하는 Graph를
현재 PostgreSQL 데이터를 기준으로 다시 동기화한다."
```

Event가 변경 데이터 자체를 전달하는 방식보다
현재 PostgreSQL 상태를 다시 조회하는 방식을 사용했습니다.

```text
Node / Edge / Layout 변경
        ↓
WarehouseGraphChangedEvent(warehouseId)
        ↓
GraphSyncService
        ↓
PostgreSQL 현재 상태 조회
        ↓
Warehouse 단위 Neo4j Sync
```

이를 통해 Warehouse Domain은
Neo4j 내부 Graph 구조를 직접 알 필요가 없도록 책임을 분리했습니다.

---

## 5. Graph Sync가 발생하는 시점

Warehouse Graph에 영향을 주는 데이터가 변경되면
동일한 Event를 통해 Graph Sync를 요청합니다.

대표적으로 다음과 같은 작업이 해당됩니다.

```text
Node 생성 / 수정 / 제거
Edge 생성 / 수정 / 제거
Warehouse Layout 수정
Warehouse JSON Import
Personal Warehouse 생성
```

각 기능에서 Neo4j를 직접 수정하지 않고

```text
Domain 변경
     ↓
Event 발행
     ↓
GraphSyncService
```

라는 동일한 흐름을 사용합니다.

그래서 Warehouse 변경 로직과
Graph 저장 로직을 별도로 유지할 수 있습니다.

---

## 6. Warehouse Scope 분리

LARO에서는 Graph도 Warehouse 단위로 구분합니다.

```text
Warehouse A
 ├─ Nodes
 ├─ Edges
 └─ Resources

Warehouse B
 ├─ Nodes
 ├─ Edges
 └─ Resources
```

Graph 객체는 자신이 속한 Warehouse Scope를 기준으로 관리하며,
외부 Graph 식별자에도 Warehouse 범위를 포함할 수 있습니다.

```text
WH-002::R3_9
```

이를 통해 동일한 Node Code가 존재하더라도
다른 Warehouse에 속하면 서로 다른 객체로 구분할 수 있습니다.

```text
WH-002::R3_9
WH-003::R3_9
```

이 구조는 USER / GUEST별 Personal Warehouse에서도 동일하게 적용됩니다.

```text
Shared Template
       ↓
Personal Warehouse A
       ↓
Graph A

Shared Template
       ↓
Personal Warehouse B
       ↓
Graph B
```

Personal Warehouse 생성 이후에는
새롭게 생성된 Warehouse ID를 기준으로 Graph Sync를 수행합니다.

사용자별 Warehouse 복제 구조는  
[06. Multi-user Isolation](./06-multi-user-isolation.md)에서 자세히 설명합니다.

---

## 7. PostgreSQL과 Neo4j 역할

두 저장소의 역할은 다음과 같이 구분했습니다.

| 저장소 | 역할 |
|---|---|
| **PostgreSQL** | Warehouse, Node, Edge, Robot, Inventory 등 서비스 기준 데이터 |
| **Neo4j** | Warehouse 내부 Node·Edge 및 공간 객체의 이동·접근 관계 |

예를 들어 Warehouse Map 자체의 기준은 PostgreSQL에 있으며,
Neo4j는 Planning에서 사용할 수 있는 관계 구조를 제공합니다.

```text
PostgreSQL
     │
     │ Warehouse Data
     ▼
GraphSyncService
     │
     ▼
Neo4j
     │
     └── Node / Edge / Resource 관계
```

Graph가 서비스 데이터의 원본이 되지 않도록 역할을 명확하게 구분했습니다.

---

## 8. AI Planning과 Graph

Backend는 AI Planning에서 사용할 수 있도록
현재 Warehouse의 Graph 구조를 제공합니다.

```text
Warehouse
    ↓
Node / Edge
    ↓
이동·접근 관계
    ↓
AI Planning Context
    ↓
Optimization / MAPF
```

AI가 임의의 Map을 생성하는 것이 아니라
Backend가 관리하는 현재 Warehouse 구조를 기준으로
Planning할 수 있도록 구성했습니다.

Warehouse Graph 조회 구조는  
[03. Warehouse Domain](./03-warehouse-domain.md)의 `WarehouseGraphService`에서 설명합니다.

---

## 9. Graph Sync 실패 시 기준

`AFTER_COMMIT`을 사용하면 PostgreSQL Rollback 이전에
Neo4j가 먼저 변경되는 문제는 방지할 수 있습니다.

다만 다음 상황은 발생할 수 있습니다.

```text
PostgreSQL COMMIT 성공
        ↓
Neo4j Graph Sync 실패
```

이 경우에도 PostgreSQL에는 정상적인 기준 데이터가 남아 있습니다.

Neo4j는 PostgreSQL을 기반으로 다시 생성할 수 있는 Projection이기 때문에,
Graph Sync를 다시 수행하여 복구할 수 있는 구조입니다.

즉 설계 기준은

```text
PostgreSQL 정합성 우선
        +
Neo4j 재동기화 가능
```

으로 두었습니다.

운영 규모가 커질 경우에는
Graph Sync 실패에 대한 재시도나 별도 실패 이벤트 처리 구조를
추가할 수 있습니다.

---

## 10. 주요 Component

| Component | 역할 |
|---|---|
| `WarehouseGraphChangedEvent` | Graph Sync가 필요한 Warehouse ID 전달 |
| `GraphSyncService` | PostgreSQL 데이터를 기준으로 Neo4j Graph 동기화 |
| `WarehouseGraphService` | Warehouse Node·Edge Graph 구성 및 조회 |
| `WarehouseTemplateCloneService` | Personal Warehouse 생성 후 Graph Sync Trigger |

핵심 흐름은 다음과 같습니다.

```text
PostgreSQL
Source of Truth
      ↓
COMMIT
      ↓
WarehouseGraphChangedEvent
      ↓
GraphSyncService
      ↓
Neo4j
Graph Projection
```

PostgreSQL과 Neo4j를 동시에 직접 수정하는 대신,
**기준 데이터와 동기화 시점을 명확하게 분리한 것**이
Digital Twin Graph 연동에서의 핵심 설계입니다.

---

## 관련 문서

- [03. Warehouse Domain](./03-warehouse-domain.md)
- [05. AI Integration](./05-ai-integration.md)
- [06. Multi-user Isolation](./06-multi-user-isolation.md)

---

[← 03. Warehouse Domain](./03-warehouse-domain.md) · [README](../README.md)
