# LARO

> Digital Twin 기반 자율 창고 운영 및 다중 로봇 작업 최적화 시스템

## 프로젝트 소개

**LARO**는 창고 구조와 로봇 상태를 Digital Twin으로 관리하고, 여러 로봇의 작업 할당과 이동 경로를 최적화하는 자율 창고 운영 시스템입니다.

창고의 이동 구조를 Node와 Edge 기반 그래프로 표현하고, 로봇의 위치·배터리·작업 상태를 통합 관리하여 충돌과 정체를 줄이고 효율적인 창고 운영을 지원합니다.

본 저장소는 팀 프로젝트에서 담당한 백엔드 설계, 구현 및 문제 해결 과정을 정리한 개인 포트폴리오 저장소입니다.

---

## 선정 배경 및 문제 정의

물류창고에서 여러 로봇이 각각의 최단 경로만 따라 이동하면 좁은 통로나 교차 구간에서 충돌, 대기 및 정체가 발생할 수 있습니다.

또한 창고 구조, 로봇 상태와 작업 정보가 분산되어 있으면 관리자가 전체 운영 상황을 파악하고 예외 상황에 대응하기 어렵습니다.

이를 해결하기 위해 다음 기능을 제공하는 시스템을 기획했습니다.

- 창고 구조와 로봇 상태의 Digital Twin 관리
- 다중 로봇의 작업 할당 및 이동 경로 최적화
- 충돌, 지연 및 정체 상황 탐지
- 장애물과 배터리 부족 발생 시 경로 재계산
- 최적화 적용 전후의 운영 성능 비교

---

## 주요 기능

### 창고 레이아웃 관리

- 창고, 이동 노드 및 연결 경로 관리
- 노드 간 거리와 단방향·양방향 이동 관리
- 보관, 이동, 입고, 출고 및 충전 구역 관리

### 로봇 및 작업 관리

- 로봇 위치, 배터리와 작업 상태 관리
- 입출고 작업 생성 및 로봇 할당
- 충전소 위치와 사용 상태 관리

### 다중 로봇 경로 최적화

- cuOpt 기반 작업 할당 및 경로 최적화
- MAPF 기반 다중 로봇 충돌 방지
- 장애물과 통로 차단 발생 시 경로 재계산

### 실시간 관제

- Redis 기반 실시간 상태 관리
- WebSocket 기반 로봇 위치 및 상태 전달
- 충돌, 지연 및 정체 상황 시각화

### AI 운영 지원

- 관리자의 자연어 명령을 작업 계획으로 변환
- 최적화 결과와 운영 현황 설명
- KPI 기반 운영 리포트 제공

---

## 담당 역할

백엔드에서 **창고 레이아웃 및 이동 그래프 관리, 경로 최적화 서버 연동 영역**을 담당하고 있습니다.

- Warehouse, WarehouseNode, WarehouseEdge, WarehouseZone API 구현
- Node와 Edge 기반 창고 이동 그래프 설계
- 이동 거리 및 단방향·양방향 통로 모델링
- ChargingStation 및 창고 레이아웃 통합 조회 구현
- PostgreSQL과 Neo4j 데이터 연동
- FastAPI 경로 최적화 서버 연동
- 경로 계산 및 재계산 결과 저장

---

## 기술 스택

### Backend

![Java](https://img.shields.io/badge/Java_17-007396?style=flat-square&logo=openjdk&logoColor=white)
![Spring Boot](https://img.shields.io/badge/Spring_Boot-6DB33F?style=flat-square&logo=springboot&logoColor=white)
![Spring Data JPA](https://img.shields.io/badge/Spring_Data_JPA-6DB33F?style=flat-square&logo=spring&logoColor=white)
![Gradle](https://img.shields.io/badge/Gradle-02303A?style=flat-square&logo=gradle&logoColor=white)

### Database

![PostgreSQL](https://img.shields.io/badge/PostgreSQL-4169E1?style=flat-square&logo=postgresql&logoColor=white)
![Redis](https://img.shields.io/badge/Redis-DC382D?style=flat-square&logo=redis&logoColor=white)
![Neo4j](https://img.shields.io/badge/Neo4j-4581C3?style=flat-square&logo=neo4j&logoColor=white)

### AI 및 최적화

- FastAPI
- NVIDIA cuOpt
- MAPF
- NetworkX

### Infrastructure

![Docker](https://img.shields.io/badge/Docker-2496ED?style=flat-square&logo=docker&logoColor=white)
![Git](https://img.shields.io/badge/Git-F05032?style=flat-square&logo=git&logoColor=white)
![GitHub](https://img.shields.io/badge/GitHub-181717?style=flat-square&logo=github&logoColor=white)

---

## 시스템 아키텍처

```mermaid
flowchart TD
    USER[관리자]
    FE[React Frontend]
    BE[Spring Boot API Server]

    POSTGRES[(PostgreSQL)]
    REDIS[(Redis)]
    NEO4J[(Neo4j)]
    FASTAPI[FastAPI AI Server]

    CUOPT[cuOpt]
    MAPF[MAPF]
    SIM[Simulation]

    USER --> FE
    FE --> BE

    BE --> POSTGRES
    BE --> REDIS
    BE --> NEO4J
    BE --> FASTAPI

    FASTAPI --> CUOPT
    FASTAPI --> MAPF
    FASTAPI --> SIM
```

| 구성 요소 | 역할 |
|---|---|
| React | 창고 구조, 로봇 상태와 시뮬레이션 결과를 시각화합니다. |
| Spring Boot | 창고, 로봇, 작업과 경로 관련 API를 제공합니다. |
| PostgreSQL | 창고, 로봇, 작업과 실행 이력을 저장합니다. |
| Redis | 로봇 위치, 배터리와 실시간 상태를 관리합니다. |
| Neo4j | Node와 Edge 기반 창고 연결 구조를 관리합니다. |
| FastAPI | 경로 최적화와 시뮬레이션을 수행합니다. |
| cuOpt | 다중 로봇의 작업 할당과 경로를 최적화합니다. |
| MAPF | 여러 로봇의 충돌 없는 이동 경로를 탐색합니다. |

---

## 기대 효과

- 다중 로봇의 충돌과 교착 상황 감소
- 로봇의 이동 거리와 평균 대기 시간 감소
- 입출고 작업 처리 시간 단축
- 창고 구조와 로봇 상태의 통합 관제
- 장애물 및 배터리 부족 상황에 대한 신속한 대응
- 최적화 적용 전후의 운영 성능 정량화
- 자연어 명령을 통한 관리자 운영 편의성 향상

---

## 개발 기록

프로젝트 진행 과정에서 담당한 기능, 설계 결정과 문제 해결 과정을 주차별로 정리합니다.

| 기간 | 주요 내용 | 문서 |
|---|---|---|
| 주제 선정 | 문제 정의, 후보 주제 비교 및 기술 검토 | [주제 선정 과정](docs/00-topic-selection.md) |
| 1주차 | 개발 환경 구성 및 Warehouse 도메인 구현 | [1주차 개발 기록](docs/week01-environment-and-warehouse.md) |
| 2주차 | Node, Edge, Zone 설계 및 CRUD 구현 | [2주차 개발 기록](docs/week02-node-edge-zone-crud.md) |
