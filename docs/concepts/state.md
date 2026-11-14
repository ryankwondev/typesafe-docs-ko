# State

> state가 무엇인지, 어떻게 구조화하는지, System One 모델에 필요한 컨텍스트를 어떻게 전달하는지 설명합니다.

**State**(상태)는 System One 모델에 평가를 맡기는 내용입니다. 고객 지원 메시지나 글의 한 단락, 사진일 수도 있고, 애플리케이션의 현재 상태일 수도 있습니다. API 요청의 `state` 필드에 담아, 답을 얻고 싶은 질문과 함께 보냅니다.

요청 하나는 state 하나를 하나 이상의 질문으로 평가합니다. 모든 질문은 같은 state를 보며, 서로 독립적으로 평가됩니다. 한 요청 안에 [Choice](../primitives/choice.md), [Score](../primitives/score.md), [Noul](../primitives/noul.md), [Box](../primitives/box.md) 질문을 섞어 쓸 수 있습니다.

## State는 단순한 문자열일 수도, 구조화된 JSON 값일 수도 있습니다

가장 단순한 state는 일반 문자열입니다.

```python
state = "My card was charged twice."
```

state는 JSON 객체나 배열일 수도 있습니다. 여기에 관련 컨텍스트, 예시, 그 밖에 모델이 질문에 답하는 데 도움이 되는 정보를 담습니다. 전문가 패널에게 판단을 구하기 전에 미리 건네는 자료라고 생각하면 됩니다. Python에서는 해당하는 문자열, 딕셔너리, 리스트를 `client.system_one(state=...)`에 그대로 넘깁니다.

| 형식 | 적합한 용도 | 예시 |
| - | - | - |
| 문자열 | 메시지, 기사, 글의 한 단락 | `"My card was charged twice."` |
| 객체 | 이름이 붙은 필드, 서로 관련된 레코드, 애플리케이션 상태 | `{"message": "My card was charged twice.", "order_id": "A-104"}` |
| 배열 | 순서가 있는 메시지나 레코드 | `["Hi", "My customer number is TS1337.", "My card was charged twice."]` |

대부분의 요청에는 객체를 쓰세요. state의 각 부분에 내용을 설명하는 이름이 붙고, 부분 사이의 관계도 분명하게 유지됩니다. 문자열은 사용 사례가 단순해서 텍스트 하나만 있으면 될 때 적합합니다.

> [!NOTE]
>
> state는 문자열, JSON 객체, 배열 중 하나이며 [이미지](../images.md)를 포함할 수 있습니다. 오디오와 비디오는 (아직) 기본 지원하지 않습니다.

```json title="state로 쓰는 고객 지원 대화"
{
  "ticket": {
    "subject": "Duplicate charge",
    "messages": [
      {"from": "customer", "text": "I was charged twice for order A-104. Please refund the duplicate."},
      {"from": "support", "text": "We are checking the charges."}
    ]
  },
  "order": {
    "id": "A-104",
    "charges": [
      {"amount_usd": 49, "status": "captured"},
      {"amount_usd": 49, "status": "captured"}
    ]
  },
  "refund_policy": "Duplicate charges are eligible for a refund."
}
```

이 객체는 대화, 주문, 정책을 모두 담고 있지만 하나의 state입니다. 여러 부분을 서로 비교해야 내릴 수 있는 판단이라면 관련 정보를 한데 묶어 넣으세요.

## 내용과 질문을 분리하세요

state에는 내용과 이를 뒷받침하는 사실을 담습니다. [질문](../primitives.md)은 그 자료를 두고 모델이 내려야 할 판단을 정의합니다. 예를 들어 환불 요청과 정책은 state에 두고, 고객이 환불을 요청했는지와 정책상 환불이 가능한지를 질문으로 묻습니다.

instructions(지시문), criteria(판단 기준), 질문 유형, 그리고 하나의 state에 여러 질문을 하는 방법은 [Primitives (Questions)](../primitives.md)를 참고하세요.

요청 스키마는 [API 레퍼런스](https://docs.typesafe.ai/api)에서, 설치 방법과 타입이 지정된 입력, 응답 처리는 [클라이언트 SDK](../sdk.md)에서 확인할 수 있습니다.


이 문서는 개발자 문서 플랫폼인 [Mintlify](https://mintlify.com)로 제작하고 호스팅합니다.
