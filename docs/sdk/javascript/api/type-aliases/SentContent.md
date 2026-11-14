# Type Alias: SentContent&lt;T&gt;

```ts
type SentContent<T> = T extends Asset | TextWithAssets ? string : T extends object ? { readonly [key in keyof T]: SentContent<T[key]> } : T;
```

API가 돌려주는 형태의 콘텐츠이며, asset(첨부 자산)과 `withAssets` 텍스트는 문자열로 쓰여 있습니다.

## Type Parameters

### T

`T`


이 문서는 개발자 문서 플랫폼인 [Mintlify](https://mintlify.com)로 제작하고 호스팅합니다.
