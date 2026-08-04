# 03. 창고 그래프 도메인 구현 및 문제 해결

> 이 문서는 프로젝트 초기 백엔드 개발 단계에서 창고 그래프 도메인을 구현하고, 개발 환경과 API 오류를 해결한 과정을 정리한 기록입니다.

## 1. 개발 범위

백엔드 3인 분업 중 창고 구조와 이동 그래프, 경로 최적화 서버 연동 영역을 담당했습니다.

초기 개발 단계에서는 향후 경로 최적화의 입력 데이터가 되는 창고 레이아웃을 관리하기 위해 다음 도메인을 우선 구현했습니다.

* Warehouse
* WarehouseNode
* WarehouseEdge
* WarehouseZone

이후 담당 범위는 다음 기능으로 확장했습니다.

* ChargingStation 관리
* 창고 레이아웃 통합 조회
* PostgreSQL–Neo4j 그래프 동기화
* FastAPI 초기 계획 및 재계획 연동
* 최적화 계획 검증과 실행 상태 반영

---

## 2. 개발 환경 구성

### 사용 기술

| 구분             | 기술                           |
| -------------- | ---------------------------- |
| Language       | Java 17                      |
| Framework      | Spring Boot, Spring Data JPA |
| Database       | PostgreSQL, Redis, Neo4j     |
| Infrastructure | Docker, Docker Compose       |
| Build          | Gradle                       |
| Test           | Postman                      |

PostgreSQL, Redis, Neo4j는 Docker Compose로 실행하고, 개발 중에는 Spring Boot를 IntelliJ 또는 터미널에서 직접 실행했습니다.

```text
로컬 개발 환경

Spring Boot
→ 로컬에서 실행

PostgreSQL·Redis·Neo4j
→ Docker 컨테이너로 실행
```

### Docker Compose 구성

| 서비스        | 역할                         |         포트 |
| ---------- | -------------------------- | ---------: |
| PostgreSQL | 창고·로봇·작업 등 영속 데이터 저장       |       5432 |
| Redis      | 로봇 위치와 시뮬레이션 Runtime 상태 관리 |       6379 |
| Neo4j      | Node와 Edge 기반 창고 그래프 관리    | 7474, 7687 |

---

## 3. Local·Docker Profile 분리

Spring Boot를 로컬에서 실행할 때와 Docker 컨테이너에서 실행할 때 데이터베이스 주소가 달라지는 문제를 방지하기 위해 Profile을 분리했습니다.

```text
application.yaml
application-local.yaml
application-docker.yaml
```

### Local Profile

Spring Boot는 내 컴퓨터에서 실행하고, 데이터 저장소만 Docker 컨테이너로 실행합니다.

```text
PostgreSQL → localhost:5432
Redis      → localhost:6379
Neo4j      → localhost:7687
```

### Docker Profile

Spring Boot까지 Docker Compose로 실행하는 경우에는 Docker Compose의 서비스 이름을 사용합니다.

```text
PostgreSQL → postgres:5432
Redis      → redis:6379
Neo4j      → neo4j:7687
```

컨테이너 내부에서 `localhost`는 현재 컨테이너 자신을 의미하므로, 다른 컨테이너에는 서비스 이름으로 접근해야 합니다.

환경별 설정을 분리해 로컬 개발 설정과 Docker 실행 설정이 섞이지 않도록 구성했습니다.

---

## 4. 창고 그래프 도메인 구현

창고의 이동 구조를 다음과 같이 표현했습니다.

```text
Warehouse
 ├─ WarehouseZone
 └─ WarehouseNode
      └─ WarehouseEdge
```

### 도메인별 역할

| 도메인             | 역할                   | 주요 데이터             |
| --------------- | -------------------- | ------------------ |
| `Warehouse`     | 창고 레이아웃의 최상위 기준      | 이름, 크기, 설명, 소유 사용자 |
| `WarehouseNode` | 로봇이 이동할 수 있는 그래프 정점  | 창고, 좌표, 구역 정보      |
| `WarehouseEdge` | 두 Node 사이의 이동 가능한 연결 | 출발·도착 Node, 거리, 방향 |
| `WarehouseZone` | 창고 내부 공간의 용도 구분      | 구역명, 유형, 좌표 범위     |

각 도메인에 생성·전체 조회·단건 조회·수정·삭제 API를 구현했습니다.

