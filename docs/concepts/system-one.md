# System One

> System One 모델은 소프트웨어를 위해 빠르고 구조화된 결정을 내립니다. Jev는 TypeSafe의 주력 모델이자 최초의 System One 모델입니다.

System One 모델은 소프트웨어가 바로 사용할 수 있는 빠르고 구조화된 결정을 내리도록 만든 AI 모델군입니다. System One 모델은 [state(상태)](state.md)를 평가해 타입이 지정된 답과 확률을 반환합니다.

Jev는 TypeSafe의 주력 모델이자 최초의 System One 모델입니다.

System One 모델은 LLM처럼 자연어 입력을 이해합니다. 다만 생성된 텍스트가 아니라 타입이 지정된 결정과 확률을 반환합니다. Jev는 [이미지](../images.md)도 이해합니다.

## LLM과 다른 점

System One 모델은 calibrated(보정된) 결정을 내리도록 학습됩니다. 즉, 확률이 불확실성을 제대로 반영하도록 실제 결과에 맞춰 최적화되어 있습니다. calibration(보정)은 예측을 묶은 집단 단위로 측정하므로, 개별 답 하나하나가 맞는다고 보장하지는 않습니다.

System One 모델은 답장을 쓰거나 코드를 만들거나 추론 과정을 설명하는 글을 생성하지 않습니다. 가능한 답은 여러분이 [primitive(프리미티브: 기본 구성 요소)](../primitives.md)로 정의합니다.

| Primitive | 질문 | 답 공간 예시 | 출력 예시 |
| - | - | - | - |
| [Choice](../primitives/choice.md) | 이 티켓을 어느 팀이 처리해야 하나요? | `billing`, `technical`, `account` 중 하나 | `choice: "billing"` |
| [Score](../primitives/score.md) | 이 고객은 얼마나 불만스러워하나요? | 0 = 차분함, 1 = 불만스러움, 2 = 매우 불만스러움 | `score: 1.4` |
| [Noul](../primitives/noul.md) | 이 메시지는 환불을 요청하나요? | 참 또는 거짓 | `noul: 0.95` |
| [Box](../primitives/box.md) | 이 차의 손상 부위는 어디인가요? | 네 변의 위치. 각각 이미지 너비나 높이에 대한 비율로, 0에서 1 사이 | `box: {top: 0.41, bottom: 0.68, left: 0.22, right: 0.57}` |

위 설정과 값은 설명을 위한 예시입니다. 사용할 수 있는 설정 옵션과 전체 응답 필드는 각 primitive 페이지에서 설명합니다.

System One 모델이 어떻게 동작하고 어떻게 학습되는지는 [AI 입문](../introduction/machine-learning-primer.md)을 읽어 보세요.

> [!NOTE]
>
> System One이라는 이름은 Daniel Kahneman이 저서 *Thinking, Fast and Slow*에서 널리 알린 개념에서 따왔습니다. System 1 사고는 빠르고 직관적입니다. System 2는 더 느리고 신중합니다. 여기서는 빠르고 초점이 분명한 판단에 무게를 둡니다.

## 더 큰 워크플로 안에서 내리는 빠른 판단

환불 요청을 예로 들면 애플리케이션은 다음과 같이 동작할 수 있습니다.

1. 고객의 메시지, 관련 거래 내역, 환불 정책을 담은 state를 만듭니다.
2. 서로 독립적인 질문을 한꺼번에 묻습니다. 환불을 요청했는지, 증거로 볼 때 중복 결제인지, 정책상 환불이 가능한지를 묻습니다.
3. 답을 코드의 결정론적 검사와 결합한 뒤, 해당 건을 조치 또는 검토 단계로 보냅니다.

primitive가 실제로 동작하는 모습을 확인했다면 이를 조합해 더 큰 시스템을 만들 수 있습니다. System One 모델은 자유 형식 텍스트가 아니라 타입이 지정되고 범위가 제한된 출력을 반환하므로, 코드에서 답을 검사하고 조합해 예측 가능한 워크플로를 구성할 수 있습니다. 전체 워크플로는 [TypeSafe로 구축하는 방법](how-to-build-with-system-one.md)을 참고하세요.

일부 질문 유형의 답에는 [confidence(신뢰도)](../confidence.md)도 함께 들어 있습니다(어떤 유형이 해당하는지는 각 유형의 페이지에 나와 있습니다). 이를 보고 언제 바로 조치하고 언제 사람이나 추론 모델에 넘길지 정할 수 있습니다.

## System One 모델 호출하기

System One 모델은 [클라이언트 SDK](../sdk.md)나 [HTTP API](https://docs.typesafe.ai/api)의 `POST /v1/systemone`으로 호출합니다. `model` 필드로 요청을 처리할 모델을 고릅니다. 이 문서의 예제는 `jev-latest`를 사용하며, 이는 SDK의 기본값이기도 합니다. 사용할 수 있는 모델과 가격, 별칭은 [Models](../models.md)를 참고하세요.

입력을 준비하려면 [State](state.md)부터, 물어볼 수 있는 질문 유형을 살펴보려면 [Primitives (Questions)](../primitives.md)부터 읽어 보세요.


이 문서는 개발자 문서 플랫폼인 [Mintlify](https://mintlify.com)로 제작하고 호스팅합니다.
