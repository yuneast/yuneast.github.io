---
title: "외부 사이트 DOM 변경에도 안정적인 스크래핑 유지하기"
date: 2026-02-10
categories: [Backend]
tags: [스크래핑, 데이터 파싱, Classic ASP, 안정성]
---

## 문제: 예고 없는 DOM 구조 변경

OddsPortal에서 실시간 배당 데이터를 스크래핑하는 엔진을 구축했다. 그런데 외부 사이트는 예고 없이 DOM 구조를 바꾼다.

**발생한 문제:**
- OddsPortal이 디자인 개편으로 DOM 구조 변경
- 기존 CSS 셀렉터가 작동하지 않아 데이터 수집 실패
- 축구, 농구, 야구 등 종목마다 HTML 구조가 달라서 일괄 대응 불가

**비즈니스 임팩트:**
- 배당 데이터 수집 중단 시 베팅 서비스 운영 불가
- 데이터 유실 시 사용자 신뢰도 하락

## 분석: 왜 자주 깨질까?

**1. 종목별로 데이터 구조가 다름**

```html
<!-- 축구 -->
<div class="soccer-odds">
  <span class="team-home">Manchester United</span>
  <span class="odds">1.85</span>
</div>

<!-- 농구 -->
<table class="basketball-table">
  <tr>
    <td class="team-name">LA Lakers</td>
    <td class="point-spread">+5.5</td>
  </tr>
</table>
```

축구는 div, 농구는 table. CSS 셀렉터를 하나로 통일할 수 없다.

**2. 팀명, 리그명 표기가 제각각**

```
OddsPortal: "Man Utd"
자사 DB: "Manchester United"

OddsPortal: "EPL"
자사 DB: "프리미어 리그"
```

문자열 비교로는 매칭이 안 된다.

**3. 클래스명이 난독화됨**

```html
<!-- 이전 -->
<div class="odds-value">1.85</div>

<!-- 변경 후 -->
<div class="a3f9d2">1.85</div>
```

프론트엔드 빌드 도구가 클래스명을 난독화하면 셀렉터가 무용지물이 된다.

## 해결: 다층 예외 처리 + 매핑 테이블

### 1. 여러 셀렉터 패턴 시도

하나의 셀렉터가 실패하면 다음 패턴으로 fallback한다.

```vbscript
Function GetOddsValue(html)
    Dim patterns(2)
    patterns(0) = "div.odds-value"
    patterns(1) = "span.odd"
    patterns(2) = "td[class*='odds']"

    For Each pattern In patterns
        value = ExtractBySelector(html, pattern)
        If value <> "" Then
            GetOddsValue = value
            Exit Function
        End If
    Next

    ' 모든 패턴 실패 시 로그 기록, 기본값 반환
    LogError "모든 셀렉터 패턴 실패"
    GetOddsValue = "N/A"
End Function
```

**핵심:** 하나가 실패해도 다음 패턴으로 시도해서 데이터 수집 지속.

### 2. 종목별 파서 분리

축구, 농구, 야구 각각 별도 파서를 구현했다.

```vbscript
Function ParseData(html, sport)
    Select Case sport
        Case "soccer"
            Set parser = New SoccerParser
        Case "basketball"
            Set parser = New BasketballParser
        Case "baseball"
            Set parser = New BaseballParser
        Else
            LogError "알 수 없는 종목: " & sport
            Exit Function
    End Select

    ParseData = parser.Parse(html)
End Function
```

**핵심:** 종목별로 다른 HTML 구조에 대응.

### 3. 팀명, 리그명 매핑 테이블

외부 사이트의 팀명과 자사 DB의 팀명을 매핑하는 테이블을 만들었다.

```sql
CREATE TABLE team_mapping (
    external_name VARCHAR(100),  -- "Man Utd"
    internal_name VARCHAR(100),  -- "Manchester United"
    sport VARCHAR(20)
)
```

```vbscript
Function NormalizeTeamName(externalName, sport)
    sql = "SELECT internal_name FROM team_mapping " & _
          "WHERE external_name = '" & externalName & "' " & _
          "AND sport = '" & sport & "'"

    rs = ExecuteQuery(sql)
    If Not rs.EOF Then
        NormalizeTeamName = rs("internal_name")
    Else
        ' 매핑 없으면 원본 그대로 + 로그 기록
        LogWarning "매핑 없음: " & externalName
        NormalizeTeamName = externalName
    End If
End Function
```

**핵심:** 외부 표기와 내부 표기를 분리해서 데이터 정합성 확보.

### 4. 실패 시 이전 데이터 유지

데이터 수집 실패 시 DB의 이전 데이터를 그대로 유지했다.

```vbscript
Function UpdateOdds(teamId, newOdds)
    If newOdds = "" Or newOdds = "N/A" Then
        ' 새 데이터 없으면 업데이트 안 함 (이전 값 유지)
        LogWarning "배당 데이터 수집 실패, 이전 값 유지: teamId=" & teamId
        Exit Function
    End If

    sql = "UPDATE odds SET value = " & newOdds & ", " & _
          "updated_at = GETDATE() " & _
          "WHERE team_id = " & teamId
    ExecuteQuery(sql)
End Function
```

**핵심:** 일부 데이터 수집 실패해도 서비스 전체가 멈추지 않음.

## 결과: 데이터 유실률 1% 미만

**측정:**
- 일 평균 수집 시도: 50,000건
- 수집 실패: 300건 이하
- 유실률: 300 / 50,000 = 0.6%

**비즈니스 효과:**
- DOM 구조 변경 시에도 서비스 중단 없음
- 매핑 테이블 업데이트만으로 새 팀/리그 추가 가능
- 장애 대응 공수 90% 절감 (매번 코드 수정 불필요)

## 배운 점

**1. 외부 의존성은 언제든 깨진다**

외부 사이트 구조는 언제든 바뀐다는 전제로 개발해야 한다.

**2. 다층 방어가 답**

하나의 패턴이 아니라 여러 fallback 패턴을 준비해야 안정성이 확보된다.

**3. 데이터 정규화는 필수**

외부 표기와 내부 표기를 분리하지 않으면 매번 코드를 수정해야 한다.

**4. 실패를 전체 장애로 번지지 않게**

일부 데이터 수집 실패는 불가피하다. 전체 서비스가 멈추지 않도록 설계하는 게 중요하다.
