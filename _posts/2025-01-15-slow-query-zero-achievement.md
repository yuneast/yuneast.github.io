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

일주일간 모니터링한 결과, 1초 이상 걸리는 쿼리가 하루 평균 15~20건 발견됐다. 패턴을 분류해보니 크게 3가지였다.

### 패턴 1: 인덱스 미사용 풀 스캔

가장 많은 케이스. 조건절 컬럼에 인덱스가 없어서 테이블 전체를 스캔하고 있었다.

```sql
-- 학원별 학생 목록 조회 (슬로우 쿼리)
SELECT * FROM students
WHERE academy_id = 123
AND status = 'ACTIVE'
ORDER BY created_at DESC;

-- EXPLAIN 결과
-- type: ALL, rows: 45000, Extra: Using where; Using filesort
```

학생 테이블이 4만 5천 건인데, 인덱스 없이 전체를 스캔하고 있었다.

**해결:**

```sql
CREATE INDEX idx_students_academy_status ON students(academy_id, status, created_at);
```

복합 인덱스를 만들 때 **카디널리티가 높은 컬럼을 앞에** 배치했다. `academy_id`가 학원별로 고유하니 가장 앞에, `status`는 ACTIVE/INACTIVE 2가지이니 중간에, `created_at`은 정렬용이니 마지막에 배치했다.

인덱스 적용 후:

```
-- type: ref, rows: 120, Extra: Using index condition
```

4만 5천 건 풀 스캔 → 120건 인덱스 스캔으로 변경. 실행 시간 1.2초 → 3ms.

### 패턴 2: 서브쿼리 N+1

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

### 패턴 3: LIKE '%keyword%' 양방향 와일드카드

학생 검색 기능에서 이름, 전화번호를 검색하는 쿼리.

```sql
SELECT * FROM students
WHERE academy_id = 123
AND (name LIKE '%김%' OR phone LIKE '%1234%');
```

`LIKE '%value%'`는 인덱스를 사용할 수 없다. 왼쪽 와일드카드가 있으면 B-Tree 인덱스의 정렬 순서를 활용할 수 없기 때문이다.

이 경우는 인덱스로 해결할 수 없다. 대안을 검토했다:

| 방법 | 장점 | 단점 |
|------|------|------|
| Full-Text Index | MySQL 기본 기능 | 한글 지원 부족 |
| Elasticsearch | 강력한 검색 기능 | 인프라 추가 필요 |
| 쿼리 범위 축소 | 추가 인프라 불필요 | 근본 해결은 아님 |

서비스 규모를 고려해서 Elasticsearch 도입 대신 **쿼리 범위를 축소**하는 방향으로 결정했다.

**해결: `academy_id` 인덱스로 1차 필터링**

```sql
-- academy_id 인덱스로 먼저 범위를 좁히고, 그 안에서 LIKE 검색
SELECT * FROM students
WHERE academy_id = 123  -- 인덱스 스캔 (평균 300건)
AND (name LIKE '%김%' OR phone LIKE '%1234%');  -- 300건 내에서 LIKE
```

`academy_id` 인덱스로 4만 5천 건 → 학원별 평균 300건으로 범위를 좁히면, 300건 내에서의 LIKE 검색은 충분히 빠르다. 실행 시간 1.5초 → 5ms.

## 개선 결과 정리

| 쿼리 유형 | Before | After | 방법 |
|-----------|--------|-------|------|
| 인덱스 미사용 | 1.2초 | 3ms | 복합 인덱스 추가 |
| 서브쿼리 N+1 | 2.5초 | 15ms | JOIN + GROUP BY |
| LIKE 양방향 | 1.5초 | 5ms | 인덱스로 범위 축소 |

3개월간 모니터링 결과, **1초 이상 슬로우 쿼리 0건** 유지 중.

## 쿼리 최적화 체크리스트

이번 작업을 하면서 정리한 체크리스트:

1. **EXPLAIN을 먼저 확인한다.** type이 ALL이면 풀 스캔이다. 인덱스를 의심하자.
2. **WHERE 절 컬럼에 인덱스가 있는지 확인한다.** 없으면 추가한다.
3. **복합 인덱스는 카디널리티 순서대로.** 고유값이 많은 컬럼을 앞에.
4. **서브쿼리보다 JOIN을 우선 검토한다.** 특히 반복 실행되는 상관 서브쿼리는 JOIN으로 변환.
5. **LIKE '%value%'는 인덱스를 못 탄다.** 다른 조건으로 범위를 먼저 좁히자.
6. **인덱스를 과도하게 만들지 않는다.** INSERT/UPDATE 성능에 영향을 준다. 실제 슬로우 쿼리에 필요한 것만.

## 마무리

쿼리 최적화의 시작은 모니터링이다. 슬로우 쿼리 로그를 켜지 않으면 어떤 쿼리가 느린지 알 수 없다. 느린 쿼리를 찾았으면 EXPLAIN으로 실행 계획을 확인하고, 대부분의 경우 적절한 인덱스 추가로 해결된다. 복잡한 쿼리 튜닝보다 인덱스 한 줄이 더 효과적인 경우가 많다.
