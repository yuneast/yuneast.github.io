# CLAUDE.md

이 파일은 Claude Code (claude.ai/code)가 이 저장소에서 작업할 때 따라야 할 지침을 제공합니다.

## 공통 코딩 규칙

### 핵심 원칙

- **YAGNI (You Aren't Gonna Need It)**: 지금 당장 사용하지 않는 함수나 파일은 만들지 않기
- **KISS (Keep It Simple, Stupid)**: 단순하고 직관적인 코드 작성
- **DRY (Don't Repeat Yourself)**: 코드 중복 금지, 공통 로직은 분리하여 재사용
- **오버엔지니어링 금지**: 필요한 최소 구성만 유지

### 성능 규칙

- **2중 for문(중첩 루프) 금지**
  - Map/Set/Dictionary 인덱싱, 배치 조회, 그룹핑으로 대체
- **N+1 쿼리 금지** (ORM 사용 시)
  - 루프 내 지연 로딩 접근 금지
  - 단일 쿼리 또는 ID 수집 후 IN 조회 사용

### 코드 품질 규칙

- **DTO/VO 사용**: API 응답에 엔티티/모델 직접 노출 금지
- **단일 책임 원칙**: 각 클래스/함수는 하나의 책임만
- **레이어 분리**: Controller/Router → Service → Repository/DAO

### 예외 처리 규칙

- **예외 처리 필수**
  - 외부 API 호출, 파일 I/O, DB 트랜잭션 등 예외 발생 가능한 모든 코드
  - 예외를 무시하지 말고 로깅 또는 적절한 응답 반환
- **유효성 검증 필수**
  - 입력 계층에서 검증 (Controller/Router)
  - 비즈니스 로직 검증 (Service)
  - null/undefined 체크

### Git 커밋 메시지 규칙

**형식:** `[태그] [기능이름] 커밋 설명`

**태그 종류:**
- **[FEA]**: 새로운 기능 추가
- **[REF]**: 코드 리팩토링
- **[BUG]**: 버그 수정
- **[CNF]**: 설정 파일 변경 (application.yml, .env 등)
- **[STYLE]**: 코드 포맷팅 (로직 변경 없음)
- **[DOCS]**: 문서 수정

**주의사항:**
- 커밋 메시지는 한글로 작성
- 한 커밋에는 하나의 논리적 변경사항만 포함
- **Co-Authored-By 사용 금지**

## 프로젝트 개요

Jekyll 기반 기술 블로그입니다. minimal-mistakes 테마를 사용하며, GitHub Pages로 호스팅됩니다.

## 핵심 아키텍처

### 디렉토리 구조

- `_posts/`: 블로그 글 (YYYY-MM-DD-title.md 형식)
- `_pages/`: 정적 페이지 (about, categories, tags)
- `_config.yml`: Jekyll 설정 (테마, 플러그인, 기본값)
- `_data/navigation.yml`: 네비게이션 메뉴 설정
- `.github/workflows/pages.yml`: GitHub Actions 빌드/배포

### 블로그 글 작성 규칙

**Front Matter 필수 항목:**
```yaml
---
title: "글 제목"
date: YYYY-MM-DD
categories: [카테고리]
tags: [태그1, 태그2]
---
```

**글 작성 원칙:**
- 문제 → 분석 → 해결 → 결과 구조로 작성
- 코드 예시 포함
- 실무 경험 기반
- 자연스러운 문체 (AI 티 나는 표현 금지)

### 일반적인 작업 흐름

**새 블로그 글 추가:**
1. `_posts/YYYY-MM-DD-title.md` 파일 생성
2. Front Matter 작성
3. 커밋 & 푸시 → GitHub Actions 자동 빌드/배포

**설정 변경:**
- `_config.yml` 수정 후 커밋 & 푸시
