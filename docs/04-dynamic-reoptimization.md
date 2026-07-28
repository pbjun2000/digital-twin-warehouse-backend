# 동적 재최적화 및 실시간 알림 개발 기록

## 1. 개발 배경

창고 시뮬레이션 실행 중에는 로봇 장애, 배터리 부족, 경로 차단, 충돌 위험과 같은 예외 상황이 발생할 수 있다.

초기 최적화 결과만 사용하는 경우, 이미 배정된 작업과 경로를 예외 상황에 맞게 변경하기 어렵다. 이를 해결하기 위해 현재 로봇 상태와 남은 작업을 기준으로 작업 배정과 이동 경로를 다시 계산하는 동적 재최적화 기능을 구현했다.

이번 개발의 주요 목표는 다음과 같다.

- 실행 중인 시뮬레이션의 현재 상태 수집
- FastAPI 재최적화 서버 연동
- 작업 담당 로봇 재배정
- Redis 로봇 상태 갱신
- 재최적화 결과 및 변경 이력 저장
- 시뮬레이션 실행별 재최적화 이력 조회
- WebSocket을 통한 재최적화 완료 알림

---

## 2. 전체 처리 흐름

재최적화 처리 흐름은 다음과 같이 구성했다.

```text
예외 이벤트 발생
→ 실행 중인 SimulationRun 조회
→ Redis에서 현재 로봇 상태 조회
→ DB에서 미완료 작업 조회
→ FastAPI 재최적화 요청
→ 작업 담당 로봇 변경
→ Redis 로봇 상태 갱신
→ 재최적화 결과 및 경로 저장
→ 작업 재배정 이력 저장
→ 트랜잭션 커밋
→ WebSocket 완료 이벤트 전송
```

---

## 3. 재최적화 API 구현

재최적화 요청 API는 시뮬레이션 실행 ID를 기준으로 동작한다.

```http
POST /api/optimizations/simulation-runs/{simulationRunId}/reoptimize
```

요청 예시는 다음과 같다.

```json
{
  "reason": "ROBOT_FAILURE",
  "triggerRobotId": 2,
  "blockedEdgeIds": [],
  "description": "로봇 장애로 인한 재최적화 요청"
}
```

재최적화 사유는 다음과 같이 구분했다.

- `ROBOT_FAILURE`: 로봇 장애
- `LOW_BATTERY`: 배터리 부족
- `OBSTACLE_DETECTED`: 장애물 또는 경로 차단

재최적화 요청 시 해당 시뮬레이션 실행이 실제로 진행 중인지 검증했다.

```java
if (simulationRun.getStatus() != SimulationRunStatus.RUNNING) {
    throw new BusinessException(
            ErrorCode.SIMULATION_RUN_NOT_RUNNING
    );
}
```

실행 중인 시뮬레이션이 아닌 경우 재최적화를 수행하지 않도록 제한했다.

---

## 4. 현재 로봇 상태 수집

실시간 로봇 상태는 Redis 기반의 `SimulationRunStateStore`에서 조회했다.

```java
List<RobotState> robotStates =
        simulationRunStateStore.findAll(simulationRunId);
```

FastAPI에는 다음 정보를 전달한다.

- 로봇 ID
- 현재 노드 ID
- 배터리 잔량
- 로봇 상태

```java
new ReoptimizationOptimizationRequest.RobotStateInput(
        state.robotId(),
        state.currentNodeId(),
        state.batteryLevel() == null
                ? null
                : state.batteryLevel().doubleValue(),
        state.status().name()
)
```

DB의 정적인 로봇 정보가 아니라 Redis에 저장된 실행 중 상태를 사용해 현재 상황을 반영하도록 했다.

---

## 5. 남은 작업 조회

재최적화 대상에는 아직 완료되지 않은 작업만 포함했다.

```java
private static final List<TaskStatus> ACTIVE_TASK_STATUSES = List.of(
        TaskStatus.PENDING,
        TaskStatus.ASSIGNED,
        TaskStatus.IN_PROGRESS
);
```

작업 요청 시각을 기준으로 정렬해 조회했다.

```java
taskRepository
        .findAllBySimulationRun_IdAndStatusInOrderByRequestedAtAsc(
                simulationRunId,
                ACTIVE_TASK_STATUSES
        );
```

FastAPI에 전달하는 주요 작업 정보는 다음과 같다.

