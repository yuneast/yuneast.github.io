---
title: "프리랜서 시절, '내 작업만 끝내면 돼' 코드와의 전쟁"
date: 2017-01-15
categories: [회고]
tags: [프리랜서, 레거시 코드, 리팩토링, 외주 개발]
---

프리랜서로 일하면서 가장 힘들었던 건 다른 개발자가 남긴 코드였다. 이 글에서는 외주 특성상 발생하는 레거시 코드 문제와 내가 어떻게 대응했는지 정리한다.

## 문제 발생

다른 개발자가 남긴 코드는 유지보수가 불가능했다.

**전형적인 외주 코드:**

```php
// 결제 처리
function pay($user_id, $amount, $type) {
    if ($type == 1) {
        // 카드 결제
        $result = card_pay($user_id, $amount);
        if ($result == "success") {
            update_db($user_id, $amount, 1);
            send_mail($user_id, "결제 완료");
            return "ok";
        }
    } else if ($type == 2) {
        // 계좌이체
        $result = bank_pay($user_id, $amount);
        if ($result == "ok") {
            update_db($user_id, $amount, 2);
            send_mail($user_id, "결제 완료");
            return "ok";
        }
    } else if ($type == 3) {
        // 포인트 결제
        $point = get_point($user_id);
        if ($point >= $amount) {
            $result = point_pay($user_id, $amount);
            if ($result == "success") {
                update_db($user_id, $amount, 3);
                send_mail($user_id, "결제 완료");
                return "ok";
            }
        }
    }
    return "fail";
}
```

**문제:**
- 타입 번호 1, 2, 3이 뭔지 주석도 없음
- 결제 성공 응답이 "success", "ok" 제각각
- 중복 코드 (update_db, send_mail 반복)
- 에러 처리 없음 (실패 시 로그도 없음)
- 함수명에서 뭘 하는지 유추해야 함

이런 코드가 수백 개 파일에 걸쳐 있었다.

## 분석: 왜 이런 코드가 나올까?

**1. 외주 특성상 장기 유지보수 책임이 없음**

정규직은 내가 작성한 코드를 내가 유지보수한다. 버그 나면 내가 고치고, 기능 추가 요청 들어오면 내가 수정한다.

하지만 외주는 다르다. 계약 기간 끝나면 다른 프로젝트로 이동한다. 내 코드를 다른 사람이 보든 말든 상관없다.

**2. 빠른 납품이 최우선**

외주는 기간이 짧다. 1-3개월 안에 끝내야 한다. 리팩토링할 시간 없다. 일단 돌아가게만 만들고 납품한다.

**3. 코드 리뷰가 없음**

회사에서는 PR 올리면 다른 개발자가 리뷰한다. 하지만 외주는 각자 작업하고 합치기만 한다. 내 코드를 누가 봐주지 않는다.

**4. 문서화 동기 부족**

문서 작성해봤자 나중에 내가 볼 일 없다. 주석 달아봤자 인정받지 못한다. 그냥 기능 구현만 하고 끝낸다.

## 내가 마주한 현실

**프로젝트 A: Laravel 쇼핑몰**

이전 개발자가 남긴 코드:
- Controller에 SQL 쿼리 직접 작성 (ORM 안 씀)
- 비즈니스 로직이 View 파일 안에 섞여 있음
- .env 파일에 DB 비밀번호 하드코딩
- 에러 발생 시 백지 화면만 뜸 (로그 없음)

**프로젝트 B: Python 부동산 자동화**

- 함수명이 func1, func2, func3
- 변수명이 a, b, c, tmp
- 300줄짜리 함수 (들여쓰기 8단계)
- 전역변수 남발

**프로젝트 C: React 렌트카 플랫폼**

- 모든 상태를 App.js에서 관리
- props drilling 10단계
- useEffect 안에서 또 useEffect 호출
- API 호출 로직이 컴포넌트마다 중복

## 해결: 어떻게 버텼나

### 1. 최소한의 리팩토링

납품 기한은 지켜야 하니까 전체 리팩토링은 불가능했다. 내가 건드리는 부분만 최소한으로 정리했다.

**Before:**
```php
function pay($user_id, $amount, $type) {
    if ($type == 1) {
        $result = card_pay($user_id, $amount);
        if ($result == "success") {
            update_db($user_id, $amount, 1);
            send_mail($user_id, "결제 완료");
            return "ok";
        }
    }
    // ... 반복
}
```

