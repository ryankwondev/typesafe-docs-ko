# 모델

Jev는 TypeSafe의 대표 모델이자 최초의 [System One 모델](concepts/system-one.md)입니다. 이 페이지의 모든 모델은 같은 엔드포인트인 `POST /v1/systemone`으로 제공됩니다. 어떤 모델이 요청을 처리할지는 요청의 `model` 필드로 고릅니다. 요청의 전체 형태는 [API 레퍼런스](https://docs.typesafe.ai/api)를 참고하세요.

## 현재 모델

| | Jev 1.14 | Jev 1.13 |
| :- | :- | :- |
| 모델 ID | `jev-1.14.0` | `jev-1.13.0` |
| 가격 (Btok당 / Mtok당) | \$42 / \$0.042 | \$42 / \$0.042 |
| 컨텍스트 길이 | 요청당 64k / 질문당 32k | 요청당 64k / 질문당 32k |
| 입력 | 텍스트, 이미지 | 텍스트, 이미지 |
| 질문 유형 | Choice, Score, Noul, Box | Choice, Score, Noul |

Jev 1.14는 가장 최신 Jev 릴리스이며 2026년 10월 14일부터 사용할 수 있습니다.

* **가격:** 입력 token(토큰) 수에 따라 비용을 냅니다. 출력 token은 무료입니다. Btok은 token 10억 개, Mtok은 token 100만 개입니다.
* **컨텍스트 길이:** 요청당 컨텍스트와 질문당 컨텍스트라는 두 가지 한도가 있습니다. [컨텍스트 길이](#context-length)를 참고하세요. 둘 다 늘리려면 [Long context](#long-context)를 참고하세요.
* **입력:** Jev는 문자열, JSON 객체, 또는 텍스트 값의 배열로 보낸 자연어 텍스트를 읽습니다. 요청 어디에 있든 PNG와 JPEG [이미지](images.md)도 읽습니다. 오디오와 비디오는 (아직) 기본으로 지원하지 않습니다. `state`(상태)가 받는 형태는 [State](concepts/state.md)를 참고하세요.
* **질문 유형:** [Box](primitives/box.md) 질문에는 Jev 1.14가 필요합니다.

## 컨텍스트 길이

Jev는 `state`를 한 번 읽어 들인 뒤 모든 질문을 그 state에 비추어 병렬로 평가합니다. 요청은 두 가지 컨텍스트 한도에 들어맞아야 합니다.

* **요청당 컨텍스트** (`per_request_context`): `state`와 모든 질문을 합친 것입니다.
* **질문당 컨텍스트** (`per_question_context`): `state`와 질문 하나를 합친 것입니다.

한 요청의 질문 수에는 별도의 제한이 없습니다. 요청이 두 한도에 모두 들어맞기만 하면 질문은 몇 개든 넣을 수 있습니다. 많은 질문을 한 요청에 담는 방법은 [추측성 fan-out(팬아웃)](patterns/fan-out.md)을, state가 커질수록 정확도가 어떻게 달라지는지는 [Model jaggedness(모델의 들쭉날쭉함)](model-jaggedness.md)를 참고하세요.

두 한도를 모두 늘리려면 [Long context](#long-context)를 참고하세요.

### Long context

Long context(긴 컨텍스트)는 기본적으로 꺼져 있으며 Jev 1.14와 Jev 1.13 모두에서 동작합니다. 켜려면 필요한 요청마다 `TypeSafe-Danger-Zone-Long-Context: true` 헤더를 보내세요. 이 헤더가 있으면 요청당 컨텍스트와 질문당 컨텍스트가 모두 262k token이 됩니다. 헤더가 없는 요청에는 위 표의 기본 한도가 그대로 적용됩니다.

```bash
curl https://api.typesafe.ai/v1/systemone \
  -H "Authorization: Bearer $TYPESAFE_API_KEY" \
  -H "Content-Type: application/json" \
  -H "TypeSafe-Danger-Zone-Long-Context: true" \
  -d '{
    "model": "jev-latest",
    "state": "<a long contract, transcript, or record>",
    "questions": {
      "has_termination_clause": {
        "type": "noul",
        "instructions": "Does the `state` include a clause that lets either party end the agreement early?"
      }
    }
  }'
```

Long context의 token당 비용은 다른 요청과 같습니다. 다만 기본 한도 안에 머무를 때보다 느리고 정확도가 낮으므로 필요한 요청에만 켜세요.

## Rate limit

rate limit(요청 한도)은 조직 단위로 설정됩니다. 조직의 현재 한도는 [개발자 콘솔](https://console.typesafe.ai/rate-limits)에서 확인할 수 있습니다.

한도는 초당 token 수와 초당 요청 수로 측정합니다. 둘 중 하나라도 넘은 요청은 `429 Too Many Requests`를 반환합니다. TypeSafe의 [클라이언트 SDK](sdk.md)는 기본적으로 이런 요청을 retry(재시도)하며, 시도 사이의 대기 시간을 점점 늘리고, 응답에 `retry-after` 헤더가 있으면 그 값을 따릅니다. HTTP API를 직접 호출한다면 [rate limit 처리하기](api/errors.md#handling-rate-limits)를 참고하세요.

커스텀 플랜과 엔터프라이즈 플랜에서는 더 높은 한도를 이용할 수 있습니다. 한도 상향을 요청하려면 [rate limit 상향 요청 양식](https://support.typesafe.ai/forms/rate-limit-increase-request)을 작성하세요.

## Alias

alias(별칭)는 버전이 지정된 모델 ID를 가리키는 모델 이름입니다. 모델 ID를 보낼 때와 똑같이 `model` 필드에 넣어 보내면 됩니다.

| Alias | 가리키는 대상 | 의미 |
| :- | :- | :- |
| `jev-latest` | `jev-1.14.0` | 가장 최근의 안정적인 공식 릴리스입니다. TypeSafe 클라이언트 SDK의 기본값이며, 이 문서의 예제에서 쓰는 이름입니다. |
| `jev-preview` | `jev-1.14.0` | 공식 여부와 관계없이 가장 최근의 릴리스입니다. 프리뷰 빌드가 나오면 `jev-latest`보다 앞서 나갑니다. |

> [!WARNING]
>
> `jev-preview`는 현재 `jev-latest`와 같은 모델을 가리킵니다. 지금은 이용할 수 있는 프리뷰 빌드가 없습니다.

alias는 새 릴리스가 출시되면 옮겨 가므로, 코드를 전혀 바꾸지 않아도 답이 달라질 수 있습니다. 각 응답의 `model` 필드에는 실제로 답한 버전 지정 ID가 들어 있으므로 어떤 모델이 각 결과를 만들었는지 로그로 남길 수 있습니다. 특정 버전에 맞춰 confidence(신뢰도) threshold(임계값)를 조정해 두었다면 alias 대신 그 버전의 ID를 보내고, 준비가 되었을 때 새 버전으로 전환하세요.

## Jev 커스터마이징

Jev는 고객 데이터로 fine-tuning(미세 조정)하거나 LoRA로 적응시키지 않습니다. calibrated(보정된) 결정을 반환하도록 [RLCD](introduction/machine-learning-primer.md)로 학습되었으며, 모든 계정이 같은 가중치를 사용합니다. 계정별 가중치가 아니라 요청을 통해 Jev의 답을 도메인에 맞게 조정합니다.

* 자체 보유한 콘텐츠, 레코드, 참고 자료는 `state` 필드에 넣으세요. [State](concepts/state.md)를 참고하세요.
* 도메인 규칙과 경계 사례는 각 질문의 `instructions`(지시문)와 `criteria`(판단 기준)에 담으세요. [TypeSafe로 구축하는 방법](concepts/how-to-build-with-system-one.md)과 [고급: 구조](primitives/advanced.md)를 참고하세요.
* 범위가 넓은 판단은 원자적인 질문으로 쪼개고 출력을 코드에서 결합하세요. [복합 점수 산정](patterns/composite-scoring.md)을 참고하고, Jev의 확률로 후속 단계의 고전적 모델을 학습시키는 방법은 [AutoResearch cookbook(예제 모음)](cookbooks/autoresearch_feature_discovery.md)을 참고하세요.

## 데이터 처리

Jev는 고객의 요청이나 응답으로 학습되지 않습니다. 데이터 처리 계약(Data Processing Agreement), 개인정보 처리방침(Privacy Policy), 엔터프라이즈 고객을 위한 zero data retention(ZDR, 데이터 무보존)의 세부 사항은 [법적 고지](legal.md)를 참고하세요.

## 모델 목록 조회

`GET /v1/models`는 계정이 `model` 필드에 보낼 수 있는 이름들을 설명, 출시일과 함께 반환합니다. 현재는 alias를 나열합니다. `jev-1.14.0` 같은 버전 지정 ID는 목록에 나오든 나오지 않든 `model` 필드에서 받아들여집니다.

```bash title="cURL"
curl https://api.typesafe.ai/v1/models \
  -H "Authorization: Bearer $TYPESAFE_API_KEY"
```

```python title="Python"
from typesafe_sdk import TypeSafeClient

with TypeSafeClient() as client:
    for model in client.models.list().models:
        print(model.name, model.release_date, model.description)
```

```typescript title="JavaScript"
import { TypeSafeClient } from "@typesafe-ai/sdk";

const client = new TypeSafeClient();
const models = await client.models.list();
for (const model of models) {
  console.log(model.name, model.release_date, model.description);
}
```

**models** `array` *필수*

모델 또는 alias마다 항목 하나가 들어 있습니다.

**properties(속성)**

**name** `string` *필수*

`model` 필드가 받는 모델 ID 또는 alias입니다.

**description** `string` *필수*

모델의 용도입니다.

**release\_date** `string` *필수*

모델 또는 alias가 출시된 시점입니다.

전체 메서드 시그니처는 [Python](sdk/python/api/clients/sync.md#typesafe_sdk.Models.list) 및 [JavaScript](sdk/javascript/api/interfaces/Models.md) SDK 레퍼런스를 참고하세요.


이 문서는 개발자 문서 플랫폼인 [Mintlify](https://mintlify.com)로 제작하고 호스팅합니다.
