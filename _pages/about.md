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

장애 대응 자동화와 분산 시스템 동시성 제어를 직접 설계·구현한 백엔드 엔지니어입니다. 경력 6년 4개월. 문제를 발견하면 알림을 추가하고, 반복되면 자동화합니다.

**대표 성과 3가지**

- **LLM 자동 장애 대응 파이프라인을 직접 만들어 운영에 투입** — 오류 로그 수신 → 레포 분석 → 수정 PR 자동 생성. 장애 대응 공수 90% 절감 (오픈소스 `auto-code-fixer` → 공부선배 실무 적용)
- **동시 수주 20~30명 환경의 배차 중복 수락을 Redis 분산 락으로 제거** — 기사 2,000명·일 배차 30~40건 서비스에서 Race Condition 재발 없음
- **회원관리 + 전자결재 시스템을 1인 풀스택으로 설계·구축·운영 중** — 4 vCPU / 4 GB 단일 VPS에 앱·DB·모니터링을 모두 올려 **상시 메모리 1.9 GB(여유 52%)로 운영**. Spring Boot + Next.js, 커밋 832건 (cjbadminton.kr)

운영 이슈가 생기면 모니터링·알림·자동화까지 연결하는 것을 기본으로 삼습니다. 단순 기능 구현보다 **왜 그 방법인가**를 먼저 정리하고, 대안을 비교한 근거를 남깁니다.

