# 변경 이력

## v0.7.0 (2026-10-14)

### 호환성이 깨지는 변경

* Node.js 22 이상이 필요합니다.
* 요청 페이로드 타입을 더 이상 export하지 않습니다.

### 새 기능

* 이미지를 지원합니다. `TypeSafe-Danger-Zone-Images: true` 헤더로 켜는 실험적 기능입니다. `image()`나 `Image.fromBase64()`로 이미지를 만든 다음 `state`(상태), 질문의 instructions(지시문), criteria(판단 기준)에 넣거나, `withAssets`와 `withAssets.join`으로 텍스트 안에 쓰거나, `assets`에 직접 정한 이름으로 넣어 참조할 수 있습니다.
* `box` 질문을 추가했습니다. instructions가 설명하는 대상을 이미지에서 찾아 주며, `jev-preview` 모델과 함께 사용합니다.
* Node.js 22.15 이상에서 1 KiB 이상인 요청 본문을 zstd로 압축합니다. 압축 없이 보내려면 `compression: false`를 넘깁니다.

### 버그 수정

* usage의 token(토큰) 수가 null이어도 허용합니다.
* 요청 유효성 검사 항목을 추가했습니다.
* 요청이 유효하지 않으면 예외를 던지는 대신 반환한 promise를 거부합니다.
* 런타임에 만든 score criteria 목록을 허용합니다.
* Node.js 22에서 요청이 본문 전송 도중 타임아웃되거나 중단될 때 처리되지 않은 거부(unhandled rejection)가 발생하던 문제를 고쳤습니다.

## v0.6.0 (2026-09-15)

### 호환성이 깨지는 변경

* `Score.criteria`를 정수 키 딕셔너리 대신 순서가 있는 시퀀스로 받습니다.

## v0.5.7 (2026-09-11)

TypeSafe JavaScript 및 TypeScript SDK의 첫 공개 릴리스입니다. 자세한 내용은 [문서](https://docs.typesafe.ai/go/js-sdk.md)를 참고하세요.


이 문서는 개발자 문서 플랫폼인 [Mintlify](https://mintlify.com)로 제작하고 호스팅합니다.
