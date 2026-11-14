# Type Alias: ContentValue

```ts
type ContentValue = 
  | string
  | number
  | boolean
  | null
  | Asset
  | TextWithAssets
  | ContentValue[]
  | {
[key: string]: ContentValue;
};
```

JSON과 호환되는 값이며, asset(첨부 자산)도 담을 수 있습니다. 각 asset은 `assets`에서 정한 이름으로 쓰이고,
이름이 없으면 자신의 참조로 쓰입니다.


이 문서는 개발자 문서 플랫폼인 [Mintlify](https://mintlify.com)로 제작하고 호스팅합니다.
