# 04. Digital Twin Graph

> LARO에서는 Warehouse의 영속 데이터는 PostgreSQL에서 관리하고,  
> Node · Edge와 공간 객체의 이동·접근 관계는 Neo4j Graph로 구성합니다.

---

## 1. PostgreSQL과 Neo4j 역할 분리

Warehouse에서는 개별 객체의 정보뿐 아니라
객체 사이의 연결 관계도 중요합니다.

예를 들어 다음과 같은 관계를 표현해야 합니다.

```text
어떤 Node와 Node가 연결되어 있는가?
특정 Rack에는 어느 위치에서 접근할 수 있는가?
Charging Station은 어떤 이동 경로와 연결되어 있는가?
특정 Edge가 변경되면 Warehouse 이동 구조는 어떻게 달라지는가?
```

초기에는 Warehouse의 Node와 Edge를 PostgreSQL에 저장하고,
필요할 때 NetworkX로 Graph를 구성하는 방식도 검토했습니다.

NetworkX는 구현이 비교적 간단하고
BFS나 최단 경로와 같은 Graph Algorithm을 수행하기에는 충분했습니다.

하지만 LARO에서는 Graph를 일회성 계산에만 사용하는 것이 아니라,

- Warehouse의 Node·Edge 관계를 지속적으로 관리하고
- Backend에서 관계 기반으로 조회하며
- AI Planning에서도 동일한 이동 구조를 반복적으로 활용

할 필요가 있었습니다.

따라서 관계 자체를 Graph 구조로 저장하고 조회할 수 있는
Neo4j를 선택했습니다.

PostgreSQL은 Warehouse와 관련 Resource의 기준 데이터를 관리하고,
Neo4j는 이동·접근 관계를 Graph 형태로 표현하도록 역할을 분리했습니다.

```text
PostgreSQL
Warehouse · Node · Edge · Resource
        ↓
    Graph Sync
        ↓
Neo4j
Warehouse Graph
```

Neo4j를 원본 데이터베이스로 사용하지 않고,
PostgreSQL 데이터를 기반으로 구성되는
**Graph Projection**으로 사용했습니다.

---

## 2. PostgreSQL을 Source of Truth로 사용

PostgreSQL과 Neo4j에 Warehouse 정보가 함께 존재하기 때문에
두 저장소의 기준을 명확하게 정할 필요가 있었습니다.

LARO에서는 PostgreSQL을 Warehouse 데이터의
`Source of Truth`로 사용합니다.

```text
PostgreSQL
Source of Truth
      │
      ▼
Neo4j
Graph Projection
```

Warehouse 구조가 변경되면
먼저 PostgreSQL에 변경 사항을 저장하고,
Commit된 현재 데이터를 기준으로 Neo4j Graph를 갱신합니다.

따라서 두 저장소의 상태가 달라지는 경우에도
PostgreSQL 데이터를 기준으로 Graph를 다시 구성할 수 있습니다.

---

## 3. AFTER_COMMIT 기반 Graph Sync

PostgreSQL과 Neo4j를 하나의 Service 로직에서
순차적으로 직접 수정하면 Transaction 처리 시점에 따라
두 저장소의 상태가 달라질 수 있습니다.

예를 들어,

```text
PostgreSQL 변경
      ↓
Neo4j 변경
      ↓
PostgreSQL Rollback
```

과 같은 상황에서는
Neo4j에만 변경 내용이 남을 수 있습니다.

이를 방지하기 위해 Graph Sync는
PostgreSQL Transaction이 정상적으로 Commit된 이후에 실행하도록 구성했습니다.

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

Transaction이 Rollback되면
Graph Sync 단계까지 진행되지 않습니다.

---

## 4. WarehouseGraphChangedEvent

Graph Sync가 필요한 경우
변경된 Node나 Edge 데이터 전체를 Event에 담지 않고,
대상 Warehouse의 ID만 전달합니다.

```java
public record WarehouseGraphChangedEvent(
    Long warehouseId
) {
}
```

Event의 역할은 다음과 같습니다.

```text
"이 Warehouse의 현재 데이터를 기준으로
Neo4j Graph를 다시 동기화한다."
```

실제 동기화 과정에서는
PostgreSQL의 현재 Warehouse 데이터를 다시 조회합니다.

```text
Warehouse Domain 변경
        ↓
WarehouseGraphChangedEvent
        │
        └─ warehouseId
                ↓
GraphSyncService
        ↓
PostgreSQL 현재 상태 조회
        ↓
Neo4j Warehouse Graph 갱신
```

이렇게 구성하여
Warehouse Domain 로직과 Neo4j 저장 로직을 분리했습니다.

---

## 5. Graph Sync 발생 시점

Warehouse Graph에 영향을 주는 데이터가 변경되면
동일한 Event를 통해 Graph Sync를 요청합니다.