- Email: ydj0617@gmail.com
- GitHub: [github.com/yuneast](https://github.com/yuneast)
- Blog: [yuneast.github.io](https://yuneast.github.io)

---

## 기술 스택

**Backend**: Java, Spring Boot, JPA, QueryDSL, Python, FastAPI, PHP, Laravel

**Frontend**: JavaScript, TypeScript, React, Next.js (App Router)

**Database**: PostgreSQL, MySQL, Redis, Flyway, SQLAlchemy

**DevOps / 운영**: Docker, Docker Compose, AWS, GitHub Actions, Caddy, Uptime Kuma, pg_stat_statements

---

## 경력

### 청주시배드민턴협회

**회원관리·이적동의서 전자결재 시스템 / 백엔드·풀스택 개발자** (2026.02 ~ 재직 중)

서비스: 청주시 전역 배드민턴 동호회·회원 대상 회원관리 및 이적동의서 전자결재 시스템 (cjbadminton.kr), 협회 주도 전수 온보딩 / **기획·설계·개발·배포·운영 1인 단독**

제약: **단일 VPS 4 vCPU / 4 GB RAM / 50 GB 디스크**에 prod(앱·DB·리버스프록시)·모니터링·dev 스택을 모두 수용 — 스케일아웃 없이 리소스 안에서 버티는 것이 전제

**핵심 성과:**
- 이적동의서를 종이 서명 + 카카오톡 전달 + 협회 카페 업로드로 처리 → 이력 추적 불가, 서류 분실·위변조 위험, 기존 동호회가 승인을 거부·방치하면 절차 자체가 중단 → **4단계 전자결재 워크플로우**(신규요청 → 기존동호회 승인 → 협회 승인 → 확정)로 전환 → 모든 이적 이력이 DB에 남고 진행 상태를 당사자가 직접 조회
  - [선택 근거] 단순 서류 업로드 게시판(이력 추적 여전히 불가) · 범용 전자결재 SaaS(협회 직권처리 같은 도메인 규칙 반영 불가) 대비, 분쟁 해결 권한을 워크플로우에 내장할 수 있는 자체 구축 채택
- 승인 거부·방치로 절차가 막히는 구조적 분쟁 → **7일 미응답 시 협회 직권처리 발동 + 거부/직권처리 사유 입력 필수화** → 분쟁이 멈추지 않고 사유가 이력에 남는 구조로 전환
- 협회 사전 등록 명단과 회원가입 계정이 분리되어 이력이 끊김 → 이름+연락처 **정확 일치**만 자동 연동(claim)하는 공용 매칭 로직 설계 → 기존 소속·이적 이력을 신규 계정이 그대로 승계
  - [선택 근거] 이름만 일치 또는 연락처 부분 일치까지 허용하면 오매칭으로 타인 이력이 섞임 → 가입 편의보다 데이터 무결성을 우선해 "정확 일치"로 제한, 미일치 건은 가입을 막지 않고 `미배정`으로 남겨 관리자가 사후 배정
- 회원명부 엑셀 대량 업로드 시 **메모리가 피크 3.1 GB / 4 GB까지 치솟는 것을 Netdata로 관측** → 행마다 반복 실행되는 매칭 쿼리가 원인 → **쿼리 배치 처리 + 응답 비동기 전환, 처리자명 조회를 루프당 1회로 축소(N+1 제거)**, 대진표 목록의 팀·인원 수 집계도 단일 쿼리로 통합 → 대량 업로드 응답 지연 해소 및 피크 메모리 완화
- 대진표 점수를 두 명이 동시에 제출하면 확인 대기가 중복 생성 → 경기당 1건으로 제약 + 제출자만 수정 가능하도록 변경 → 동시 제출 중복 레코드 제거
- 운영 가시성 전무한 초기 상태 → **Uptime Kuma 상태페이지(status.cjbadminton.kr) + 텔레그램 장애 알림 + 슬로우 쿼리 로깅 + pg_stat_statements/pgHero + Netdata 리소스 대시보드** 도입 → 장애·성능 저하를 사용자 제보 전에 감지
- 배포가 DB 컨테이너를 건드릴 위험 → prod DB를 앱 compose에서 **별도 lifecycle로 완전 분리 + 호스트 포트 미노출 + 외부 도커 네트워크 격리**, Caddy가 TLS만 종료하고 내부망으로 프록시 → 앱 재배포·재빌드가 DB에 영향 없음
- 4 GB RAM 단일 서버에 모든 스택을 올려야 해 런타임 메모리가 곧 장애 요인 → **멀티스테이지 빌드로 런타임을 JDK 없는 JRE·alpine 이미지로 축소**, Next.js `output: "standalone"`으로 서버 번들만 배포, prod는 host 포트를 열지 않고 Caddy 단일 진입점으로 통합 → **상시 메모리 1.9 GB / 4 GB (여유 약 52%)로 안정 운영**
  - [선택 근거] 알림 기능(카카오 알림톡·SMS 푸시) 도입(상시 워커·외부 연동으로 메모리·운영 부담↑) 대비, 사이트 내 상태 조회만으로 요구사항이 충족됨을 협회와 확인 → 1차 범위에서 제외해 리소스 절약
- DB 유실 대비 없음 → **매일 pg_dump → Cloudflare R2 자동 백업**, 스키마는 `ddl-auto` 제거 후 Flyway 마이그레이션(45개)으로 전환 → 스키마 변경 이력 추적 및 롤백 가능

**주요 업무:**
- 백엔드 13개 도메인 설계·구현 (auth, member, club/association, transfer, verification, bracket, tournament, board, sms, visitor, health, config, common)
- 권한 4단계 RBAC 설계 (최종관리자 / 협회관리자 / 동호회관리자 / 일반회원)
- 회원가입·아이디 찾기·비밀번호 재설정 SMS 본인인증 구현
- 친선경기(랜덤 복식)·토너먼트·조별리그 대진표 생성 엔진 구현 (급수·실력 반영 자동 팀 편성)
- Next.js App Router 프론트엔드 전체 구현 (공개 페이지 / 인증 / 협회·동호회·최종관리자 대시보드)
- dev·prod 완전 분리 Docker 스택 구성, 설계 문서 29건 선작성 후 구현, 백엔드 테스트 114개 작성

**기술**: Java, Spring Boot, Gradle(Kotlin DSL), PostgreSQL, Flyway, Next.js, TypeScript, shadcn/ui, Docker Compose, Caddy, Cloudflare R2

---

### (주)공부선배

**학원 운영관리(ERP) 서비스 / 백엔드 개발자** (2024.08 ~ 2025.12)

서비스: 학원 운영관리 ERP — **고객 학원 400여 곳, 사용자 1만 명** 규모의 B2B 멀티테넌트 서비스. 등·하원 시간대에 요청이 집중되는 피크 구조 / 인증·알림·대시보드 도메인 90% 담당

**핵심 성과:**
- PHP Fatal Error를 매번 수동 분석·수정 → **LLM 자동 트리거 오류 분석 + 수정 PR 자동 생성 파이프라인 구축** → 장애 대응 공수 90% 절감
  - [선택 근거] Slack 알림(여전히 수동 처리) · in-process 자동화(배포 결합도↑) 대비, DevOps 전담 인력 없는 환경에서 독립 서버 LLM 파이프라인이 운영 부담 최소 → 채택
- 3개 서비스(ERP, 입시설계, 마켓) 로그인 정보가 각각 다른 테이블에 분산 → **중앙 인증 서버 분리를 직접 제안**하고 타팀과 협업해 통합 인증 구축 → 1시간 점검으로 무중단 전환 완료
  - [선택 근거] 서비스 간 토큰 교환(API 호출↑) · OAuth 도입(일정 내 구축 불가) 대비, 기존 세션 구조 최소 변경으로 무중단 전환 가능 → 쿠키 기반 통합 인증 서버 채택
- 장애 감지가 사용자 제보에 의존 → 스케줄러 로그 모니터링 + 타임아웃 시 사내메신저 자동 알림 → 대응 시간 50% 단축
- 슬로우 쿼리 다수 발생 → 실행 계획 분석 후 쿼리·인덱스 최적화 → 슬로우 쿼리 0건 유지
- 운영 중 SQL 에러 빈번 발생 → 이스케이프 처리 문제 확인, Prepared Statement 전환 → SQL 에러 제거

**주요 업무:**
- 데일리 스탠드업에서 DB 설계·개발 방향 공유, PR 자동 어사인 기반 코드 리뷰
- 요구사항 불명확 시 PM 직접 확인, 팀 미팅 주도로 스펙 확정
- 컨트롤러 집중 로직 → Service-Repository 패턴 리팩토링, 배포 전 목 데이터 + 트랜잭션 롤백 검증

**기술**: Java, Spring Boot, JPA, QueryDSL, PHP, MySQL, Redis, Docker, AWS CloudWatch

---

### 가나다콜

**고소작업차 실시간 배차 서비스 / 백엔드 개발자** (2023.03 ~ 2024.07)

서비스: 고소작업차 실시간 배차 서비스 (기사 2,000명, 일 배차 30-40건), 백엔드 단독 담당

**핵심 성과:**
- 20-30명이 동시에 수주해 중복 수락 발생 → **Redis 분산 락으로 동시성 제어** → 배차 중복 수락 Race Condition 재발 없음
  - [선택 근거] DB 트랜잭션 락(락 보유 시간↑, DB 부하) · 낙관적 락(충돌 시 재시도로 UX 문제) 대비, 기존 Redis 인프라 활용 + ms 단위 TTL 제어 가능 → 채택
- 차고지 기준 일괄 알림으로 무관한 지역에도 발송 → 사용자 위치 기반 GIS + 반경 거리 설정 구현 → 불필요한 알림 60% 감소

**주요 업무:**
- 배차 시스템 아키텍처 설계, 배차 매칭 엔진 구현
- 주문·배차·정산 도메인 로직 구현

**기술**: Java, Spring Boot, JPA, QueryDSL, MySQL, Redis, FCM, AWS CloudWatch

---

### (주)유토빌

**공동주택 관리 서비스 / 백엔드 개발자** (2021.09 ~ 2023.02)

서비스: 공동주택 관리 서비스 (20개 단지, 단지당 수백~수천 세대), 개발팀 3명 중 백엔드 주도

**핵심 성과:**
- 수동 배포로 릴리즈가 지연되고 배포자마다 절차가 달랐음 → **GitHub Actions CI/CD 파이프라인 구축**, 배포 트리거를 PR 머지로 단일화 → 수동 절차 제거, 배포 담당자 의존 해소
- 홈케어 이커머스가 추가되며 서비스별 로그인 분리 → 쿠키 기반 통합 인증을 **최초 설계**하고 서비스 중단 없이 전환 → 2개 서비스 단일 로그인
- 코드 리뷰 절차 부재, 주니어 2명이 단독으로 기능을 끝내지 못하는 상태 → PR 기반 리뷰 도입 + 멘토링 → 주니어 2명 독립 개발 가능 수준으로 성장
- 팀 기술 스택 선정(Python/FastAPI) 및 백엔드 아키텍처 설계 주도

**주요 업무:**
- 관리비 대납, 주차 관리, 투표 기능 API 개발

**기술**: Python, PHP, FastAPI, MySQL, Docker, GitHub Actions

---

### 콘디

**인플루언서 중개 서비스 / 창업자 (풀스택 개발)** (2020.06 ~ 2021.08)

서비스: 인플루언서-광고주 매칭 플랫폼, 0부터 기획·설계·단독 개발

**핵심 성과:**
- 0부터 MVP 개발 및 출시 → 가입자 3,000명 규모 서비스 운영
- 블로거 지수 분석 요청이 Django 워커를 점유해 서버 전체가 멈춤 → **Celery 비동기 큐 적용** → 동시 분석 요청 시 502 에러 제거
- 중복 크롤링으로 응답 지연 → Redis 캐싱 적용 → API 응답 시간 5초 → 즉시 응답

**주요 업무:**
- 블로거 지수 산출 알고리즘 설계·구현 (게시글 크롤링 → 키워드 추출 → 검색 순위 확인 → 가중치 적용 종합 점수 산출)

**기술**: Python, Django, MySQL, Celery, Redis, Crawling

---

## 오픈소스

### [auto-code-fixer](https://github.com/yuneast/auto-code-fixer)

운영 서버에서 에러가 발생하면 오류 로그를 수신해 해당 레포를 clone하고, Claude Code SDK로 분석·수정한 뒤 브랜치·커밋·PR까지 자동 생성하는 서버.

- 운영 서버에서 에러 감지 → 오류 정보 전송 → 레포 clone → Claude Code SDK가 프로젝트 분석·수정 → 브랜치 생성, 커밋, PR 푸시
- 실무(공부선배)에서 PHP Fatal Error 자동 수정 파이프라인으로 투입, **장애 대응 공수 90% 절감**
- **기술**: Python, Claude Code SDK, GitHub API

### 기술 블로그 — [yuneast.github.io](https://yuneast.github.io)

실무에서 겪은 장애·병목의 원인 분석과 선택 근거를 기록. Redis 분산 락, Celery 비동기 처리, 쿠키 기반 SSO, LLM 자동 오류 수정 파이프라인 등 15편 이상.

---

## 학력

- **학점은행제** - 컴퓨터공학 전공 (2025.04 ~ , 2026년 학위 취득 예정)
- **브렌트 인터내셔널 스쿨 수빅** - 졸업 (2015.06 ~ 2016.12)

---

## 자격증

- **네트워크관리사 2급** (2025.12)
- **TESAT 3급** (2026.01)

---

## 병역

- **면제** (장기대기 면제)

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
