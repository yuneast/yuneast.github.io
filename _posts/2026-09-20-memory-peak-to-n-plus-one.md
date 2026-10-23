---
title: "메모리가 3.1GB까지 치솟았다 - 모니터링에서 N+1까지 거슬러 올라간 기록"
date: 2026-09-20
categories: [backend]
tags: [monitoring, netdata, n-plus-one, jpa, performance, postgresql]
---

엑셀 업로드 한 번에 메모리 그래프가 수직으로 올라갔다. 4GB 서버에서 3.1GB. 남은 게 900MB였다.

## 배경

배드민턴협회 회원관리 시스템을 혼자 만들고 운영하고 있다. 서버는 4 vCPU / 4 GB 단일 VPS 한 대고, 그 위에 앱·DB·리버스프록시·모니터링이 전부 올라가 있다. 스케일아웃은 없다. 이 안에서 버텨야 한다.

협회가 기존 회원 명단을 엑셀로 올려서 한 번에 등록하는 기능이 있다. 동호회 하나 분량을 올리면 수백 행이 들어온다.

## 발견: 사용자가 말해주기 전에 그래프가 말해줬다

운영 가시성이 없는 상태로 시작하면 "느리다"는 제보를 받고 나서야 움직이게 된다. 그게 싫어서 서비스 초기에 모니터링을 먼저 붙였다. 서버 리소스는 Netdata로, DB 쿼리는 `pg_stat_statements`와 pgHero로 본다.

업로드 기능을 테스트하다가 Netdata 메모리 그래프를 봤다.

```
평시      1.9 GB / 4 GB   (여유 52%)
업로드 중  3.1 GB / 4 GB   (여유 22%)
```

1.2GB가 한 번에 올라갔다. 업로드가 끝나면 다시 내려오긴 했지만, 이 시점에 다른 요청이 몰리거나 업로드가 동시에 두 건 들어오면 메모리가 바닥난다. 4GB에서 900MB 남는 상태는 안전하지 않다.

## 원인: 행마다 쿼리가 돌고 있었다

pgHero에서 호출 횟수 순으로 정렬하니 바로 보였다. 업로드 한 번에 같은 형태의 쿼리가 수백 번 실행되고 있었다.

회원 명부를 올리면 각 행마다 **이미 등록된 회원인지 이름+연락처로 매칭**해야 한다. 그 매칭을 행 단위로 하고 있었다.

```java
// 문제가 있던 코드
for (RosterRow row : rows) {
    // 행마다 조회 1번
    Optional<Member> existing = memberRepository
            .findByNameAndPhone(row.getName(), row.getPhone());

    if (existing.isPresent()) {
        updateMember(existing.get(), row);
    } else {
        createMember(row);
    }

    // 처리자 이름을 쓰려고 행마다 또 조회
    String operatorName = userRepository.findById(operatorId)
            .map(User::getName)
            .orElse("-");
    writeHistory(row, operatorName);
}
```

두 가지가 겹쳐 있었다.

**1. 매칭 쿼리가 행 수만큼 실행된다.** 500행이면 500번이다. 전형적인 N+1이다.

**2. 처리자 이름을 루프 안에서 매번 조회한다.** 루프가 끝날 때까지 같은 값이 바뀌지 않는데도 매번 DB에 묻는다. 이건 N+1이라기보다 그냥 실수다.

메모리가 오른 건 결과를 전부 리스트에 모아 들고 있었던 것도 한몫했다. 엔티티 수백 개와 그에 딸린 영속성 컨텍스트가 업로드가 끝날 때까지 메모리에 남아 있었다.

## 고친 것

### 매칭을 배치로

행별 조회를 없애고, 올라온 전체 키를 한 번에 조회해서 메모리에서 맞췄다.

```java
// 1. 업로드된 모든 (이름, 연락처) 키를 한 번에 조회
List<String> phones = rows.stream().map(RosterRow::getPhone).toList();
Map<String, Member> existingByKey = memberRepository.findAllByPhoneIn(phones)
        .stream()
        .collect(Collectors.toMap(
                m -> matchKey(m.getName(), m.getPhone()),
                Function.identity(),
                (a, b) -> a));

// 2. 루프 밖에서 한 번만 조회
String operatorName = userRepository.findById(operatorId)
        .map(User::getName)
        .orElse("-");

// 3. 루프 안에서는 DB를 치지 않는다
for (RosterRow row : rows) {
    Member existing = existingByKey.get(matchKey(row.getName(), row.getPhone()));
    if (existing != null) {
        updateMember(existing, row);
    } else {
        createMember(row);
    }
    writeHistory(row, operatorName);
}
```

쿼리가 **행 수만큼 → 2번**으로 줄었다.

### 매칭은 정확히 일치만

여기서 설계 판단이 하나 들어간다. 이름과 연락처가 **정확히 일치할 때만** 기존 회원으로 연결했다. 오탈자까지 받아주면 등록은 편해지지만, 잘못 붙으면 **남의 이적 이력이 내 계정에 붙는다.** 가입 편의보다 데이터 무결성을 택했다. 일치하지 않는 행은 등록을 막지 않고 `미배정`으로 남겨 관리자가 나중에 배정한다.

### 응답을 비동기로

업로드 처리가 끝날 때까지 요청을 붙잡고 있을 이유가 없었다. 접수 응답을 먼저 돌려주고 처리는 뒤에서 돌게 바꿨다. 사용자는 바로 화면을 돌려받고, 서버는 피크를 길게 분산시킨다.

## 결과

- 업로드 응답 지연 해소
- 업로드 중 피크 메모리 완화, 평시 1.9GB / 4GB(여유 52%) 유지
- 쿼리 호출 수가 행 수에 비례하지 않게 됨

## 배운 것

**1. 4GB 서버에서는 N+1이 느림이 아니라 장애다.** 메모리가 넉넉한 환경에서 N+1은 "응답이 좀 느리다"로 끝난다. 여유가 900MB인 서버에서는 같은 코드가 OOM 직전까지 간다. 제약이 작으면 같은 실수의 결과가 달라진다.

**2. 모니터링을 먼저 붙여서 얻은 건 이 사건 하나다.** Netdata가 없었으면 "업로드가 좀 느리네" 정도로 넘겼을 것이다. 그래프가 수직으로 올라가는 걸 눈으로 보니까 원인을 찾을 동기가 생겼다. 모니터링은 장애가 났을 때 보는 게 아니라, **장애가 되기 전에 보이게 하는 것**이다.

**3. 루프 안의 DB 호출은 일단 의심한다.** 처리자 이름을 500번 조회하던 코드는 N+1 이론을 몰라서 생긴 게 아니다. 루프 안에 있으면 안 되는 걸 그냥 못 본 것이다. 코드 리뷰어가 없는 1인 개발에서는 이런 게 오래 남는다. 그래서 지표를 리뷰어로 쓴다.

## 마무리

문제를 찾은 순서가 거꾸로였다는 게 이번 기록의 핵심이다. 보통은 코드를 읽다가 N+1을 발견한다. 이번엔 메모리 그래프 → 쿼리 호출 수 → 루프 안의 조회 순서로 거슬러 올라갔다.

혼자 개발하면 코드를 봐줄 사람이 없다. 대신 지표는 거짓말을 하지 않는다.
