# LARO

> Digital Twin 기반 자율 창고 운영 및 다중 로봇 작업 최적화 시스템
>
> 프로젝트 기간: 2026.07 ~ 2026.08  
> 역할: Backend Developer  
> 프로젝트 형태: KT AIVLE School 9기 Big Project

KT AIVLE School 9기에서 2개월간  
**Backend 3명 · AI 2명 · Frontend 1명으로 구성된 6인 팀**이 개발한 B2B형 MVP 프로젝트입니다.

LARO(LLM Autonomous Robot Orchestration)는  
운영자의 자연어 요청과 현재 Warehouse 상태를 기반으로 AI가 작업 계획을 생성하고,
다중 로봇의 작업 배정·경로 계획·재계획을 수행하는 Digital Twin 기반 자율 창고 시뮬레이션 플랫폼입니다.

```text
사용자 요청 + Digital Twin State
              ↓
         AI Planning
              ↓
        NVIDIA cuOpt
      작업 배정 · 수행 순서
              ↓
            MAPF
       충돌 없는 이동 계획
              ↓
         Simulation
              ↓
        Runtime State
              ↓
         Replanning
              └────→ AI Planning
```

> 본 저장소는 팀 프로젝트에서 제가 담당한 Backend 설계·구현과  
> 문제 해결 과정을 중심으로 정리한 개인 포트폴리오입니다.

---

## 프로젝트 개요

물류센터에 다수의 AGV·AMR이 도입되면서
개별 로봇의 이동뿐 아니라 여러 로봇의 작업과 이동을 동시에 조율하는 것이 중요해지고 있습니다.

실제 운영에서는 신규 작업, 배터리 변화, 통로 상태 등으로
초기에 생성된 계획의 유효성이 계속 달라질 수 있습니다.

LARO는 현재 Warehouse 상태를 기준으로 AI가 실행 계획을 구성하고,
Optimization Solver와 MAPF를 통해 작업 배정과 이동 계획을 계산한 뒤
Digital Twin 환경에서 이를 실행·관제하고 상태 변화에 따라 재계획합니다.

---

## 개발 전략

2개월의 프로젝트 기간 안에서 아래 핵심 실행 흐름을 우선 연결했습니다.

```text
Warehouse State
      ↓
AI Planning
      ↓
Simulation
```

이후 USER / GUEST별 독립적인 Simulation 환경과 Dynamic Replanning으로 범위를 확장하고,
최종적으로 동일 Optimization Solver 조건에서 Rule 방식과 Agent 방식의 실행 결과를 비교했습니다.

---

## 주요 기능

### Digital Twin 창고 구성

- 창고 시설과 Robot 이동 지점을 배치하고 `Node · Edge` 기반 이동 구조 구성
- 입고·출고·충전 설비와 Storage Location을 포함한 Warehouse Layout 관리
- 구성한 Warehouse 데이터를 AI Planning과 Simulation에서 공통으로 활용

---

### AI 기반 작업 계획

- 운영자의 자연어 요청과 현재 Warehouse State를 기반으로 Mission 구성
- 요청과 운영 조건을 해석해 Mission / Constraints 생성
- 작업 배정과 수행 순서는 Optimization Solver가 계산
- 다중 Robot 이동 경로는 MAPF를 통해 계산

```text
LLM / Agent
Mission · Constraints
        ↓
Optimization Solver
작업 배정 · 방문 순서
        ↓
MAPF
다중 로봇 이동 계획
        ↓
Robot Execution Plan
MOVE · WAIT · SERVICE
```

> LLM은 운영 조건을 해석하고,  
> Solver는 최적화 계산을 담당하도록 역할을 분리했습니다.

---

### 다중 로봇 작업·경로 최적화

- 여러 Task를 Robot별로 배정하고 수행 순서 계산
- MAPF를 통해 여러 Robot의 이동 경로와 충돌 가능성을 함께 고려
- 결과를 `MOVE · WAIT · SERVICE` 단위의 실행 가능한 Robot Plan으로 구성

