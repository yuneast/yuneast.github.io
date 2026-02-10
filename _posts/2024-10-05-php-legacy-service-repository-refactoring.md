---
title: "PHP 레거시 코드를 Service-Repository 패턴으로 리팩토링한 이야기"
date: 2024-10-05
categories: [backend]
tags: [php, refactoring, design-pattern, legacy, clean-architecture]
---

"이 코드 누가 짰어요?" 레거시 코드를 처음 봤을 때 누구나 한 번쯤 하는 말이다. 하지만 중요한 건 누가 짰는지가 아니라, 지금 어떻게 개선할 것인지다. 학원 ERP 서비스의 PHP 레거시 코드를 Service-Repository 패턴으로 리팩토링한 과정을 정리한다.

## 기존 코드의 문제

입사 후 처음 마주한 PHP 코드의 구조는 이랬다:

```php
// controllers/StudentController.php
class StudentController {
    public function getStudentList($request) {
        $db = Database::getInstance();

        $academyId = $request->get('academy_id');
        $keyword = $request->get('keyword');

        // 컨트롤러에서 직접 SQL 작성
        $sql = "SELECT s.*, c.class_name, p.parent_name, p.parent_phone
                FROM students s
                LEFT JOIN classes c ON s.class_id = c.id
                LEFT JOIN parents p ON s.parent_id = p.id
                WHERE s.academy_id = '$academyId'";

        if ($keyword) {
            $sql .= " AND (s.name LIKE '%$keyword%' OR p.parent_phone LIKE '%$keyword%')";
        }

        $students = $db->query($sql);

        // 컨트롤러에서 비즈니스 로직 처리
        foreach ($students as &$student) {
            $attendanceSql = "SELECT COUNT(*) as cnt FROM attendance
                             WHERE student_id = '{$student['id']}'
                             AND date = CURDATE()";
            $attendance = $db->query($attendanceSql);
            $student['is_attended'] = $attendance[0]['cnt'] > 0;

            // SMS 발송 로직까지 컨트롤러에...
            if (!$student['is_attended'] && date('H') >= 10) {
                $this->sendAbsenceNotification($student);
            }
        }

        return response()->json($students);
    }
}
```

문제를 정리하면:

1. **컨트롤러에 SQL이 직접 박혀있다** - 쿼리 수정하려면 컨트롤러를 건드려야 한다
2. **문자열 보간으로 SQL 작성** - SQL Injection 취약점
3. **컨트롤러에서 비즈니스 로직 처리** - 출석 확인, 알림 발송까지 컨트롤러가 담당
4. **루프 안에서 쿼리 실행** - N+1 문제 (학생 100명이면 출석 쿼리 100번)
5. **테스트 불가능** - 모든 게 한 메서드에 엉겨있어서 단위 테스트가 안 된다

## 리팩토링 방향

한 번에 전체를 뒤집으면 리스크가 크다. 단계적으로 접근했다:

1단계: Repository 분리 (SQL을 한 곳에)
2단계: Service 분리 (비즈니스 로직 분리)
3단계: N+1 문제 해결
4단계: SQL Injection 방어 (Prepared Statement)

## 1단계: Repository 분리

SQL을 Repository 클래스로 옮겼다. 이 단계에서는 SQL 자체를 변경하지 않고, 위치만 옮기는 데 집중했다.

```php
// repositories/StudentRepository.php
class StudentRepository {
    private $db;

    public function __construct() {
        $this->db = Database::getInstance();
    }

    public function findByAcademyId(int $academyId, ?string $keyword = null): array {
        $sql = "SELECT s.*, c.class_name, p.parent_name, p.parent_phone
                FROM students s
                LEFT JOIN classes c ON s.class_id = c.id
                LEFT JOIN parents p ON s.parent_id = p.id
                WHERE s.academy_id = ?";

        $params = [$academyId];

        if ($keyword) {
            $sql .= " AND (s.name LIKE ? OR p.parent_phone LIKE ?)";
            $params[] = "%$keyword%";
            $params[] = "%$keyword%";
        }

        return $this->db->prepare($sql, $params);
    }
}
```

