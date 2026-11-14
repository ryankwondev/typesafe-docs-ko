# Class: PermissionDeniedError

HTTP 403: 접근이 거부되었습니다.

## Extends

* [`APIError`](APIError.md)

## Constructors

### Constructor

```ts
new PermissionDeniedError(
   status, 
   body, 
   headers, 
   message?
): PermissionDeniedError;
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

`PermissionDeniedError`

#### Inherited from

[`APIError`](APIError.md).[`constructor`](APIError.md#sdk-constructor)

## Properties

### body

```ts
readonly body: unknown;
```

파싱된 JSON 또는 응답 텍스트입니다. 본문이 비어 있으면 `undefined`입니다.

#### Inherited from

[`APIError`](APIError.md).[`body`](APIError.md#sdk-body)

***

### headers

```ts
readonly headers: Headers;
```

HTTP 응답 헤더입니다.

#### Inherited from

[`APIError`](APIError.md).[`headers`](APIError.md#sdk-headers)

***

### requestId

```ts
readonly requestId: string | undefined;
```

`x-typesafe-request-id` 헤더에서 가져온 요청 ID입니다. 헤더가 없으면 `undefined`입니다.

#### Inherited from

[`APIError`](APIError.md).[`requestId`](APIError.md#sdk-requestid)

***

### status

```ts
readonly status: number;
```

HTTP 응답 상태 코드입니다.

#### Inherited from

[`APIError`](APIError.md).[`status`](APIError.md#sdk-status)

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

[`APIError`](APIError.md)

#### Inherited from

[`APIError`](APIError.md).[`fromResponse`](APIError.md#sdk-fromresponse)


이 문서는 개발자 문서 플랫폼인 [Mintlify](https://mintlify.com)로 제작하고 호스팅합니다.
