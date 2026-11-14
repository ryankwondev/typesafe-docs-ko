# Class: TypeSafeError

SDK 오류의 기본 클래스입니다.

## Extends

* `Error`

## Extended by

* [`APIConnectionError`](APIConnectionError.md)
* [`APIError`](APIError.md)
* [`APIUserAbortError`](APIUserAbortError.md)

## Constructors

### Constructor

```ts
new TypeSafeError(message, options?): TypeSafeError;
```

#### Parameters

##### message

`string`

##### options?

`ErrorOptions`

#### Returns

`TypeSafeError`

#### Overrides

```ts
Error.constructor
```


이 문서는 개발자 문서 플랫폼인 [Mintlify](https://mintlify.com)로 제작하고 호스팅합니다.
