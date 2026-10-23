---
title: "폴링을 SSE로 바꾼 이유 - WebSocket을 쓰지 않은 선택"
date: 2023-09-15
categories: [backend]
tags: [sse, server-sent-events, websocket, fcm, spring-boot, realtime]
---

기사님들이 "새 수주가 늦게 뜬다"고 했다. 로그를 보니 늦게 뜨는 게 아니라, 앱이 늦게 물어보고 있었다.

## 배경

고소작업차 실시간 배차 서비스였다. 발주가 들어오면 조건에 맞는 기사들에게 알리고, 먼저 수락한 한 명에게 배차가 확정되는 구조다. 기사 약 2,000명, 일 배차 30~40건 규모였다.

"실시간"이라고 이름을 붙여놨지만, 초기 구현은 실시간이 아니었다. 기사 앱이 일정 주기로 서버에 새 수주가 있는지 물어보는 폴링 방식이었다.

## 문제: 폴링의 두 가지 비용

**첫째, 지연이 주기만큼 생긴다.** 폴링 주기를 10초로 두면 발주가 올라온 직후에 들어온 요청은 최대 10초 뒤에야 받는다. 배차는 먼저 누른 사람이 가져가는 구조라, 이 지연이 기사에게는 손해로 직결된다. 주기를 줄이면 지연은 줄지만 다음 문제가 커진다.

**둘째, 요청이 기사 수에 비례해 늘어난다.** 접속 기사가 500명이고 주기가 5초면 초당 100건이다. 대부분은 "새 수주 없음"을 받아 가는 빈 응답이다. 아무 일도 일어나지 않는 새벽에도 서버는 같은 양의 요청을 받는다.

```
[2023-08-02 03:14:20] GET /api/orders/new  driver=1042  → 200 []
[2023-08-02 03:14:25] GET /api/orders/new  driver=1042  → 200 []
[2023-08-02 03:14:30] GET /api/orders/new  driver=1042  → 200 []
```

새 수주가 뜰 때까지 이 로그가 계속 쌓인다.

## 선택: WebSocket이 아니라 SSE

실시간으로 바꾸기로 하고 세 가지를 비교했다.

**폴링 주기 단축** — 가장 쉽지만 요청량이 비례해서 늘고 지연도 완전히 없어지지 않는다. 근본 해결이 아니다.

**WebSocket** — 가장 먼저 떠오르는 선택이었다. 그런데 이 기능에 필요한 통신은 **서버 → 기사 앱 단방향**이었다. 기사가 수락을 누르는 동작은 이미 일반 HTTP POST로 처리하고 있었고, 그걸 소켓으로 옮길 이유가 없었다. 양방향 채널을 열면 연결 상태 관리, 재연결, 핑/퐁, 프로토콜 설계가 전부 내 몫이 된다. 백엔드를 혼자 맡고 있었고, 안 써도 되는 복잡도를 떠안을 상황이 아니었다.

**SSE (Server-Sent Events)** — 서버에서 클라이언트로 밀어주는 단방향 스트림이다. HTTP 위에서 동작해서 별도 프로토콜이 없고, 끊기면 브라우저·클라이언트가 자동 재연결한다. 필요한 게 딱 이것이었다.

단방향이면 SSE, 양방향이 필요하면 WebSocket. 기준을 이렇게 두고 SSE를 택했다.

## 구현

연결을 보관하고, 발주가 생기면 해당 기사들에게만 밀어준다.

```java
@Service
public class SseService {

    private final Map<Long, SseEmitter> emitters = new ConcurrentHashMap<>();
    private static final long TIMEOUT = 30 * 60 * 1000L; // 30분

    public SseEmitter subscribe(Long driverId) {
        SseEmitter emitter = new SseEmitter(TIMEOUT);

        emitter.onCompletion(() -> emitters.remove(driverId));
        emitter.onTimeout(() -> emitters.remove(driverId));
        emitter.onError(e -> emitters.remove(driverId));

        emitters.put(driverId, emitter);

        // 연결 직후 더미 이벤트 — 프록시가 응답을 버퍼링해 연결이 끊기는 것을 방지
        send(driverId, "connected", Map.of("driverId", driverId));

        return emitter;
    }

    public void send(Long driverId, String eventName, Object data) {
        SseEmitter emitter = emitters.get(driverId);
        if (emitter == null) return;

        try {
            emitter.send(SseEmitter.event().name(eventName).data(data));
        } catch (IOException e) {
            emitters.remove(driverId);
        }
    }
}
```

컨트롤러는 한 줄이다.

```java
@GetMapping(value = "/api/orders/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public SseEmitter stream(@AuthenticationPrincipal Driver driver) {
    return sseService.subscribe(driver.getId());
}
```

발주가 생기면 조건에 맞는 기사에게 이벤트를 밀어준다.

```java
List<Long> targetDriverIds = findMatchingDrivers(order);
for (Long driverId : targetDriverIds) {
    sseService.send(driverId, "new-order", toOrderPayload(order));
}
```

## FCM은 버리지 않았다

SSE는 앱이 열려 있고 연결이 살아 있을 때만 동작한다. 기사가 앱을 닫아두거나 백그라운드로 내려두면 스트림은 끊긴다. 배차는 앱을 안 보고 있을 때도 알아야 하는 정보다.

그래서 **역할을 나눴다.**

- **SSE** — 앱을 보고 있는 기사에게 목록을 즉시 갱신
- **FCM** — 앱이 닫혀 있는 기사에게 푸시로 알림

둘 중 하나를 고르는 문제가 아니라, 커버하는 상황이 다른 두 수단이었다. 처음엔 FCM만 있었고 목록 갱신은 폴링이었는데, 앞단을 SSE로 바꾸면서 화면이 즉시 반응하게 됐다.

## 걸렸던 것들

**1. 프록시 버퍼링.** 리버스 프록시가 응답을 모아서 보내려고 하면 이벤트가 제때 나가지 않는다. 연결 직후 더미 이벤트를 한 번 흘려보내고, 프록시 쪽 버퍼링을 끄는 설정을 함께 봐야 한다.

**2. 타임아웃을 안 두면 연결이 쌓인다.** 비정상 종료된 클라이언트의 emitter가 맵에 남는다. 타임아웃을 두고 `onTimeout`·`onError`에서 반드시 제거해야 한다. 이걸 빼먹으면 메모리가 천천히 샌다.

**3. 멀티 인스턴스에서는 이대로 안 된다.** 연결을 서버 메모리의 `ConcurrentHashMap`에 들고 있으므로, 서버가 2대가 되면 다른 인스턴스에 붙은 기사에게는 이벤트가 가지 않는다. 이 서비스는 단일 인스턴스로 충분한 규모였지만, 확장할 때는 Redis Pub/Sub 같은 걸 앞에 둬서 모든 인스턴스가 이벤트를 받도록 바꿔야 한다. **지금 안 해도 되는 일과, 나중에 반드시 해야 하는 일을 구분해서 기록으로 남겨두는 게 중요하다.**

## 마무리

"실시간"이 필요하다고 하면 반사적으로 WebSocket을 떠올리기 쉽다. 그런데 요구사항을 적어보면 단방향 푸시로 끝나는 경우가 꽤 많다. 그럴 때 SSE는 HTTP 위에서 돌고, 재연결이 공짜고, 구현이 짧다.

양방향이 정말 필요한지 한 번만 물어보면, 안 써도 되는 복잡도를 피할 수 있다.
