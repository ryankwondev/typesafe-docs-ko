# Interface: BoxResponse

`box` 질문에 대한 답입니다.

## Properties

### box

```ts
readonly box: BoxCoordinates;
```

instructions(지시문)가 가리키는 대상이 이미지 안에서 차지하는 위치입니다.

***

### confidence

```ts
readonly confidence: number | null;
```

box에 대한 confidence(신뢰도)이며 0에서 1 사이입니다. API가 값을 알려 주지 않으면 `null`입니다.

***

### type

```ts
readonly type: "box";
```


이 문서는 개발자 문서 플랫폼인 [Mintlify](https://mintlify.com)로 제작하고 호스팅합니다.
