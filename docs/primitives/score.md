# Score

> Score는 순서가 있고 말로 설명된 단계에 비추어 내용을 평가하는 System One 질문 유형입니다. 답에는 점수, 각 단계의 확률, confidence가 들어 있습니다.

Score는 답이 단계별로 설명할 수 있는 스펙트럼 위의 한 위치일 때 사용합니다. 예를 들어 버그가 얼마나 심각한지, 고객이 얼마나 만족하는지, 지원자의 Python 경험이 얼마나 되는지 등입니다. 답이 서로 순서가 없는 고정된 선택지 집합 중 하나라면 [Choice](choice.md)를 사용하세요. 예 또는 아니요라면 [Noul](noul.md)을 사용하세요. [질문 유형 선택하기](../primitives.md#choose-a-question-type)에서 세 가지를 모두 비교합니다.

Score의 답은 `score`에 담긴, 여러분이 정한 단계 위의 한 위치이며 두 단계 사이에 놓일 수도 있습니다. 모델은 `probabilities`에 모든 단계의 확률을, 그리고 답에 대한 `confidence`(신뢰도) 값도 함께 반환합니다.

Score 질문 예시

질문 예시

단계

State (평가할 내용)

답

각 단계의 확률

Confidence

score

<details markdown="1">
  <summary>score와 confidence를 계산하는 방법</summary>

  Score:

  각 단계 번호에 그 단계의 확률을 곱한 다음 결과를 모두 더합니다.

  ≈

  Confidence:

  TypeSafe는 확률이 단계들에 걸쳐 얼마나 퍼져 있는지로 이 값을 계산합니다. 확률이 전부 한 단계에 몰려 있으면 1.0이고, 고르게 퍼질수록 confidence는 낮아집니다.
</details>

각 단계 앞의 숫자는 위치이며, [단계](#levels)에서 설명합니다.

## 요청 구조

[TypeSafe API](https://docs.typesafe.ai/api)로 보내는 POST 요청 본문의 최상위 필드 세 개는 다른 질문 유형과 같습니다. 평가할 내용인 `state`(상태), `model`, 그리고 `questions`입니다. 각 Score 질문에는 다음 필드가 있습니다.

* `type`: 항상 `"score"`입니다.
* `instructions`: 모델이 답할 질문입니다. 무엇을 평가하는지 적습니다.
* `criteria`: 척도의 낮은 쪽 끝에서 높은 쪽 끝까지 순서대로 나열한 단계 설명 배열입니다. 단계가 적어도 두 개는 있어야 하며, API는 10개까지 받습니다.

아래 요청에서 state는 버그 리포트이고, 질문은 이 버그가 얼마나 심각한지입니다.

Playground에서 실행해 보기 →

질문 id는 여러분이 정하며, 여기서는 `bug_severity`입니다. 이 id는 모델에 전달되지 않습니다. 답은 같은 id 아래에 반환됩니다.

### 단계

`criteria`(판단 기준)의 각 항목이 단계입니다. 가능한 답의 스펙트럼 위에 있는 한 지점을 말로 설명한 것입니다. 단계의 번호는 `criteria` 배열 안에서의 위치이며 0부터 시작하므로, 위의 세 항목은 단계 0, 1, 2입니다. 배열의 순서가 곧 번호입니다.

모델은 설명만 받고 그 밖의 것은 받지 않으며, 각 단계는 state를 대상으로 따로따로 판단됩니다.

응답의 `score`는 단계 스펙트럼 위의 한 위치입니다. 세 단계짜리 척도에서는 0에서 2까지 움직이며, 두 단계 사이에 놓일 수 있습니다.

[클라이언트 SDK](../sdk.md)는 타입이 지정된 질문을 제공합니다. Python에서는 같은 질문을 `Score`로 씁니다.

```python
from typesafe_sdk import Score, TypeSafeClient

with TypeSafeClient() as client:
    response = client.system_one(
        state="The export button crashes the settings page in Safari. It works in Chrome, but a few of our customers only use Safari.",
        questions={
            "bug_severity": Score(
                instructions="How severe is the reported issue?",
                criteria=[
                    "Cosmetic; no impact to functionality",
                    "Broken or degraded feature, but workaround exists",
                    "Blocking issue; no workaround exists",
                ],
            ),
        },
    )

    print(response.answers["bug_severity"].score)
```

Jev 같은 System One 모델을 호출하려면 `system_one` 메서드나 `https://api.typesafe.ai/v1/systemone` 엔드포인트를 사용하세요. `model` 필드로 요청을 처리할 모델을 고릅니다. 코드의 어느 지점에서 호출해야 하는지는 [TypeSafe로 구축하는 방법](../concepts/how-to-build-with-system-one.md)에서 다룹니다.

[클라이언트 SDK](../sdk.md) 중 하나를 사용하거나 [TypeSafe API](https://docs.typesafe.ai/api)를 직접 호출하세요. 코딩 에이전트에 연동 코드 작성을 맡긴다면, 에이전트가 요청과 응답의 형태를 알 수 있도록 먼저 [TypeSafe 에이전트 스킬](../agent-skill.md#installation)을 설치하세요.

> [!NOTE]
>
> `instructions`(지시문)와 `criteria`의 각 단계는 문자열, 객체, 배열 중 어느 것이든 될 수 있습니다. 문자열로 시작하세요. 단계에 설명과 함께 예시 상황 몇 가지가 필요할 때는 객체를 쓰세요. 아래의 [구조화된 단계 설명](#structured-level-descriptions)과 [API 레퍼런스](https://docs.typesafe.ai/api)를 참고하세요.

## 응답 구조

응답의 `answers`에는 질문마다 항목이 하나씩 있으며, 요청에서 쓴 id 아래에 들어 있습니다. 다음은 위 예시 요청에 대한 응답입니다.

```json
{
  "model": "jev-1.13.0",
  "answers": {
    "bug_severity": {
      "type": "score",
      "score": 1.43,
      "confidence": 0.35,
      "legend": {
        "0": "Cosmetic; no impact to functionality",
        "1": "Broken or degraded feature, but workaround exists",
        "2": "Blocking issue; no workaround exists"
      },
      "probabilities": {
        "0": 0.0,
        "1": 0.57,
        "2": 0.43
      }
    }
  },
  "usage": {
    "input_tokens": 332,
    "output_tokens": 18
  }
}
```

각 Score 답에는 다섯 가지 값이 있습니다.

* `type`: System One 질문의 유형입니다.
* `probabilities`: 각 단계의 확률이며, 문자열로 된 단계 번호가 키입니다. 모든 값의 합은 1입니다.
* `score`: 단계 번호를 늘어놓은 수직선 위의 위치이며, 0에서 최상위 단계 번호(여기서는 2)까지입니다. 각 단계 번호에 그 확률을 곱해 모두 더한 값입니다. 0 x 0.0 + 1 x 0.57 + 2 x 0.43 = 1.43.
* `legend`: 각 단계 번호를 그 설명에 다시 대응시킨 것입니다.
* [`confidence`](../confidence.md): `probabilities`가 얼마나 퍼져 있는지로 계산한 0에서 1 사이의 숫자입니다. 한 단계에만 봉우리가 솟아 있으면 confidence가 높습니다. 확률이 여러 단계에 퍼져 있으면 confidence가 낮습니다.

점수 1.43은 모델의 판단이 단계 1과 2 사이에서 갈리되 단계 1 쪽으로 기울어 있다는 뜻입니다. 이는 리포트 내용과 맞아떨어집니다. 내보내기 기능이 고장 났고, 대부분의 고객은 Chrome으로 바꾸면 우회할 수 있지만 Safari만 쓰는 고객은 그럴 수 없습니다. 모델은 "workaround exists"에 0.57, "no workaround"에 0.43을 두었고, 판단이 갈렸으므로 confidence는 0.35입니다.

Python SDK에서 `ScoreAnswer`는 `score`, `confidence`, `probabilities`, `legend`를 타입이 지정된 필드로 갖습니다. SDK는 `probabilities`와 `legend`의 키로 문자열이 아니라 정수 단계 번호를 씁니다.

## Score 읽는 법

입력에 따라 점수가 어떻게 달라지는지 살펴보겠습니다. 예를 들어 위 요청의 질문과 단계를 그대로 사용합니다.

```
"How severe is the reported issue?"
  → 0: Cosmetic; no impact to functionality
  → 1: Broken or degraded feature, but workaround exists
  → 2: Blocking issue; no workaround exists
```

버그 리포트에 따라 점수가 어떻게 바뀌는지 볼 수 있습니다.

<table>
  <thead>
    <tr>
      <th colSpan="3" />

      <th colSpan="3">`probabilities`</th>
    </tr>

    <tr>
      <th>State</th>
      <th>`score`</th>
      <th>`confidence`</th>
      <th>단계 0</th>
      <th>단계 1</th>
      <th>단계 2</th>
    </tr>
  </thead>

  <tbody>
    <tr>
      <td>The export button is misaligned by a few pixels on the settings page.</td>
      <td>0.0</td><td>1.0</td><td>1.0</td><td>0.0</td><td>0.0</td>
    </tr>

    <tr>
      <td>The PDF export button does nothing when clicked. I can still export to CSV and convert it myself, but that takes ages.</td>
      <td>1.0</td><td>1.0</td><td>0.0</td><td>1.0</td><td>0.0</td>
    </tr>

    <tr>
      <td>Export to PDF fails with a spinner that never finishes. Some of our team say CSV export still works for them, others say it fails too.</td>
      <td>1.11</td><td>0.84</td><td>0.0</td><td>0.89</td><td>0.11</td>
    </tr>

    <tr>
      <td>The export button crashes the settings page in Safari. It works in Chrome, but a few of our customers only use Safari.</td>
      <td>1.43</td><td>0.35</td><td>0.0</td><td>0.57</td><td>0.43</td>
    </tr>

    <tr>
      <td>Nobody on our team can log in since this morning. We get a 500 error on every attempt.</td>
      <td>2.0</td><td>1.0</td><td>0.0</td><td>0.0</td><td>1.0</td>
    </tr>
  </tbody>
</table>

이 예시들에서 confidence 1.0은 반환된 분포가 확률을 전부 한 단계에 두었다는 뜻입니다. 이는 모델의 답이 어떤 모양인지를 말해 줄 뿐, 답이 맞는다는 보장은 아닙니다.

점수는 단계 번호를 확률로 가중 평균한 값입니다. 세 번째와 네 번째 예시에서는 확률이 단계 1과 2로 나뉩니다. 단계 2에 무게가 더 실릴수록 점수가 올라갑니다. 우회 방법이 없는 고객의 비율을 측정하는 값은 아닙니다.

서로 다른 분포가 같은 점수를 낼 수 있습니다. 점수 1.0은 확률이 전부 단계 1에 있다는 뜻일 수도 있고, 단계 0과 2에 절반씩 있다는 뜻일 수도 있습니다. 이런 경우를 구별하려면 점수와 함께 `probabilities`와 `confidence`를 읽으세요.

소수점이 있는 점수는 위치입니다. 리포트를 심각도순으로 정렬하는 데 쓸 수도 있고, 코드에 결과 하나가 필요할 때는 가장 가까운 단계로 반올림할 수도 있습니다. [엔터티 정렬 cookbook(예제 모음)](../cookbooks/entity_alignment.md)은 가장 가까운 단계로 반올림해 결정을 내리는 예를 보여 줍니다.

Score의 confidence가 낮다면 보통 셋 중 하나입니다. 이 state에서는 단계들이 겹치거나, 질문이 둘 이상을 측정하고 있거나, state에 위치를 정할 만큼 정보가 충분하지 않은 경우입니다. 코드에서 confidence를 활용하는 방법은 [Confidence](../confidence.md) 문서에서 다룹니다.

## 좋은 단계 작성하기

정도가 아니라 상황을 묘사하세요. "Broken or degraded feature, but workaround exists"는 모델이 state와 맞춰 볼 무언가를 줍니다. "Moderately severe"는 그렇지 않습니다. 구체적인 설명은 모델이 단계를 구별하는 데 도움이 될 수 있습니다. 답은 정답을 아는 예시와 대조해 확인하세요. confidence가 높아졌다는 사실만으로는 설명이 더 나아졌다고 볼 수 없습니다.

모든 단계는 따로따로 평가됩니다. 모델은 단계의 번호나 이웃 단계를 보지 못하므로 "worse than the previous level" 같은 표현은 모델에 아무 의미가 없고, 설명이나 instructions에 숫자를 넣어도 도움이 되지 않습니다. 위 표의 버튼 정렬 어긋남 리포트에 대해 단계를 숫자로만 썼을 때 일어나는 일은 다음과 같습니다.

```
instructions: "Rate severity from 0 to 2, where 2 is worst"
criteria: ["0", "1", "2"]
→ score 0.55, confidence 0.33, probabilities 0: 0.45, 1: 0.55, 2: 0.0
```

같은 리포트를 설명이 있는 세 단계로 평가하면 confidence 1.0으로 0.0점이 나옵니다. 숫자만 있으면 모델은 맞춰 볼 대상이 없어서 확률을 0과 1로 나눕니다.

단계는 뚜렷이 구별되게 설명할 수 있는 만큼, 최대 10개까지 사용하세요. 세 개여도 괜찮습니다. 뚜렷이 구별되게 설명할 수 없는 단계는 추가하지 마세요.

Score 질문 하나는 한 가지 차원만 다루게 하세요. 설명에 "punctual and smart and experienced"라고 쓰여 있다면 질문은 세 가지를 측정하는 셈이고, 하나는 높고 다른 하나는 낮은 입력은 위치를 정할 수 없습니다. confidence가 떨어지고 점수의 의미도 흐려집니다. 측정 대상마다 Score 질문을 하나씩 두도록 나누고, 다음 절에서 보듯 코드에서 조합하세요.

척도의 맨 위에 다르게 대응해야 하는 드문 극단적 사례가 있다면 그 사례에 별도의 단계를 주세요. "very angry"에서 끝나는 감정 척도에는 "abusive or threatening"을 추가할 수 있습니다. 이 단계가 없으면 두 메시지가 모두 최상단에 가까운 점수를 받을 수 있습니다. 점수만으로는 둘을 구별하지 못할 수 있습니다.

중간이 전혀 없고 답이 몇 개의 이산적인 카테고리 중 하나라면 대신 [Choice](choice.md)를 사용하거나 질문을 여러 [Noul](noul.md) 질문으로 나누세요. 단계를 여러분의 데이터로 직접 시험해 보는 것이 중요합니다. 같은 척도라도 문구가 다르면 여러분의 데이터에서 다르게 동작할 수 있습니다.

## 복잡한 판단을 여러 Score 질문으로 나누기

여러 요소에 좌우되는 복잡한 판단은 요소마다 Score 질문을 하나씩 두도록 나누는 것이 가장 좋습니다. 그런 다음 Jev가 반환한 Score들을 코드에서 조합해 판단을 내릴 수 있습니다. 어떤 Score 질문은 다른 질문보다 더 중요할 수 있으므로, 각 Score 질문에 상대적 중요도에 따른 가중치를 주세요. 가중치는 여러분이 정합니다. 조합한 결과가 팀이 내렸을 결정과 맞지 않으면 코드에서 가중치를 바꾸고 다시 실행하세요. Score 질문들은 요청 하나에 담아 보내세요. 병렬로 평가됩니다. 질문을 추가해도 응답 시간은 거의 변하지 않고, 질문 token(토큰)이 조금 더 들 뿐입니다. [여러 질문을 함께 묻기](../primitives.md#ask-multiple-questions-together)를 참고하세요.

아래 요청은 위 표의 스피너 티켓에 컨텍스트를 조금 더한 것입니다. Score 질문 세 개를 묻습니다. 버그가 얼마나 심각한지, 고객이 얼마나 불만스러워하는지, 리포트가 엔지니어에게 작업할 단서를 얼마나 주는지입니다.

Playground에서 실행해 보기 →

Jev의 응답:

```json
{
  "model": "jev-1.13.0",
  "answers": {
    "severity": {
      "type": "score",
      "score": 1.24,
      "confidence": 0.64,
      "legend": {
        "0": "Cosmetic; no impact to functionality",
        "1": "Broken or degraded feature, but workaround exists",
        "2": "Blocking issue; no workaround exists"
      },
      "probabilities": {
        "0": 0.0,
        "1": 0.76,
        "2": 0.24
      }
    },
    "frustration": {
      "type": "score",
      "score": 1.28,
      "confidence": 0.58,
      "legend": {
        "0": "Calm, just stating facts",
        "1": "Frustrated but civil",
        "2": "Very angry, strong language or threatening to leave"
      },
      "probabilities": {
        "0": 0.0,
        "1": 0.72,
        "2": 0.28
      }
    },
    "report_quality": {
      "type": "score",
      "score": 3.0,
      "confidence": 1.0,
      "legend": {
        "0": "No detail; just says something is broken",
        "1": "Names the feature but no steps or environment",
        "2": "Steps to reproduce or environment, but not both",
        "3": "Steps to reproduce and environment"
      },
      "probabilities": {
        "0": 0.0,
        "1": 0.0,
        "2": 0.0,
        "3": 1.0
      }
    }
  },
  "usage": {
    "input_tokens": 468,
    "output_tokens": 43
  }
}
```

각 질문은 티켓을 대상으로 따로따로 답이 매겨지고 점수를 받습니다.

* `severity`는 confidence 0.64로 1.24입니다. 첫 예제와 같은 해석입니다. 내보내기 기능이 고장 났고 일부는 우회 방법이 있습니다.
* `frustration`은 confidence 0.58로 1.28입니다. 표현은 정중하지만 "third time"과 "I'm done"이 점수의 일부를 최상위 단계 쪽으로 끌어올려, 모델은 "frustrated but civil"과 "very angry"에 0.72와 0.28을 나누어 둡니다. 이 티켓에서는 두 단계가 겹치며, confidence가 중간 수준인 이유가 여기에 있습니다.
* `report_quality`는 confidence 1.0으로 3.0입니다. 재현 단계와 브라우저 버전이 모두 적혀 있습니다.

세 척도는 길이가 서로 다르므로, 조합하기 전에 각 점수를 정규화하세요. 네 단계짜리 척도는 0에서 3을, 세 단계짜리 척도는 0에서 2를 반환하므로 한쪽의 최고 점수가 다른 쪽의 최고 점수보다 큽니다. 각 점수를 최상위 단계 번호인 `len(criteria) - 1`로 나누어 모든 점수를 0에서 1 사이에 놓으세요. 그러면 가중치가 말 그대로의 의미를 갖습니다. severity에 0.6, frustration에 0.3을 주면 severity가 두 배로 반영됩니다.

아래 TypeSafe Python SDK 코드는 세 질문을 묻고, 각 점수를 정규화하고, 예시 우선순위 계산식으로 조합합니다.

```python
from typesafe_sdk import Score, TypeSafeClient

TRIAGE_QUESTIONS = {
    "severity": Score(
        instructions="How severe is the reported issue?",
        criteria=[
            "Cosmetic; no impact to functionality",
            "Broken or degraded feature, but workaround exists",
            "Blocking issue; no workaround exists",
        ],
    ),
    "frustration": Score(
        instructions="How frustrated is the customer?",
        criteria=[
            "Calm, just stating facts",
            "Frustrated but civil",
            "Very angry, strong language or threatening to leave",
        ],
    ),
    "report_quality": Score(
        instructions="How much does the report give an engineer to work with?",
        criteria=[
            "No detail; just says something is broken",
            "Names the feature but no steps or environment",
            "Steps to reproduce or environment, but not both",
            "Steps to reproduce and environment",
        ],
    ),
}


def normalized(answers, question_id: str) -> float:
    """Put a score on 0 to 1 by dividing by its top level number."""
    top_level = len(TRIAGE_QUESTIONS[question_id].criteria) - 1
    return answers[question_id].score / top_level


def priority(ticket: str) -> float:
    with TypeSafeClient() as client:
        response = client.system_one(
            state=ticket,
            questions=TRIAGE_QUESTIONS,
        )
    answers = response.answers

    severity = normalized(answers, "severity")
    frustration = normalized(answers, "frustration")
    report_quality = normalized(answers, "report_quality")

    # 리포트가 자세하면 엔지니어가 조사하기 쉬우므로 우선순위를 조금 높입니다.
    return 0.6 * severity + 0.3 * frustration + 0.1 * report_quality
```

위 예시 응답의 경우 정규화된 점수는 severity가 0.62, frustration이 0.64, report quality가 1.0입니다. 우선순위는 `0.6 × 0.62 + 0.3 × 0.64 + 0.1 × 1.0 = 0.664`이며 반올림하면 `0.66`입니다.

가중치는 여러분의 코드에 있으므로, 숫자가 어떻게 만들어지는지 정확히 볼 수 있고 순위가 팀의 판단과 맞지 않을 때 바꿀 수 있습니다. 나중에 Score 질문이 더 필요하면 `TRIAGE_QUESTIONS`에 추가하세요. 호출 횟수는 그대로 한 번입니다. 복잡한 판단을 별개의 Score로 쪼갠 다음 코드에서 가중치로 조합하는 이 기법을 [Composite scoring](../patterns/composite-scoring.md) 패턴이라고 합니다.

## 구조화된 단계 설명

각 단계를 기본적인 텍스트 설명으로 시작하세요. 여러분이 보기에 분명한 입력인데도 모델이 계속 이웃한 두 단계 사이의 점수를 낸다면, 각 단계에 문자열 대신 객체를 주세요. 그 단계가 무엇을 다루는지 적는 필드와 예시 상황 몇 가지를 담는 필드를 둡니다. 모델이 같은 종류끼리 비교할 수 있도록 모든 단계에 같은 필드 이름을 쓰세요.

아래 요청은 앞에서 쓴 스피너 티켓이지만 각 단계에 예시를 붙였습니다.

Playground에서 실행해 보기 →

응답:

```json
{
  "model": "jev-1.13.0",
  "answers": {
    "bug_severity": {
      "type": "score",
      "score": 1.09,
      "confidence": 0.87,
      "legend": {
        "0": {
          "what": "Cosmetic; no impact to functionality",
          "examples": [
            "typo in a label",
            "misaligned icon"
          ]
        },
        "1": {
          "what": "Broken or degraded feature, but workaround exists",
          "examples": [
            "export fails in one browser but works in another"
          ]
        },
        "2": {
          "what": "Blocking issue; no workaround exists",
          "examples": [
            "cannot log in",
            "data loss"
          ]
        }
      },
      "probabilities": {
        "0": 0.0,
        "1": 0.91,
        "2": 0.09
      }
    }
  },
  "usage": {
    "input_tokens": 379,
    "output_tokens": 18
  }
}
```

일반 문자열을 썼을 때 이 티켓은 confidence 0.84로 1.11점을 받았습니다. 예시를 붙이면 confidence 0.87로 1.09점을 받습니다. 일반 문자열만으로도 이미 위치가 잘 잡혔기 때문에 변화는 작습니다. 다음 표에서 보듯, 일반 문자열로는 모델의 판단이 갈리는 경우에 효과가 더 큽니다.

예시는 모델을 특정 방향으로 이끌며, 실제 입력과 닮았을 때만 도움이 됩니다. 아래 표는 첫 예제의 Safari 리포트를 서로 다른 세 가지 단계 객체 세트로 평가한 결과입니다.

| 단계 설명 | `score` | `confidence` |
| - | - | - |
| 일반 문자열: 예시가 든 객체 없음 | 1.43 | 0.35 |
| 유용한 예시를 담은 examples 배열 추가: "export fails in one browser but works in another" | 1.03 | 0.96 |
| 브라우저와 무관한 예시를 담은 examples 배열 추가: "search fails, but browsing categories still works" | 1.43 | 0.35 |

이 비교에서 입력과 맞아떨어지는 예시는 거의 모든 확률을 한 단계에 집중시킵니다. 무관한 예시는 일반 문자열과 같은 결과를 냅니다. confidence가 더 높다고 해서 어느 답이 맞는지가 입증되지는 않습니다. 기대하는 단계를 이미 아는 예시를 고르고, 수정한 설명을 채택하기 전에 별도의 입력으로 시험해 보세요.


이 문서는 개발자 문서 플랫폼인 [Mintlify](https://mintlify.com)로 제작하고 호스팅합니다.
