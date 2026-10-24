---
title: "대진표를 엑셀로 짜면 세 시간 - 급수와 소속을 동시에 만족시키는 조 편성"
date: 2026-10-24
categories: [backend]
tags: [algorithm, scheduling, constraint, backtracking, java]
---

대회 전날 밤에 임원들이 엑셀을 붙잡고 있었다. 참가 신청서를 보고 복식 조를 짜고, 대진표를 그리고, 같은 동호회끼리 1회전에서 만나면 지웠다가 다시 그렸다. 매번 세 시간쯤 걸린다고 했다.

## 배경

배드민턴협회 회원관리 시스템에 대회 기능을 붙이고 있다. 제일 손이 많이 가는 게 **대진표 생성**이었다.

규칙이 몇 겹으로 있었다.

- 참가자는 **급수**(A~D조)가 있고, 같은 급수끼리 경기한다
- 혼합복식은 남녀 한 명씩 묶는다
- 랜덤 복식은 서로 모르는 사람끼리 묶는 게 취지여서, **같은 동호회는 한 팀이 되면 안 된다**
- 1회전에서 같은 동호회끼리 붙는 것도 피한다
- 참가 팀 수가 2의 거듭제곱이 아니면 **부전승**을 줘야 한다

사람이 하면 세 시간, 그리고 매번 누락이 생겼다.

## 문제

처음 접근은 단순했다. 급수별로 나눠서 셔플하고 둘씩 묶었다. 대부분 잘 됐는데 **마지막에 꼭 막혔다.**

```
D조 참가자 14명
  청주클럽 6명 / 상당클럽 5명 / 흥덕클럽 3명

앞에서부터 묶다 보면 끝에 청주클럽 2명만 남는다
→ 같은 동호회 금지 위반, 다시 짤 수밖에 없음
```

셔플하고 다시 돌리면 또 다른 데서 막혔다. 참가자 구성에 따라 **몇 번을 돌려도 안 되는 경우**가 있었다.

## 원인

그리디로 앞에서부터 묶으면, 뒤에 남는 사람들의 소속을 고려하지 못한다. 가장 인원이 많은 동호회를 나중으로 미루면 반드시 그 동호회끼리 남는다.

그리고 **애초에 불가능한 입력**도 있었다. 14명 중 8명이 한 동호회면 어떻게 묶어도 그 동호회끼리 한 팀이 생긴다. 알고리즘이 못 푸는 게 아니라 조건이 모순이다. 이 두 경우를 구분하지 못한 게 진짜 문제였다.

## 해결

### 많은 쪽을 먼저 쓴다

인원이 많은 동호회부터 배치하면 뒤에 쏠리지 않는다.

```java
private List<Team> pair(List<Member> members) {
    // 동호회별로 묶고, 인원이 많은 순으로 정렬
    Deque<List<Member>> groups = members.stream()
        .collect(Collectors.groupingBy(Member::getClubId))
        .values().stream()
        .sorted(Comparator.comparingInt(List<Member>::size).reversed())
        .collect(Collectors.toCollection(ArrayDeque::new));

    List<Team> teams = new ArrayList<>();
    while (groups.size() >= 2) {
        List<Member> a = groups.pollFirst();   // 가장 많은 쪽
        List<Member> b = groups.pollLast();    // 가장 적은 쪽
        teams.add(Team.of(a.remove(a.size() - 1), b.remove(b.size() - 1)));
        if (!a.isEmpty()) reinsertBySize(groups, a);
        if (!b.isEmpty()) reinsertBySize(groups, b);
    }
    if (!groups.isEmpty()) throw new PairingImpossibleException(groups.peek());
    return teams;
}
```

제일 많은 동호회와 제일 적은 동호회를 번갈아 묶는다. 큰 쪽을 계속 소비하니까 끝에 한 동호회만 남는 경우가 줄어든다.

### 불가능한 입력은 먼저 판정한다

위 방식으로도 안 되는 경우가 있다. 그걸 **시도해 보고 실패하는 대신 미리 계산**했다.

```java
// 한 동호회 인원이 전체의 절반을 넘으면 어떤 방법으로도 불가능
int max = countByClub.values().stream().max(Integer::compare).orElse(0);
if (max * 2 > total) {
    return Infeasible.of(dominantClub, max, total);
}
```

