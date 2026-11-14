# Primitive(질문)

> System One 질문 유형, 각 유형이 반환하는 타입이 지정된 답, 유형을 고르는 방법, 여러 질문을 한꺼번에 묻는 방법을 다룹니다.

TypeSafe의 primitive(프리미티브: 기본 구성 요소)는 코드에서 조합해 쓰는 작고 타입이 지정된 구성 단위입니다. primitive는 쌍으로 이루어집니다. 질문은 [System One 모델](concepts/system-one.md)이 [state(상태)](concepts/state.md)에 대해 내릴 판단 하나를 정의하고, 답은 그 결과로 돌아오는 타입이 지정된 값입니다. 이 답들을 코드에서 조합해 결정을 내립니다. 질문 유형마다 반환하는 답의 형태가 다릅니다.

| 유형 | 답하는 내용 | 반환값 |
| - | - | - |
| [Choice](primitives/choice.md) | 이 선택지 중 어느 것인가요? | `choice`, `probabilities`, `confidence` |
| [Score](primitives/score.md) | 어느 단계인가요? | `score`, `legend`, `probabilities`, `confidence` |
| [Noul](primitives/noul.md) | 이것이 참인가요? | `noul` (0~1) |
| [Box](primitives/box.md) | 이미지의 어디에 있나요? | `box` (`top`, `bottom`, `left`, `right`), `confidence` |

질문을 하나만 물을 수도 있고 여러 개를 함께 보낼 수도 있습니다. 한 요청의 모든 질문은 같은 state를 보고, 서로 독립적으로 평가되며, 직접 정한 ID 아래에 타입이 지정된 답을 반환합니다.

## 질문 하나에 즉각적인 판단 하나만 요청하기

System One 모델은 빠르고 초점이 분명한 판단을 위해 만들어졌습니다. 해당 분야를 아는 사람이 적절한 맥락만 주어지면 1초 만에 내릴 수 있는 판단을 요청하세요. "Does this message convey urgency?"(이 메시지가 긴급함을 전달하나요?)는 좋은 질문입니다. "Analyze this message and determine the best course of action"(이 메시지를 분석해서 최선의 대응 방안을 정해 주세요)은 그렇지 않습니다. 이런 질문은 느린 추론이 필요하며, 작업을 작은 질문들로 쪼개고 답을 코드에서 조합해야 한다는 신호입니다.

