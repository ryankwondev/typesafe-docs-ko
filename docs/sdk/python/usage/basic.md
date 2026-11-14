# 기본 사용법

> TypeSafe Python SDK를 사용하는 방법과 자주 쓰는 패턴을 안내합니다.

## System One API 호출하기

> [tab] Async
>
> ```python theme={null}
> import asyncio
>
> from typesafe_sdk import AsyncTypeSafeClient, Choice, Noul, Score
>
>
> async def main() -> None:
>     async with AsyncTypeSafeClient() as client:
>         result = await client.system_one(
>             "I was charged twice. Please help ASAP.",
>             {
>                 "billing": Noul(instructions="Is this about billing?"),
>                 "tone": Choice(
>                     instructions="What is the tone?",
>                     criteria={"calm": None, "angry": None},
>                 ),
>                 "urgency": Score(
>                     instructions="How urgent is this?",
>                     criteria=["low", "medium", "high"],
>                 ),
>             },
>         )
>         print(
>             result.nouls["billing"].noul,
>             result.choices["tone"].choice,
>             result.scores["urgency"].score,
>         )
>
>
> asyncio.run(main())
> ```

> [tab] Sync
>
> ```python theme={null}
> from typesafe_sdk import Choice, Noul, Score, TypeSafeClient
>
> client = TypeSafeClient()
> state = "I was charged twice. Please help ASAP."
> questions = {
>     "billing": Noul(instructions="Is this about billing?"),
>     "tone": Choice(
>         instructions="What is the tone?", criteria={"calm": None, "angry": None}
>     ),
>     "urgency": Score(
>         instructions="How urgent is this?", criteria=["low", "medium", "high"]
>     ),
> }
> result = client.system_one(state, questions)
> print(
>     result.nouls["billing"].noul,
>     result.choices["tone"].choice,
>     result.scores["urgency"].score,
> )
> ```

## 모델 선택하기

사용할 수 있는 모델을 조회합니다.

> [tab] Async
>
> ```python theme={null}
> import asyncio
>
> from typesafe_sdk import AsyncTypeSafeClient
>
>
> async def main() -> None:
>     async with AsyncTypeSafeClient() as client:
>         print(await client.models.list())
>
>
> asyncio.run(main())
> ```

> [tab] Sync
>
> ```python theme={null}
> from typesafe_sdk import TypeSafeClient
>
> print(TypeSafeClient().models.list())
> ```

클라이언트를 생성할 때 모델을 지정합니다.

> [tab] Async
>
> ```python theme={null}
> client = AsyncTypeSafeClient(model="jev")
> ```

> [tab] Sync
>
> ```python theme={null}
> client = TypeSafeClient(model="jev")
> ```

