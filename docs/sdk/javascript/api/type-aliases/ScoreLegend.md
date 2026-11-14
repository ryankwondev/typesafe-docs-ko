# Type Alias: ScoreLegend&lt;T&gt;

```ts
type ScoreLegend<T> = { readonly [score in ScoreOf<T>]: SentContent<T[score]> };
```

점수를 키로 하는 rubric(루브릭: 채점 기준) 설명입니다.

## Type Parameters

### T

`T` *extends* [`ScoreCriteria`](ScoreCriteria.md)


이 문서는 개발자 문서 플랫폼인 [Mintlify](https://mintlify.com)로 제작하고 호스팅합니다.
