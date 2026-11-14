# Variable: noul

```ts
const noul: {
  (instructions, criteria?): NoulQuestion;
  (instructions, criteria): NoulQuestion;
};
```

예/아니요 질문을 만듭니다. 두 결과 각각에 설명을 붙일 수 있습니다.

## Call Signature

```ts
(instructions, criteria?): NoulQuestion;
```

### Parameters

#### instructions

`NonNullable`\<[`EntryType`](../type-aliases/EntryType.md)>

#### criteria?

\| \{
`false?`: [`EntryType`](../type-aliases/EntryType.md);
`true?`: [`EntryType`](../type-aliases/EntryType.md);
}
\| `null`

##### Type Literal

\{
`false?`: [`EntryType`](../type-aliases/EntryType.md);
`true?`: [`EntryType`](../type-aliases/EntryType.md);
}

###### false?

[`EntryType`](../type-aliases/EntryType.md)

아니요 결과에 대한 설명입니다.

###### true?

[`EntryType`](../type-aliases/EntryType.md)

예 결과에 대한 설명입니다.

***

`null`

### Returns

[`NoulQuestion`](../interfaces/NoulQuestion.md)

## Call Signature

```ts
(instructions, criteria): NoulQuestion;
```

### Parameters

#### instructions

[`EntryType`](../type-aliases/EntryType.md) | `undefined`

#### criteria

\| \{
`false?`: [`EntryType`](../type-aliases/EntryType.md);
`true`: `NonNullable`\<[`EntryType`](../type-aliases/EntryType.md)>;
}
\| \{
`false`: `NonNullable`\<[`EntryType`](../type-aliases/EntryType.md)>;
`true?`: [`EntryType`](../type-aliases/EntryType.md);
}

### Returns

[`NoulQuestion`](../interfaces/NoulQuestion.md)

## Param

**instructions**

질문입니다. 텍스트, JSON 객체나 배열로 지정합니다. `null`이거나 생략한 경우에는
`criteria`가 결과를 하나 이상 설명해야 합니다.

## Param

**criteria**

예 결과와 아니요 결과에 대한 설명입니다.


이 문서는 개발자 문서 플랫폼인 [Mintlify](https://mintlify.com)로 제작하고 호스팅합니다.
