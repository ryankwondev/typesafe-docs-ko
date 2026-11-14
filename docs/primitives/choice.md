# Choice

> Choice는 정해진 집합에서 선택지 하나를 고르는 System One 질문 유형입니다. 답에는 선택된 선택지, 각 선택지의 확률, confidence가 들어 있습니다.

Choice는 답이 고정된 선택지 집합 중 하나일 때 사용합니다. 예를 들어 어느 팀이 티켓을 처리할지, 상품이 어느 카테고리에 속하는지, 코드 조각이 어떤 언어로 쓰였는지 등입니다. 답이 스펙트럼 위의 한 위치라면 [Score](score.md)를 사용하세요. 예 또는 아니요라면 [Noul](noul.md)을 사용하세요. [질문 유형 선택하기](../primitives.md#choose-a-question-type)에서 세 가지를 모두 비교합니다.

Choice의 답은 `choice`에 담긴 선택된 선택지입니다. 모델은 `probabilities`에 모든 선택지의 확률을, 그리고 선택된 선택지에 대한 `confidence`(신뢰도) 값도 함께 반환합니다.

질문 예시:

```
"What programming language is this code written in"
  → options: python, javascript, typescript, go, rust, other

"What type of meeting is this based on the title and description"
  → options: standup, planning, retrospective, one on one, brainstorm, none of the above

"Which product category does this item belong to"
  → options: electronics, clothing, home garden, food and beverage
```

## 요청 구조

[TypeSafe API](https://docs.typesafe.ai/api)로 보내는 POST 요청 본문에는 정해진 구조가 있습니다. 최상위에는 필드가 세 개 있습니다. 평가할 내용인 `state`(상태), `model`, 그리고 여러분이 정한 질문 id를 질문 객체에 대응시키는 맵인 `questions`입니다. 각 Choice 질문에는 다음 필드가 있습니다.

* `type`: 항상 `"choice"`입니다.
* `instructions`: 모델이 답할 질문입니다.
* `criteria`: 답의 선택지를 담은 맵입니다. 각 키는 선택지 이름이고 각 값은 그 선택지에 대한 설명입니다.

아래 요청에서 state는 온라인 신발 가게에 들어온 지원 티켓이고, 질문은 어느 팀이 이 티켓을 처리해야 하는지입니다.

Playground에서 실행해 보기 →

질문 id는 여러분이 정하며, 여기서는 `department`입니다. 답은 같은 id 아래에 반환됩니다. 모델은 질문 id를 전혀 보지 못합니다. 선택지 이름과 설명은 둘 다 모델에 전달되므로, 선택지들이 서로 구별되도록 설명을 작성하세요.

[클라이언트 SDK](../sdk.md)는 타입이 지정된 질문을 제공합니다. Python에서는 같은 질문을 `Choice`로 씁니다.

```python
from typesafe_sdk import Choice, TypeSafeClient

with TypeSafeClient() as client:
    response = client.system_one(
        state="My running shoes arrived in the wrong size. Can I swap them for a size 10?",
        questions={
            "department": Choice(
                instructions="Which team should handle this?",
                criteria={
                    "returns": "Exchanges, wrong or damaged items",
                    "shipping": "Delivery status, delays, lost packages",
                    "billing": "Charges, invoices, payment problems",
                },
            ),
        },
    )

    print(response.answers["department"].choice)
```

Jev 같은 System One 모델을 호출하려면 `system_one` 메서드나 `https://api.typesafe.ai/v1/systemone` 엔드포인트를 사용하세요. `model` 필드로 요청을 처리할 모델을 고릅니다. 코드의 어느 지점에서 호출해야 하는지는 [TypeSafe로 구축하는 방법](../concepts/how-to-build-with-system-one.md)에서 다룹니다.

[클라이언트 SDK](../sdk.md) 중 하나를 사용하거나 [HTTP API](https://docs.typesafe.ai/api)를 직접 호출하세요. 코딩 에이전트에 연동 코드 작성을 맡긴다면, 에이전트가 요청과 응답의 형태를 알 수 있도록 먼저 [TypeSafe 에이전트 스킬](../agent-skill.md#installation)을 설치하세요.

> [!NOTE]
>
> `instructions`(지시문)와 `criteria`(판단 기준)의 각 항목은 문자열, 객체, 배열 중 어느 것이든 될 수 있습니다. 문자열로 시작하세요. 선택지가 무엇을 다루는지, 무엇을 다루지 않는지, 그리고 몇 가지 예시처럼 설명에 여러 종류의 안내가 필요할 때는 객체를 쓰세요. 아래의 [구조화된 instructions와 criteria](#structured-instructions-and-criteria)와 [API 레퍼런스](https://docs.typesafe.ai/api)를 참고하세요.

## 응답 구조

응답의 `answers`에는 질문마다 항목이 하나씩 있으며, 요청에서 쓴 id 아래에 들어 있습니다. 다음은 위 예시 요청에 대한 응답입니다.

```json
{
  "model": "jev-1.13.0",
  "answers": {
    "department": {
      "type": "choice",
      "choice": "returns",
      "confidence": 1.0,
      "probabilities": {
        "shipping": 0.0,
        "returns": 1.0,
        "billing": 0.0
      }
    }
  },
  "usage": {
    "input_tokens": 328,
    "output_tokens": 34
  }
}
```

`type` 외에 각 Choice 답에는 세 가지 값이 있습니다.

* `choice`: 확률이 가장 높은 선택지입니다.
* `probabilities`: 모든 선택지에 대한 전체 확률 분포입니다. 모든 값의 합은 1입니다.
* [`confidence`](../confidence.md): `probabilities`가 얼마나 퍼져 있는지로 계산한 0에서 1 사이의 숫자입니다. 확률이 여러 선택지에 고르게 퍼진 평평한 모양이면 confidence가 낮습니다. 한 선택지에만 봉우리가 솟아 있으면 confidence가 높습니다.

이 티켓은 쉬운 경우라서 확률이 전부 `returns`에 몰리고 confidence는 1.0입니다. 사이즈가 잘못 왔다는 내용과 환불이 누락되었다는 내용을 함께 언급하는 티켓이라면 확률이 `returns`와 `billing`으로 나뉘고 confidence는 떨어집니다.

## 권장 사항: 요청 하나에 질문을 둘 이상 넣으세요

코드에 필요할 수 있는 Choice 질문은 질문마다 요청을 따로 보내지 말고 요청 하나에 모두 담으세요. 질문은 병렬로 평가됩니다. 질문을 추가해도 응답 시간은 거의 변하지 않고, 필요 없는 답은 코드에서 무시하면 됩니다. 다만 추가한 질문만큼 token(토큰) 비용은 듭니다. 자세한 설명은 [여러 질문을 함께 묻기](../primitives.md#ask-multiple-questions-together)에 있으며, 다음 절에서는 요청 하나에 Choice 질문 다섯 개를 담은 예를 보여 줍니다.

Choice 질문 하나에 들어가는 선택지에도 같은 논리가 적용됩니다. Choice 질문은 선택지를 255개까지 받을 수 있고 선택지 하나를 추가하는 데는 token이 몇 개밖에 들지 않으므로, 간추린 목록이 아니라 팀, 카테고리, 상품의 전체 목록을 모델에 주세요. 목록이 모든 입력을 포괄하지 못할 수 있다면 `other`나 `none of the above` 선택지를 추가해, 다른 선택지가 모두 맞지 않는다고 모델이 답할 수 있게 하세요.

> [!TIP]
>
> `none of the above` 선택지는 여러 사용 사례에서 잘 동작하지 않을 수 있습니다. 대안으로, 같은 요청에 맞는 선택지가 하나라도 있는지 확인하는 companion [Noul](noul.md)(동반 Noul) 질문을 함께 넣으세요. 예를 들면 "Does `message` name any of `our_products`?" 같은 질문입니다. 그러면 코드는 Noul의 답이 예일 때만 Choice의 답을 읽습니다.

깊은 계층 구조나 큰 taxonomy(분류 체계)로 문서를 분류하려면 Choice 질문을 단계별로 이어 붙이세요. [Hierarchical Classification cookbook(예제 모음)](../cookbooks/hierarchical_classification.md)은 Choice 확률을 대상으로 beam search(빔 서치)를 실행하는 방법을 보여 줍니다. 탐욕적으로 경로 하나만 따라가지 않고 각 단계에서 가장 좋은 후보 경로 `K`개를 유지하는 방식입니다.

## 더 복잡한 예제

위의 기본 예제는 티켓을 팀에 배정합니다. 더 큰 지원 시스템이라면 반품 사유, 배송 문제, 고객이 원하는 것, 고객의 어조도 필요할 수 있습니다.

아래 요청은 첫 번째보다 더 모호한 티켓에 대해 Choice 질문 다섯 개를 묻습니다. 이 티켓은 세 팀과 관련이 있고 고객이 무엇을 원하는지 말하지 않습니다.

Playground에서 실행해 보기 →

이 Choice 질문 중 두 개는 미리 던져 두는 추측성 질문입니다. `return_reason`은 `department`가 `returns`일 때만, `shipping_issue`는 `shipping`일 때만 의미가 있습니다. `tone` 질문은 선택지 이름만으로 뜻이 분명하므로 설명에 `null`을 사용합니다.

Jev의 응답:

```json
{
  "model": "jev-1.13.0",
  "answers": {
    "department": {
      "type": "choice",
      "choice": "returns",
      "confidence": 0.42,
      "probabilities": {
        "shipping": 0.04,
        "billing": 0.35,
        "returns": 0.61
      }
    },
    "return_reason": {
      "type": "choice",
      "choice": "wrong_size",
      "confidence": 1.0,
      "probabilities": {
        "other": 0.0,
        "wrong_size": 1.0,
        "changed_mind": 0.0,
        "damaged": 0.0,
        "wrong_item": 0.0
      }
    },
    "shipping_issue": {
      "type": "choice",
      "choice": "delayed",
      "confidence": 0.67,
      "probabilities": {
        "wrong_address": 0.0,
        "other": 0.26,
        "not_delivered": 0.0,
        "damaged_in_transit": 0.0,
        "delayed": 0.74
      }
    },
    "requested_resolution": {
      "type": "choice",
      "choice": "refund",
      "confidence": 0.2,
      "probabilities": {
        "replacement": 0.34,
        "refund": 0.4,
        "information": 0.02,
        "exchange": 0.24
      }
    },
    "tone": {
      "type": "choice",
      "choice": "frustrated",
      "confidence": 0.76,
      "probabilities": {
        "frustrated": 0.84,
        "angry": 0.16,
        "calm": 0.0
      }
    }
  },
  "usage": {
    "input_tokens": 589,
    "output_tokens": 212
  }
}
```

각 질문은 티켓을 대상으로 따로따로 답이 매겨집니다.

* `department`의 답은 확률 0.61의 `returns`이지만, 이중 결제 때문에 `billing`도 0.35입니다. 이 티켓은 두 팀에 걸쳐 있고, 갈라진 confidence 0.42가 이를 반영합니다.
* `return_reason`은 confidence 1.0의 `wrong_size`입니다. 티켓에 분명히 적혀 있으므로 예상대로입니다.
* `shipping_issue`의 답은 `delayed`와 `other`로 나뉩니다. 추측성 질문인 데다 `department`가 shipping으로 나오지 않았으므로, 아래 예제 코드에서 보듯 코드가 무시해도 됩니다.
* `requested_resolution`의 답은 0.40으로 `refund` 쪽에 기울어 있고 나머지 대부분을 `replacement`와 `exchange`가 나눠 가지며, confidence는 0.20입니다. 이중 결제는 환불을, 잘못된 사이즈는 교환을 시사하는데, 고객은 어느 쪽을 원하는지 끝내 말하지 않습니다.
* `tone`의 답은 확률 0.84, confidence 0.76의 `frustrated`입니다.

아래 예제 코드는 필요한 답만 읽고 나머지는 무시하며, confidence가 낮은 답은 바로 조치하지 않고 되물어야 할 이유로 취급합니다.

```python
from typesafe_sdk import Choice, TypeSafeClient

TRIAGE_QUESTIONS = {
    "department": Choice(
        instructions="Which team should handle this?",
        criteria={
            "returns": "Exchanges, wrong or damaged items",
            "shipping": "Delivery status, delays, lost packages",
            "billing": "Charges, invoices, payment problems",
        },
    ),
    "return_reason": Choice(
        instructions="If the customer wants to return something, why?",
        criteria={
            "wrong_size": "The item doesn't fit",
            "wrong_item": "A different product was delivered",
            "damaged": "The item arrived broken or faulty",
            "changed_mind": "The item is fine, the customer no longer wants it",
            "other": "A return reason that fits none of the above",
        },
    ),
    "shipping_issue": Choice(
        instructions="If this is a shipping problem, which kind is it?",
        criteria={
            "not_delivered": "The package never arrived",
            "delayed": "The package is late but still on its way",
            "wrong_address": "The package went to the wrong place",
            "damaged_in_transit": "The package arrived damaged",
            "other": "A shipping problem that fits none of the above",
        },
    ),
    "requested_resolution": Choice(
        instructions="What does the customer want to happen?",
        criteria={
            "exchange": "Swap the item for a different one",
            "refund": "Money back",
            "replacement": "The same item sent again",
            "information": "Just an answer, no action needed",
        },
    ),
    "tone": Choice(
        instructions="What is the customer's tone?",
        criteria={"calm": None, "frustrated": None, "angry": None},
    ),
}


def triage(ticket: str) -> None:
    with TypeSafeClient() as client:
        response = client.system_one(
            state=ticket,
            questions=TRIAGE_QUESTIONS,
        )
    answers = response.answers

    department = answers["department"]
    if department.confidence < 0.3:
        # 어느 팀에 보낼지 분명하지 않습니다. 사람이 결정하게 합니다.
        send_to_manual_triage(ticket)
        return

    if department.choice == "returns":
        # return_reason 답은 여기서만 사용합니다
        assign(ticket, team="returns", issue=answers["return_reason"].choice)
    elif department.choice == "shipping":
        # shipping_issue 답은 여기서만 사용합니다
        assign(ticket, team="shipping", issue=answers["shipping_issue"].choice)
    else:
        assign(ticket, team="billing")

    # 확률을 유의미하게 나눠 가진 두 번째 팀에도 사본을 보냅니다
    for team, probability in department.probabilities.items():
        if team != department.choice and probability > 0.25:
            notify(ticket, team=team)

    resolution = answers["requested_resolution"]
    if resolution.confidence < 0.5:
        # 고객이 무엇을 원하는지 말하지 않았습니다. 추측하지 말고 물어봅니다.
        ask_customer_what_they_want(ticket)
    elif resolution.choice == "refund":
        flag_for_refund_approval(ticket)

    if answers["tone"].choice == "angry":
        flag_for_senior_agent(ticket)
```

위 티켓의 경우 이 코드는 티켓을 `wrong_size` 문제로 반품 팀에 배정하고, 결제 팀의 확률 0.35가 threshold(임계값) 0.25를 넘으므로 결제 팀에 사본을 보내며, 해결 방식의 confidence 0.20이 0.5보다 낮으므로 고객에게 무엇을 원하는지 묻습니다. `shipping_issue` 답은 사용하지 않습니다.

요청 한 번에 답 다섯 개를 받고, routing(라우팅) 로직은 평범한 `if` 문입니다. 나중에 고객의 언어나 티켓이 어떤 상품에 관한 것인지 알아야 한다면 `TRIAGE_QUESTIONS`에 Choice 질문을 하나 더 추가하면 됩니다. 호출 횟수는 그대로 한 번입니다.

[스마트 홈 비서 데모](../demos/smart-home.md)는 모든 사용자 요청을 한 번의 호출로 긴 Choice 질문 목록에 비추어 평가합니다. 요청 카테고리, 방, 기기, 동작 등입니다. 이 질문들 대부분은 개별 요청과 관련이 없으며 코드가 무시합니다.

## 구조화된 instructions와 criteria

선택지마다 한 줄짜리 설명으로 시작하세요. 두 선택지가 비슷해서 모델이 계속 혼동한다면 각 선택지를 문자열 대신 객체로 설명하세요. 선택지가 무엇을 다루는지, 무엇이 이웃한 선택지에 속하는지, 그리고 몇 가지 예시 입력을 담는 필드를 두세요.

아래의 두 선택지 return\_policy와 return\_status는 혼동하기 쉽습니다. 어느 쪽에 관한 티켓이든 반품과 환불을 언급할 수 있으므로, 각 선택지에 자신이 다루지 않는 것을 적어 둡니다.

Playground에서 실행해 보기 →

응답은 confidence 1.0의 `return_status`입니다.

```json
{
  "model": "jev-1.13.0",
  "answers": {
    "return_topic": {
      "type": "choice",
      "choice": "return_status",
      "confidence": 1.0,
      "probabilities": {
        "return_policy": 0.0,
        "return_status": 1.0
      }
    }
  },
  "usage": {
    "input_tokens": 407,
    "output_tokens": 32
  }
}
```

`question`, `focus`, `what`, `not_for`, `examples`라는 필드 이름은 API의 일부가 아니며 예약어도 아닙니다. 선택지 이름을 정할 때와 마찬가지로 여러분이 직접 정합니다. 모델은 값과 함께 이름도 보므로, 뒤따르는 내용이 무엇인지 알려 주는 짧은 이름을 쓰세요.


이 문서는 개발자 문서 플랫폼인 [Mintlify](https://mintlify.com)로 제작하고 호스팅합니다.
