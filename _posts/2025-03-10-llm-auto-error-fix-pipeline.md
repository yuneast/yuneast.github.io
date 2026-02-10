---
title: "PHP Fatal Error를 LLM이 자동으로 고치게 만든 이야기"
date: 2025-03-10
categories: [backend]
tags: [llm, automation, php, error-handling, devops]
---

"PHP Fatal Error 발생 → 사내메신저 알림 확인 → 코드 확인 → 원인 분석 → 수정 → PR 생성 → 코드 리뷰 → 배포". 이 흐름을 자동화하면 어떨까? 이 글에서는 PHP Fatal Error 발생 시 LLM이 자동으로 오류를 분석하고 수정한 뒤 PR을 생성하는 파이프라인을 구축한 경험을 공유한다.

## 문제 인식

학원 ERP 서비스에는 PHP 레거시 코드가 상당 부분 남아있었다. PHP는 타입 안전성이 약하다 보니 런타임에 Fatal Error가 종종 발생했다:

- `TypeError: Cannot access offset of type string on string`
- `Undefined variable` 참조
- `Call to undefined method` 호출
- `null` 체크 누락

이런 에러들의 특징은:
- 대부분 원인이 명확하다 (타입 미스매치, null 체크 누락 등)
- 수정도 단순하다 (타입 체크 추가, null 분기 추가 등)
- 하지만 감지부터 수정까지 평균 1시간이 걸린다

반복적이고 패턴화된 작업이라면 자동화할 수 있지 않을까?

## 파이프라인 설계

```
PHP Fatal Error 발생
    ↓
에러 로그 감지 (로그 모니터링)
    ↓
에러 정보 수집 (파일 경로, 라인, 에러 메시지, 스택 트레이스)
    ↓
릴리즈 브랜치에서 프로젝트 clone
    ↓
Claude Code SDK 서버에 에러 정보 전달
    ↓
Claude Code가 프로젝트 전체를 해석하여 수정
    ↓
자동으로 브랜치 생성, 커밋, PR 생성
    ↓
사내 메신저로 PR 링크 알림
    ↓
개발자가 리뷰 후 머지
```

핵심 원칙: **Claude Code가 직접 배포하지 않는다.** PR을 생성하고 사람이 리뷰하는 단계를 반드시 거친다. **Claude Code SDK**를 사용해서 프로젝트 전체 컨텍스트를 이해하고 수정한다.

## 구현

### 1. CloudWatch 에러 로그 감지

AWS CloudWatch Logs를 통해 PHP Fatal Error를 감지한다.

```python
import re
import boto3
from datetime import datetime, timedelta

class CloudWatchErrorWatcher:
    def __init__(self, log_group: str, log_stream: str):
        self.client = boto3.client('logs', region_name='ap-northeast-2')
        self.log_group = log_group
        self.log_stream = log_stream
        self.last_timestamp = int((datetime.now() - timedelta(minutes=5)).timestamp() * 1000)

    def watch(self) -> list[dict]:
        response = self.client.get_log_events(
            logGroupName=self.log_group,
            logStreamName=self.log_stream,
            startTime=self.last_timestamp,
            limit=100
        )

        events = response.get('events', [])
        if events:
            self.last_timestamp = events[-1]['timestamp'] + 1

        return self._parse_fatal_errors(events)

    def _parse_fatal_errors(self, events: list) -> list[dict]:
        pattern = r'PHP Fatal error:\s+(.+?) in (.+?) on line (\d+)'
        errors = []

        for event in events:
            message = event['message']
            match = re.search(pattern, message)

            if match:
                errors.append({
                    'message': match.group(1),
                    'file': match.group(2),
                    'line': int(match.group(3)),
                    'timestamp': event['timestamp']
                })

        return errors
```

### 2. 컨텍스트 수집

LLM에게 에러를 고치라고만 하면 안 된다. 충분한 컨텍스트를 제공해야 한다.

```python
class ErrorContextCollector:
    def collect(self, error: dict) -> dict:
        file_path = error['file']
        error_line = error['line']

        # 에러 발생 파일의 전체 코드
        with open(file_path, 'r') as f:
            source_code = f.read()

        # 에러 발생 라인 주변 코드 (앞뒤 20줄)
        lines = source_code.split('\n')
        start = max(0, error_line - 20)
        end = min(len(lines), error_line + 20)
        context_lines = lines[start:end]

        return {
            'error_message': error['message'],
            'file_path': file_path,
            'error_line': error_line,
            'full_source': source_code,
            'context': '\n'.join(context_lines),
            'context_start_line': start + 1,
        }
```

### 3. Claude Code SDK 서버 구축

**Claude Code SDK**를 사용해서 프로젝트 전체를 이해하고 컨텍스트를 파악한다.

```python
import subprocess
import shutil
from pathlib import Path

class ClaudeCodeFixer:
    def __init__(self, claude_sdk_server_url: str, repo_url: str):
        self.server_url = claude_sdk_server_url
        self.repo_url = repo_url
        self.work_dir = Path('/tmp/auto-fix-workspace')

    def fix(self, context: dict) -> dict:
        # 1. 릴리즈 브랜치에서 프로젝트 clone
        if self.work_dir.exists():
            shutil.rmtree(self.work_dir)

        subprocess.run([
            'git', 'clone',
            '--branch', 'release',
            '--depth', '1',
            self.repo_url,
            str(self.work_dir)
        ], check=True)

        # 2. Claude Code SDK에 에러 정보와 프로젝트 경로 전달
        prompt = f"""PHP Fatal Error가 발생했습니다.

**에러 메시지:** {context['error_message']}
**파일:** {context['file_path']}
**라인:** {context['error_line']}

프로젝트를 분석하고 에러를 최소한으로 수정해주세요.
기존 로직을 변경하지 말고 에러만 해결하세요."""

        # Claude Code SDK 서버에 요청
        response = requests.post(
            f"{self.server_url}/api/fix",
            json={
                'project_path': str(self.work_dir),
                'error_context': context,
                'prompt': prompt
            }
        )

        result = response.json()
        return {
            'description': result['description'],
            'fixed_files': result['fixed_files'],
            'work_dir': str(self.work_dir)
        }
```

