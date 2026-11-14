# Function: image()

```ts
function image(data): Image;
```

`systemOne`에 쓸 이미지 asset(첨부 자산)을 만듭니다.

이미지는 `state`(상태)나 질문 콘텐츠에 넣거나, `withAssets`
tagged template(태그드 템플릿)에 보간합니다. 어느 쪽이든 이미지는 자신의 참조인 `String(image)`로 전송됩니다.
직접 정한 이름으로 참조하려면 대신 `assets`에 넣어 넘깁니다.

이미지는 PNG 또는 JPEG여야 합니다. API는 파일 바이트를 보고 형식을 판별하며,
다른 형식이면 `BadRequestError`로 거부합니다. `Blob` 데이터는 요청을 보낼 때 읽으며,
`Blob`의 `type`은 무시합니다. base64 텍스트라면 `Image.fromBase64`를 사용하세요.

## Parameters

### data

`ArrayBuffer` | `ArrayBufferView`\<`ArrayBufferLike`> | `Blob`

이미지 파일 바이트, 또는 `File` 같은 `Blob`입니다.

## Returns

[`Image`](../classes/Image.md)

## Throws

`data`가 비어 있거나, 파일 바이트도 `Blob`도 아닌 경우.

## Example

```ts
// Node: `openAsBlob`은 파일을 Blob으로 읽으며, "node:fs"에서 export됩니다.
const { answers } = await client.systemOne({
  state: image(await openAsBlob("product.png")),
  questions: { damaged: noul("Is the product damaged?") },
});
```


이 문서는 개발자 문서 플랫폼인 [Mintlify](https://mintlify.com)로 제작하고 호스팅합니다.
