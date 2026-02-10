---
title: "About"
permalink: /about/
layout: single
author_profile: true
toc: true
toc_sticky: true
---

<div id="pdf-download-btn" style="text-align: right; margin-bottom: 1em;">
  <button onclick="saveAsPdf()" style="padding: 8px 16px; background: #494e52; color: #fff; border: none; border-radius: 4px; cursor: pointer; font-size: 0.85em;">
    <i class="fas fa-file-pdf"></i> PDF로 저장
  </button>
</div>

## 윤동준 (Dongjun Yun)

11년차 백엔드 개발자. 문제를 발견하고, 분석하고, 해결합니다.

- Email: ydj0617@gmail.com
- GitHub: [github.com/yuneast](https://github.com/yuneast)
- Blog: [yuneast.github.io](https://yuneast.github.io)

---

## 기술 스택

**Backend**: Java, Spring Boot, Python, FastAPI, PHP, Laravel

**Frontend**: JavaScript, TypeScript, React, Next.js

**Database**: PostgreSQL, MySQL, Redis, JPA, QueryDSL, SQLAlchemy

**DevOps**: Docker, AWS, GitHub Actions

---

## 경력

### (주)공부선배

**학원 운영관리(ERP) 서비스 / 백엔드 개발자** (2024.08 ~ 2025.12)

학원 운영관리 서비스 (사용자 1만 명, 피크 타임 초당 10건+ 요청), 인증·알림·대시보드 도메인 90% 담당

**기여:**
- 데일리 스탠드업에서 DB 설계·개발 방향 공유, PR 자동 어사인 코드 리뷰
- 요구사항 불명확 시 PM 직접 확인, 팀 미팅 주도로 스펙 확정
- 운영 중 SQL 에러 빈번 발생, 이스케이프 문제 확인 후 Prepared Statement로 전환
- PHP 레거시 코드 리팩토링, Service-Repository 패턴 도입
- 3개 서비스(ERP, 입시설계, 마켓)의 로그인 정보가 각각 다른 테이블에 분산, 중앙 인증 서버 분리를 제안하고 다른 팀과 협업하여 쿠키 기반 통합 인증 구축
- 스케줄러 로그 모니터링, 타임아웃 시 사내메신저 자동 알림
- PHP Fatal Error 발생 시 LLM 자동 트리거, 오류 분석·수정 후 PR 자동 생성 파이프라인 구축

**성과:**
- 쿼리 최적화로 슬로우 쿼리 0건 수준 유지
- Prepared Statement 전환으로 이스케이프 문제로 인한 SQL 에러 제거
- 3개 서비스(ERP, 입시설계, 마켓)의 분산된 로그인 테이블을 중앙 인증 서버로 통합, 짧은 점검(1시간)으로 전환
- 장애 감지 자동화로 대응 시간 50% 단축
- LLM 자동 오류 수정 및 PR 생성으로 장애 대응 공수 90% 절감

**기술**: Java, Spring Boot, JPA, QueryDSL, PHP, MySQL, Redis, Docker, AWS CloudWatch

---

### 가나다콜

**고소작업차 실시간 배차 서비스 / 백엔드 개발자** (2023.03 ~ 2024.07)

실시간 배차 서비스 (기사 2,000명, 일 배차 30-40건), 백엔드 단독 담당

**기여:**
- 배차 시스템 아키텍처 설계, 배차 매칭 엔진 구현
- 주문·배차·정산 도메인 로직 구현
- 20-30명 동시 수주로 중복 수락 발생, Redis 분산 락으로 동시성 제어
- 차고지 기반 알림으로 불필요한 지역에 알림 발송, 사용자 위치 기반 GIS와 반경 거리 설정 기능 구현

**성과:**
- 분산 락 도입으로 배차 중복 수락 Race Condition 완전 해결
- 위치 기반 알림 최적화로 불필요한 알림 60% 감소

**기술**: Java, Spring Boot, JPA, QueryDSL, MySQL, Redis, FCM, AWS CloudWatch

---

### (주)유토빌

**공동주택 관리 서비스 / 백엔드 개발자** (2021.09 ~ 2023.02)

공동주택 관리 서비스 (20개 단지, 단지당 수백~수천 세대), 개발팀 3명 (백엔드 주니어 1명, 프론트 주니어 1명)

**기여:**
- 팀 기술 스택 선정 (Python/FastAPI), 백엔드 아키텍처 설계 주도
- 관리비 대납, 주차 관리, 투표 기능 API 개발
- PR 기반 코드 리뷰 도입, 주니어 2명 PR 리뷰·버그 해결 멘토링
- 홈케어 이커머스 추가로 다중 로그인 불편, 쿠키 기반 통합 인증으로 무중단 전환
- GitHub Actions CI/CD 파이프라인 구축, 배포 자동화

**성과:**
- 무중단 인증 전환, 2개 서비스 통합 인증 완성
- CI/CD 도입으로 배포 주기 단축, 배포 안정성 확보
- 코드 리뷰 정착, 주니어 개발자 2명 독립 개발 가능 수준으로 성장

**기술**: Python, PHP, FastAPI, MySQL, Docker, GitHub Actions, CI/CD

---

### 콘디

**인플루언서 중개 서비스 / 창업자 (풀스택 개발)** (2020.06 ~ 2021.08)

**기여:**
- 인플루언서-광고주 매칭 플랫폼 0부터 기획, 서비스 설계 및 단독 개발
- Django 풀스택 개발, 블로거 지수 산출 알고리즘 설계 및 구현 (최근 게시글 30개 크롤링 → 키워드 추출 → 네이버 검색 순위 확인 → 가중치 적용 종합 점수 산출)
- 블로거 지수 분석 요청 시 Django 워커 점유로 서버 전체 멈춤, Celery 비동기 큐와 Redis 캐싱 적용

**성과:**
- 0부터 MVP 개발 및 출시, 가입자 3,000명 규모 서비스 운영
- Celery 비동기 처리로 Django 워커 점유 문제 해결, 동시 분석 요청 시 502 에러 제거
- Redis 캐싱으로 중복 크롤링 방지, 캐시 히트 시 API 응답 시간 5초 → 즉시 응답으로 개선

**기술**: Python, Django, Django Template, MySQL, Celery, Redis, Crawling


---

### 호주 온라인 스포츠 베팅 서비스 (Sportsbet 계열)

**데이터 엔지니어링 및 풀스택 개발** (2018.06 ~ 2018.12)

호주 정부 승인 합법 스포츠 베팅 업체, 실시간 배당 데이터 수집 엔진 및 UI 단독 개발

**기여:**
- Classic ASP 기반 실시간 배당 데이터 스크래핑 엔진 구축
- 소스 사이트 DOM 구조 변경, 종목별 상이한 데이터 스키마 대응 예외 처리 로직 구현
- 팀명, 리그명 불일치 해소 매핑 엔진 설계 및 DB 동기화
- 실시간 배당 흐름 시각화 대시보드 디자인, 프론트엔드 개발

**성과:**
- 예외 처리 로직으로 소스 사이트 구조 변경 시에도 데이터 유실률 1% 미만 유지
- 원시 데이터 수집부터 DB 스키마 설계, 웹 렌더링까지 전체 파이프라인 단독 구축

**기술**: Classic ASP, VBScript, MS-SQL, JavaScript, CSS

---

### 뉴소프트

**프리랜서 풀스택 개발** (2015.08 ~ 2018.03)

- Python 기반 부동산 전월세 신고 자동화 서비스 개발
- PHP Laravel 기반 가상화폐 선물 거래소 백엔드 개발
- Solidity 기반 Polygon 네트워크 DeFi Farm 스마트 컨트랙트 구현
- C# 기반 마케팅 자동화 도구 개발
- React, Node.js 기반 장기렌트카 견적 비교 플랫폼 개발

**기술**: PHP, Laravel, Python, C#, React, Node.js, Solidity, MySQL, Redis

---

## 학력

- **학점은행제** - 컴퓨터공학 전공 (2025.04 ~ , 2026년 학위 취득 예정)
- **브렌트 인터내셔널 스쿨 수빅** - 졸업 (2015.06 ~ 2016.12)

---

## 자격증

- **네트워크관리사 2급** (2025.12)
- **TESAT 3급** (2026.01)

<script src="https://cdnjs.cloudflare.com/ajax/libs/html2pdf.js/0.10.1/html2pdf.bundle.min.js"></script>
<script>
function saveAsPdf() {
  var content = document.querySelector('.page__content');
  var clone = content.cloneNode(true);
  var btn = clone.querySelector('#pdf-download-btn');
  if (btn) btn.remove();
  var toc = clone.querySelector('.sidebar__right');
  if (toc) toc.remove();

  var opt = {
    margin: [10, 15, 10, 15],
    filename: '윤동준_이력서.pdf',
    image: { type: 'jpeg', quality: 0.98 },
    html2canvas: { scale: 2, useCORS: true },
    jsPDF: { unit: 'mm', format: 'a4', orientation: 'portrait' },
    pagebreak: { mode: ['avoid-all', 'css', 'legacy'] }
  };

  html2pdf().set(opt).from(clone).save();
}
</script>
