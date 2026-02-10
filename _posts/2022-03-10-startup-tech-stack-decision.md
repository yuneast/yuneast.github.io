---
title: "스타트업 초기 기술 스택 선정 - Python/FastAPI를 선택한 이유"
date: 2022-03-10
categories: [backend]
tags: [fastapi, python, tech-stack, startup, architecture]
---

공동주택 관리 서비스를 만들 때, 팀 기술 스택을 Python/FastAPI로 결정했다. Node.js, Django, Spring Boot 등 여러 선택지가 있었는데 왜 FastAPI였을까? 기술 스택 선정 과정과 6개월 사용 후 회고를 정리한다.

## 상황

2021년 9월, 기존 공동주택 관리 서비스를 재개발하기로 결정했다.

**배경:**
- 외주로 받은 PHP 기반 서비스 운영 중
- 트래픽 증가로 성능 문제 발생
- 레거시 코드가 너무 낡아서 튜닝보다 재개발이 빠를 것 같았음

**팀 구성:**
- 백엔드 개발자 1명 (나)
- 프론트엔드 개발자 1명 (React 경험 있음)

**서비스 요구사항:**
- 관리비 대납, 주차 관리, 투표 기능
- 20개 단지, 단지당 수백~수천 세대
- 관리자 + 입주민 웹/앱
- 빠른 재개발 (3개월 내 기존 기능 이전)

**내가 할 수 있는 것:**
- Python (Django, Flask 경험)
- Java (Spring Boot 경험)
- Node.js (Express 경험)

## 후보 기술 스택

### 1. Django (Python)

**장점:**
- ORM, Admin, Auth 등 기본 제공
- 빠른 개발 속도
- 내가 가장 익숙함

**단점:**
- 무겁다 (우리 서비스엔 과한 기능들)
- API 개발에는 DRF 추가 필요
- 동기 처리 기반 (비동기 지원 약함)

### 2. Spring Boot (Java)

**장점:**
- 안정성, 성능
- 대규모 서비스 레퍼런스 많음
- 타입 안전성

**단점:**
- 러닝 커브 높음 (향후 팀 확장 시 온보딩 어려움)
- 개발 속도 느림 (보일러플레이트 많음)
- 빠른 재개발에 부적합

### 3. Node.js + Express

**장점:**
- 비동기 I/O
- 프론트-백엔드 언어 통일 (JavaScript)
- 빠른 프로토타이핑

**단점:**
- 타입 안정성 약함 (TypeScript 도입 필요)
- 체계 없음 (자유도가 너무 높음)
- 내 경험 부족

### 4. FastAPI (Python)

**장점:**
- Python의 간결함 + 타입 힌트
- 자동 문서화 (Swagger)
- 비동기 지원 (async/await)
- 빠른 개발 속도
- 러닝 커브 낮음 (Python 개발자면 하루 만에 시작 가능)

**단점:**
- 상대적으로 신생 프레임워크 (2018년 출시)
- Django만큼 풍부한 생태계 아님
- 대규모 서비스 레퍼런스 적음

## 선택 기준

재개발에서 중요한 건 **속도**다. 3개월 안에 기존 기능을 모두 이전해야 한다.

**우선순위:**
1. 빠른 개발 속도 (가장 중요)
2. 유지보수성 (PHP 레거시 반복 안 함)
3. 확장성 (트래픽 증가 대비)
4. 향후 팀 확장 대비 (온보딩 용이성)

## FastAPI 선택 이유

### 1. 빠른 개발 속도

FastAPI는 코드가 간결하다.

**Django DRF:**
```python
# serializers.py
class StudentSerializer(serializers.ModelSerializer):
    class Meta:
        model = Student
        fields = '__all__'

# views.py
class StudentViewSet(viewsets.ModelViewSet):
    queryset = Student.objects.all()
    serializer_class = StudentSerializer

# urls.py
router = DefaultRouter()
router.register(r'students', StudentViewSet)
```

**FastAPI:**
```python
from typing import List
from fastapi import FastAPI, Depends
from sqlalchemy.orm import Session

app = FastAPI()

@app.get("/students", response_model=List[StudentResponse])
def get_students(db: Session = Depends(get_db)):
    return db.query(Student).all()
```

3개 파일이 1개로 줄었다. 보일러플레이트가 적다.

### 2. 타입 힌트 + 자동 검증

FastAPI는 Python 타입 힌트를 런타임 검증에 사용한다.

```python
from pydantic import BaseModel, Field

class StudentCreate(BaseModel):
    name: str = Field(..., min_length=2, max_length=50)
    age: int = Field(..., ge=1, le=150)
    phone: str = Field(..., regex=r'^\d{3}-\d{4}-\d{4}$')

@app.post("/students")
def create_student(student: StudentCreate):
    # student.name이 2~50자인지, age가 1~150인지 자동 검증됨
    return {"message": "학생 생성 완료"}
```

유효성 검증 코드를 따로 안 써도 된다.

