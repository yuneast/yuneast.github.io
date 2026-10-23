---
title: "통합 인증을 했더니 401이 터졌다 - Refresh 토큰 동시 요청과 싱글플라이트"
date: 2025-09-05
categories: [backend]
tags: [auth, jwt, refresh-token, redis, concurrency, singleflight, idempotency]
---

통합 인증 전환은 1시간 점검으로 끝났다. 문제는 그다음 주에 왔다. "갑자기 로그아웃돼요"라는 문의가 들어오기 시작했다.

## 배경

학원 ERP에서 3개 서비스의 로그인 정보가 각각 다른 테이블에 흩어져 있던 걸, 중앙 인증 서버로 분리해 쿠키 기반 통합 인증으로 묶었다. 그 과정은 [이전 글](/backend/cookie-based-sso-implementation/)에 적었다.

전환 자체는 깔끔했다. Access Token은 짧게, Refresh Token으로 갱신하는 흔한 구조였다.

## 증상: 멀쩡하던 세션이 끊긴다

로그를 보니 패턴이 있었다.

```
[09:14:02.113] GET /api/students      user=4821  → 401 (access expired)
[09:14:02.115] GET /api/attendance    user=4821  → 401 (access expired)
[09:14:02.118] GET /api/notices       user=4821  → 401 (access expired)
[09:14:02.140] POST /auth/refresh     user=4821  → 200  new_refresh=R2
[09:14:02.151] POST /auth/refresh     user=4821  → 200  new_refresh=R3
[09:14:02.157] POST /auth/refresh     user=4821  → 401  invalid refresh token
[09:14:02.160] 세션 종료 → 로그인 화면
```

대시보드를 열면 화면 하나에서 API를 여러 개 동시에 부른다. Access Token이 만료된 순간에 그 요청들이 **동시에 401을 받고, 각자 refresh를 호출한다.**

## 원인: Refresh Token 회전과 동시성이 충돌했다

Refresh Token은 한 번 쓰면 새 것으로 바꾸는 게 원칙이다(회전, rotation). 탈취된 토큰이 계속 쓰이는 걸 막기 위해서다. 그래서 구현이 이랬다.

```java
@Transactional
public TokenPair refresh(String refreshToken) {
    RefreshToken saved = refreshTokenRepository.findByToken(refreshToken)
            .orElseThrow(() -> new InvalidTokenException());

    // 쓴 토큰은 폐기하고 새로 발급
    refreshTokenRepository.delete(saved);
    return issueNewPair(saved.getUserId());
}
```

단일 요청에서는 맞다. 그런데 동시에 세 번 들어오면 이렇게 된다.

1. 요청 A, B, C가 **모두 같은 R1을 들고** 들어온다
2. A가 R1을 소비하고 R2를 발급한다
3. B도 R1으로 조회에 성공한다 (A의 삭제가 아직 커밋되지 않았거나, 이미 읽어둔 상태) → R3 발급
4. C가 들어오니 R1이 이미 없다 → **401**
5. 클라이언트는 401을 받고 "Refresh도 실패했다"고 판단해 로그아웃시킨다

여기에 더해, B가 발급한 R3가 A가 발급한 R2를 덮어쓰면서 **클라이언트가 들고 있는 토큰이 서버 기준으로 무효**가 되는 경우도 생겼다. 어느 쪽이든 결과는 로그아웃이다.

## 해결 1: 싱글플라이트 — 갱신을 한 번으로 모은다

같은 사용자의 refresh 요청이 동시에 들어오면, **한 번만 실제로 갱신하고 나머지는 그 결과를 기다려 같은 토큰을 받게** 했다. Redis에 짧은 TTL의 락을 둔다.

