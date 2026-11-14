# Interface: RetryPolicy

retry(재시도) 설정입니다. 일부만 재정의하면 지정하지 않은 필드는 클라이언트 설정이나 SDK 기본값을 따릅니다.

## Properties

### apiConnectionError

```ts
readonly apiConnectionError: boolean;
```

응답 본문 수신이 중간에 끊긴 경우를 포함해 연결 실패(`APIConnectionError`)를 재시도합니다. 기본값: true.

***

### apiTimeoutError

```ts
readonly apiTimeoutError: boolean;
```

`APITimeoutError`를 재시도할지 여부입니다. 기본값: true.

***

### backoffInitialMs

```ts
readonly backoffInitialMs: number;
```

첫 번째 backoff(백오프) 대기 시간(밀리초)이며, `backoffMaxMs`에 이를 때까지 두 배씩 늘어납니다. 기본값: 500.

***

### backoffJitter

```ts
readonly backoffJitter: number;
```

각 backoff 대기 시간에서 무작위로 빼는 비율이며, 0에서 1 사이입니다. 기본값: 0.25.

***

### backoffMaxMs

```ts
readonly backoffMaxMs: number;
```

backoff 대기 시간의 최댓값(밀리초)입니다. 기본값: 5000.

***

### httpStatuses

```ts
readonly httpStatuses: ReadonlySet<number>;
```

재시도할 HTTP 상태 코드입니다. 기본값: 408, 429, 500~599.

***

### maxRetries

```ts
readonly maxRetries: number;
```

첫 시도 이후의 최대 재시도 횟수입니다. `0`이면 재시도하지 않습니다. 기본값: 2.

***

### maxRetryAfterMs

```ts
readonly maxRetryAfterMs: number;
```

서버가 지정한 재시도 대기 시간을 따르는 최댓값(밀리초)입니다. 이보다 긴 대기 시간에는 backoff를 사용합니다. 기본값: 60000.

***

### respectRetryAfter

```ts
readonly respectRetryAfter: boolean;
```

`Retry-After`와 `retry-after-ms`를 `maxRetryAfterMs` 범위 안에서 따릅니다. 기본값: true.


이 문서는 개발자 문서 플랫폼인 [Mintlify](https://mintlify.com)로 제작하고 호스팅합니다.
