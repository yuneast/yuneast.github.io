---
title: "위치 기반 알림 최적화로 불필요한 알림 60% 줄이기"
date: 2024-02-10
categories: [backend]
tags: [gis, fcm, push-notification, mysql, spatial, spring-boot]
---

배차 서비스에서 새 배차가 등록되면 모든 기사에게 알림을 보내고 있었다. 서울에 있는 기사에게 부산 배차 알림이 가고, 인천 기사에게 대전 배차 알림이 갔다. 불필요한 알림을 줄이기 위해 위치 기반 알림 시스템을 구현한 과정을 정리한다.

## 문제

고소작업차 배차 서비스에서 기사 약 2,000명에게 FCM 푸시 알림을 보내고 있었다. 초기에는 단순하게 전체 기사에게 알림을 발송했다.

```java
// 기존 로직: 모든 기사에게 알림 발송
public void notifyNewDispatch(Dispatch dispatch) {
    List<Driver> allDrivers = driverRepository.findAll();

    for (Driver driver : allDrivers) {
        fcmService.send(driver.getFcmToken(), dispatch.toMessage());
    }
}
```

문제점:
- 일 1,500건 알림 발송 (배차 40건 x 기사 수)
- 기사 불만: "관련 없는 지역 알림이 너무 많아요"
- 배차 수락률 40%로 낮음 (관심 없는 알림이 대부분)
- FCM 발송 비용 증가

## 해결 방향

기사의 현재 위치(차고지)를 기반으로 배차 현장과의 거리를 계산해서, 설정한 반경 내의 기사에게만 알림을 보내는 방식으로 변경했다.

### DB 설계

MySQL 8.0의 Spatial 기능을 사용했다.

```sql
-- 기사 테이블에 위치 컬럼 추가
ALTER TABLE drivers ADD COLUMN location POINT SRID 4326;
ALTER TABLE drivers ADD COLUMN notify_radius_km INT DEFAULT 50;
ALTER TABLE drivers ADD SPATIAL INDEX idx_driver_location (location);

-- 배차 테이블에 현장 위치 추가
ALTER TABLE dispatches ADD COLUMN site_location POINT SRID 4326;
```

`SRID 4326`은 WGS84 좌표계로, GPS 좌표(위도/경도)를 사용하는 표준이다.

### 기사 위치 업데이트 API

기사 앱에서 주기적으로 위치를 업데이트한다.

```java
@RestController
@RequestMapping("/api/drivers")
@RequiredArgsConstructor
public class DriverLocationController {

    private final DriverRepository driverRepository;

    @PutMapping("/me/location")
    public ResponseEntity<Void> updateLocation(
            @AuthDriver Driver driver,
            @RequestBody LocationUpdateRequest request) {

        driver.updateLocation(request.getLatitude(), request.getLongitude());
        driverRepository.save(driver);

        return ResponseEntity.ok().build();
    }
}
```

```java
// Driver 엔티티
@Entity
public class Driver {

    @Column(columnDefinition = "POINT SRID 4326")
    private Point location;

    @Column
    private Integer notifyRadiusKm = 50; // 기본 50km

    public void updateLocation(double latitude, double longitude) {
        GeometryFactory factory = new GeometryFactory(new PrecisionModel(), 4326);
        this.location = factory.createPoint(new Coordinate(longitude, latitude));
    }
}
```

주의: `Point` 생성 시 **경도(longitude)가 먼저, 위도(latitude)가 나중**이다. 이 순서를 실수하면 위치가 완전히 틀어진다.

### 반경 내 기사 조회

MySQL의 `ST_Distance_Sphere` 함수로 두 좌표 간 거리를 미터 단위로 계산한다.

```java
// DriverRepository
public interface DriverRepository extends JpaRepository<Driver, Long> {

    @Query(value = """
        SELECT d.* FROM drivers d
        WHERE d.location IS NOT NULL
        AND d.fcm_token IS NOT NULL
        AND ST_Distance_Sphere(d.location, ST_GeomFromText(:sitePoint, 4326))
            <= d.notify_radius_km * 1000
        """, nativeQuery = true)
    List<Driver> findDriversInRadius(@Param("sitePoint") String sitePoint);
}
```

