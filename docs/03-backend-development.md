# 백엔드 개발 1차 기록

## 1. 개발 진행 배경

프로젝트 1~2주차에는 후보 주제를 조사하고 비교한 뒤, 최종적으로 다음 주제를 선정했습니다.

> **LARO: Digital Twin 기반 자율 창고 운영 및 다중 로봇 작업 최적화 시스템**

3주차 초반에는 서비스 흐름, 시스템 구성, 데이터 저장 구조, 백엔드 역할 분담과 ERD를 구체화했습니다.

이후 3주차 후반부터 실제 개발을 시작했으며, 첫 2일 동안 다음 작업을 진행했습니다.

- Spring Boot 프로젝트 실행 환경 확인
- PostgreSQL, Redis, Neo4j 로컬 개발환경 구성
- Local 및 Docker Profile 분리
- Warehouse CRUD 구현 및 테스트
- WarehouseNode CRUD 구현 및 테스트
- WarehouseEdge CRUD 구현 및 테스트
- WarehouseZone CRUD 구현 및 테스트
- 기능별 브랜치, Pull Request 및 병합 진행

---

## 2. 개인 담당 영역

백엔드 3인 중 창고 구조와 이동 그래프, 경로 최적화 서버 연동 영역을 담당했습니다.

### 담당 범위

- Warehouse 관리
- WarehouseNode 관리
- WarehouseEdge 관리
- WarehouseZone 관리
- ChargingStation 관리
- 창고 레이아웃 통합 조회
- Neo4j 그래프 동기화
- FastAPI 경로 최적화 서버 연동
- 경로 계산 및 재계산 결과 저장

이번 개발에서는 향후 경로 최적화에 사용할 창고 레이아웃 데이터를 먼저 관리할 수 있도록 Warehouse, Node, Edge와 Zone API를 구현했습니다.

---

## 3. 개발환경 구성

### 사용 기술

- Java 17
- Spring Boot
- Spring Data JPA
- PostgreSQL
- Redis
- Neo4j
- Docker
- Docker Compose
- Gradle
- Postman
- Git 및 GitHub

### Docker Compose 구성

프로젝트에서 사용하는 데이터 저장소를 Docker Compose로 실행했습니다.

| 서비스 | 역할 | 포트 |
|---|---|---:|
| PostgreSQL | 창고, 로봇, 작업 등 정형 데이터 저장 | 5432 |
| Redis | 로봇 위치, 배터리 등 실시간 상태 관리 | 6379 |
| Neo4j | Node와 Edge 기반 창고 그래프 관리 | 7474, 7687 |

로컬 개발 중에는 Spring Boot를 IDE 또는 터미널에서 직접 실행하고, 데이터 저장소만 Docker 컨테이너로 실행하는 방식을 사용했습니다.

---

## 4. 실행 환경 분리

Docker 내부에서 접근할 때와 로컬 Spring Boot에서 접근할 때 데이터베이스 주소가 달라지는 문제를 방지하기 위해 Spring Profile을 분리했습니다.

```text
application.yaml
application-local.yaml
application-docker.yaml
```

### Local Profile

로컬에서 실행한 Spring Boot가 Docker 컨테이너에 접근합니다.

```text
PostgreSQL : localhost:5432
Redis      : localhost:6379
Neo4j      : localhost:7687
```

### Docker Profile

Spring Boot까지 Docker Compose로 실행할 경우 서비스 이름을 사용합니다.

```text
PostgreSQL : postgres:5432
Redis      : redis:6379
Neo4j      : neo4j:7687
```

환경별 설정을 분리하여 로컬 개발 설정과 컨테이너 실행 설정이 섞이지 않도록 구성했습니다.

---

## 5. Warehouse 도메인 구현

Warehouse는 창고 레이아웃 데이터가 소속되는 기준이 되는 도메인입니다.

### 주요 역할

- 창고 기본 정보 관리
- 창고 크기와 레이아웃 정보 관리
- 사용자와 창고의 소유 관계 관리
- Node, Zone 등 하위 데이터의 기준 제공

### 구현 구조

```text
Controller
  ↓
Service
  ↓
Repository
  ↓
PostgreSQL
```

