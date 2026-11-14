# Variable: score

```ts
const score: {
<T>  (instructions, criteria): ScoreQuestion<T>;
<T>  (instructions, criteria): ScoreQuestion<readonly [T[number], T[number], T[number]]>;
};
```

순서가 있는 rubric(루브릭: 채점 기준)을 사용하는 score 질문을 만듭니다.

## Call Signature

```ts
<T>(instructions, criteria): ScoreQuestion<T>;
```

### Type Parameters

#### T

`T` *extends* [`ScoreCriteria`](../type-aliases/ScoreCriteria.md)

### Parameters

#### instructions

[`EntryType`](../type-aliases/EntryType.md)

#### criteria

`T`

### Returns

[`ScoreQuestion`](../interfaces/ScoreQuestion.md)\<`T`>

## Call Signature

```ts
<T>(instructions, criteria): ScoreQuestion<readonly [T[number], T[number], T[number]]>;
```

### Type Parameters

#### T

`T` *extends* readonly [`EntryType`](../type-aliases/EntryType.md)\[]

### Parameters

#### instructions

[`EntryType`](../type-aliases/EntryType.md)

#### criteria

`T` & `number` *extends* `T`\[`"length"`] ? `unknown` : `never`

### Returns

[`ScoreQuestion`](../interfaces/ScoreQuestion.md)\<readonly \[`T`\[`number`], `T`\[`number`], `T`\[`number`]]>

## Param

**instructions**

질문입니다. 텍스트, JSON 객체나 배열, 또는 `null`로 지정합니다.

## Param

**criteria**

0부터 시작하는 점수를 인덱스로 하는 설명이며, 두 개 이상이어야 합니다. 항목은 `null`이어도 됩니다.
`string[]`처럼 런타임에 만든 목록도 받지만, 항목이 두 개 미만이면 `systemOne`이
거부합니다.


이 문서는 개발자 문서 플랫폼인 [Mintlify](https://mintlify.com)로 제작하고 호스팅합니다.
