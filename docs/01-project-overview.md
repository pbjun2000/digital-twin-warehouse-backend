# 01. Project Overview

> LARO — Digital Twin 기반 자율 창고 운영 및 다중 로봇 작업 최적화 시스템

---

## 1. 프로젝트 소개

LARO(LLM Autonomous Robot Orchestration)는  
창고·재고·로봇의 현재 상태를 기반으로 작업 계획을 생성하고,
여러 로봇의 작업 배정과 이동 계획을 최적화하여 Simulation으로 실행·관제하는 시스템입니다.

운영 중 작업 조건이나 로봇 상태가 변경되면
현재 실행 상태를 반영해 계획을 다시 구성할 수 있도록 설계했습니다.

```text
Digital Twin
     ↓
AI Planning
     ↓
Task Optimization
     ↓
Multi-Robot Path Planning
     ↓
Simulation
     ↓
Real-time Monitoring
     ↓
Dynamic Replanning
```

---

## 2. 해결하고자 한 문제

다수의 로봇이 하나의 창고에서 작업하면
각 로봇의 최단 경로만 계산하는 것만으로는 전체 운영을 조율하기 어렵습니다.

실제 실행 중에는 다음과 같은 상태가 계속 변합니다.

- 신규 입·출고 작업
- Robot 위치 및 작업 상태
- Battery 상태
- 통로 사용 가능 여부
- Inventory 및 저장 위치

작업 배정과 이동 경로도 서로 영향을 주기 때문에
현재 Warehouse 상태를 하나의 실행 환경으로 관리하고,
변경된 상태를 다시 Planning에 반영할 수 있는 구조가 필요했습니다.

---

## 3. 전체 처리 흐름

```text
사용자 요청 / 작업 정보
        ↓
Spring Boot
Warehouse · Robot · Task 상태 관리
        ↓
FastAPI AI Planning Server
        ↓
요청 분류 · Routing
Rule / Agent 처리 또는 Human Review 분기
        ↓
Mission · Constraints
        ↓
Optimization Solver
작업 배정 · 수행 순서 최적화
        ↓
MAPF
다중 로봇 이동 계획
        ↓
MOVE · WAIT · SERVICE
        ↓
Spring Boot
Plan 검증 및 Simulation 적용
        ↓
Runtime State
        ↓
Frontend 실시간 관제
```

Frontend, Backend, AI Planning Server가 각각 역할을 나누고,
Backend가 관리하는 Warehouse와 Robot 데이터를 기준으로
Planning과 Simulation이 동일한 실행 환경을 사용하도록 구성했습니다.

---

## 4. 시스템 구성

| 영역 | 주요 역할 |
|---|---|
| **Frontend** | 창고 편집, AI 요청, Simulation 및 Robot 상태 시각화 |
| **Spring Boot** | Warehouse·Robot·Task 관리, 권한 검증, AI 연동, Simulation 실행 |
| **AI Planning** | 자연어 요청 해석, Rule/Agent Routing 및 Human Review 분기, Mission·Constraint 구성, 작업 최적화 및 재계획 |
| **Optimization / MAPF** | Robot 작업 배정, 수행 순서 및 다중 로봇 이동 계획 계산 |
| **PostgreSQL** | Warehouse, Robot, Inventory, Task, Scenario 등 영속 데이터 |
| **Redis** | Robot 위치·배터리·실행 상태 등 Runtime State |
| **Neo4j** | Node·Edge와 창고 객체 간 이동·접근 관계 |

PostgreSQL은 서비스의 기준 데이터를 관리하고,
Redis와 Neo4j는 각각 Runtime State와 Graph 탐색이라는 목적에 맞게 사용했습니다.

---

## 5. 담당 영역

팀 내 Backend 개발을 담당하며  
**Warehouse · Robot · Graph와 사용자별 실행 환경 영역**을 중심으로 구현했습니다.

### Warehouse Domain

- Warehouse / Zone / Node / Edge API 설계 및 구현
- Charging Station 및 Storage 데이터 연동
- Robot 및 Warehouse Layout 데이터 연동
- Warehouse Layout 통합 조회

### Digital Twin Graph

- PostgreSQL Warehouse 데이터와 Neo4j Graph 연동
- `WarehouseGraphChangedEvent` 기반 Graph Sync 구현
- Transaction Commit 이후 Neo4j가 동기화되도록 `AFTER_COMMIT` 구조 적용
- Warehouse 단위 Graph Scope 구성

### Multi-user Simulation

- Shared Warehouse 기반 Personal Warehouse 생성
- USER / GUEST별 독립적인 실행 환경 구성
- Warehouse Resource 소유권 검증 및 접근 제어
- Zone · Node · Edge · Inventory · Robot · Scenario 등 실행 데이터 분리

### Warehouse / AI Data Integration

- AI Planning에서 활용할 Node · Edge 기반 Warehouse Graph API 구현
- DB Numeric PK와 외부 `nodeCode` / `edgeCode` 식별 구조 분리
- Backend / AI / Frontend 통합 과정에서 Warehouse 데이터 연동 참여

세부 구현은 각 기술 문서에서 별도로 정리했습니다.

---

## 6. 상세 문서

| 문서 | 내용 |
|---|---|
| [02. Backend Design](./02-backend-design.md) | Backend 책임과 서비스 간 역할 분리 |
| [03. Warehouse Domain](./03-warehouse-domain.md) | Warehouse·Zone·Node·Edge 설계 |
| [04. Digital Twin Graph](./04-digital-twin-graph.md) | PostgreSQL·Neo4j Graph Sync |
| [05. Warehouse-AI Integration](./05-warehouse-ai-integration.md) | Warehouse Graph 및 AI Planning 데이터 제공 |
| [06. Multi-user Isolation](./06-multi-user-isolation.md) | 사용자별 Digital Twin 실행 환경 격리 |

---

[← README](../README.md)
