# Box

> Box는 이미지 안에서 무언가의 위치를 찾는 System One 질문 유형입니다. 답은 대상을 둘러싼 상자의 네 변이며, 이미지 너비와 높이에 대한 비율로 표현됩니다.

**Dreamer-ready.** 이미지와 Box 질문은 아직 운영 환경에 쓸 준비가 되었다고 말할 만큼 저희 기준에 이르지 못했습니다. 그래도 꿈꾸는 사람과 만드는 사람이 마음껏 써 보기를 기대하고 있습니다. 프로토타입을 만들고 아이디어를 시험하는 데 사용하세요. 이 페이지에 나온 요청 형식, 답, 제한 사항은 릴리스마다 바뀔 수 있고, 정확도도 모델을 업데이트하면서 달라집니다. 무엇을 만들었는지, 무엇이 제대로 동작하지 않는지 [Discord](https://discord.gg/typesafe)에서 알려 주세요.

이미지나 Box 질문을 사용하려면 이미지나 Box 질문이 들어 있는 요청마다 `TypeSafe-Danger-Zone-Images: true` 헤더를 보내세요. [이미지 켜기](../images.md#turning-on-images)를 참고하세요.

Box는 이미지 안에서 무언가가 어디에 있는지 알아야 할 때 사용합니다. 조리대 위의 머그잔, 스캔한 서식의 서명, 문 앞에 있는 사람 같은 것입니다. 답은 bounding box(경계 상자), 즉 대상을 포함하는 가장 작은 직사각형이며 네 변으로 주어집니다. Box 질문을 하려면 요청에 이미지가 있어야 합니다. 이미지를 보내는 방법은 [Images](../images.md)를 참고하세요.

각 변은 이미지 크기에 대한 비율이며 왼쪽 위 모서리를 기준으로 잽니다. `left`와 `right`는 너비에 대한 비율이고 `top`과 `bottom`은 높이에 대한 비율이므로, 같은 답을 어떤 이미지 크기에서든 그대로 쓸 수 있습니다.

![이미지 안에 커피 머그잔이 그려져 있고, 머그잔 둘레에 분홍색 상자가 있는 그림. 상자의 왼쪽 변은 이미지 너비의 0.44 지점에, 오른쪽 변은 0.84 지점에 있습니다. 위쪽 변은 높이의 0.25 지점에, 아래쪽 변은 0.82 지점에 있습니다. 이미지의 왼쪽 위 모서리는 (0, 0)이고 오른쪽 아래 모서리는 (1, 1)입니다.](https://mintcdn.com/ts-docs/OmOS-OoUA_8VxwSR/images/images/box-edges-light.webp?fit=max\&auto=format\&n=OmOS-OoUA_8VxwSR\&q=85\&s=30d1e2ea5b3abfe929b7a0b9dd8ce0d9)

## 요청 구조

Box 질문에는 필드가 두 개 있습니다.

* `type`: 항상 `"box"`입니다.
* `instructions`: 무엇을 어느 이미지에서 찾을지 적습니다. 꼭 질문 형태일 필요는 없습니다. "the coffee mug in `photo`"면 충분합니다.

Box는 `criteria`를 받지 않습니다. 아래 요청은 주방 사진에서 머그잔을 찾습니다. 사진은 파일 파트로 보내고 이미지 헤더를 함께 붙입니다.

```bash
curl https://api.typesafe.ai/v1/systemone \
  -H "Authorization: Bearer $TYPESAFE_API_KEY" \
  -H "TypeSafe-Danger-Zone-Images: true" \
  -F 'request={
    "model": "jev-latest",
    "state": {"photo": "6e1d0b7c-kitchen"},
    "assets": {"6e1d0b7c-kitchen": {"type": "image", "part": "kitchen-photo"}},
    "questions": {
      "mug": {"type": "box", "instructions": "the coffee mug in `photo`"}
    }
  };type=application/json' \
  -F 'kitchen-photo=@kitchen.jpg;type=image/jpeg'
```

사진을 base64로 인코딩해 JSON 본문에 담아 보내거나 SDK를 사용하려면 [이미지 업로드](../images.md#uploading-images)를 참고하세요.

한 요청 안에 Box 질문과 다른 유형의 질문을 섞을 수 있으며, 모든 질문은 같은 state(상태)를 대상으로 평가됩니다.

## 응답 구조

```json
{
  "model": "jev-1.14.0",
  "answers": {
    "mug": {
      "type": "box",
      "box": {
        "top": 0.25,
        "bottom": 0.82,
        "left": 0.44,
        "right": 0.84
      },
      "confidence": 0.92
    }
  },
  "usage": {"input_tokens": 290, "output_tokens": 21}
}
```

각 Box 답에는 다음 값이 들어 있습니다.

* `type`: System One 질문의 유형입니다.
* `box`: 네 변 `top`, `bottom`, `left`, `right`이며 각각 0에서 1 사이입니다. `left`는 `right`보다 작고 `top`은 `bottom`보다 작습니다.
* `confidence`: 0에서 1 사이의 숫자입니다. 높을수록 모델이 자신의 답을 더 확신한다는 뜻입니다. [confidence 활용하기](#using-confidence)를 참고하세요.

## Box 읽는 법

상자를 그리거나 상자에 맞춰 잘라 내려면 각 변에 이미지의 픽셀 단위 너비나 높이를 곱하세요.

```python
from PIL import Image

image = Image.open("kitchen.jpg")
width, height = image.size
mug = response["answers"]["mug"]
edges = mug["box"]

box = (
    round(edges["left"] * width),
    round(edges["top"] * height),
    round(edges["right"] * width),
    round(edges["bottom"] * height),
)
image.crop(box).save("mug.jpg")
```

### Confidence 활용하기

Box의 confidence(신뢰도)에 따라 어떻게 처리할지는 애플리케이션에 달려 있습니다. 흔히 쓰는 방식은 threshold(임계값)를 정하고 그보다 낮은 답은 더 조심스럽게 다루는 것입니다. 아직 권장 threshold는 없으므로 여러분의 이미지로 직접 조정하세요. confidence가 threshold보다 낮을 때는 다음과 같이 할 수 있습니다.

* **더 나은 이미지로 다시 묻기.** 더 큰 버전의 이미지를 보내거나, 상자 주변 영역을 더 높은 해상도로 잘라 낸 뒤 그 이미지에 같은 Box 질문을 던지세요. 큰 이미지에서 작은 대상을 찾는 방법은 [Box 질문 작성하기](#writing-box-questions)에서 다룹니다.
* **잘라 내기 전에 상자에 여백 더하기.** 대상의 어느 부분도 잘리면 안 될 때는 잘라 내기 전에 각 변을 바깥쪽으로 옮기세요.

아래 코드는 confidence가 threshold보다 낮을 때 상자에 여백을 더하되, 상자가 이미지 밖으로 나가지 않게 합니다.

```python
PAD = 0.05  # 각 변에 더할 여백으로, 이미지 크기에 대한 비율입니다. 이미지에 맞게 조정하세요.
MIN_CONFIDENCE = 0.6  # 예시 threshold입니다. 여러분의 이미지로 직접 조정하세요.

if mug["confidence"] < MIN_CONFIDENCE:
    edges = {
        "left": max(0.0, edges["left"] - PAD),
        "top": max(0.0, edges["top"] - PAD),
        "right": min(1.0, edges["right"] + PAD),
        "bottom": min(1.0, edges["bottom"] + PAD),
    }
```

그런 다음 여백을 더한 `edges`로 위 코드처럼 잘라 냅니다.

## Box 질문 작성하기

**대상과 이미지를 명시하세요.** "The coffee mug in `photo`"는 무엇을 어디에서 찾을지 명시합니다. 요청에 이미지가 여러 장 있으면 백틱 안의 경로가 어느 이미지를 살펴볼지 알려 줍니다.

**대상은 사진일 수도 있습니다.** 대상 이미지에 대한 참조를 instructions(지시문)에 직접 넣으세요.

```json
"instructions": "the person in `photo` who matches this badge photo: 9a21c7d3-badge"
```

**Box의 답은 언제나 상자입니다.** "찾지 못함"이라는 답은 없으므로, 이미지에 없는 대상을 묻는 Box 질문도 네 변을 반환합니다. 같은 요청에서 대상이 이미지에 있는지 [Noul](noul.md)로 묻고, Noul 답이 낮으면 상자를 무시하세요.

```json
"questions": {
  "mug_visible": {"type": "noul", "instructions": "Is there a coffee mug in `photo`?"},
  "mug": {"type": "box", "instructions": "the coffee mug in `photo`"}
}
```

**질문 하나에 상자 하나.** Box는 대상 하나를 찾도록 설계되었고, 모든 답은 상자 하나입니다. 서로 다른 여러 대상을 찾으려면 같은 요청에서 대상마다 Box 질문을 하나씩 하세요. 이미지에 같은 종류의 대상이 둘 이상 있을 수 있다면, 원하는 대상만 들어맞도록 묘사하세요. 예: "the mug nearest the sink in `photo`". 말로 구별할 수 없을 때는 Box 옆에 `none`, `one`, `several`을 선택지로 하는 Choice를 추가하세요. 답이 `several`이면 이미지를 여러 영역으로 나누고 영역마다 다시 물으세요.

**작은 대상을 찾기 전에 큰 이미지를 잘라 내세요.** 아주 큰 이미지에서 너비가 수십 픽셀밖에 안 되는 대상은 놓칠 수 있습니다. 먼저 영역을 찾으세요. 예를 들어 이미지를 타일로 나누어 선택지로 삼는 Choice를 쓸 수 있습니다. 그런 다음 해당 타일에 Box 질문을 던지고, 그 변을 다시 전체 이미지에 대한 비율로 환산하세요.


이 문서는 개발자 문서 플랫폼인 [Mintlify](https://mintlify.com)로 제작하고 호스팅합니다.
