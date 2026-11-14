# Interface: RequestOptions

클라이언트 설정을 재정의하는 호출 단위 옵션입니다.

## Properties

### headers?

```ts
optional headers?: Record<string, string>;
```

추가 헤더이며, `defaultHeaders` 위에 덮어써서 병합됩니다.

***

### retry?

```ts
optional retry?: Partial<RetryPolicy>;
```

이 호출에만 적용할 retry(재시도) 재정의 값입니다. 생략한 필드는 클라이언트 설정을 따릅니다.

***

### signal?

```ts
optional signal?: AbortSignal;
```

요청과 대기 중인 재시도를 취소하는 신호입니다.

***

### timeout?

```ts
optional timeout?: number;
```

시도 한 번당 타임아웃(밀리초)입니다. 재시도 전체에 걸친 총 시간 한도는 없습니다.


이 문서는 개발자 문서 플랫폼인 [Mintlify](https://mintlify.com)로 제작하고 호스팅합니다.
