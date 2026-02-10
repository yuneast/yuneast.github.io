---
title: "크론 서버가 멈췄는데 아무도 몰랐다 - 스케줄러 헬스체크 자동화"
date: 2024-11-25
categories: [devops]
tags: [scheduler, monitoring, cloudwatch, automation, alerting]
---

오전 10시에 실행돼야 할 배치 작업이 멈췄다. 오후 2시에 사용자가 "데이터가 안 들어왔어요"라고 문의해서 알게 됐다. 4시간 동안 아무도 몰랐다. 이 글에서는 크론 서버 헬스체크를 자동화해서 장애를 즉시 감지하는 시스템을 구축한 과정을 정리한다.

## 문제 발생

학원 ERP 서비스에서 여러 배치 작업을 크론으로 실행하고 있었다:

- 매일 오전 9시: 출결 알림 발송
- 매일 오전 10시: 수강료 납부 알림 발송
- 매일 오후 2시: 통계 데이터 집계
- 매일 오후 11시: 백업 작업

어느 날 오전 10시 수강료 알림이 발송되지 않았다. 사용자가 "알림이 안 왔어요"라고 문의해서 확인해보니 크론 서버가 4시간 전부터 멈춰있었다.

```bash
# 크론 서버 로그 확인
$ tail -f /var/log/cron.log

[2024-11-10 10:00:01] 수강료 알림 작업 시작
[2024-11-10 10:00:03] DB 커넥션 획득
[2024-11-10 10:00:05] 알림 대상자 조회 중...
# 여기서 멈춤 (이후 로그 없음)
```

원인: DB 쿼리가 무한 대기 상태로 빠졌고, 타임아웃 설정이 없어서 프로세스가 영원히 끝나지 않았다.

## 기존 모니터링의 한계

크론 서버를 모니터링하는 방법이 없었다. 배치 작업이 실행되고 있는지, 정상 종료됐는지 알 수 없었다.

**문제점:**
- 작업이 멈춰도 알림이 없다
- 사용자가 먼저 발견한다
- 장애 시간이 길어진다 (평균 2~4시간)

## 해결 방향

크론 작업이 정해진 시간 내에 완료되지 않으면 자동으로 알림을 보내는 시스템이 필요했다.

### 설계 원칙

1. **크론 작업마다 실행 시작/종료 로그 기록**
2. **CloudWatch에서 로그 패턴 모니터링**
3. **타임아웃 시간 초과 시 알람 발생**
4. **사내 메신저로 즉시 알림**

## 구현

### 1. 크론 작업 래퍼 스크립트

모든 크론 작업을 래퍼 스크립트로 감싸서 시작/종료 로그를 CloudWatch에 기록한다.

```bash
#!/bin/bash
# cron_wrapper.sh

TASK_NAME=$1
TIMEOUT=$2
COMMAND=$3

LOG_GROUP="/aws/cron/scheduler"
LOG_STREAM="$TASK_NAME"

# 시작 로그
aws logs put-log-events \
  --log-group-name "$LOG_GROUP" \
  --log-stream-name "$LOG_STREAM" \
  --log-events timestamp=$(date +%s)000,message="[START] $TASK_NAME 시작"

# 타임아웃과 함께 작업 실행
timeout $TIMEOUT bash -c "$COMMAND"
EXIT_CODE=$?

if [ $EXIT_CODE -eq 124 ]; then
  # 타임아웃 발생
  aws logs put-log-events \
    --log-group-name "$LOG_GROUP" \
    --log-stream-name "$LOG_STREAM" \
    --log-events timestamp=$(date +%s)000,message="[TIMEOUT] $TASK_NAME 타임아웃 ($TIMEOUT초 초과)"
  exit 1
elif [ $EXIT_CODE -ne 0 ]; then
  # 에러 발생
  aws logs put-log-events \
    --log-group-name "$LOG_GROUP" \
    --log-stream-name "$LOG_STREAM" \
    --log-events timestamp=$(date +%s)000,message="[ERROR] $TASK_NAME 실패 (exit code: $EXIT_CODE)"
  exit 1
else
  # 정상 종료
  aws logs put-log-events \
    --log-group-name "$LOG_GROUP" \
    --log-stream-name "$LOG_STREAM" \
    --log-events timestamp=$(date +%s)000,message="[SUCCESS] $TASK_NAME 완료"
fi
```