- 작업 ID
- 기존 담당 로봇 ID
- 시작 노드
- 도착 노드
- 작업 유형
- 현재 작업 상태

신규 작업뿐 아니라 이미 배정되었거나 진행 중인 작업도 재최적화 대상에 포함되도록 구성했다.

---

## 6. FastAPI 재최적화 서버 연동

수집한 시뮬레이션 정보, 로봇 상태, 남은 작업을 재최적화 요청 DTO로 변환한 뒤 FastAPI 서버에 전달했다.

```java
ReoptimizationOptimizationRequest fastApiRequest =
        new ReoptimizationOptimizationRequest(
                simulationRunId,
                simulationRun.getWarehouse().getId(),
                request.reason(),
                request.triggerRobotId(),
                request.blockedEdgeIds(),
                request.description(),
                robots,
                tasks
        );
```

재최적화 요청은 `OptimizationClient`가 담당한다.

```java
ReoptimizationResponse response =
        optimizationClient.reoptimize(fastApiRequest);
```

개발 초기에는 실제 AI 최적화 서버가 완성되지 않은 상태였기 때문에, Mock FastAPI 서버를 구성해 요청과 응답 구조를 먼저 검증했다.

Mock 서버는 다음 형태의 결과를 반환하도록 구성했다.

```json
{
  "requestId": "278afe03-9853-4366-94db-0574d31d2747",
  "status": "SUCCESS",
  "assignments": [
    {
      "taskId": 2,
      "robotId": 3
    }
  ],
  "routes": [
    {
      "robotId": 3,
      "nodePath": [1, 2],
      "totalDistance": 10.0,
      "estimatedTime": 5.0
    }
  ]
}
```

이를 통해 실제 최적화 알고리즘이 완성되기 전에도 백엔드의 재배정, 결과 저장, WebSocket 전송 흐름을 먼저 구현할 수 있었다.

---

## 7. 작업 담당 로봇 재배정

FastAPI가 반환한 작업 배정 결과를 기준으로 각 작업의 담당 로봇을 변경했다.

작업 상태에 따라 배정 메서드를 구분했다.

```java
if (task.getStatus() == TaskStatus.PENDING) {
    task.assignRobot(robot);
} else if (task.getStatus() == TaskStatus.ASSIGNED
        || task.getStatus() == TaskStatus.IN_PROGRESS) {
    task.reassignRobot(robot);
}
```

재배정 전에 다음 조건을 검증했다.

- 작업이 요청한 `SimulationRun`에 속하는지
- 로봇이 해당 시뮬레이션의 창고에 속하는지
- 작업과 로봇이 실제로 존재하는지

다른 창고의 로봇이나 다른 시뮬레이션의 작업이 잘못 연결되는 것을 방지했다.

---

## 8. 작업 변경 이력 저장

재최적화 과정에서 작업 담당 로봇이 실제로 변경된 경우에만 이력을 저장했다.

```java
if (previousRobotId == null
        || !previousRobotId.equals(robot.getId())) {

    TaskAssignmentResult assignmentResult =
            TaskAssignmentResult.create(
                    task.getId(),
                    previousRobotId,
                    robot.getId()
            );

    assignmentResults.add(assignmentResult);
}
```

예를 들어 작업이 기존에도 Robot 3에 배정되어 있고 재최적화 결과도 Robot 3인 경우에는 변경 이력을 생성하지 않는다.

반대로 Robot 2에서 Robot 3으로 변경된 경우 다음 정보가 저장된다.

- 작업 ID
- 기존 로봇 ID
- 변경된 로봇 ID
- 연결된 최적화 결과 ID

이를 통해 단순 최적화 결과뿐 아니라 실제 담당 로봇 변경 내역까지 추적할 수 있도록 했다.

---

## 9. Redis 로봇 상태 갱신

재배정된 로봇은 Redis에서 `ASSIGNED` 상태로 변경하고 현재 작업 ID를 연결했다.

```java
new RobotState(
        state.robotId(),
        state.warehouseId(),
        state.currentNodeId(),
        state.batteryLevel(),
        RobotStatus.ASSIGNED,
        task.getId(),
        LocalDateTime.now()
)
```

로봇 장애가 재최적화 원인인 경우, 장애가 발생한 로봇은 `ERROR` 상태로 변경하고 현재 작업을 해제했다.

