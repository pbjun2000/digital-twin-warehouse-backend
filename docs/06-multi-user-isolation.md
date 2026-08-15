# 06. Multi-user Isolation

> LARO에서는 여러 사용자가 동시에 Simulation을 실행하더라도  
> **Inventory · Robot · Scenario · Warehouse Graph 상태가 서로 영향을 주지 않도록 USER/GUEST별 독립적인 Digital Twin 실행 환경**을 구성했습니다.

---

## 1. 문제 상황

초기 구조에서는 하나의 Shared Warehouse를 여러 사용자가 공통으로 사용할 수 있었습니다.

```text
                 Shared Warehouse
                       │
          ┌────────────┼────────────┐
          │            │            │
        User A       User B       Guest
          │            │            │
          └──── 동일 Warehouse ─────┘
```

창고 구조를 단순 조회하는 경우에는 문제가 없지만,
Simulation 실행 중에는 Warehouse에 연결된 상태가 계속 변경됩니다.

예를 들어,

```text
Inventory 수량 변경
Robot 위치 변경
Robot 상태 변경
Task 진행 상태 변경
Simulation 실행 상태 변경
```

등이 발생합니다.

따라서 여러 사용자가 하나의 Warehouse를 실행하면

```text
User A가 Inventory 변경
        ↓
User B Simulation에도 동일한 Inventory 반영

User B가 Robot 상태 변경
        ↓
Guest 실행 환경에도 영향
```

과 같이 **사용자별 Simulation 상태가 서로 충돌할 수 있는 문제**가 있었습니다.

---

## 2. Shared Warehouse를 Template으로 변경

문제를 해결하기 위해 Shared Warehouse를
직접 실행하는 Warehouse가 아니라 **Template**으로 사용하도록 구조를 변경했습니다.

```text
BEFORE

Shared Warehouse
     │
     ├── User A 실행
     ├── User B 실행
     └── Guest 실행


AFTER

Shared Warehouse Template
        │
        ├── Personal Warehouse A → User A
        │
        ├── Personal Warehouse B → User B
        │
        └── Personal Warehouse C → Guest
```

실제 Simulation은 항상 사용자에게 할당된
Personal Warehouse에서 실행합니다.

---

## 3. Warehouse 소유 구조

Warehouse에는 공유 여부와 사용자 소유 관계를 함께 관리합니다.

개념적으로 다음과 같은 구조입니다.

```text
Warehouse
├─ is_shared
├─ user_id
├─ guest_session_id
└─ source_template_id
```

### Shared Warehouse

```text
is_shared = true
```

모든 사용자가 조회할 수 있는 Template입니다.

Simulation 실행 상태를 직접 저장하는 용도로 사용하지 않습니다.

### USER Personal Warehouse

```text
is_shared = false
user_id = 현재 로그인 사용자
source_template_id = 원본 Shared Warehouse
```

### GUEST Personal Warehouse

```text
is_shared = false
guest_session_id = Guest 식별자
source_template_id = 원본 Shared Warehouse
```

USER와 GUEST를 동일한 방식으로 처리하지 않고,
각 인증 방식에 맞는 소유 식별자를 사용했습니다.

---

## 4. Personal Warehouse 생성 API

Shared Warehouse에서 개인 실행 환경을 생성하기 위해
USER와 GUEST API를 분리했습니다.

### USER

```http
POST /api/warehouses/{templateWarehouseId}/personal-copy
```

현재 로그인된 사용자를 기준으로
Personal Warehouse를 생성합니다.

### GUEST

```http
POST /api/warehouses/{templateWarehouseId}/guest-personal-copy
```

Guest JWT의 식별 정보를 기준으로
Guest 전용 Personal Warehouse를 생성합니다.

Frontend에서는 현재 로그인 형태를 확인한 뒤
각 API를 선택합니다.

```text
Shared Warehouse 선택
        ↓
현재 사용자 확인
    ↙               ↘
 USER              GUEST
  ↓                  ↓
personal-copy   guest-personal-copy
    ↘               ↙
      Personal Warehouse
              ↓
          Simulation
```

---

## 5. Guest 식별

Guest는 일반 회원처럼 DB User ID를 갖지 않습니다.

따라서 Guest 인증 과정에서 발급된 JWT의 Subject를
Guest Session 식별자로 사용합니다.