### 2. 크론탭 수정

기존 크론탭을 래퍼 스크립트를 사용하도록 변경했다.

```bash
# 기존 크론탭
0 10 * * * python /app/send_payment_notification.py

# 변경 후 (타임아웃 5분)
0 10 * * * /app/cron_wrapper.sh "payment_notification" 300 "python /app/send_payment_notification.py"
```

### 3. CloudWatch 메트릭 필터

CloudWatch에서 `[TIMEOUT]`과 `[ERROR]` 패턴을 감지하는 메트릭 필터를 생성했다.

```json
{
  "filterName": "CronTimeoutFilter",
  "filterPattern": "[TIMEOUT]",
  "metricTransformations": [
    {
      "metricName": "CronTimeout",
      "metricNamespace": "Scheduler",
      "metricValue": "1"
    }
  ]
}
```

```json
{
  "filterName": "CronErrorFilter",
  "filterPattern": "[ERROR]",
  "metricTransformations": [
    {
      "metricName": "CronError",
      "metricNamespace": "Scheduler",
      "metricValue": "1"
    }
  ]
}
```

### 4. CloudWatch 알람

메트릭이 1 이상이면 알람을 발생시킨다.

```json
{
  "AlarmName": "CronSchedulerFailure",
  "MetricName": "CronTimeout",
  "Namespace": "Scheduler",
  "Statistic": "Sum",
  "Period": 60,
  "EvaluationPeriods": 1,
  "Threshold": 1,
  "ComparisonOperator": "GreaterThanOrEqualToThreshold",
  "AlarmActions": [
    "arn:aws:sns:ap-northeast-2:xxxx:scheduler-alert"
  ]
}
```

### 5. SNS → Lambda → 사내 메신저

SNS 알람이 발생하면 Lambda가 사내 메신저로 알림을 전송한다.

```python
import json
import urllib.request

def lambda_handler(event, context):
    message = event['Records'][0]['Sns']['Message']
    alarm_data = json.loads(message)

    task_name = alarm_data.get('Trigger', {}).get('Dimensions', [{}])[0].get('value', 'Unknown')

    # 사내 메신저 웹훅
    webhook_url = "https://messenger.company.com/webhook/xxxx"

    payload = {
        "text": f"⚠️ 크론 작업 장애 발생\n작업명: {task_name}\n상태: 타임아웃 또는 에러\n시간: {alarm_data['StateChangeTime']}"
    }

    req = urllib.request.Request(
        webhook_url,
        data=json.dumps(payload).encode('utf-8'),
        headers={'Content-Type': 'application/json'}
    )

    urllib.request.urlopen(req)

    return {'statusCode': 200}
```

## 개선: 작업별 타임아웃 설정

작업마다 예상 실행 시간이 다르다. 출결 알림은 1분이면 끝나지만, 통계 집계는 10분이 걸린다. 작업별로 타임아웃을 다르게 설정했다.

```bash
# crontab
0 9 * * * /app/cron_wrapper.sh "attendance_notification" 120 "python /app/send_attendance_notification.py"
0 10 * * * /app/cron_wrapper.sh "payment_notification" 300 "python /app/send_payment_notification.py"
0 14 * * * /app/cron_wrapper.sh "stats_aggregation" 600 "python /app/aggregate_stats.py"
0 23 * * * /app/cron_wrapper.sh "backup" 1800 "bash /app/backup.sh"
```

## 실패 재시도 로직 추가

일시적 네트워크 오류로 작업이 실패할 수 있다. 3번까지 재시도하도록 래퍼 스크립트를 개선했다.

