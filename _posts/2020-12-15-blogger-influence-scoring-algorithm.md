---
title: "블로거 영향력을 점수로 측정하기 - 크롤링부터 가중치 계산까지"
date: 2020-12-15
categories: [backend]
tags: [algorithm, crawling, python, ranking, scoring]
---

"이 블로거의 영향력이 얼마나 되나요?" 인플루언서 매칭 플랫폼을 만들면서 광고주가 가장 많이 하던 질문이다. 팔로워 수나 방문자 수는 조작 가능하다. 실제 영향력을 측정하는 알고리즘을 설계한 과정을 정리한다.

## 문제

콘디(인플루언서 중개 플랫폼)에서 광고주는 블로거를 선택해서 광고를 의뢰한다. 어떤 블로거를 선택할지 판단 기준이 필요했다.

**기존 지표의 문제:**
- 팔로워 수: 조작 가능 (팔로워 구매)
- 방문자 수: 확인 불가 (블로거가 공개 안 하면 모름)
- 구독자 수: 휴면 계정 포함

실제 영향력을 측정할 수 있는 지표가 필요했다.

## 영향력의 정의

블로거의 영향력은 "검색 노출력"이라고 정의했다.

**핵심 가설:**
- 검색 상위 노출 = 많은 사람이 본다
- 여러 키워드에서 상위 노출 = 영향력 크다
- 1위보다 10위는 영향력이 낮다 (순위별 차등 점수)

## 알고리즘 설계

### 1단계: 블로그 크롤링

블로그 URL로 최근 게시글 30개를 크롤링한다.

```python
import requests
from bs4 import BeautifulSoup

def crawl_blog_posts(blog_url: str, limit: int = 30) -> list[dict]:
    """네이버 블로그 최근 게시글 크롤링"""
    posts = []

    # 블로그 RSS 피드 사용
    rss_url = f"{blog_url}/rss"
    response = requests.get(rss_url)
    soup = BeautifulSoup(response.content, 'xml')

    items = soup.find_all('item', limit=limit)

    for item in items:
        posts.append({
            'title': item.find('title').text,
            'link': item.find('link').text,
            'description': item.find('description').text,
            'pub_date': item.find('pubDate').text
        })

    return posts
```

### 2단계: 키워드 추출

각 게시글에서 핵심 키워드를 추출한다. 형태소 분석으로 명사만 추출하고, 불용어를 제거한다.

```python
from konlpy.tag import Okt

okt = Okt()

STOPWORDS = ['오늘', '어제', '정말', '너무', '정도', '제가', '저는', ...]

def extract_keywords(text: str, top_n: int = 5) -> list[str]:
    """텍스트에서 핵심 키워드 추출"""
    # 명사 추출
    nouns = okt.nouns(text)

    # 2글자 이상, 불용어 제거
    keywords = [
        word for word in nouns
        if len(word) >= 2 and word not in STOPWORDS
    ]

    # 빈도수 계산
    from collections import Counter
    counter = Counter(keywords)

    # 상위 N개 반환
    return [word for word, _ in counter.most_common(top_n)]
```

### 3단계: 네이버 검색 순위 확인

추출한 키워드로 네이버 검색했을 때 해당 게시글이 몇 위에 노출되는지 확인한다.

```python
def check_search_ranking(keyword: str, post_url: str) -> int:
    """네이버 검색 결과에서 게시글 순위 확인"""
    search_url = f"https://search.naver.com/search.naver?query={keyword}"
    response = requests.get(search_url, headers={
        'User-Agent': 'Mozilla/5.0'
    })
    soup = BeautifulSoup(response.text, 'html.parser')

    # 블로그 검색 결과 추출
    blog_results = soup.select('.blog_link')

    for rank, result in enumerate(blog_results[:30], start=1):
        if post_url in result['href']:
            return rank

    return None  # 30위 밖
```

### 4단계: 순위별 가중치 적용

검색 순위에 따라 차등 점수를 부여한다.

```python
def calculate_ranking_score(rank: int) -> float:
    """검색 순위에 따른 점수 계산"""
    if rank is None:
        return 0

    # 1~3위: 높은 가중치
    if rank <= 3:
        return 10 - (rank - 1) * 2  # 10, 8, 6

    # 4~10위: 중간 가중치
    elif rank <= 10:
        return 6 - (rank - 3) * 0.5  # 5.5, 5, 4.5, ..., 2

    # 11~30위: 낮은 가중치
    elif rank <= 30:
        return 2 - (rank - 10) * 0.05  # 1.95, 1.9, ..., 0.05

    else:
        return 0
```

**가중치 설계 근거:**
- 1위와 2위는 클릭률 차이가 크다 (1위 가중치 2배)
- 11위 이후는 2페이지라 노출이 급감 (낮은 점수)
- 30위 이후는 실질적 영향력 없음 (점수 없음)

### 5단계: 블로거 종합 점수 산출

모든 게시글의 키워드별 점수를 합산한다.

```python
def calculate_blogger_score(blog_url: str) -> float:
    """블로거 영향력 점수 계산"""
    posts = crawl_blog_posts(blog_url, limit=30)

    total_score = 0
    keyword_count = 0

    for post in posts:
        # 게시글에서 키워드 추출
        text = post['title'] + ' ' + post['description']
        keywords = extract_keywords(text, top_n=5)

        for keyword in keywords:
            # 검색 순위 확인
            rank = check_search_ranking(keyword, post['link'])

            # 점수 계산
            score = calculate_ranking_score(rank)
            total_score += score

            if rank:
                keyword_count += 1

    # 평균 점수 (노출되는 키워드 수로 나눔)
    if keyword_count == 0:
        return 0

    return total_score / keyword_count
```

