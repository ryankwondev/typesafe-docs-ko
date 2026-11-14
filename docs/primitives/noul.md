# Noul

> Noul 질문은 Jev에게 예/아니요 질문을 평가하게 하고, 답이 예일 확률을 반환받습니다.

Noul은 답이 예 또는 아니요일 때 사용합니다. 예를 들어 이 메시지가 환불을 요청하는지, 이 이력서가 분산 시스템을 언급하는지, 이 댓글에 개인 정보가 들어 있는지 등입니다. 답이 여러 선택지 중 하나라면 [Choice](choice.md)를 사용하세요. 스펙트럼 위의 한 위치라면 [Score](score.md)를 사용하세요. [질문 유형 선택하기](../primitives.md#choose-a-question-type)에서 세 가지를 모두 비교합니다.

Noul의 답은 답이 예일 확률을 나타내는 숫자 하나이며, 0은 아니요를, 1은 예를 뜻합니다.

## 요청 구조

[TypeSafe API](https://docs.typesafe.ai/api)로 보내는 POST 요청 본문의 최상위 필드 세 개는 다른 질문 유형과 같습니다. 평가할 내용인 `state`(상태), `model`, 그리고 `questions`입니다. 각 Noul 질문에는 다음 필드가 있습니다.

* `type`: 항상 `"noul"`입니다.
* `instructions`: 모델이 답할 예/아니요 질문, 또는 모델이 참인지 판단할 진술문입니다.
* `criteria`: 선택 사항입니다. 예와 아니요가 각각 무엇을 뜻하는지 설명하는 `true`와 `false`를 담은 객체입니다.

아래 요청에서 state는 지원 메시지이고, 두 질문은 고객이 사람 상담원을 원하는지와 이전에 지원팀에 문의한 적이 있는지입니다.

Playground에서 실행해 보기 →

질문 id는 여러분이 정하며, 여기서는 `is_human_escalation`과 `is_repeat_contact`입니다. id는 모델에 전달되지 않습니다. 각 답은 같은 id 아래에 반환됩니다. 첫 번째 질문은 `instructions`(지시문)만 사용합니다. 두 번째 질문은 `criteria`(판단 기준)를 추가해 무엇이 예이고 무엇이 아니요인지 알려 줍니다.

[Python SDK](../sdk/python.md)에서는 같은 질문을 `Noul` 객체로 씁니다.

```python
from typesafe_sdk import Noul, NoulCriteria, TypeSafeClient

with TypeSafeClient() as client:
    response = client.system_one(
        model="jev-latest",
        state="I have asked three times now. Can I please just talk to a real person?",
        questions={
            "is_human_escalation": Noul(
                instructions="Is the customer asking for a human agent?",
            ),
            "is_repeat_contact": Noul(
                instructions="Has the customer contacted support about this before?",
                criteria=NoulCriteria(
                    true="Mentions a prior attempt, ticket, or that they have asked before",
                    false="No sign of any previous contact",
                ),
            ),
        },
    )

    print(response.answers["is_human_escalation"].noul)
    print(response.answers["is_repeat_contact"].noul)
```

`system_one` 메서드와 `https://api.typesafe.ai/v1/systemone` 엔드포인트는 둘 다 Jev가 속한 모델군인 [System One](../concepts/system-one.md)에서 이름을 따왔습니다. 코드의 어느 지점에서 사용해야 하는지는 [TypeSafe로 구축하는 방법](../concepts/how-to-build-with-system-one.md)에서 다룹니다.

코딩 에이전트를 사용한다면, 에이전트가 요청과 응답의 형태를 알 수 있도록 먼저 [TypeSafe 에이전트 스킬](../agent-skill.md#installation)을 설치하세요.

> [!NOTE]
>
> `instructions`는 문자열, 객체, 배열 중 어느 것이든 될 수 있습니다. 문자열로 시작하세요. state와 비교할 레코드처럼 질문에 데이터가 함께 필요하거나, 질문의 일부를 코드에서 만들 때는 객체를 쓰세요. 구조가 언제 도움이 되는지는 [질문을 구조화하세요](../concepts/how-to-build-with-system-one.md#use-structure-in-the-questions)에서 설명하며, [아래 예제](#structured-instructions)에서는 코드에서 만든 질문으로 이를 보여 줍니다.

## 응답 구조

응답의 `answers`에는 질문마다 항목이 하나씩 있으며, 요청에서 쓴 id 아래에 들어 있습니다.

```json
{
  "model": "jev-1.13.0",
  "answers": {
    "is_human_escalation": {
      "type": "noul",
      "noul": 0.99
    },
    "is_repeat_contact": {
      "type": "noul",
      "noul": 0.93
    }
  },
  "usage": {
    "input_tokens": 360,
    "output_tokens": 39
  }
}
```

여기서는 두 답이 모두 1에 가깝습니다. 고객이 "talk to a real person"이라고 말하므로 `is_human_escalation`은 0.99입니다. "I have asked three times now"는 `is_repeat_contact`의 `true` 설명과 일치하므로 0.93입니다.

## Noul 읽는 법

이 숫자 하나에 답과 확신의 정도가 함께 담겨 있습니다. 1에 가까운 값은 강한 예입니다. 0에 가까운 값은 강한 아니요입니다. 0.5에 가까운 값은 모델이 예와 아니요에 비슷한 확률을 준다는 뜻입니다.

아래 표는 여러 고객 메시지에 대해 `is_human_escalation` 질문을 던졌을 때 기록된 `jev-1.13.0`의 답입니다.

| State | `noul` |
| - | - |
| Thanks, that fixed it! | 0.02 |
| How do I reset my password? | 0.07 |
| I need this sorted today, whatever it takes. | 0.26 |
| Are you a bot? | 0.40 |
| Is there any way to speak to someone about my invoice? | 0.84 |
| I have asked three times now. Can I please just talk to a real person? | 0.99 |

처음 두 개와 마지막 두 개는 분명합니다. "I need this sorted today"는 급하다는 뜻이지만 사람을 요청하지는 않으므로 0.26을 받습니다. "Are you a bot?"은 직접 요청하지는 않으면서 사람을 원한다는 기색을 내비치며, 모델의 판단은 0.40으로 거의 반반으로 갈립니다. 둘 다 코드의 threshold(임계값)를 기준으로 결정을 내려야 하는 종류의 메시지입니다.

[Choice](choice.md)나 [Score](score.md)와 달리 Noul에는 별도의 `confidence`(신뢰도) 값이 없습니다. Noul의 확률 분포에는 예와 아니요라는 두 가지 결과만 있으므로 `noul` 값 하나로 분포 전체를 나타낼 수 있습니다. Choice나 Score는 확률을 여러 선택지나 단계에 걸쳐 분산시키고, `confidence`가 그 퍼진 정도를 요약합니다.

대개는 코드에서 `noul`에 threshold를 적용해 불리언으로 바꿉니다.

```python
wants_human = response.answers["is_human_escalation"].noul > 0.9

if wants_human:
    route_to_agent(ticket)
else:
    route_to_bot(ticket)
```

예는 0.7, 아니요는 0.3에서 시작하는 것이 좋습니다. 답이 예인지만 중요한 경우가 많고, 그럴 때는 0.7 하나만 있으면 됩니다. threshold를 어디에 둘지는 틀렸을 때의 비용에 달려 있습니다. 누군가를 호출하거나 환불을 진행하는 일처럼 잘못된 예에 따라 조치했을 때 비용이 크다면 threshold를 높이세요. 안전 문제를 표시하지 못하는 일처럼 실제 예를 놓쳤을 때 비용이 크다면 threshold를 낮추세요. threshold를 몇 안 되는 테스트 케이스에 지나치게 맞추지는 마세요. 0.3과 0.7 사이의 값은 두 코드 경로 어느 쪽으로도 보내지 않고 사람에게 보낼 수 있습니다. 이는 [Confidence](../confidence.md#three-paths-for-using-confidence-in-your-code) 페이지가 Choice와 Score의 답에 대해 설명하는 세 갈래 분기와 같습니다.

Noul 값은 0에서 1까지 움직이지만, 여러분이 물어본 대상의 정도를 나타내는 척도는 아닙니다. 답이 예일 확률입니다. 질문이 실제로는 정도를 묻는 것이라면 이 값은 그 정도를 측정하지 못합니다. 아래에서는 지원자 네 명에 대해 "Is the candidate strong in Python?"을 묻고, 그 옆에 네 단계로 이루어진 [Score](score.md) 질문을 나란히 두었습니다. 네 단계는 경험 없음, 약간 익숙함, 업무에서 꾸준히 사용, 깊은 전문성입니다.

| 지원자 | Noul: "Is the candidate strong in Python?" | Score: "How much Python experience does the candidate have?" |
| - | - | - |
| My experience is in Java and Go. I have not used Python. | 0.03 | 0.0 (No experience) |
| I have used Python occasionally for small scripts alongside my main Java work. | 0.14 | 1.0 (Some familiarity) |
| I used Python every day for two years in my last job, mostly data pipelines. | 0.81 | 2.05 (Regular use in a job) |
| I have written Python daily for eight years, including maintaining a large Django codebase. | 0.92 | 2.89 (Deep expertise) |

Noul은 "strong"이라는 명제 하나를 판단하며, 값은 그 명제가 참일 가능성입니다. 코드에서 0과 1 사이 범위에 단계를 만들 수는 있습니다. 예를 들어 0.3에서 0.7을 "some experience"로 둘 수 있습니다. 하지만 모델은 그 단계를 보지 못하므로, 답의 어떤 부분도 그 단계를 기준으로 판단된 것이 아닙니다. 중간값은 중간 정도의 경험을 뜻할 수도, 불분명한 경우를 뜻할 수도 있으며, 지원자 사이의 간격도 여러분이 정한 것이 아닙니다. Score는 각 단계 설명을 따로따로 판단하므로, 모든 지원자가 여러분이 작성한 단계 위나 그 근처에 놓였고, 반환된 확률은 모델이 판단을 단계 사이에 어떻게 나누었는지 보여 줍니다. 결과에 동의하지 않는다면 단계의 문구를 고쳐서 다시 실행하세요. 이 차이는 [질문 유형 선택하기](../primitives.md#choose-a-question-type)에서 설명합니다.

## Noul 질문 작성하기

Noul 하나에는 예/아니요 질문 하나만 물으세요. "Is the customer angry and asking for a refund?"처럼 질문에 조건이 두 개 있으면 모델이 둘을 한꺼번에 판단해야 해서 값의 의미가 흐려집니다. Noul 두 개로 묻고 코드에서 조합하세요.

높은 값이 예를 뜻하도록 질문을 표현하세요. "Does the message contain personal data?"는 분명합니다. "Is the message free of personal data?"는 의미를 뒤집으므로, 나중에 이 값을 읽는 코드가 거꾸로 해석하게 됩니다.

진술문도 질문만큼 잘 동작합니다. "The customer is requesting a refund"라면 1에 가까운 값은 이 진술이 참이라는 뜻입니다. 두 가지 표현을 여러분의 데이터로 모두 시험해 보고 어느 쪽이 더 나은지 확인하세요.

예와 아니요의 경계를 모호하지 않게 만드세요. "Does this candidate have any Python experience?"는 "any"가 중간 지대를 남기지 않으므로 잘 동작합니다. 경계가 미묘할 때는 위의 `is_repeat_contact` 질문처럼 `true`와 `false` 설명을 담은 `criteria`를 추가하세요. 대부분의 Noul은 instructions만으로 충분하므로, `criteria`가 있을 때와 없을 때를 모두 시험해 보고 여러분의 문서에서 더 나은 답을 내는 쪽을 쓰세요.

## 권장 사항: 요청 하나에 질문을 둘 이상 넣으세요

조건 체크리스트가 있다면 요청 하나에 Noul 질문을 여러 개 넣으세요. 조건마다 질문을 하나씩 두고, 그 조합이 무엇을 뜻하는지는 코드가 결정합니다. 질문은 병렬로 평가되므로 Noul을 추가해도 응답 시간은 거의 변하지 않습니다. 더 자세한 설명은 [여러 질문을 함께 묻기](../primitives.md#ask-multiple-questions-together)에 있습니다.

## 코드에서 여러 Noul 답 처리하기

위의 질문 두 개짜리 요청만으로도 코드가 메시지를 어디로 보낼지 정하기에 충분합니다. 아래 예제는 고객이 사람을 요청하면 사람에게 넘기고, 이전에 문의한 적이 있으면 우선순위를 높입니다. 어느 질문에서든 값이 중간에 걸치면 코드 경로가 아니라 검토자에게 보냅니다.

```python
from typesafe_sdk import Noul, NoulCriteria, TypeSafeClient

SUPPORT_QUESTIONS = {
    "is_human_escalation": Noul(
        instructions="Is the customer asking for a human agent?",
    ),
    "is_repeat_contact": Noul(
        instructions="Has the customer contacted support about this before?",
        criteria=NoulCriteria(
            true="Mentions a prior attempt, ticket, or that they have asked before",
            false="No sign of any previous contact",
        ),
    ),
}

YES = 0.7
NO = 0.3


def route(message: str) -> None:
    with TypeSafeClient() as client:
        response = client.system_one(
            model="jev-latest",
            state=message,
            questions=SUPPORT_QUESTIONS,
        )
    answers = response.answers

    wants_human = answers["is_human_escalation"].noul
    repeat = answers["is_repeat_contact"].noul

    if NO < wants_human < YES or NO < repeat < YES:
        # 모델이 어느 쪽으로도 확신하지 못합니다. 사람이 결정하게 합니다.
        send_to_review(message)
        return

    priority = "high" if repeat >= YES else "normal"
    if wants_human >= YES:
        route_to_agent(message, priority=priority)
    else:
        route_to_bot(message, priority=priority)
```

위 메시지의 경우 `is_human_escalation`의 noul 답 값은 0.99이고 `is_repeat_contact`는 0.93이므로, 코드는 이 메시지를 높은 우선순위로 상담원에게 보냅니다. "How do I reset my password?"라는 메시지는 두 질문 모두 0.07이며 봇으로 보내집니다.

threshold는 여러분의 코드에 있습니다. 검토자에게 가는 메시지가 너무 많으면 `NO`와 `YES` 사이의 간격을 좁히세요. 잘못 분배되는 메시지가 너무 많이 통과하면 간격을 넓히세요. 나중에 메시지가 결제를 언급하는지, 개인 정보가 들어 있는지 알아야 한다면 `SUPPORT_QUESTIONS`에 Noul을 하나 더 추가하세요. 호출 횟수는 그대로 한 번입니다.

## 구조화된 instructions

instructions는 문자열 대신 객체가 될 수 있으며, 이때 질문은 한 필드에, 보조 데이터는 다른 필드에 둡니다. 이 방식이 언제 도움이 되는지는 [질문을 구조화하세요](../concepts/how-to-build-with-system-one.md#use-structure-in-the-questions)에서 다룹니다. 여기서는 코드로 만든 질문에 사용합니다. 방금 도착한 이력서를 지원자 데이터베이스에서 같은 사람일 가능성이 있는 레코드들과 비교하는 예입니다. 각 레코드는 있는 그대로 `potential_duplicate` 필드에 들어가고, `question`은 모든 레코드에 똑같으며, 모든 레코드를 요청 하나로 확인합니다. 코드에서 생성한 질문 키에는 각 레코드의 데이터베이스 ID가 들어 있습니다.

Playground에서 실행해 보기 →

응답:

```json
{
  "model": "jev-1.13.0",
  "answers": {
    "same_as_record_18": {
      "type": "noul",
      "noul": 0.74
    },
    "same_as_record_42": {
      "type": "noul",
      "noul": 0.09
    },
    "same_as_record_77": {
      "type": "noul",
      "noul": 0.08
    }
  },
  "usage": {
    "input_tokens": 535,
    "output_tokens": 58
  }
}
```

각 답은 이력서가 해당 레코드에 있는 사람의 것일 확률입니다. 레코드 18은 이름 철자가 다르지만 지역과 고용주가 일치해 0.74를 받습니다. 레코드 42는 이름은 같지만 도시와 고용주가 달라 0.09를 받습니다. 레코드 77은 이름이 비슷하고 지역이 같지만 고용주가 달라 0.08을 받습니다. [코드에서 여러 Noul 답 처리하기](#handling-multiple-noul-answers-in-code)에서처럼 코드에서 각 값에 threshold를 적용하고, 중간값은 사람에게 보내세요.

Python SDK에서는 지원자 레코드로 질문을 만듭니다. 질문 문구는 고정되어 있고 레코드만 바뀝니다.

```python
from typesafe_sdk import Noul, TypeSafeClient

SAME_PERSON = "Is the resume for the same person as `potential_duplicate`?"


def duplicate_questions(candidates: list[dict]) -> dict[str, Noul]:
    """One Noul per candidate record, all asking the same question."""
    return {
        f"same_as_record_{candidate['id']}": Noul(
            instructions={
                "potential_duplicate": {
                    "name": candidate["name"],
                    "location": candidate["location"],
                    "last_employer": candidate["last_employer"],
                },
                "question": SAME_PERSON,
            },
        )
        for candidate in candidates
    }


def find_duplicates(resume: dict, candidates: list[dict]) -> list[str]:
    with TypeSafeClient() as client:
        response = client.system_one(
            model="jev-latest",
            state={"resume": resume},
            questions=duplicate_questions(candidates),
        )
    return [
        question_id
        for question_id, answer in response.answers.items()
        if answer.noul > 0.7
    ]
```

[구조화 데이터 추출 캐스케이드 cookbook(예제 모음)](../cookbooks/sde_cascade.md)은 구조화된 instructions로 추출한 레코드를 검증합니다. 모든 필드에 같은 질문 세트를 적용합니다. 각 질문의 `instructions` 객체는 `main_question` 속성에 질문 문구를 담습니다. 필드마다 바뀌는 `field_spec`과 `extracted_field` 속성도 있습니다.

## Cookbook에서 보는 Noul

Noul 질문을 사용하는 앱을 cookbook에서 살펴보세요.

* [병렬 질문](../cookbooks/parallel_questions.md)은 질문 13개로 이루어진 규제 체크리스트를 요청 하나로 문서 하나에 대해 실행합니다.
* [Self-consistency: nouls](../cookbooks/consistency_noul_cookbook.md)는 보험 청구 건 하나의 40가지 버전에 Noul 질문 14개를 던지고, 다시 물었을 때 결정이 얼마나 자주 바뀌는지 측정합니다.
* [Re-ranking(재순위화)](../cookbooks/rerank_typesafe.md)은 threshold가 아니라 확률 자체를 사용합니다. 질의와 후보의 쌍마다 Noul을 하나씩 던진 다음 그 값으로 후보를 정렬합니다.
* [한 줄씩 검색하기](../cookbooks/semantic_find.md)는 일치하는 줄을 찾는 Choice와, 문서에 답이 있기는 한지 확인하는 Noul을 짝지어 씁니다.
* [구조 복원](../cookbooks/autoformat.md)은 줄 쌍마다 줄바꿈이 문장을 갈라놓았는지 Noul 하나로 물어, 일반 텍스트에서 문단을 다시 만듭니다.


이 문서는 개발자 문서 플랫폼인 [Mintlify](https://mintlify.com)로 제작하고 호스팅합니다.