클라이언트 요청과 Entity를 직접 연결하지 않고 요청 및 응답 DTO를 분리했습니다.

```text
WarehouseCreateRequest
WarehouseUpdateRequest
WarehouseResponse
```

Entity에는 Setter를 두지 않고 생성 및 수정 메서드를 통해 상태를 변경하도록 구성했습니다.

---

## 6. WarehouseNode 도메인 구현

WarehouseNode는 창고 내에서 로봇이 이동할 수 있는 지점을 좌표로 표현합니다.

### 주요 데이터

- Node ID
- 소속 Warehouse
- X 좌표
- Y 좌표
- 구역 식별 정보

Node를 단순 화면 좌표가 아니라 경로 탐색 그래프의 정점으로 사용할 수 있도록 별도 도메인으로 분리했습니다.

### API

```http
POST   /api/warehouse-nodes
GET    /api/warehouse-nodes
GET    /api/warehouse-nodes/{nodeId}
PATCH  /api/warehouse-nodes/{nodeId}
DELETE /api/warehouse-nodes/{nodeId}
```

Postman을 통해 생성, 전체 조회, 단건 조회, 수정 및 삭제 동작을 확인했습니다.

---

## 7. WarehouseEdge 도메인 구현

WarehouseEdge는 두 WarehouseNode 사이의 연결 관계를 표현합니다.

### 주요 데이터

- 출발 노드
- 도착 노드
- 이동 거리
- 이동 방향

이동 방향은 Enum으로 관리했습니다.

```java
public enum DirectionType {
    BOTH,
    A_TO_B,
    B_TO_A
}
```

### 방향별 의미

| 값 | 설명 |
|---|---|
| BOTH | 출발 노드와 도착 노드 사이 양방향 이동 가능 |
| A_TO_B | 출발 노드에서 도착 노드 방향으로만 이동 가능 |
| B_TO_A | 도착 노드에서 출발 노드 방향으로만 이동 가능 |

이를 통해 양방향 통로뿐 아니라 일방통행 통로도 표현할 수 있도록 설계했습니다.

### 생성 요청 예시

```http
POST /api/warehouse-edges
```

```json
{
  "fromNodeId": 1,
  "toNodeId": 2,
  "distance": 10.0,
  "directionType": "BOTH"
}
```

### 응답 예시

```json
{
  "id": 1,
  "fromNodeId": 1,
  "toNodeId": 2,
  "distance": 10.0,
  "directionType": "BOTH"
}
```

---

## 8. WarehouseZone 도메인 구현

WarehouseZone은 창고 내부 공간을 용도별로 구분하는 도메인입니다.

### ZoneType

```java
public enum ZoneType {
    STORAGE,
    MOVING,
    INBOUND,
    OUTBOUND,
    CHARGING,
    RESTRICTED
}
```

### 주요 데이터

- 소속 Warehouse
- 구역 이름
- 구역 유형
- 구역 설명
- 최소 X 좌표
- 최대 X 좌표
- 최소 Y 좌표
- 최대 Y 좌표

구역의 최소·최대 좌표를 저장하여 직사각형 형태의 창고 영역을 표현하도록 구현했습니다.

### 생성 요청 예시

```http
POST /api/warehouse-zones
```

```json
{
  "warehouseId": 1,
  "name": "A구역",
  "zoneType": "STORAGE",
  "description": "완제품 보관 구역",
  "minX": 0.0,
  "maxX": 100.0,
  "minY": 0.0,
  "maxY": 50.0
}
```

---

## 9. 공통 CRUD 구조

WarehouseNode, WarehouseEdge와 WarehouseZone에 다음 CRUD API를 구현했습니다.

| 기능 | HTTP Method | 정상 응답 |
|---|---|---|
| 생성 | POST | 201 Created |
| 전체 조회 | GET | 200 OK |
| 단건 조회 | GET | 200 OK |
| 수정 | PATCH | 200 OK |
| 삭제 | DELETE | 204 No Content |

각 기능은 다음 계층으로 구성했습니다.

```text
Controller
  ↓
Request DTO
  ↓
Service
  ↓
Repository
  ↓
Entity
  ↓
Response DTO
```

Postman을 사용하여 생성부터 삭제까지의 전체 흐름을 확인했습니다.