```text
Guest Login
     ↓
JWT 발급
     ↓
JWT Subject
     ↓
guest_session_id
     ↓
Personal Warehouse Ownership
```

Backend에서는 `AuthenticatedRequesterResolver`를 통해
현재 요청자가 USER인지 GUEST인지 구분하고
각 요청자의 식별 정보를 추출합니다.

```text
Authenticated Request
        ↓
AuthenticatedRequesterResolver
        ↓
 ┌──────────────┬────────────────┐
 │ USER         │ GUEST          │
 │ userId       │ guestSessionId │
 └──────────────┴────────────────┘
```

이를 통해 Guest 역시 회원과 동일하게
자신에게 할당된 Warehouse만 실행할 수 있도록 구성했습니다.

---

## 6. Deep Clone

Personal Warehouse를 만들 때
`Warehouse` Entity 한 개만 복사해서는 실행 환경을 분리할 수 없습니다.

Simulation에 영향을 주는 Warehouse 종속 데이터까지 함께 복제합니다.

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

즉,

```text
Template
WH-001

Inventory
Robot
Node / Edge
Scenario
```

를 그대로 공유하는 것이 아니라,

```text
User A
WH-002
├─ Inventory A
├─ Robot A
├─ Node / Edge A
└─ Scenario A

User B
WH-003
├─ Inventory B
├─ Robot B
├─ Node / Edge B
└─ Scenario B
```

처럼 실행 데이터까지 분리합니다.

---

## 7. Clone 과정의 ID 재매핑

Deep Clone에서는 기존 Entity의 ID를 그대로 사용할 수 없습니다.

예를 들어 Template Warehouse의 Node를 새 Warehouse로 복제하면
새로운 DB ID가 생성됩니다.

```text
Template

Node #10
Node #11
Node #12
```

복제 후:

```text
Personal Warehouse

Node #201
Node #202
Node #203
```

따라서 Edge나 Storage 등이 기존 Template Node를 계속 참조해서는 안 됩니다.

```text
Old Node ID
     ↓
New Node ID Mapping
     ↓
Edge / Storage / Charging Station
참조 관계 재구성
```

`WarehouseTemplateCloneService`에서는
복제 과정에서 새로운 Entity 관계를 다시 연결하여
Personal Warehouse 내부에서 독립적인 참조 관계를 갖도록 구성했습니다.

---

## 8. 중복 Personal Warehouse 방지

사용자가 동일한 Shared Template을 여러 번 선택했다고 해서
매번 새로운 Warehouse를 생성하면 불필요한 복제 데이터가 계속 증가할 수 있습니다.

따라서 다음 관계를 기준으로
이미 생성된 Personal Warehouse가 있는지 확인합니다.

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

```text
Personal Copy 요청
        ↓
기존 Copy 존재?
    ↙             ↘
  YES              NO
   ↓                ↓
기존 Warehouse     Deep Clone
반환               수행
```

이를 통해 동일 사용자가 같은 Template을 반복 선택하더라도
기존 실행 환경을 재사용할 수 있도록 구성했습니다.

---

## 9. Shared Template 직접 실행 차단

Personal Warehouse 구조를 만들더라도
클라이언트가 Shared Warehouse ID를 직접 API에 전달할 가능성이 있습니다.

예를 들어,

```http
POST /api/simulation-runs
```

요청에 Shared Warehouse ID를 직접 넣을 수 있습니다.

따라서 Simulation 실행 경계에서 다시 Warehouse를 검증합니다.

```text
Simulation 요청
       ↓
Warehouse 조회
       ↓
Shared Warehouse?
   ↙           ↘
 YES            NO
  ↓              ↓
실행 차단       Ownership 검증
```

Shared Template은 Simulation 실행 대상으로 사용할 수 없습니다.

즉 Frontend에서만 Shared Warehouse 사용을 막는 것이 아니라
**Backend 실행 경계에서도 직접 검증**합니다.

---

## 10. Warehouse Ownership Validation

Personal Warehouse 역시
ID를 알고 있다는 이유만으로 접근할 수 있어서는 안 됩니다.

따라서 Simulation 실행 시 요청자의 소유권을 검증합니다.

### USER

