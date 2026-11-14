# 스마트 홈 어시스턴트 데모

> 데모 코드: Jev로 사용자 요청을 평가하는 스마트 홈 어시스턴트입니다.

## 실제 동작 보기

[스마트 홈 어시스턴트 데모 영상](https://player.vimeo.com/video/1230923853)

## 동작 방식

### Speculative fan-out

이 데모가 보여 주는 핵심 패턴은 [speculative fan-out(추측성 팬아웃)](../patterns/fan-out.md)입니다. 사용자 요청마다 긴 질문 목록으로 평가하는데, 그중 상당수는 대부분의 요청에서 결국 관련이 없는 질문입니다.

"Turn off all of the lights in the house"를 수행하는 데 코드에 필요한 답은 네 개뿐입니다.

* "What category of request is this?" (smarthome command)
* "What domain is this request targeting?" (whole house)
* "What type of device is this request targeting?" (lights)
* "What action should be taken on the lights?" (turn off)

마지막 질문은 요청이 조명에 내리는 명령이라고 가정하는데, 그 사실이 확인되기도 전에 Jev에게 묻습니다. 이런 질문이 speculative 질문입니다. 일찍 물어 두면 모든 질문을 병렬로 Jev에 보낼 수 있고, 해당하지 않는 답은 코드에서 무시합니다. 그러면 질문 한 세트로 여러 종류의 요청을 처리할 수 있습니다.

#### 잘못된 방법: 순차적인 API 호출

더 느린 방법은 질문을 여러 API 호출로 나누고, 앞선 답에서 필요하다고 확인된 뒤에야 다음 질문을 하는 것입니다.

* "What category of request is this?" (smarthome command)

그런 다음, 스마트 홈 명령임을 알고 난 뒤에야 다음을 묻습니다.

* "What domain is this request targeting?" (whole house)
* "What type of device is this request targeting?" (lights)

그런 다음, 대상이 조명임을 알고 난 뒤에야 다음을 묻습니다.

* "What action should be taken on the lights?" (turn off)

이 방식은 질문 수를 최소로 줄이는 데 맞춰져 있지만, 모든 질문을 처음에 한 번의 API 호출로 묶어 보내는 것보다 결국 훨씬 느리고 비용도 더 듭니다.

### Jev와 LLM 함께 쓰기

어시스턴트는 두 경우에 LLM을 호출하며, 둘 다 생성된 텍스트가 필요한 경우입니다.

**복합적인 사용자 요청 나누기:** 이 데모의 질문 중에는 사용자 요청이 서로 다른 동작을 둘 이상 요구하는지 판별하는 Noul 질문이 있습니다. 답이 참이면 시스템은 LLM으로 요청을 더 쪼갤 수 없는 명령들의 목록으로 나눕니다. 그렇게 나눈 요청은 Jev가 하나씩 평가합니다.

**대화형 LLM으로 넘기기:** 사용자 질의가 일반적인 정보나 대화를 구하는 요청이라고 Jev가 판단하면 시스템은 LLM을 호출해 자유 형식의 응답을 생성합니다. 이렇게 하면 대화형 시스템에서 동작이 정해져 있는 요청은 빠르고 비용 효율적으로 처리하면서, 필요할 때는 생성형 LLM의 유연함도 그대로 활용할 수 있습니다. Jev의 첫 응답은 LLM 응답에 비해 워낙 빨라서 전체 시스템에 더해지는 지연 시간은 무시할 만한 수준입니다.

## 코드

이 데모는 TypeSafe API로 사용자 요청을 평가하는 간단한 Vite/React 단일 페이지 앱입니다. 코드는 오픈소스로 공개할 예정이며, 공개되면 여기에 링크를 걸겠습니다.


이 문서는 개발자 문서 플랫폼인 [Mintlify](https://mintlify.com)로 제작하고 호스팅합니다.
