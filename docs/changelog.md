# 모델 변경 이력

> Jev 릴리스별 주요 변경 사항입니다.

각 항목은 Jev 릴리스에서 무엇이 바뀌었는지 정리합니다. 현재 호출할 수 있는 모델과 그 가격, 컨텍스트 한도는 [모델](models.md)을 참고하세요. 아직 남아 있는 약점은 [Model jaggedness(모델의 들쭉날쭉함)](model-jaggedness.md)를 참고하세요.

**Jev 1.14**

2026년 10월 14일

* 모델 ID는 `jev-1.14.0`입니다.
* 다국어 지원이 개선되었습니다.
* 새로운 질문 유형인 [Box](primitives/box.md)가 추가되었습니다. 이미지 안에서 대상의 위치를 찾아 bounding box(경계 상자)로 답합니다. Box 질문에는 Jev 1.14가 필요합니다.
* 요청의 `state`, `instructions`, `criteria`에 [이미지](images.md)를 넣을 수 있습니다. Jev 1.13에서도 동작합니다.
* [Long context(긴 컨텍스트)](models.md#long-context)는 long-context 헤더를 보내는 요청에 한해 두 컨텍스트 한도를 모두 262k token(토큰)으로 늘립니다. Jev 1.13에서도 동작합니다.


이 문서는 개발자 문서 플랫폼인 [Mintlify](https://mintlify.com)로 제작하고 호스팅합니다.
