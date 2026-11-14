# 코딩 에이전트와 Jev

> 코딩 에이전트를 쓸 때 Jev가 무엇이고 무엇이 아닌지 설명합니다.

코딩 에이전트에 연결할 모델을 찾다가 TypeSafe를 알게 되었다면 여기서 시작하세요. Jev는 Claude Code, Cursor, opencode, Copilot, Muse Spark, Grok Bot 같은 도구 뒤에 있는 LLM을 그대로 갈아 끼울 수 있는 대체품이 **아닙니다**. 대신 코딩 에이전트는 평소처럼 쓰면서, Jev로 결정을 내리는 코드를 그 에이전트로 작성할 수 있습니다.

## Jev는 채팅이나 코드 자동 완성용 LLM이 아닙니다

Jev는 [System One 모델](../concepts/system-one.md)입니다. 텍스트를 생성하지도, 코드를 작성하지도, 대화를 이어 가지도 않습니다. [state(상태)](../concepts/state.md)와 타입이 지정된 [질문](../primitives.md) 집합을 받아, 코드에서 바로 쓸 수 있는 구조화된 답을 반환합니다.

* 선택지 목록에서 고른 `choice`와 선택지별 확률.
* 직접 정의한 rubric(루브릭: 채점 기준)에 따른 `score`.
* 참/거짓 진술에 대한 `noul` (0–1).

코딩 에이전트는 자연어 지시에 따라 텍스트를 스트리밍하고, 도구를 호출하고, 파일을 편집하는 LLM에 의존합니다. Jev는 그중 어느 것도 하지 않습니다. 코딩 에이전트를 Jev 기반 에이전트로 바꿔 주는 `model: "jev-latest"` 같은 설정은 없습니다. 두 시스템이 서로 다른 문제를 풀기 때문입니다.

## 아마도 원하셨을 것

하려던 일과 맞는 행을 고르세요.

| 하고 싶었던 일 | 이렇게 하세요 |
| - | - |
| 코딩 에이전트가 *TypeSafe를 사용하는 코드를 작성하는* 일을 더 잘하게 만들기 | [TypeSafe agent skill](../agent-skill.md)을 설치하세요. Claude Code, Codex를 비롯한 에이전트에게 TypeSafe API, [primitive(프리미티브: 기본 구성 요소)](../primitives.md), [패턴](../patterns.md)에 관한 전체 맥락을 제공해, 에이전트가 올바른 TypeSafe 연동 코드를 생성할 수 있게 합니다. |
| 만들고 있는 앱이나 에이전트 안에서 routing(라우팅), 분류, 점수 매기기, guardrail(가드레일)을 비롯한 구조화된 결정에 Jev 사용하기 | [빠른 시작](quickstart.md)으로 시작한 다음 [TypeSafe로 구축하는 방법](../concepts/how-to-build-with-system-one.md)을 읽고, [확률 기반 routing](../patterns/probability-routing.md)이나 [Noul로 먼저 선별하기](../patterns/noul-screen.md) 같은 일반적인 아키텍처는 [패턴](../patterns.md)에서 확인하세요. |
| 코딩 에이전트를 구동하는 모델을 교체하거나 바꿔 끼우기 | Jev는 이 용도에 맞는 도구가 아닙니다. LLM 기반 코딩 에이전트는 계속 쓰고, 제품에서 빠르고 calibrated(보정된) 구조화된 결정이 필요한 곳마다 Jev를 따로 사용하세요. |
| 코드를 작성하기 전에 Jev를 먼저 써 보기 | [Playground](https://console.typesafe.ai/playground)를 열고, 텍스트를 state로 붙여 넣고, 질문을 몇 개 추가하세요. 단계별 안내는 [빠른 시작](quickstart.md)을 참고하세요. |

## Jev를 쓸 만한 경우

Jev는 코딩 에이전트용 LLM이 아니지만, 코딩 에이전트로 만들고 있는 에이전트나 앱 *안에서는* 딱 맞는 도구인 경우가 많습니다. 코드에서 다음과 같은 일이 필요할 때 Jev를 사용하세요.

* 요청을 정해진 목적지 중 하나로 routing하고, 그 routing을 얼마나 확신할 수 있는지 알아야 할 때.
* rubric(긴급도, 품질, 위험)에 따라 무언가에 점수를 매기고 그 숫자로 분기해야 할 때.
* 동작을 실행하기 전에 어떤 진술이 문서, 메시지, 레코드에 대해 참인지 확인해야 할 때.
* LLM에게 "return JSON"(JSON을 반환해)이라고 요청하는 취약한 prompt(프롬프트)를, 구조상 타입이 지정된 값을 반환할 수밖에 없는 호출로 대체해야 할 때.

이 중 하나라도 지금 만들고 있는 것과 맞는다면, 가장 빠른 길은 [빠른 시작](quickstart.md)을 본 다음 질문 유형을 다루는 [primitive](../primitives.md) 레퍼런스를 보는 것입니다.

## 다음 단계

* [System One](../concepts/system-one.md): System One 모델이 무엇이고 LLM과 어떻게 다른지 설명합니다.
* [빠른 시작](quickstart.md): Playground에서, HTTP로, 또는 Python SDK로 Jev를 사용해 봅니다.
* [Agent skill](../agent-skill.md): 코딩 에이전트에게 TypeSafe API에 관한 맥락을 제공합니다.
* [패턴](../patterns.md): TypeSafe로 구축할 때 자주 쓰는 아키텍처를 소개합니다.


이 문서는 개발자 문서 플랫폼인 [Mintlify](https://mintlify.com)로 제작하고 호스팅합니다.
