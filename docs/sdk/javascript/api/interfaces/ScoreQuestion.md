# Interface: ScoreQuestion&lt;T&gt;

순서가 있는 rubric(루브릭: 채점 기준)에 따라 점수를 매기는 질문입니다.

## Type Parameters

### T

`T` *extends* [`ScoreCriteria`](../type-aliases/ScoreCriteria.md) = [`ScoreCriteria`](../type-aliases/ScoreCriteria.md)

## Properties

### criteria

```ts
criteria: T;
```

선택할 수 있는 결과들에 대한 설명입니다.

***

### instructions?

```ts
optional instructions?: EntryType;
```

질문입니다. 텍스트, JSON 객체, 배열 중 하나로 지정하며, 생략하거나 `null`로 둘 수 있습니다.

***

### type

```ts
type: "score";
```


이 문서는 개발자 문서 플랫폼인 [Mintlify](https://mintlify.com)로 제작하고 호스팅합니다.
