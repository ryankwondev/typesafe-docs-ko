# 이미지

> System One 요청에 이미지를 보내는 방법을 다룹니다. state 안에, 질문의 instructions 안에, 그리고 질문을 판단하는 기준인 criteria로 이미지를 넣을 수 있습니다.

**Dreamer-ready(몽상가를 위한 단계).** 이미지와 Box 질문은 아직 프로덕션에 쓸 준비가 되었다고 말할 만큼 TypeSafe의 기준에 도달하지 못했습니다. 그래도 꿈꾸고 만드는 분들이 마음껏 가지고 놀아 보시기를 기대하고 있습니다. 프로토타입을 만들고 아이디어를 시험하는 데 사용하세요. 이 페이지에 나온 요청 형식, 답, 한도는 릴리스마다 바뀔 수 있고, 모델을 업데이트하면서 정확도도 달라집니다. 무엇을 만들었는지, 무엇이 깨지는지 [Discord](https://discord.gg/typesafe)에서 알려 주세요.

이미지나 Box 질문을 사용하려면 이미지 또는 Box 질문이 들어 있는 요청마다 `TypeSafe-Danger-Zone-Images: true` 헤더를 보내야 합니다. [이미지 켜기](images.md#turning-on-images)를 참고하세요.

System One 요청에서 텍스트가 들어갈 수 있는 곳이라면 어디든 이미지를 넣을 수 있습니다. `state`(상태), 질문의 `instructions`(지시문), 그리고 `criteria`(판단 기준)가 모두 해당합니다. 모든 질문 유형이 이미지를 받으며, [Box](primitives/box.md) 질문 유형은 이미지 안에서 대상이 어디에 있는지를 반환합니다.

![reference 세 개가 강조되어 있고 각각이 가리키는 이미지와 연결된 요청 본문: door.jpg는 state의 camera 값이고, badge.jpg는 Noul 질문의 instructions 안에 있으며, thumb.jpg는 Choice 질문의 criteria에서 thumbs\_up 선택지의 값입니다. 이미지 자체는 요청의 assets에 나열되어 있습니다.](https://mintcdn.com/ts-docs/OmOS-OoUA_8VxwSR/images/images/anatomy-light.webp?fit=max\&auto=format\&n=OmOS-OoUA_8VxwSR\&q=85\&s=7d5e99842cb8a482309f65a6d4aa195e)

## 이미지 켜기

이미지와 [Box](primitives/box.md) 질문은 기본적으로 꺼져 있습니다. 사용하려면 이미지 또는 Box 질문이 들어 있는 요청마다 `TypeSafe-Danger-Zone-Images: true` 헤더를 보내세요. 방법은 [이미지 업로드](#uploading-images) 아래의 예제에 나와 있습니다. 이 페이지의 다른 예제는 요청 본문만 보여 줍니다.

## 요청에 이미지 보내기

각 이미지는 요청 최상위의 `assets` 맵에 한 번만 나열합니다. 이 맵에서 이미지의 키가 reference(참조) 문자열입니다. 이미지가 들어가야 할 자리에 reference를 적으면 되며, 문자열 값 전체로 써도 되고 더 긴 문자열 안에 넣어도 됩니다. 모델은 그 자리에서 이미지를 받습니다.

다음 요청은 발견된 개의 사진을 state로 보내고, 실종견 전단에 실린 사진을 질문 하나에 직접 넣습니다.

```json
{
  "model": "jev-latest",
  "state": {
    "photo": "a3f1c2e8-found"
  },
  "assets": {
    "a3f1c2e8-found": {"type": "image", "format": "jpg", "data_base64": "/9j/4AAQSkZJRg..."},
    "77b0d4a1-poster": {"type": "image", "format": "jpg", "data_base64": "/9j/4AAQSkZJRg..."}
  },
  "questions": {
    "same_dog": {
      "type": "noul",
      "instructions": "Is the dog in `photo` the same dog as in this picture: 77b0d4a1-poster?"
    },
    "has_collar": {
      "type": "noul",
      "instructions": "Is the dog in `photo` wearing a collar?"
    },
    "coat": {
      "type": "choice",
      "instructions": "What is the coat of the dog in `photo` like?",
      "criteria": {
        "short": "Short and smooth.",
        "long": "Long or feathered.",
        "curly": "Curly or wiry."
      }
    }
  }
}
```

질문에서는 다른 필드를 가리킬 때와 똑같이, state에 있는 이미지를 백틱으로 감싼 이름이나 경로로 가리킬 수 있습니다([특정 필드 참조하기](primitives.md#reference-specific-fields) 참고). 한 질문에서만 쓰는 이미지는 위의 전단 사진처럼 그 질문의 `instructions`나 `criteria`에 직접 넣습니다.

Jev 1.13과 Jev 1.14 모두 이미지를 받습니다. [Box](primitives/box.md) 질문에는 Jev 1.14가 필요합니다.

### Asset 필드

**assets** `map\<string, Asset\>`

요청에 포함된 이미지들이며, reference 문자열을 키로 씁니다. 모든 reference는 `state`, `instructions`, `criteria` 중 어딘가에 반드시 나타나야 합니다.

**Asset 속성**

**type** `"image"`

asset(첨부 자산)의 종류입니다. 선택 사항입니다. 보낸다면 반드시 `"image"`여야 합니다.

**part** `string`

multipart 요청에서 이미지를 담고 있는 파일 파트의 이름입니다. [이미지 업로드](#uploading-images)를 참고하세요.

**data\_base64** `string`

base64로 인코딩한 이미지 파일입니다. `part`를 지정하지 않았다면 필수입니다. `=` 패딩이 있는 표준 base64 알파벳을 사용하고, 줄바꿈이나 다른 공백을 넣지 말고, `data:` 접두사도 붙이지 마세요.

**format** `"png" | "jpg" | "jpeg"`

소문자로 쓴 이미지 형식입니다. `data_base64`를 쓸 때는 필수이고 `part`를 쓸 때는 선택 사항입니다. `"jpeg"`는 `"jpg"`의 다른 이름입니다.

이미지마다 `part`나 `data_base64` 중 한 가지 방식으로만 보내세요. asset에 둘 다 있으면 API는 `part`를 사용합니다.

### Reference

reference는 요청의 문자열 어디에서든 일반 텍스트로 매칭되므로, 값 전체로 쓸 때뿐 아니라 문장 중간에서도 동작합니다. `cat`처럼 짧은 reference는 `category` 안에서도 매칭될 수 있습니다. 텍스트에 우연히 나타날 수 없는 reference를 사용하세요. 요청마다, 이미지마다 새 UUID를 쓰는 것이 가장 안전합니다.

* **모든 asset은 참조되어야 합니다.** 어디에서도 참조되지 않는 asset이 있는 요청은 `400`을 반환합니다.
* **값 전체가 일치하는 경우를 먼저 매칭합니다.** 정확히 reference와 같은 문자열은 그 이미지가 됩니다. 더 긴 문자열에서는 reference가 왼쪽에서 오른쪽으로 치환되며, 매칭이 서로 겹치는 일은 없습니다.
* **사용할 때마다 이미지가 다시 추가됩니다.** 파일은 한 번만 업로드되지만, reference가 나타나는 자리마다 모델이 읽는 내용에 이미지 사본이 하나씩 더해지고, 모든 사본이 [확장된 요청 한도](#limits)에 포함됩니다.
* **객체의 키는 치환되지 않습니다.** Choice 선택지 이름, 질문 ID를 비롯한 모든 객체 키는 텍스트로 남습니다. Choice 선택지를 그림으로 정의하려면 [Criteria](#criteria)에서처럼 선택지의 설명에 reference를 넣으세요.
* **답에는 이미지가 아니라 reference가 들어갑니다.** Score의 `legend`처럼 답이 criteria를 그대로 되풀이하는 곳이라면 어디든 그렇습니다.
* **키는 짧게 유지하세요.** reference 하나는 UTF-8 기준 최대 1,024바이트이고, 한 요청의 모든 reference를 합쳐 최대 65,536바이트입니다. 이 한도는 문자 수가 아니라 바이트 수를 셉니다. reference는 비어 있거나 공백으로만 이루어질 수 없습니다.

Python SDK와 JavaScript SDK는 reference를 대신 만들어 줍니다. [이미지 업로드](#uploading-images)를 참고하세요.

## 이미지 업로드

최신 Python 또는 JavaScript SDK를 쓴다면 텍스트가 들어갈 수 있는 곳 어디에든 이미지 객체를 넣으면 됩니다. SDK가 reference를 만들고 파일을 업로드합니다. 다음 예제는 현관 초인종 카메라 사진 한 장을 state로 보내고 그 사진에 대해 질문 두 개를 합니다.

```python title="Python"
from pathlib import Path

from typesafe_sdk import Choice, Image, Noul, TypeSafeClient

with TypeSafeClient() as client:
    result = client.system_one(
        state={"door": Image(Path("door-camera.jpg"))},
        questions={
            "person_present": Noul(instructions="Is there a person in `door`?"),
            "carrying": Choice(
                instructions="What is the person in `door` carrying?",
                criteria={
                    "package": "A box or parcel.",
                    "bag": "A bag of groceries or shopping.",
                    "nothing": "Empty hands.",
                },
            ),
        },
        extra_headers={"TypeSafe-Danger-Zone-Images": "true"},
    )
    print(result.answers)
```

```typescript title="JavaScript"
import { openAsBlob } from "node:fs";
import { choice, image, noul, TypeSafeClient } from "@typesafe-ai/sdk";

const client = new TypeSafeClient();
const door = image(await openAsBlob("door-camera.jpg"));

const { answers } = await client.systemOne(
  {
    state: { door },
    questions: {
      person_present: noul("Is there a person in `door`?"),
      carrying: choice("What is the person in `door` carrying?", {
        package: "A box or parcel.",
        bag: "A bag of groceries or shopping.",
        nothing: "Empty hands.",
      }),
    },
  },
  { headers: { "TypeSafe-Danger-Zone-Images": "true" } },
);
console.log(answers);
```

이미지를 문장 안에 넣거나 reference를 직접 정하려면 [SDK 가이드](sdk.md)를 참고하세요.

### HTTP로 보내기

SDK 없이 보낼 때는 `multipart/form-data` 요청에서 각 이미지를 파일 파트로 보내세요. multipart가 더 나은 선택입니다. multipart는 파일을 있는 그대로 보내지만 base64는 이미지마다 크기를 약 3분의 1만큼 늘리고, 늘어난 크기는 32 MiB인 [요청 본문 한도](#limits)에 포함됩니다. multipart로 보낼 수 없을 때만 이미지를 base64로 인코딩해 JSON에 넣으세요.

> [tab] Multipart(권장)
>
> JSON 본문을 `request`라는 이름의 첫 번째 파트로, 콘텐츠 타입 `application/json`으로 보냅니다. 각 이미지는 별도의 파일 파트로 보내고, `assets`에서 해당 이미지의 `part`를 그 파일 파트의 이름으로 설정합니다.
>
> ```bash theme={null}
> curl https://api.typesafe.ai/v1/systemone \
>   -H "Authorization: Bearer $TYPESAFE_API_KEY" \
>   -H "TypeSafe-Danger-Zone-Images: true" \
>   -F 'request={
>     "model": "jev-latest",
>     "state": {"door": "a3f1c2e8-door"},
>     "assets": {"a3f1c2e8-door": {"type": "image", "part": "door-photo"}},
>     "questions": {
>       "person_present": {"type": "noul", "instructions": "Is there a person in `door`?"}
>     }
>   };type=application/json' \
>   -F 'door-photo=@door-camera.jpg;type=image/jpeg'
> ```
>
> multipart 요청의 규칙은 다음과 같습니다.
>
> * 첫 번째 파트는 이름이 `request`이고 JSON 본문을 담습니다.
> * 파트 이름은 고유해야 하고 비어 있으면 안 됩니다. `request`는 JSON 본문용으로 예약되어 있습니다.
> * 각 이미지 파트의 콘텐츠 타입은 `image/png`, `image/jpeg`, `image/jpg`, `application/octet-stream` 중 하나이거나 지정하지 않아야 합니다. 그 밖의 콘텐츠 타입은 `400`을 반환합니다.
> * 각 파일 파트는 정확히 하나의 asset에 속합니다. 어떤 asset도 가리키지 않는 파트, 같은 파트를 가리키는 두 asset, 가리키는 파트가 없는 asset은 모두 `400`을 반환합니다.
>
> 같은 이미지를 여러 곳에서 쓰려면 그 자리마다 reference를 적으세요.

> [tab] JSON 안의 Base64
>
> [Asset 필드](#asset-fields)의 규칙에 따라 파일을 base64로 인코딩해 `data_base64`에 넣고 `format`을 설정합니다. Python의 `base64.b64encode`가 올바른 인코딩을 만들어 줍니다. 다음 Python 예제는 사진 한 장을 보내고 그 사진에 대해 질문 두 개를 합니다.
>
> ```python theme={null}
> import base64
> import os
> import uuid
>
> import requests
>
> def image_asset(path: str) -> tuple[str, dict]:
>     """Return a new reference and the asset entry for one image file."""
>     fmt = "png" if path.lower().endswith(".png") else "jpg"
>     with open(path, "rb") as f:
>         data = base64.b64encode(f.read()).decode("ascii")
>     return str(uuid.uuid4()), {"type": "image", "format": fmt, "data_base64": data}
>
> photo_key, photo = image_asset("door-camera.jpg")
>
> response = requests.post(
>     "https://api.typesafe.ai/v1/systemone",
>     headers={
>         "Authorization": f"Bearer {os.environ['TYPESAFE_API_KEY']}",
>         "TypeSafe-Danger-Zone-Images": "true",
>     },
>     json={
>         "model": "jev-latest",
>         "state": {"door": photo_key},
>         "assets": {photo_key: photo},
>         "questions": {
>             "person_present": {
>                 "type": "noul",
>                 "instructions": "Is there a person in `door`?",
>             },
>             "carrying": {
>                 "type": "choice",
>                 "instructions": "What is the person in `door` carrying?",
>                 "criteria": {
>                     "package": "A box or parcel.",
>                     "bag": "A bag of groceries or shopping.",
>                     "nothing": "Empty hands.",
>                 },
>             },
>         },
>     },
> )
> print(response.json()["answers"])
> ```

## 이미지 배치

### State

`state`에 넣은 이미지는 그 안의 텍스트나 JSON과 마찬가지로 state의 일부입니다. 모델은 이미지를 state의 나머지 부분과 함께 평가하고, 요청의 모든 질문이 이 이미지를 대상으로 평가됩니다. state에는 이미지와 텍스트를 섞어 넣을 수 있고, 다른 state 데이터처럼 이름을 붙이거나 중첩해서 이미지를 여러 장 담을 수도 있습니다. 다음 state는 카메라 프레임의 짧은 이력을 오래된 것부터 순서대로 담고 있습니다.

```json
{
  "state": {
    "frames": [
      {"seconds_ago": 2, "frame": "5c0e9a12-f1"},
      {"seconds_ago": 1, "frame": "1d7be0c4-f2"},
      {"seconds_ago": 0, "frame": "e48a6f30-f3"}
    ]
  }
}
```

질문에서는 "Did the door in `frames` open during these frames?"(이 프레임들 사이에 `frames`의 문이 열렸나요?)라고 물을 수도 있고, `` `frames[2].frame` ``처럼 경로로 프레임 하나를 가리킬 수도 있습니다.

### Instructions

질문의 `instructions`에 넣은 이미지는 그 질문의 일부이며 그 질문과 함께만 전송됩니다. 질문에서 이미지가 필요한 문장 위치에 reference를 넣으세요.

```json
{
  "model": "jev-latest",
  "state": {"door": "e48a6f30-door"},
  "questions": {
    "is_badge_holder": {
      "type": "noul",
      "instructions": "Is the person in `door` the same person as in this badge photo: 9a21c7d3-badge?"
    }
  }
}
```

`instructions`에 넣은 이미지는 그 이미지를 포함한 질문마다 한 번씩 모델에 전송됩니다. 여러 질문이 같은 이미지를 쓴다면 `state`에 한 번만 넣고 각 질문에서 경로로 가리키세요. 그러면 token(토큰)을 덜 쓰고 [크기 한도](#limits)도 덜 차지합니다.

### Criteria

Choice의 선택지, Score의 단계, Noul의 `true`와 `false` 정의에는 각각 이미지를 넣을 수 있습니다. 다음 Choice는 손동작을 분류하며, 각 선택지를 예시 그림으로 정의합니다. 선택지 이름은 코드가 `choice`로 받는 값이고, 그림은 모델이 손과 비교하는 대상입니다.

```json
{
  "model": "jev-latest",
  "state": {"camera": "e48a6f30-cam"},
  "questions": {
    "gesture": {
      "type": "choice",
      "instructions": "Which gesture is the hand in `camera` making?",
      "criteria": {
        "thumbs_up": "0b6c51aa-thumb",
        "open_palm": {
          "description": "A hand held up with the palm facing the camera.",
          "examples": ["4fd2e7b9-palm1", "c83a0e15-palm2"]
        },
        "fist": "a7e94d02-fist",
        "none": "No hand, or a hand that matches none of the examples."
      }
    }
  }
}
```

선택지는 이미지 하나만으로 이루어질 수도 있고(`thumbs_up`), 설명에 예시를 하나 이상 붙인 형태일 수도 있고(`open_palm`), 텍스트만일 수도 있습니다(`none`). `description`과 `examples`라는 키는 이 예제를 위해 고른 이름일 뿐 필수 필드가 아닙니다. criteria는 어떤 [구조](#structured-data)든 받습니다.

Score와 Noul의 criteria도 같은 방식으로 이미지를 받습니다.

* **Score 단계**에는 각각 예시 그림을 넣을 수 있습니다. 손이 얼마나 펴져 있는지를 재는 Score라면 꽉 쥔 주먹, 반쯤 편 손, 활짝 편 손을 세 단계로 쓸 수 있습니다.
* **Noul criteria**는 `true`와 `false`를 그림으로 정의할 수 있습니다. 예: `{"true": {"description": "Correctly assembled", "examples": ["<reference>"]}, "false": "Parts missing or in the wrong place."}`.

### 구조화된 데이터

state, instructions, criteria는 중첩된 JSON [구조](primitives/advanced.md)를 받으며, reference는 그 구조의 어느 깊이에서든 동작합니다. 질문에서는 배열 안의 그림 하나를 `` `lineup[2]` ``처럼, 이름이 붙은 객체 안의 그림 하나를 `` `faces.happy` ``처럼 가리킬 수 있습니다. criterion 하나에 예시 그림 여러 장을 배열로 담을 수도 있습니다.

## 패턴

| 패턴 | 동작 방식 | 이미지를 넣는 위치 |
| - | - | - |
| 알고 있는 그림과 비교하기 | state가 알고 있는 그림과 같은 사람, 제품, 장소를 보여 주는지 묻습니다. | 알고 있는 그림은 `instructions`, 사진은 `state` |
| 예시로 선택지 정의하기 | Choice 선택지마다 예시 그림을 하나 또는 몇 장씩 써서 정의합니다. | `criteria` |
| 무엇이 바뀌었는지 | 이전 프레임이나 짧은 프레임 이력을 보내고 움직임, 새로 나타난 것, 차이점을 묻습니다. | `state` |
| 필요할 때 더 자세히 보기 | 먼저 이미지의 작은 사본을 놓고 묻습니다. 답에 더 세밀한 정보가 필요할 때만 더 큰 사본(크기 한도 이내)이나 원본 해상도로 잘라 낸 부분을 보냅니다. | `state` |
| 큰 이미지에서 찾기 | 이미지를 타일로 자르고, 타일들을 선택지로 하는 Choice를 묻습니다. 대상이 타일의 대부분을 차지할 때까지 선택된 타일에 같은 과정을 반복합니다. | `criteria` |
| 위치 찾기 | [Box](primitives/box.md) 질문으로 대상의 가장자리를 묻습니다. | `state`, 필요하면 대상의 그림을 `instructions`에 추가 |

## 한도

| 한도 | 값 |
| - | - |
| 형식 | PNG 또는 JPEG이며 정지 이미지만 가능합니다. 애니메이션 PNG는 거부됩니다. |
| 파일 크기 | 이미지 파일당 16 MiB입니다. base64의 경우 디코딩한 뒤의 크기입니다. |
| 너비와 높이 | 각각 280~1568픽셀입니다. 어느 한 변이라도 280픽셀 미만인 이미지는 거부됩니다. |
| 총 픽셀 수(너비 × 높이) | 78,400~2,458,624이며, 280 × 280 정사각형부터 1568 × 1568 정사각형까지의 넓이입니다. |
| 요청 본문 | JSON, base64 텍스트, multipart 프레이밍을 포함해 32 MiB입니다. HTTP 헤더는 포함되지 않습니다. [압축](api/errors.md#compression)을 써도 한도는 늘어나지 않습니다. |
| 요청의 모든 이미지 | 32 MiB이며, API가 처리한 뒤의 서로 다른 이미지 전체가 이 한도를 나눠 씁니다. 처리 과정에서 각 이미지를 다시 인코딩하므로 보낸 파일보다 커질 수 있고, 그래서 작은 파일만 업로드한 요청도 한도를 넘을 수 있습니다. |
| 확장된 요청 | 모든 reference가 이미지로 치환된 뒤의 텍스트, 질문, 구조, 이미지를 합쳐 32 MiB입니다. reference를 쓸 때마다 이미지가 다시 추가됩니다. 처리 후 10 MiB인 이미지를 네 번 참조하면 한도를 넘습니다. |

모든 한도는 범위의 양 끝 값을 포함합니다. 한도를 벗어난 이미지는 거부되며, 한도에 맞게 크기가 조정되는 일은 없습니다. 요청에 든 이미지 중 하나라도 거부되면 요청 전체가 실패합니다. 일부만 답이 오는 경우는 없습니다. 상태 코드는 [오류](#errors)를 참고하세요.

한 요청에 이미지가 몇 장 들어갈 수 있는지는 이 한도들과 [컨텍스트 길이](models.md)에 따라 달라집니다. 이미지에는 별도의 [rate limit(요청 한도)](api/errors.md#handling-rate-limits)이 없습니다. 이미지가 든 요청도 다른 요청과 같은 한도에서 차감됩니다.

요청 한도 안에 머무르려면 이미지 수를 줄이거나, 더 작은 이미지를 보내거나, 여러 질문이 쓰는 이미지를 각 질문의 `instructions`가 아니라 `state`에 한 번만 넣거나, 작업을 여러 요청으로 나누세요.

## 이미지 준비하기

각 이미지는 똑바로 세워 회전하고 [한도](#limits) 이내의 크기로 맞춘 PNG 또는 JPEG로 보내세요. 보내기 전에 이미지마다 다음과 같이 준비합니다.

* **보내기 전에 크기를 조정하세요.** 가장 긴 변이 최대 1568픽셀이 되도록 이미지를 축소합니다. 일반적인 1,200만 화소 휴대폰 사진은 1568 × 1176이 됩니다. 최소 크기보다 작은 이미지는 확대하거나 여백을 덧대세요.
* **픽셀 자체를 똑바로 회전하세요.** API는 모든 메타데이터(EXIF, 색상 프로파일, 주석)를 제거하며 EXIF 방향 정보를 적용하지 않습니다. 사진 뷰어에서는 똑바로 보이는 사진이 모델에는 옆으로 누운 채 전달될 수 있으므로, 먼저 픽셀 자체를 회전해 두세요.
* **정지 이미지를 보내세요.** 애니메이션이나 동영상은 원하는 프레임을 PNG 또는 JPEG로 내보내세요. GIF, WebP, HEIC 같은 다른 형식은 PNG 또는 JPEG로 변환하세요.
* **표준 JPEG를 쓰세요.** 표준 8비트 baseline JPEG가 가장 안전하게 보낼 수 있는 JPEG입니다. progressive JPEG는 baseline으로 다시 저장하세요.
* **거부된 파일은 다시 내보내세요.** 컴퓨터에서는 문제없이 열리는 파일을 API가 거부한다면, 표준 이미지 편집기나 라이브러리로 열어서 PNG 또는 JPEG로 다시 내보내세요.

다음 Python 예제는 Pillow 라이브러리로 사진 한 장을 준비합니다.

```python
from PIL import Image, ImageOps

with Image.open("photo.jpg") as original:
    img = ImageOps.exif_transpose(original)  # 픽셀을 똑바로 회전
    img.thumbnail((1568, 1568))  # 가장 긴 변을 최대 1568로, 가로세로 비율 유지
    # thumbnail은 축소만 합니다. 최소 크기보다 작은 이미지는 따로 확대하거나 여백을 덧대세요
    img.convert("RGB").save("photo-ready.jpg", "JPEG", quality=90)  # baseline JPEG
```

## 이미지 token

각 이미지는 입력 token으로 계산되며, 텍스트와 같은 token당 [입력 가격](models.md)으로 과금됩니다. 이미지에 대한 별도 요금은 없습니다. 응답의 `usage.input_tokens` 필드에는 요청에 든 모든 이미지의 token이 포함됩니다. 이미지 token은 요청의 [컨텍스트 길이](models.md)에도 포함됩니다.

이미지 token 사용량을 추정하는 방법은 곧 안내할 예정입니다.

## 오류

| 상태 | 원인 |
| - | - |
| `400` | 이미지가 켜져 있지 않습니다. 요청에 이미지나 Box 질문이 있는데 `TypeSafe-Danger-Zone-Images: true` 헤더가 없습니다. [이미지 켜기](#turning-on-images)를 참고하세요. |
| `400` | 이미지 또는 asset이 거부되었습니다. 파일이 PNG나 JPEG가 아니거나, 손상되었거나 불완전하거나, 애니메이션인 경우, 이미지가 최소 너비, 높이, 총 픽셀 수에 못 미치는 경우, 어디에서도 참조되지 않는 asset이 있는 경우, base64나 `format`이 유효하지 않은 경우, reference가 비어 있는 경우, multipart 요청이 [multipart 규칙](#over-http)을 어긴 경우입니다. |
| `400` | `max_tokens_exceeded`: 이미지와 텍스트를 합친 양이 요청의 [컨텍스트 길이](models.md)를 넘습니다. 이미지 수나 크기를 줄이거나, 이미지를 `instructions`에서 `state`로 옮기세요. |
| `413` | 요청이 크기 한도를 넘었습니다. 이미지 파일, 이미지의 너비, 높이, 총 픽셀 수, 요청 본문, 모든 이미지의 합계, 확장된 요청, reference 길이 중 하나가 해당합니다. [한도](#limits)를 참고하세요. |

요청에 문제가 여러 개 있으면 오류는 그중 하나만 알려 줍니다. [한도](#limits)와 [이미지 준비하기](#preparing-images)에 비추어 요청을 점검하세요.


이 문서는 개발자 문서 플랫폼인 [Mintlify](https://mintlify.com)로 제작하고 호스팅합니다.