### 3. 자동 문서화

코드만 작성하면 Swagger 문서가 자동 생성된다. 프론트 개발자와 협업할 때 API 스펙을 따로 설명할 필요 없다.

```python
@app.get("/students/{student_id}", response_model=StudentResponse)
def get_student(student_id: int):
    """학생 단건 조회"""
    return db.query(Student).filter(Student.id == student_id).first()
```

`http://localhost:8000/docs` 접속하면 Swagger UI가 뜬다.

### 4. 비동기 지원

외부 API 호출이나 크롤링 같은 I/O 작업이 많을 때 비동기 처리가 유리하다.

```python
import httpx

@app.get("/weather")
async def get_weather():
    async with httpx.AsyncClient() as client:
        response = await client.get("https://api.weather.com/...")
        return response.json()
```

Django는 비동기 지원이 약하다. FastAPI는 기본이 비동기다.

### 5. 향후 팀 확장 대비

FastAPI는 Python만 알면 시작할 수 있다. 팀이 커지면 주니어 개발자가 들어올 텐데, 온보딩이 쉬울 거라고 판단했다.

실제로 2개월 후 주니어 백엔드 개발자 1명이 합류했을 때, 1주일 만에 기능 개발을 시작했다. Django DRF보다 배우기 쉬웠다.

## 6개월 사용 후 회고

### 장점이 맞았던 것

1. **개발 속도가 빨랐다.**
   - 3개월 만에 PHP 기존 기능 모두 이전
   - 관리비 대납, 주차 관리, 투표 기능 재구현 완료

2. **주니어 온보딩이 쉬웠다.**
   - 2개월 후 합류한 주니어 백엔드 개발자가 1주일 만에 기능 개발 시작
   - Python만 알면 FastAPI 이해 가능

3. **문서화 자동화가 큰 도움이 됐다.**
   - 프론트 개발자가 `/docs`만 보고 API 이해
   - 카톡으로 스펙 설명하는 시간 절약

4. **타입 힌트가 버그를 줄였다.**
   - IDE 자동완성 지원
   - 런타임 검증으로 잘못된 요청 조기 차단

### 예상 못 한 단점

1. **ORM 선택지가 제한적이다.**
   - FastAPI는 ORM을 제공하지 않는다
   - SQLAlchemy를 직접 설정해야 함
   - Django ORM만큼 직관적이지 않음

2. **Admin 패널이 없다.**
   - Django Admin 같은 관리자 페이지가 없다
   - 별도 개발 필요 (React Admin 사용)

3. **배포 설정이 복잡하다.**
   - Uvicorn, Gunicorn 조합 직접 설정
   - Django처럼 `python manage.py runserver`로 끝나지 않음

4. **레퍼런스가 적다.**
   - 특정 에러 해결법을 검색하면 Django는 많은데 FastAPI는 적음
   - Stack Overflow 답변 수가 적다

## 다시 선택한다면?

**Yes.** FastAPI를 다시 선택할 것이다.

이유:
- 빠른 재개발 속도가 가장 중요했다
- 단점(ORM, Admin)은 해결 가능했다
- PHP 레거시를 벗어나 유지보수가 편해졌다

**하지만 상황에 따라 다르다:**

- **대규모 트래픽이 예상된다면:** Spring Boot 고려
- **관리자 기능이 많다면:** Django (Admin 패널)
- **팀원이 모두 JavaScript에 익숙하다면:** Node.js + TypeScript

## 기술 스택 선정 체크리스트

스타트업에서 기술 스택을 선택할 때 고려한 점:

1. **팀이 익숙한가?**
   - 새 기술 학습 시간 < 익숙한 기술로 빠르게 개발
   - 단, 너무 레거시는 피한다

2. **주니어 온보딩이 쉬운가?**
   - 팀이 성장하면 주니어가 들어온다
   - 러닝 커브가 낮은 기술이 유리

3. **커뮤니티가 활성화돼있나?**
   - 문제 발생 시 해결책을 찾기 쉬운가
   - 라이브러리 생태계가 풍부한가

4. **요구사항에 맞는가?**
   - 실시간 채팅 → WebSocket 지원
   - 대용량 배치 → Java Spring Batch
   - 빠른 프로토타입 → Python/Node.js

5. **확장 가능한가?**
   - 트래픽이 늘어날 때 대응 가능한가
   - 마이크로서비스 전환이 쉬운가

## 마무리

기술 스택 선정은 정답이 없다. 팀 상황, 서비스 특성, 시장 상황에 따라 다르다. 중요한 건 **왜 이 기술을 선택했는지** 설명할 수 있어야 한다는 것이다.

FastAPI는 우리 팀과 잘 맞았다. 빠른 개발 속도로 3개월 만에 PHP 레거시를 벗어났고, 이후 트래픽 증가에도 안정적으로 대응할 수 있었다. 단점도 있었지만 장점이 훨씬 컸다. 레거시 재개발이나 스타트업 초기라면 FastAPI를 추천한다.
