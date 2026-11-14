# Interface: NoulQuestion

예/아니요 질문이며, 두 결과 각각에 설명을 붙일 수 있습니다.

## Properties

### criteria?

```ts
optional criteria?: 
  | {
  false?: EntryType;
  true?: EntryType;
}
  | null;
```

예 결과와 아니요 결과에 대한 설명입니다. `instructions`를 생략하거나 `null`로 두었다면 둘 중 하나 이상은 반드시 있어야 합니다.

#### Union Members

##### Type Literal

```ts
{
  false?: EntryType;
  true?: EntryType;
}
```

##### false?

```ts
optional false?: EntryType;
```

아니요 결과에 대한 설명입니다.

##### true?

```ts
optional true?: EntryType;
```

예 결과에 대한 설명입니다.

***

`null`

***

### instructions?

```ts
optional instructions?: EntryType;
```

질문입니다. 텍스트, JSON 객체, 배열 중 하나로 지정하며, `criteria`가 결과를 설명하고 있다면 생략할 수 있습니다.

***

### type

```ts
type: "noul";
```


이 문서는 개발자 문서 플랫폼인 [Mintlify](https://mintlify.com)로 제작하고 호스팅합니다.
