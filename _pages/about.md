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

고등학교를 필리핀에서 마치고, 정규 CS 학위 과정 없이 독학으로 개발을 시작했습니다. 2020년 인플루언서 매칭 플랫폼을 창업해 0부터 혼자 만들어 가입자 3,000명까지 운영했고, 이후 배차·공동주택·학원 ERP 백엔드를 거쳐 **경력 6년 4개월**이 됐습니다. 현재는 청주시배드민턴협회의 회원관리·전자결재 시스템을 혼자 설계·개발·운영하고 있습니다. (컴퓨터공학 학위는 학점은행제로 2026년 취득 예정)

문제를 발견하면 알림을 붙이고, 반복되면 자동화합니다. 그래서 2026년에는 **운영 서버의 에러를 LLM이 분석해 수정 PR까지 자동으로 올리는 파이프라인**을 직접 만들어 실무에 투입했습니다.

백엔드 설계부터 배포·모니터링까지 혼자 세울 수 있습니다. 팀이 작고 맡을 범위가 넓을 때 가장 빠르게 기여합니다.

**대표 성과**

- **장애 대응 공수 90% 절감** — 수동으로 분석·수정하던 PHP Fatal Error를 LLM 자동 분석 + PR 자동 생성 파이프라인으로 대체 (직접 만든 오픈소스 `auto-code-fixer`를 실무 투입)
- **동시 수주 20~30명 환경의 배차 중복 수락 제거** — Redis 분산 락으로 Race Condition 해소, 기사 2,000명 서비스에서 재발 없음
- **4 vCPU / 4 GB 단일 서버에서 상시 메모리 1.9 GB(여유 52%) 유지** — 앱·DB·모니터링·dev 스택을 한 대에 올린 1인 운영 환경

