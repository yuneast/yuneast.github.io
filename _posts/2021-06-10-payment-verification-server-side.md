---
title: "결제 금액을 클라이언트가 보내면 안 되는 이유 - 서버 재조회로 위변조 막기"
date: 2021-06-10
categories: [backend]
tags: [payment, pg, security, django, iamport, validation]
---

결제 연동을 처음 붙일 때 가장 자연스러워 보이는 흐름이 있다. 결제창이 성공을 알려주면, 그 결과를 서버에 보내서 저장하는 것이다. 이게 왜 위험한지는 개발자 도구를 한 번 열어보면 알게 된다.

## 배경

인플루언서와 광고주를 매칭하는 플랫폼을 혼자 만들어 운영하고 있었다. 광고주가 캠페인을 등록할 때 결제를 하고, 결제가 완료되면 캠페인이 공개되는 구조다. 국내 PG 연동은 아임포트(현 포트원)를 썼다.

수익 구조는 두 가지였다.

- **캠페인 건별 결제** — 광고주가 캠페인을 올릴 때마다 결제
- **구독 + 크레딧** — 무료 플랜은 블로거 지수 조회권을 차감, 유료 플랜은 무제한

둘 다 "결제가 됐는지"를 서버가 정확히 알아야 동작하는 기능이다.

## 처음 떠올린 흐름의 문제

아임포트 결제창은 콜백으로 결제 결과를 준다. `imp_uid`(PG가 발급한 거래 번호), `merchant_uid`(내가 만든 주문 번호), 그리고 금액과 성공 여부가 함께 온다.

```javascript
IMP.request_pay(params, function (rsp) {
  if (rsp.success) {
    // 이 값들을 서버에 그대로 보내서 저장하면?
    axios.post('/payment/complete', {
      imp_uid: rsp.imp_uid,
      merchant_uid: rsp.merchant_uid,
      amount: rsp.paid_amount,   // ← 여기가 문제
      status: 'paid',            // ← 여기도
    });
  }
});
```

이 흐름의 문제는 **금액과 상태를 클라이언트가 말해준다**는 것이다. 브라우저에서 돌아가는 코드는 사용자가 고칠 수 있다. 개발자 도구로 요청을 잡아 `amount`를 `100`으로 바꿔 보내면, 서버는 100원이 결제됐다고 믿고 캠페인을 열어준다.

결제창에서 실제로 결제가 안 됐어도 `success: true`로 꾸며 보낼 수 있다. 서버가 PG에게 직접 물어보지 않으면 확인할 방법이 없다.

## 고친 흐름: 서버가 PG에게 직접 묻는다

클라이언트에서는 **거래 번호만** 받는다. 금액과 상태는 받지 않고, 받아도 쓰지 않는다.

```python
from iamport import Iamport

iamport = Iamport(imp_key=IMP_KEY, imp_secret=IMP_SECRET)

def payment_complete(request):
    merchant_uid = request.POST.get("merchant_uid")

    # 1. PG에 직접 조회 — 클라이언트가 보낸 금액/상태는 쓰지 않는다
    pg_data = iamport.find(merchant_uid=merchant_uid)

    # 2. 서버가 알고 있는 "청구해야 할 금액"을 꺼낸다
    order = Order.objects.get(merchant_uid=merchant_uid)

    # 3. 대조
    if pg_data["status"] != "paid":
        return JsonResponse({"result": "fail", "reason": "결제 미완료"})

    if int(pg_data["amount"]) != int(order.expected_amount):
        # 금액 불일치 — 기록을 남기고 거절한다
        log_payment_mismatch(order, pg_data)
        return JsonResponse({"result": "fail", "reason": "금액 불일치"})

    # 4. 대조를 통과한 뒤에야 활성화
    Payment.objects.create(
        merchant_uid=merchant_uid,
        imp_uid=pg_data["imp_uid"],
        amount=pg_data["amount"],
    )
    order.activate()

    return JsonResponse({"result": "success"})
```

핵심은 **3번과 4번의 순서**다. 대조가 끝나기 전에는 아무것도 활성화하지 않는다.

그리고 비교 대상이 되는 금액(`order.expected_amount`)은 **주문을 만들 때 서버가 계산해서 저장해둔 값**이다. 클라이언트가 보낸 값과 비교하면 의미가 없다. 사용자가 둘 다 고칠 수 있으니 같은 값이 나올 뿐이다.

## 같이 챙겨야 했던 것들

**1. `merchant_uid`는 서버가 만든다.** 주문 번호를 클라이언트가 만들어 보내면 남의 주문 번호를 끼워넣을 수 있다. 서버에서 발급하고 유니크 제약을 걸었다.

```python
merchant_uid = models.CharField(max_length=255, unique=True)
```

유니크 제약은 **같은 결제가 두 번 반영되는 것**도 막아준다. 네트워크가 끊겨 콜백이 두 번 오는 일이 실제로 있다.

**2. 중복 호출을 전제로 짠다.** 결제 완료 처리는 여러 번 불려도 결과가 같아야 한다. 이미 처리된 주문이면 성공을 그대로 돌려주고, 크레딧을 두 번 지급하지 않는다.

**3. 금액 불일치는 거절하고 기록한다.** 그냥 실패 응답만 주고 끝내면 누가 시도했는지 알 수 없다. 로그를 남겨두면 나중에 패턴이 보인다.

**4. 결제 취소·환불 경로도 같은 원칙이다.** 취소 요청도 클라이언트 말을 믿지 말고 PG 조회로 상태를 확인한 뒤 반영한다.

## 배운 것

결제는 **클라이언트를 신뢰할 수 없는 입력**으로 다루는 연습이 되는 영역이다. 평소에 폼 검증을 서버에서 한 번 더 하는 것과 같은 이야기인데, 돈이 걸려 있으니 결과가 분명하게 드러난다.

규칙으로 정리하면 이렇다. **클라이언트에서는 "무엇에 대한 결제인지"만 받고, "얼마가 결제됐는지"는 서버가 PG에게 직접 묻는다.** 이 한 줄만 지켜도 금액 위변조 경로는 막힌다.

혼자 만들던 서비스라 리뷰해줄 사람이 없었다. 결제를 붙이면서는 PG사 개발 문서의 "주의사항" 항목을 처음부터 끝까지 읽었다. 거기에 이런 내용이 다 적혀 있다. 문서를 읽는 게 가장 빠른 코드 리뷰였다.
