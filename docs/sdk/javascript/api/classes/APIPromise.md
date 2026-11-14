# Class: APIPromise&lt;T&gt;

파싱된 결과로 이행되는 promise이며, HTTP 응답에도 접근할 수 있습니다.

2xx가 아닌 응답은 `APIError`로 거부되며, `asResponse()`를 거치는 경우에도 마찬가지입니다.

## Extends

* `Promise`\<`T`>

## Type Parameters

### T

`T`

## Constructors

### Constructor

```ts
new APIPromise<T>(responsePromise, parseResponse): APIPromise<T>;
```

#### Parameters

##### responsePromise

`Promise`\<`Response`>

##### parseResponse

(`response`) => `Promise`\<`T`>

#### Returns

`APIPromise`\<`T`>

#### Overrides

```ts
Promise<T>.constructor
```

## Methods

### asResponse()

```ts
asResponse(): Promise<Response>;
```

본문을 파싱하지 않은 원본 `Response`로 이행됩니다. SDK 요청은 요청 타임아웃 안에서 본문 전체를
버퍼에 담은 뒤에 응답을 넘겨주며, 그 뒤에 본문을 읽는 일은 호출자가 맡습니다.
본문은 호출자의 것이므로, 같은 promise에서 파싱된 결과까지 `await`하지 마세요.

#### Returns

`Promise`\<`Response`>

***

### catch()

```ts
catch<TResult>(onrejected?): Promise<T | TResult>;
```

Promise가 거부될 때만 호출되는 콜백을 등록합니다.

#### Type Parameters

##### TResult

`TResult` = `never`

#### Parameters

##### onrejected?

((`reason`) => `TResult` | `PromiseLike`\<`TResult`>) | `null`

Promise가 거부될 때 실행할 콜백입니다.

#### Returns

`Promise`\<`T` | `TResult`>

콜백이 완료되면 처리되는 Promise입니다.

#### Overrides

```ts
Promise.catch
```

***

### finally()

```ts
finally(onfinally?): Promise<T>;
```

Promise가 처리 완료(이행 또는 거부)될 때 호출되는 콜백을 등록합니다. 콜백에서는
이행된 값을 바꿀 수 없습니다.

#### Parameters

##### onfinally?

(() => `void`) | `null`

Promise가 처리 완료(이행 또는 거부)될 때 실행할 콜백입니다.

#### Returns

`Promise`\<`T`>

콜백이 완료되면 처리되는 Promise입니다.

#### Overrides

```ts
Promise.finally
```

***

### map()

```ts
map<U>(fn): APIPromise<U>;
```

파싱된 결과를 변환합니다. HTTP 응답을 공유하며 본문 파싱도 한 번만 수행합니다.

#### Type Parameters

##### U

`U`

#### Parameters

##### fn

(`data`) => `U`

#### Returns

`APIPromise`\<`U`>

***

### then()

```ts
then<TResult1, TResult2>(onfulfilled?, onrejected?): Promise<TResult1 | TResult2>;
```

Promise의 이행이나 거부, 또는 둘 모두에 대한 콜백을 등록합니다.

#### Type Parameters

##### TResult1

`TResult1` = `T`

##### TResult2

`TResult2` = `never`

#### Parameters

##### onfulfilled?

((`value`) => `TResult1` | `PromiseLike`\<`TResult1`>) | `null`

Promise가 이행될 때 실행할 콜백입니다.

##### onrejected?

((`reason`) => `TResult2` | `PromiseLike`\<`TResult2`>) | `null`

Promise가 거부될 때 실행할 콜백입니다.

#### Returns

`Promise`\<`TResult1` | `TResult2`>

두 콜백 중 실행된 쪽이 완료되면 처리되는 Promise입니다.

#### Overrides

```ts
Promise.then
```

***

### withResponse()

```ts
withResponse(): Promise<WithResponse<T>>;
```

파싱된 결과, HTTP 응답, 요청 ID를 함께 반환합니다.

#### Returns

`Promise`\<[`WithResponse`](../interfaces/WithResponse.md)\<`T`>>


이 문서는 개발자 문서 플랫폼인 [Mintlify](https://mintlify.com)로 제작하고 호스팅합니다.
