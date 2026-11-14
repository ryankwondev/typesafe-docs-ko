# Interface: TextWithAssets

`withAssets` tagged template(태그드 템플릿)으로 만든 텍스트이며, 그 안에 보간된 asset(첨부 자산)을 함께 담습니다.

## Properties

### assets

```ts
readonly assets: readonly Image[];
```

텍스트에 보간된 asset이며, 중첩된 `withAssets` 텍스트 안의 asset도 포함합니다.

***

### text

```ts
readonly text: string;
```

각 asset을 해당 참조로 바꾼 텍스트입니다.

## Methods

### toJSON()

```ts
toJSON(): string;
```

텍스트입니다. 요청 안에서 이 값이 나타나는 모든 자리에 `JSON.stringify`가 이 텍스트를 씁니다.

#### Returns

`string`

***

### toString()

```ts
toString(): string;
```

#### Returns

`string`


이 문서는 개발자 문서 플랫폼인 [Mintlify](https://mintlify.com)로 제작하고 호스팅합니다.
