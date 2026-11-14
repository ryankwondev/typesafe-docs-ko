# Function: choice()

```ts
function choice<T>(instructions, criteria): ChoiceQuestion<T>;
```

이름이 붙은 선택지 가운데 하나를 고르는 질문을 만듭니다.

## Type Parameters

### T

`T` *extends* [`ChoiceCriteria`](../type-aliases/ChoiceCriteria.md)

## Parameters

### instructions

[`EntryType`](../type-aliases/EntryType.md)

질문입니다. 텍스트, JSON 객체나 배열, 또는 `null`로 지정합니다.

### criteria

`T`

레이블을 설명에 대응시킨 맵입니다. 설명이 없는 레이블은 `null`로 둡니다.

## Returns

[`ChoiceQuestion`](../interfaces/ChoiceQuestion.md)\<`T`>


이 문서는 개발자 문서 플랫폼인 [Mintlify](https://mintlify.com)로 제작하고 호스팅합니다.