```text
Authenticated User
       ↓
Warehouse.user_id
       ↓
현재 User ID와 동일?
    ↙             ↘
  YES              NO
   ↓                ↓
실행 허용          403
```

### GUEST

```text
Guest JWT
      ↓
guest_session_id
      ↓
Warehouse.guest_session_id
      ↓
현재 Guest와 동일?
    ↙              ↘
  YES               NO
   ↓                 ↓
실행 허용           403
```

이를 통해 다음과 같은 접근을 차단합니다.

```text
User A → User B Warehouse
Guest A → Guest B Warehouse
USER → 다른 사용자의 Warehouse
GUEST → USER Warehouse
```

즉 인증 여부뿐 아니라
**실제 Resource Ownership을 실행 시점에 검증**합니다.

---

## 11. IDOR 방지

Warehouse ID는 클라이언트가 요청에 직접 전달하는 값이기 때문에
단순히 인증된 사용자라는 이유만으로 신뢰할 수 없습니다.

예를 들어 User A가 자신의 요청에서

```text
warehouseId = User B의 Warehouse ID
```

로 변경해서 직접 요청할 수 있습니다.

따라서 Backend에서는 요청으로 전달된 ID 자체를 신뢰하지 않고
Warehouse의 실제 소유 정보를 확인합니다.

```text
Client Request
warehouseId = 123
       ↓
JWT Authentication
       ↓
Warehouse #123 조회
       ↓
Ownership Validation
       ↓
허용 / 차단
```

이 구조를 통해 Broken Access Control / IDOR 형태의
다른 사용자 Resource 접근을 방지합니다.

---

## 12. Personal Warehouse와 Neo4j Graph

Warehouse 데이터만 PostgreSQL에서 분리하고
Neo4j Graph를 Template과 공유하면
AI Planning 단계에서는 여전히 사용자별 Digital Twin이 분리되지 않습니다.

따라서 Personal Warehouse Deep Clone 이후
새로운 Warehouse ID를 기준으로 Graph Sync Event를 발생시킵니다.

```text
Shared Template
       ↓
Personal Warehouse Deep Clone
       ↓
PostgreSQL COMMIT
       ↓
WarehouseGraphChangedEvent
       │
       └── personalWarehouseId
                   ↓
             AFTER_COMMIT
                   ↓
             GraphSyncService
                   ↓
       Personal Warehouse Graph
```

결과적으로:

```text
User A
PostgreSQL WH-002
       ↕
Neo4j WH-002 Graph

User B
PostgreSQL WH-003
       ↕
Neo4j WH-003 Graph
```

처럼 Warehouse Scope별로
Digital Twin Graph까지 분리됩니다.

> Graph Sync 구조는  
> [04. Digital Twin Graph](./04-digital-twin-graph.md)에서 상세히 설명합니다.

---

## 13. Scenario 재연결

Shared Warehouse의 Scenario 역시
Personal Warehouse 생성 과정에서 복제됩니다.

따라서 Frontend가 기존 Template의 Scenario ID를
그대로 Personal Warehouse 실행에 사용해서는 안 됩니다.

```text
Shared Warehouse
Scenario #10
      ↓
Deep Clone
      ↓
Personal Warehouse
Scenario #25
```

Frontend에서는 Personal Warehouse 생성 이후
해당 Warehouse의 Scenario를 다시 조회하여
복제된 Scenario와 연결합니다.

```text
Template Scenario
      ↓
Personal Warehouse 생성
      ↓
Personal Warehouse Scenario 조회
      ↓
Scenario Code / Name 기준 매칭
      ↓
Personal Scenario ID
      ↓
SimulationRun 생성
```

이를 통해 Warehouse만 Personal Copy로 바뀌고
Scenario는 Template Resource를 참조하는 상태를 방지했습니다.

---

## 14. Frontend 연동 흐름

최종 사용자 흐름은 다음과 같습니다.

### USER

```text
로그인
  ↓
Shared Warehouse 선택
  ↓
Personal Copy 요청
  ↓
USER Personal Warehouse
  ↓
Personal Scenario 조회
  ↓
SimulationRun 생성
  ↓
Simulation 실행
```

### GUEST

```text
Guest Login
  ↓
Shared Warehouse 선택
  ↓
Guest Personal Copy 요청
  ↓
GUEST Personal Warehouse
  ↓
Personal Scenario 조회
  ↓
SimulationRun 생성
  ↓
Simulation 실행
```