### 알림 발송 로직 수정

```java
@Service
@RequiredArgsConstructor
public class DispatchNotificationService {

    private final DriverRepository driverRepository;
    private final FcmService fcmService;

    public void notifyNearbyDrivers(Dispatch dispatch) {
        String sitePoint = String.format(
            "POINT(%f %f)",
            dispatch.getSiteLongitude(),
            dispatch.getSiteLatitude()
        );

        List<Driver> nearbyDrivers = driverRepository.findDriversInRadius(sitePoint);

        List<String> tokens = nearbyDrivers.stream()
            .map(Driver::getFcmToken)
            .filter(Objects::nonNull)
            .toList();

        if (!tokens.isEmpty()) {
            fcmService.sendMulticast(tokens, dispatch.toMessage());
        }
    }
}
```

### 기사별 알림 반경 설정

기사마다 활동 반경이 다르다. 서울 시내만 다니는 기사는 20km, 전국 단위로 이동하는 기사는 200km. 기사가 직접 반경을 설정할 수 있게 했다.

```java
@PutMapping("/me/notify-radius")
public ResponseEntity<Void> updateNotifyRadius(
        @AuthDriver Driver driver,
        @RequestBody @Valid RadiusUpdateRequest request) {

    driver.setNotifyRadiusKm(request.getRadiusKm());
    driverRepository.save(driver);

    return ResponseEntity.ok().build();
}
```

```java
public class RadiusUpdateRequest {

    @Min(10)
    @Max(500)
    private Integer radiusKm;
}
```

## 결과

3개월간 데이터 분석 결과:

- **알림 발송 건수: 일 1,500건 → 600건 (60% 감소)**
- **배차 수락률: 40% → 50% (10%p 상승)**
- 기사 불만 감소: 관련 없는 지역 알림 사라짐
- FCM 발송 비용 절감

수락률이 올라간 이유는 간단하다. 가까운 현장의 배차만 알림이 오니까, 실제로 수락할 의향이 있는 기사에게만 알림이 간다.

## Spatial Index 성능

처음에는 "2,000명 중에서 거리 계산하면 느리지 않을까?" 걱정했다. 테스트 결과:

```sql
-- EXPLAIN 결과
EXPLAIN SELECT * FROM drivers
WHERE ST_Distance_Sphere(location, ST_GeomFromText('POINT(127.0 37.5)', 4326)) <= 50000;

-- type: ALL, rows: 2000 (풀 스캔)
```

`ST_Distance_Sphere`는 Spatial Index를 타지 않는다. 2,000건이라 실행 시간은 수 밀리초로 문제 없었지만, 데이터가 늘어날 것을 대비해서 `MBRContains`로 1차 필터링을 추가했다.

```sql
-- Spatial Index를 활용하는 쿼리
SELECT d.* FROM drivers d
WHERE MBRContains(
    ST_Buffer(ST_GeomFromText('POINT(127.0 37.5)', 4326), 0.5),
    d.location
)
AND ST_Distance_Sphere(d.location, ST_GeomFromText('POINT(127.0 37.5)', 4326))
    <= d.notify_radius_km * 1000;
```

`MBRContains`로 사각형 영역 내 대략적 필터링 (Spatial Index 사용) → `ST_Distance_Sphere`로 정확한 거리 계산. 2단계 필터링이다.

## 주의할 점

1. **좌표 순서를 주의하자.** MySQL의 `POINT`는 `POINT(경도 위도)` 순이고, 일반적으로 사람들은 "위도 경도" 순서로 말한다. 이 차이 때문에 초기에 위치가 반대로 나오는 버그가 있었다.

2. **위치 업데이트 빈도를 조절하자.** 기사 앱에서 1초마다 위치를 보내면 서버 부하가 커진다. 이동 중에는 30초, 정차 중에는 5분 간격으로 업데이트하도록 클라이언트에서 제어했다.

3. **위치 정보 없는 기사를 처리하자.** 위치 권한을 거부하거나 GPS가 안 잡히는 기사도 있다. 이런 기사에게는 기존처럼 전체 알림을 보내는 폴백 로직이 필요하다.
