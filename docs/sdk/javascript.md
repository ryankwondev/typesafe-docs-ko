# JavaScript SDK

[TypeSafe AI](https://typesafe.ai)를 JavaScript와 TypeScript에서 사용하기 위한 SDK입니다.

## 빠른 시작

SDK를 설치합니다(Node.js 22 이상 필요).

```sh
npm install @typesafe-ai/sdk
```

환경 변수 `TYPESAFE_API_KEY`를 설정한 다음, 클라이언트를 만들어 사용합니다.

```ts
import { choice, TypeSafeClient } from "@typesafe-ai/sdk";

const client = new TypeSafeClient();
const response = await client.systemOne({
  state: { ticket: "I was charged twice. Please fix this ASAP." },
  questions: {
    category: choice("What is this ticket about?", {
      billing: null,
      technical: null,
      other: null,
    }),
  },
});

console.log(response.answers.category.choice);
```

답의 타입은 질문에서 추론됩니다. 패키지에는 ESM, CommonJS, TypeScript 선언 파일이 모두 들어 있습니다.

## 이미지

이미지는 파일 바이트나 `File` 같은 `Blob`을 `image()`에 넘겨서 만들거나, base64 텍스트를
`Image.fromBase64()`에 넘겨서 만듭니다. 이미지는 PNG 또는 JPEG여야 합니다. API는 파일 바이트를 보고
형식을 판별하며, WebP나 GIF 같은 다른 형식이면 `BadRequestError`로 거부합니다.
요청에서 이미지를 사용하는 방법은 세 가지이며, 함께 섞어 쓸 수 있습니다.

이미지 지원은 실험적 기능입니다. 사용하려면 `TypeSafe-Danger-Zone-Images: true`
헤더를 보내야 합니다.

```ts
const client = new TypeSafeClient({
  defaultHeaders: { "TypeSafe-Danger-Zone-Images": "true" },
});
```

요청 하나에만 이 헤더를 보내려면 해당 호출의 `headers` 옵션에 넣습니다.

### 콘텐츠 안에 이미지 넣기

이미지는 `state`(상태), 질문의 instructions(지시문), criteria(판단 기준) 어디에나 넣을 수 있습니다.

```ts
import { openAsBlob } from "node:fs";
import { image, noul, score, TypeSafeClient } from "@typesafe-ai/sdk";

const client = new TypeSafeClient();
const photo = image(await openAsBlob("returns/RMA-1042.jpg"));

const { answers } = await client.systemOne({
  state: { claim: "The mug arrived with a chipped rim.", photo },
  questions: {
    damaged: noul("Is the product in the photo damaged?"),
    severity: score("How severe is the damage?", ["none", "cosmetic", "unusable"]),
  },
});
console.log(answers.damaged.noul, answers.severity.score);
```

### `withAssets`로 텍스트 안에 이미지 쓰기

template literal(템플릿 리터럴)에 `withAssets` 태그를 붙이면 텍스트 안에서 이미지를 참조할 수 있습니다. 아래 라우트
핸들러는 판매자가 올린 사진을 카탈로그 사진과 대조합니다.

```ts
import { choice, image, noul, TypeSafeClient, withAssets } from "@typesafe-ai/sdk";
import { catalogPhotoUrl } from "./catalog";

const client = new TypeSafeClient();

export async function POST(request: Request): Promise<Response> {
  const form = await request.formData();
  const upload = image(form.get("photo") as File);
  const catalog = image(await (await fetch(catalogPhotoUrl(String(form.get("sku"))))).blob());

  const { answers } = await client.systemOne({
    state: withAssets`A seller uploaded ${upload} for a listing whose catalog photo is ${catalog}.`,
    questions: {
      sameProduct: noul(withAssets`Does ${upload} show the same product as ${catalog}?`),
      quality: choice(withAssets`Is ${upload} usable as a listing photo?`, {
        usable: null,
        blurry: null,
        cropped: null,
        notAProduct: null,
      }),
    },
  });
  return Response.json({ sameProduct: answers.sameProduct.noul, quality: answers.quality.choice });
}
```

이미지는 몇 번을 쓰든 한 번만 전송됩니다. `` `Compare ${upload}` ``처럼 태그가 없는 template
literal에 이미지를 넣으면 요청을 보내기 전에 호출이 `TypeSafeError`로
거부됩니다.

이미지 목록을 텍스트에 쓰려면 `withAssets.join`으로 이어 붙입니다. 동작은 `Array.prototype.join`과 같습니다.

```ts
const uploads = (form.getAll("photos") as File[]).map((file) => image(file));
const state = withAssets`The seller uploaded ${withAssets.join(uploads, ", ")}.`;
```

### `assets`로 참조 이름 직접 정하기

이미지를 `assets`에 넣고 원하는 참조 이름을 키로 지정하면, 설정에 저장해 둔 prompt(프롬프트)처럼 평범한
문자열에서도 이미지를 참조할 수 있습니다.

```ts
import { readFile } from "node:fs/promises";
import { image, noul, TypeSafeClient } from "@typesafe-ai/sdk";

const client = new TypeSafeClient();

const { answers } = await client.systemOne({
  state: "A customer returned <delivered>. The catalog photo is <catalog>.",
  questions: { matches: noul("Is <delivered> the product shown in <catalog>?") },
  assets: {
    "<delivered>": image(await readFile("returns/RMA-1042.jpg")),
    "<catalog>": image(await readFile("catalog/mug.png")),
  },
});
console.log(answers.matches.noul);
```

참조가 일반 텍스트와 겹쳐서 일치하지 않도록 `<delivered>`처럼 눈에 띄는 구분자를 사용하세요.
`assets`에 넣은 이미지를 콘텐츠나 `withAssets` 텍스트에도 넣으면, 그 자리에는 `assets`에서 정한
이름이 쓰입니다. `assets`의 모든 이름은 요청 안에 나타나야 합니다. 그렇지 않으면 요청을 보내기 전에 호출이
`TypeSafeError`로 거부됩니다.

### `box`로 위치 찾기

`box` 질문을 던지면 instructions가 설명하는 대상의 위치를 찾습니다. 답의 `box`에는 각 변의 위치가
이미지 높이에 대한 비율(`top`, `bottom`)과 너비에 대한 비율(`left`, `right`)로 들어 있으며,
API가 confidence(신뢰도)를 알려 주지 않으면 `confidence`는 `null`입니다. `box` 질문에는 `jev-preview` 모델을
사용하세요.

```ts
import { openAsBlob } from "node:fs";
import { box, image, TypeSafeClient } from "@typesafe-ai/sdk";

const client = new TypeSafeClient();

const { answers } = await client.systemOne({
  state: image(await openAsBlob("returns/RMA-1042.jpg")),
  questions: { damage: box("The damaged area of the product") },
  model: "jev-preview",
});
const { top, bottom, left, right } = answers.damage.box;
```

이미지가 포함된 요청은 `multipart/form-data`로 전송되며, 각 이미지의 파일 바이트는 별도의
파트에 담깁니다.

## 요청 압축

클라이언트는 1 KiB 이상인 요청 본문을 zstd로 압축합니다. 업로드 크기와 시간이 줄어들며,
state가 클수록 효과가 큽니다. 그보다 작은 본문은 TCP 세그먼트 하나에 들어가므로 압축하지 않고
보냅니다. zstd를 쓰려면 Node.js 22.15 이상이 필요하며, 그 밖의 런타임에서는 모든 본문을 압축하지 않고
보냅니다. 압축된 요청을 받지 않는 API로 보낼 때처럼 모든 본문을 압축 없이 보내야 한다면
`compression: false`를 넘깁니다.

```ts
const client = new TypeSafeClient({ compression: false });
```

## 문서

TypeSafe로 할 수 있는 일은 [TypeSafe 문서](../index.md)에서 확인하세요.
API 옵션과 기본값은 SDK의 [client](https://github.com/typesafe-ai/typesafe-sdk-js/blob/v0.7.0/src/client.ts)와 [types](https://github.com/typesafe-ai/typesafe-sdk-js/blob/v0.7.0/src/types.ts) 소스를 참고하세요.


이 문서는 개발자 문서 플랫폼인 [Mintlify](https://mintlify.com)로 제작하고 호스팅합니다.
