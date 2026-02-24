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

책임감 있게 모든 일에 최선을 다하는 백엔드 개발자 윤동준 입니다.

Python, FastAPI, Java, Spring Boot 을 이용한 백엔드 애플리케이션을 개발합니다. 적절한 관심사 분리를 통해 의존성을 관리하고 유연하고 확장성이 좋은 설계를 지향합니다.

안정적이고 좋은 성능의 기술의 사용과 함께 서비스 사용자, 운영 등 다양한 관점을 가지고 비즈니스 임팩트를 고민합니다.

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

서비스: 학원 운영관리 ERP (사용자 1만 명, 피크 타임 초당 10건+ 요청), 인증·알림·대시보드 도메인 90% 담당

**핵심 성과:**
- 운영 중 SQL 에러 빈번 발생 → 이스케이프 문제 확인, Prepared Statement 전환 → SQL 에러 제거
- 3개 서비스(ERP, 입시설계, 마켓) 로그인 정보가 각각 다른 테이블에 분산 → 중앙 인증 서버 분리 제안, 타팀 협업 쿠키 기반 통합 인증 구축 → 1시간 점검으로 전환 완료
- PHP Fatal Error 수동 분석·수정 → LLM 자동 트리거 오류 분석 + PR 자동 생성 파이프라인 구축 → 장애 대응 공수 90% 절감
- 장애 감지 지연 → 스케줄러 로그 모니터링 + 타임아웃 시 사내메신저 자동 알림 → 대응 시간 50% 단축
- 슬로우 쿼리 다수 발생 → 쿼리 최적화 → 슬로우 쿼리 0건 유지

**주요 업무:**
- 데일리 스탠드업 DB 설계·개발 방향 공유, PR 자동 어사인 코드 리뷰
- 요구사항 불명확 시 PM 직접 확인, 팀 미팅 주도로 스펙 확정
- 컨트롤러 집중 로직 → Service-Repository 패턴 리팩토링, 배포 전 목 데이터 + 트랜잭션 롤백 검증

**기술**: Java, Spring Boot, JPA, QueryDSL, PHP, MySQL, Redis, Docker, AWS CloudWatch

---

### 가나다콜

**고소작업차 실시간 배차 서비스 / 백엔드 개발자** (2023.03 ~ 2024.07)

서비스: 고소작업차 실시간 배차 서비스 (기사 2,000명, 일 배차 30-40건), 백엔드 단독 담당

**핵심 성과:**
- 20-30명 동시 수주로 중복 수락 발생 → Redis 분산 락으로 동시성 제어 → 배차 중복 수락 Race Condition 완전 해결
- 차고지 기반 알림으로 불필요한 지역에 알림 발송 → 사용자 위치 기반 GIS + 반경 거리 설정 구현 → 불필요한 알림 60% 감소

**주요 업무:**
- 배차 시스템 아키텍처 설계, 배차 매칭 엔진 구현
- 주문·배차·정산 도메인 로직 구현

**기술**: Java, Spring Boot, JPA, QueryDSL, MySQL, Redis, FCM, AWS CloudWatch

---

### (주)유토빌

**공동주택 관리 서비스 / 백엔드 개발자** (2021.09 ~ 2023.02)

서비스: 공동주택 관리 서비스 (20개 단지, 단지당 수백~수천 세대), 개발팀 3명 중 백엔드 주도

**핵심 성과:**
- 홈케어 이커머스 추가로 다중 로그인 불편 → 쿠키 기반 통합 인증 무중단 전환 → 2개 서비스 통합 인증 완성
- 수동 배포로 릴리즈 지연 → GitHub Actions CI/CD 파이프라인 구축 → 배포 주기 단축, 배포 안정성 확보
- PR 기반 코드 리뷰 도입, 주니어 2명 멘토링 → 주니어 개발자 2명 독립 개발 가능 수준으로 성장

**주요 업무:**
- 팀 기술 스택 선정 (Python/FastAPI), 백엔드 아키텍처 설계 주도
- 관리비 대납, 주차 관리, 투표 기능 API 개발

**기술**: Python, PHP, FastAPI, MySQL, Docker, GitHub Actions

---

### 콘디

**인플루언서 중개 서비스 / 창업자 (풀스택 개발)** (2020.06 ~ 2021.08)

서비스: 인플루언서-광고주 매칭 플랫폼, 0부터 기획·설계·단독 개발

**핵심 성과:**
- 0부터 MVP 개발 및 출시 → 가입자 3,000명 규모 서비스 운영
- 블로거 지수 분석 요청 시 Django 워커 점유로 서버 전체 멈춤 → Celery 비동기 큐 적용 → 동시 분석 요청 시 502 에러 제거
- 중복 크롤링으로 응답 지연 → Redis 캐싱 적용 → API 응답 시간 5초 → 즉시 응답

**주요 업무:**
- Django 풀스택 개발, 블로거 지수 산출 알고리즘 설계·구현 (게시글 크롤링 → 키워드 추출 → 검색 순위 확인 → 가중치 적용 종합 점수 산출)

**기술**: Python, Django, Django Template, MySQL, Celery, Redis, Crawling


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
