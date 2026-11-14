# Class: TypeSafeClient

TypeSafe AI API용 클라이언트입니다.

## Constructors

### Constructor

```ts
new TypeSafeClient(config?): TypeSafeClient;
```

TypeSafe AI API용 클라이언트를 생성합니다.

명시적으로 넘긴 옵션이 환경 변수보다 우선하고, 그다음이 SDK 기본값입니다.
환경 변수 값이 비어 있거나 공백뿐이면 무시합니다.

#### Parameters

##### config?

[`TypeSafeClientConfig`](../interfaces/TypeSafeClientConfig.md) = `{}`

#### Returns

`TypeSafeClient`

#### Throws

API 키가 없거나, 설정이 유효하지 않거나, 지원하지 않는 런타임이거나,
요청한 압축 방식을 런타임이 지원하지 않는 경우.

## Properties

### baseURL

```ts
readonly baseURL: string;
```

끝의 슬래시를 제거한 API 루트입니다.

***

### compression

```ts
readonly compression: Compression;
```

요청 본문 압축 방식입니다. 본문을 압축하지 않고 보내는 경우에는 `false`입니다.

***

### defaultHeaders

```ts
readonly defaultHeaders: Readonly<Record<string, string>>;
```

모든 요청에 함께 보내는 추가 헤더입니다.

***

### defaultModel

```ts
readonly defaultModel: string;
```

요청에서 `model`을 생략했을 때 사용하는 모델입니다.

***

### fetch

```ts
readonly fetch: Fetch;
```

HTTP fetch 구현입니다.

***

### logger

```ts
readonly logger: Logger;
```

설정된 로거이며, `logLevel`에 맞춰 필터링됩니다.

***

### logLevel

```ts
readonly logLevel: LogLevel;
```

설정된 로그 상세 수준입니다.

***

### models

```ts
readonly models: Models;
```

계정에서 사용할 수 있는 모델입니다.

***

### retry

```ts
readonly retry: RetryPolicy;
```

생성자에서 넘긴 재정의 값을 반영한 retry(재시도) 설정입니다.

***

### timeout

```ts
readonly timeout: number;
```

시도 한 번당 타임아웃(밀리초)입니다.

## Methods

### systemOne()

```ts
systemOne<Q>(request, options?): APIPromise<SystemOneResult<Q>>;
```

텍스트, 구조화된 state(상태), asset(첨부 자산)에 대해 이름을 붙인 질문들에 답합니다.

#### Type Parameters

##### Q

`Q` *extends* [`Questions`](../interfaces/Questions.md)

#### Parameters

##### request

[`SystemOneRequest`](../interfaces/SystemOneRequest.md)\<`Q`>

state와 질문, 그리고 선택 사항인 모델 재정의와 asset입니다.
`state`, 질문 콘텐츠, `withAssets` 텍스트에 들어 있는 asset은 각자의 참조로 전송됩니다.

##### options?

[`RequestOptions`](../interfaces/RequestOptions.md) = `{}`

호출 단위의 타임아웃, retry, 헤더, 취소 설정입니다.

#### Returns

[`APIPromise`](APIPromise.md)\<[`SystemOneResult`](../interfaces/SystemOneResult.md)\<`Q`>>

질문 이름과 criteria(판단 기준)에 따라 타입이 정해진 답과 함께 모델, token(토큰) 사용량을 반환합니다. 아래의
모든 실패는 반환된 promise를 거부하며, 호출 자체가 예외를 던지지는 않습니다.

#### Throws

state가 `null`이거나 생략된 경우, 질문이 비어 있는 경우, score criteria가
항목이 두 개 이상인 목록이 아닌 경우, choice criteria가 비어 있지 않은 맵이 아닌 경우, noul 질문에
instructions(지시문)도 없고 설명이 달린 결과도 없는 경우, 요청에 첨부하지 않은 asset의 참조가
문자열에 들어 있는 경우, `assets`의 이름이 요청 안에 나타나지 않는 경우, 또는 `Blob`
asset을 읽을 수 없는 경우.

#### Throws

재시도 후에도 서버가 2xx가 아닌 응답을 반환하는 경우.

#### Throws

재시도 후에도 요청이 연결에 실패하거나 타임아웃되는 경우.

#### Throws

호출자가 요청을 중단한 경우.

#### Example

```ts
const { answers } = await client.systemOne({
  state: "I was charged twice. Please help.",
  questions: { billing: noul("Is this about billing?") },
});
console.log(answers.billing.noul);
```


이 문서는 개발자 문서 플랫폼인 [Mintlify](https://mintlify.com)로 제작하고 호스팅합니다.