---

### Simulation & Monitoring

- AI가 생성한 Plan을 Backend에서 검증한 뒤 Simulation 실행 데이터로 적용
- Robot 위치·배터리·작업 진행 상태 관리
- Task · Robot · Event 정보를 관제 화면에서 확인

---

### Dynamic Replanning

실행 중 운영자의 추가 요청이나 Warehouse 상태 변화가 발생하면
현재 실행 상태와 남아 있는 작업을 기반으로 새로운 Plan을 생성합니다.

```text
현재 Runtime State
        ↓
추가 요청 / 상태 변화
        ↓
AI Replanning
        ↓
새로운 Plan 생성
        ↓
Backend Plan 적용
        ↓
Simulation 재개
```

---

### 사용자별 독립 Simulation

- Shared Warehouse를 Template으로 사용
- USER / GUEST별 Personal Warehouse 생성
- 재고·Robot·Scenario 등 실행 데이터를 사용자별로 독립 관리
- Warehouse Ownership 검증을 통해 다른 사용자의 실행 환경 접근 차단

---

## 시스템 아키텍처

```text
┌─────────────────────────────┐
│        React Frontend       │
└──────────────┬──────────────┘
               │ REST / STOMP
               ▼
┌─────────────────────────────┐
│         Spring Boot         │
│                             │
│ Auth · Warehouse · Robot    │
│ Task · Simulation · AI 연동 │
└───────┬───────────┬─────────┘
        │           │
        │           └──────→ Redis
        │                   Runtime State
        ▼
     FastAPI
        │
        ▼
 GPT-5-mini / LangGraph
        │
        ▼
 Optimization Solver
   cuOpt / OR-Tools
        │
        ▼
       MAPF


PostgreSQL
    │
    │ AFTER_COMMIT
    ▼
  Neo4j
Warehouse Graph
```

### Data Store 역할

| 저장소 | 역할 |
|---|---|
| PostgreSQL | Warehouse, Robot, WarehouseItem, Task, Scenario 등 영속 데이터 |
| Redis | Robot 위치·배터리·실행 상태 등 Runtime State |
| Neo4j | Node·Edge 및 Warehouse 객체 간 이동·접근 관계 |

---

# 담당 역할 및 기여

팀 내 Backend 개발을 담당하며  
**Warehouse · Robot · Graph · 사용자별 실행 환경 영역**을 구현했습니다.

- Warehouse / Zone / Node / Edge REST API 및 관련 도메인 구현
- Robot 및 Warehouse Layout 데이터 연동
- USER / GUEST별 독립적인 Simulation 실행 환경 설계
- Shared Warehouse → Personal Warehouse Deep Clone 구현
- Warehouse Resource 소유권 검증 및 접근 제어
- PostgreSQL → Neo4j Warehouse Graph Sync 구현
- AI Planning에서 활용할 Warehouse Graph API 구현
- AI Planning과 Node·Edge 데이터 및 외부 식별자 연동
- Backend / AI / Frontend 통합 과정 참여

특히 다음 세 가지를 주요 Backend 설계 과제로 다뤘습니다.

1. **여러 사용자의 Simulation 실행 상태를 어떻게 격리할 것인가**
2. **PostgreSQL과 Neo4j의 역할과 데이터 일관성을 어떻게 관리할 것인가**
3. **Warehouse 데이터를 AI Planning에서 사용할 수 있는 형태로 어떻게 제공할 것인가**

---

# 핵심 설계 및 구현

## 1. 사용자별 Digital Twin 실행 환경 격리

여러 사용자가 동일한 Shared Warehouse에서 Simulation을 실행하면
Robot·Inventory·Scenario 등 실행 상태가 서로 영향을 줄 수 있습니다.

단순히 조회 결과를 `userId` 기준으로 필터링하는 것보다
**실행 환경 자체를 분리하는 것이 상태 충돌 방지에 더 명확하다**고 판단했습니다.