## 실제 데이터 검증

초기 알고리즘을 100명의 블로거에게 적용해봤다.

**문제 발견:**

1. **최신 게시글 위주로 점수가 높다**
   - 과거 인기 게시글은 반영 안 됨
   - 해결: 최근 3개월 게시글로 확장

2. **키워드 중복 문제**
   - "맛집", "강남 맛집", "강남역 맛집" 같은 유사 키워드
   - 해결: 키워드 정규화 (형태소 기준 유사도 제거)

3. **광고 게시글 필터링 필요**
   - 광고 게시글이 검색 상위 노출되는 경우
   - 해결: 제목에 "협찬", "AD" 포함 시 제외

## 개선 버전

```python
def calculate_blogger_score_v2(blog_url: str) -> dict:
    """블로거 영향력 점수 계산 (개선)"""
    posts = crawl_blog_posts(blog_url, limit=50)  # 50개로 확장

    # 3개월 이내 게시글만 필터링
    recent_posts = filter_recent_posts(posts, months=3)

    # 광고 게시글 제외
    non_ad_posts = [p for p in recent_posts
                    if not is_ad_post(p['title'])]

    keyword_scores = {}

    for post in non_ad_posts:
        text = post['title'] + ' ' + post['description']
        keywords = extract_keywords(text, top_n=5)

        # 키워드 정규화 (유사 키워드 통합)
        keywords = normalize_keywords(keywords)

        for keyword in keywords:
            if keyword in keyword_scores:
                continue  # 이미 체크한 키워드 스킵

            rank = check_search_ranking(keyword, post['link'])
            score = calculate_ranking_score(rank)

            if rank:
                keyword_scores[keyword] = {
                    'rank': rank,
                    'score': score
                }

    # 총점, 평균, 상위 노출 키워드 수 반환
    total_score = sum(k['score'] for k in keyword_scores.values())
    avg_score = total_score / len(keyword_scores) if keyword_scores else 0

    return {
        'total_score': total_score,
        'avg_score': avg_score,
        'top_keywords_count': len([k for k in keyword_scores.values()
                                    if k['rank'] <= 10]),
        'keywords': keyword_scores
    }
```

## 결과

블로거 점수 분포:

| 등급 | 점수 범위 | 블로거 수 | 특징 |
|------|-----------|-----------|------|
| S | 8점 이상 | 5% | 여러 키워드 1~3위 노출 |
| A | 6~8점 | 15% | 여러 키워드 상위 10위 이내 |
| B | 4~6점 | 30% | 일부 키워드 상위 노출 |
| C | 2~4점 | 35% | 소수 키워드만 노출 |
| D | 2점 미만 | 15% | 검색 노출 거의 없음 |

**광고주 반응:**
- "이제 객관적으로 블로거를 비교할 수 있어요"
- "점수가 높은 블로거가 실제로 효과가 좋았어요"

## 한계

1. **크롤링 비용이 높다.**
   - 블로거 1명당 평균 5초 소요
   - 1,000명 분석 시 1시간 이상

2. **네이버 검색만 반영한다.**
   - 구글, 다음 검색 미반영
   - 인스타, 유튜브는 분석 불가

3. **캐시로 인해 이전 데이터를 사용한다.**
   - 1시간 캐싱으로 크롤링 비용 절감
   - 검색 순위는 계속 변하는데 캐시된 점수는 최대 1시간 전 데이터

4. **블로거가 직접 공개한 데이터가 아니다.**
   - 크롤링은 블로거 동의 없이 진행
   - 윤리적 문제 논란 가능

## 개선 방향

1. **캐싱 도입** (이미 적용, Celery 비동기 포스트 참고)
   - Redis에 분석 결과 1시간 캐싱
   - 중복 분석 방지

2. **증분 분석**
   - 전체 재분석 대신 신규 게시글만 추가 분석
   - DB에 게시글별 점수 저장

3. **블로거 자체 공개 데이터 활용**
   - 네이버 블로그 통계 API (방문자 수, 체류 시간)
   - 블로거 동의하에 데이터 수집

## 배운 점

1. **완벽한 지표는 없다.** 검색 순위도 하나의 근사치일 뿐, 영향력의 전부가 아니다. 여러 지표를 조합해서 판단해야 한다.

2. **알고리즘 검증이 중요하다.** 100명 데이터로 검증했더니 문제가 보였다. 가설을 세우고 검증하고 개선하는 과정이 필수다.

3. **가중치 설계가 핵심이다.** 1위와 2위의 차이를 얼마로 둘지, 11위 이후를 어떻게 처리할지가 점수 분포를 결정한다.

4. **실행 비용을 고려해야 한다.** 정확도를 높이려다 크롤링 비용이 커지면 서비스 운영이 어렵다. 정확도와 비용의 트레이드오프를 찾아야 한다.

## 마무리

블로거 영향력 점수는 "검색 노출력"으로 측정할 수 있다. 크롤링 → 키워드 추출 → 검색 순위 확인 → 가중치 적용 → 종합 점수 산출. 이 과정을 자동화하면 객관적인 지표를 만들 수 있다. 완벽하진 않지만, 광고주가 블로거를 선택하는 데 도움이 됐다. 알고리즘 설계에서 가장 중요한 건 "무엇을 측정할 것인가"를 명확히 정의하는 것이다.