```java
new RobotState(
        state.robotId(),
        state.warehouseId(),
        state.currentNodeId(),
        state.batteryLevel(),
        RobotStatus.ERROR,
        null,
        LocalDateTime.now()
)
```

이를 통해 DB의 작업 배정 결과와 Redis의 실시간 로봇 상태가 함께 갱신되도록 했다.

---

## 10. 재최적화 결과 및 경로 저장

재최적화 결과는 `OptimizationResult` 엔티티에 저장했다.

저장하는 주요 정보는 다음과 같다.

- 최적화 유형
- FastAPI 요청 ID
- 창고 ID
- 시뮬레이션 실행 ID
- 처리 상태
- 재최적화 사유
- 장애 발생 로봇 ID
- 요청 설명

초기 최적화와 재최적화 결과를 구분하기 위해 최적화 유형을 분리했다.

```text
INITIAL
REOPTIMIZATION
```

로봇별 이동 경로는 `RobotRouteResult`에 저장했다.

```java
RobotRouteResult routeResult =
        RobotRouteResult.create(
                route.robotId(),
                convertNodePathToJson(route.nodePath()),
                route.totalDistance(),
                route.estimatedTime()
        );
```

노드 경로 목록은 JSON 문자열로 변환해 저장했다.

```java
private String convertNodePathToJson(List<Long> nodePath) {
    try {
        return objectMapper.writeValueAsString(nodePath);
    } catch (JsonProcessingException e) {
        throw new IllegalStateException(
                "재최적화 경로 노드 목록을 JSON으로 변환하지 못했습니다.",
                e
        );
    }
}
```

---

## 11. 재최적화 이력 조회 API

시뮬레이션 실행별 재최적화 결과를 조회할 수 있도록 이력 조회 API를 구현했다.

```http
GET /api/optimizations/simulation-runs/{simulationRunId}/reoptimization-histories
```

조회 결과에는 다음 정보가 포함된다.

- 최적화 결과 ID
- 재최적화 요청 ID
- 처리 상태
- 재최적화 사유
- 장애 발생 로봇 ID
- 설명
- 로봇별 경로
- 작업 재배정 내역
- 생성 시각

최신 재최적화 결과부터 확인할 수 있도록 최신순으로 반환했다.

이를 통해 프론트엔드에서 특정 시뮬레이션의 재최적화 발생 이력과 변경 내용을 조회할 수 있게 했다.

---

## 12. WebSocket 완료 이벤트 구현

재최적화 처리가 완료된 시점을 프론트엔드에 실시간으로 전달하기 위해 STOMP 기반 WebSocket 이벤트를 추가했다.

WebSocket 연결 Endpoint는 다음과 같다.

```text
/ws
```

시뮬레이션 실행별 구독 Topic은 다음과 같다.

```text
/topic/simulation-runs/{simulationRunId}/reoptimization
```

재최적화 완료 이벤트 예시는 다음과 같다.

```json
{
  "eventType": "REOPTIMIZATION_COMPLETED",
  "simulationRunId": 2,
  "optimizationResultId": 6,
  "requestId": "278afe03-9853-4366-94db-0574d31d2747",
  "status": "SUCCESS",
  "reason": "ROBOT_FAILURE",
  "triggerRobotId": 2,
  "changedTaskIds": [],
  "affectedRobotIds": [2, 3],
  "occurredAt": "2026-07-24T07:20:08.52312152"
}
```

이벤트에는 다음 정보가 포함된다.

- 이벤트 유형
- 시뮬레이션 실행 ID
- 저장된 최적화 결과 ID
- FastAPI 요청 ID
- 처리 상태
- 재최적화 사유
- 장애 발생 로봇 ID
- 변경된 작업 ID 목록
- 영향을 받은 로봇 ID 목록
- 이벤트 발생 시각

---

## 13. 트랜잭션 커밋 이후 메시지 발행

재최적화 결과가 DB에 저장되기 전에 WebSocket 메시지를 전송하면, 프론트엔드가 이벤트를 받은 직후 최신 데이터를 조회했을 때 아직 결과가 조회되지 않는 문제가 발생할 수 있다.

또한 DB 저장 중 오류가 발생해 트랜잭션이 롤백되었는데도 완료 이벤트가 전달될 수 있다.

이를 방지하기 위해 트랜잭션 커밋 이후에만 WebSocket 메시지를 전송하도록 구성했다.

