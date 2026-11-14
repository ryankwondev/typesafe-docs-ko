# Advanced: structure

> instructions, Choice 선택지, Score 단계, Noul criteria에는 모두 JSON 구조를 쓸 수 있습니다.

System One 모델은 구조를 이해하도록 학습되어 있습니다.

## 구조를 쓸 수 있는 위치

아래 필드는 모두 [`EntryType`](../sdk/javascript/api/type-aliases/EntryType.md)입니다.

| 필드 | 적용 대상 | 허용되는 형태 |
| - | - | - |
| `instructions` | Choice, Score, Noul, Box | `string`, `object`, `array`, `null` 중 하나 |
| `criteria` 값 (선택지 설명) | Choice | `string`, `object`, `array`, `null` 중 하나 |
| `criteria` 항목 (단계 설명) | Score | `string`, `object`, `array`, `null` 중 하나 |
| `criteria.true`와 `criteria.false` | Noul | `string`, `object`, `array`, `null` 중 하나 |

## 질문을 구조화해야 할 때

* **명확성에 도움이 될 때.** 질문이 여러 부분으로 이루어져 있다면 JSON 형태로 담는 편이 더 명확합니다. 키마다 이름이 붙기 때문입니다.
* **질문에 보조 데이터가 필요할 때.** 스키마, taxonomy(분류 체계), 데이터베이스 행은 이미 JSON입니다. 문자열 템플릿에 직렬화해 넣지 말고 JSON을 통째로 쓰거나 관련된 하위 필드만 넘기세요.

## 구조화된 instructions

`field` 객체 하나가 검사할 필드를 설명하고, 각 질문은 키로 이 객체를 참조합니다. 같은 형태로 값을 검증하는 Noul, 후보 중 하나를 고르는 Choice, 값을 척도 위에 놓는 Score 두 개를 모두 구성할 수 있습니다.

Playground에서 실행해 보기 →

코드에서는 후보 레코드를 순회하면서 필드마다 이런 질문을 하나씩 만들어, 모두 요청 하나에 담아 보낼 수 있습니다. [SDE cascade cookbook(예제 모음)](../cookbooks/sde_cascade.md)이 이와 비슷한 방식을 씁니다.

배열도 쓸 수 있습니다. instructions(지시문)가 확인하거나 비교할 대상의 목록일 때 배열을 쓰세요.

```json
"instructions": {
  "question": "Does the claimed sender identity conflict with the sending domain?",
  "compare": ["ticket.sender.display_name", "ticket.sender.email"],
  "focus": "Compare the named organization with the email domain."
}
```

## 구조화된 Choice 선택지

Choice 선택지의 설명도 구조화된 객체가 될 수 있습니다.

### 경계를 분명히 하는 JSON 루브릭

Playground에서 실행해 보기 →

이 예제는 각 선택지가 무엇을 다루고 무엇을 다루지 *않는지* 모델에 알려 줍니다. 선택지 사이의 경계가 더 뚜렷해집니다.

### Taxonomy 탐색하기

깊은 taxonomy로 분류하려면 단계마다 Choice 질문을 하나씩 던지면서 코드에서 트리를 따라 내려가세요. 각 단계에서 선택지는 현재 노드의 자식들이고, 각 선택지의 값은 그 자식의 하위 트리입니다. 이렇게 하면 모델이 어느 가지를 고르기 전에 그 아래에 무엇이 있는지 볼 수 있습니다. 항목이 속하는 리프의 이름을 가지 이름만으로는 짐작하기 어려울 때 특히 중요합니다.

여기서 state(상태)는 상품 목록이고, 첫 번째 질문은 최상위 부문을 고릅니다.

Playground에서 실행해 보기 →

이 병은 두 부문 어디에 넣어도 그럴듯합니다. 하위 트리를 보여 주면 모델은 `Sporting Goods > Cycling > Bike Bottles & Cages`와 `Home & Kitchen > Drinkware > Water Bottles`가 모두 존재한다는 것을 알고, 상품 목록이 자전거 케이지를 강조한다는 점과 일상용 음료 용기라는 점을 견주어 볼 수 있습니다. 이 답의 `probabilities`를 보면 두 가지를 모두 탐색해야 할 만큼 확률이 팽팽하게 갈리는지 알 수 있습니다.

부문이 정해지면 그 부문의 자식들을 선택지로, 각 자식의 하위 트리를 값으로 삼아 다음 Choice 질문을 던지고, 리프에 닿을 때까지 반복합니다. 코드에서는 중첩된 dict를 순회하는 루프로 구현할 수 있으며, 이때 각 질문의 `criteria`는 그냥 현재 노드입니다. [Hierarchical Classification cookbook](../cookbooks/hierarchical_classification.md)은 이와 비슷하게 트리를 따라 내려가는 예제를 보여 주며, 확률이 비슷할 때 여러 후보 경로를 함께 유지하는 beam search(빔 서치)도 다룹니다.

> [!NOTE]
>
> 하위 트리는 커질 수 있습니다. 가지가 너무 크면 값을 직계 자식과 리프 일부 표본으로 줄이세요.

## 구조화된 Score 단계

Score `criteria` 배열의 각 항목은 객체가 될 수 있습니다.

Playground에서 실행해 보기 →

## 구조화된 Noul criteria

Noul의 `criteria`는 선택 사항입니다. 예와 아니요의 경계가 미묘할 때는 구조화된 `true`와 `false` 설명을 써서 양쪽에 정의와 예시를 두고 경계를 분명하게 못박을 수 있습니다.

Playground에서 실행해 보기 →


이 문서는 개발자 문서 플랫폼인 [Mintlify](https://mintlify.com)로 제작하고 호스팅합니다.
