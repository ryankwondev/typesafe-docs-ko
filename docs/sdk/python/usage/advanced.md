# 고급 사용법

> TypeSafe Python SDK의 성능 튜닝, 로깅, forward compatibility(전방 호환성)를 다룹니다.

## 성능

애플리케이션이 초당 수백 건의 요청을 보낸다면 아래 권장 사항을 따르세요.

### 클라이언트 재사용하기

요청을 여러 번 보낼 때는 클라이언트 인스턴스를 반드시 재사용하세요. 클라이언트를 새로 만들면 연결을 새로 맺고 TLS 핸드셰이크를 다시 하는 오버헤드가 생깁니다.

### `zstd`를 쓸 수 있는지 확인하기

클라이언트는 1 KiB 이상인 요청 본문을 [zstd](https://facebook.github.io/zstd/)로 압축합니다. 업로드 크기와 시간이 줄어들며, state(상태)가 클수록 효과가 큽니다.

Python 3.14보다 낮은 버전에서는 `zstd`가 설치되어 있어야 합니다.

```shell
uv add "typesafe-sdk[zstd]"
```

압축은 클라이언트 생성자의 `compression` 인자로 설정하거나 끌 수 있습니다.

### 비동기 코드에는 `aiohttp` 사용하기

요청 빈도가 높은 비동기 코드라면 [`httpx-aiohttp`](https://pypi.org/project/httpx-aiohttp/)가 제공하는 [aiohttp](https://docs.aiohttp.org/) 기반 HTTP 클라이언트를 전달하세요.

```shell
uv add "httpx-aiohttp[httpx2]"
```

```python
from httpx_aiohttp.httpx2 import Httpx2AiohttpClient

from typesafe_sdk import AsyncTypeSafeClient

client = AsyncTypeSafeClient(http_client=Httpx2AiohttpClient())
```

### 스레드마다 클라이언트 만들기

`TypeSafeClient`는 여러 스레드가 함께 쓸 수 있지만, 부하가 크면 연결 풀에서 병목이 생깁니다. 스레드마다 클라이언트를 하나씩 만들고, 그 스레드가 보내는 모든 요청에 재사용하세요.

> [!NOTE]
>
> **예시**
>
> ```python theme={null}
> import threading
>
> from typesafe_sdk import TypeSafeClient
>
>
> class ThreadClients(threading.local):
>     def __init__(self) -> None:
>         self.client = TypeSafeClient()
>
>
> clients = ThreadClients()
>
> # 어느 스레드에서든 `clients.client`는 그 스레드 전용 클라이언트입니다.
> ```

### HTTP/2

클라이언트는 기본적으로 HTTP/1.1을 사용하며, HTTP/2로 바꿀 수 있습니다.

* **HTTP/1.1**은 연결 하나로 요청을 한 번에 하나씩만 보내므로, 동시 요청마다 연결을 하나씩 엽니다. 열 수 있는 연결 수는 [연결 한도](advanced.md#keep-a-connection-limit)까지입니다. 동시 처리량은 연결 수에 비례해 늘어나지만, 새 연결마다 TLS 핸드셰이크 비용이 듭니다.
* **HTTP/2**는 한 클라이언트의 모든 요청을 연결 하나에 다중화해서 보냅니다. 핸드셰이크 한 번과 소켓 하나면 충분하지만, 그 연결 하나가 클라이언트의 처리량 상한이 됩니다.

요청 빈도가 높다면 기본값인 HTTP/1.1을 그대로 쓰세요. HTTP/2는 동시성이 낮거나 중간 정도이고 연결 수를 줄이고 싶을 때 쓰면 됩니다. 연결 수를 제한하는 프록시나 방화벽 뒤에 있는 경우가 그런 예입니다.

HTTP/2를 켜려면 `'typesafe-sdk[http2]'` extra(추가 설치 옵션)를 설치하고 HTTP/2 클라이언트를 전달하세요. 자세한 내용은 [`httpx2` HTTP/2 가이드](https://pydantic.dev/docs/httpx2/guides/http2/)를 참고하세요.

> [tab] Async
>
> ```python theme={null}
> import httpx2
>
> from typesafe_sdk import AsyncTypeSafeClient
>
> client = AsyncTypeSafeClient(http_client=httpx2.AsyncClient(http2=True))
> ```

> [tab] Sync
>
> ```python theme={null}
> import httpx2
>
> from typesafe_sdk import TypeSafeClient
>
> client = TypeSafeClient(http_client=httpx2.Client(http2=True))
> ```

### 연결 한도 유지하기

기본 클라이언트는 연결을 최대 100개까지 열고, 유휴 상태인 연결도 30초 동안 모두 열어 둡니다. 그래서 요청이 한꺼번에 몰려도 새 연결을 여는 대신 기존 연결을 재사용합니다. `http_client`를 직접 전달할 때는 한도를 알맞게 설정하세요.

> [!NOTE]
>
> **예시**
>
> > [tab] Async
> >
> > ```python theme={null}
> > import httpx2
> >
> > from typesafe_sdk import AsyncTypeSafeClient
> >
> > limits = httpx2.Limits(
> >     max_connections=200, max_keepalive_connections=200, keepalive_expiry=30
> > )
> > client = AsyncTypeSafeClient(http_client=httpx2.AsyncClient(limits=limits))
> > ```
>
> > [tab] Sync
> >
> > ```python theme={null}
> > import httpx2
> >
> > from typesafe_sdk import TypeSafeClient
> >
> > limits = httpx2.Limits(
> >     max_connections=200, max_keepalive_connections=200, keepalive_expiry=30
> > )
> > client = TypeSafeClient(http_client=httpx2.Client(limits=limits))
> > ```

## 로깅

SDK는 `typesafe_sdk` 로거에 로그를 남깁니다. [표준 logging](https://docs.python.org/3/library/logging.html) 가이드에 따라 설정하세요.

```python
import logging

logging.getLogger("typesafe_sdk").setLevel(logging.DEBUG)
```

또는 SDK를 import하기 전에 `TYPESAFE_LOG_LEVEL`을 `debug`, `info`, `warning`, `error`, `off` 중 하나로 설정해도 됩니다.

`info`는 요청마다 요약 한 줄을 남기고, `debug`는 여기에 더해 요청과 응답의 헤더와 본문도 남깁니다. 비밀 값이 담긴 헤더(authorization, API 키, 쿠키, 이름에 `token`이나 `secret`이 들어간 모든 헤더)는 로그 출력에서 가려집니다. 요청과 응답 본문은 가려지지 **않습니다**.

## Forward compatibility

SDK는 TypeSafe API가 바뀌어도 계속 동작합니다. 따라서 SDK 릴리스가 새 API 기능을 정식으로 지원하기 전에도 그 기능을 먼저 쓸 수 있습니다.

### 추가 요청 필드

API 요청 필드를 추가로 보내려면 [`extra_body`](../api/clients/sync.md)를 사용하세요. 아래의 `beam_width` 필드는 설명을 위한 예시입니다. API가 지원하는 필드만 보내세요.

> [tab] Async
>
> ```python theme={null}
> import asyncio
>
> from typesafe_sdk import AsyncTypeSafeClient, Noul
>
>
> async def main() -> None:
>     async with AsyncTypeSafeClient() as client:
>         await client.system_one(
>             "I was charged twice.",
>             {"billing": Noul(instructions="About billing?")},
>             extra_body={"beam_width": 4},
>         )
>
>
> asyncio.run(main())
> ```

> [tab] Sync
>
> ```python theme={null}
> from typesafe_sdk import Noul, TypeSafeClient
>
> with TypeSafeClient() as client:
>     client.system_one(
>         "I was charged twice.",
>         {"billing": Noul(instructions="About billing?")},
>         extra_body={"beam_width": 4},
>     )
> ```

### 원시 질문 딕셔너리

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
>         await client.system_one(
>             "I was charged twice.",
>             {
>                 "billing": {
>                     "type": "noul",
>                     "instructions": "About billing?",
>                     "weight": 2,
>                 }
>             },
>         )
>
>
> asyncio.run(main())
> ```

> [tab] Sync
>
> ```python theme={null}
> from typesafe_sdk import TypeSafeClient
>
> with TypeSafeClient() as client:
>     client.system_one(
>         "I was charged twice.",
>         {"billing": {"type": "noul", "instructions": "About billing?", "weight": 2}},
>     )
> ```

> [!TIP]
>
> **팁**
>
> 알 수 없는 필드는 forward compatibility를 위한 비상 수단입니다. 이때 나오는 타입 검사 오류는 무시하되, 가능하면 SDK를 업그레이드하는 쪽을 택하세요.

### 알 수 없는 답변 종류

SDK는 인식하지 못하는 답변 종류를 만나면 경고를 로그에 남기고 건너뜁니다. 그런 답변까지 포함한 전체 API 응답을 확인하려면 `raw_http_response`를 사용하세요.

> [tab] Async
>
> ```python theme={null}
> import asyncio
>
> from typesafe_sdk import AsyncTypeSafeClient, Noul
>
>
> async def main() -> None:
>     async with AsyncTypeSafeClient() as client:
>         result = await client.system_one(
>             "I was charged twice.",
>             {"billing": Noul(instructions="Is this about billing?")},
>         )
>         print(result.raw_http_response.json()["answers"])
>
>
> asyncio.run(main())
> ```

> [tab] Sync
>
> ```python theme={null}
> from typesafe_sdk import Noul, TypeSafeClient
>
> result = TypeSafeClient().system_one(
>     "I was charged twice.",
>     {"billing": Noul(instructions="Is this about billing?")},
> )
> raw_answers = result.raw_http_response.json()["answers"]
> ```

### 알 수 없는 응답 필드

SDK가 인식하는 응답에 알 수 없는 필드가 추가로 들어 있으면 무시합니다.


이 문서는 개발자 문서 플랫폼인 [Mintlify](https://mintlify.com)로 제작하고 호스팅합니다.