| 기능    | HTTP Method | 정상 응답            |
| ----- | ----------- | ---------------- |
| 생성    | `POST`      | `201 Created`    |
| 전체 조회 | `GET`       | `200 OK`         |
| 단건 조회 | `GET`       | `200 OK`         |
| 수정    | `PATCH`     | `200 OK`         |
| 삭제    | `DELETE`    | `204 No Content` |

### WarehouseNode

`WarehouseNode`는 단순한 화면 좌표가 아니라 로봇 경로 탐색에 사용되는 그래프의 정점입니다.

```text
WarehouseNode
→ 로봇의 이동 기준점
→ Edge 연결의 시작점과 도착점
→ 향후 경로 최적화 입력 데이터
```

Node를 별도 도메인으로 분리해 창고 화면 표현과 경로 탐색에서 동일한 기준점을 사용할 수 있도록 했습니다.

### WarehouseEdge

`WarehouseEdge`는 두 Node 사이의 이동 가능한 경로를 표현합니다.

```java
public enum DirectionType {
    BOTH,
    A_TO_B,
    B_TO_A
}
```

| 방향       | 의미                       |
| -------- | ------------------------ |
| `BOTH`   | 양방향 이동 가능                |
| `A_TO_B` | 시작 Node에서 도착 Node 방향만 이동 |
| `B_TO_A` | 도착 Node에서 시작 Node 방향만 이동 |

이동 방향을 Enum으로 관리해 일반 통로와 일방통행 통로를 동일한 구조로 표현했습니다.

### WarehouseZone

`WarehouseZone`은 창고 내부 공간을 용도에 따라 구분합니다.

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

구역은 최소·최대 X, Y 좌표를 저장해 직사각형 범위로 표현했습니다.

이를 통해 보관, 이동, 입고, 출고, 충전 및 제한 구역을 데이터로 구분할 수 있도록 했습니다.

---

## 5. 공통 구현 방식

### 계층 분리

각 기능은 다음 구조로 구현했습니다.

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

Controller가 Entity를 직접 요청과 응답에 사용하지 않도록 DTO를 분리했습니다.

이를 통해 다음 책임을 구분했습니다.

* Controller: HTTP 요청과 응답 처리
* Service: 비즈니스 규칙과 트랜잭션 처리
* Repository: 데이터 조회와 저장
* Entity: 도메인 상태와 상태 변경
* DTO: 외부 요청·응답 구조

### Entity 상태 변경 제한

Entity에 일괄적인 Setter를 제공하지 않고, 생성과 수정 목적이 드러나는 메서드를 정의했습니다.

```java
WarehouseEdge.create(
    fromNode,
    toNode,
    distance,
    directionType
);
```

```java
warehouseEdge.update(
    fromNode,
    toNode,
    distance,
    directionType
);
```

이를 통해 Entity의 상태가 임의로 변경되는 것을 줄이고, 변경 의도를 코드에 명확하게 표현했습니다.

---

## 6. 문제 해결 기록

### 6.1 Spring Boot가 `.env` 값을 읽지 못한 문제

#### 문제

Docker Compose에서는 `.env`에 작성한 PostgreSQL 설정이 적용됐지만, 로컬에서 Spring Boot를 실행하면 환경변수가 주입되지 않았습니다.

```text
${POSTGRES_USER}
${POSTGRES_PASSWORD}
```

설정값 대신 위 문자열이 그대로 전달되면서 PostgreSQL 연결에 실패했습니다.

#### 원인

Docker Compose는 프로젝트 루트의 `.env`를 자동으로 읽지만, 로컬에서 실행한 Spring Boot는 `.env` 파일을 운영체제 환경변수로 자동 등록하지 않습니다.

#### 해결

PowerShell에서 필요한 환경변수를 등록한 뒤 Spring Boot를 실행했습니다.

```powershell
$env:POSTGRES_DB="warehouse"
$env:POSTGRES_USER="warehouse"
$env:POSTGRES_PASSWORD="warehouse1234"

.\gradlew.bat bootRun
```

이를 통해 Docker Compose의 환경변수 처리와 로컬 Spring Boot의 환경변수 처리가 서로 다르다는 점을 확인했습니다.

---

### 6.2 Docker backend와 로컬 Spring Boot의 포트 충돌

#### 문제

`docker compose up -d`를 실행하면 backend 컨테이너도 함께 실행됐습니다.

이 상태에서 로컬 Spring Boot를 실행하면 두 서버가 모두 8080 포트를 사용해 실행에 실패했습니다.

#### 해결

