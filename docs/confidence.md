# Confidence

> Jev가 확실성을 어떻게 알려 주는지, 확률과는 어떻게 다른지, 그리고 이를 이용해 시스템 동작을 어떻게 제어하는지 설명합니다.

Jev가 반환하는 모든 Score와 Choice 답에는 `probabilities` 속성이 들어 있습니다. 이 속성은 선택지(Choice의 경우) 또는 단계(Score의 경우)에 걸친 확률 분포를 나타냅니다. 모델이 얼마나 확신하는지는 이 분포의 *형태*에서 드러납니다. 확률이 한 결과에 몰려 있으면 확신하는 답이고, 넓게 퍼져 있으면 불확실한 답입니다.

답의 `confidence`(신뢰도) 속성은 그 형태를 0에서 1 사이의 숫자 하나로 압축한 값입니다. 모든 확률이 한 결과에 몰려 있으면 1이고, 확률이 고르게 퍼져 있으면 0입니다. (Noul 답에는 이 값이 없습니다. 아래 [Noul](#noul)을 참고하세요.) 이 값은 답이 얼마나 확실한지를 설명하지만, 코드가 행동의 근거로 삼기에 항상 알맞은 숫자인 것은 아닙니다.

## Confidence는 확률에서 도출됩니다

`confidence`는 답에 이미 들어 있는 확률 분포로부터 계산한 통계량입니다. TypeSafe가 대신 계산해서 모든 Choice와 Score 답에 담아 반환하므로, 일반적인 경우라면 따로 할 일이 없습니다.

선택지가 세 개인 Choice 질문

확률 분포에 따라 confidence가 어떻게 달라지는지 확인해 보세요

Confidence

확률

슬라이더를 움직여 선택지의 확률을 바꿔 보세요. 합계가 100%로 유지되도록 나머지 확률이 함께 조정됩니다.

뚜렷한 1위
넓게 퍼짐
균등 분할

<details markdown="1">
  <summary>이 데모가 Confidence를 계산하는 방법</summary>

  TypeSafe는 확률이 선택지들에 어떻게 퍼져 있는지를 보고 confidence를 계산합니다. 확률이 전부 한 선택지에 있으면 1.0이고, 고르게 퍼질수록 confidence는 낮아집니다. 선택지가 세 개일 때 [Choice 공식](#choice)은 `(3 × largest probability − 1) / 2`이며, 이 데모가 보여 주는 값이 바로 이것입니다.
</details>

[Choice](primitives/choice.md)에서는 선택지들에 걸친 `probabilities`가 이 분포입니다. [Score](primitives/score.md)에서는 단계들에 걸친 분포입니다. 두 경우 모두 분포가 평평할수록 confidence가 낮습니다. Choice의 confidence가 낮다면 대개 다른 선택지를 뚜렷하게 앞서는 선택지가 없다는 뜻이고, Score의 confidence가 낮다면 대개 단계 정의가 모호하거나, 여러 차원이 섞여 있거나, state(상태)에 판단할 근거가 충분하지 않다는 뜻입니다.

질문 유형별 정확한 공식은 [Confidence 계산 방법](#how-confidence-is-calculated)에 있습니다. 어떤 `confidence` 값이든 답 자체의 `probabilities`에서 어떻게 나오는지 확인할 수 있습니다.

## "모르겠습니다"는 유용한 신호입니다

사람이든 기계든, 지능을 가진 시스템이 불확실성을 솔직하게 표현하지 못한다면 그 시스템은 신뢰할 수 없습니다.

Confidence는 모델이 "이건 잘 모르겠습니다"라고 말할 수 있는 장치를 기본으로 제공합니다. 덕분에 코드에서 확실성 수준에 따라 다른 동작을 구현할 수 있고, 이것이 실제로 믿고 맡길 수 있는 시스템을 만드는 토대가 됩니다.

## 코드에서 confidence를 활용하는 세 가지 경로

처음에는 confidence를 세 구간으로 나누고 구간마다 시스템이 다르게 동작하도록 하는 패턴이 유용합니다.

**높은 confidence:** 자동으로 실행합니다. 모델이 상황을 분명하게 파악했으므로 사람이 개입하지 않고 진행해도 됩니다.

**중간 confidence:** 신중하게 진행합니다. 모델이 그럴듯한 답을 내놓았지만 확신하지는 못합니다. 상황에 따라 사용자에게 확인을 요청하거나, 검토 대상으로 표시하거나, 실행하기 전에 정보를 더 모을 수 있습니다.

**낮은 confidence:** 실행하지 않습니다. 사람에게 넘기거나, 추가 설명을 요청하거나, 다른 시스템으로 대체합니다. 정보가 충분하지 않거나 질문이 잘 맞지 않는다고 모델이 알려 주는 것입니다.

이 경계를 어디에 그을지는 걸려 있는 위험의 크기에 따라 달라집니다.

## Threshold는 위험에 비례합니다

confidence threshold(임계값)는 숫자 하나로 정해지지 않습니다. 같은 시스템 안에서도 동작마다, 잘못 판단했을 때의 결과에 따라 서로 다른 threshold를 써야 합니다.

```python
response = client.system_one(
    state=user_message,
    questions={
        "action": Choice(
            instructions="What is the user trying to do?",
            criteria={
                "check_balance": "View account balance",
                "approve_transfer": "Approve the pending withdrawal request",
                "support": "Get help with an issue",
            },
        ),
    },
)

action = response.answers["action"]
confidence = action.confidence

if confidence < 0.5:
    # 모델이 정말로 확신하지 못합니다. 추측하지 않습니다.
    route_to_human(user_message)

elif action.choice == "check_balance":
    # 위험이 낮습니다. 잘못된 화면을 보여 줘도 되돌릴 수 있습니다.
    show_balance(account_id)

elif action.choice == "approve_transfer":
    if confidence > 0.9:
        # 위험이 높고 confidence도 높습니다. 확인을 거쳐 진행합니다.
        confirm_then_execute(account_id)
    else:
        # 위험이 높고 confidence는 중간입니다. 먼저 검증합니다.
        ask_user_to_confirm(account_id)
```

confidence 하한인 0.5는 모델이 정말로 불확실하다고 알려 온 경우를 모두 걸러 냅니다. 그 위에서는 확인 없이 실행하기 위한 threshold가 읽기 전용 작업보다 파괴적인 작업에서 더 높습니다. 위험 허용 범위를 코드가 담고 있는 것입니다.

> [!NOTE]
>
> 올바른 threshold 값은 도메인과, 해당 사용 사례에서 모델이 내는 성능에 따라 달라집니다. 보수적인 threshold로 시작해서 직접 가진 데이터로 테스트하고, 결과를 관찰하면서 조정하세요.

## Confidence를 근거로 행동해야 할 때

Confidence는 답의 확률이 얼마나 집중되어 있는지를 측정하며, 이는 선택된 답이 정답일 확률과는 다릅니다. threshold를 걸기 전에 코드가 무엇을 알아야 하는지부터 정하세요. Choice에서 confidence 0.5는 선택지가 두 개일 때와 열 개일 때 서로 다른 최고 확률에 해당합니다.

| 선택지 수 | 최고 확률 | `confidence` |
| - | - | - |
| 2 | 0.75 | 0.5 |
| 3 | 0.67 | 0.5 |
| 10 | 0.55 | 0.5 |

선택지 수가 고정된 Choice 질문에서 confidence는 최고 확률을 변환한 값이므로, 둘을 모두 확인해도 독립적인 근거가 더해지지 않습니다. 2위가 바짝 따라붙은 경우와 확률이 여러 대안에 퍼진 경우를 구별하려면 나머지 확률이나 [1위 대 2위 비율](#choice)을 살펴보세요.

대신 threshold를 걸 대상은 다음과 같습니다.

* **Noul이라면 값 자체입니다.** 이 값이 이미 진술이 참일 확률입니다. 시작점으로 삼기 좋은 threshold는 0.7과 0.3입니다. 0.7보다 높으면 '예', 0.3보다 낮으면 '아니요'로 읽습니다. 0.3과 0.7 사이의 구간은 어느 한쪽으로 반올림하지 말고 코드에서 별도의 결과로 다루세요.
* **Choice라면 코드가 실행에 옮길 선택지의 확률입니다.** 이 값은 고른 선택지가 정답일 확률이므로, 여기에 건 threshold는 누구나 읽을 수 있는 위험에 관한 진술이 됩니다. 확률이 `{"check_balance": 0.05, "approve_transfer": 0.92, "support": 0.03}`인 답에서 0.92는 사용자가 `approve_transfer`를 의도했을 확률이고, 0.9보다 높을 때만 실행하는 규칙은 "잘못된 선택지일 가능성이 10% 미만일 때 실행한다"는 뜻입니다. [확률 기반 routing(라우팅)](patterns/probability-routing.md)은 이런 방식으로 동작마다 다른 threshold를 설정합니다.
* **한 단계에 따라 결정이 갈리는 Score라면 그 단계의 확률입니다.** 위치 값이 두 단계 사이에 놓이면서, 서로 떨어진 두 단계에 확률이 높게 몰린 갈라진 답을 가릴 수 있습니다. 인접한 단계에 놓인 확률은 confidence를 거의 낮추지 않으므로 confidence로도 이 갈림은 드러나지 않습니다([두 단계가 답을 나눠 가질 때](patterns/confidence-routing.md#when-two-levels-split-the-answer)).

Score 답이 애초에 한쪽으로 정해졌는지를 코드가 알아야 한다면 confidence를 먼저 확인하세요. Score의 confidence는 확률이 가장 유력한 단계에서 얼마나 멀리 떨어져 있는지를 측정하므로, 위치 값만으로는 잡아내지 못하는, 멀리 떨어진 단계들에 퍼진 답을 잡아냅니다. [Confidence 기반 routing](patterns/confidence-routing.md)은 confidence를 먼저 확인한 다음 score에 따라 routing합니다.

Confidence는 질문들 사이의 불확실성을 비교할 때도 유용합니다. 선택지나 단계가 몇 개든 똑같이 0에서 1 사이의 척도를 쓰기 때문입니다.

## Confidence 계산 방법

질문 유형마다 분포를 요약하는 방식이 조금씩 다릅니다. Noul은 결과가 두 개이고, Choice는 순서 없는 선택지가 몇 개든 있을 수 있으며, Score의 단계는 순서가 있습니다. 그래서 아래의 각 공식은 바로 앞 공식을 바탕으로 확장됩니다.

TypeSafe의 `confidence`는 분포를 요약하는 합리적인 방법 중 하나일 뿐, 유일한 방법은 아닙니다. 고정된 척도를 반환하는 이유는 모든 답에 무난한 기본값이 따라오도록 하기 위해서입니다. 통계량을 직접 고르고 검증하지 않아도, 첫 호출부터 모든 질문에 똑같은 0에서 1 사이의 척도로 `confidence`에 threshold를 걸 수 있습니다. 아래 공식은 정확한 정의이고 모든 답에 전체 `probabilities`가 들어 있으므로, 애플리케이션에 가장 중요한 척도가 따로 있다면 그것을 직접 계산해서 쓸 수도 있습니다.

### Noul

Noul 답은 답이 '예'일 확률 $p$ 하나이며, TypeSafe는 이에 대해 별도의 `confidence`를 반환하지 않습니다. 확률이 이미 불확실성을 담고 있기 때문입니다. 0.5에 가까운 값은 모델이 잘 모르겠다고 말하는 것입니다. 시작점으로는 '예'에 0.7, '아니요'에 0.3이 적당하며, 이 값을 언제 옮겨야 하는지는 [Noul](primitives/noul.md#reading-a-noul) 페이지에서 다룹니다.

그래도 confidence와 같은 형태의 숫자가 필요하다면, 예를 들어 Noul과 Choice에 같은 코드로 threshold를 걸고 싶다면, 0.5로부터의 거리를 사용하세요.

$$
\text{confidence} = |2p - 1|
$$

이 값은 $p = 0.5$일 때 0이고 $p = 0$ 또는 $p = 1$일 때 1입니다. 아래의 Choice 공식을 '예/아니요' Choice에 적용한 것과도 같으므로 Choice의 confidence와 같은 척도 위에 놓입니다.

### Choice

Choice는 같은 발상을 선택지가 몇 개든 적용할 수 있도록 확장합니다. 선택지가 $n$개인 Choice에서 $p_{\max}$가 선택된 선택지의 확률일 때 공식은 다음과 같습니다.

$$
\text{confidence} = \frac{p_{\max} - \frac{1}{n}}{1 - \frac{1}{n}}
$$

이 공식은 최고 확률이 선택지당 $\frac{1}{n}$인 균등 분할보다 얼마나 높은지를, 균등 분할이 0이고 확실한 상태가 1인 척도로 측정합니다. 최고 확률만 반영되므로 $(0.6, 0.3, 0.1)$과 $(0.6, 0.2, 0.2)$는 confidence가 둘 다 0.4입니다.

```python
def choice_confidence(probabilities: list[float]) -> float:
    n = len(probabilities)
    return (max(probabilities) - 1 / n) / (1 - 1 / n)


choice_confidence(list(answer.probabilities.values()))
```

[이 페이지 맨 위의 탐색기](#confidence-is-derived-from-the-probabilities)는 선택지가 세 개일 때의 이 공식을 사용합니다.

같은 `probabilities`에서 계산하는 더 단순한 척도 두 가지도 실제로는 매우 효과적인 경우가 많으므로 `confidence`와 함께 시험해 볼 만합니다.

* **최고 확률**, $p_{\max}$. "선택된 선택지가 맞을 가능성이 얼마나 되는가"로 바로 읽히므로 threshold를 놓고 판단하기 쉽습니다. 다만 그 의미가 선택지 수에 따라 달라집니다. 0.5는 선택지가 두 개일 때는 약한 답이고 열 개일 때는 강한 답이므로, threshold는 질문마다 따로 정하세요.
* **1위 대 2위 비율**, $p_{\max} / p_{\text{second}}$. 선택된 선택지가 2위를 얼마나 뚜렷하게 앞서는지를 측정하고, 나머지가 어떻게 퍼져 있는지는 무시합니다. 실제 결정은 상위 두 후보 사이에서 갈리는 경우가 많은데, 이 비율은 바로 그 지점을 겨냥합니다.

### Score

Score의 단계에는 순서가 있으므로, Score 공식은 확률이 가장 유력한 단계에서 얼마나 멀리 놓여 있는지도 반영합니다. 단계가 $0$부터 $n - 1$까지 번호가 붙은 $n$개인 Score에서, $p_i$가 단계 $i$의 확률이고 $m$이 가장 유력한 단계일 때 공식은 다음과 같습니다.

$$
\text{confidence} = \max\left(0,\ 1 - \frac{\sum_i p_i \, |i - m|}{\text{MAD}_{\text{unif}}}\right)
\qquad
\text{MAD}_{\text{unif}} = \frac{1}{n} \sum_i \left| i - \frac{n - 1}{2} \right|
$$

분자의 합은 답과 가장 유력한 단계 사이의 거리를 단계 수 단위로 잰 뒤 확률로 가중 평균한 값입니다. $\text{MAD}_{\text{unif}}$는 모든 단계에 고르게 퍼진 분포에 대해, 가운데 단계를 기준으로 같은 방식의 평균 거리를 구한 값입니다. Confidence는 이 둘을 비교하며, 답이 균등 분포만큼 또는 그보다 더 퍼져 있으면 0으로 고정됩니다.

인접한 단계에 놓인 확률은 더 멀리 떨어진 단계에 놓인 같은 크기의 확률보다 confidence를 덜 낮춥니다. 단계가 세 개일 때 $(0, 0.5, 0.5)$의 confidence는 0.25입니다. 모델이 인접한 두 단계 사이에서 갈팡질팡하고 있기 때문입니다. $(0.5, 0, 0.5)$의 confidence는 0입니다. 양쪽 끝 사이에서 갈팡질팡하고 있기 때문입니다. Choice 공식이라면 두 분포 모두 0.25가 됩니다.

단계가 다섯 개인 Score 질문

단계 사이의 거리에 따라 confidence가 어떻게 달라지는지 확인해 보세요

Confidence

확률

슬라이더를 움직여 단계의 확률을 바꿔 보세요. 합계가 100%로 유지되도록 나머지 확률이 함께 조정됩니다.

뚜렷한 정점
인접 단계 사이에서 갈림
양 끝 사이에서 갈림
균등 분포

<dl aria-live="polite">
  <dt>Score</dt><dd />

  <dt>같은 확률에 Choice 공식을 적용한 값</dt><dd />
</dl>

<details markdown="1">
  <summary>이 데모가 Confidence를 계산하는 방법</summary>

  이 데모는 [Score 공식](#score)을 사용합니다. 먼저 답과 정점 사이의 거리를 단계 수 단위로 재서 확률로 가중 평균합니다. 이 값을 1.2로 나누는데, 1.2는 다섯 단계에 고르게 퍼진 분포를 가운데 단계 기준으로 쟀을 때의 같은 평균입니다. 그 결과를 1에서 빼되 0 아래로는 내려가지 않게 합니다. 정점 바로 옆 단계에 놓인 확률은 정점에서 먼 곳에 놓인 확률보다 confidence를 덜 낮추며, Choice 공식은 이 점을 반영하지 않습니다.
</details>

```python
def score_confidence(probabilities: list[float]) -> float:
    n = len(probabilities)
    m = probabilities.index(max(probabilities))
    spread = sum(p * abs(i - m) for i, p in enumerate(probabilities))
    even_spread = sum(abs(i - (n - 1) / 2) for i in range(n)) / n
    return max(0.0, 1 - spread / even_spread)


levels = sorted(answer.probabilities)
score_confidence([answer.probabilities[level] for level in levels])
```

[Score 응답 예시](primitives/score.md#response-structure)의 `bug_severity` 답은 확률이 $(0, 0.57, 0.43)$입니다. 가장 유력한 단계는 1이고, 퍼진 정도는 $0.43$이며, 단계가 세 개일 때 $\text{MAD}_{\text{unif}}$는 $\frac{2}{3}$이므로 confidence는 $1 - 0.43 / \frac{2}{3} \approx 0.35$입니다.


이 문서는 개발자 문서 플랫폼인 [Mintlify](https://mintlify.com)로 제작하고 호스팅합니다.