```java
TransactionSynchronizationManager.registerSynchronization(
        new TransactionSynchronization() {
            @Override
            public void afterCommit() {
                messagingTemplate.convertAndSend(
                        topic,
                        event
                );
            }
        }
);
```

이 구조를 통해 다음을 보장했다.

- DB 저장 성공 후에만 완료 이벤트 발행
- 트랜잭션 롤백 시 이벤트 미발행
- 프론트엔드가 이벤트 수신 후 최신 결과를 안정적으로 재조회 가능

---

## 14. WebSocket 연동 테스트

SockJS와 STOMP를 사용하는 테스트 페이지를 만들어 실제 구독 여부를 검증했다.

구독 주소는 다음과 같다.

```text
/topic/simulation-runs/2/reoptimization
```

재최적화 REST API를 호출한 뒤 다음 이벤트가 수신되는 것을 확인했다.

```text
연결 성공
구독 주소: /topic/simulation-runs/2/reoptimization
재최적화 완료 이벤트 수신
```

REST 응답과 WebSocket 이벤트의 `requestId`가 동일한 것도 확인했다.

```text
278afe03-9853-4366-94db-0574d31d2747
```

이를 통해 다음 전체 흐름이 정상 동작하는 것을 검증했다.

```text
재최적화 REST 요청
→ FastAPI 응답 수신
→ 작업 재배정
→ Redis 상태 갱신
→ DB 결과 저장
→ 트랜잭션 커밋
→ WebSocket 이벤트 발행
→ STOMP 구독 클라이언트 수신
```

---

## 15. 구현 결과

이번 개발을 통해 실행 중인 창고 시뮬레이션에서 예외 상황이 발생했을 때 현재 상태를 기준으로 작업과 경로를 다시 계산할 수 있는 구조를 구현했다.

주요 구현 결과는 다음과 같다.

- Redis 기반 실시간 로봇 상태 수집
- 미완료 작업 기반 재최적화 요청
- FastAPI 재최적화 서버 연동
- 작업 담당 로봇 재배정
- 장애 로봇 및 재배정 로봇 상태 갱신
- 재최적화 결과와 로봇별 경로 저장
- 작업 담당 변경 이력 저장
- 시뮬레이션 실행별 재최적화 이력 조회
- 트랜잭션 커밋 이후 WebSocket 완료 이벤트 발행
- REST 및 STOMP 통합 테스트 완료

---

## 16. 문제 해결 및 배운 점

### 16.1 동일 로봇 재배정 이력 문제

재최적화 응답에서 기존 담당 로봇과 동일한 로봇이 반환될 수 있었다.

처음에는 모든 결과를 변경 이력으로 저장했지만, 실제 변경이 없는 경우까지 이력에 포함되는 문제가 있었다.

기존 로봇 ID와 새 로봇 ID를 비교해 실제 변경이 발생한 경우에만 `TaskAssignmentResult`를 저장하도록 수정했다.

### 16.2 DB 저장과 WebSocket 전송 시점 문제

트랜잭션 내부에서 즉시 WebSocket 메시지를 전송하면, DB 커밋보다 메시지가 먼저 전달될 가능성이 있었다.

`TransactionSynchronizationManager`의 `afterCommit()`을 사용해 DB 저장이 완료된 이후에만 메시지가 전송되도록 개선했다.

### 16.3 AI 서버 미완성 상태의 연동 문제

실제 최적화 서버 개발이 완료될 때까지 기다리면 백엔드 개발과 테스트가 지연될 수 있었다.

Mock FastAPI 서버를 구성해 실제 API와 동일한 요청 및 응답 형식을 먼저 정의하고, 백엔드 로직을 독립적으로 구현했다.

이를 통해 AI 서버가 완성되면 Mock 서버 주소만 실제 서버로 변경해 연동할 수 있는 구조를 만들었다.

---

## 17. 향후 개발 계획

- 실제 AI 최적화 서버와 요청 및 응답 DTO 최종 통합
- 경로 차단 Edge 정보를 활용한 재최적화 검증
- 배터리 부족 상황의 충전소 이동 로직 연동
- 프론트엔드 시뮬레이션 화면과 WebSocket 연결
- 재최적화 완료 이벤트 수신 후 로봇, 작업, 경로 데이터 재조회
- 초기 최적화와 재최적화 결과 비교 화면 구현
- Docker Compose 기반 전체 서비스 통합
- AWS 환경 배포 및 외부 WebSocket 연결 검증
