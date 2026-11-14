# Class: APITimeoutError

타임아웃 안에 응답 전체가 도착하지 않았습니다. `APIConnectionError`의 한 종류입니다.

## Extends

* [`APIConnectionError`](APIConnectionError.md)

## Constructors

### Constructor

```ts
new APITimeoutError(timeoutMs, options?): APITimeoutError;
```

#### Parameters

##### timeoutMs

`number`

##### options?

`ErrorOptions`

#### Returns

`APITimeoutError`

#### Overrides

[`APIConnectionError`](APIConnectionError.md).[`constructor`](APIConnectionError.md#sdk-constructor)

## Properties

### timeoutMs

```ts
readonly timeoutMs: number;
```

설정된 타임아웃(밀리초)입니다.


이 문서는 개발자 문서 플랫폼인 [Mintlify](https://mintlify.com)로 제작하고 호스팅합니다.