대표적인 대상은 다음과 같습니다.

```text
Node 변경
Edge 변경
Warehouse Layout 변경
Warehouse Import
Personal Warehouse 생성
```

각 기능에서 Neo4j를 직접 수정하는 대신

```text
Domain 변경
      ↓
Event 발행
      ↓
GraphSyncService
```

라는 공통 흐름을 사용합니다.

이를 통해 Warehouse 관련 기능이
Neo4j 구현 세부사항에 직접 의존하지 않도록 구성했습니다.

---

## 6. Warehouse Scope 기반 Graph 분리

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

동일한 Node Code가 존재하더라도
Warehouse가 다르면 서로 다른 Graph 객체로 구분할 수 있도록
Warehouse Scope를 포함한 식별 구조를 사용합니다.

```text
WH-002::R3_9
WH-003::R3_9
```

이 구조는 USER / GUEST별 Personal Warehouse에도 동일하게 적용됩니다.

```text
Shared Template
       ↓
Personal Warehouse A
       ↓
Neo4j Graph A


Shared Template
       ↓
Personal Warehouse B
       ↓
Neo4j Graph B
```

Personal Warehouse가 생성되면
새 Warehouse ID를 기준으로 Graph Sync가 수행됩니다.

사용자별 Warehouse 분리 구조는  
[06. Multi-user Isolation](./06-multi-user-isolation.md)에서 설명합니다.

---

## 7. Warehouse Graph 제공

Backend에서는 현재 Warehouse의
Node · Edge 이동 구조를 외부에서 사용할 수 있도록 Graph API를 제공합니다.

```text
PostgreSQL
Warehouse / Node / Edge
        ↓
WarehouseGraphService
        ↓
WarehouseGraphResponse
        ↓
Frontend / AI Planning
```

Neo4j는 Warehouse 객체 간 관계를 관리하고,
Graph API는 Backend가 관리하는 Warehouse 이동 데이터를
외부에서 사용할 수 있는 형태로 제공합니다.

AI Planning에서도 별도의 Map을 임의로 구성하는 것이 아니라
Backend가 제공하는 Warehouse Graph 데이터를 활용할 수 있도록 구성했습니다.

Graph API 구조는  
[03. Warehouse Domain](./03-warehouse-domain.md)에서,
AI에 제공하는 데이터 구조는  
[05. Warehouse-AI Integration](./05-warehouse-ai-integration.md)에서 설명합니다.

---

## 8. Graph Sync 실패 시 기준

`AFTER_COMMIT` 구조를 사용하더라도 다음 상황은 발생할 수 있습니다.

```text
PostgreSQL COMMIT 성공
        ↓
Neo4j Graph Sync 실패
```

`AFTER_COMMIT`은 PostgreSQL Transaction이 실패했는데
Neo4j만 먼저 변경되는 문제를 방지하지만,
Commit 이후 발생한 Neo4j Sync 실패까지 자동으로 복구하지는 못합니다.

이 경우에도 PostgreSQL에는 정상적인 기준 데이터가 남아 있습니다.

Neo4j는 PostgreSQL 데이터를 기반으로 다시 구성할 수 있는
Graph Projection이기 때문에,
PostgreSQL의 현재 상태를 기준으로 Graph를 다시 동기화할 수 있습니다.

```text
PostgreSQL
Source of Truth
        ↓
Graph 재동기화
        ↓
Neo4j
Graph Projection
```

현재 MVP에서는 PostgreSQL의 정합성을 우선하고
Neo4j를 재생성 가능한 Projection으로 관리하는 구조까지 구현했습니다.

서비스 규모가 커질 경우에는 Graph Sync 실패를 안정적으로 재처리할 수 있도록

- **Retry**
- **Transactional Outbox**

와 같은 구조를 적용하는 방향을 검토할 수 있습니다.

---

## 9. 주요 Component

| Component | 역할 |
|---|---|
| `WarehouseGraphChangedEvent` | Graph Sync 대상 Warehouse ID 전달 |
| `GraphSyncService` | PostgreSQL 데이터를 기준으로 Neo4j Graph 동기화 |
| `WarehouseGraphService` | Warehouse Node · Edge Graph 구성 및 조회 |

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

Digital Twin Graph에서는
**PostgreSQL과 Neo4j를 동시에 직접 수정하는 대신,
기준 데이터와 동기화 시점을 분리하는 구조**를 중심으로 구현했습니다.

---

## 관련 문서

- [01. Project Overview](./01-project-overview.md)
- [02. Backend Design](./02-backend-design.md)
- [03. Warehouse Domain](./03-warehouse-domain.md)
- [05. Warehouse-AI Integration](./05-warehouse-ai-integration.md)
- [06. Multi-user Isolation](./06-multi-user-isolation.md)

---

[← 03. Warehouse Domain](./03-warehouse-domain.md) · [README](../README.md)
