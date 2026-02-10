---
title: "PHP Fatal Error를 LLM이 자동으로 고치게 만든 이야기"
date: 2025-03-10
categories: [backend]
tags: [llm, automation, php, error-handling, devops]
---

"PHP Fatal Error 발생 → 슬랙 알림 확인 → 코드 확인 → 원인 분석 → 수정 → PR 생성 → 코드 리뷰 → 배포". 이 흐름을 자동화하면 어떨까? 이 글에서는 PHP Fatal Error 발생 시 LLM이 자동으로 오류를 분석하고 수정한 뒤 PR을 생성하는 파이프라인을 구축한 경험을 공유한다.

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
LLM에 코드 + 에러 정보 전달
    ↓
LLM이 수정된 코드 생성
    ↓
자동으로 브랜치 생성, 커밋, PR 생성
    ↓
사내 메신저로 PR 링크 알림
    ↓
개발자가 리뷰 후 머지
```

핵심 원칙: **LLM이 직접 배포하지 않는다.** PR을 생성하고 사람이 리뷰하는 단계를 반드시 거친다.

## 구현

### 1. 에러 로그 감지

PHP 에러 로그를 주기적으로 감시하는 스크립트를 만들었다.

```python
import re
import time
from pathlib import Path

class ErrorLogWatcher:
    def __init__(self, log_path: str):
        self.log_path = Path(log_path)
        self.last_position = self._get_file_size()

    def watch(self) -> list[dict]:
        current_size = self._get_file_size()

        if current_size <= self.last_position:
            return []

        with open(self.log_path, 'r') as f:
            f.seek(self.last_position)
            new_content = f.read()

        self.last_position = current_size
        return self._parse_fatal_errors(new_content)

    def _parse_fatal_errors(self, content: str) -> list[dict]:
        pattern = r'PHP Fatal error:\s+(.+?) in (.+?) on line (\d+)'
        errors = []

        for match in re.finditer(pattern, content):
            errors.append({
                'message': match.group(1),
                'file': match.group(2),
                'line': int(match.group(3)),
            })

        return errors

    def _get_file_size(self) -> int:
        return self.log_path.stat().st_size if self.log_path.exists() else 0
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

### 3. LLM 호출

프롬프트 설계가 핵심이다. "코드를 고쳐줘"가 아니라, 정확한 지시를 내린다.

```python
import openai

class LLMFixer:
    def __init__(self, api_key: str):
        self.client = openai.OpenAI(api_key=api_key)

    def fix(self, context: dict) -> dict:
        prompt = f"""다음 PHP 코드에서 Fatal Error가 발생했습니다.

**에러 메시지:** {context['error_message']}
**파일:** {context['file_path']}
**라인:** {context['error_line']}

**전체 코드:**
```php
{context['full_source']}
```

규칙:
1. 에러가 발생한 부분만 최소한으로 수정하세요.
2. 기존 로직을 변경하지 마세요.
3. 수정된 전체 파일 코드를 반환하세요.
4. 수정 내용을 한 줄로 설명하세요.

응답 형식:
DESCRIPTION: (수정 설명)
CODE:
```php
(수정된 전체 코드)
```"""

        response = self.client.chat.completions.create(
            model="gpt-4o",
            messages=[
                {"role": "system", "content": "PHP 코드 버그 수정 전문가입니다. 최소한의 변경만 합니다."},
                {"role": "user", "content": prompt}
            ],
            temperature=0,
        )

        return self._parse_response(response.choices[0].message.content)

    def _parse_response(self, text: str) -> dict:
        desc_match = re.search(r'DESCRIPTION:\s*(.+)', text)
        code_match = re.search(r'```php\n(.+?)```', text, re.DOTALL)

        return {
            'description': desc_match.group(1).strip() if desc_match else '',
            'fixed_code': code_match.group(1).strip() if code_match else '',
        }
```

### 4. 자동 PR 생성

수정된 코드를 브랜치에 커밋하고 PR을 생성한다.

