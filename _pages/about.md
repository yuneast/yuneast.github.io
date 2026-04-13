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

장애 대응 자동화 파이프라인과 분산 시스템 동시성 제어를 직접 설계·구현한 백엔드 엔지니어입니다. 문제를 발견하면 알림을 추가하고, 반복되면 자동화합니다.

Java, Spring Boot, Python, FastAPI를 주로 사용하며, 단순 기능 구현보다 **왜 그 방법인가**를 먼저 고민합니다. 운영 이슈가 생기면 모니터링·알림·자동화까지 연결하는 것을 자연스럽게 여깁니다.

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
  - [선택 근거] 서비스 간 토큰 교환(API 호출↑) · OAuth 도입(일정 내 구축 불가) 대비, 기존 세션 구조 최소 변경으로 무중단 전환 가능 → 쿠키 기반 통합 인증 서버 채택
- PHP Fatal Error 수동 분석·수정 → LLM 자동 트리거 오류 분석 + PR 자동 생성 파이프라인 구축 → 장애 대응 공수 90% 절감
  - [선택 근거] Slack 알림(여전히 수동 처리) · in-process 자동화(배포 결합도↑) 대비, DevOps 전담 인력 없는 환경에서 독립 서버 LLM 파이프라인이 운영 부담 최소 → 채택
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
  - [선택 근거] DB 트랜잭션 락(락 보유 시간↑, DB 부하) · 낙관적 락(충돌 시 재시도로 UX 문제) 대비, 기존 Redis 인프라 활용 + ms 단위 TTL 제어 가능 → 채택
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

## 오픈소스

### [auto-code-fixer](https://github.com/yuneast/auto-code-fixer)

운영 서버에서 에러 발생 시 오류 로그를 수신하고, 해당 레포를 clone하여 Claude Code SDK로 자동 분석·수정 후 커밋·푸시하는 서버.

- 운영 서버 try-except 등에서 에러 감지 → 오류 정보를 auto-code-fixer 서버로 전송 → 레포 clone → Claude Code SDK가 프로젝트 분석·수정 → 브랜치 생성, 커밋, PR 푸시
- 실무에서 PHP Fatal Error 자동 수정 파이프라인으로 활용, 장애 대응 공수 90% 절감
- **기술**: Python, Claude Code SDK, GitHub API

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
