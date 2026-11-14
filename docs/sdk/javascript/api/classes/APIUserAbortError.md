# Class: APIUserAbortError

호출자가 `AbortSignal`로 요청을 취소했습니다.

## Extends

* [`TypeSafeError`](TypeSafeError.md)

## Constructors

### Constructor

```ts
new APIUserAbortError(message?, options?): APIUserAbortError;
```

#### Parameters

##### message?

`string` = `"Request was aborted."`

##### options?

`ErrorOptions`

#### Returns

`APIUserAbortError`

#### Overrides

[`TypeSafeError`](TypeSafeError.md).[`constructor`](TypeSafeError.md#sdk-constructor)


이 문서는 개발자 문서 플랫폼인 [Mintlify](https://mintlify.com)로 제작하고 호스팅합니다.
