---
title: "학원 400곳이 한 DB를 쓴다 - 멀티테넌트에서 WHERE 하나를 빼먹으면"
date: 2025-05-20
categories: [backend]
tags: [multi-tenant, jpa, hibernate-filter, data-isolation, spring-boot, legacy]
---

레거시 PHP를 Java로 이관하면서 가장 무서웠던 건 성능이 아니었다. **다른 학원의 학생 명단이 보이는 것**이었다.

## 배경

학원 ERP를 운영하고 있었다. 학원 400곳 정도가 같은 서비스를 쓰고, 학원마다 원장·강사·학생·학부모 계정이 따로 있다. 테넌트는 학원이다.

레거시 PHP는 단일 스키마에 `academy_id` 컬럼을 두는 방식이었다. 모든 쿼리가 `WHERE academy_id = ?`를 직접 달고 있었다.

이관하면서 테넌트 분리 전략을 다시 정해야 했다.

## 문제

선택지는 셋이었다.

**① 테넌트별 DB** — 완전 격리. 한 학원 데이터가 다른 학원에 새는 게 구조적으로 불가능하다. 하지만 400개 DB에 커넥션 풀을 어떻게 나눌지가 안 풀렸다. 풀을 5개씩만 잡아도 2,000 커넥션이다. 스키마 변경도 400번 돌려야 하고, 중간에 하나 실패하면 버전이 갈린다.

**② 테넌트별 스키마** — 같은 DB 안에서 스키마만 분리. 커넥션은 공유되지만 마이그레이션 문제는 그대로다. 400개 스키마에 Flyway를 돌려야 한다.

**③ 단일 스키마 + `academy_id`** — 레거시와 같은 구조. 커넥션·마이그레이션 문제가 없다. 대신 **격리가 코드 책임이 된다.**

③을 골랐다. 이유는 둘이다. 레거시와 데이터 구조가 같아야 이관 중에 PHP와 Java가 같은 테이블을 동시에 읽을 수 있고, 400개 단위의 운영 작업을 혼자 감당할 수 없었다.

문제는 ③의 대가다. `WHERE academy_id = ?`를 **한 군데라도 빼먹으면 전체 학원 데이터가 노출된다.** 레거시에서 이미 그런 쿼리를 몇 개 발견했다. 통계 화면이었고, 다행히 집계값이라 개별 정보는 안 나왔지만 숫자는 전부 섞여 있었다.

## 원인

사람이 매번 조건을 붙이는 구조에서는 **언젠가 빠진다.** 리뷰로 잡자는 건 대책이 아니다. 신규 기능을 만들 때마다 400곳의 데이터 격리가 개발자 한 명의 집중력에 걸린다.

빼먹는 게 불가능하게 만들어야 했다.

## 해결

### 조건을 쿼리가 아니라 세션에 걸기

Hibernate의 `@Filter`로 엔티티 단위 조건을 걸고, 요청마다 세션에 테넌트를 주입했다.

```java
@Entity
@FilterDef(name = "tenant", parameters = @ParamDef(name = "academyId", type = Long.class))
@Filter(name = "tenant", condition = "academy_id = :academyId")
public class Student extends TenantEntity { ... }
```

```java
@Component
@RequiredArgsConstructor
public class TenantFilterAspect {
    private final EntityManager em;

    public void apply(Long academyId) {
        em.unwrap(Session.class)
          .enableFilter("tenant")
          .setParameter("academyId", academyId);
    }
}
```

테넌트 ID는 요청 쪽에서 받지 않았다. **인증 토큰에서 꺼냈다.** 파라미터로 받으면 값을 바꿔 보내는 순간 격리가 무너진다.

### 빼먹은 곳을 테스트로 잡기

`@Filter`는 JPQL·엔티티 조회에만 붙는다. 네이티브 쿼리에는 안 붙는다. 통계 화면처럼 무거운 쿼리는 네이티브로 쓸 수밖에 없었다.

그래서 **테넌트 컬럼이 있는 테이블을 참조하면서 조건이 없는 네이티브 쿼리**를 찾는 테스트를 넣었다.

```java
@Test
void 네이티브_쿼리는_테넌트_조건을_가져야_한다() {
    for (NativeQueryInfo q : scanNativeQueries()) {
        if (q.referencesTenantTable() && !q.hasTenantPredicate()) {
            fail("테넌트 조건 누락: " + q.location());
        }
    }
}
```

완벽한 검사는 아니다. 문자열을 조립해서 만드는 쿼리는 못 잡는다. 하지만 **"리뷰에서 눈으로 본다"보다는 훨씬 촘촘했다.** 이 테스트에서 레거시 이관 대상 쿼리 몇 개가 걸렸다.

### 학원마다 다른 것은 데이터로

학원 400곳은 운영 방식이 제각각이었다. 수업 단위가 월 기준인 곳과 회차 기준인 곳, 출결을 학부모에게 바로 알리는 곳과 주간 요약만 보내는 곳이 섞여 있었다.

이걸 분기문으로 넣기 시작하면 코드가 학원 수만큼 갈라진다. 학원별 설정 테이블을 두고 **동작을 설정값으로 읽게** 했다.

```java
public NotifyPolicy policyOf(Long academyId) {
    return settingRepository.findByAcademyId(academyId)
        .map(AcademySetting::toNotifyPolicy)
        .orElse(NotifyPolicy.DEFAULT);   // 설정이 없으면 기본 정책
}
```

`orElse(DEFAULT)`가 핵심이다. 신규 학원이 들어올 때 설정 행을 안 만들어도 서비스가 돌아야 한다.

## 결과

- 신규 기능에서 테넌트 조건을 코드로 쓸 필요가 없어졌다
- 조건이 누락된 네이티브 쿼리가 빌드 단계에서 걸린다
- 학원별 운영 차이가 코드 분기가 아니라 설정 행으로 들어갔다

## 배운 것

**1. 격리를 사람의 규율에 맡기면 언젠가 깨진다.** "`academy_id` 꼭 넣기"를 문서로 적어두는 것과, 안 넣으면 빌드가 깨지게 만드는 것은 다른 일이다. 반복되는 실수는 교육이 아니라 구조로 막아야 한다.

**2. 완전 격리가 늘 정답은 아니다.** 테넌트별 DB는 보안 측면에서 분명히 낫다. 하지만 400개를 혼자 운영해야 하는 조건에서는 **마이그레이션 실패 하나가 더 큰 사고**다. 선택은 이상적 설계가 아니라 운영 인력까지 포함해서 해야 했다.

**3. 테넌트 ID는 사용자 입력이 아니다.** 요청 파라미터로 받는 순간 그건 더 이상 격리가 아니고 요청자의 선언이다. 인증 주체에서 유도되는 값이어야 한다.

## 남은 한계

`@Filter`는 **읽기에만** 붙는다. 쓰기는 막아주지 않는다. 다른 테넌트의 ID로 저장 요청을 보내면 통과한다. 엔티티 저장 전에 테넌트를 검증하는 리스너를 두긴 했지만, 벌크 업데이트까지 전부 덮지는 못했다. 쓰기 쪽은 여전히 코드 리뷰가 마지막 방어선이다.
