# Class: Image

PNG 또는 JPEG 이미지 asset(첨부 자산)이며, `image`나 `Image.fromBase64`로 만듭니다.

## Constructors

### Constructor

```ts
new Image(data): Image;
```

#### Parameters

##### data

`ArrayBuffer` | `ArrayBufferView`\<`ArrayBufferLike`> | `Blob`

#### Returns

`Image`

#### Throws

`data`가 비어 있거나, 파일 바이트도 `Blob`도 아닌 경우.

## Properties

### data

```ts
readonly data: Uint8Array<ArrayBufferLike> | Blob;
```

이미지 파일 바이트, 또는 요청을 보낼 때 읽어 들이는 `Blob`입니다.

***

### type

```ts
readonly type: "image" = "image";
```

## Methods

### toJSON()

```ts
toJSON(): string;
```

참조 문자열입니다. 요청 안에서 이미지가 나타나는 모든 자리에 `JSON.stringify`가 이 값을 씁니다.

#### Returns

`string`

***

### toString()

```ts
toString(): string;
```

이 이미지를 전송할 때 붙는 고유한 참조 문자열이며, 텍스트 안에서 이미지를 참조할 때 씁니다.

#### Returns

`string`

***

### fromBase64()

```ts
static fromBase64(text): Image;
```

base64로 인코딩된 파일 바이트로 이미지를 만듭니다. 디코딩은 호출 즉시 이루어집니다.

#### Parameters

##### text

`string`

base64로 인코딩된 이미지 파일 바이트입니다. `data:` URL 접두어는 붙이지 않습니다.

#### Returns

`Image`

#### Throws

`text`가 유효한 base64가 아니거나, 디코딩한 결과에 바이트가 하나도 없는 경우.

#### Example

```ts
const photo = Image.fromBase64(upload.photoBase64);
```


이 문서는 개발자 문서 플랫폼인 [Mintlify](https://mintlify.com)로 제작하고 호스팅합니다.
