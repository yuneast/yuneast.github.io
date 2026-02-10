---
title: "글로벌 락으로 서버가 멈췄다 - Celery와 Redis로 동기 처리 병목 해결하기"
date: 2021-03-20
categories: [backend]
tags: [celery, redis, python, django, async, performance]
---

인플루언서 매칭 플랫폼을 운영하면서 블로거 지수 분석 기능을 만들었다. 블로그 데이터를 크롤링하고 여러 지표를 계산해서 점수를 매기는 기능이다. 문제는 이 분석이 요청할 때마다 서버 전체를 멈추게 만들었다는 것이다.

## 배경

콘디(인플루언서 중개 플랫폼)에서는 광고주가 블로거를 선택할 때 "블로거 지수"를 참고한다. 블로거 지수는 블로그의 일 방문자 수, 포스팅 빈도, 댓글 수, 이웃 수 등을 종합해서 산출하는 점수다.

### 분석 과정

1. 블로그 URL로 최근 게시글 30개 크롤링
2. 각 게시글의 조회수, 댓글 수, 좋아요 수 수집
3. 블로그 전체 통계 (일 방문자 수, 이웃 수) 수집
4. 지표별 가중치 적용해서 종합 점수 산출

하나의 블로거를 분석하는 데 약 5초가 걸렸다. 크롤링 대기 시간이 대부분이다.

## 문제 발생

Django에서 분석 로직을 동기적으로 처리하고 있었다:

```python
# views.py
def analyze_blogger(request, blogger_id):
    blogger = Blogger.objects.get(id=blogger_id)

    # 5초간 서버가 블로킹됨
    score = BloggerAnalyzer.analyze(blogger.blog_url)

    blogger.score = score
    blogger.save()

    return JsonResponse({'score': score})
```

Django는 기본적으로 동기 WSGI 서버로 동작한다. Gunicorn 워커를 4개 띄우고 있었는데, 4명이 동시에 분석을 요청하면 모든 워커가 점유돼서 다른 요청을 처리하지 못한다.

더 심각한 문제는 **분석 중에 DB 커넥션을 물고 있다**는 점이었다. Django ORM은 요청 시작 시 커넥션을 가져오고 요청이 끝나야 반환한다. 5초간 블로킹되면 커넥션도 5초간 잠긴다.

```
[문제 상황]
워커1: 블로거A 분석 중 (5초 블로킹) → DB 커넥션 점유
워커2: 블로거B 분석 중 (5초 블로킹) → DB 커넥션 점유
워커3: 블로거C 분석 중 (5초 블로킹) → DB 커넥션 점유
워커4: 블로거D 분석 중 (5초 블로킹) → DB 커넥션 점유
→ 신규 요청: 502 Bad Gateway
```

실제로 광고주 여러 명이 동시에 블로거 분석을 요청하면 서비스 전체가 응답 불가 상태가 됐다.

## 해결: Celery + Redis

무거운 분석 작업을 별도 워커 프로세스로 분리했다. Django는 작업을 큐에 넣고 즉시 응답하고, 실제 분석은 Celery 워커가 처리한다.

### 아키텍처 변경

```
Before:
  Client → Django → 크롤링+분석 (5초 블로킹) → 응답

After:
  Client → Django → Redis 큐에 작업 등록 → 즉시 응답 (task_id 반환)
                      ↓
              Celery Worker → 크롤링+분석 → 결과를 Redis에 저장
                      ↓
  Client → Django → Redis에서 결과 조회 → 응답
```

### Celery 태스크 정의

```python
# tasks.py
from celery import shared_task
from django.core.cache import cache

@shared_task(bind=True, max_retries=2, default_retry_delay=10)
def analyze_blogger_task(self, blogger_id: int):
    try:
        blogger = Blogger.objects.get(id=blogger_id)
        score = BloggerAnalyzer.analyze(blogger.blog_url)

        blogger.score = score
        blogger.analyzed_at = timezone.now()
        blogger.save()

        # 결과 캐싱 (1시간)
        cache_key = f"blogger_score:{blogger_id}"
        cache.set(cache_key, score, timeout=3600)

        return {'blogger_id': blogger_id, 'score': score}

    except Exception as exc:
        raise self.retry(exc=exc)
```