원하는 판단이 서로 독립적인 여러 요소에 달려 있다면 요소마다 따로 묻고 그 답을 직접 짠 로직으로 결합하세요. "rate this startup pitch"(이 스타트업 피치를 평가해 줘)라고 묻는 대신 시장 규모, 기술적 실현 가능성, 차별성을 각각 묻고, 상대적 중요도에 따라 코드에서 가중치를 줍니다. 우선순위가 바뀌면 prompt(프롬프트)를 다시 쓰는 대신 가중치 값을 바꾸면 됩니다. 방법은 [여러 질문을 함께 묻기](#ask-multiple-questions-together)에서 설명합니다.

## 질문 정의하기

모든 질문에는 ID, `type`, `instructions`(지시문)가 있습니다. Choice와 Score 질문은 `criteria`(판단 기준)도 받는데, Choice 질문에서는 선택지를, Score에서는 단계를 정의합니다. Noul 질문은 '예'와 '아니요'가 무엇을 뜻하는지 명확히 하는 선택 항목으로 `criteria`를 받습니다.

* ID. `refund_requested`처럼 직접 고르는 키입니다. 응답에서 답을 식별하는 데 쓰입니다.
* `type`. `choice`, `score`, `noul`, `box` 중 하나입니다.
* `instructions`. state에 대해 묻는 질문입니다. 평가 로직이 들어가는 곳입니다. 명확하고 구체적인 질문으로 쓰거나, 모델이 판단할 진술로 쓰세요. 대부분의 질문은 문자열 하나로 충분합니다. 객체나 배열로 쓸 수도 있는데, 그러면 질문을 한 필드에, 질문이 가리키는 데이터를 다른 필드에 담을 수 있습니다. [질문에 구조 사용하기](concepts/how-to-build-with-system-one.md#use-structure-in-the-questions)를 참고하세요.
* `criteria`. 가능한 답입니다. Choice 질문에서는 선택지의 맵, Score에서는 순서가 있는 단계 목록, Noul에서는 '예'와 '아니요'에 대한 선택적인 설명입니다. 구체적인 형태는 질문 유형별 페이지에서 다룹니다.

다음 질문은 고객이 환불을 요청했는지 묻습니다.

```python
from typesafe_sdk import Noul

questions = {
    "refund_requested": Noul(
        instructions="Does the customer request a refund?",
    ),
}
```

> [!TIP]
>
> 질문 ID는 코드를 위한 것입니다. 모델에는 전송되지 않습니다. ID만 봐도 뜻이 분명해 보이더라도 `instructions`에 완전한 질문을 쓰세요.

## 질문 유형 고르기

필요한 답의 형태에 맞는 유형을 고르세요.

* **Choice**는 답이 서로 순서가 없는 정해진 선택지 집합 중 하나일 때 알맞습니다. 티켓을 담당 부서로 routing(라우팅)하기, 문서 유형 분류하기, 프로그래밍 언어 감지하기 같은 경우입니다. 선택지 목록을 빠짐없이 제공하세요. Choice 질문은 맞는 것이 없어도 언제나 하나를 고릅니다. 그러니 목록이 모든 입력을 포괄하지 못할 수 있다면, 맞는 선택지가 하나라도 있는지 묻는 [동반 Noul](patterns/companion-noul.md)을 같은 요청에 추가하고, Noul이 '예'라고 할 때만 Choice에 따라 행동하세요. TypeSafe의 테스트에서는 이 방식이 `other`나 `none of the above` 선택지를 두는 것보다 잘 동작했습니다. 그런 선택지는 실제 선택지들과 확률을 놓고 경쟁하기 때문입니다.

* **Score**는 답이 연속적인 스펙트럼 위에 놓이고 그 스펙트럼의 각 지점이 무엇을 뜻하는지 설명할 수 있을 때 알맞습니다. 버그 심각도, 고객의 불만 정도, 숙련도 같은 경우입니다. 단계는 직접 정의하며, 모델은 그 단계들 위의 위치를 반환합니다.

* **Noul**은 확률 자체가 유용한 신호인 깔끔한 예/아니요 질문에 알맞습니다. 이 메시지에 개인 식별 정보가 들어 있는지, 고객이 환불을 요청하고 있는지, 이력서에 분산 시스템이 언급되어 있는지 같은 경우입니다.

> [!NOTE]
>
> 예/아니요 판단에는 Noul을, 스펙트럼 위의 위치를 측정할 때는 Score를 사용하세요. "Is this candidate strong in Python?"(이 지원자는 Python에 강한가요?)이라는 질문에는 "strong"의 명확한 정의가 필요합니다. Noul 값 0.5는 모델이 '예'와 '아니요'에 같은 확률을 준다는 뜻입니다. 지원자의 숙련도가 중간이라는 뜻이 아닙니다. 정의가 불분명하면 그 확률을 해석하기 어려워집니다.
>
> 숙련도를 측정하고 싶다면 경험 없음, 어느 정도 익숙함, 매일 사용, 깊은 전문성처럼 단계를 정의한 Score를 사용하세요. 예/아니요 결정이 필요하다면 "Does the resume state that the candidate has used Python at work?"(이력서에 지원자가 업무에서 Python을 사용했다고 적혀 있나요?)처럼 조건을 명확하게 정의하세요.

두 유형이 모두 맞아 보인다면, 코드가 답에 따라 바로 행동할 수 있는 쪽을 고르세요. `refund`, `rebook`, `information` 중에서 고르는 Choice는 세 가지 코드 경로에 그대로 대응합니다. 고객의 불만 정도를 재는 Score는 threshold(임계값)에 대응합니다. Noul은 `if` 하나에 대응합니다.

## 돌아오는 값

답도 primitive입니다. 각 질문 유형은 타입이 지정된 값을 반환하며, 코드에서는 이 값을 비교하고, threshold를 걸고, 정렬하고, 다른 로직에 넘기고, 후속 요청의 state에 넣을 수 있습니다([한 질문이 다른 질문에 의존할 때](#when-one-question-depends-on-another) 참고).

| 유형 | 답 필드 | 읽는 방법 |
| - | - | - |
| Choice | `choice`, `probabilities`, `confidence` | `choice`는 선택된 선택지입니다. `probabilities`는 모든 선택지에 걸친 분포입니다. `confidence`는 그 분포가 얼마나 뾰족하게 몰려 있는지를 요약합니다. |
| Score | `score`, `legend`, `probabilities`, `confidence` | `score`는 단계들 위의 위치이며, 두 단계 사이에 놓일 수 있습니다. `legend`는 단계를 번호별로 다시 보여 줍니다. `probabilities`는 단계들에 걸친 분포입니다. |
| Noul | `noul` | 답이 '예'일 확률입니다. 1에 가까우면 강한 '예', 0에 가까우면 강한 '아니요', 0.5에 가까우면 불확실하다는 뜻입니다. Noul에는 별도의 `confidence`가 없습니다. |
| Box | `box`, `confidence` | `box`는 네 가장자리 `top`, `bottom`, `left`, `right`를 담으며, 각각 이미지의 너비 또는 높이에 대한 비율을 0에서 1 사이로 나타냅니다. Box는 언제나 위치를 반환하므로, 대상이 이미지에 있기는 한지 묻는 Noul 질문과 짝지어 쓰세요. |

이 답들은 두 가지 성질 덕분에 조합할 수 있습니다.

* **모든 답은 제공한 선택지 안으로 제한됩니다.** 모델은 선택지나 단계에 대한 확률 분포를 반환하며, 그 밖의 값은 절대 반환하지 않습니다. 생성된 산문에서 값을 다시 뽑아낼 일이 코드에는 전혀 없습니다.
* **모든 답은 독립적입니다.** 한 질문의 답이 다른 질문의 숨은 맥락이 되지 않습니다. 질문을 추가하거나 빼도 나머지 질문의 결과는 달라지지 않습니다.

일부 질문 유형은 `confidence`(신뢰도)도 반환합니다. 답에 이 값이 들어 있는지는 유형별 페이지에 나와 있습니다. [Confidence](confidence.md)에서는 이 값을 이용해 언제 자동으로 실행하고 언제 사람에게 넘길지 정하는 방법을 설명합니다.

## 특정 필드 참조하기

평가 대상 콘텐츠인 [state](concepts/state.md)는 대화, 레코드, 정책처럼 여러 부분으로 이루어진 JSON 객체인 경우가 많습니다. 질문이 그중 한 부분에 관한 것이라면, 점과 인덱스로 쓴 해당 키의 경로를 백틱까지 포함해 `instructions`에 적어 그 부분을 지목하세요. 그러면 모델이 state의 어느 부분을 판단해야 하는지 알 수 있습니다.

State 페이지에 나온 고객 지원 대화를 예로 들어 보겠습니다.

```json
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

다음 두 질문은 고객의 메시지, 정책, 결제 내역을 경로로 가리킵니다.

```python
questions = {
    "refund_requested": {
        "type": "noul",
        "instructions": "Does `ticket.messages[0].text` request a refund?",
    },
    "policy_supports_refund": {
        "type": "noul",
        "instructions": (
            "Does `refund_policy` support the refund requested "
            "in `ticket.messages[0].text`, given `order.charges`?"
        ),
    },
}
```

경로를 명시하면 구조화된 state의 어느 부분이 각 판단의 근거가 되어야 하는지 분명해집니다. 입력을 구조화하는 방법은 [State](concepts/state.md)를 참고하세요.

## 여러 질문을 함께 묻기

같은 state를 쓰는 질문은 모두 한 요청에 담아 보내세요. 질문 유형은 자유롭게 섞을 수 있습니다. System One 모델은 요청의 모든 질문을 병렬로 평가합니다. 질문을 추가해도 응답 시간은 거의 달라지지 않고, 추가한 질문만큼의 token(토큰) 비용만 드는데 이 비용은 저렴합니다. 필요 없을지도 모르는 질문을 하나 더 묻는 비용은 거의 없는 셈입니다.

다음 요청은 고객 메시지를 분류하고, 긴급한지 확인하고, 불만 정도에 점수를 매기는 일을 한 번에 처리합니다.

Playground에서 실행해 보기 →

TypeSafe의 [클라이언트 SDK](sdk.md)는 타입이 지정된 질문과 답을 제공합니다. Python에서는 `Choice`, `Noul`, `Score` 객체로 이루어진 `questions` 딕셔너리를 `client.system_one(...)`에 전달합니다. 다음 요청은 티켓과 환불 정책을 한 번만 보내고 질문마다 타입이 지정된 답을 받습니다.

```python
from typesafe_sdk import Choice, Noul, Score, TypeSafeClient

state = {
    "ticket_message": "My flight was cancelled. Can I get a refund?",
    "refund_policy": "Cancelled flights are eligible for a full refund.",
}

with TypeSafeClient() as client:
    response = client.system_one(
        state=state,
        questions={
            "refund_requested": Noul(
                instructions="Does `ticket_message` request a refund?",
            ),
            "request_type": Choice(
                instructions="What is the main request in `ticket_message`?",
                criteria={
                    "refund": "The customer wants money returned.",
                    "rebooking": "The customer wants a replacement flight.",
                    "information": "The customer is asking for information only.",
                },
            ),
            "frustration": Score(
                instructions="How frustrated does the customer appear in `ticket_message`?",
                criteria=[
                    "Calm and neutral.",
                    "Concerned but civil.",
                    "Very angry or using strong language.",
                ],
            ),
        },
    )

print(response.answers["refund_requested"].noul)
print(response.answers["request_type"].choice)
print(response.answers["frustration"].score)
```

사용하는 언어에서의 설치와 사용법은 [클라이언트 SDK](sdk.md)를 참고하세요.

### 추측성 질문 묻기

일부 입력에서만 답이 의미 있는 질문까지 포함해, 코드에 필요할 수 있는 질문을 모두 묻고 어떤 답을 쓸지는 코드가 정하게 하세요. 티켓이 버그 리포트가 아닌 것으로 밝혀지면 심각도 답은 무시하면 됩니다. TypeSafe는 이것을 [추측성 fan-out(팬아웃)](patterns/fan-out.md) 패턴이라고 부릅니다. [병렬 질문 cookbook(예제 모음)](cookbooks/parallel_questions.md)은 질문 13개를 호출 한 번으로 묶으면 13번 따로 호출할 때보다 12.2배 저렴하고 11.8배 빠르며 답은 달라지지 않는다는 것을 보여 줍니다.

> [!TIP]
>
> 코딩 에이전트는 사람보다 요청 하나에 질문 하나만 넣는 습관에 더 쉽게 빠집니다. [TypeSafe agent skill](agent-skill.md#installation)은 일부 입력에서만 의미 있는 질문까지 포함해 요청마다 질문을 많이 넣으라고 에이전트에게 알려 줍니다.

### 물을 수 있는 질문 수

한 요청에 넣을 수 있는 질문 수에는 제한이 없습니다. 요청을 제한하는 것은 크기입니다. 요청은 모델의 [요청당 컨텍스트와 질문당 컨텍스트](models.md#context-length)에 들어맞아야 합니다.

질문은 `state`에 비해 짧은 경우가 많고, 그럴 때는 `state`가 두 컨텍스트의 대부분을 차지합니다. 질문이 한 요청에 다 들어가지 않는다면, 같은 `state`를 쓰는 여러 요청에 질문을 나눠 담는 방법이 있습니다.

### 복잡한 판단을 여러 질문으로 나누기

여러 가지에 좌우되는 판단은 한 가지당 질문 하나로 나누는 것이 가장 좋습니다. 답은 코드에서 결합하고, 상대적 중요도에 따라 답마다 가중치를 주세요. 가중치는 직접 정합니다. 결합한 결과가 팀이 내렸을 결정과 맞지 않으면 코드에서 가중치를 바꾸고 다시 실행하세요. 질문들이 한 요청 안에서 병렬로 실행되므로 질문을 추가해도 응답 시간은 거의 달라지지 않습니다. 나누는 데 드는 비용은 질문 token 몇 개가 더 붙는 정도입니다.

예를 들어 티켓 우선순위는 Score 질문 세 개로 구성할 수 있습니다. 버그가 얼마나 심각한지, 고객이 얼마나 불만스러워하는지, 엔지니어가 작업에 착수할 단서가 리포트에 얼마나 있는지입니다. Score 페이지의 [복잡한 판단을 여러 Score로 나누기](primitives/score.md#splitting-a-complex-judgment-into-several-score-questions)에서 이 요청과, 답을 정규화하고 가중치를 주는 코드를 단계별로 살펴봅니다. 이 기법을 [복합 점수 산정](patterns/composite-scoring.md) 패턴이라고 합니다.

### 한 질문이 다른 질문에 의존할 때

같은 요청의 질문들은 독립적입니다. 한 답이 다른 질문의 맥락이 되지 않습니다. 뒤의 판단이 앞의 답에 의존한다면 코드에서 두 번째 요청을 보내세요. 의존 관계가 진짜인 경우는 첫 번째 답을 받기 전에는 코드가 두 번째 요청을 만들 수 없을 때뿐입니다. 즉 state에 넣을 데이터를 더 가져오거나, state를 무엇으로 구성할지 정하거나, 다음 질문의 선택지를 고르는 데 그 답이 필요한 경우입니다. 그렇지 않다면 질문을 함께 묻고 답을 코드에서 결합하세요.

요청을 두 번 보내는 것은 원칙이 아니라 예외입니다. 두 번째 요청의 질문을 원래 state에 대해 물을 수 있었다면 첫 번째 요청에서 묻고, 필요 없는 답은 코드가 무시하게 하세요. 세 가지 cookbook이 진짜 이유가 있어서 두 번째 요청을 보냅니다. [스킬 추천](cookbooks/skill_suggestion.md)은 한 요청으로 스킬 182개의 순위를 매긴 다음, 상위 세 개의 전문을 가져와 더 나은 근거를 놓고 다시 판단합니다. [구조 복원](cookbooks/autoformat.md)은 각 줄바꿈이 문장을 끊었는지 묻고, 그 답에 따라 줄을 블록으로 합친 다음 블록을 분류하는데, 이 블록은 첫 번째 요청이 답하기 전에는 존재하지 않았습니다. [계층적 분류](cookbooks/hierarchical_classification.md)는 각 Choice 답을 이용해 다음 요청이 어떤 선택지를 제시할지 정합니다.

워크플로를 초점이 분명한 판단들로 나누는 방법은 [TypeSafe로 구축하는 방법](concepts/how-to-build-with-system-one.md)을 참고하세요.

## 다음 단계

> [card] [Choice](primitives/choice.md)
>
> 고정된 목록에서 선택지 하나를 고릅니다.

> [card] [Score](primitives/score.md)
>
> 순서가 있는 단계에 따라 state를 평가합니다.

> [card] [Noul](primitives/noul.md)
>
> 진술이 참일 확률을 구합니다.

이들이 시스템 아키텍처로 어떻게 조합되는지 보려면 [패턴](patterns.md)으로 이동하세요.


이 문서는 개발자 문서 플랫폼인 [Mintlify](https://mintlify.com)로 제작하고 호스팅합니다.