사용할 수 있는 모델과 각 모델의 기능, 요금, 모델 별칭은 [Models 리소스 레퍼런스](https://docs.typesafe.ai/go/models.md)에서 자세히 확인하세요.

## 미디어 asset 첨부하기

System One 요청에는 미디어 asset(첨부 자산)을 포함할 수 있습니다.

> [!NOTE]
>
> **지원하는 asset 유형**
>
> 지금은 이미지만 지원하며, 앞으로 더 많은 모달리티를 지원할 예정입니다.
>
> > [!WARNING]
> >
> > **이미지는 아직 dreamer-ready 단계입니다**
> >
> > 2026년 11월 14일 기준으로 이미지 지원은 dreamer-ready 단계에 머물러 있습니다. 그래도 써 보고 싶다면 HTTP 헤더로 이 기능을 직접 켤 수 있습니다.
> >
> > ```python theme={null}
> > client = TypeSafeClient(headers={"TypeSafe-Danger-Zone-Images": "true"})
> > ```
>
> 이미지 지원에 관한 자세한 내용은 다음 문서를 참고하세요.
>
> * [**API 문서**](https://docs.typesafe.ai/go/images.md): TypeSafe의 System One API에서 이미지를 켜고 사용하는 방법
> * [**SDK 문서**](../api/types/assets.md#typesafe_sdk.Image): `typesafe-sdk`로 이미지를 다루는 방법에 관한 설명과 코드 예제

> [!NOTE]
>
> **업로드 형식**
>
> asset이 포함된 요청은 `multipart/form-data`로 전송되며, asset마다 원시 바이트가 별도의 파트에 담깁니다.

***

`state`(상태)나 `questions`의 내용 안에 있는 **asset 참조**는 클라이언트에 따로 전달한 실제 asset 데이터로 대체됩니다.

**asset 참조**를 만드는 방법은 두 가지입니다.

1. [자동 참조](basic.md#automatic-references): asset을 내용 안에 직접 넣거나 문자열에 포매팅해 넣습니다. 그러면 SDK가 asset마다 고유한 참조를 자동으로 생성합니다.
2. [이름 있는 참조](basic.md#named-references): asset의 참조를 직접 정하고, `system_one`의 `assets` 인자로 전달합니다.

두 가지 asset 참조는 한 요청 안에서 함께 쓸 수 있습니다.

### 자동 참조

#### 구조화된 객체 안의 asset

asset을 `state`에 직접 넣거나, 질문의 instructions(지시문)와 criteria(판단 기준)에 직접 넣습니다.

> [tab] Async
>
> ```python theme={null}
> from pathlib import Path
>
> from typesafe_sdk import AsyncTypeSafeClient, Image, Noul, SystemOneResponse
>
>
> async def compare_return(
>     client: AsyncTypeSafeClient, returned: Path, listing: Path, *, model: str
> ) -> SystemOneResponse:
>     return await client.system_one(
>         state=["The customer returned this item:", Image(returned)],
>         questions={
>             "same_item": Noul(
>                 instructions=["Is it the item in this listing photo?", Image(listing)]
>             )
>         },
>         model=model,
>     )
> ```

> [tab] Sync
>
> ```python theme={null}
> from pathlib import Path
>
> from typesafe_sdk import Image, Noul, SystemOneResponse, TypeSafeClient
>
>
> def compare_return(
>     client: TypeSafeClient, returned: Path, listing: Path, *, model: str
> ) -> SystemOneResponse:
>     return client.system_one(
>         state=["The customer returned this item:", Image(returned)],
>         questions={
>             "same_item": Noul(
>                 instructions=["Is it the item in this listing photo?", Image(listing)]
>             )
>         },
>         model=model,
>     )
> ```

> [!NOTE]
>
> **참고**
>
> 내부적으로는 asset마다 uuid 형태의 고유한 참조가 자동으로 부여됩니다.

#### 텍스트 안의 asset

`client.attach_assets`로 asset을 클라이언트에 첨부하고, 텍스트 안에서는 `str(asset)`으로 참조합니다.

> [tab] Async
>
> ```python theme={null}
> from pathlib import Path
>
> from typesafe_sdk import AsyncTypeSafeClient, Image, Noul, SystemOneResponse
>
>
> async def check_damage(
>     client: AsyncTypeSafeClient, returned: Path, *, model: str
> ) -> SystemOneResponse:
>     photo = Image(returned)
>     with client.attach_assets(photo):
>         return await client.system_one(
>             state=f"The customer says {photo} arrived scratched.",
>             questions={"damaged": Noul(instructions="Is the item damaged?")},
>             model=model,
>         )
> ```

> [tab] Sync
>
> ```python theme={null}
> from pathlib import Path
>
> from typesafe_sdk import Image, Noul, SystemOneResponse, TypeSafeClient
>
>
> def check_damage(
>     client: TypeSafeClient, returned: Path, *, model: str
> ) -> SystemOneResponse:
>     photo = Image(returned)
>     with client.attach_assets(photo):
>         return client.system_one(
>             state=f"The customer says {photo} arrived scratched.",
>             questions={"damaged": Noul(instructions="Is the item damaged?")},
>             model=model,
>         )
> ```

> [!WARNING]
>
> **경고**
>
> `with client.attach_assets` context manager(컨텍스트 매니저) 블록을 빠뜨리지 마세요! 이 블록이 없으면 SDK가 텍스트로 렌더링된 asset 참조를 원래 asset 데이터와 연결하지 못합니다.

### 이름 있는 참조

asset의 참조를 직접 정하려면, 그 참조를 키로 삼아 `assets`로 전달합니다. 일반 텍스트와 우연히 일치하지 않도록 `<product>`처럼 눈에 띄는 구분자를 쓰세요.

> [tab] Async
>
> ```python theme={null}
> from pathlib import Path
>
> from typesafe_sdk import AsyncTypeSafeClient, Image, Noul, SystemOneResponse
>
>
> async def inspect_product(
>     client: AsyncTypeSafeClient, photo: Path, *, model: str
> ) -> SystemOneResponse:
>     return await client.system_one(
>         state="Inspect this returned product: <product>",
>         questions={"damaged": Noul(instructions="Is the product damaged?")},
>         assets={"<product>": Image(photo)},
>         model=model,
>     )
> ```

> [tab] Sync
>
> ```python theme={null}
> from pathlib import Path
>
> from typesafe_sdk import Image, Noul, SystemOneResponse, TypeSafeClient
>
>
> def inspect_product(
>     client: TypeSafeClient, photo: Path, *, model: str
> ) -> SystemOneResponse:
>     return client.system_one(
>         state="Inspect this returned product: <product>",
>         questions={"damaged": Noul(instructions="Is the product damaged?")},
>         assets={"<product>": Image(photo)},
>         model=model,
>     )
> ```

> [!TIP]
>
> **팁**
>
> `assets`로 전달한 asset을 내용 안에 직접 넣어도 됩니다. 이 경우 asset은 지정한 이름으로 한 번만 전송됩니다.

## 타입이 지정된 `system_one` 응답

`system_one`에 응답 모델을 전달하면 응답을 더 *type-safe(타입 안전)*하게 다룰 수 있습니다.

> [tab] Async
>
> ```python theme={null}
> import asyncio
>
> from typesafe_sdk import AsyncTypeSafeClient, Noul, NoulAnswer, SystemOneResponse
>
>
> class BillingResponse(SystemOneResponse):
>     billing: NoulAnswer
>
>
> async def main() -> None:
>     async with AsyncTypeSafeClient() as client:
>         result = await client.system_one(
>             "I was charged twice.",
>             {"billing": Noul(instructions="Is this about billing?")},
>             response_model=BillingResponse,
>         )
>         assert 0 <= result.billing.noul <= 1
>         assert result.billing == result.nouls["billing"]
>         print(result.request_id)
>
>
> asyncio.run(main())
> ```

> [tab] Sync
>
> ```python theme={null}
> from typesafe_sdk import Noul, NoulAnswer, SystemOneResponse, TypeSafeClient
>
>
> class BillingResponse(SystemOneResponse):
>     billing: NoulAnswer
>
>
> with TypeSafeClient() as client:
>     result = client.system_one(
>         "I was charged twice.",
>         {"billing": Noul(instructions="Is this about billing?")},
>         response_model=BillingResponse,
>     )
>     assert 0 <= result.billing.noul <= 1
>     assert result.billing == result.nouls["billing"]
>     print(result.request_id)
> ```

### 사용자 정의 응답 타입

`SystemOneResponse`를 상속하지 않고 완전히 새로운 응답 모델을 정의할 수도 있습니다.

> [tab] Async
>
> ```python theme={null}
> import asyncio
>
> from pydantic import BaseModel
>
> from typesafe_sdk import AsyncTypeSafeClient, Noul, NoulAnswer
>
>
> class BillingAnswers(BaseModel):
>     billing: NoulAnswer
>
>
> class BillingResponse(BaseModel):
>     answers: BillingAnswers
>
>
> async def main() -> None:
>     async with AsyncTypeSafeClient() as client:
>         result = await client.system_one(
>             "I was charged twice.",
>             {"billing": Noul(instructions="Is this about billing?")},
>             response_model=BillingResponse,
>         )
>         assert 0 <= result.answers.billing.noul <= 1
>
>
> asyncio.run(main())
> ```

> [tab] Sync
>
> ```python theme={null}
> from pydantic import BaseModel
>
> from typesafe_sdk import Noul, NoulAnswer, TypeSafeClient
>
>
> class BillingAnswers(BaseModel):
>     billing: NoulAnswer
>
>
> class BillingResponse(BaseModel):
>     answers: BillingAnswers
>
>
> result = TypeSafeClient().system_one(
>     "I was charged twice.",
>     {"billing": Noul(instructions="Is this about billing?")},
>     response_model=BillingResponse,
> )
> assert 0 <= result.answers.billing.noul <= 1
> ```

## Retries

직접 만든 [`RetryPolicy`](../api/retries.md)를 클라이언트나 개별 호출의 `retry`로 전달해 retry(재시도) 동작을 설정합니다. API 키가 유효하지 않으면 요청이나 retry가 일어나기 전, 클라이언트를 생성하는 시점에 `TypeSafeError`가 발생합니다.

클라이언트에 설정하는 경우:

> [tab] Async
>
> ```python theme={null}
> from typesafe_sdk import AsyncTypeSafeClient, RetryPolicy
>
> client = AsyncTypeSafeClient(
>     retry=RetryPolicy(max_retries=3, backoff_max=0.2, timeout=1.0)
> )
> ```

> [tab] Sync
>
> ```python theme={null}
> from typesafe_sdk import RetryPolicy, TypeSafeClient
>
> client = TypeSafeClient(retry=RetryPolicy(max_retries=3, backoff_max=0.2, timeout=1.0))
> ```

호출마다 설정하는 경우:

> [tab] Async
>
> ```python theme={null}
> import asyncio
>
> from typesafe_sdk import AsyncTypeSafeClient, RetryPolicy
>
>
> async def main() -> None:
>     async with AsyncTypeSafeClient() as client:
>         await client.system_one(
>             state,
>             questions,
>             retry=RetryPolicy(max_retries=3, backoff_max=0.2, timeout=1.0),
>         )
>
>
> asyncio.run(main())
> ```

> [tab] Sync
>
> ```python theme={null}
> from typesafe_sdk import RetryPolicy
>
> client.system_one(
>     state, questions, retry=RetryPolicy(max_retries=3, backoff_max=0.2, timeout=1.0)
> )
> ```

## 오류 처리

SDK가 발생시키는 [예외](../api/exceptions.md)를 처리합니다.

> [tab] Async
>
> ```python theme={null}
> import asyncio
>
> from typesafe_sdk import AsyncTypeSafeClient, TypeSafeAPIError
>
>
> async def main() -> None:
>     async with AsyncTypeSafeClient() as client:
>         try:
>             await client.system_one(state, questions)
>         except TypeSafeAPIError as error:
>             print(error.status, error.request_id)
>
>
> asyncio.run(main())
> ```

> [tab] Sync
>
> ```python theme={null}
> from typesafe_sdk import TypeSafeAPIError
>
> try:
>     client.system_one(state, questions)
> except TypeSafeAPIError as error:
>     print(error.status, error.request_id)
> ```

## 환경 변수

SDK는 다음 환경 변수를 읽어서 사용합니다.

| 변수 | 설정 대상 | 기본값 |
| - | - | - |
| `TYPESAFE_API_KEY` | API 키(필수) | 없음 |
| `TYPESAFE_BASE_URL` | API 루트 URL | `https://api.typesafe.ai` |
| `TYPESAFE_DEFAULT_MODEL` | 기본 모델 | `jev-latest` |
| `TYPESAFE_LOG_LEVEL` | `typesafe_sdk` 로거의 수준. import할 때 한 번 적용됩니다 | 설정되지 않음 |

SDK 기본값은 [상수 레퍼런스](../api/constants.md)를 참고하세요.

`api_key`나 `TYPESAFE_API_KEY`로 전달한 API 키는 앞뒤 공백이 제거됩니다. 키 파일에서 딸려 온 줄바꿈도 함께 제거됩니다. 빈 키, 중간에 공백이 있는 키, 제어 문자나 ASCII가 아닌 문자가 들어 있는 키는 요청을 보내기 전에 거부됩니다. 빈 키를 명시적으로 전달하면 환경 변수 값으로 대체되지 않습니다.

## base URL 설정하기

SDK를 다른 API url과 함께 쓰려면 클라이언트의 `base_url`이나 환경 변수 `TYPESAFE_BASE_URL`을 설정하세요. 이때 대체 API는 [TypeSafe OpenAPI 스펙](https://api.typesafe.ai/docs/)을 따라야 합니다.

예를 들어 AI 게이트웨이의 API 키와 모델 ID를 사용해 게이트웨이를 거쳐 연결할 수 있습니다.

> [tab] OpenRouter
>
> OpenRouter API 키와 [OpenRouter 모델 ID](https://openrouter.ai/~typesafe/jev-latest/)를 사용합니다.
>
> > [tab] Async 클라이언트
> >
> > ```python theme={null}
> > import asyncio
> > import os
> >
> > from typesafe_sdk import AsyncTypeSafeClient, Noul
> >
> >
> > async def main() -> None:
> >     async with AsyncTypeSafeClient(
> >         api_key=os.environ["OPENROUTER_API_KEY"],
> >         base_url="https://openrouter.ai/api",
> >         model="~typesafe/jev-latest",
> >     ) as client:
> >         result = await client.system_one(
> >             "I was charged twice.",
> >             {"billing": Noul(instructions="Is this about billing?")},
> >         )
> >         print(result.nouls["billing"].noul)
> >
> >
> > asyncio.run(main())
> > ```
>
> > [tab] Sync 클라이언트
> >
> > ```python theme={null}
> > import os
> >
> > from typesafe_sdk import Noul, TypeSafeClient
> >
> > with TypeSafeClient(
> >     api_key=os.environ["OPENROUTER_API_KEY"],
> >     base_url="https://openrouter.ai/api",
> >     model="~typesafe/jev-latest",
> > ) as client:
> >     result = client.system_one(
> >         "I was charged twice.",
> >         {"billing": Noul(instructions="Is this about billing?")},
> >     )
> >     print(result.nouls["billing"].noul)
> > ```

> [tab] Vercel AI Gateway
>
> [Vercel의 TypeSafe 호환 API](https://vercel.com/docs/ai-gateway/sdks-and-apis/typesafe)도 SDK와 함께 쓸 수 있습니다.
>
> > [tab] Async 클라이언트
> >
> > ```python theme={null}
> > import asyncio
> > import os
> >
> > from typesafe_sdk import AsyncTypeSafeClient, Noul
> >
> >
> > async def main() -> None:
> >     async with AsyncTypeSafeClient(
> >         api_key=os.environ["AI_GATEWAY_API_KEY"],
> >         base_url="https://ai-gateway.vercel.sh/typesafe",
> >         model="typesafe-ai/jev",
> >     ) as client:
> >         result = await client.system_one(
> >             "I was charged twice.",
> >             {"billing": Noul(instructions="Is this about billing?")},
> >         )
> >         print(result.nouls["billing"].noul)
> >
> >
> > asyncio.run(main())
> > ```
>
> > [tab] Sync 클라이언트
> >
> > ```python theme={null}
> > import os
> >
> > from typesafe_sdk import Noul, TypeSafeClient
> >
> > with TypeSafeClient(
> >     api_key=os.environ["AI_GATEWAY_API_KEY"],
> >     base_url="https://ai-gateway.vercel.sh/typesafe",
> >     model="typesafe-ai/jev",
> > ) as client:
> >     result = client.system_one(
> >         "I was charged twice.",
> >         {"billing": Noul(instructions="Is this about billing?")},
> >     )
> >     print(result.nouls["billing"].noul)
> > ```

> [tab] Pydantic AI Gateway
>
> [Pydantic AI Gateway API 키](https://pydantic.dev/articles/jev-pydantic-ai-gateway)를 사용합니다.
>
> > [tab] Async 클라이언트
> >
> > ```python theme={null}
> > import asyncio
> > import os
> >
> > from typesafe_sdk import AsyncTypeSafeClient, Noul
> >
> >
> > async def main() -> None:
> >     async with AsyncTypeSafeClient(
> >         api_key=os.environ["PYDANTIC_AI_GATEWAY_API_KEY"],
> >         base_url="https://gateway-us.pydantic.dev/proxy/typesafe",
> >         model="jev-latest",
> >     ) as client:
> >         result = await client.system_one(
> >             "I was charged twice.",
> >             {"billing": Noul(instructions="Is this about billing?")},
> >         )
> >         print(result.nouls["billing"].noul)
> >
> >
> > asyncio.run(main())
> > ```
>
> > [tab] Sync 클라이언트
> >
> > ```python theme={null}
> > import os
> >
> > from typesafe_sdk import Noul, TypeSafeClient
> >
> > with TypeSafeClient(
> >     api_key=os.environ["PYDANTIC_AI_GATEWAY_API_KEY"],
> >     base_url="https://gateway-us.pydantic.dev/proxy/typesafe",
> >     model="jev-latest",
> > ) as client:
> >     result = client.system_one(
> >         "I was charged twice.",
> >         {"billing": Noul(instructions="Is this about billing?")},
> >     )
> >     print(result.nouls["billing"].noul)
> > ```


이 문서는 개발자 문서 플랫폼인 [Mintlify](https://mintlify.com)로 제작하고 호스팅합니다.
