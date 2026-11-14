# Type Alias: ResultFor&lt;T&gt;

```ts
type ResultFor<T> = T extends NoulQuestion ? NoulResponse : T extends ScoreQuestion<infer S> ? ScoreResponse<S> : T extends ChoiceQuestion<infer E> ? ChoiceResponse<E> : T extends BoxQuestion ? BoxResponse : never;
```

질문에 대응하는 답의 타입이며, 질문의 criteria(판단 기준) 키를 그대로 보존합니다.

## Type Parameters

### T

`T` *extends* [`Question`](Question.md)


이 문서는 개발자 문서 플랫폼인 [Mintlify](https://mintlify.com)로 제작하고 호스팅합니다.
