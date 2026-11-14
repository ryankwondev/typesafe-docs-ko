# AI 입문

> TypeSafe가 생성된 텍스트를 최적화하는 대신 calibrated(보정된) 확률을 내는 결정 모델을 학습시키는 이유를 설명합니다.

대부분의 AI 제품은 모델과 사람 사이의 대화를 중심으로 만들어집니다. TypeSafe는 다른 가설에서 출발합니다. 대규모 자동화는 AI와 AI, AI와 소프트웨어 사이의 상호작용이 주를 이룰 것이므로, 채팅 인터페이스보다 기계 인터페이스가 더 중요하다는 것입니다.

> **TypeSafe는 이것을 Machine Native Intelligence(기계 네이티브 지능)라고 부릅니다:**
>
> 구조, 안정성, 관측 가능성, 테스트 가능성, 속도, 일관성, 낮은 비용처럼 소프트웨어와 같은 속성을 갖춘 AI입니다.

## 신(God)이 아니라 프로덕션(prod)을 만듭니다

TypeSafe는 모든 것을 해내는 모델을 만들려는 것이 아닙니다. 코드가 들여다보고 그에 따라 행동할 수 있는 좁은 범위의 결정이 필요한 프로덕션 시스템을 위해 설계되었습니다.

TypeSafe는 대규모 AI 자동화에서 기계 간 상호작용이 99%, 사람과의 상호작용이 1%에 가까울 것으로 예상합니다. 그러면 설계 목표가 읽기에 기분 좋은 응답에서 소프트웨어 안에서 예측 가능하게 동작하는 출력으로 옮겨 갑니다.

[TypeSafe 선언문](https://typesafe.ai/manifesto)을 읽어 보세요.

## 세 가지 post-training(사후 학습) 접근법

사전 학습된 언어 모델은 지금까지 크게 두 가지 방식으로 조정되어 왔습니다. TypeSafe는 여기에 세 번째 방식을 더합니다. RLHF와 RLVR은 맥락을 위해 함께 보여 드리는 것이며, TypeSafe의 학습 경로는 RLCD입니다.

> [card] RLHF
>
> **Reinforcement learning from human feedback**(사람의 피드백을 통한 강화 학습)은 사전 학습된 모델을 챗봇으로 바꿔 놓았습니다. 사람들이 선호하는 응답을 만들어 내도록 모델을 학습시킵니다.

> [card] RLVR
>
> **Reinforcement learning with verifiable rewards**(검증 가능한 보상을 이용한 강화 학습)는 수학 같은 작업에 강하지만 더 느리고 비용이 많이 드는 추론 모델을 만들어 냈습니다.

> [card] RLCD
>
> **Reinforcement learning for calibrated decisions**(보정된 결정을 위한 강화 학습)는 Jev가 생성된 텍스트 대신 결정과 calibrated 확률을 반환하도록 학습시킵니다.

RLHF는 InstructGPT와 ChatGPT를 학습시키는 데 쓰였으며, TypeSafe의 공동 창업자인 [Diogo Almeida가 공동으로 고안](https://scholar.google.com/citations?user=0T4y07QAAAAJ\&hl=en)했습니다.

![사전 학습된 언어 모델이 흐리게 표시된 RLHF 및 RLVR 경로와 강조된 RLCD 결정 모델 경로로 갈라지는 그림.](https://mintcdn.com/ts-docs/aFVnpmCIX68NpsV1/images/ai-primer/training-paths-light.webp?fit=max\&auto=format\&n=aFVnpmCIX68NpsV1\&q=85\&s=61898215ac31388d3be15bf583b743ee)

## RLCD와 calibrated 결정

RLCD는 다른 종류의 출력 계약을 목표로 최적화합니다.

* 모델은 텍스트를 생성하지 않습니다.
* 결정과 확률을 반환합니다.
* 확률이 높을수록 답이 정답일 가능성도 그만큼 커야 합니다.

calibration(보정)은 소프트웨어가 불확실성을 활용할 수 있게 해 줍니다. 잘 calibrated된 모델이 내놓은 많은 예측을 놓고 보면 다음과 같습니다.

* 확률 `0.2`가 부여된 결과는 약 20%의 비율로 일어나야 합니다.
* 확률 `0.8`이 부여된 결과는 약 80%의 비율로 일어나야 합니다.
* 확률 `1.0`이 부여된 결과는 100% 일어나야 합니다.

이 비율은 예측 집단에 대한 설명이지, 개별 답 하나에 대한 보장이 아닙니다. 소프트웨어가 언제 실행하고 언제 상위로 넘겨야 할지 정하는 방법은 [Confidence](../confidence.md)를 참고하세요.

## RLHF의 문제점

RLHF는 사람들이 선호하는 말을 하도록 모델을 가르칩니다. 이 목표는 챗봇에는 잘 맞지만, 아첨이나 자신만만하게 들리는 hallucination(환각)에 보상을 줄 수도 있습니다.

선호도 최적화는 **mode dropping**(모드 누락)도 일으킵니다. 모델이 지시 따르기 같은 특정 스타일을 선호하도록 학습하면서 다른 가능한 출력의 확률은 낮추는 현상입니다.

![기반 모델의 확률 분포를 RLHF 이후 좁아지고 mode dropping이 일어난 분포와 비교한 그림.](https://mintcdn.com/ts-docs/aFVnpmCIX68NpsV1/images/ai-primer/mode-dropping-light.webp?fit=max\&auto=format\&n=aFVnpmCIX68NpsV1\&q=85\&s=d51758a6212b526fc243cc9a81572cc7)

> [!WARNING]
>
> 사람에게는 설득력 있게 보이는 출력이라도 사람이 지켜보지 않는 자동화에 쓸 만큼 안정적이지는 않을 수 있습니다. 사람의 선호와 기계의 신뢰성은 서로 다른 최적화 목표입니다.

mode dropping은 **mode collapse**(모드 붕괴)의 약한 형태입니다. 생성적 적대 신경망의 고전적인 실패 유형인 mode collapse에서는 생성자가 같은 종류의 출력을 반복해서 만들어 내도록 학습합니다. 그 출력이 판별자를 계속 속이기 때문입니다.

> [accordion] Mode collapse 비유
>
> ![반복되는 글자들이 mode collapse를 겪는 GAN을 보여 주는 그림.](https://mintcdn.com/ts-docs/aFVnpmCIX68NpsV1/images/ai-primer/mode-collapse-light.webp?fit=max\&auto=format\&n=aFVnpmCIX68NpsV1\&q=85\&s=2896f125ad1a5835b31b088fbc64eff1)

RLHF는 대화형 모델에는 여전히 잘 맞습니다. TypeSafe의 입장은 프로덕션 자동화에는 다른 학습 목표, 즉 제약된 결정과 calibrated 불확실성을 중심에 둔 목표가 필요하다는 것입니다.


이 문서는 개발자 문서 플랫폼인 [Mintlify](https://mintlify.com)로 제작하고 호스팅합니다.