```python
import subprocess
from datetime import datetime

class AutoPRCreator:
    def __init__(self, repo_path: str):
        self.repo_path = repo_path

    def create_pr(self, file_path: str, fixed_code: str, description: str) -> str:
        timestamp = datetime.now().strftime('%Y%m%d%H%M%S')
        branch_name = f"auto-fix/{timestamp}"

        commands = [
            ['git', 'checkout', '-b', branch_name],
        ]

        for cmd in commands:
            subprocess.run(cmd, cwd=self.repo_path, check=True)

        # 수정된 코드 저장
        with open(file_path, 'w') as f:
            f.write(fixed_code)

        commit_commands = [
            ['git', 'add', file_path],
            ['git', 'commit', '-m', f'[BUG] LLM 자동 수정: {description}'],
            ['git', 'push', 'origin', branch_name],
        ]

        for cmd in commit_commands:
            subprocess.run(cmd, cwd=self.repo_path, check=True)

        # gh CLI로 PR 생성
        result = subprocess.run(
            ['gh', 'pr', 'create',
             '--title', f'[Auto Fix] {description}',
             '--body', f'LLM이 자동 생성한 수정 PR입니다.\n\n수정 내용: {description}\n\n반드시 코드 리뷰 후 머지해주세요.'],
            cwd=self.repo_path,
            capture_output=True, text=True, check=True
        )

        # main 브랜치로 복귀
        subprocess.run(['git', 'checkout', 'main'], cwd=self.repo_path, check=True)

        return result.stdout.strip()  # PR URL
```

### 5. 전체 파이프라인 연결

```python
class AutoFixPipeline:
    def __init__(self, log_path: str, repo_path: str, api_key: str):
        self.watcher = ErrorLogWatcher(log_path)
        self.collector = ErrorContextCollector()
        self.fixer = LLMFixer(api_key)
        self.pr_creator = AutoPRCreator(repo_path)
        self.notifier = MessengerNotifier()

    def run(self):
        errors = self.watcher.watch()

        for error in errors:
            context = self.collector.collect(error)
            fix_result = self.fixer.fix(context)

            if not fix_result['fixed_code']:
                self.notifier.send(f"LLM 수정 실패: {error['message']}")
                continue

            pr_url = self.pr_creator.create_pr(
                error['file'],
                fix_result['fixed_code'],
                fix_result['description']
            )

            self.notifier.send(
                f"LLM 자동 수정 PR 생성\n"
                f"에러: {error['message']}\n"
                f"수정: {fix_result['description']}\n"
                f"PR: {pr_url}"
            )
```

## 결과

파이프라인 도입 후:
- 수동 오류 수정 평균 1시간 → LLM 자동 수정 10분
- **장애 대응 공수 90% 절감**
- 개발자는 수정 코드를 리뷰하고 머지만 하면 됨

## 한계와 주의점

1. **단순한 에러만 처리 가능하다.** 타입 에러, null 체크 누락 같은 패턴화된 에러에만 효과적이다. 비즈니스 로직 버그는 LLM이 판단할 수 없다.

2. **반드시 사람이 리뷰해야 한다.** LLM이 생성한 코드를 자동 배포하면 안 된다. 의도하지 않은 로직 변경이 생길 수 있다.

3. **프롬프트 엔지니어링이 중요하다.** "최소한의 변경만 하라"는 지시가 없으면 LLM이 주변 코드까지 리팩토링하는 경우가 있었다. 에러 수정만 하도록 제한해야 한다.

4. **비용을 고려해야 한다.** GPT-4 API 호출 비용이 발생한다. 에러 빈도가 높으면 비용이 늘어난다. 하지만 개발자 시간 대비 API 비용은 미미하다.

## 마무리

이 파이프라인의 가치는 "LLM이 코드를 잘 고친다"가 아니다. **반복적이고 패턴화된 작업에서 개발자를 해방시킨다**는 점이 핵심이다. 에러 알림을 보고 → 코드를 열고 → null 체크를 추가하는 작업을 매번 사람이 할 필요가 없다. 사람은 LLM이 올린 PR을 리뷰하고, 비즈니스 로직에 영향이 없는지 확인하는 데 집중하면 된다.