따라서 Shared Warehouse는 Template으로 유지하고
USER / GUEST별 Personal Warehouse를 생성했습니다.

```text
Shared Warehouse Template
        │
        ├──→ User A → Personal Warehouse A
        ├──→ User B → Personal Warehouse B
        └──→ Guest  → Personal Warehouse C
```

Personal Warehouse 생성 시 Simulation에 필요한 데이터를 함께 복제합니다.

```text
Warehouse
Zone
Node
Edge
ChargingStation
StorageLocation
WarehouseItem
Robot
Scenario
```

USER는 `user_id`,
GUEST는 `guest_session_id`를 기준으로 소유권을 관리합니다.

Simulation 실행 시에도 Warehouse Ownership을 다시 검증해
다른 사용자의 `warehouseId`를 직접 전달하는 방식의 접근을 차단했습니다.

### Deep Clone을 선택한 이유

실행 상태만 별도로 분리하는 구조도 고려할 수 있었지만,
2개월 MVP에서는

- 사용자 간 상태 격리를 명확하게 보장할 수 있고
- 구현 및 검증 범위를 비교적 명확하게 관리할 수 있다는 점

을 우선해 Personal Warehouse 전체를 Deep Clone하는 방식을 적용했습니다.

### Trade-off

Deep Clone은 상태 격리가 명확한 대신
Warehouse 데이터 규모와 사용자 수가 증가할수록

- 복제 비용
- 저장 공간
- 초기 생성 비용

이 증가할 수 있습니다.

따라서 서비스 규모가 커질 경우에는
공통 Warehouse Template은 공유하고

**Runtime State만 사용자별로 분리하거나
Copy-on-write 방식으로 변경하는 구조**를 검토할 수 있습니다.

---

## 2. PostgreSQL과 Neo4j 역할 분리

초기에는 Warehouse의 Node와 Edge를 PostgreSQL에 저장하고,
필요할 때 NetworkX를 사용해 Graph를 구성하는 방식도 검토했습니다.

NetworkX는 구현이 비교적 간단하고
BFS나 최단 경로와 같은 Graph Algorithm을 실행하기에는 충분했습니다.

하지만 프로젝트에서는 Graph를 단순한 일회성 계산에만 사용하는 것이 아니라,

- Warehouse의 Node–Edge 관계를 지속적으로 관리하고
- Backend에서 Graph 구조를 조회하며
- AI Planning에서도 동일한 이동 관계를 반복적으로 활용해야 했습니다.

따라서 관계 자체를 Graph 구조로 저장하고 조회할 수 있는
Neo4j를 사용했습니다.

다만 PostgreSQL과 Neo4j에 데이터를 독립적으로 관리하면
동일 데이터가 서로 달라지는 문제가 발생할 수 있습니다.

이를 줄이기 위해 역할을 다음과 같이 분리했습니다.

```text
PostgreSQL
= Source of Truth

Neo4j
= Graph Projection
```

Warehouse의 기준 데이터는 PostgreSQL에서 관리하고,
Neo4j는 Node·Edge 및 Warehouse 객체 간 관계를 표현하는
재생성 가능한 Graph Projection으로 사용했습니다.

### AFTER_COMMIT 기반 Graph Sync

두 저장소를 하나의 요청에서 직접 동시에 수정하면

```text
PostgreSQL 실패 / Neo4j 성공
PostgreSQL 성공 / Neo4j 실패
```

처럼 Partial Failure가 발생할 수 있습니다.

따라서 PostgreSQL Transaction이 정상적으로 Commit된 이후에만
Neo4j Graph Sync를 수행하도록 구성했습니다.

```text
Warehouse 데이터 변경
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
Neo4j Warehouse Graph Sync
```

```java
@TransactionalEventListener(
    phase = TransactionPhase.AFTER_COMMIT
)
```

