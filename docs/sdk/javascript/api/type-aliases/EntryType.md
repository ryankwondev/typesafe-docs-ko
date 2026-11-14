# Type Alias: EntryType

```ts
type EntryType = 
  | string
  | Asset
  | TextWithAssets
  | {
[key: string]: ContentValue;
}
  | ContentValue[]
  | null;
```

instructions(지시문)와 criteria(판단 기준)에 쓰는 값으로, 텍스트, asset(첨부 자산), JSON 객체나 배열, 또는 `null`입니다.

그 안의 각 asset은 `assets`에서 정한 이름으로 쓰이고, 이름이 없으면 자신의 참조로 쓰이며,
요청과 함께 전송됩니다.


이 문서는 개발자 문서 플랫폼인 [Mintlify](https://mintlify.com)로 제작하고 호스팅합니다.