---

## 10. Entity 상태 변경 방식

Entity에 Lombok `@Setter`를 사용하지 않고, 생성과 수정에 필요한 메서드를 Entity 내부에 정의했습니다.

### 생성

```java
WarehouseEdge.create(
    fromNode,
    toNode,
    distance,
    directionType
);
```

### 수정

```java
warehouseEdge.update(
    fromNode,
    toNode,
    distance,
    directionType
);
```

이를 통해 Entity 상태가 변경되는 위치를 제한하고, 도메인 객체의 상태 변경 의도를 코드에서 명확하게 드러내고자 했습니다.

요청 DTO에는 JSON 역직렬화를 위해 Getter, Setter와 기본 생성자를 사용했습니다.

---

## 11. 문제 해결 기록

### 11.1 Spring Boot가 `.env` 값을 읽지 못하는 문제

#### 문제

Docker Compose에서는 `.env` 파일의 PostgreSQL 설정이 적용됐지만, 로컬에서 Spring Boot를 실행했을 때 환경변수 값이 주입되지 않았습니다.

설정값 대신 다음 문자열이 그대로 전달되었습니다.

```text
${POSTGRES_USER}
${POSTGRES_PASSWORD}
```

#### 원인

Docker Compose는 프로젝트 루트의 `.env` 파일을 자동으로 읽지만, 로컬에서 실행한 Spring Boot는 해당 파일을 자동으로 환경변수로 등록하지 않습니다.

#### 해결

PowerShell에서 PostgreSQL 환경변수를 직접 등록한 후 서버를 실행했습니다.

```powershell
$env:POSTGRES_DB="warehouse"
$env:POSTGRES_USER="warehouse"
$env:POSTGRES_PASSWORD="warehouse1234"

.\gradlew.bat bootRun
```

이를 통해 로컬 Spring Boot가 PostgreSQL에 정상적으로 연결되는 것을 확인했습니다.

---

### 11.2 Docker backend와 로컬 Spring Boot의 포트 충돌

#### 문제

`docker compose up -d` 실행 시 backend 컨테이너도 함께 실행되었고, 로컬 Spring Boot와 동일한 8080 포트를 사용하면서 서버 실행이 실패했습니다.

#### 해결

로컬 개발 시 backend 컨테이너만 중지하고 데이터 저장소는 계속 실행했습니다.

```powershell
docker compose stop backend
```

이후 로컬 Spring Boot를 실행했습니다.

```powershell
.\gradlew.bat bootRun
```

로컬 개발과 전체 Docker 실행 방식을 구분하여 포트 충돌을 해결했습니다.

---

### 11.3 Edge 생성 시 Node ID가 null로 전달되는 것으로 판단한 문제

#### 문제

WarehouseEdge 생성 요청에서 다음 오류가 발생했습니다.

```text
The given id must not be null
```

처음에는 요청 DTO에 `fromNodeId` 또는 `toNodeId`가 바인딩되지 않은 것으로 판단했습니다.

#### 확인 과정

- Postman 요청 URL 확인
- Body의 JSON 형식 확인
- DTO 필드명 확인
- DTO Getter 및 Setter 확인
- Controller에 요청 값 출력 코드 추가

이후 서버 로그를 다시 확인한 결과, 실제 요청이 Controller에 도달하지 못하고 있음을 확인했습니다.

---

### 11.4 WarehouseEdge Controller가 등록되지 않는 문제

#### 문제

`POST /api/warehouse-edges` 요청 시 Edge Controller가 실행되지 않고 다음 오류가 발생했습니다.

```text
No static resource api/warehouse-edges
```

Spring MVC가 요청을 처리할 Controller를 찾지 못하고 정적 리소스 요청으로 처리하고 있었습니다.

#### 원인

Windows 환경에서 Controller 패키지 폴더명을 변경한 뒤, 이전 빌드 결과가 남아 새 Controller가 정상적으로 반영되지 않은 것으로 판단했습니다.

#### 해결

서버를 종료한 뒤 기존 빌드 결과를 제거하고 다시 실행했습니다.

```powershell
.\gradlew.bat clean bootRun
```

