# Interface: ChoiceQuestion&lt;T&gt;

이름이 붙은 선택지 가운데 하나를 고르는 질문입니다.

## Type Parameters

### T

`T` *extends* [`ChoiceCriteria`](../type-aliases/ChoiceCriteria.md) = [`ChoiceCriteria`](../type-aliases/ChoiceCriteria.md)

## Properties

### criteria

```ts
criteria: T;
```

선택할 수 있는 결과들에 대한 설명입니다. 레이블이 하나 이상 있어야 합니다.

***

### instructions?

```ts
optional instructions?: EntryType;
```

질문입니다. 텍스트, JSON 객체, 배열 중 하나로 지정하며, 생략하거나 `null`로 둘 수 있습니다.

***

### type

```ts
type: "choice";
```


이 문서는 개발자 문서 플랫폼인 [Mintlify](https://mintlify.com)로 제작하고 호스팅합니다.
