# Interface: SystemOneRequest&lt;Q&gt;

`systemOne`에 넘기는 state(상태)와 이름을 붙인 질문들입니다.

요청 변수에 들어 있는 그 밖의 프로퍼티도 그대로 전달되며, 값이 `null`인 것도 포함됩니다.

## Type Parameters

### Q

`Q` *extends* [`Questions`](Questions.md) = [`Questions`](Questions.md)

## Properties

### assets?

```ts
optional assets?: Record<string, Image>;
```

참조를 asset(첨부 자산)에 대응시킨 맵이며, asset을 이름으로 참조할 때 씁니다.

참조는 `state`, 질문의 instructions(지시문), criteria(판단 기준) 안의 문자열 값에서
글자 그대로 일치하는 부분을 찾습니다. 일반 텍스트와 겹치지 않도록 `<photo>`처럼 눈에 띄는 구분자를 사용하세요.
`state`나 질문 콘텐츠에 직접 넣은 asset, 또는 `withAssets` 텍스트에 넣은 asset은
여기에 나열하지 않아도 전송됩니다. 모든 이름은 요청 안에 나타나야 합니다. 그렇지 않으면 요청을 보내기 전에
호출이 거부됩니다.

***

### model?

```ts
optional model?: string;
```

모델 재정의 값입니다. 생략하면 `defaultModel`을 따릅니다.

***

### questions

```ts
questions: Q;
```

답을 식별하는 데 쓰는 이름을 키로 하는 질문 모음이며, 비어 있으면 안 됩니다.

***

### state

```ts
state: NonNullable<EntryType>;
```

평가할 대상입니다. 텍스트, asset, JSON 객체, 배열 중 하나입니다.


이 문서는 개발자 문서 플랫폼인 [Mintlify](https://mintlify.com)로 제작하고 호스팅합니다.
