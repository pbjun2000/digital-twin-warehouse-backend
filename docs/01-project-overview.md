# 01. Project Overview

> **LARO — Digital Twin 기반 자율 창고 운영 및 다중 로봇 작업 최적화 시스템**

---

## 1. 프로젝트 개요

LARO(LLM Autonomous Robot Orchestration)는  
창고·재고·로봇의 현재 상태를 기반으로 AI가 작업 계획을 생성하고,
다중 로봇의 작업 배정·이동 계획·실시간 실행·재계획까지 연결하는 창고 운영 시스템입니다.

프로젝트의 핵심은 개별 로봇의 단순 경로 탐색이 아니라,

**Digital Twin 기반 상태 통합 → AI Planning → 작업 최적화 → 다중 로봇 이동 계획 → 실시간 관제 → Dynamic Replanning**

을 하나의 운영 흐름으로 구성하는 것입니다.

---

## 2. 해결하고자 한 문제

다수의 로봇이 동일한 창고에서 작업하면
각 로봇의 이동 경로만 독립적으로 계산하는 것으로는 전체 운영을 안정적으로 조율하기 어렵습니다.

운영 중에는 다음과 같은 상태 변화가 계속 발생합니다.

- 신규 입·출고 작업 발생
- 로봇 위치·작업 상태 변화
- 배터리 상태 변화
- 통로 차단
- 재고 및 저장 위치 상태 변화

또한 작업 배정, 이동 경로, 충돌 가능성과 현재 로봇 상태가 서로 영향을 주기 때문에
여러 정보를 함께 고려할 수 있는 운영 구조가 필요했습니다.

LARO는 현재 Warehouse 상태를 기준으로 실행 계획을 생성하고,
실행 중 조건이 변경되면 현재 상태를 유지하면서 계획을 다시 구성하는 방식으로 이를 해결합니다.

---

## 3. 전체 처리 흐름

```text
사용자 요청 / 작업 정보
        ↓
Spring Boot
Warehouse · Robot · Task 상태 구성
        ↓
FastAPI AI Planning Server
        ↓
Rule / Agent Routing
        ↓
Mission · Constraints 구성
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
Plan 검증 · 실행 데이터 연결
        ↓
Simulation Playback
        ↓
Redis Runtime State
        ↓
WebSocket / STOMP
        ↓
Frontend 실시간 관제
```

LLM이 생성한 결과를 그대로 실행하지 않고,
Solver와 Backend 검증 단계를 거쳐 실제 서비스의 Task와 Simulation 데이터에 연결하도록 구성했습니다.

---

## 4. 시스템 구성

### Backend

Spring Boot가 서비스의 중심에서 다음 책임을 담당합니다.

- 사용자 인증 및 Resource 접근 제어
- Warehouse / Robot / Task 데이터 관리
- AI Planning 요청 데이터 구성
- AI 결과 검증 및 실행 데이터 Mapping
- Simulation 실행 상태 관리
- 실시간 Runtime State 전달

### AI Planning

FastAPI 기반 AI Server는 현재 창고 상태와 사용자 요청을 기반으로
실행 가능한 작업 계획을 구성합니다.

```text
Natural Language / Structured Operation
                ↓
         Rule / Agent
                ↓
      Mission · Constraints
                ↓
       Optimization Solver
                ↓
              MAPF
```

LLM과 Solver의 책임을 분리하여
자연어 해석과 조건 정식화는 AI가 담당하고,
실제 작업 배정과 이동 계획 계산은 최적화 영역에서 처리합니다.

### Data Layer

| 저장소 | 역할 |
|---|---|
| **PostgreSQL** | Warehouse, Robot, Inventory, Task, Scenario 등 영속 데이터 |
| **Redis** | Robot 위치·배터리·실행 상태 등 Runtime State |
| **Neo4j** | Node·Edge 기반 Warehouse 이동·접근 관계 |

