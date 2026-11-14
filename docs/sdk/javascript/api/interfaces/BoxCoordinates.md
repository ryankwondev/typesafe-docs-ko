# Interface: BoxCoordinates

box의 각 변을 이미지 높이에 대한 비율(`top`, `bottom`)과 너비에 대한 비율(`left`, `right`)로 나타냅니다.

## Properties

### bottom

```ts
readonly bottom: number;
```

아래쪽 변의 위치입니다. 이미지 위쪽 끝에서부터 잰 거리를 이미지 높이에 대한 비율로 나타내며, 0에서 1 사이입니다.

***

### left

```ts
readonly left: number;
```

왼쪽 변의 위치입니다. 이미지 왼쪽 끝에서부터 잰 거리를 이미지 너비에 대한 비율로 나타내며, 0에서 1 사이입니다.

***

### right

```ts
readonly right: number;
```

오른쪽 변의 위치입니다. 이미지 왼쪽 끝에서부터 잰 거리를 이미지 너비에 대한 비율로 나타내며, 0에서 1 사이입니다.

***

### top

```ts
readonly top: number;
```

위쪽 변의 위치입니다. 이미지 위쪽 끝에서부터 잰 거리를 이미지 높이에 대한 비율로 나타내며, 0에서 1 사이입니다.


이 문서는 개발자 문서 플랫폼인 [Mintlify](https://mintlify.com)로 제작하고 호스팅합니다.
