# TypeSafe로 구축하는 방법

> 제어권은 코드에 두고 Jev에는 범위가 좁고 구조화된 결정만 맡겨 AI 기반 소프트웨어를 설계합니다.

Jev는 에이전트가 아니라 AI 기반 소프트웨어를 만들기 위한 TypeSafe의 모델입니다. 코드를 생성하지도, 다음 행동을 스스로 고르지도 않습니다. 대신 소프트웨어에 끼워 넣을 수 있는 AI primitive(프리미티브: 기본 구성 요소)를 제공합니다. 그래서 제어권은 코드에 그대로 남고, 모델은 비정형 데이터에 대한 상식 수준의 판단을 맡습니다.

> [!NOTE]
>
> **요약:** 일반적인 소프트웨어 워크플로를 만들고, AI가 필요한 곳에만 Jev를 넣으세요.
>
> * 제어 흐름, 결정론적 규칙, 부수 효과는 코드에 둡니다.
> * 범위가 넓은 판단은 명시적인 instructions(지시문)와 criteria(판단 기준)를 갖춘, 좁고 타입이 지정된 질문으로 쪼갭니다.
> * 각 질문에는 필요한 컨텍스트만 줍니다.
> * 확률과 confidence(신뢰도)를 보고 바로 조치할지, 검토를 요청할지, 상위 단계로 넘길지 정합니다.
> * 서로 독립적인 질문은 한꺼번에 묻고, 답은 코드에서 조합합니다.

## 세 가지 소프트웨어 아키텍처

TypeSafe는 **AI 기반 소프트웨어**를 만들기 위해 설계되었습니다. 여기서는 코드가 워크플로를 책임지고, AI는 범위가 좁고 구조화된 결정을 처리합니다.

> [tab] 전통적인 소프트웨어
>
> 전통적인 코드는 단순한 소프트웨어 primitive로 이루어진 복잡한 의사 결정 트리입니다. primitive 하나하나를 신뢰할 수 있으므로 개발자는 이를 조합해 더 높은 수준의 추상화를 만들 수 있습니다.

> [tab] LLM 에이전트
>
> 에이전트는 지시를 처리하고 다음 단계를 스스로 고릅니다. 사람이 과정을 지켜보고 있을 때는 잘 동작하지만, 루프를 한 번 돌 때마다 궤도를 벗어날 기회가 하나씩 더 생깁니다.

> [tab] AI 기반 소프트웨어
>
> 코드가 결정론적 작업을 처리하고 제어 흐름을 책임집니다. 모델은 시스템에 프로그래밍 가능한 상식이 필요하거나 비정형 데이터를 해석해야 하는 곳에만 등장합니다. 각 AI 작업은 원자적이고 범위가 제한된 상태로 유지됩니다.

