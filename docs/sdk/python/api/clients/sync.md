# 동기 클라이언트

> TypeSafeClient로 질문을 보내고, 모델 목록을 조회하고, 동기 방식의 TypeSafe API 요청을 설정합니다.

## typesafe\_sdk.TypeSafeClient

시그니처 복사
<pre tabIndex="0" aria-label="SDK signature">`TypeSafeClient(
&#x20;   \*,
&#x20;   api\_key: [str](https://docs.python.org/3/builtins/stdtypes.html#str) | None = None,
&#x20;   model: [str](https://docs.python.org/3/builtins/stdtypes.html#str) | None = None,
&#x20;   retry: [RetryPolicy](../retries.md#typesafe_sdk.RetryPolicy) | None = None,
&#x20;   timeout: [float](https://docs.python.org/3/builtins/functions.html#float)
&#x20;   | httpx2.Timeout
&#x20;   | None = None,
&#x20;   headers: [Mapping](https://docs.python.org/3/library/collections.abc.html#collections.abc.Mapping)\[[str](https://docs.python.org/3/builtins/stdtypes.html#str), [str](https://docs.python.org/3/builtins/stdtypes.html#str)] | None = None,
&#x20;   transport: httpx2.BaseTransport
&#x20;   | None = None,
&#x20;   http\_client: httpx2.[Client](https://pydantic.dev/docs/httpx2/api/api/#httpx2.Client) | None = None,
&#x20;   base\_url: [str](https://docs.python.org/3/builtins/stdtypes.html#str) | None = None,
&#x20;   compression: [Literal](https://docs.python.org/3/library/typing.html#typing.Literal)\["zstd", False]
&#x20;   | None = None,
)`</pre>

