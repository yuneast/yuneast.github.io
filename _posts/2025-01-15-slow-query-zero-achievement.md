---
title: "슬로우 쿼리 0건 달성기 - MySQL 쿼리 최적화 실전"
date: 2025-01-15
categories: [backend]
tags: [mysql, query-optimization, slow-query, index, performance]
---

"슬로우 쿼리 로그 좀 켜볼까요?" 학원 ERP 서비스에서 간헐적으로 API 응답이 느려지는 문제가 있었다. 슬로우 쿼리 로그를 켜보니 원인이 보였다. 이 글에서는 슬로우 쿼리를 0건으로 만든 과정을 정리한다.

## 현황 파악

MySQL의 슬로우 쿼리 로그를 활성화했다.

```sql
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 1; -- 1초 이상 쿼리 기록
SET GLOBAL log_output = 'TABLE';
```

일주일간 모니터링한 결과, 1초 이상 걸리는 쿼리가 하루 평균 15~20건 발견됐다. 인덱스는 대체로 잘 잡혀있었지만, **쿼리 구조 자체가 문제**였다. 패턴을 분류해보니 크게 2가지였다.

### 패턴 1: 서브쿼리 N+1 (가장 많았던 문제)

출결 현황 대시보드에서 발생한 쿼리. 반별 출석률을 계산하기 위해 반마다 서브쿼리를 실행하고 있었다.

```sql
-- 반별 출석률 조회 (슬로우 쿼리)
SELECT c.class_name,
  (SELECT COUNT(*) FROM attendance a
   WHERE a.class_id = c.id AND a.date = CURDATE() AND a.status = 'PRESENT')
  AS present_count,
  (SELECT COUNT(*) FROM students s
   WHERE s.class_id = c.id AND s.status = 'ACTIVE')
  AS total_count
FROM classes c
WHERE c.academy_id = 123;
```

반이 30개면 서브쿼리가 60번 실행된다.

**해결: JOIN + GROUP BY로 변환**

```sql
SELECT c.class_name,
  COALESCE(a.present_count, 0) AS present_count,
  COALESCE(s.total_count, 0) AS total_count
FROM classes c
LEFT JOIN (
  SELECT class_id, COUNT(*) AS present_count
  FROM attendance
  WHERE date = CURDATE() AND status = 'PRESENT'
  GROUP BY class_id
) a ON a.class_id = c.id
LEFT JOIN (
  SELECT class_id, COUNT(*) AS total_count
  FROM students
  WHERE status = 'ACTIVE'
  GROUP BY class_id
) s ON s.class_id = c.id
WHERE c.academy_id = 123;
```

서브쿼리 60번 → JOIN 2번으로 줄었다. 실행 시간 2.5초 → 15ms.

### 패턴 2: 전화번호 LIKE 검색

학생 검색 기능에서 전화번호를 검색하는 쿼리가 느렸다.

```sql
SELECT * FROM students
WHERE academy_id = 123
AND phone LIKE '%1234%';
```

`LIKE '%value%'`는 인덱스를 사용할 수 없다. 왼쪽 와일드카드가 있으면 B-Tree 인덱스의 정렬 순서를 활용할 수 없기 때문이다.

**해결: 전화번호 분할 저장**

전화번호를 하이픈 기준으로 분할해서 별도 컬럼에 저장하는 방식으로 변경했다.

```sql
-- 기존 테이블 구조
phone VARCHAR(20)  -- '010-1234-5678'

-- 변경 후
phone VARCHAR(20),          -- '010-1234-5678' (원본 유지)
phone_part1 VARCHAR(5),     -- '010'
phone_part2 VARCHAR(5),     -- '1234'
phone_part3 VARCHAR(5),     -- '5678'
INDEX idx_phone_parts(phone_part1, phone_part2, phone_part3)
```

이제 전화번호 검색이 LIKE가 아닌 **정확한 일치 또는 전방 일치**로 바뀐다:

```sql
-- '010'으로 시작하는 번호 검색
SELECT * FROM students
WHERE academy_id = 123
AND phone_part1 = '010';  -- 인덱스 사용 가능

-- '1234'가 포함된 번호 검색 (중간 또는 끝)
SELECT * FROM students
WHERE academy_id = 123
AND (phone_part2 = '1234' OR phone_part3 = '1234');  -- 인덱스 사용 가능
```

LIKE 패턴 매칭 → 정확한 일치 비교로 변경. 실행 시간 1.5초 → 5ms.

## 개선 결과 정리

| 쿼리 유형 | Before | After | 방법 |
|-----------|--------|-------|------|
| 서브쿼리 N+1 | 2.5초 | 15ms | JOIN + GROUP BY로 변환 |
| 전화번호 LIKE 검색 | 1.5초 | 5ms | 전화번호 분할 저장 + 정확한 일치 검색 |

3개월간 모니터링 결과, **1초 이상 슬로우 쿼리 0건** 유지 중.

## 쿼리 최적화 체크리스트

이번 작업을 하면서 정리한 체크리스트:

1. **인덱스보다 쿼리 구조를 먼저 의심하자.** 인덱스가 잘 잡혀있어도 쿼리 구조가 잘못되면 느리다.
2. **서브쿼리보다 JOIN을 우선 검토한다.** 특히 반복 실행되는 상관 서브쿼리는 N+1 문제를 일으킨다. JOIN + GROUP BY로 변환하자.
3. **LIKE '%value%'는 피할 수 있으면 피한다.** 데이터를 분할 저장해서 정확한 일치 검색으로 바꾸는 게 근본적인 해결책이다.
4. **EXPLAIN으로 실행 계획을 확인한다.** 예상과 다르게 동작하는 쿼리가 많다.
5. **슬로우 쿼리 로그를 주기적으로 모니터링한다.** 문제를 알아야 고칠 수 있다.
6. **인덱스를 과도하게 만들지 않는다.** INSERT/UPDATE 성능에 영향을 준다. 실제 슬로우 쿼리에 필요한 것만.

## 마무리

쿼리 최적화의 시작은 모니터링이다. 슬로우 쿼리 로그를 켜지 않으면 어떤 쿼리가 느린지 알 수 없다. 이번 경험에서 배운 건, **인덱스가 잘 잡혀있어도 쿼리 구조가 잘못되면 느리다**는 것이다. 서브쿼리 N+1이 가장 흔한 문제였고, JOIN으로 변환하는 것만으로도 대부분 해결됐다. 인덱스를 추가하기 전에 쿼리 구조부터 점검하자.
