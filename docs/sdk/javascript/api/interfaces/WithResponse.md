# Interface: WithResponse&lt;T&gt;

파싱된 데이터와 함께 HTTP 응답, 요청 ID를 담습니다.

## Type Parameters

### T

`T`

## Properties

### data

```ts
data: T;
```

파싱된 응답 본문입니다.

***

### requestId

```ts
requestId: string | undefined;
```

`x-typesafe-request-id` 헤더에서 가져온 요청 ID입니다. 헤더가 없으면 `undefined`입니다.

***

### response

```ts
response: Response;
```

HTTP 응답이며, 본문은 파싱 과정에서 이미 소비된 상태입니다.


이 문서는 개발자 문서 플랫폼인 [Mintlify](https://mintlify.com)로 제작하고 호스팅합니다.
