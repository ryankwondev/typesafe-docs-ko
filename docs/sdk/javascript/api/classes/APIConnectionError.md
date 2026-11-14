# Class: APIConnectionError

요청을 보내거나 응답 본문을 받는 데 실패했습니다(DNS, TLS, 연결 종료 등).

## Extends

* [`TypeSafeError`](TypeSafeError.md)

## Extended by

* [`APITimeoutError`](APITimeoutError.md)

## Constructors

### Constructor

```ts
new APIConnectionError(message?, options?): APIConnectionError;
```

#### Parameters

##### message?

`string` = `"Connection error."`

##### options?

`ErrorOptions`

#### Returns

`APIConnectionError`

#### Overrides

[`TypeSafeError`](TypeSafeError.md).[`constructor`](TypeSafeError.md#sdk-constructor)


이 문서는 개발자 문서 플랫폼인 [Mintlify](https://mintlify.com)로 제작하고 호스팅합니다.
