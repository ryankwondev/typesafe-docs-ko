# Interface: TypeSafeClientConfig

클라이언트 옵션입니다. 명시적으로 넘긴 값이 환경 변수보다 우선하고, 그다음이 SDK 기본값입니다.

## Properties

### apiKey?

```ts
optional apiKey?: string;
```

필수 API 키입니다. 지정하지 않으면 `TYPESAFE_API_KEY`를 사용합니다.

***

### baseURL?

```ts
optional baseURL?: string;
```

API 루트입니다. 지정하지 않으면 `TYPESAFE_BASE_URL`을, 그것도 없으면 `https://api.typesafe.ai`를 사용합니다.

***

### compression?

```ts
optional compression?: Compression;
```

요청 본문 압축 방식이며, `Content-Encoding`으로 전송됩니다. 기본값: 지원하는 런타임(Node.js 22.15 이상)에서는
`"zstd"`, 그 밖에는 `false`. 1 KiB 미만인 본문은 항상 압축하지 않고
보냅니다. 압축된 요청을 받지 않는 API로 보낼 때처럼 모든 본문을 압축 없이 보내야 한다면
`false`를 넘깁니다.

***

### dangerouslyAllowBrowser?

```ts
optional dangerouslyAllowBrowser?: boolean;
```

브라우저에서의 사용을 허용합니다. 이 경우 API 키가 페이지 사용자에게 노출됩니다. 기본값: false.

***

### defaultHeaders?

```ts
optional defaultHeaders?: Record<string, string>;
```

추가 요청 헤더입니다. 호출 단위로 지정한 헤더가 우선합니다.

***

### defaultModel?

```ts
optional defaultModel?: string;
```

기본 모델입니다. 지정하지 않으면 `TYPESAFE_DEFAULT_MODEL`을, 그것도 없으면 `jev-latest`를 사용합니다.

***

### fetch?

```ts
optional fetch?: Fetch;
```

전송 계층 설정이나 테스트에 쓰는 사용자 정의 HTTP fetch 구현입니다. 기본값: 전역 `fetch`.

***

### logger?

```ts
optional logger?: Logger;
```

`logLevel` 이상의 로그만 받도록 필터링되는 로거입니다. 기본값: 접두어를 붙인 `console`.

***

### logLevel?

```ts
optional logLevel?: LogLevel;
```

로그 레벨입니다. 지정하지 않으면 `TYPESAFE_LOG_LEVEL`을, 그것도 없으면 `warn`을 사용합니다.
`info`는 요청 요약을 기록하고, `debug`는 여기에 헤더와 본문을 더합니다.
알려진 인증 정보 헤더는 가려지지만, 본문은 가려지지 않습니다.

***

### retry?

```ts
optional retry?: Partial<RetryPolicy>;
```

retry(재시도) 재정의 값입니다. 생략한 필드는 `RetryPolicy`의 기본값을 사용합니다.

***

### timeout?

```ts
optional timeout?: number;
```

시도 한 번당 타임아웃(밀리초)이며, 재시도 전체에 걸친 총 시간 한도는 없습니다. 기본값: 10000.


이 문서는 개발자 문서 플랫폼인 [Mintlify](https://mintlify.com)로 제작하고 호스팅합니다.
