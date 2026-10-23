---
title: "결제사가 REST를 안 줬다 - SOAP/WSDL로 해외 PG 연동하기"
date: 2022-12-10
categories: [backend]
tags: [soap, wsdl, zeep, payment, python, fastapi, integration]
---

"결제 연동 문서 좀 주세요"라고 했더니 `.asmx?wsdl` 링크가 왔다. 2022년에 SOAP을 만질 일이 있을 거라고는 생각해본 적이 없었다.

## 배경

필리핀 시장을 대상으로 하는 공동주택 관리 서비스를 만들고 있었다. 입주민이 앱으로 관리비를 확인하는 기능까지는 만들었는데, 정작 낼 수가 없었다. 현지 입주민이 쓸 수 있는 결제 수단이 붙어 있지 않았기 때문이다.

국내 PG는 쓸 수 없었다. 결제 통화가 페소(PHP)고, 현지 사용자가 쓰는 결제 수단(현지 은행 계좌이체, 편의점 수납 등)을 지원해야 했다. 현지에서 많이 쓰는 PG사를 붙이기로 했다.

## 문제: 문서가 WSDL이었다

연동 문서를 받아보니 REST API가 아니었다. SOAP 웹서비스였고, 엔드포인트는 이런 형태였다.

```
https://gw.example-pg.ph/PaymentWebService/MerchantService.asmx?wsdl
```

`.asmx`는 ASP.NET 웹서비스 확장자다. 요청은 XML 봉투(Envelope)로 감싸서 보내고, 응답도 XML로 돌아온다. JSON을 주고받는 것과는 전제가 다르다.

처음엔 `requests`로 XML을 직접 만들어 POST할 생각을 했다. 그러다 멈췄다. SOAP은 WSDL에 타입과 메서드 시그니처가 정의되어 있고, 그걸 손으로 베껴 쓰는 건 틀리기 쉽다. 파라미터 하나만 순서가 어긋나도 서버는 이유를 알려주지 않고 그냥 실패한다.

## 해결: zeep으로 WSDL을 읽게 하기

Python에는 `zeep`이라는 SOAP 클라이언트가 있다. WSDL URL을 주면 스펙을 파싱해서 메서드를 객체처럼 호출할 수 있게 해준다.

```python
from zeep import Client

class PaymentGateway:
    def __init__(self, merchant_id: str, secret_key: str, production: bool = False):
        self.merchant_id = merchant_id
        self.secret_key = secret_key
        self.base_url = PRODUCTION_URL if production else TEST_URL

    def create_payment_url(self, transaction_id, amount, description, email, extra=None):
        wsdl = f"{self.base_url}/PaymentWebService/MerchantService.asmx?wsdl"
        client = Client(wsdl)

        token = client.service.GetTxnToken(
            merchantId=self.merchant_id,
            password=self.secret_key,
            merchantTxnId=transaction_id,
            amount=self.format_amount(amount),
            ccy="PHP",
            description=description,
            email=email,
            param1=json.dumps(extra) if extra else "",
            param2="",
        )

        return f"{self.base_url}/Pay.aspx?tokenid={token}"
```

`client.service.GetTxnToken(...)`이 전부다. XML 봉투를 만들고 보내고 파싱하는 건 zeep이 한다. WSDL에 정의된 이름을 그대로 키워드 인자로 쓰면 되니, 파라미터 순서를 틀릴 일도 없다.

### 금액 포맷에서 한 번 막혔다

처음에 금액을 그냥 넘겼다가 계속 실패했다. 이 PG는 소수점 둘째 자리까지 고정된 문자열을 요구했다. `1000`이 아니라 `1000.00`이어야 한다.

```python
from decimal import Decimal

def format_amount(self, amount) -> str:
    return "%.2f" % Decimal(amount)
```

`float`을 쓰지 않고 `Decimal`을 거친 이유는 금액이기 때문이다. 부동소수점으로 더하고 빼다 보면 `0.01`이 어긋나는 순간이 온다. 결제 금액에서 그건 사고다.

### 서명 값 생성

요청 위변조를 막기 위해 파라미터를 정해진 순서로 이어 붙이고 해시를 만들어 보내야 했다.

```python
def digest(self, transaction_id, amount, currency, description, email) -> str:
    raw = ":".join([
        self.merchant_id,
        transaction_id,
        amount,
        currency,
        description,
        email,
        self.secret_key,
    ])
    return hashlib.sha1(raw.encode()).hexdigest()
```

**순서가 스펙에 적힌 그대로여야 한다.** 여기서 한 칸 밀리면 서버는 "서명 불일치"가 아니라 그냥 거절한다. 디버깅할 때 제일 시간을 많이 쓴 부분이다.

## 삽질한 것들

**1. WSDL을 매 호출마다 받아오고 있었다.** `Client(wsdl)`를 메서드 안에서 생성하니 결제 요청마다 WSDL을 다시 내려받는다. 호출이 잦지 않아 당장 문제는 없었지만, 클라이언트를 재사용하도록 바꾸는 게 맞다. SOAP 클라이언트 생성은 JSON 요청 한 번과는 비용이 다르다.

**2. 테스트 키와 라이브 키를 코드에 같이 두고 주석으로 전환했다.** 지금 보면 부끄러운 부분이다. 환경변수로 빼고 운영/테스트 분기를 설정으로 처리해야 한다. 소스에 결제 시크릿을 두면 커밋 히스토리에 영원히 남는다. 지워도 히스토리에는 남아 있다.

**3. 실패 응답이 불친절하다.** REST API처럼 `{"error": "..."}`를 기대하면 안 된다. 토큰이 비어서 돌아오거나 예외가 던져지는 식이라, 어디서 틀렸는지 응답만 보고는 알 수 없다. 스펙 문서와 내가 보낸 값을 한 줄씩 대조하는 수밖에 없었다.

## 마무리

레거시 프로토콜을 만나는 건 생각보다 흔하다. 특히 금융·결제·공공처럼 오래 운영된 영역은 바깥에서 보기엔 낡아 보여도 그 위에서 돈이 돌고 있어서 쉽게 바뀌지 않는다.

이때 배운 건, **낡은 프로토콜을 직접 손으로 구현하려 들지 말자**는 것이다. SOAP이든 뭐든 그 시대에 만들어진 성숙한 클라이언트 라이브러리가 거의 있다. zeep에게 WSDL을 읽히고 나는 비즈니스 로직만 쓰면 되는 일을, XML 문자열을 조립하면서 며칠 태울 수도 있었다.

그리고 금액은 `Decimal`로 다루자. 이건 프로토콜과 무관하게 항상 맞는 말이다.
