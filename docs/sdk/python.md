# TypeSafe Python SDK

> TypeSafe Python SDK를 설치하고 비동기 또는 동기 방식으로 API를 호출해 봅니다.

[Python SDK 소스 코드는 GitHub](https://github.com/typesafe-ai/typesafe-sdk-python)에서 볼 수 있습니다.

[TypeSafe](https://typesafe.ai) API를 위한 비동기 및 동기 Python 클라이언트입니다. TypeSafe 사용법은 [여기](../index.md)에서 확인하세요.

## 빠른 시작

1. SDK를 설치합니다.

   > [tab] uv
   >
   > ```sh theme={null}
   > uv add typesafe-sdk
   > ```

   > [tab] pip
   >
   > ```sh theme={null}
   > pip install typesafe-sdk
   > ```

   다음 extra(추가 설치 옵션)를 지원합니다(예: `typesafe-sdk[http2]`).

   * `http2`: [HTTP/2 지원](python/usage/advanced.md#http2)을 켭니다.
   * `pil`: [Pillow 이미지](python/api/types/assets.md#typesafe_sdk.Image.from_pil)를 전송합니다.
   * `zstd`: Python 3.13 이하에서 [요청 본문을 압축](python/usage/advanced.md#ensure-zstd-is-available)합니다.
2. 환경 변수 `TYPESAFE_API_KEY`를 설정합니다(키는 [여기](https://console.typesafe.ai/)에서 발급받습니다).
3. System One API를 호출합니다.

   > [tab] Async
   >
   > [AsyncTypeSafeClient](python/api/clients/async.md)를 사용하는 경우:
   >
   > ```python theme={null}
   > from typesafe_sdk import AsyncTypeSafeClient, Choice, Noul, Score
   >
   >
   > async def main() -> None:
   >     async with AsyncTypeSafeClient() as client:
   >         response = await client.system_one(
   >             state={"ticket": "I was charged twice. Please fix this ASAP."},
   >             questions={
   >                 "billing": Noul(instructions="Is this ticket about billing?"),
   >                 "tone": Choice(
   >                     instructions="What is the customer's tone?",
   >                     criteria={"calm": None, "frustrated": None, "angry": None},
   >                 ),
   >                 "urgency": Score(
   >                     instructions="How urgent is this ticket?",
   >                     criteria=["can wait", "this week", "today"],
   >                 ),
   >             },
   >         )
   >
   >     print(response.nouls["billing"].noul)
   >     print(response.choices["tone"].choice)
   >     print(response.scores["urgency"].score)
   > ```

   > [tab] Sync
   >
   > [TypeSafeClient](python/api/clients/sync.md)를 사용하는 경우:
   >
   > ```python theme={null}
   > from typesafe_sdk import Choice, Noul, Score, TypeSafeClient
   >
   > with TypeSafeClient() as client:
   >     response = client.system_one(
   >         state={"ticket": "I was charged twice. Please fix this ASAP."},
   >         questions={
   >             "billing": Noul(instructions="Is this ticket about billing?"),
   >             "tone": Choice(
   >                 instructions="What is the customer's tone?",
   >                 criteria={"calm": None, "frustrated": None, "angry": None},
   >             ),
   >             "urgency": Score(
   >                 instructions="How urgent is this ticket?",
   >                 criteria=["can wait", "this week", "today"],
   >             ),
   >         },
   >     )
   >
   > print(response.nouls["billing"].noul)
   > print(response.choices["tone"].choice)
   > print(response.scores["urgency"].score)
   > ```

## 다음 단계

[사용 가이드](python/usage/basic.md)에서 다음 내용을 알아보세요.

* [System One 모델 선택하기](python/usage/basic.md#choosing-a-model)
* [미디어 입력 보내기](python/usage/basic.md#attaching-media-assets)
* [타입이 지정된 응답 사용하기](python/usage/basic.md#typed-system_one-responses)
* [retry(재시도) 정책 설정하기](python/usage/basic.md#retries)
* [성능 최적화하기](python/usage/advanced.md#performance)


이 문서는 개발자 문서 플랫폼인 [Mintlify](https://mintlify.com)로 제작하고 호스팅합니다.
