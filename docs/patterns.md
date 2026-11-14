# 패턴

> TypeSafe로 시스템을 구축하기 위한 아키텍처 패턴입니다.

TypeSafe는 더 큰 시스템 안에 자리 잡고 AI로 결정을 내려 주도록 설계되었습니다. TypeSafe를 최대한 활용하려면, 하나하나 분리된 원자적인 결정들이 조합되어 복잡한 시스템 동작을 이룬다는 관점으로 생각하는 법을 익히는 것이 중요합니다.

이 섹션은 [TypeSafe primitive(프리미티브: 기본 구성 요소)](primitives.md)를 알고 있고 [confidence(신뢰도)가 어떻게 동작하는지](confidence.md) 이해하고 있다고 가정합니다. 그렇지 않다면 두 문서를 먼저 읽으세요.

## 패턴 목록

| 패턴 | 하는 일 | 이점 |
| - | - | - |
| [추측성 fan-out(팬아웃)](patterns/fan-out.md) | 필요할지 확실하지 않은 질문까지 포함해 많은 질문을 한 요청에 담아 보내고, 무엇이 관련 있는지는 코드가 판단하게 합니다 | 비용, 속도 |
| [Noul로 먼저 선별하기](patterns/noul-screen.md) | 비용이 적게 드는 예/아니요 질문 하나를 먼저 묻고, 통과했을 때만 비용이 큰 단계를 실행합니다 | 비용, 속도 |
| [동반 Noul](patterns/companion-noul.md) | Choice 질문에 맞는 선택지가 하나라도 있는지 묻는 Noul 질문을 짝지어, 맞는 것이 없을 때 코드가 알아챌 수 있게 합니다 | 안정성 |
| [Confidence 기반 routing(라우팅)](patterns/confidence-routing.md) | confidence가 낮은 Score 답은 사람에게 보내고, 나머지는 score에 따라 routing합니다 | 안정성, 안전성 |
| [확률 기반 routing](patterns/probability-routing.md) | 위험한 동작일수록 더 높은 확률 threshold(임계값)를 설정합니다 | 안정성, 안전성 |
| [복합 점수 산정](patterns/composite-scoring.md) | 여러 차원의 분석을 하나의 점수로 결합합니다 | 비용, 안정성, 속도 |

## 목표별 패턴

모든 패턴은 하나의 결정을 작은 System One 질문들로 나누고, 결정 자체는 코드에 맡깁니다. Jev는 각 질문에 확률로 답하고, 그다음에 무슨 일이 일어날지는 코드가 정합니다. 필요한 것에 맞춰 고르세요.

* **비용을 줄이고 싶을 때.** [추측성 fan-out](patterns/fan-out.md)은 모든 것을 한 요청으로 묻습니다. [Noul로 먼저 선별하기](patterns/noul-screen.md)는 필요 없는 입력에 비용이 큰 단계가 실행되지 않도록 막습니다.
* **안전하게 실행하고 싶을 때.** [확률 기반 routing](patterns/probability-routing.md)은 위험한 동작일수록 더 높은 threshold를 설정합니다. [Confidence 기반 routing](patterns/confidence-routing.md)은 척도 전반에 퍼진 Score 답을 사람에게 보냅니다. [동반 Noul](patterns/companion-noul.md)은 맞는 것이 하나도 없는 경우를 잡아냅니다.
* **답을 결합하고 싶을 때.** [복합 점수 산정](patterns/composite-scoring.md)은 직접 정한 가중치로 여러 Score를 합산합니다.

> [!TIP]
>
> TypeSafe는 사람들이 primitive를 어떻게 활용하고 있는지 늘 궁금합니다. 여기에 실릴 만한 뛰어난 사용 사례를 찾았다면 언제든지 알려 주세요!


이 문서는 개발자 문서 플랫폼인 [Mintlify](https://mintlify.com)로 제작하고 호스팅합니다.
