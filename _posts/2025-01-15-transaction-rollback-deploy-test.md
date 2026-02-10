---
title: "배포 전 안전장치: PHP 테스트로 트랜잭션 롤백 검증"
date: 2025-01-15
categories: [Backend, Testing]
tags: [PHP, PHPUnit, Transaction, Deploy]
---

컨트롤러에 모든 로직이 집중된 레거시 코드를 리팩토링할 때, 배포 전 운영 DB로 검증이 필요했다. PHP 테스트 코드에 트랜잭션 롤백을 추가해 데이터 오염 없이 배포 검증을 자동화한 과정을 정리한다.

## 배경

학원 운영관리 시스템의 PHP 레거시 코드는 컨트롤러에 모든 비즈니스 로직이 집중되어 있었다.

**문제점:**
- 컨트롤러 메서드 하나에 수백 줄의 로직
- DB 쿼리, 비즈니스 로직, 응답 생성이 모두 섞여 있음
- 테스트 코드 없음

Service-Repository 패턴으로 리팩토링하기로 했지만, 배포 전 운영 DB에서 실제 동작을 확인해야 했다.

## 문제 발생

**고민:**
- 로컬 환경만으로는 운영 환경 이슈를 발견하기 어려움
- 스테이징 환경이 없어 운영 DB로 직접 테스트해야 함
- 운영 DB로 테스트하면 실제 데이터가 저장될 위험

## 해결 방법

PHP 테스트 코드를 작성하고, 트랜잭션 롤백을 추가했다. 테스트가 성공하면 배포되게 설정했다.

**핵심:**
1. 목 데이터로 실제 API 테스트
2. 테스트 종료 시 자동으로 트랜잭션 롤백
3. 테스트 성공하면 배포 진행

### 테스트 코드

**DatabaseTransactions 트레이트 활용:**

```php
<?php

namespace Tests\Feature;

use Tests\TestCase;
use Illuminate\Foundation\Testing\DatabaseTransactions;

class StudentApiTest extends TestCase
{
    use DatabaseTransactions; // 테스트 종료 시 자동 롤백

    /**
     * 학생 등록 API 테스트
     */
    public function test_student_registration()
    {
        // 목 데이터
        $data = [
            'name' => '홍길동_TEST',
            'phone' => '010-1234-5678',
            'email' => 'test@example.com',
            'parent_phone' => '010-9876-5432',
        ];

        // API 호출
        $response = $this->postJson('/api/students', $data);

        // 응답 검증
        $response->assertStatus(201)
                 ->assertJson([
                     'message' => '학생 등록 완료',
                 ]);

        // DB 확인
        $this->assertDatabaseHas('students', [
            'name' => $data['name'],
            'phone' => $data['phone'],
        ]);
    }

    /**
     * 수업 배정 API 테스트
     */
    public function test_class_assignment()
    {
        // 목 데이터
        $student = [
            'name' => '김철수_TEST',
            'phone' => '010-1111-2222',
        ];
        $class = [
            'name' => '수학 심화반',
            'start_time' => '14:00',
        ];

        // 학생 등록
        $studentResponse = $this->postJson('/api/students', $student);
        $studentId = $studentResponse->json('data.id');

        // 수업 생성
        $classResponse = $this->postJson('/api/classes', $class);
        $classId = $classResponse->json('data.id');

        // 수업 배정
        $assignmentData = [
            'student_id' => $studentId,
            'class_id' => $classId,
        ];
        $response = $this->postJson('/api/class-assignments', $assignmentData);

        // 검증
        $response->assertStatus(201);
        $this->assertDatabaseHas('class_assignments', $assignmentData);
    }

    /**
     * 출석 체크 API 테스트
     */
    public function test_attendance_check()
    {
        // 목 데이터
        $student = $this->postJson('/api/students', [
            'name' => '이영희_TEST',
            'phone' => '010-3333-4444',
        ])->json('data');

        $attendanceData = [
            'student_id' => $student['id'],
            'date' => '2025-01-15',
            'status' => 'present',
        ];

        // API 호출
        $response = $this->postJson('/api/attendances', $attendanceData);

        // 검증
        $response->assertStatus(201);
        $this->assertDatabaseHas('attendances', $attendanceData);
    }
}
```