- Email: ydj0617@gmail.com
- GitHub: [github.com/yuneast](https://github.com/yuneast)
- Blog: [yuneast.github.io](https://yuneast.github.io)

---

## 기술 스택

**Backend**: Java, Spring Boot, JPA, QueryDSL, Python, FastAPI, PHP, Laravel

**Frontend**: TypeScript, React, Next.js (App Router)

**Database**: PostgreSQL, MySQL, Redis, Flyway

**운영**: Docker, AWS, GitHub Actions, Caddy, Uptime Kuma, pg_stat_statements

---

## 대표 프로젝트

### auto-code-fixer — LLM 장애 자동 수정 파이프라인

[github.com/yuneast/auto-code-fixer](https://github.com/yuneast/auto-code-fixer) · Python, Claude Code SDK, GitHub API

운영 서버에서 에러가 발생하면 오류 로그를 수신해 레포를 clone하고, LLM이 코드를 분석·수정한 뒤 브랜치·커밋·PR까지 자동으로 올리는 서버입니다.

사람이 로그를 열어보기 전에 수정 PR이 먼저 올라와 있는 상태를 목표로 만들었습니다. 실무(공부선배)에 투입해 **PHP Fatal Error 대응 공수를 90% 줄였습니다.**

- [선택 근거] Slack 알림만 붙이는 방식(여전히 사람이 수동 처리) · 애플리케이션 내부에 자동화를 넣는 방식(배포와 결합도 상승) 대비, DevOps 전담 인력이 없는 환경에서는 **독립 서버로 분리한 파이프라인**이 운영 부담이 가장 작다고 판단해 채택

---

## 경력

### 청주시배드민턴협회 — 백엔드·풀스택 (2026.02 ~ 재직 중)

**회원관리 + 이적동의서 전자결재 시스템** · [cjbadminton.kr](https://cjbadminton.kr)

청주시 전역의 배드민턴 동호회·회원을 협회가 전수 온보딩하는 시스템입니다. **기획·설계·개발·배포·운영을 혼자 합니다.** 서버는 단일 VPS 4 vCPU / 4 GB — 스케일아웃 없이 이 안에서 버티는 것이 전제입니다.

**핵심 성과:**

- 이적동의서가 종이 서명 → 카카오톡 전달 → 협회 카페 업로드로 처리되던 탓에 이력 추적이 불가능하고, **기존 동호회가 승인을 거부하거나 방치하면 절차 자체가 멈췄습니다** (지역 언론에 보도된 반복적 갈등) → 4단계 전자결재 워크플로우로 전환하고 **7일 미응답 시 협회 직권처리 + 사유 입력 필수화**를 규칙으로 못박음 → 분쟁이 교착되지 않고, 거부·직권처리 사유가 모두 이력에 남는 구조
  - [선택 근거] 단순 서류 업로드 게시판(이력 추적 여전히 불가) · 범용 전자결재 SaaS(협회 직권처리 같은 도메인 규칙 반영 불가) 대비, **분쟁 해결 권한을 워크플로우에 내장**할 수 있는 자체 구축을 선택
- 협회가 미리 올려둔 회원 명단과 본인이 직접 만든 계정이 분리되어 이력이 끊김 → 이름+연락처 **정확 일치만** 자동 연동(claim)하는 공용 매칭 로직 설계 → 기존 소속·이적 이력을 신규 계정이 그대로 승계
  - [선택 근거] 이름만 일치하거나 연락처가 부분 일치해도 연동하면 **타인의 이적 이력이 섞입니다.** 가입 편의보다 데이터 무결성을 우선해 정확 일치로 제한하고, 미일치 건은 가입을 막지 않고 `미배정`으로 남겨 관리자가 사후 배정
- 회원명부 엑셀 대량 업로드 중 **메모리가 피크 3.1 GB / 4 GB까지 치솟는 것을 직접 붙인 Netdata로 관측** → 행마다 반복 실행되는 매칭 쿼리를 원인으로 특정 → 쿼리 배치 처리 + 응답 비동기 전환, 처리자명 조회를 루프당 1회로 축소(N+1 제거) → 업로드 응답 지연 해소, 피크 메모리 완화
- 운영 가시성이 전무한 상태에서 시작해, 상태페이지·장애 알림·슬로우 쿼리 로깅·리소스 대시보드를 **직접 구축해 혼자 운영 중** → 장애와 성능 저하를 사용자 제보 전에 감지

**주요 업무:**
- 백엔드 13개 도메인 설계·구현, 권한 4단계 RBAC (최종관리자 / 협회관리자 / 동호회관리자 / 일반회원)
- 회원가입·아이디 찾기·비밀번호 재설정 SMS 본인인증
- 친선경기(랜덤 복식)·토너먼트·조별리그 대진표 생성 엔진 — 급수·실력을 반영한 자동 팀 편성, 동시 점수 제출 시 중복 레코드 방지
- Next.js App Router 프론트엔드 전체 구현 (공개 페이지 / 인증 / 3종 관리자 대시보드)
- 운영 기반: dev·prod 완전 분리 Docker 스택, prod DB를 앱과 별도 lifecycle로 격리(재배포가 DB 무영향), Caddy 단일 진입점 + 자동 HTTPS, 매일 pg_dump → Cloudflare R2 자동 백업, Flyway 마이그레이션 45개
- 2개월간 설계 문서 29건을 먼저 쓰고 구현, 커밋 832건, 백엔드 테스트 114개

**기술**: Java, Spring Boot, Gradle(Kotlin DSL), PostgreSQL, Flyway, Next.js, TypeScript, Docker, Caddy

---

### (주)공부선배 — 백엔드 (2024.08 ~ 2025.12)

**학원 운영관리 ERP** · 고객 학원 400여 곳, 사용자 1만 명 B2B 멀티테넌트 · 인증·알림·대시보드 도메인 90% 담당

등·하원 시간대에 요청이 몰리는 피크 구조라, 평소에 괜찮던 쿼리가 그 시간에 터집니다.

**핵심 성과:**

- PHP Fatal Error를 매번 사람이 열어보고 고치던 구조 → **LLM 자동 오류 분석 + 수정 PR 생성 파이프라인 구축** → 장애 대응 공수 90% 절감 (위 `auto-code-fixer`)
- 3개 서비스(ERP, 입시설계, 마켓)의 로그인 정보가 각각 다른 테이블에 흩어져 있었음 → **중앙 인증 서버 분리를 직접 제안**하고 타팀과 협업해 쿠키 기반 통합 인증 구축 → 1시간 점검으로 무중단 전환
  - [선택 근거] 서비스 간 토큰 교환(API 호출 증가) · OAuth 전면 도입(일정 내 구축 불가) 대비, **기존 세션 구조를 최소로만 바꿔** 무중단 전환이 가능한 쿠키 기반 중앙 인증을 채택
- 장애를 사용자 제보로 알게 되던 상태 → 스케줄러 로그 모니터링 + 타임아웃 시 사내메신저 자동 알림 → 대응 시간 50% 단축
- 슬로우 쿼리 다수 → 실행 계획 분석 후 쿼리·인덱스 최적화 → **슬로우 쿼리 0건 유지.** 이스케이프 처리 문제로 반복되던 SQL 에러도 Prepared Statement 전환으로 제거

**주요 업무:**
- 요구사항이 불명확하면 PM에게 직접 확인하고 팀 미팅을 주도해 스펙을 확정
- 컨트롤러에 몰려 있던 로직을 Service-Repository 패턴으로 리팩토링, 배포 전 목 데이터 + 트랜잭션 롤백으로 검증
- 데일리 스탠드업에서 DB 설계·개발 방향 공유, PR 자동 어사인 기반 코드 리뷰

**기술**: Java, Spring Boot, JPA, QueryDSL, PHP, MySQL, Redis, Docker, AWS CloudWatch

---

### 가나다콜 — 백엔드 단독 (2023.03 ~ 2024.07)

**고소작업차 실시간 배차 서비스** · 기사 2,000명, 일 배차 30~40건 · 백엔드 전체를 혼자 담당

**핵심 성과:**

- 하나의 배차 건에 기사 20~30명이 동시에 수주를 눌러 **중복 수락이 발생**했습니다 → Redis 분산 락으로 동시성 제어 → 중복 수락 재발 없음
  - [선택 근거] DB 트랜잭션 락(락 보유 시간 증가, DB 부하) · 낙관적 락(충돌 시 재시도로 기사 쪽 UX 저하) 대비, 이미 쓰던 Redis 인프라를 활용하면서 **ms 단위 TTL 제어**가 가능한 분산 락을 채택
- 차고지 기준으로 알림을 일괄 발송해 무관한 지역 기사에게도 알림이 갔음 → 사용자 실시간 위치 기반 GIS + 반경 설정 구현 → **불필요한 알림 60% 감소**

배차 시스템 아키텍처와 매칭 엔진을 설계하고, 주문·배차·정산 도메인을 구현했습니다.

**기술**: Java, Spring Boot, JPA, QueryDSL, MySQL, Redis, FCM

---

### (주)유토빌 — 백엔드 (2021.09 ~ 2023.02)

**공동주택 관리 서비스** · 20개 단지, 단지당 수백~수천 세대 · 개발팀 3명 중 백엔드 주도

**핵심 성과:**

- 배포가 수동이라 릴리즈가 지연되고 배포자마다 절차가 달랐음 → GitHub Actions CI/CD 구축, **배포 트리거를 PR 머지로 단일화** → 수동 절차 제거, 특정 담당자 의존 해소
- 홈케어 이커머스가 추가되며 서비스별 로그인이 분리됨 → 쿠키 기반 통합 인증을 **최초 설계**하고 서비스 중단 없이 전환 → 2개 서비스 단일 로그인
- 코드 리뷰 절차가 없고 주니어 2명이 기능을 단독으로 끝내지 못하는 상태 → PR 기반 리뷰 도입 + 멘토링 → **주니어 2명 독립 개발 가능 수준으로 성장**

팀 기술 스택(Python/FastAPI) 선정과 백엔드 아키텍처 설계를 주도했고, 관리비 대납·주차 관리·투표 기능 API를 개발했습니다.

**기술**: Python, PHP, FastAPI, MySQL, Docker, GitHub Actions

---

### 콘디 — 창업자 / 풀스택 (2020.06 ~ 2021.08)

**인플루언서-광고주 매칭 플랫폼** · 0부터 기획·설계·단독 개발, 가입자 3,000명

**핵심 성과:**

- 블로거 지수 분석 요청이 Django 워커를 점유해 **서버 전체가 멈췄습니다** → Celery 비동기 큐 적용 → 동시 분석 요청 시 502 에러 제거
- 중복 크롤링으로 응답이 지연됨 → Redis 캐싱 적용 → API 응답 시간 **5초 → 즉시 응답**

블로거 지수 산출 알고리즘을 직접 설계·구현했습니다 (게시글 크롤링 → 키워드 추출 → 검색 순위 확인 → 가중치 적용 종합 점수).

**기술**: Python, Django, MySQL, Celery, Redis, Crawling

---

## 기술 블로그

[yuneast.github.io](https://yuneast.github.io) — 실무에서 겪은 장애와 병목의 원인 분석, 그리고 그때 왜 그 방법을 골랐는지를 기록합니다. Redis 분산 락, Celery 비동기 처리, 쿠키 기반 SSO, LLM 자동 오류 수정 파이프라인 등 15편 이상.

---

## 학력 · 자격 · 병역

- **학점은행제** 컴퓨터공학 전공 (2025.04 ~ , 2026년 학위 취득 예정)
- **브렌트 인터내셔널 스쿨 수빅** 졸업 (2015.06 ~ 2016.12)
- **네트워크관리사 2급** (2025.12)
- **병역** 면제 (장기대기 면제)

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
