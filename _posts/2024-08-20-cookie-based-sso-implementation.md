---
title: "쿠키 기반 통합 인증(SSO) 구현기 - 3개 서비스를 1시간 점검으로 통합하기"
date: 2024-08-20
categories: [backend]
tags: [sso, authentication, cookie, spring-boot, session]
---

여러 서비스가 각자 로그인 시스템을 갖고 있으면 사용자는 서비스를 이동할 때마다 다시 로그인해야 한다. 이 글에서는 3개 서비스의 인증을 쿠키 기반 SSO로 통합하면서 점검 시간을 1시간으로 최소화한 경험을 정리한다.

## 배경

학원 운영관리(ERP) 서비스를 개발하면서 총 3개의 서비스를 운영하고 있었다:

1. **ERP 서비스** (Spring Boot) - 학원 관리, 출결, 수강 관리
2. **알림 서비스** (Spring Boot) - SMS, 푸시 알림
3. **대시보드 서비스** (PHP Legacy) - 학원장용 통계 대시보드

각 서비스마다 독립적인 세션 관리를 하고 있어서, 학원 관리자가 ERP에서 대시보드로 이동하면 다시 로그인해야 했다. 사용자 불만이 꾸준히 들어왔다.

## 설계 방향

SSO 구현 방식을 검토했다:

| 방식 | 장점 | 단점 |
|------|------|------|
| JWT 토큰 | 서버 간 의존성 없음 | 토큰 폐기 어려움, 크기 큼 |
| OAuth 2.0 | 표준화된 프로토콜 | 별도 인증 서버 필요, 구현 복잡 |
| 쿠키 기반 공유 세션 | 구현 단순, 즉시 폐기 가능 | 같은 도메인 필요 |

3개 서비스가 모두 같은 도메인(`*.service.com`) 하위에 있었기 때문에 쿠키 기반 공유 세션을 선택했다. Redis를 세션 스토어로 사용하면 서비스 간 세션 공유가 가능하다.

## 구현

### 1. 인증 서버 분리

기존 ERP에 박혀있던 인증 로직을 별도 인증 서버로 분리했다.

```java
@RestController
@RequestMapping("/auth")
@RequiredArgsConstructor
public class AuthController {

    private final AuthService authService;

    @PostMapping("/login")
    public ResponseEntity<LoginResponse> login(
            @RequestBody LoginRequest request,
            HttpServletResponse response) {

        String sessionId = authService.login(request);

        // 상위 도메인 쿠키 설정
        Cookie cookie = new Cookie("SESSION_ID", sessionId);
        cookie.setDomain(".service.com");
        cookie.setPath("/");
        cookie.setHttpOnly(true);
        cookie.setSecure(true);
        cookie.setMaxAge(60 * 60 * 8); // 8시간
        response.addCookie(cookie);

        return ResponseEntity.ok(new LoginResponse(sessionId));
    }
}
```

핵심은 `cookie.setDomain(".service.com")`이다. 상위 도메인으로 쿠키를 설정하면 `erp.service.com`, `noti.service.com`, `dashboard.service.com` 모든 하위 도메인에서 같은 쿠키를 읽을 수 있다.

### 2. Redis 세션 스토어

```java
@Service
@RequiredArgsConstructor
public class SessionService {

    private final StringRedisTemplate redisTemplate;
    private static final long SESSION_TTL = 8 * 60 * 60; // 8시간

    public void createSession(String sessionId, UserInfo userInfo) {
        String key = "session:" + sessionId;
        Map<String, String> sessionData = Map.of(
            "userId", String.valueOf(userInfo.getId()),
            "role", userInfo.getRole().name(),
            "academyId", String.valueOf(userInfo.getAcademyId())
        );

        redisTemplate.opsForHash().putAll(key, sessionData);
        redisTemplate.expire(key, SESSION_TTL, TimeUnit.SECONDS);
    }

    public Optional<UserInfo> getSession(String sessionId) {
        String key = "session:" + sessionId;
        Map<Object, Object> data = redisTemplate.opsForHash().entries(key);

        if (data.isEmpty()) {
            return Optional.empty();
        }

        // 접근 시마다 TTL 갱신 (슬라이딩 세션)
        redisTemplate.expire(key, SESSION_TTL, TimeUnit.SECONDS);

        return Optional.of(UserInfo.from(data));
    }
}
```

### 3. 각 서비스에 인증 필터 추가

Spring Boot 서비스에는 인터셉터를, PHP 서비스에는 미들웨어를 추가했다.

**Spring Boot 인터셉터:**

```java
@Component
@RequiredArgsConstructor
public class AuthInterceptor implements HandlerInterceptor {

    private final SessionService sessionService;

    @Override
    public boolean preHandle(HttpServletRequest request,
                            HttpServletResponse response,
                            Object handler) {
        String sessionId = extractSessionId(request);

        if (sessionId == null) {
            response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
            return false;
        }

        Optional<UserInfo> userInfo = sessionService.getSession(sessionId);
        if (userInfo.isEmpty()) {
            response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
            return false;
        }

        request.setAttribute("currentUser", userInfo.get());
        return true;
    }

    private String extractSessionId(HttpServletRequest request) {
        if (request.getCookies() == null) return null;

        return Arrays.stream(request.getCookies())
            .filter(c -> "SESSION_ID".equals(c.getName()))
            .map(Cookie::getValue)
            .findFirst()
            .orElse(null);
    }
}
```

**PHP 미들웨어:**

```php
class AuthMiddleware {
    private $redis;

    public function handle($request, $next) {
        $sessionId = $_COOKIE['SESSION_ID'] ?? null;

        if (!$sessionId) {
            return redirect('/login');
        }

        $sessionData = $this->redis->hGetAll("session:" . $sessionId);

        if (empty($sessionData)) {
            return redirect('/login');
        }

        // TTL 갱신
        $this->redis->expire("session:" . $sessionId, 8 * 3600);

        $request->setUser($sessionData);
        return $next($request);
    }
}
```

### 4. 마이그레이션 전략

기존 사용자 세션을 끊지 않고 전환하는 게 목표였다. 아래 순서로 진행했다:

1. **사전 배포 (점검 없이)**: 인증 서버 배포, 각 서비스에 인증 필터 추가 (비활성 상태)
2. **점검 시작**: 기존 세션 데이터를 Redis로 마이그레이션 스크립트 실행
3. **인증 필터 활성화**: 환경변수 변경으로 신규 인증 로직 ON
4. **점검 종료**: 사용자 재로그인 없이 서비스 이용 가능

점검 시간 1시간, 기존 사용자는 재로그인 없이 전환 완료.

## 결과

- 3개 서비스 통합 인증 완성
- 점검 시간 1시간으로 최소화
- 사용자 재로그인 불편 제거
- 이후 신규 서비스 추가 시 인증 필터만 붙이면 됨

## 배운 점

1. **쿠키 도메인 설정이 핵심이다.** `.service.com`처럼 앞에 점을 붙이면 모든 서브도메인에서 접근 가능하다. 이걸 모르면 서비스마다 별도 쿠키를 관리하게 된다.

2. **마이그레이션 전략이 기능 구현보다 중요하다.** SSO 자체는 어렵지 않다. 기존 사용자 세션을 끊지 않고 전환하는 게 진짜 어려운 부분이다.

3. **PHP와 Java가 같은 Redis를 읽어야 한다.** 세션 데이터의 직렬화 형식을 통일해야 한다. 처음에 Java의 기본 직렬화를 사용했다가 PHP에서 읽지 못하는 문제가 있었고, 단순한 Hash 구조로 변경해서 해결했다.