### 뷰 수정

```python
# views.py
def analyze_blogger(request, blogger_id):
    # 캐시에 결과가 있으면 즉시 반환
    cache_key = f"blogger_score:{blogger_id}"
    cached_score = cache.get(cache_key)

    if cached_score is not None:
        return JsonResponse({'score': cached_score, 'status': 'completed'})

    # 비동기 태스크 실행
    task = analyze_blogger_task.delay(blogger_id)

    return JsonResponse({
        'task_id': task.id,
        'status': 'processing',
        'message': '분석이 시작되었습니다. 잠시 후 결과를 확인해주세요.'
    })


def check_analysis_result(request, task_id):
    task = AsyncResult(task_id)

    if task.ready():
        result = task.get()
        return JsonResponse({
            'status': 'completed',
            'score': result['score']
        })

    return JsonResponse({'status': 'processing'})
```

### Redis 캐싱 추가

같은 블로거를 여러 광고주가 조회하는 경우가 많았다. 분석 결과를 Redis에 캐싱해서 중복 크롤링을 방지했다.

```python
# analyzer.py
class BloggerAnalyzer:
    @staticmethod
    def analyze(blog_url: str) -> float:
        cache_key = f"blog_analysis:{hashlib.md5(blog_url.encode()).hexdigest()}"

        cached = cache.get(cache_key)
        if cached:
            return cached

        # 크롤링 + 분석 (5초 소요)
        posts = BlogCrawler.crawl_recent_posts(blog_url, count=30)
        stats = BlogCrawler.get_blog_stats(blog_url)

        score = ScoreCalculator.calculate(posts, stats)

        cache.set(cache_key, score, timeout=3600)
        return score
```

## Before / After

| 항목 | Before | After |
|------|--------|-------|
| 분석 요청 응답 | 5초 (블로킹) | 즉시 (비동기) |
| 동시 분석 | 워커 수 제한 (4건) | Celery 워커로 독립 처리 |
| 서버 멈춤 | 동시 분석 시 502 발생 | Django는 영향 없음 |
| 중복 크롤링 | 매번 크롤링 | Redis 캐싱으로 재사용 |

API 응답 시간이 5초에서 즉시 응답으로 변했고, 캐시 히트 시 크롤링 없이 바로 결과를 반환한다.

## Celery 설정 팁

```python
# celery.py
from celery import Celery

app = Celery('condi')
app.config_from_object('django.conf:settings', namespace='CELERY')

# 동시 실행 워커 수 제한 (크롤링 대상 서버 부하 고려)
app.conf.worker_concurrency = 4

# 작업 시간 제한 (크롤링이 무한 대기하는 걸 방지)
app.conf.task_time_limit = 30

# 실패 시 결과 보관 (디버깅용)
app.conf.result_expires = 3600
```

`worker_concurrency`를 너무 높게 설정하면 크롤링 대상 서버에 과부하를 줄 수 있다. 적절한 수준으로 제한했다.

## 배운 점

1. **동기 웹 서버에서 무거운 작업을 직접 처리하면 안 된다.** 워커 수가 한정돼있기 때문에 오래 걸리는 작업이 워커를 점유하면 서비스 전체가 멈춘다.

2. **비동기 처리는 UX 변경이 동반된다.** "요청하면 바로 결과가 나오는" 동기 방식에서 "요청하면 작업 ID를 받고, 나중에 결과를 조회하는" 비동기 방식으로 프론트엔드도 변경해야 한다. 광고주에게 "분석 중입니다" 상태를 보여주는 UI를 추가했다.

3. **캐싱은 비동기와 별개로 효과가 크다.** Celery를 도입하지 않았더라도 Redis 캐싱만으로 중복 크롤링을 줄일 수 있었다. 두 가지를 함께 적용하면 시너지가 크다.
