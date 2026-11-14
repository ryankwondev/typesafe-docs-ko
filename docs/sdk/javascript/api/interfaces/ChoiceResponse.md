# Interface: ChoiceResponse&lt;T&gt;

선택된 레이블과 레이블별 확률입니다.

## Type Parameters

### T

`T` *extends* [`ChoiceCriteria`](../type-aliases/ChoiceCriteria.md) = [`ChoiceCriteria`](../type-aliases/ChoiceCriteria.md)

## Properties

### choice

```ts
readonly choice: keyof T & string;
```

선택된 레이블입니다.

***

### confidence

```ts
readonly confidence: number;
```

선택된 레이블에 대해 보고된 confidence(신뢰도)입니다.

***

### probabilities

```ts
readonly probabilities: { readonly [label in string | number | symbol]: number };
```

레이블을 키로 하는 확률입니다.

***

### type

```ts
readonly type: "choice";
```


이 문서는 개발자 문서 플랫폼인 [Mintlify](https://mintlify.com)로 제작하고 호스팅합니다.
