---
title: "4GB 서버 한 대에 전부 올리기 - 1인 운영의 구성"
date: 2026-10-20
categories: [devops]
tags: [docker, docker-compose, caddy, postgresql, backup, monitoring, vps]
---

4 vCPU / 4 GB VPS 한 대. 여기에 운영 서비스와 DB와 모니터링과 개발 환경을 다 올려야 했다. 늘릴 수 없으니 안 쓰는 걸 줄이는 수밖에 없었다.

## 전제

배드민턴협회 회원관리·전자결재 시스템을 혼자 만들고 운영한다. 기획부터 배포, 장애 대응까지 1인이다. 서버를 더 살 수도 있지만 협회 예산이고, 그보다 **한 대로 되는 일을 두 대로 하지 않는 게 운영 부담이 작다.**

올라가 있는 것은 이렇다.

```
prod       frontend(Next.js) · backend(Spring Boot) · Caddy
prod DB    PostgreSQL · 백업 컨테이너   (별도 lifecycle)
monitoring Uptime Kuma                 (별도 lifecycle)
dev        frontend · backend          (DB는 외부 서버)
```

평시 메모리는 1.9 GB / 4 GB. 여유 52%를 유지한다.

## 1. 런타임 이미지를 작게

가장 먼저 한 건 이미지에서 안 쓰는 걸 빼는 것이었다. 멀티스테이지 빌드로 **빌드 도구를 런타임에 안 가져간다.**

```dockerfile
# --- build ---
FROM eclipse-temurin:21-jdk AS build
WORKDIR /app
COPY gradlew ./
COPY gradle ./gradle
COPY build.gradle.kts settings.gradle.kts ./
RUN chmod +x gradlew && ./gradlew dependencies --no-daemon || true
COPY src ./src
RUN ./gradlew bootJar --no-daemon

# --- run ---
FROM eclipse-temurin:21-jre AS run
WORKDIR /app
COPY --from=build /app/build/libs/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

런타임은 `jre`다. JDK와 Gradle, 소스는 빌드 스테이지에 남고 최종 이미지에 들어오지 않는다.

프런트엔드는 Next.js의 `standalone` 출력을 쓴다.

```js
// next.config.ts
const nextConfig = {
  output: "standalone",  // 서버 구동에 필요한 것만 추려서 번들
};
```

`node_modules` 전체를 올리는 것과 `standalone` 산출물만 올리는 것의 차이가 꽤 크다. 베이스도 `node:22-alpine`을 쓴다.

## 2. prod DB를 앱과 분리

처음엔 하나의 compose 파일에 전부 넣었다. 그러다 깨달았다. **`docker compose up --build`를 하면 DB 컨테이너도 같이 재시작 후보가 된다.** 배포 한 번에 DB가 내려갈 수 있는 구성이었다.

그래서 파일을 쪼갰다.

```
docker-compose.prod.yml            앱 (frontend/backend/caddy)
docker-compose.prod-db.yml         DB + 백업        ← 별도 lifecycle
docker-compose.prod-monitoring.yml Uptime Kuma      ← 별도 lifecycle
```

앱을 몇 번 재배포해도 DB 컨테이너는 건드려지지 않는다. 연결은 외부 도커 네트워크로 한다.

```yaml
# docker-compose.prod.yml (발췌)
services:
  backend:
    networks: [prod-net]
    # host 포트를 열지 않는다 — Caddy 경유만
networks:
  prod-net:
    external: true
```

DB는 **호스트 포트를 노출하지 않는다.** 서비스명으로만 접근 가능하다. 방화벽 규칙을 잘못 건드려도 외부에서 DB에 닿을 경로가 애초에 없다.

## 3. Caddy를 단일 진입점으로

앱 컨테이너들의 host 포트를 다 닫고, Caddy만 80/443을 연다. TLS는 Caddy가 자동으로 받아온다.

```
cjbadminton.kr {
    reverse_proxy frontend:3000
}

api.cjbadminton.kr {
    reverse_proxy backend:8080
}
```

인증서 갱신 스크립트를 따로 돌리지 않아도 된다. 1인 운영에서 **잊어도 되는 일을 하나 줄이는 것**은 메모리 절약보다 가치가 크다.

## 4. 매일 백업을 외부로

DB가 한 대에 있으면 그 한 대가 날아가는 게 유일한 치명적 시나리오다. `pg_dump`를 매일 돌려 오브젝트 스토리지(Cloudflare R2)로 보낸다.

같은 서버에 백업을 두면 의미가 없다. **서버가 죽는 상황을 대비하는 백업이 서버 안에 있으면 같이 죽는다.**

스키마도 추적 대상으로 바꿨다. 처음엔 `ddl-auto`로 테이블을 만들다가 Flyway로 전환했다. 지금 마이그레이션이 45개다. 어떤 변경이 언제 들어갔는지 남고, 되돌릴 수 있다. `ddl-auto`는 로컬에서는 편하지만 운영에서 쓰면 **스키마가 코드의 부작용으로 바뀐다.**

## 5. 기능을 안 만든 것도 결정이다

알림 기능(카카오 알림톡, SMS 푸시)을 1차 범위에서 뺐다. 상시 돌아야 하는 워커와 외부 연동이 생기고, 그만큼 메모리와 운영 부담이 올라간다.

협회에 물어보니 **사이트에서 상태를 확인하는 것만으로 충분하다**고 했다. 안 만들어도 되는 기능이었다. 4GB에서 돌려야 하는 상황에서, 요구사항을 다시 확인한 게 리소스 최적화 중에 제일 효과가 컸다.

## 지금 보이는 한계

**JVM 힙 상한을 안 걸었다.** `-Xmx` 없이 기본값으로 돌고 있다. 평시 여유가 52%라 당장 문제는 없지만, 컨테이너 메모리 제한과 힙 상한을 명시해두는 게 맞다. 피크가 3.5GB를 넘나들기 시작하면 바로 손대야 하는 부분이다.

**p95가 2초 근처다.** 자랑할 숫자가 아니다. `pg_stat_statements`에 데이터는 쌓여 있으니 느린 쿼리를 특정해서 내리는 게 다음 작업이다.

**단일 장애 지점이다.** 서버 한 대가 멈추면 서비스가 멈춘다. 상태페이지와 텔레그램 알림으로 멈춘 걸 빨리 아는 것까지는 해뒀지만, 그게 이중화는 아니다. 이 서비스의 규모와 예산에서는 받아들인 트레이드오프다.

## 마무리

제약이 있는 환경이 나쁘기만 한 건 아니었다. 메모리를 아껴야 하니 이미지에 뭐가 들어가는지 보게 되고, 서버가 한 대니 배포가 DB를 건드리지 않는지 확인하게 됐다. 넉넉한 환경이었다면 그냥 넘어갔을 것들이다.

1인 운영에서 중요한 건 성능 숫자보다 **잊어도 되는 일을 늘리는 것**이다. 인증서 갱신, 백업, 스키마 변경 추적을 자동으로 돌게 해두면, 남은 시간을 기능에 쓸 수 있다.