`WarehouseGraphChangedEvent`는 변경된 데이터 전체를 전달하지 않고,
동기화가 필요한 `warehouseId`를 전달하는 Trigger 역할을 합니다.

Graph Sync 시 PostgreSQL에서 현재 Warehouse 데이터를 다시 조회해
해당 Warehouse 범위의 Graph를 갱신합니다.

이를 통해 PostgreSQL을 Source of Truth로 유지하면서
Neo4j를 재생성 가능한 Graph Projection으로 관리했습니다.

### 현재 구조의 한계

`AFTER_COMMIT`은 PostgreSQL Commit 이전에 Neo4j가 갱신되는 문제는 방지하지만,

```text
PostgreSQL COMMIT 성공
        ↓
Neo4j Graph Sync 실패
```

상황까지 자동으로 복구하지는 못합니다.

서비스 규모가 커질 경우에는
실패한 Sync 작업을 안정적으로 재처리할 수 있도록

**Retry 또는 Transactional Outbox 구조**를 적용하는 방향을 검토할 수 있습니다.

---

## 3. Warehouse Graph API 및 AI Planning 데이터 제공

AI Planning에서 현재 Warehouse의 이동 구조를 활용할 수 있도록
Node · Edge 기반 Graph 조회 API를 구현했습니다.

```text
PostgreSQL Warehouse
        ↓
WarehouseGraphService
        ↓
WarehouseGraphResponse
        ├─ nodeCode
        ├─ edgeCode
        └─ Node / Edge 속성
                ↓
           AI Planning
```

Backend 내부 PK에 직접 의존하기보다
AI Planning에서 Node와 Edge를 식별할 수 있도록
외부 식별자를 함께 제공했습니다.

이를 통해 Backend의 Warehouse 데이터와
AI Planning이 사용하는 Graph 구조를 연결했습니다.

---

# Troubleshooting

## 1. Backend와 AI의 Replanning 책임 중복

### 문제

초기에는 Backend에서 재계획 로직을 처리하는 구조를 구현했지만,
AI Planning 기능이 확장되면서 Backend와 AI가
동일한 재계획 책임을 가지는 문제가 발생했습니다.

### 원인

기존 Backend 중심 Replanning 구조에
AI Agent 기반 Replanning 기능이 추가되면서
기존 책임을 그대로 유지한 채 기능이 확장되었습니다.

### 개선

각 서비스의 책임을 다시 정의했습니다.

```text
AI Planning
→ 요청 해석
→ Mission / Constraints 구성
→ 계획 생성 및 재계획

Optimization Solver
→ 작업 배정
→ 수행 순서 최적화

MAPF
→ 충돌 없는 다중 Robot 이동 계획

Backend
→ 요청 검증
→ Runtime State 관리
→ Plan 적용
```

Backend에서 중복되는 Replanning 판단 책임을 제거하고,
AI는 Planning에 집중하고
Backend는 실제 실행 상태 관리와 Plan 적용을 담당하도록 변경했습니다.

---

## 2. Personal Warehouse 복제 후 Scenario 참조 불일치

### 문제

Shared Warehouse를 Personal Warehouse로 Deep Clone하면
Scenario도 함께 복제되면서 새로운 ID가 생성됩니다.

하지만 기존 Template의 Scenario ID를 그대로 사용할 경우
Personal Warehouse와 Scenario 간 참조가 일치하지 않는 문제가 발생했습니다.

```text
Shared Warehouse
Scenario #10
      ↓
Deep Clone
      ↓
Personal Warehouse
Scenario #25
```

### 원인

Warehouse와 Scenario가 복제되면서 새로운 ID가 생성되지만,
기존 흐름에서는 Template Scenario의 ID를 계속 사용하고 있었습니다.

### 개선

Personal Warehouse 생성 이후
해당 Warehouse의 Scenario 목록을 다시 조회하고,
Scenario Code 또는 Name을 기준으로
복제된 Personal Scenario를 다시 연결했습니다.