```java
public TokenPair refresh(String refreshToken) {
    Long userId = parseUserId(refreshToken);
    String lockKey = "auth:refresh:lock:" + userId;

    // 먼저 들어온 요청만 갱신을 수행한다
    boolean acquired = redis.opsForValue()
            .setIfAbsent(lockKey, "1", Duration.ofSeconds(5));

    if (!acquired) {
        // 다른 요청이 갱신 중 — 결과가 캐시에 올라오면 그걸 쓴다
        TokenPair fresh = awaitRecentlyIssued(userId);
        if (fresh != null) return fresh;
        throw new RetryableAuthException();
    }

    try {
        TokenPair pair = rotate(refreshToken);
        cacheRecentlyIssued(userId, pair, Duration.ofSeconds(10));
        return pair;
    } finally {
        redis.delete(lockKey);
    }
}
```

## 해결 2: 원자적으로 소비한다

"조회한 뒤 삭제"는 그 사이에 다른 요청이 끼어들 수 있다. Redis에 토큰을 두고 **가져오면서 동시에 삭제**하도록 바꿨다. `GETDEL`은 두 동작이 하나의 명령이라 중간이 없다.

```java
// 조회와 삭제가 한 번에 — 성공한 요청만 값을 받는다
String stored = redis.opsForValue().getAndDelete("auth:refresh:" + refreshToken);
if (stored == null) {
    // 이미 누가 소비했다 → 재사용 시도이거나 동시 요청
    throw new AlreadyConsumedException();
}
```

이렇게 하면 **"토큰을 소비한 요청"이 정확히 하나로 결정된다.** Check-Then-Act가 한 명령으로 합쳐졌다.

## 해결 3: 늦게 온 요청을 실패로 보지 않는다

마지막이 중요했다. 위 둘을 적용해도 C는 여전히 "이미 소비된 토큰"을 들고 온다. 이때 **401을 주면 클라이언트는 로그아웃한다.**

그래서 짧은 유예 구간을 뒀다. 직전에 발급된 결과를 몇 초간 들고 있다가, 이미 소비된 토큰으로 들어온 요청에는 **그 결과를 그대로 돌려준다.** 같은 요청이 여러 번 와도 결과가 같은 멱등 처리다.

```java
catch (AlreadyConsumedException e) {
    TokenPair recent = getRecentlyIssued(userId);   // 10초 캐시
    if (recent != null) return recent;              // 동시 요청으로 간주
    throw new InvalidTokenException();              // 유예 구간 밖 → 진짜 재사용 시도
}
```

유예 구간을 길게 두면 탈취된 토큰의 재사용을 허용하는 셈이 된다. 짧게 둬서 **동시 요청은 흡수하고, 한참 뒤의 재사용은 거절**하는 선을 찾았다.

## 결과

- 동시 요청으로 인한 401 및 강제 로그아웃 차단
- 토큰 중복 발급 차단 — 동시에 N개가 들어와도 발급은 1회

## 배운 것

**1. 인증은 동시성 문제다.** 토큰 회전은 보안을 위한 설계인데, 그 "한 번만 쓸 수 있다"는 성질이 동시 요청과 정면으로 충돌한다. 설계 문서에는 요청이 한 번에 하나씩 오는 것처럼 그려져 있다.

**2. 조회 후 변경은 전부 의심 대상이다.** 배차 중복 수락도, 이것도 같은 모양이었다. `findBy...` → 검사 → `save`가 보이면 동시에 들어올 때 어떻게 되는지 한 번 생각해보는 습관이 생겼다.

**3. 에러 코드가 사용자 경험을 결정한다.** 기술적으로는 "이미 소비된 토큰"이 맞는 상태다. 그런데 그걸 401로 내려보내면 사용자는 로그아웃된다. **같은 내부 상태를 어떤 응답으로 바꿀지가 실제 피해 크기를 정한다.**

## 마무리

전환 작업을 끝내고 "성공했다"고 적었는데, 그 뒤 일주일이 실제 작업이었다. 무중단으로 넘긴 건 시작이고, 넘긴 구조가 실제 트래픽 패턴에서 버티는지는 다른 문제다.

로컬에서 refresh를 한 번씩 호출하면 절대 재현되지 않는다. 브라우저가 API를 동시에 여섯 개 부르는 순간에만 터진다.