PostgreSQL을 기준 데이터로 사용하고,
Neo4j는 Warehouse 이동 관계를 탐색하기 위한 Graph Projection으로 구성했습니다.

---

## 5. 담당 영역

팀 내 Backend 개발자로서  
**Warehouse · Robot · Graph와 AI 실행 환경 연동 영역**을 담당했습니다.

### Warehouse Domain

- Warehouse / Zone / Node / Edge API 설계 및 구현
- Charging Station 및 Storage 관련 데이터 연동
- Warehouse Layout 통합 조회
- JSON 기반 창고 구조 데이터 연동

### Digital Twin Graph

- PostgreSQL Warehouse 데이터와 Neo4j Graph 연동
- Warehouse 단위 Graph Sync 구조 구현
- `WarehouseGraphChangedEvent` 기반 변경 감지
- `AFTER_COMMIT` 이후 Neo4j 동기화

### Multi-user Simulation

- Shared Warehouse → Personal Warehouse Deep Clone
- USER / GUEST별 독립 실행 환경 구성
- `user_id` / `guest_session_id` 기반 소유권 관리
- 다른 사용자의 Warehouse 직접 접근 차단

### AI / Service Integration

- AI Plan과 Backend 실행 데이터 연결
- AI Task ID → Backend Task ID Mapping
- Simulation 실행 구조 연동
- Backend / AI / Frontend 통합 과정 참여

---

## 6. Backend 핵심 설계 포인트

### 사용자별 실행 환경 분리

공유 Warehouse를 여러 사용자가 동시에 실행할 경우
Inventory와 Robot 상태가 서로 영향을 주는 문제를 해결하기 위해
실행 환경을 사용자 단위로 분리했습니다.

```text
Shared Warehouse Template
          ↓
Personal Warehouse Deep Clone
          ↓
USER / GUEST Ownership
          ↓
Independent Simulation
```

---

### PostgreSQL → Neo4j 동기화

PostgreSQL을 Warehouse 데이터의 Source of Truth로 유지하고
Transaction이 성공한 경우에만 Neo4j Graph를 갱신하도록 구성했습니다.

```text
PostgreSQL 변경
      ↓
Transaction COMMIT
      ↓
WarehouseGraphChangedEvent
      ↓
AFTER_COMMIT
      ↓
Neo4j Graph Sync
```

---

### AI Plan 실행 연결

AI가 반환한 계획을 즉시 실행하지 않고
Backend에서 실행 상태와 Task 관계를 검증한 후 Simulation에 적용합니다.

```text
AI Plan
   ↓
READY 상태 확인
   ↓
AI Task ID ↔ BE Task ID
   ↓
Robot Plan / Step
   ↓
Simulation Playback
```

---

## 7. 기술 스택

| 영역 | 기술 |
|---|---|
| Backend | Java 17, Spring Boot, Spring Data JPA, Spring Security |
| AI | Python, FastAPI, GPT-5-mini, LangGraph |
| Optimization | NVIDIA cuOpt, OR-Tools, MAPF |
| Database | PostgreSQL, Redis, Neo4j |
| Frontend | React, Vite, JavaScript |
| Realtime | WebSocket, STOMP |
| Infrastructure | AWS, Docker |

---

## 8. 상세 문서

프로젝트의 세부 설계와 구현 과정은 다음 문서에서 설명합니다.

| 문서 | 내용 |
|---|---|
| [02. Backend Design](./02-backend-design.md) | Backend 책임과 서비스 간 역할 분리 |
| [03. Warehouse Domain](./03-warehouse-domain.md) | Warehouse·Zone·Node·Edge 설계 |
| [04. Digital Twin Graph](./04-digital-twin-graph.md) | PostgreSQL·Neo4j Graph Sync |
| [05. AI Integration](./05-ai-integration.md) | AI Plan과 Backend 실행 데이터 연동 |
| [06. Multi-user Isolation](./06-guest-access.md) | USER/GUEST 실행 환경 격리 |

---

[← README](../README.md)
