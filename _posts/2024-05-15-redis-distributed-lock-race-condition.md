---
title: "Redis 분산 락으로 배차 중복 수락 Race Condition 해결하기"
date: 2024-05-15
categories: [backend]
tags: [redis, distributed-lock, race-condition, spring-boot, concurrency]
---

실시간 배차 서비스에서 20~30명의 기사가 동시에 수주 버튼을 누르면 어떤 일이 발생할까? 이 글에서는 실제 서비스에서 발생한 배차 중복 수락 문제를 Redis 분산 락으로 해결한 과정을 정리한다.

## 배경

고소작업차 배차 서비스를 운영하면서 기사 약 2,000명, 일 배차 30~40건 규모의 트래픽을 처리하고 있었다. 배차 요청이 들어오면 조건에 맞는 기사들에게 FCM 푸시 알림을 보내고, 기사가 "수락" 버튼을 누르면 해당 배차가 확정되는 구조다.

## 문제 발생

서비스 오픈 초기에는 문제가 없었다. 그런데 인기 지역 배차 건에서 동시 수락 요청이 몰리기 시작했다. 20~30명이 거의 동시에 수락 버튼을 누르는 상황이 발생한 것이다.

```
[2024-03-12 09:15:32.001] 기사A 배차#1234 수락 요청
[2024-03-12 09:15:32.003] 기사B 배차#1234 수락 요청
[2024-03-12 09:15:32.005] 기사A 배차#1234 수락 완료
[2024-03-12 09:15:32.007] 기사B 배차#1234 수락 완료  // 중복 수락!
```

기사 A와 B 모두 수락 처리가 되어버렸다. 두 기사 모두 현장에 출동하는 황당한 상황이 발생했다.

## 원인 분석

기존 수락 로직은 이런 흐름이었다:

```java
@Transactional
public void acceptDispatch(Long dispatchId, Long driverId) {
    Dispatch dispatch = dispatchRepository.findById(dispatchId)
        .orElseThrow(() -> new NotFoundException("배차 없음"));

    if (dispatch.getStatus() != DispatchStatus.PENDING) {
        throw new AlreadyAcceptedException("이미 수락된 배차");
    }

    dispatch.accept(driverId);
    dispatchRepository.save(dispatch);
}
```

전형적인 **Check-Then-Act** 패턴이다. `findById`로 상태를 확인하고 → 상태가 PENDING이면 → 수락 처리. 단일 스레드에서는 문제없지만, 동시 요청이 들어오면 두 트랜잭션 모두 PENDING 상태를 읽고 각각 수락 처리를 해버린다.

### 왜 DB 트랜잭션으로 안 되나?

MySQL의 기본 격리 수준인 REPEATABLE READ에서는 각 트랜잭션이 시작 시점의 스냅샷을 읽는다. 두 트랜잭션이 거의 동시에 시작되면 둘 다 PENDING 상태를 보게 된다.

`SELECT ... FOR UPDATE` 비관적 락도 고려했지만:
- 서버가 2대 이상이면 DB 락만으로 부족할 수 있다
- 락 대기 시간이 길어지면 커넥션 풀이 소진될 위험이 있다
- 배차 수락은 빠른 응답이 필수 (기사 UX)

## 해결: Redis 분산 락

Redis의 `SET NX EX` 명령으로 분산 락을 구현했다. Redisson 라이브러리를 사용하면 더 편하지만, 이 케이스는 단순해서 직접 구현했다.

### 핵심 아이디어

배차 ID를 키로 한 락을 먼저 획득한 기사만 수락 가능하고, 나머지는 즉시 실패 응답을 받는다.

```java
@Component
@RequiredArgsConstructor
public class DispatchLockManager {

    private final StringRedisTemplate redisTemplate;
    private static final long LOCK_TIMEOUT_SECONDS = 5;

    public boolean tryLock(Long dispatchId, Long driverId) {
        String key = "dispatch:lock:" + dispatchId;
        String value = String.valueOf(driverId);

        Boolean acquired = redisTemplate.opsForValue()
            .setIfAbsent(key, value, LOCK_TIMEOUT_SECONDS, TimeUnit.SECONDS);

        return Boolean.TRUE.equals(acquired);
    }

    public void unlock(Long dispatchId, Long driverId) {
        String key = "dispatch:lock:" + dispatchId;
        String currentValue = redisTemplate.opsForValue().get(key);

        // 본인이 획득한 락만 해제
        if (String.valueOf(driverId).equals(currentValue)) {
            redisTemplate.delete(key);
        }
    }
}
```

### 수락 로직 수정

```java
@Service
@RequiredArgsConstructor
public class DispatchService {

    private final DispatchRepository dispatchRepository;
    private final DispatchLockManager lockManager;

    public DispatchAcceptResult acceptDispatch(Long dispatchId, Long driverId) {
        // 1. 분산 락 획득 시도
        if (!lockManager.tryLock(dispatchId, driverId)) {
            return DispatchAcceptResult.ALREADY_TAKEN;
        }

        try {
            // 2. DB에서 상태 재확인 (방어 코드)
            Dispatch dispatch = dispatchRepository.findById(dispatchId)
                .orElseThrow(() -> new NotFoundException("배차 없음"));

            if (dispatch.getStatus() != DispatchStatus.PENDING) {
                return DispatchAcceptResult.ALREADY_TAKEN;
            }

            // 3. 수락 처리
            dispatch.accept(driverId);
            dispatchRepository.save(dispatch);

            return DispatchAcceptResult.SUCCESS;
        } finally {
            lockManager.unlock(dispatchId, driverId);
        }
    }
}
```

### 왜 SET NX EX인가?

- `NX` (Not Exists): 키가 없을 때만 설정 → 선착순 1명만 락 획득
- `EX` (Expire): TTL 설정 → 서버 장애로 unlock이 안 돼도 자동 해제
- 원자적 연산: SET NX와 EX가 하나의 명령으로 실행

## 결과

분산 락 적용 후 3개월간 모니터링한 결과:

- **중복 수락 에러 0건** (로그 분석으로 확인)
- 락 획득 실패 시 즉시 응답 → 기사 UX 개선
- 락 대기가 없으므로 커넥션 풀 소진 문제 없음

## 주의할 점

1. **TTL 설정은 반드시 해야 한다.** 서버가 비정상 종료되면 unlock이 호출되지 않는다. TTL이 없으면 영원히 락이 걸린 채로 남는다.

2. **본인 락만 해제해야 한다.** TTL로 락이 자동 해제된 후 다른 기사가 획득한 락을 원래 기사가 해제하면 안 된다. 값 비교로 방어한다.

3. **Redis 단일 장애 지점을 고려해야 한다.** 이 서비스 규모에서는 Redis Standalone으로 충분했지만, 대규모 서비스라면 Redlock 알고리즘이나 Redis Cluster를 검토해야 한다.

## 마무리

동시성 문제는 테스트 환경에서 재현이 어렵다. 로컬에서 10번 돌려보고 "잘 되네"라고 넘어가면 프로덕션에서 꼭 터진다. Redis 분산 락은 구현이 간단하면서도 동시성 문제를 깔끔하게 해결해준다. 다만 은탄환은 아니므로, 서비스 규모와 요구사항에 맞는 수준의 락 전략을 선택하는 게 중요하다.
