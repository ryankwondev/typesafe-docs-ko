# Type Alias: ScoreOf&lt;T&gt;

```ts
type ScoreOf<T> = number extends T["length"] ? number : Extract<keyof T, `${number}`>;
```

rubric(루브릭: 채점 기준)에서 추론한 점수 키입니다. 길이가 고정된 튜플이면 그 인덱스들이 되고, 그렇지 않으면 `number`가 됩니다.

## Type Parameters

### T

`T` *extends* [`ScoreCriteria`](ScoreCriteria.md)


이 문서는 개발자 문서 플랫폼인 [Mintlify](https://mintlify.com)로 제작하고 호스팅합니다.
