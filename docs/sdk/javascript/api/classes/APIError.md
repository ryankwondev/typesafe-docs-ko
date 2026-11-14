# Class: APIError

API가 실패를 나타내는 HTTP 응답을 반환했습니다.

## Extends

* [`TypeSafeError`](TypeSafeError.md)

## Extended by

* [`AuthenticationError`](AuthenticationError.md)
* [`BadRequestError`](BadRequestError.md)
* [`InternalServerError`](InternalServerError.md)
* [`NotFoundError`](NotFoundError.md)
* [`PermissionDeniedError`](PermissionDeniedError.md)
* [`RateLimitError`](RateLimitError.md)
* [`UnprocessableEntityError`](UnprocessableEntityError.md)

## Constructors

### Constructor

```ts
new APIError(
   status, 
   body, 
   headers, 
   message?
): APIError;
```

#### Parameters

##### status

`number`

##### body

`unknown`

##### headers

`Headers`

##### message?

`string`

#### Returns

`APIError`

#### Overrides

[`TypeSafeError`](TypeSafeError.md).[`constructor`](TypeSafeError.md#sdk-constructor)

## Properties

### body

```ts
readonly body: unknown;
```

파싱된 JSON 또는 응답 텍스트입니다. 본문이 비어 있으면 `undefined`입니다.

***

### headers

```ts
readonly headers: Headers;
```

HTTP 응답 헤더입니다.

***

### requestId

```ts
readonly requestId: string | undefined;
```

`x-typesafe-request-id` 헤더에서 가져온 요청 ID입니다. 헤더가 없으면 `undefined`입니다.

***

### status

```ts
readonly status: number;
```

HTTP 응답 상태 코드입니다.

## Methods

### fromResponse()

```ts
static fromResponse(
   status, 
   body, 
   headers
): APIError;
```

HTTP 상태 코드에 맞는 오류 하위 클래스를 생성합니다.

#### Parameters

##### status

`number`

##### body

`unknown`

##### headers

`Headers`

#### Returns

`APIError`


이 문서는 개발자 문서 플랫폼인 [Mintlify](https://mintlify.com)로 제작하고 호스팅합니다.