**Laravel 없이 직접 트랜잭션 관리:**

```php
<?php

namespace Tests;

use PHPUnit\Framework\TestCase as BaseTestCase;

abstract class TestCase extends BaseTestCase
{
    protected $pdo;

    protected function setUp(): void
    {
        parent::setUp();

        // DB 연결
        $this->pdo = new PDO(
            'mysql:host=localhost;dbname=production',
            'user',
            'password'
        );

        // 트랜잭션 시작
        $this->pdo->beginTransaction();
    }

    protected function tearDown(): void
    {
        // 트랜잭션 롤백
        $this->pdo->rollBack();

        parent::tearDown();
    }
}
```

### Jenkins 파이프라인

**테스트 성공 시에만 배포:**

```groovy
pipeline {
    agent any

    stages {
        stage('Test') {
            steps {
                echo '=== 배포 전 검증 시작 ==='
                sh 'php vendor/bin/phpunit tests/Feature/StudentApiTest.php'
            }
        }

        stage('Deploy') {
            when {
                expression { currentBuild.result == null || currentBuild.result == 'SUCCESS' }
            }
            steps {
                echo '테스트 성공! 배포를 진행합니다.'
                sh 'git pull origin main'
                sh 'composer install --no-dev'
                sh 'php artisan migrate --force'
                sh 'php artisan config:cache'
            }
        }
    }

    post {
        failure {
            echo '테스트 실패! 배포를 중단합니다.'
        }
    }
}
```

**배포 방법:**
- Jenkins에서 재생 버튼 클릭
- Test 스테이지 자동 실행
- 테스트 성공 시 Deploy 스테이지 진행
- 테스트 실패 시 배포 중단

### 실행 결과

```
Started by user admin
Running in Durability level: MAX_SURVIVABILITY
[Pipeline] Start of Pipeline
[Pipeline] stage
[Pipeline] { (Test)
[Pipeline] echo
=== 배포 전 검증 시작 ===
[Pipeline] sh
+ php vendor/bin/phpunit tests/Feature/StudentApiTest.php
PHPUnit 9.5.10

...                                                                 3 / 3 (100%)

Time: 00:02.156, Memory: 18.00 MB

OK (3 tests, 9 assertions)
[Pipeline] }
[Pipeline] stage
[Pipeline] { (Deploy)
[Pipeline] echo
테스트 성공! 배포를 진행합니다.
[Pipeline] sh
+ git pull origin main
Already up to date.
[Pipeline] sh
+ composer install --no-dev
Installing dependencies from lock file
...
[Pipeline] }
[Pipeline] End of Pipeline
Finished: SUCCESS
```

## 결과

**성과:**
- 운영 DB로 배포 전 검증, 데이터 오염 없음
- 테스트 자동화로 배포 안정성 확보
- 테스트 실패 시 배포 자동 중단

**프로세스:**
1. 목 데이터로 실제 API 호출
2. 테스트 종료 시 자동 롤백
3. 테스트 성공 시에만 배포 진행

## 배운 점

### 1. 트랜잭션 롤백으로 안전한 검증

운영 DB로 테스트해도 트랜잭션 롤백으로 데이터 오염을 완전히 방지할 수 있다.

### 2. 테스트 자동화가 배포 안정성을 높인다

배포 전 자동으로 테스트가 실행되면 휴먼 에러를 방지할 수 있다.

### 3. 스테이징 환경이 없을 때의 대안

스테이징 환경 구축이 어려운 소규모 팀에서 실용적인 방법이다.
