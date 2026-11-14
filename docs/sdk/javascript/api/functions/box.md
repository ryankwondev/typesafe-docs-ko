# Function: box()

```ts
function box(instructions?): BoxQuestion;
```

instructions(지시문)가 설명하는 대상을 요청의 이미지에서 찾는 질문을 만듭니다.
`box` 질문에는 `jev-preview` 모델을 사용하세요.

## Parameters

### instructions?

[`EntryType`](../type-aliases/EntryType.md) = `null`

찾을 대상입니다. 텍스트, asset(첨부 자산), JSON 객체나 배열로 지정하며, 기본값은 `null`입니다.

## Returns

[`BoxQuestion`](../interfaces/BoxQuestion.md)

## Example

```ts
const { answers } = await client.systemOne({
  state: image(await openAsBlob("photo.jpg")),
  questions: { dog: box("The dog") },
  model: "jev-preview",
});
const { top, bottom, left, right } = answers.dog.box;
```


이 문서는 개발자 문서 플랫폼인 [Mintlify](https://mintlify.com)로 제작하고 호스팅합니다.