로컬 개발 시에는 backend 컨테이너만 중지하고 데이터 저장소 컨테이너는 유지했습니다.

```powershell
docker compose stop backend
```

이후 Spring Boot를 로컬에서 실행했습니다.

```powershell
.\gradlew.bat bootRun
```

```text
로컬 개발

backend 컨테이너 → 중지
Spring Boot       → 로컬 실행
DB 컨테이너       → 계속 실행
```

전체 Docker 실행과 로컬 개발 환경을 구분해 포트 충돌을 해결했습니다.

---

### 6.3 Edge 생성 API가 Controller에 도달하지 않은 문제

#### 문제

`POST /api/warehouse-edges` 요청 중 다음 오류가 발생했습니다.

```text
The given id must not be null
```

처음에는 요청 DTO의 `fromNodeId` 또는 `toNodeId`가 바인딩되지 않은 것으로 판단했습니다.

다음 항목을 차례로 확인했습니다.

* 요청 URL과 HTTP Method
* JSON Body 형식
* DTO 필드명과 Getter·Setter
* Controller로 전달되는 요청값

확인 과정에서 실제 요청이 Edge Controller에 도달하지 않고 있다는 것을 발견했습니다.

추가로 다음 오류가 확인됐습니다.

```text
No static resource api/warehouse-edges
```

#### 원인

Spring MVC가 요청을 처리할 Controller를 찾지 못해 정적 리소스 요청으로 처리하고 있었습니다.

Windows 환경에서 패키지 폴더명을 변경한 뒤 이전 빌드 결과가 남아, 새로운 Controller가 정상적으로 반영되지 않은 것으로 판단했습니다.

#### 해결

기존 빌드 결과를 제거한 뒤 애플리케이션을 다시 실행했습니다.

```powershell
.\gradlew.bat clean bootRun
```

재빌드 후 Edge 생성 API가 정상적으로 등록됐고, 생성부터 수정·삭제까지 동작하는 것을 확인했습니다.

#### 배운 점

오류 메시지만 보고 DTO나 Repository 문제로 단정하지 않고, 요청이 어느 계층까지 전달됐는지 확인해야 한다는 점을 배웠습니다.

```text
요청 URL
→ HTTP Method
→ Controller 매핑
→ DTO 바인딩
→ Service
→ Repository
→ Database
```

또한 여러 실행 로그가 섞여 있을 수 있으므로 현재 실행 중인 서버와 최신 요청 로그를 구분해서 확인하는 과정이 중요했습니다.

---

## 7. 테스트 및 구현 결과

Postman을 사용해 Warehouse, Node, Edge, Zone의 CRUD 흐름을 확인했습니다.

### API 테스트

| 테스트    | 결과               |
| ------ | ---------------- |
| 데이터 생성 | `201 Created`    |
| 전체 조회  | `200 OK`         |
| 단건 조회  | `200 OK`         |
| 데이터 수정 | `200 OK`         |
| 데이터 삭제 | `204 No Content` |

### 컴파일 확인

```powershell
.\gradlew.bat compileJava
```

```text
BUILD SUCCESSFUL
```

### 초기 개발 결과

* Spring Boot 로컬 실행 환경 구성
* PostgreSQL·Redis·Neo4j Docker 환경 구성
* Local·Docker Profile 분리
* Warehouse·Node·Edge·Zone API 구현
* Node와 Edge 기반 창고 이동 구조 구현
* 단방향·양방향 Edge 모델링
* ZoneType 기반 창고 영역 모델링
* Postman CRUD 테스트
* Gradle 컴파일 검증

---

## 8. 회고

초기 백엔드 개발을 통해 CRUD 기능을 구현하는 것에서 끝나지 않고, 이후 경로 탐색과 최적화에 활용할 수 있는 형태로 창고 데이터를 모델링했습니다.

특히 Node와 Edge를 분리하면서 다음 관계를 구체적으로 이해할 수 있었습니다.

```text
일반 CRUD 데이터
→ 창고 이동 그래프
→ 경로 탐색 입력
→ 다중 로봇 최적화 데이터
```

또한 개발 환경 문제와 API 오류를 해결하면서 설정 파일, 실행 프로세스, Controller 매핑과 빌드 결과까지 전체 흐름을 확인하는 습관이 중요하다는 점을 경험했습니다.

이후 개발에서는 이 창고 그래프 도메인을 기반으로 레이아웃 통합 조회, Neo4j 동기화, FastAPI 최적화 연동과 동적 재계획 기능을 확장했습니다.