절반을 넘는 동호회가 있으면 비둘기집 원리로 반드시 같은 소속 팀이 생긴다. 이걸 먼저 알려주면 운영자가 선택할 수 있다. 급수를 합치거나, 해당 동호회 신청을 조정하거나, 제약을 한 번 풀거나.

재시도로 뭉개는 것과, **왜 안 되는지 말해주는 것**은 다르다. 전자는 운영자가 "프로그램이 이상하다"고 생각하고 다시 엑셀로 돌아간다.

### 부전승은 상위 시드에

16강 자리에 11팀이 신청하면 5팀이 부전승을 받는다. 누구에게 줄지가 문제였다.

랜덤으로 주면 항의가 들어온다. 전년도 성적순으로 상위 시드에 배정하고, **배정 근거를 대진표에 같이 표시**했다.

```java
List<Team> seeded = teams.stream()
    .sorted(Comparator.comparing(Team::getSeedRank))
    .toList();

int byes = nextPowerOfTwo(seeded.size()) - seeded.size();
for (int i = 0; i < byes; i++) {
    bracket.placeBye(seeded.get(i));   // 상위 시드부터
}
```

기술적으로 어려운 부분은 아니다. 다만 **규칙이 보이지 않으면 결과를 의심받는다**는 게 이 기능의 본질이었다.

### 생성 결과를 검증한다

만든 대진표가 규칙을 지켰는지 따로 확인했다.

```java
public List<Violation> validate(Bracket b) {
    List<Violation> v = new ArrayList<>();
    for (Team t : b.getTeams()) {
        if (t.sameClub())      v.add(Violation.sameClubTeam(t));
        if (!t.sameGrade())    v.add(Violation.gradeMismatch(t));
    }
    for (Match m : b.firstRoundMatches()) {
        if (m.sharesClub())    v.add(Violation.sameClubFirstRound(m));
    }
    return v;
}
```

자동 생성한 걸 그대로 믿지 않는 장치다. 1회전 동호회 중복은 완전히 피할 수 없는 경우가 있어서 **차단이 아니라 경고**로 띄웠다. 운영자가 보고 수동으로 바꿀 수 있게 했다.

## 결과

- 대진표 생성이 수 초로 끝났다 (기존: 엑셀 수작업 3시간 내외, 임원 진술 기준)
- 같은 동호회 한 팀 편성이 생성 단계에서 차단됐다
- 불가능한 참가자 구성을 대회 전날이 아니라 **신청 마감 직후**에 알 수 있게 됐다
- 부전승 배정 근거가 대진표에 표시되어 문의가 줄었다

## 배운 것

**1. "안 됩니다"와 "이래서 안 됩니다"의 차이가 크다.** 제약 조건이 모순인 입력에서 그냥 실패를 반환했을 때는 운영자가 프로그램을 안 믿었다. 어느 동호회가 몇 명이라서 불가능한지 보여주니까 그 자리에서 조정하고 다시 돌렸다. 에러 메시지는 개발자를 위한 게 아니었다.

**2. 제약 조건은 전부 같은 등급이 아니다.** 같은 급수끼리 경기는 절대 규칙이고, 1회전 동호회 회피는 선호였다. 처음엔 둘 다 똑같이 강제해서 답이 안 나오는 경우가 많았다. 어느 규칙을 포기할 수 있는지는 코드가 정할 수 없고 운영자에게 물어봐야 했다.

**3. 자동 생성 기능은 검증 기능이 반이다.** 사람이 짜면 눈으로 확인한다. 자동 생성은 결과를 아무도 안 본다. 생성 로직을 만들 때 검증 로직을 같이 만들지 않으면, 틀린 대진표가 조용히 대회장까지 간다.

## 남은 한계

경기장 코트 수와 시간대를 고려한 **경기 일정 배치**는 아직 수동이다. 대진표는 누가 누구와 붙는지만 정하고, 어느 코트 몇 시에 할지는 임원이 짠다. 코트 3면에 선수 대기 시간까지 넣으면 이건 조 편성보다 훨씬 무거운 문제다. 다음 대회까지 붙이는 게 목표다.