Claude Code SDK는 단순히 에러가 발생한 파일만 보는 게 아니라, **프로젝트 전체 구조를 파악**한다. 관련 파일, 의존성, 함수 호출 관계를 이해하고 수정하기 때문에 더 정확하다.

### 4. 자동 PR 생성

Claude Code가 수정한 코드를 브랜치에 커밋하고 PR을 생성한다.

```python
import subprocess
from datetime import datetime

class AutoPRCreator:
    def create_pr(self, work_dir: str, description: str) -> str:
        timestamp = datetime.now().strftime('%Y%m%d%H%M%S')
        branch_name = f"auto-fix/{timestamp}"

        # 브랜치 생성 및 전환
        subprocess.run(
            ['git', 'checkout', '-b', branch_name],
            cwd=work_dir, check=True
        )

        # Claude Code가 수정한 파일들 커밋
        commit_commands = [
            ['git', 'add', '-A'],
            ['git', 'commit', '-m', f'[BUG] Claude Code 자동 수정: {description}'],
            ['git', 'push', 'origin', branch_name],
        ]

        for cmd in commit_commands:
            subprocess.run(cmd, cwd=work_dir, check=True)

        # gh CLI로 PR 생성
        result = subprocess.run(
            ['gh', 'pr', 'create',
             '--title', f'[Auto Fix] {description}',
             '--body', f'Claude Code가 자동 생성한 수정 PR입니다.\n\n수정 내용: {description}\n\n반드시 코드 리뷰 후 머지해주세요.'],
            cwd=work_dir,
            capture_output=True, text=True, check=True
        )

        return result.stdout.strip()  # PR URL
```

### 5. 전체 파이프라인 연결

```python
class AutoFixPipeline:
    def __init__(self, log_group: str, log_stream: str, claude_server_url: str, repo_url: str):
        self.watcher = CloudWatchErrorWatcher(log_group, log_stream)
        self.collector = ErrorContextCollector()
        self.fixer = ClaudeCodeFixer(claude_server_url, repo_url)
        self.pr_creator = AutoPRCreator()
        self.notifier = MessengerNotifier()

    def run(self):
        errors = self.watcher.watch()

        for error in errors:
            try:
                context = self.collector.collect(error)

                # Claude Code가 릴리즈 브랜치 clone 후 프로젝트 해석하여 수정
                fix_result = self.fixer.fix(context)

                if not fix_result.get('fixed_files'):
                    self.notifier.send(f"Claude Code 수정 실패: {error['message']}")
                    continue

                # 수정된 코드로 PR 생성
                pr_url = self.pr_creator.create_pr(
                    fix_result['work_dir'],
                    fix_result['description']
                )

                self.notifier.send(
                    f"Claude Code 자동 수정 PR 생성\n"
                    f"에러: {error['message']}\n"
                    f"수정: {fix_result['description']}\n"
                    f"PR: {pr_url}"
                )

            except Exception as e:
                self.notifier.send(f"자동 수정 파이프라인 실패: {str(e)}")
```

## 결과

파이프라인 도입 후:
- 수동 오류 수정 평균 1시간 → Claude Code 자동 수정 10분
- **장애 대응 공수 90% 절감**
- CloudWatch로 에러 감지 즉시 자동화 파이프라인 트리거
- 개발자는 Claude Code가 생성한 PR을 리뷰하고 머지만 하면 됨
- 프로젝트 전체 컨텍스트를 이해하기 때문에 정확도가 높음

## 한계와 주의점

1. **단순한 에러만 처리 가능하다.** 타입 에러, null 체크 누락 같은 패턴화된 에러에만 효과적이다. 비즈니스 로직 버그는 Claude Code가 판단할 수 없다.

2. **반드시 사람이 리뷰해야 한다.** Claude Code가 생성한 코드를 자동 배포하면 안 된다. 의도하지 않은 로직 변경이 생길 수 있다.

3. **프롬프트 엔지니어링이 중요하다.** "최소한의 변경만 하라"는 지시가 없으면 Claude Code가 주변 코드까지 리팩토링하는 경우가 있었다. 에러 수정만 하도록 제한해야 한다.

4. **Claude Code SDK 서버 관리가 필요하다.** 별도 서버를 띄워서 Claude Code SDK를 실행해야 한다. 프로젝트 전체를 해석할 수 있다는 장점이 크다.

## 마무리

이 파이프라인의 가치는 "Claude Code가 코드를 잘 고친다"가 아니다. **반복적이고 패턴화된 작업에서 개발자를 해방시킨다**는 점이 핵심이다. 에러 알림을 보고 → 코드를 열고 → null 체크를 추가하는 작업을 매번 사람이 할 필요가 없다. 사람은 Claude Code가 올린 PR을 리뷰하고, 비즈니스 로직에 영향이 없는지 확인하는 데 집중하면 된다.

Claude Code SDK는 **프로젝트 전체 컨텍스트를 이해**한다. 단순히 에러가 발생한 파일만 보는 게 아니라, 관련 파일, 의존성, 함수 호출 관계를 파악하고 수정한다. 릴리즈 브랜치를 clone해서 전체 프로젝트를 분석하는 방식이 번거로워 보이지만, 정확도 면에서 우수했다.
