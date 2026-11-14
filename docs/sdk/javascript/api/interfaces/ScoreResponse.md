# Interface: ScoreResponse&lt;T&gt;

기대 점수와 함께 rubric(루브릭: 채점 기준), 점수별 확률을 담습니다.

## Type Parameters

### T

`T` *extends* [`ScoreCriteria`](../type-aliases/ScoreCriteria.md) = [`ScoreCriteria`](../type-aliases/ScoreCriteria.md)

## Properties

### confidence

```ts
readonly confidence: number;
```

점수에 대해 보고된 confidence(신뢰도)입니다.

***

### legend

```ts
readonly legend: ScoreLegend<T>;
```

점수를 키로 하는 rubric 설명입니다.

***

### probabilities

```ts
readonly probabilities: { readonly [score in number | `${number}`]: number };
```

점수를 키로 하는 확률입니다.

***

### score

```ts
readonly score: number;
```

기대 점수이며, rubric의 정수 단계 사이의 값이 될 수 있습니다.

***

### type

```ts
readonly type: "score";
```


이 문서는 개발자 문서 플랫폼인 [Mintlify](https://mintlify.com)로 제작하고 호스팅합니다.