**After:**
```php
class PaymentService {
    const TYPE_CARD = 1;
    const TYPE_BANK = 2;
    const TYPE_POINT = 3;

    public function process($userId, $amount, $type) {
        $strategy = $this->getPaymentStrategy($type);
        $result = $strategy->pay($userId, $amount);

        if ($result->isSuccess()) {
            $this->saveTransaction($userId, $amount, $type);
            $this->sendNotification($userId);
            return Response::success();
        }

        return Response::fail($result->getErrorMessage());
    }
}
```

**개선점:**
- 타입을 상수로 정의
- Strategy 패턴으로 분기 제거
- 중복 코드 제거
- 에러 메시지 반환

### 2. 주석 대신 코드로 설명

주석은 코드와 동기화가 안 맞는 경우가 많다. 함수명, 변수명으로 의도를 드러냈다.

**Before:**
```python
def func1(a, b):
    # 결제 금액 계산
    c = a * b
    # 할인 적용
    if c > 100000:
        c = c * 0.9
    return c
```

**After:**
```python
def calculate_payment_with_discount(price: int, quantity: int) -> int:
    total_amount = price * quantity

    if total_amount > 100000:
        return apply_bulk_discount(total_amount)

    return total_amount

def apply_bulk_discount(amount: int) -> int:
    DISCOUNT_RATE = 0.9
    return int(amount * DISCOUNT_RATE)
```

### 3. 레이어 분리

Controller, Service, Repository 패턴을 최소한으로라도 적용했다.

**Before (Controller에 모든 로직):**
```php
public function store(Request $request) {
    $user = User::find($request->user_id);
    $point = $user->point;
    if ($point < $request->amount) {
        return response()->json(['error' => '포인트 부족']);
    }
    $user->point = $point - $request->amount;
    $user->save();

    DB::table('orders')->insert([
        'user_id' => $user->id,
        'amount' => $request->amount,
        'created_at' => now()
    ]);

    return response()->json(['success' => true]);
}
```

**After (레이어 분리):**
```php
// Controller
public function store(CreateOrderRequest $request) {
    $result = $this->orderService->create($request->validated());
    return response()->json($result);
}

// Service
public function create(array $data) {
    DB::beginTransaction();
    try {
        $this->pointService->deduct($data['user_id'], $data['amount']);
        $order = $this->orderRepository->create($data);
        DB::commit();
        return ['success' => true, 'order_id' => $order->id];
    } catch (Exception $e) {
        DB::rollBack();
        Log::error('주문 생성 실패', ['error' => $e->getMessage()]);
        return ['success' => false, 'message' => '주문 생성 실패'];
    }
}
```

### 4. 에러 처리 추가

외주 코드는 에러 처리가 없는 경우가 많다. 최소한 try-catch와 로그는 추가했다.

```php
try {
    $result = $this->externalApi->call($params);
} catch (ApiException $e) {
    Log::error('외부 API 호출 실패', [
        'params' => $params,
        'error' => $e->getMessage()
    ]);
    throw new ServiceException('외부 API 호출 실패');
}
```

## 결과: 배운 점

**1. 코드는 작성하는 시간보다 읽는 시간이 더 길다**

외주는 빨리 끝내고 나가는 게 이득이라고 생각한다. 하지만 나중에 그 코드를 수정할 사람은 10배 더 고생한다.

**2. "일단 돌아가게" 만드는 건 절반만 끝낸 것**

기능 구현은 절반이다. 나머지 절반은 유지보수 가능하게 만드는 것이다.

**3. 리팩토링은 선택이 아니라 필수**

리팩토링 안 하면 나중에 기능 하나 추가하는 데 10배 시간이 걸린다. 초기에 구조를 잡는 게 장기적으로 이득이다.

**4. 회사 경력의 중요성**

프리랜서로만 일하면 코드 리뷰, 협업, 장기 유지보수 경험을 쌓기 어렵다. 회사에 들어가야 제대로 된 개발 프로세스를 배울 수 있다.

## 지금의 나

프리랜서 시절 레거시 코드와 싸우면서 배운 것:
- 코드는 작성자만 보는 게 아니다
- 주석보다 명확한 네이밍이 낫다
- 레이어 분리는 선택이 아니라 필수다
- 에러 처리 없는 코드는 시한폭탄이다

지금은 회사에서 일하면서 코드 리뷰, PR, 장기 유지보수를 경험하고 있다. 내가 작성한 코드를 1년 뒤에도 내가 고친다. 그래서 지금은 "내 작업만 끝내면 돼"가 아니라 "나중에 내가 봐도 이해할 수 있게" 작성한다.

프리랜서 시절의 고통이 지금의 나를 만들었다.