![전통적인 소프트웨어, 에이전트, AI 기반 소프트웨어를 서로 다른 세 가지 시스템 아키텍처로 나타낸 그림.](https://mintcdn.com/ts-docs/aFVnpmCIX68NpsV1/images/how-to-build-with-typesafe/software-architectures-light.webp?fit=max\&auto=format\&n=aFVnpmCIX68NpsV1\&q=85\&s=35c7622176190d1b1f19dc712f2fbf11)

## Jev를 조합할 수 있는 이유

> [card] 구조화된 출력
>
> Jev는 설계 단계부터 타입 안전합니다. 결정과 확률은 코드가 기대하는 구조화된 소프트웨어 타입과 JSON 스키마를 따르므로, 생성된 산문에서 값을 다시 뽑아낼 일이 없습니다.

> [card] 병렬 실행
>
> 질문은 서로 독립적으로, 병렬로 평가됩니다. 한 primitive의 결과가 숨은 컨텍스트가 되어 다른 primitive의 결과를 바꾸는 일이 없습니다.

> [card] 비교 가능
>
> 출력은 정렬할 수 있고, 똑똑한 `if` 문과 threshold(임계값), 비교 연산의 근거로 쓸 수 있습니다.

> [card] 빠른 속도
>
> 대부분의 호출은 약 100 ms 안에 끝납니다. Jev는 실시간 요청 경로와 사용자 인터페이스에 쓸 수 있을 만큼 빠릅니다.

> [card] Calibrated confidence
>
> [RLCD](../introduction/machine-learning-primer.md)는 과신하는 쪽으로 치우치지 않고 calibrated(보정된) 확률로 불확실성을 전달합니다.

> [card] 자기 일관성
>
> Jev는 평가를 반복해도 안정적인 답을 반환하도록 설계되었습니다. [자기 일관성 cookbook(예제 모음)](../cookbooks/consistency_noul_cookbook.md)을 참고하세요.

모든 출력이 제공된 선택지 안으로 제한되므로, 모델은 스키마 밖의 값을 지어내지 않고 그 선택지 전체에 대한 확률 분포를 반환합니다. TypeSafe의 목표는 속도와 비용 대비 지능의 비율을 100배 넘게 끌어올리는 것입니다. 그 바탕에는 지능이 저렴해질수록 수요가 훨씬 많이 생긴다는 판단이 있습니다.

## System One 워크플로 설계하기

### 1. 코드로 할 수 있으면 코드로 하세요

결정론적 작업은 코드에 두세요. 코드는 안정적이고 저렴합니다. 소프트웨어 워크플로로 같은 동작을 표현할 수 있다면 에이전트식 `while` 루프는 피하세요.

> [accordion] 예시: 결정론적 규칙은 코드에 두기
>
> ```python theme={null}
> days_overdue = (today - invoice.due_date).days
>
> if days_overdue > 30:
>     route_to_collections(invoice)
> ```

모델의 결정을 코드와 조합하는 범위가 정해진 방법은 [System One 패턴](../patterns.md)에서 살펴보세요.

### 2. 입력 state를 분해하세요

지금 묻는 질문과 관련된 컨텍스트만 넣으세요. 그러면 모델이 엉뚱한 정보에 주의를 빼앗기거나 context rot(컨텍스트 부패)을 겪는 일을 피할 수 있습니다. 최신 정보를 자체 지식 베이스에서 가져올 수 있다면 모델 가중치에 저장된 지식에 기대지 마세요.

> [accordion] 예시: 관련된 컨텍스트만 보내기
>
> Playground에서 실행해 보기 →

### 3. 입력 state를 구조화하세요

`state`와 `questions` 필드에는 중첩 JSON을 사용하세요. 모호함을 없앨 수 있다면 질문이 특정 값을 가리키게 하고, 질문 안에서 각 경로를 백틱 문자로 감싸세요.

> [accordion] 예시: 중첩된 값 참조하기
>
> 질문이 중첩된 특정 값을 가리키게 하려면 `support.tickets[0].message`처럼 점과 인덱스로 이루어진 경로를 백틱으로 감싸서 쓰세요.
>
> Playground에서 실행해 보기 →

### 4. 질문을 분해하세요

가능한 한 명시적이고, 좁고, 구체적이고, 원자적인 질문을 하세요. 복잡하거나 정의가 불분명한 질문은 각각 한 가지 속성만 평가하는 별도의 질문으로 쪼개세요.

> [!NOTE]
>
> 이 가이드에서 아마 가장 중요한 개념입니다. 범위가 넓은 질문은 답 하나 뒤에 여러 판단을 감춥니다. 원자적인 질문은 그 판단들을 드러내므로 코드에서 하나씩 살펴보고, 조정하고, 조합할 수 있습니다.

> [accordion] 예시: 스팸 탐지 분해하기
>
> Playground에서 실행해 보기 →
>
> Playground에서 실행해 보기 →

> [accordion] 예시: 도구 호출 기록 검증하기
>
> Playground에서 실행해 보기 →
>
> Playground에서 실행해 보기 →

### 5. 질문을 구조화하세요

질문은 짧게 유지하세요. `instructions`와 `criteria`는 보통 문자열이며, 짧고 모호하지 않은 질문이라면 문자열 하나로 충분합니다. 객체나 배열로 쓸 수도 있습니다. 질문은 한 필드에 넣고, 질문을 뒷받침하는 데이터는 다른 필드에 넣으세요.

구조화는 다음과 같은 상황에서 도움이 됩니다.

* 질문에 컨텍스트나 예시가 필요한 경우. 배경 정보를 담은 긴 문장이나 예시 입력 목록은 질문 옆의 이름 붙은 필드에 두는 것이 좋습니다. 그러면 질문을 다시 쓰지 않고도 코드에서 내용을 추가하거나 바꿔 끼울 수 있습니다.
* 질문의 일부가 코드에서 오는 경우. 값이 데이터베이스에서 온다면 문자열 템플릿에 끼워 넣지 말고 별도의 필드에 넣으세요.
* 여러 질문의 instructions가 비슷한 경우. 요청 하나는 state 하나를 받고 여러 질문을 담을 수 있습니다. 보조 데이터를 추가하면 질문들을 서로 구별하는 데 도움이 됩니다.

> [accordion] 예시: 코드에서 가져온 레코드 참조하기
>
> 이 Noul 질문은 state에 있는 이력서를 지원자 데이터베이스의 레코드와 비교합니다. 레코드는 있는 그대로 `potential_duplicate`에 들어가고, 질문은 이를 이름으로 가리킵니다.
>
> Playground에서 실행해 보기 →

코드에서 가져온 "potential\_duplicate" 데이터는 시간이 지나면서 바뀔 수 있습니다. "question"은 백틱을 사용해 이를 참조합니다.

`criteria` 안의 설명도 객체가 될 수 있습니다. Choice 질문에서는 각 선택지의 설명을 객체로 만들어, 그 선택지가 무엇을 다루는지, 무엇이 다른 선택지에 속하는지, 그리고 몇 가지 예시를 담을 수 있습니다. 모델이 선택지를 직접 비교할 수 있도록 모든 선택지에 같은 필드 이름을 쓰세요.

> [accordion] 예시: 대조형 Choice criteria 정의하기
>
> Playground에서 실행해 보기 →

질문 유형별 페이지에 실제 예제가 있습니다.

* [Noul](../primitives/noul.md#structured-instructions)은 이력서 하나를 여러 지원자 레코드와 비교합니다. 레코드마다 질문을 하나씩 두며, 질문은 코드에서 만듭니다.
* [Choice](../primitives/choice.md#structured-instructions-and-criteria)는 혼동하기 쉬운 두 선택지를 각각 무엇을 다루는지, 무엇에는 해당하지 않는지, 그리고 예시로 설명합니다.
* [Score](../primitives/score.md#structured-level-descriptions)는 각 단계에 설명과 예시 상황을 붙입니다.

[구조화 데이터 추출 캐스케이드 cookbook](../cookbooks/sde_cascade.md)은 같은 문구를 공유하는 경우를 보여 줍니다. 추출한 레코드의 모든 필드에 같은 질문 묶음을 던지는 예제입니다.

짧고 모호하지 않은 질문이나 기준은 문자열로 남겨 두어도 됩니다. 구조는 그대로 두면 서로 뒤섞일 안내 내용을 분리할 수 있을 때 추가하세요. 구조를 쓸 수 있는 모든 위치는 [Advanced: structure](../primitives/advanced.md)를 참고하세요.

### 6. 질문을 많이 하세요

하나의 요청에서 같은 state에 대해 좁고 독립적인 질문을 많이 하세요. 이것이 API에서 비용 대비 효과와 지능을 극대화하는 방법입니다. 질문은 병렬로 실행되고, 코드는 모델을 순차적으로 여러 번 왕복 호출하지 않고도 질문들의 신호를 조합할 수 있습니다.

[Speculative fan-out(추측성 팬아웃) 패턴](../patterns/fan-out.md)과 [병렬 질문 cookbook](../cookbooks/parallel_questions.md)을 참고하세요.

### 7. 질문의 출력을 코드에서 조합하세요 (또는 전통적인 ML 모델에 입력하세요)

독립적인 답은 결정론적 규칙이나 가중합으로 조합하세요. 조합 방식 자체를 학습시키려면 확률을 후속 단계의 전통적인 머신러닝 모델에 feature(피처)로 넣으세요.

> [accordion] 예시: 가중 점수로 신호 조합하기
>
> ```python theme={null}
> answers = response.answers
>
> # 독립적인 신호를 애플리케이션에 맞는 하나의 점수로 조합합니다.
> quality = (
>     0.4 * answers["answers_request"].noul
>     + 0.4 * answers["citations_are_supported"].noul
>     + 0.2 * (1 - answers["contradicts_context"].noul)
> )
> ```

[Composite scoring](../patterns/composite-scoring.md)은 개별 판단을 보존하면서 조합하는 방법을 보여 줍니다. 후속 모델을 학습시킬 레이블이 없다면 비용이 큰 추론 모델 여러 개를 앙상블로 묶어 레이블을 생성하세요. [AutoResearch cookbook](../cookbooks/autoresearch_feature_discovery.md)은 Jev의 출력으로 전통적인 모델을 학습시키는 방법을 보여 줍니다.

### 8. 불확실성에 따라 분기하세요

confidence가 높은 답과 낮은 답에 코드가 서로 다른 조치를 하게 만드세요. 불확실한 경우는 사람이나 더 비싼 추론 모델에 넘깁니다. threshold는 여러분의 데이터에서 confidence와 정확도의 관계를 그래프로 그려 보며 검증하세요. Noul이라면 예는 0.7, 아니요는 0.3에서 시작하는 것이 좋습니다. [Noul 읽는 법](../primitives/noul.md#reading-a-noul)을 참고하세요.

> [accordion] 예시: confidence에 따라 분기하기
>
> ```python theme={null}
> answer = response.answers["card_help_topic"]
>
> if answer.confidence < 0.8:
>     route_to_human_review(ticket)
> else:
>     route_to_handler(answer.choice, ticket)
> ```

threshold를 고르고 각 조치의 위험도에 맞추는 방법은 [Confidence](../confidence.md)와 [확률에 따른 routing(라우팅)](../patterns/probability-routing.md)을 참고하세요.

> [!TIP]
>
> 질문을 분해한다고 왕복 호출이 늘어나지는 않습니다. 같은 state에 대한 질문은 병렬로 실행됩니다.

## 전체를 하나로 합치기

아래 고객 지원 티켓 워크플로는 결정론적 작업을 코드에 두고, 관련된 구조화 컨텍스트만 보내고, 하나의 요청에서 여러 원자적 질문을 평가하고, 명시적인 confidence threshold로 답을 조합합니다.

```python title="triage_ticket.py"
from typesafe_sdk import Choice, Noul, NoulCriteria, Score, TypeSafeClient


def triage_ticket(ticket, customer):
    # 결정론적으로 처리할 수 있는 상태는 모델을 호출하지 않고 처리합니다.
    if ticket["status"] == "closed":
        return "no_action"

    open_orders = [
        order for order in customer["orders"] if order["status"] != "delivered"
    ]

    # 아래 질문에 필요한 구조화 컨텍스트만 넣습니다.
    state = {
        "ticket": {
            "message": ticket["message"],
            "sender": ticket["sender"],
            "links": ticket["links"],
        },
        "customer": {
            "plan": customer["plan"],
            "open_orders": open_orders,
        },
        "policy": {
            "sensitive_credentials": ["password", "security code", "API key"],
        },
    }

    # 구조화된 원자적 질문을 한꺼번에 물어 병렬로 실행되게 합니다.
    questions = {
        "topic": Choice(
            instructions={
                "question": "Which team should handle `ticket.message`?",
                "focus": "Classify the customer's primary request.",
            },
            criteria={
                "billing": {
                    "what": "Charges, invoices, refunds, or subscriptions",
                    "not_for": "Order tracking or account access",
                    "examples": ["I was charged twice", "Where is my refund?"],
                },
                "orders": {
                    "what": "Order status, delivery, cancellation, or returns",
                    "not_for": "Charges or account access",
                    "examples": ["Where is my order?", "Cancel my shipment"],
                },
                "account": {
                    "what": "Login, profile, permissions, or security",
                    "not_for": "Charges or order tracking",
                    "examples": ["Reset my password", "I cannot sign in"],
                },
            },
        ),
        "requests_credentials": Noul(
            instructions={
                "question": "Does the message request a sensitive credential?",
                "compare": [
                    "`ticket.message`",
                    "`policy.sensitive_credentials`",
                ],
                "focus": "Look for a request to disclose the credential itself.",
            },
            criteria=NoulCriteria(
                true={
                    "what": "Asks the recipient to disclose a listed credential",
                    "examples": [
                        "Reply with your password",
                        "Send us your API key",
                    ],
                },
                false={
                    "what": "Does not ask the recipient to disclose a credential",
                    "not_for": "A legitimate instruction to reset a credential",
                    "examples": ["Use this link to reset your password"],
                },
            ),
        ),
        "sender_identity_mismatch": Noul(
            instructions={
                "question": "Does the claimed sender identity conflict with its domain?",
                "compare": [
                    "`ticket.sender.display_name`",
                    "`ticket.sender.email`",
                ],
                "focus": "Compare the named organization with the email domain.",
            },
            criteria=NoulCriteria(
                true={
                    "what": "Claims an organization unrelated to the email domain",
                    "examples": ["Acme Payroll sent from claim-bonus.example"],
                },
                false={
                    "what": "The identity and domain agree or make no conflicting claim",
                    "examples": ["Acme Payroll sent from acme.example"],
                },
            ),
        ),
        "unexpected_reward": Noul(
            instructions={
                "question": "Does the message announce an unexpected reward?",
                "inspect": "`ticket.message`",
                "focus": "Look for an unsolicited prize, payment, or reward claim.",
            },
            criteria=NoulCriteria(
                true={
                    "what": "Announces an unrequested prize, payment, or reward",
                    "examples": ["You were selected for a $1,000 bonus"],
                },
                false={
                    "what": "Contains no reward claim or discusses an expected payment",
                    "not_for": "A customer asking about a known refund or payroll deposit",
                    "examples": ["When will my approved refund arrive?"],
                },
            ),
        ),
        "refund_requested": Noul(
            instructions={
                "question": "Does the customer explicitly request a refund or credit?",
                "inspect": "`ticket.message`",
                "focus": "Require a requested remedy, not a billing complaint alone.",
            },
            criteria=NoulCriteria(
                true={
                    "what": "Directly asks for money back or an account credit",
                    "examples": ["Please refund the duplicate charge"],
                },
                false={
                    "what": "Does not ask for a refund or credit",
                    "not_for": "A complaint or billing question without a requested remedy",
                    "examples": ["Why was I charged twice?"],
                },
            ),
        ),
        "mentions_open_order": Noul(
            instructions={
                "question": "Does the message refer to a supplied open order?",
                "compare": [
                    "`ticket.message`",
                    "`customer.open_orders`",
                ],
                "focus": "Match an order id or other identifying details.",
            },
            criteria=NoulCriteria(
                true={
                    "what": "Refers to an open order by id or identifying details",
                    "examples": ["Where is order A-104?"],
                },
                false={
                    "what": "Does not identify any supplied open order",
                    "not_for": "A generic order question with no matching details",
                    "examples": ["How long does shipping usually take?"],
                },
            ),
        ),
        "frustration": Score(
            instructions={
                "question": "How frustrated does the customer appear?",
                "inspect": "`ticket.message`",
                "focus": "Judge expressed frustration, not issue severity.",
            },
            criteria=[
                {
                    "what": "Calm and matter-of-fact",
                    "signals": ["Neutral wording", "No complaint about the experience"],
                },
                {
                    "what": "Frustrated but civil",
                    "signals": ["Expresses annoyance", "Remains constructive"],
                },
                {
                    "what": "Very angry or threatening to leave",
                    "signals": ["Hostile language", "Threatens cancellation or churn"],
                },
            ],
        ),
    }

    with TypeSafeClient() as client:
        response = client.system_one(
            state=state,
            questions=questions,
        )

    # 독립적인 스팸 신호를 코드에서 정한 가중치로 조합합니다.
    answers = response.answers
    spam_risk = (
        0.45 * answers["requests_credentials"].noul
        + 0.30 * answers["sender_identity_mismatch"].noul
        + 0.25 * answers["unexpected_reward"].noul
    )

    # 불확실한 판단은 추측하지 않고 상위 단계로 넘깁니다.
    spam_is_uncertain = 0.4 < spam_risk < 0.6
    if spam_is_uncertain or answers["topic"].confidence < 0.75:
        return route_to_human_review(ticket)
    if spam_risk >= 0.6:
        return quarantine_as_spam(ticket)

    # 미리 받아 둔 답 가운데 이 경로에서 어떤 것이 의미 있는지는 코드가 정합니다.
    if answers["topic"].choice == "billing":
        return route_to_billing(
            ticket,
            refund_requested=answers["refund_requested"].noul >= 0.7,
        )
    if answers["topic"].choice == "orders":
        return route_to_orders(
            ticket,
            mentions_open_order=answers["mentions_open_order"].noul >= 0.7,
        )

    priority = (
        "high"
        if answers["frustration"].confidence >= 0.7
        and answers["frustration"].score >= 1.5
        else "normal"
    )
    return route_to_account_support(ticket, priority=priority)
```


이 문서는 개발자 문서 플랫폼인 [Mintlify](https://mintlify.com)로 제작하고 호스팅합니다.
