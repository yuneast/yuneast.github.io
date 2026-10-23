---
title: "도메인이 21개가 되자 진입점이 무너졌다 - import_module로 라우터 등록 자동화"
date: 2022-09-15
categories: [backend]
tags: [fastapi, python, architecture, import-module, router, rbac]
---

`main.py`를 열었더니 `include_router`가 60줄 가까이 이어져 있었다. 도메인을 하나 추가하면 네 군데를 고쳐야 했고, 어느 날 하나를 빼먹어서 API가 404를 냈다.

## 배경

공동주택 관리 서비스 백엔드를 혼자 맡고 있었다. 필리핀 시장을 대상으로 한 서비스였고, 특징은 **같은 데이터를 네 역할이 다르게 본다**는 것이었다.

- **입주민** — 본인 세대 정보, 관리비, 민원 접수
- **관리사무소** — 단지 전체 관리, 민원 처리, 시설 예약 승인
- **경비실** — 방문차량 출입, 택배 수령
- **본사(지점)** — 여러 단지 통합 대시보드

도메인은 관리비, 시설 예약, 민원, 택배, 주차, 투표, 공지, 장터, 부동산, 주소, 알림, 약관… 계속 늘어났다. 최종적으로 21개 도메인에 API가 200개를 넘겼다.

## 문제: 라우터가 늘어나는 속도

FastAPI는 라우터를 앱에 등록해줘야 한다. 처음엔 당연히 이렇게 썼다.

```python
from home_management.cost.routers import resident_cost, manage_cost
from home_management.facility.routers import resident_facility, manage_facility
from home_management.help_desk.routers import resident_help_desk, manage_help_desk, security_help_desk
# ... 계속

app.include_router(resident_cost.router)
app.include_router(manage_cost.router)
app.include_router(resident_facility.router)
app.include_router(manage_facility.router)
# ... 계속
```

도메인이 21개고 역할이 4종이니 라우터 파일이 수십 개가 된다. 문제가 두 가지였다.

**1. 추가할 때마다 진입점을 고쳐야 한다.** 새 도메인을 만들면 import 한 줄, include 한 줄을 `main.py`에 넣어야 한다. 기능 코드와 아무 상관없는 파일을 매번 건드린다.

**2. 빼먹으면 런타임에 조용히 사라진다.** include를 안 하면 에러가 나지 않는다. 그냥 그 API가 없는 상태로 서버가 뜬다. 프런트에서 404를 받고 나서야 알게 된다. 실제로 한 번 겪었다.

## 해결: 선언만 하면 등록되게

라우터 목록을 **데이터로 선언**하고, `import_module`로 동적으로 불러 등록하도록 바꿨다.

```python
from importlib import import_module

router_modules = {
    "base": [
        {"auth": ["token", "auth"]},
        {"email": ["email"]},
        {"payment": ["payment"]},
        {"flea_market": ["flea_market"]},
        {"real_estate": ["real_estate"]},
    ],
    "home_management": [
        {"address": ["address"]},
        {"cost": ["resident_cost", "manage_cost"]},
        {"facility": ["resident_facility", "manage_facility"]},
        {"help_desk": ["resident_help_desk", "manage_help_desk", "security_help_desk"]},
        {"parking": ["resident_parking_vehicle", "manage_parking_vehicle", "security_parking_vehicle"]},
        {"opinion_poll": ["resident_opinion_poll", "manage_opinion_poll"]},
        # ...
    ],
}

for package, domains in router_modules.items():
    for domain in domains:
        for domain_name, routers in domain.items():
            for router_name in routers:
                module = import_module(f"{package}.{domain_name}.routers.{router_name}")
                app.include_router(module.router)
```

도메인을 추가할 때 **딕셔너리에 한 줄만 넣으면** 된다. import 구문도, include 호출도 손으로 쓰지 않는다.

### 파일명이 규칙이 됐다

이 구조의 부수 효과가 더 좋았다. 라우터 파일명을 `<역할>_<도메인>.py`로 고정하게 되면서, **파일명만 보면 누가 쓰는 API인지 알 수 있게 됐다.**

```
home_management/help_desk/routers/
├── resident_help_desk.py   입주민 — 민원 접수, 내 민원 조회
├── manage_help_desk.py     관리사무소 — 민원 처리, 배정
└── security_help_desk.py   경비실 — 현장 확인
```

역할별로 파일이 갈라져 있으니 권한 분기가 파일 단위로 정리된다. 전에는 하나의 라우터 안에서 `if user.role == ...`로 분기하다가 조건이 엉켰던 적이 있었다.

## 트레이드오프

자동화하면서 잃은 것도 있다. 정직하게 적어둔다.

**1. IDE가 추적하지 못한다.** 문자열로 모듈을 불러오니 "이 라우터가 어디서 등록되는지" 점프가 안 된다. 코드 검색으로 찾아야 한다.

**2. 오타가 런타임에 터진다.** 딕셔너리의 모듈명을 잘못 적으면 `ModuleNotFoundError`로 **서버가 아예 뜨지 않는다.** 이건 장점이기도 하다. 전에는 빼먹으면 조용히 사라졌는데, 이제는 틀리면 바로 알 수 있다. 조용한 실패보다 시끄러운 실패가 낫다.

**3. 등록 순서가 암묵적이다.** 경로가 겹치는 라우터가 있으면 등록 순서가 동작에 영향을 준다. 딕셔너리 순서에 의존하게 되므로, 경로가 겹치지 않도록 설계하는 쪽이 안전하다.

디렉토리를 스캔해서 전부 자동 등록하는 방법도 있었지만 택하지 않았다. **무엇이 등록되는지 한 곳에서 읽을 수 있는 게 더 중요하다**고 봤다. 선언 목록은 남겨두고, 반복 작업만 자동화하는 선에서 멈췄다.

## 마무리

도메인이 5개일 때는 `include_router`를 손으로 쓰는 게 아무 문제가 아니다. 21개가 되면 진입점이 병목이 된다. 코드량이 늘어난 게 아니라 **"기능을 추가할 때 건드려야 하는 파일 수"**가 늘어난 게 문제였다.

반복이 보이면 그 반복을 없애는 것보다, **그 반복을 선언으로 바꿀 수 있는지** 먼저 보면 좋다. 선언으로 남겨두면 나중에 읽을 사람(=반년 뒤의 나)이 전체 구조를 한눈에 볼 수 있다.
