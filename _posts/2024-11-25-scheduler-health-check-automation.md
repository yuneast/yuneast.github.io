---
title: "크론 서버가 멈췄는데 아무도 몰랐다 - 스케줄러 헬스체크 자동화"
date: 2024-11-25
categories: [devops]
tags: [scheduler, monitoring, cloudwatch, automation, alerting]
---

크론 서버가 죽었는데 아무도 몰랐다. 청구서 발행이 안 돼서 사용자 문의가 폭주하고 나서야 알게 됐다. 이 글에서는 크론잡 헬스체크를 자동화해서 장애를 즉시 감지하는 시스템을 구축한 과정을 정리한다.

## 문제 발생

학원 ERP 레거시 PHP 서비스에서 5-60개의 크론잡이 돌고 있었다:

어느 날, 크론 서버가 죽었다. 5-60개의 크론잡이 모두 멈췄고, 특히 청구서 발행이 안 돼서 학부모들에게 문의 전화가 폭주했다.

```bash
# CloudWatch Logs Insights 쿼리로 로그 확인
fields @timestamp, @message
| filter @message like /청구서 발행/
| sort @timestamp desc
| limit 20

# 결과: 크론잡 시작 로그는 있는데 종료 로그가 없음
[2024-11-01 10:00:01] 청구서 발행 작업 시작
# 이후 로그 없음
```

CloudWatch Logs에 모든 로그를 넣고 쿼리로 확인하고 있었지만, 능동적으로 모니터링하지 않아서 문제를 놓쳤다.

## 기존 모니터링의 한계

크론 서버를 모니터링하는 방법이 없었다. 배치 작업이 실행되고 있는지, 정상 종료됐는지 알 수 없었다.

**문제점:**
- 작업이 멈춰도 알림이 없다
- 사용자가 먼저 발견한다
- 장애 시간이 길어진다 (평균 2~4시간)

## 해결 방향

크론잡이 정상 실행됐는지 자동으로 모니터링하고, 문제 발생 시 즉시 알림을 보내는 시스템이 필요했다.

### 설계 원칙

1. **크론잡 실행마다 DB에 작업 내역 INSERT**
2. **CloudWatch Logs에 모든 로그 적재, 쿼리로 확인 가능**
3. **PM2로 모니터링 스크립트 실행**
4. **타임아웃 시 사내메신저 자동 알림**

## 구현

### 1. 크론잡 실행 시 DB에 작업 내역 INSERT

레거시 PHP 크론잡이 실행될 때마다 DB에 실행 기록을 남긴다.

```php
<?php
// cron_jobs/invoice_generator.php
require_once __DIR__ . '/config/database.php';

function generate_invoices() {
    global $db;
    $task_name = "invoice_generation";
    $start_time = date('Y-m-d H:i:s');

    try {
        // 크론잡 시작 로그 INSERT
        $stmt = $db->prepare("
            INSERT INTO cron_job_logs (task_name, status, started_at)
            VALUES (?, 'RUNNING', ?)
        ");
        $stmt->execute([$task_name, $start_time]);
        $log_id = $db->lastInsertId();

        // 청구서 발행 로직
        $invoices = create_monthly_invoices();
        send_invoice_emails($invoices);

        // 성공 로그 업데이트
        $stmt = $db->prepare("
            UPDATE cron_job_logs
            SET status = 'SUCCESS',
                completed_at = ?,
                message = ?
            WHERE id = ?
        ");
        $message = count($invoices) . "건 청구서 발행 완료";
        $stmt->execute([date('Y-m-d H:i:s'), $message, $log_id]);

    } catch (Exception $e) {
        // 실패 로그 업데이트
        $stmt = $db->prepare("
            UPDATE cron_job_logs
            SET status = 'FAILED',
                completed_at = ?,
                error_message = ?
            WHERE id = ?
        ");
        $stmt->execute([date('Y-m-d H:i:s'), $e->getMessage(), $log_id]);
        throw $e;
    }
}

generate_invoices();
```

### 2. CloudWatch Logs에 로그 전송

PHP 애플리케이션 로그를 CloudWatch Logs로 전송한다.

```php
<?php
// logging_config.php
require_once __DIR__ . '/vendor/autoload.php';

use Aws\CloudWatchLogs\CloudWatchLogsClient;

function log_to_cloudwatch($message, $level = 'INFO') {
    $client = new CloudWatchLogsClient([
        'version' => 'latest',
        'region' => 'ap-northeast-2'
    ]);

    $client->putLogEvents([
        'logGroupName' => '/aws/cron/scheduler',
        'logStreamName' => 'invoice_generation',
        'logEvents' => [[
            'message' => "[$level] $message",
            'timestamp' => time() * 1000
        ]]
    ]);
}

// 사용
log_to_cloudwatch("[START] 청구서 발행 작업 시작");
log_to_cloudwatch("[SUCCESS] {$count}건 청구서 발행 완료");
```

### 3. PM2로 모니터링 스크립트 실행

DB의 크론잡 로그를 주기적으로 체크하는 모니터링 스크립트를 PM2로 실행한다.

