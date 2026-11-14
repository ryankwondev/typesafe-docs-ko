# Variable: withAssets

```ts
const withAssets: {
(strings, ...values): TextWithAssets;
  join: TextWithAssets;
};
```

template literal(템플릿 리터럴)에 붙이는 태그이며, 그 안에 보간된 asset(첨부 자산)이 텍스트와 함께 전송되도록 합니다.

각 asset은 자신의 참조로 바뀌고, 그 밖의 값은 태그가 없는 template literal에서와 마찬가지로
문자열로 변환됩니다. asset 목록을 구분자와 함께 쓰려면
`withAssets.join`을 사용하세요.

## Type Declaration

## Parameters

### strings

`TemplateStringsArray`

### values

...readonly `unknown`\[]

## Returns

[`TextWithAssets`](../interfaces/TextWithAssets.md)

### join()

```ts
join(values, separator?): TextWithAssets;
```

`Array.prototype.join`처럼 `values`를 `separator`로 이어 붙이고, 그 안에 있는 asset을
텍스트와 함께 전송합니다. 결과는 콘텐츠로 넘기거나 `withAssets` 안에 보간할 수 있습니다.

#### Parameters

##### values

readonly `unknown`\[]

##### separator?

`string`

값 사이에 쓰이는 문자열입니다. 기본값: `","`.

#### Returns

[`TextWithAssets`](../interfaces/TextWithAssets.md)

#### Example

```ts
const { answers } = await client.systemOne({
  state: withAssets`The seller uploaded ${withAssets.join(uploads, ", ")}.`,
  questions: { same: noul("Do the uploads show the same product?") },
});
```

## Example

```ts
const { answers } = await client.systemOne({
  state: withAssets`Compare ${photo} with the listing.`,
  questions: { same: noul(withAssets`Is ${photo} the product in ${listing}?`) },
});
```


이 문서는 개발자 문서 플랫폼인 [Mintlify](https://mintlify.com)로 제작하고 호스팅합니다.