```php
// repositories/AttendanceRepository.php
class AttendanceRepository {
    private $db;

    public function __construct() {
        $this->db = Database::getInstance();
    }

    public function findTodayAttendedStudentIds(array $studentIds): array {
        if (empty($studentIds)) return [];

        $placeholders = implode(',', array_fill(0, count($studentIds), '?'));
        $sql = "SELECT student_id FROM attendance
                WHERE student_id IN ($placeholders)
                AND date = CURDATE()";

        $rows = $this->db->prepare($sql, $studentIds);

        return array_column($rows, 'student_id');
    }
}
```

여기서 N+1 문제도 같이 해결했다. 루프 안에서 학생별로 출석 쿼리를 날리던 걸, **학생 ID 목록으로 한 번에 조회**하도록 변경했다.

## 2단계: Service 분리

비즈니스 로직을 Service 클래스로 옮겼다.

```php
// services/StudentService.php
class StudentService {
    private StudentRepository $studentRepo;
    private AttendanceRepository $attendanceRepo;
    private NotificationService $notificationService;

    public function __construct(
        StudentRepository $studentRepo,
        AttendanceRepository $attendanceRepo,
        NotificationService $notificationService
    ) {
        $this->studentRepo = $studentRepo;
        $this->attendanceRepo = $attendanceRepo;
        $this->notificationService = $notificationService;
    }

    public function getStudentListWithAttendance(int $academyId, ?string $keyword): array {
        $students = $this->studentRepo->findByAcademyId($academyId, $keyword);

        if (empty($students)) return [];

        // 출석 정보 일괄 조회 (N+1 → 1)
        $studentIds = array_column($students, 'id');
        $attendedIds = $this->attendanceRepo->findTodayAttendedStudentIds($studentIds);
        $attendedIdSet = array_flip($attendedIds);

        foreach ($students as &$student) {
            $student['is_attended'] = isset($attendedIdSet[$student['id']]);
        }

        return $students;
    }
}
```

## 3단계: 컨트롤러 정리

컨트롤러는 요청을 받아서 Service에 위임하고 응답만 내려준다.

```php
// controllers/StudentController.php
class StudentController {
    private StudentService $studentService;

    public function __construct(StudentService $studentService) {
        $this->studentService = $studentService;
    }

    public function getStudentList($request) {
        $academyId = $request->get('academy_id');
        $keyword = $request->get('keyword');

        $students = $this->studentService->getStudentListWithAttendance(
            $academyId,
            $keyword
        );

        return response()->json($students);
    }
}
```

## Before / After 비교

| 항목 | Before | After |
|------|--------|-------|
| 컨트롤러 역할 | SQL + 비즈니스 로직 + 응답 | 요청 → Service 위임 → 응답 |
| SQL 위치 | 컨트롤러 여기저기 | Repository에 집중 |
| SQL Injection | 취약 (문자열 보간) | Prepared Statement |
| N+1 문제 | 학생 수만큼 쿼리 | IN절로 1번 |
| 테스트 가능성 | 불가 | Service, Repository 각각 테스트 가능 |

## 리팩토링하면서 지킨 원칙

1. **동작하는 코드를 먼저 만들고, 그 다음에 리팩토링한다.** 기존 기능이 깨지지 않는 게 최우선이다. Repository로 옮기고 → 동작 확인 → Service로 옮기고 → 동작 확인. 이 순서를 지켰다.

2. **한 번에 하나만 바꾼다.** SQL을 옮기면서 동시에 쿼리를 최적화하면 문제가 생겼을 때 원인 파악이 어렵다.

3. **점진적으로 진행한다.** 전체 코드를 한 번에 리팩토링하지 않았다. 새 기능 개발이나 버그 수정과 함께 해당 영역의 코드를 리팩토링하는 방식으로 자연스럽게 진행했다.

## 마무리

레거시 코드 리팩토링은 화려하지 않다. 사용자에게 보이는 변화도 거의 없다. 하지만 코드를 유지보수하는 개발자에게는 큰 차이를 만든다. 다음에 이 코드를 수정할 사람이 "이 코드 누가 짰어요?"라고 묻지 않게 되는 것, 그게 리팩토링의 가치라고 생각한다.