```python
# monitor_cron_jobs.py
import time
import datetime
import requests
from app.models import CronJobLog
from app.database import db

MESSENGER_WEBHOOK_URL = "https://hooks.slack.com/services/xxxx"
TIMEOUT_MINUTES = 3

def check_cron_server():
    """크론 서버 상태 체크"""
    # 가장 최근 실행된 크론잡 로그 조회
    latest_log = CronJobLog.query.order_by(
        CronJobLog.started_at.desc()
    ).first()

    if not latest_log:
        send_alert("⚠️ 크론잡 실행 기록이 없습니다")
        return

    # RUNNING 상태로 3분 이상 지속 시 서버 죽음으로 판단
    if latest_log.status == "RUNNING":
        elapsed = (datetime.datetime.now() - latest_log.started_at).seconds / 60
        if elapsed > TIMEOUT_MINUTES:
            send_alert(
                f"🚨 크론 서버 다운 감지\n"
                f"작업: {latest_log.task_name}\n"
                f"시작: {latest_log.started_at}\n"
                f"경과: {elapsed:.1f}분"
            )

    # FAILED 상태 체크
    elif latest_log.status == "FAILED":
        send_alert(
            f"❌ 크론잡 실패\n"
            f"작업: {latest_log.task_name}\n"
            f"에러: {latest_log.error_message}"
        )

def send_alert(message: str):
    """사내메신저 알림 발송"""
    requests.post(MESSENGER_WEBHOOK_URL, json={"text": message})

if __name__ == "__main__":
    while True:
        check_cron_server()
        time.sleep(60)  # 1분마다 체크
```

### 4. PM2로 모니터링 스크립트 실행

```bash
# PM2로 모니터링 스크립트 실행
pm2 start monitor_cron_jobs.py --name cron-monitor

# PM2 상태 확인
pm2 list

# PM2 로그 확인
pm2 logs cron-monitor
```

PM2는 프로세스가 죽으면 자동 재시작하므로, 모니터링 스크립트 자체의 안정성도 확보된다.

### 5. CloudWatch Logs Insights로 로그 분석

CloudWatch Logs에 쌓인 로그를 쿼리로 분석한다.

```
-- 최근 1시간 동안 FAILED 로그 조회
fields @timestamp, @message
| filter @message like /FAILED/
| sort @timestamp desc
| limit 20

-- 특정 크론잡의 실행 기록 조회
fields @timestamp, @message
| filter @message like /invoice_generation/
| sort @timestamp desc
| limit 50

-- 타임아웃 발생 건수
fields @timestamp
| filter @message like /타임아웃/
| stats count() by bin(5m)
```

## DB 스키마

크론잡 실행 내역을 저장하는 테이블 구조:

```sql
CREATE TABLE cron_job_logs (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    task_name VARCHAR(100) NOT NULL,
    status VARCHAR(20) NOT NULL,  -- RUNNING, SUCCESS, FAILED
    started_at DATETIME NOT NULL,
    completed_at DATETIME,
    message VARCHAR(500),
    error_message TEXT,
    INDEX idx_task_started (task_name, started_at DESC)
);
```

## 결과

| 항목 | Before | After |
|------|--------|-------|
| 장애 감지 시간 | 사용자 문의 폭주 후 (수시간) | PM2 모니터링으로 즉시 (1분 이내) |
| 장애 대응 시간 | 평균 4시간 | 평균 2시간 |
| 장애 대응 시간 단축 | - | 50% |
| 모니터링 | 없음 | DB 로그 + CloudWatch Logs + PM2 |

3개월간 모니터링 결과:
- 타임아웃 알람 4건 발생 (모두 즉시 감지)
- 사내메신저로 자동 알림 발송
- 사용자 문의 폭주 사례 0건

**장애 대응 시간이 50% 단축된 이유:**
- Before: 사용자 문의 폭주 (수시간) + 원인 파악 (1시간) = 평균 4시간
- After: PM2 모니터링 즉시 감지 (1분) + CloudWatch 로그로 원인 파악 (30분) = 평균 2시간

## 주의할 점

1. **PM2 프로세스 자체도 모니터링해야 한다.** PM2가 죽으면 모니터링도 멈춘다. PM2를 systemd로 관리해서 서버 재시작 시 자동 실행되게 했다.

2. **CloudWatch 로그 비용을 고려한다.** 로그 보관 기간을 7일 정도로 제한했다. 중요한 로그는 DB에 영구 저장한다.

3. **타임아웃 시간은 여유있게 설정한다.** 작업의 평균 실행 시간보다 2~3배 여유를 둔다. 너무 짧으면 오탐이 발생한다.

4. **알림 피로도를 주의한다.** 사소한 경고까지 메신저로 보내면 무시하게 된다. 실제 장애(타임아웃, FAILED 상태)만 알림을 보낸다.

## 배운 점

1. **모니터링이 없으면 문제가 있어도 모른다.** 크론잡은 조용히 실패한다. 사용자가 발견하기 전에 먼저 알아야 한다.

2. **DB에 작업 내역을 남기는 게 가장 확실하다.** 로그는 유실될 수 있지만 DB INSERT는 트랜잭션으로 보장된다.

3. **PM2는 프로세스 관리에 편리하다.** 자동 재시작, 로그 관리, 상태 확인이 간단하다.

4. **CloudWatch Logs Insights는 강력하다.** SQL 비슷한 쿼리로 로그를 분석할 수 있어서 문제 원인 파악이 빠르다.

## 마무리

크론잡은 백그라운드에서 조용히 실행되기 때문에, 문제가 생겨도 알기 어렵다. DB에 작업 내역을 INSERT하고, PM2로 모니터링 스크립트를 돌리고, CloudWatch Logs로 로그를 분석하면 장애를 즉시 감지할 수 있다. 이 시스템으로 장애 대응 시간을 50% 단축했다.