```bash
#!/bin/bash
# cron_wrapper.sh (개선)

TASK_NAME=$1
TIMEOUT=$2
COMMAND=$3
MAX_RETRIES=3

for i in $(seq 1 $MAX_RETRIES); do
  timeout $TIMEOUT bash -c "$COMMAND"
  EXIT_CODE=$?

  if [ $EXIT_CODE -eq 0 ]; then
    # 성공
    aws logs put-log-events \
      --log-group-name "$LOG_GROUP" \
      --log-stream-name "$LOG_STREAM" \
      --log-events timestamp=$(date +%s)000,message="[SUCCESS] $TASK_NAME 완료 (시도 $i/$MAX_RETRIES)"
    exit 0
  fi

  # 실패 시 다음 재시도까지 10초 대기
  if [ $i -lt $MAX_RETRIES ]; then
    sleep 10
  fi
done

# 모든 재시도 실패
aws logs put-log-events \
  --log-group-name "$LOG_GROUP" \
  --log-stream-name "$LOG_STREAM" \
  --log-events timestamp=$(date +%s)000,message="[ERROR] $TASK_NAME 실패 (재시도 $MAX_RETRIES회 모두 실패)"
exit 1
```

## 결과

| 항목 | Before | After |
|------|--------|-------|
| 장애 감지 시간 | 사용자 문의 시 (평균 2~4시간) | 즉시 (1분 이내) |
| 장애 대응 시간 | 평균 4시간 | 평균 2시간 |
| 장애 대응 시간 단축 | - | 50% |
| 모니터링 | 없음 | CloudWatch 실시간 모니터링 |

3개월간 모니터링 결과:
- 타임아웃 알람 5건 발생 (모두 즉시 감지)
- 재시도로 복구된 케이스 3건
- 수동 개입이 필요한 케이스 2건

**장애 대응 시간이 50% 단축된 이유:**
- Before: 사용자 문의 (평균 2~4시간) + 원인 파악 (1시간) = 평균 4시간
- After: 즉시 감지 (1분) + 원인 파악 (1시간) = 평균 2시간 (로그가 있어서 원인 파악도 더 빠름)

## 주의할 점

1. **타임아웃을 너무 짧게 설정하지 않는다.** 작업의 최대 예상 실행 시간보다 20~30% 여유를 둔다. 너무 짧으면 정상 작업도 타임아웃으로 종료된다.

2. **CloudWatch 로그 비용을 고려한다.** 모든 크론 작업의 시작/종료 로그를 기록하면 로그 양이 늘어난다. 로그 보관 기간을 7일 정도로 제한했다.

3. **재시도 횟수를 제한한다.** 무한 재시도는 문제를 숨길 수 있다. 3회 정도가 적당하다.

4. **알림 피로도를 주의한다.** 사소한 경고까지 메신저로 보내면 알림을 무시하게 된다. 실제 장애(타임아웃, 재시도 실패)만 알림을 보낸다.

## 배운 점

1. **모니터링이 없으면 문제가 있어도 모른다.** 크론 작업은 조용히 실패한다. 사용자가 발견하기 전에 먼저 알아야 한다.

2. **타임아웃은 필수다.** 무한 대기는 서버를 멈춘다. 모든 외부 호출(DB, API, 파일 I/O)에 타임아웃을 설정한다.

3. **재시도 로직은 간단하지만 효과적이다.** 일시적 네트워크 오류는 재시도만으로 해결된다. 3건 중 3건이 재시도로 복구됐다.

4. **로그는 구조화해서 남긴다.** `[START]`, `[SUCCESS]`, `[ERROR]`, `[TIMEOUT]` 같은 패턴을 일관되게 사용하면 CloudWatch 메트릭 필터로 쉽게 감지할 수 있다.

## 마무리

크론 작업은 백그라운드에서 조용히 실행되기 때문에, 문제가 생겨도 알기 어렵다. 헬스체크 자동화는 복잡하지 않다. 시작/종료 로그를 남기고, 타임아웃을 설정하고, CloudWatch로 감시하면 된다. 이 간단한 시스템이 장애 대응 시간을 절반으로 줄였다.
