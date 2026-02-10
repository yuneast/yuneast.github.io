---
title: "GitHub Actions로 CI/CD 파이프라인 구축하기 - 수동 배포에서 자동 배포까지"
date: 2022-06-15
categories: [devops]
tags: [github-actions, cicd, docker, deployment, automation]
---

"배포할게요, 잠깐만요." 매주 금요일 오후 4시, 배포 담당자가 서버에 SSH 접속해서 `git pull` → `docker-compose restart`를 실행한다. 배포가 끝나면 직접 브라우저를 열어서 주요 기능을 확인한다. 이 과정을 자동화한 이야기를 정리한다.

## 기존 배포 프로세스

공동주택 관리 서비스에서 배포는 이렇게 진행되고 있었다:

1. 개발자가 main 브랜치에 머지
2. 배포 담당자(나)가 서버에 SSH 접속
3. `git pull origin main`
4. `pip install -r requirements.txt` (의존성 변경 시)
5. `docker-compose restart`
6. 브라우저에서 직접 확인
7. 문제 있으면 `git revert` 후 재배포

문제점:
- 주 1회 배포 (쌓인 변경사항이 많아 리스크 증가)
- 사람이 직접 실행 (실수 가능성)
- 테스트 없이 배포 (버그가 프로덕션에서 발견)
- 롤백에 시간 소요 (수동 revert)

## 목표

- PR 머지 시 자동 테스트 실행
- 테스트 통과 시 자동 배포
- 실패 시 슬랙 알림
- 롤백 자동화

## GitHub Actions 워크플로우 구성

### 1. CI 파이프라인 (PR 테스트)

PR이 생성되면 자동으로 테스트를 실행하는 워크플로우.

```yaml
# .github/workflows/ci.yml
name: CI

on:
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest

    services:
      mysql:
        image: mysql:8.0
        env:
          MYSQL_ROOT_PASSWORD: test
          MYSQL_DATABASE: test_db
        ports:
          - 3306:3306
        options: >-
          --health-cmd="mysqladmin ping"
          --health-interval=10s
          --health-timeout=5s
          --health-retries=3

      redis:
        image: redis:7
        ports:
          - 6379:6379

    steps:
      - uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'
          cache: 'pip'

      - name: Install dependencies
        run: pip install -r requirements.txt

      - name: Run linter
        run: |
          pip install flake8
          flake8 . --count --select=E9,F63,F7,F82 --show-source --statistics

      - name: Run tests
        env:
          DATABASE_URL: mysql://root:test@localhost:3306/test_db
          REDIS_URL: redis://localhost:6379/0
        run: pytest --tb=short -q
```

PR에 테스트 결과가 체크로 표시된다. 테스트가 실패하면 머지 버튼이 비활성화되도록 Branch Protection Rule을 설정했다.

### 2. CD 파이프라인 (자동 배포)

main에 머지되면 자동으로 배포하는 워크플로우.

```yaml
# .github/workflows/cd.yml
name: CD

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Build Docker image
        run: |
          docker build -t app:${{ github.sha }} .
          docker save app:${{ github.sha }} | gzip > app.tar.gz

      - name: Copy to server
        uses: appleboy/scp-action@v0.1.7
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          source: "app.tar.gz,docker-compose.prod.yml"
          target: "/app"

      - name: Deploy
        uses: appleboy/ssh-action@v1.0.3
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          script: |
            cd /app
            docker load < app.tar.gz
            docker tag app:${{ github.sha }} app:latest
            docker compose -f docker-compose.prod.yml up -d
            rm app.tar.gz

      - name: Health check
        uses: appleboy/ssh-action@v1.0.3
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          script: |
            for i in 1 2 3 4 5; do
              if curl -sf http://localhost:8000/health > /dev/null; then
                echo "Health check passed"
                exit 0
              fi
              echo "Attempt $i failed, waiting..."
              sleep 5
            done
            echo "Health check failed"
            exit 1

      - name: Notify success
        if: success()
        run: |
          curl -X POST ${{ secrets.SLACK_WEBHOOK }} \
            -H 'Content-type: application/json' \
            -d '{"text": "배포 성공: ${{ github.event.head_commit.message }}"}'

      - name: Notify failure
        if: failure()
        run: |
          curl -X POST ${{ secrets.SLACK_WEBHOOK }} \
            -H 'Content-type: application/json' \
            -d '{"text": "배포 실패! 확인 필요: ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}"}'
```

### Dockerfile

```dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

EXPOSE 8000

CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### Health Check 엔드포인트

배포 후 서비스가 정상 동작하는지 확인하는 엔드포인트.

```python
# health.py
from fastapi import APIRouter

router = APIRouter()

@router.get("/health")
async def health_check():
    return {"status": "ok"}
```

## 도입 결과

| 항목 | Before | After |
|------|--------|-------|
| 배포 빈도 | 주 1회 | 일 평균 3회 |
| 배포 실패율 | 간헐적 실패 | 0% (테스트 통과 후 배포) |
| 배포 소요 시간 | 30분 (수동) | 5분 (자동) |
| 롤백 시간 | 20분 | 이전 커밋 revert PR 머지 |

### 배포 빈도가 늘어난 이유

수동 배포일 때는 "한 번에 많이 모아서 배포"하는 경향이 있었다. 자동 배포가 되니 작은 변경도 바로 머지하고 배포하게 됐다. 변경사항이 작으니 문제가 생겨도 원인 파악이 빠르다.

## 팀에 미친 영향

### PR 기반 코드 리뷰 정착

CI가 돌아가면서 자연스럽게 PR 프로세스가 정착됐다. main에 직접 푸시하면 테스트 없이 배포되니, PR을 거치는 게 당연해졌다. 주니어 개발자 2명의 PR도 100% 리뷰하면서, 독립적으로 기능 개발이 가능한 수준까지 성장했다.

### 심리적 안전감

"배포해도 될까?" 하는 불안감이 줄었다. 테스트가 통과했으니 최소한의 품질이 보장되고, 문제가 생기면 revert PR 하나로 롤백할 수 있다.

## 개선 과정에서 겪은 시행착오

1. **처음에 테스트가 없었다.** CI를 만들어도 돌릴 테스트가 없으면 의미가 없다. 주요 API 엔드포인트의 통합 테스트부터 하나씩 추가했다.

2. **Secrets 관리를 잊었다.** 초기에 서버 비밀번호를 워크플로우에 하드코딩했다가 리뷰에서 걸렸다. GitHub Secrets로 분리하고, SSH Key 인증으로 변경했다.

3. **Docker 이미지 캐싱이 필요했다.** 매번 `pip install`을 새로 하면 빌드 시간이 오래 걸린다. Docker 레이어 캐싱과 pip cache를 활용해서 빌드 시간을 3분에서 1분으로 줄였다.

## 마무리

CI/CD는 특별한 기술이 아니다. GitHub Actions의 YAML 파일 몇 개로 구성할 수 있다. 중요한 건 "자동화할 수 있는 건 자동화한다"는 마인드셋이다. 수동으로 하던 배포를 자동화하면, 그 시간에 더 가치 있는 일을 할 수 있다.