[TypeSafe AI API](https://typesafe.ai)용 HTTP 클라이언트를 생성합니다.

명시적으로 전달한 옵션이 환경 변수보다 우선합니다. 환경 변수 값이 비어 있거나 공백뿐이면 무시합니다.

> [!TIP]
>
> **로깅 설정**
>
> SDK는 `typesafe_sdk` 로거에 로그를 남깁니다. 표준 logging으로 설정하거나, 간단히 기본값만 정하려면 `TYPESAFE_LOG_LEVEL`(`debug`, `info`, ...)을 설정하세요. 비밀 값이 담긴 헤더는 로그 출력에서 가려지지만, 요청과 응답 본문은 가려지지 않습니다.

Parameters(매개변수):

* **`api_key`** (`str | None`, default: `None` ) –

  필수 API 키입니다. 환경 변수 `TYPESAFE_API_KEY`로도 설정할 수 있습니다. 앞뒤 공백은 제거됩니다. 빈 키, 중간에 공백이 있는 키, 제어 문자나 ASCII가 아닌 문자가 들어 있는 키는 거부됩니다.
* **`model`** (`str | None`, default: `None` ) –

  모델 이름입니다. 환경 변수 `TYPESAFE_DEFAULT_MODEL`로도 설정할 수 있습니다.
* **`retry`** (`RetryPolicy | None`, default: `None` ) –

  retry(재시도) 동작을 제어하는 `RetryPolicy`입니다. 쓸 수 있는 옵션과 기본값은 `RetryPolicy`를 참고하세요. retry를 끄려면 `RetryPolicy(max_retries=0)`을 전달합니다.
* **`timeout`** (`float | httpx2.Timeout | None`, default: `None` ) –

  HTTP 작업의 타임아웃입니다. `http_client`를 전달했다면 `http_client.timeout`을 물려받고, 그렇지 않으면 SDK 기본값을 씁니다.
* **`headers`** (`Mapping[str, str] | None`, default: `None` ) –

  추가로 설정할 요청 헤더입니다.
* **`transport`** (`httpx2.BaseTransport | None`, default: `None` ) –

  선택적으로 지정하는 사용자 정의 HTTP transport입니다. 이 SDK 클라이언트가 닫힐 때 함께 닫힙니다.
* **`http_client`** (`httpx2.Client | None`, default: `None` ) –

  선택적으로 지정하는 `httpx2.Client`입니다. `transport`와 함께 쓸 수 없습니다. 이 SDK 클라이언트가 닫힐 때 함께 닫힙니다.
* **`base_url`** (`str | None`, default: `None` ) –

  API 루트입니다. 환경 변수 `TYPESAFE_BASE_URL`로도 설정할 수 있습니다.
* **`compression`** (`Literal['zstd', False] | None`, default: `None` ) –

  요청 본문 압축 방식입니다. `"zstd"`를 지정하거나, 본문을 압축하지 않고 보내려면 `False`를 지정합니다. [사용할 수 있는 경우](../../usage/advanced.md#ensure-zstd-is-available) 기본값은 zstd입니다.

Raises(예외):

* `TypeSafeError` –

  API 키가 없거나 유효하지 않은 경우, 타임아웃이 유효하지 않은 경우, zstd 압축을 쓸 수 없는 경우에 발생합니다.
* `ValueError` –

  `transport`와 `http_client`를 둘 다 전달한 경우에 발생합니다.

Examples(예제):

```python
from typesafe_sdk import Choice, Noul, TypeSafeClient

with TypeSafeClient() as client:
    result = client.system_one(
        state="I was charged twice. Please help.",
        questions={
            "billing": Noul(instructions="Is this about billing?"),
            "tone": Choice(
                instructions="What is the tone?",
                criteria={"calm": None, "angry": None},
            ),
        },
    )
    assert 0 <= result.nouls["billing"].noul <= 1
    assert result.choices["tone"].choice in {"calm", "angry"}
```

### models

`cached` `property`

시그니처 복사

<pre tabIndex="0" aria-label="SDK signature">
  \`models

  :

  &#x20;

  [Models](sync.md#typesafe_sdk.Models)\`
</pre>

Models API 리소스에 접근하는 접근자입니다.

Examples:

```python
with TypeSafeClient() as client:
    models = client.models.list()
```

### attach\_assets

시그니처 복사
<pre tabIndex="0" aria-label="SDK signature">`attach\_assets(
&#x20;   assets: [Asset](../types/assets.md#typesafe_sdk.Asset) | [Sequence](https://docs.python.org/3/library/collections.abc.html#collections.abc.Sequence)\[[Asset](../types/assets.md#typesafe_sdk.Asset)],
) -> [AbstractContextManager](https://docs.python.org/3/library/contextlib.html#contextlib.AbstractContextManager)\[None]`</pre>

`with` 블록 안에서 일어나는 호출에 asset(첨부 자산)을 첨부해, 문자열 안에서 asset을 참조할 수 있게 합니다.

블록을 벗어나기 전까지 이 클라이언트로 보내는 호출은, 첨부된 asset 가운데 그 참조인 `str(asset)`이 요청에 들어 있는 것을 모두 함께 전송합니다.

> [!WARNING]
>
> **경고**
>
> SDK는 첨부된 asset, `assets`로 전달한 asset, 내용 안에 직접 넣은 asset의 참조만 인식합니다. 그 밖의 asset을 가리키는 참조는 asset 없이 일반 텍스트로 전송됩니다.

Parameters:

* **`assets`** (`Asset | Sequence[Asset]`) –

  asset 하나 또는 asset의 시퀀스입니다.

Raises:

* `TypeSafeError` –

  첨부한 값이 asset이 아닌 경우에 발생합니다.

Examples:

```python
from pathlib import Path

from typesafe_sdk import Image, Noul, SystemOneResponse, TypeSafeClient


def inspect(client: TypeSafeClient, path: Path) -> SystemOneResponse:
    photo = Image(path)
    with client.attach_assets(photo):
        return client.system_one(
            state=f"Product: {photo}",
            questions={"damaged": Noul(instructions="Is the product damaged?")},
        )
```

### system\_one

> [tab] 구현
>
> 시그니처 복사
> <pre tabIndex="0" aria-label="SDK signature">`system\_one(
> &#x20;   state: [Content](../types/common.md#typesafe_sdk.Content),
> &#x20;   questions: [Mapping](https://docs.python.org/3/library/collections.abc.html#collections.abc.Mapping)\[[str](https://docs.python.org/3/builtins/stdtypes.html#str), [Question](../types/questions.md#typesafe_sdk.Question)],
> &#x20;   \*,
> &#x20;   model: [str](https://docs.python.org/3/builtins/stdtypes.html#str) | None = None,
> &#x20;   assets: [Mapping](https://docs.python.org/3/library/collections.abc.html#collections.abc.Mapping)\[[str](https://docs.python.org/3/builtins/stdtypes.html#str), [Asset](../types/assets.md#typesafe_sdk.Asset)] | None = None,
> &#x20;   retry: [RetryPolicy](../retries.md#typesafe_sdk.RetryPolicy) | None = None,
> &#x20;   timeout: [float](https://docs.python.org/3/builtins/functions.html#float)
> &#x20;   | httpx2.Timeout
> &#x20;   | None = None,
> &#x20;   extra\_headers: [Mapping](https://docs.python.org/3/library/collections.abc.html#collections.abc.Mapping)\[[str](https://docs.python.org/3/builtins/stdtypes.html#str), [str](https://docs.python.org/3/builtins/stdtypes.html#str)]
> &#x20;   | None = None,
> &#x20;   extra\_body: [Mapping](https://docs.python.org/3/library/collections.abc.html#collections.abc.Mapping)\[[str](https://docs.python.org/3/builtins/stdtypes.html#str), [JSONValue](../types/common.md#typesafe_sdk.JSONValue) | None]
> &#x20;   | None = None,
> &#x20;   response\_model: [type](https://docs.python.org/3/builtins/functions.html#type)\[ResponseT]
> &#x20;   | None = None,
> ) -> [SystemOneResponse](../types/responses.md#typesafe_sdk.SystemOneResponse) | ResponseT`</pre>

> [tab] 오버로드 1
>
> 시그니처 복사
> <pre tabIndex="0" aria-label="SDK signature">`system\_one(
> &#x20;   state: [Content](../types/common.md#typesafe_sdk.Content),
> &#x20;   questions: [Mapping](https://docs.python.org/3/library/collections.abc.html#collections.abc.Mapping)\[[str](https://docs.python.org/3/builtins/stdtypes.html#str), [Question](../types/questions.md#typesafe_sdk.Question)],
> &#x20;   \*,
> &#x20;   model: [str](https://docs.python.org/3/builtins/stdtypes.html#str) | None = None,
> &#x20;   assets: [Mapping](https://docs.python.org/3/library/collections.abc.html#collections.abc.Mapping)\[[str](https://docs.python.org/3/builtins/stdtypes.html#str), [Asset](../types/assets.md#typesafe_sdk.Asset)] | None = None,
> &#x20;   retry: [RetryPolicy](../retries.md#typesafe_sdk.RetryPolicy) | None = None,
> &#x20;   timeout: [float](https://docs.python.org/3/builtins/functions.html#float)
> &#x20;   | httpx2.Timeout
> &#x20;   | None = None,
> &#x20;   extra\_headers: [Mapping](https://docs.python.org/3/library/collections.abc.html#collections.abc.Mapping)\[[str](https://docs.python.org/3/builtins/stdtypes.html#str), [str](https://docs.python.org/3/builtins/stdtypes.html#str)]
> &#x20;   | None = None,
> &#x20;   extra\_body: [Mapping](https://docs.python.org/3/library/collections.abc.html#collections.abc.Mapping)\[[str](https://docs.python.org/3/builtins/stdtypes.html#str), [JSONValue](../types/common.md#typesafe_sdk.JSONValue) | None]
> &#x20;   | None = None,
> &#x20;   response\_model: None = None,
> ) -> [SystemOneResponse](../types/responses.md#typesafe_sdk.SystemOneResponse)`</pre>

> [tab] 오버로드 2
>
> 시그니처 복사
> <pre tabIndex="0" aria-label="SDK signature">`system\_one(
> &#x20;   state: [Content](../types/common.md#typesafe_sdk.Content),
> &#x20;   questions: [Mapping](https://docs.python.org/3/library/collections.abc.html#collections.abc.Mapping)\[[str](https://docs.python.org/3/builtins/stdtypes.html#str), [Question](../types/questions.md#typesafe_sdk.Question)],
> &#x20;   \*,
> &#x20;   model: [str](https://docs.python.org/3/builtins/stdtypes.html#str) | None = None,
> &#x20;   assets: [Mapping](https://docs.python.org/3/library/collections.abc.html#collections.abc.Mapping)\[[str](https://docs.python.org/3/builtins/stdtypes.html#str), [Asset](../types/assets.md#typesafe_sdk.Asset)] | None = None,
> &#x20;   retry: [RetryPolicy](../retries.md#typesafe_sdk.RetryPolicy) | None = None,
> &#x20;   timeout: [float](https://docs.python.org/3/builtins/functions.html#float)
> &#x20;   | httpx2.Timeout
> &#x20;   | None = None,
> &#x20;   extra\_headers: [Mapping](https://docs.python.org/3/library/collections.abc.html#collections.abc.Mapping)\[[str](https://docs.python.org/3/builtins/stdtypes.html#str), [str](https://docs.python.org/3/builtins/stdtypes.html#str)]
> &#x20;   | None = None,
> &#x20;   extra\_body: [Mapping](https://docs.python.org/3/library/collections.abc.html#collections.abc.Mapping)\[[str](https://docs.python.org/3/builtins/stdtypes.html#str), [JSONValue](../types/common.md#typesafe_sdk.JSONValue) | None]
> &#x20;   | None = None,
> &#x20;   response\_model: [type](https://docs.python.org/3/builtins/functions.html#type)\[ResponseT],
> ) -> ResponseT`</pre>

텍스트, 구조화된 state(상태), asset에 관해 이름을 붙인 질문들에 답합니다.

자세한 내용은 [System One](https://docs.typesafe.ai/go/system-one.md)을 참고하세요.

Parameters:

* **`state`** (`Content`) –

  평가할 텍스트, asset, JSON 객체 또는 배열입니다. 자세한 내용은 [state](https://docs.typesafe.ai/go/state.md)를 참고하세요.
* **`questions`** (`Mapping[str, Question]`) –

  이름을 질문 객체나 원시 딕셔너리에 대응시킨 매핑입니다. 비어 있으면 안 됩니다.
* **`model`** (`str | None`, default: `None` ) –

  이 호출에만 적용할 모델입니다. `None`이면 클라이언트 기본값을 물려받습니다.
* **`assets`** (`Mapping[str, Asset] | None`, default: `None` ) –

  `state`나 질문 안에서 asset을 가리킬 때 쓴 참조를 키로 하는 asset 매핑입니다. [미디어 asset 첨부하기](../../usage/basic.md#attaching-media-assets)를 참고하세요.
* **`retry`** (`RetryPolicy | None`, default: `None` ) –

  선택적 retry 정책입니다. 이 호출에 한해 클라이언트 수준의 값을 덮어씁니다.
* **`timeout`** (`float | httpx2.Timeout | None`, default: `None` ) –

  http 작업에 적용할 선택적 타임아웃(초)입니다. 이 호출에 한해 클라이언트 수준의 값을 덮어씁니다.
* **`extra_headers`** (`Mapping[str, str] | None`, default: `None` ) –

  추가로 설정할 요청 헤더입니다.
* **`extra_body`** (`Mapping[str, JSONValue | None] | None`, default: `None` ) –

  요청 본문의 최상위에 추가할 필드입니다. `state`, `model`, `questions`, `assets`가 설정된 뒤에 본문 위로 얕게 병합됩니다. 병합할 때는 나중에 쓴 값이 이깁니다. 즉 이 필드들과 키가 겹치면 그 값을 덮어쓰며, 객체 값은 깊게 병합되지 않고 통째로 교체됩니다.
* **`response_model`** (`type[ResponseT] | None`, default: `None` ) –

  JSON 응답 본문을 기술하는 선택적 Pydantic `BaseModel` 타입입니다. 중첩된 답변 모델도 여기에 포함됩니다.

Returns(반환값):

* `SystemOneResponse | ResponseT` –

  `response_model`의 인스턴스를 반환합니다. 사용자 정의 모델을 전달하지 않았다면 `SystemOneResponse`를 반환하며,
* `SystemOneResponse | ResponseT` –

  여기에는 질문 이름을 키로 하는 답변과 모델, token(토큰) 사용량 정보가 담깁니다.

Raises:

* `TypeSafeError` –

  질문이 비어 있거나 asset을 읽을 수 없는 경우처럼, 전송하기도 전에 요청이 유효하지 않은 것으로 판명된 경우에 발생합니다.
* `TypeSafeAPIError` –

  retry를 모두 거친 뒤에도 서버가 실패한 HTTP 응답을 반환한 경우에 발생합니다.
* `TypeSafeAPIConnectionError` –

  retry를 모두 거친 뒤에도 요청이 연결에 실패하거나 타임아웃된 경우에 발생합니다.
* `TypeSafeAPIResponseValidationError` –

  응답 본문이 응답 모델과 일치하지 않는 경우에 발생합니다.

Examples:

이름 있는 인자로 질문을 만듭니다.

```python
with TypeSafeClient() as client:
    result = client.system_one(
        state="I was charged twice. Please help.",
        questions={
            "billing": Noul(instructions="Is this about billing?"),
            "tone": Choice(
                instructions="What is the tone?",
                criteria={"calm": None, "angry": None},
            ),
        },
    )
    assert 0 <= result.nouls["billing"].noul <= 1
    assert result.choices["tone"].choice in {"calm", "angry"}
```

질문을 딕셔너리로 전달합니다.

```python
with TypeSafeClient() as client:
    result = client.system_one(
        state={"message": "I was charged twice. Please help."},
        questions={
            "billing": {"type": "noul", "instructions": "Is this about billing?"},
            "tone": {
                "type": "choice",
                "instructions": "What is the tone?",
                "criteria": {"calm": None, "angry": None},
            },
        },
    )
    assert 0 <= result.nouls["billing"].noul <= 1
    assert result.choices["tone"].choice in {"calm", "angry"}
```

### close

```python
close() -> None
```

네트워크 리소스를 해제하고 내부 HTTP 클라이언트를 닫습니다. 직접 전달한 클라이언트도 함께 닫습니다.

## Models 리소스

[`TypeSafeClient.models`](sync.md#typesafe_sdk.TypeSafeClient.models)를 통해 접근합니다.

### typesafe\_sdk.Models

계정에서 사용할 수 있는 모델에 접근합니다. `TypeSafeClient.models`를 통해 접근합니다.

#### list

시그니처 복사
<pre tabIndex="0" aria-label="SDK signature">`list(
&#x20;   \*,
&#x20;   retry: [RetryPolicy](../retries.md#typesafe_sdk.RetryPolicy) | None = None,
&#x20;   timeout: [float](https://docs.python.org/3/builtins/functions.html#float)
&#x20;   | httpx2.Timeout
&#x20;   | None = None,
&#x20;   extra\_headers: [Mapping](https://docs.python.org/3/library/collections.abc.html#collections.abc.Mapping)\[[str](https://docs.python.org/3/builtins/stdtypes.html#str), [str](https://docs.python.org/3/builtins/stdtypes.html#str)]
&#x20;   | None = None,
) -> [ListModelsResponse](../types/responses.md#typesafe_sdk.ListModelsResponse)`</pre>

계정에서 사용할 수 있는 모델 목록을 조회합니다.

Parameters:

* **`retry`** (`RetryPolicy | None`, default: `None` ) –

  선택적 retry 정책입니다. 이 호출에 한해 클라이언트 수준의 값을 덮어씁니다.
* **`timeout`** (`float | httpx2.Timeout | None`, default: `None` ) –

  이 작업에만 적용할 타임아웃입니다. `None`이면 클라이언트 설정을 물려받습니다.
* **`extra_headers`** (`Mapping[str, str] | None`, default: `None` ) –

  추가 요청 헤더를 덮어쓸 값입니다. 인증, SDK 식별, `Accept` 헤더는 보호되어 바뀌지 않습니다.

Returns:

* `ListModelsResponse` –

  `ListModelsResponse`를 반환합니다. 그 `models`에는 각 모델의 이름, 설명,
* `ListModelsResponse` –

  출시일이 담깁니다.

Raises:

* `TypeSafeAPIError` –

  retry를 모두 거친 뒤에도 서버가 실패한 HTTP 응답을 반환한 경우에 발생합니다.
* `TypeSafeAPIConnectionError` –

  retry를 모두 거친 뒤에도 요청이 연결에 실패하거나 타임아웃된 경우에 발생합니다.

Examples:

```python
from typesafe_sdk import TypeSafeClient

with TypeSafeClient() as client:
    models = client.models.list()
```


이 문서는 개발자 문서 플랫폼인 [Mintlify](https://mintlify.com)로 제작하고 호스팅합니다.