```text
Shared Scenario
      ↓
Personal Warehouse 생성
      ↓
Personal Scenario 조회
      ↓
Scenario 재매칭
      ↓
Personal Scenario ID
      ↓
SimulationRun 생성
```

이를 통해 SimulationRun이
Personal Warehouse에 복제된 Scenario를 참조하도록 수정했습니다.

---

## 3. Backend · AI 간 재고 판정 기준 불일치

### 문제

`quantity = 0`인 WarehouseItem이 존재하는 상황에서
실제 사용할 수 있는 공간이 있음에도
AI Planning이 재고 상태를 잘못 판단해
Plan 생성이 실패하는 문제가 발생했습니다.

```text
Warehouse Inventory
        ↓
Backend 재고 판정
        ↓
AI Planning 입력
        ↓
재고 상태 오판
        ↓
Planning 실패
```

### 원인

Backend와 AI가
`quantity = 0` 데이터를 처리하는 기준이 달랐기 때문에
동일한 Warehouse 상태를 서로 다르게 해석하고 있었습니다.

### 개선

Backend와 AI의 재고 판정 조건을 함께 확인하고,
`quantity = 0` 데이터를 동일한 기준으로 처리하도록 수정했습니다.

```text
Backend
재고 판정 기준
        ↓
동일한 Quantity 기준
        ↓
AI Planning
재고 판정 기준
```

이를 통해 Backend와 AI가
동일한 Warehouse 상태를 기준으로 판단하도록 맞췄습니다.

---

# 프로젝트 검증 결과

고부하 Scenario에서
동일한 Optimization Solver를 사용하고
Rule 방식과 Agent 기반 의사결정 방식을 비교했습니다.

| Scenario | Rule | Agent |
|---|---:|---:|
| 분산 출고 | 500.2s | 200.6s |
| 혼합 출고 | 196.5s | 78.3s |
| 동일 SKU | 587.5s | 210.6s |

Agent 방식은 작업량과 Robot 상태를 고려해
여러 Robot에 작업을 분산했으며,

**동일 SKU Scenario에서는 작업 완료시간이 최대 64% 단축되었습니다.**

> 동일 Solver 조건에서  
> 의사결정 방식에 따른 실행 결과를 비교한 팀 프로젝트 실험 결과입니다.

---

# Project Tech Stack

### Backend

`Java 17` `Spring Boot` `Spring Data JPA`  
`Spring Security` `WebSocket / STOMP`

### Database / Cache

`PostgreSQL` `Redis` `Neo4j`

### AI / Optimization

`FastAPI` `GPT-5-mini` `LangGraph`  
`NVIDIA cuOpt` `OR-Tools` `MAPF`

### Frontend

`React` `Vite` `JavaScript`

### Infra

`AWS` `Docker` `ECR` `ECS` `S3` `CloudFront`

---

# Documentation

상세 Backend 설계와 구현 내용은 별도 문서로 정리했습니다.

| 문서 | 내용 |
|---|---|
| [01. Project Overview](./docs/01-project-overview.md) | 프로젝트 구조 및 담당 영역 |
| [02. Backend Design](./docs/02-backend-design.md) | Backend 책임과 서비스 간 역할 분리 |
| [03. Warehouse Domain](./docs/03-warehouse-domain.md) | Warehouse · Zone · Node · Edge 설계 |
| [04. Digital Twin Graph](./docs/04-digital-twin-graph.md) | PostgreSQL · Neo4j Graph Sync |
| [05. Warehouse-AI Integration](./docs/05-warehouse-ai-integration.md) | Warehouse Graph 및 AI Planning 데이터 연동 |
| [06. Multi-user Isolation](./docs/06-multi-user-isolation.md) | 사용자별 Digital Twin 실행 환경 격리 |

---

# Repository

### Portfolio Repository

https://github.com/pbjun2000/digital-twin-warehouse-backend

### Team Project

https://github.com/kt-aivle-big-project