재빌드 후 Edge 생성 API가 정상적으로 등록되었고 다음 응답을 확인했습니다.

```text
201 Created
```

이후 전체 조회, 단건 조회, 수정 및 삭제까지 정상 동작하는 것을 확인했습니다.

---

## 12. 테스트 결과

### WarehouseEdge

| 테스트 | 결과 |
|---|---|
| Edge 생성 | 201 Created |
| 전체 조회 | 200 OK |
| 단건 조회 | 200 OK |
| 거리 및 방향 수정 | 200 OK |
| Edge 삭제 | 204 No Content |

### WarehouseZone

| 테스트 | 결과 |
|---|---|
| Zone 생성 | 201 Created |
| 전체 조회 | 200 OK |
| 단건 조회 | 200 OK |
| 구역 정보 수정 | 200 OK |
| Zone 삭제 | 204 No Content |

컴파일 확인도 함께 진행했습니다.

```powershell
.\gradlew.bat compileJava
```

```text
BUILD SUCCESSFUL
```

---

## 13. Git 및 협업 과정

창고 관련 기능은 `feature/warehouse-domain` 브랜치에서 개발했습니다.

### 작업 흐름

```text
main 최신화
  ↓
feature/warehouse-domain 개발
  ↓
compileJava 및 Postman 테스트
  ↓
commit
  ↓
원격 브랜치 push
  ↓
Pull Request 생성
  ↓
main 병합
  ↓
feature 브랜치에 최신 main 반영
```

WarehouseEdge와 WarehouseZone 개발 후 각각 Pull Request를 생성하고 main 브랜치에 병합했습니다.

main의 새로운 변경사항은 다음 명령으로 작업 브랜치에 반영했습니다.

```powershell
git switch feature/warehouse-domain
git merge main
```

충돌 없이 Fast-forward 방식으로 반영된 것을 확인했습니다.

---

## 14. 개발 결과

첫 백엔드 개발 단계에서 다음 작업을 완료했습니다.

- Spring Boot 로컬 실행 확인
- PostgreSQL, Redis, Neo4j Docker 환경 실행
- Local 및 Docker Profile 분리
- Warehouse CRUD 구현
- WarehouseNode CRUD 구현
- WarehouseEdge CRUD 구현
- WarehouseZone CRUD 구현
- Node와 Edge 기반 창고 이동 구조 구현
- 단방향 및 양방향 Edge 모델링
- ZoneType 기반 창고 구역 모델링
- Postman CRUD 테스트 완료
- Gradle 컴파일 확인
- 기능별 Pull Request 및 main 병합

---

## 15. 배운 점

Docker Compose의 `.env` 파일과 로컬 Spring Boot 환경변수는 서로 다른 방식으로 적용된다는 점을 확인했습니다.

또한 API 요청에서 오류가 발생했을 때 DTO나 Service 코드만 확인하는 것이 아니라 다음 순서로 전체 요청 흐름을 확인해야 한다는 점을 경험했습니다.

```text
요청 URL
→ HTTP Method
→ Controller 매핑
→ DTO 바인딩
→ Service 전달 값
→ Repository 조회
→ 데이터베이스 상태
```

처음에는 `id must not be null` 오류를 DTO 문제로 판단했지만, 실제로는 Controller가 등록되지 않은 다른 실행 상태의 로그가 섞여 있었습니다.

서버를 완전히 재빌드하고 현재 요청에 대한 로그를 구분해서 확인하는 과정이 중요하다는 점을 배웠습니다.

또한 Node와 Edge를 분리하여 창고 이동 구조를 구현하면서, 일반적인 CRUD 데이터가 이후 그래프 탐색과 경로 최적화 입력 데이터로 사용될 수 있도록 설계하는 경험을 할 수 있었습니다.

---

## 16. 다음 개발 계획

다음 개발에서는 창고 레이아웃의 나머지 요소와 외부 최적화 서버 연동을 단계적으로 진행할 예정입니다.

- ChargingStation 도메인 및 CRUD 구현
- WarehouseNode와 WarehouseZone 관계 개선
- 창고 레이아웃 통합 조회 API 구현
- Neo4j 그래프 동기화 구조 설계
- FastAPI 경로 최적화 요청 구조 정의