USER는 자신의 Warehouse를 직접 생성·관리할 수도 있지만,
Guest는 Shared Template을 기반으로 생성된
Guest 실행 환경을 이용하도록 구성했습니다.

---

## 15. Data Isolation 범위

최종적으로 다음 영역을 사용자별로 분리합니다.

| 영역 | 분리 기준 |
|---|---|
| Warehouse | USER / GUEST Personal Copy |
| Inventory | Personal Warehouse |
| Robot | Personal Warehouse |
| Zone | Personal Warehouse |
| Node / Edge | Personal Warehouse |
| Charging Station | Personal Warehouse |
| Storage Location | Personal Warehouse |
| Scenario | Personal Warehouse |
| Neo4j Graph | Warehouse Scope |
| SimulationRun | Personal Warehouse 기준 실행 |

이를 통해 동일한 Shared Template에서 시작하더라도
실제 실행 시점에는 각 사용자가 별도의 Digital Twin을 갖게 됩니다.

---

## 16. 주요 Component

| Component | 역할 |
|---|---|
| `WarehouseTemplateCloneService` | Shared Template을 Personal Warehouse로 Deep Clone |
| `AuthenticatedRequesterResolver` | 현재 요청자의 USER / GUEST 식별 |
| `SimulationRunService` | 실행 시 Shared 여부 및 Warehouse Ownership 검증 |
| `WarehouseGraphChangedEvent` | Personal Warehouse Graph Sync Trigger |
| `GraphSyncService` | 새 Warehouse Scope의 Neo4j Graph 동기화 |

---

## 17. 구조 변경 전후

### Before

```text
               Shared Warehouse
                     │
        ┌────────────┼────────────┐
        │            │            │
      User A       User B       Guest
        │            │            │
        └────────────┼────────────┘
                     ↓
         Shared Inventory / Robot
                     ↓
             상태 충돌 가능
```

### After

```text
              Shared Template
                     │
      ┌──────────────┼──────────────┐
      │              │              │
      ▼              ▼              ▼
Personal A      Personal B      Personal C
  User A           User B          Guest
      │              │              │
 Inventory A     Inventory B     Inventory C
 Robot A         Robot B         Robot C
 Graph A         Graph B         Graph C
      │              │              │
      ▼              ▼              ▼
Simulation A    Simulation B    Simulation C
```

Shared Warehouse는 초기 데이터의 기준으로만 사용하고
실제 실행 상태는 완전히 분리했습니다.

---

## 18. 설계 결과

Personal Warehouse 구조를 통해 다음 문제를 해결했습니다.

- 사용자 간 Inventory 상태 충돌 방지
- 사용자 간 Robot 실행 상태 충돌 방지
- USER / GUEST 실행 환경 분리
- Shared Template 원본 데이터 보호
- 다른 사용자의 Warehouse ID 직접 접근 차단
- Personal Warehouse별 Neo4j Graph 분리
- Personal Scenario 기반 Simulation 실행

핵심 구조는 다음과 같습니다.

```text
Authentication
      ↓
Requester Identification
      ↓
Shared Template
      ↓
Personal Warehouse Deep Clone
      ↓
Ownership Validation
      ↓
Personal Graph Sync
      ↓
Independent Simulation
```

> 단순히 USER와 GUEST의 로그인 방식을 분리하는 것이 아니라,  
> **실제로 변경되는 Warehouse Resource와 실행 상태까지 사용자 단위로 분리하는 것을 목표로 설계했습니다.**

---

## 향후 개선 가능 영역

현재 Guest Personal Warehouse는
Guest Session을 기준으로 독립된 실행 환경을 제공합니다.

장기 운영 환경에서는 사용이 종료된 Guest Warehouse가 계속 남지 않도록

```text
Guest Session 만료
      ↓
TTL / Last Access 확인
      ↓
Personal Warehouse 정리
      ↓
관련 Neo4j Graph 정리
```

와 같은 자동 정리 정책을 추가할 수 있습니다.

또한 동시 Personal Copy 요청이 많은 환경에서는
중복 생성 방지와 Clone 처리의 동시성 제어를 더욱 강화할 수 있습니다.

---

[← 05. AI Integration](./05-ai-integration.md) · [README](../README.md)
