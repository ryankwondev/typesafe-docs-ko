# 새 모이통 카메라 방문 기록

> 새 모이통 카메라 영상의 프레임을 질문 다섯 개와 함께 Jev에 보냅니다. 그중 하나는 새의 위치를 찾는 Box입니다. 그런 다음 일반 코드로 답을 종별 방문 기록으로 바꿉니다.

새 모이통이나 현관, 길거리에 설치한 카메라는 몇 시간 분량의 영상을 녹화합니다. Jev를 쓰면
이 영상을 분석할 수 있습니다.

영상의 프레임을 한 장씩 Jev에 보내면서 프레임마다 질문을 합니다. 새 모이통 카메라
영상이라면 이 답이 종별 방문 기록이 됩니다.

![눈 쌓인 새 모이통에 수컷 홍관조와 솜털딱따구리가 있고, 각각 종 확률이 적힌
레이블 상자 안에 들어 있습니다. 상태 표시줄에는 시각, bird\_present
확률, 지금까지 열린 방문이 표시됩니다.](https://mintcdn.com/ts-docs/l_j5jNYNp6JTBESY/cookbooks/feeder_watch/feeder-watch-hero.webp?fit=max\&auto=format\&n=l_j5jNYNp6JTBESY\&q=85\&s=b18b39e8df73dcaa8cf0f5d4c30e9b0a)

Jev에 1초에 한 프레임씩 보내고, 요청마다 그 프레임에 대해 다섯 가지를 묻습니다.

* `bird_present`, `Noul`: 새가 있는가?
* `how_many`, none, one, several 중에서 고르는 `Choice`
* `bird`, `Box`: 가장 눈에 띄는 새가 어디에 있는가?
* `species_left`와 `species_right`, 종 목록에서 고르는 `Choice` 질문 두 개:
  가장 왼쪽 새와 가장 오른쪽 새는 무슨 종인가?

이미지에서 새가 어디 있는지 찾는 데는 [Box](../primitives/box.md) 질문을 씁니다. 이 질문은
상자를 하나만 반환하므로, 두 종이 모이통에 함께 있을 때는 두 번째 요청에서 종 이름을 지정해
종마다 상자를 하나씩 요청합니다. 이 cookbook(예제 모음)은 한 번에 최대 두 종까지 상자를 칩니다. 답을
방문으로 바꾸는 일은 일반 코드가 맡습니다.

저희가 사용한 101초짜리 영상 클립에서, 새가 있는지 여부는 101개 프레임 모두에서 Jev의 답이
저희가 직접 붙인 레이블과 일치했고, 레이블이 붙은 새 관찰 133건 중 132건에서 Jev가 종을
정확히 맞혔습니다.

> **참고** - 이미지와 Box 질문은 실험적 기능("dreamer-ready")입니다. 이미지나 Box 질문이
> 들어 있는 요청에는 모두 `TypeSafe-Danger-Zone-Images: true` 헤더가
> 필요합니다. [이미지](../images.md)와 [Box](../primitives/box.md)를 참고하세요.

## 준비

Python 3.10 이상이 필요합니다. 패키지 관리자를 골라 의존성을 설치합니다.

> [tab] pip
>
> 가상 환경을 만들고 의존성을 추가합니다.
>
> ```bash wrap theme={null}
> python3 -m venv .venv
> .venv/bin/pip install 'typesafe-sdk>=0.7.5,<0.8.0' pillow matplotlib ipython 'cooksafe>=0.3.0,<0.4.0'
> ```

> [tab] uv
>
> 프로젝트를 초기화하고 의존성을 추가합니다.
>
> ```bash wrap theme={null}
> uv init --bare
> uv add 'typesafe-sdk>=0.7.5,<0.8.0' pillow matplotlib ipython 'cooksafe>=0.3.0,<0.4.0'
> ```

TypeSafe API 키를 설정합니다.

```bash
export TYPESAFE_API_KEY="your-api-key"
```

`feeder_watch.py`라는 새 파일을 만듭니다. 이 cookbook을 따라가면서 Python 코드 블록을
순서대로 그 파일에 복사하세요. 다음 명령은 pip로 설치했든 uv로 설치했든 똑같이
실행됩니다.

```bash
.venv/bin/python feeder_watch.py
```

전체를 한 번 실행하면 요청을 180개 정도 보냅니다. 상자는 Pillow로 그리고, 그림 배치는
matplotlib이 맡습니다.

코드는 공개된 답을 `feeder_watch_cache.json`으로 내려받으므로 실행 중에 요청을
보내지 않습니다. 실제 요청을 보내려면 `DOWNLOAD_CACHE = False`로 설정하고 그 파일을 삭제하세요.

아래 수치는 2026-10-15에 `jev-1.14.0`으로 얻은 것입니다.

```python
import json
import os
import statistics
import urllib.request
import zipfile
from concurrent.futures import ThreadPoolExecutor
from pathlib import Path
from time import perf_counter

import matplotlib

matplotlib.use("Agg")
import matplotlib.pyplot as plt
from IPython.display import display
from PIL import Image as PILImage
from PIL import ImageDraw, ImageFont
from cooksafe import JsonCache
from typesafe_sdk import Box, Choice, Image, Noul, SystemOneResponse, TypeSafeClient

# Box 질문에는 Jev 1.14 이상이 필요합니다.
TYPESAFE_MODEL = "jev-1.14.0"  # 이 페이지에서 쓴 모델입니다. "jev-latest"로 바꾸면 최신 모델로 실제 요청을 보냅니다
IMAGE_HEADERS = {"TypeSafe-Danger-Zone-Images": "true"}  # 이미지와 Box는 명시적으로 켜야 합니다

client = TypeSafeClient(
    api_key=os.environ.get("TYPESAFE_API_KEY", "cache-only"),  # 키 없이 실행하면 캐시를 재생합니다
    timeout=60.0,
)
```

## 프레임 내려받기

영상은 ["Busy Bird Feeder"][usfws-clip]라는 101초짜리 클립으로, 미국 어류 및 야생동물 관리국(U.S. Fish and Wildlife Service)의
Courtney Celley가 촬영해 퍼블릭 도메인으로 공개했습니다. 처음
1분 동안은 수컷 홍관조와 솜털딱따구리가 모이통에 함께 있어서, 대부분의 프레임에 서로 다른 종의
새 두 마리가 담겨 있습니다. 그 뒤에는 수컷 홍관조가 떠나고 암컷 홍관조가 모이통 뒤쪽에서
먹이를 먹는데, 거의 보이지 않습니다.

저희는 1초에 한 프레임씩 1024 × 576 픽셀로 잘라 내고, 프레임마다 온전히 보이는 모든 새의
종을 직접 레이블로 붙였습니다. 22개 프레임에는 `partial` 표시가 있습니다. 이 프레임들에서는
암컷 홍관조가 가장자리에서 잘렸거나 모이통 뒤에 거의 가려져 있기 때문입니다.
프레임은 [`usfws_frames.zip`][frames-zip]에, 레이블은
[`usfws_labels.json`][labels-json]에 있으며, 둘 다 typesafe-public-examples에 있습니다. 출처와
변경 내역은
[`DATA_LICENSE.md`][data-license]에 있습니다. 아래 코드는 두 파일을 내려받고, 프레임을
`usfws_frames/`에 압축 해제하며, 이미 디스크에 있는 것은 건너뜁니다.

[usfws-clip]: https://www.flickr.com/photos/usfwsmidwest/31524812531

[data-license]: https://github.com/typesafe-ai/typesafe-public-examples/blob/main/cookbooks/feeder_watch/DATA_LICENSE.md

[frames-zip]: https://github.com/typesafe-ai/typesafe-public-examples/blob/main/cookbooks/feeder_watch/usfws_frames.zip

[labels-json]: https://github.com/typesafe-ai/typesafe-public-examples/blob/main/cookbooks/feeder_watch/usfws_labels.json

```python
EXAMPLES = (
    "https://raw.githubusercontent.com/typesafe-ai/typesafe-public-examples/main/"
    "cookbooks/feeder_watch"
)
FRAMES = Path("usfws_frames")
LABELS = Path("usfws_labels.json")


def download(url: str, path: Path) -> Path:
    if not path.exists():
        urllib.request.urlretrieve(url, path)
    return path
```

```python
CACHE = Path("feeder_watch_cache.json")
# True이면 공개된 답으로 CACHE를 채우므로 실행 중에 요청을 보내지 않습니다. 실제 요청을 보내려면
# False로 바꾸고 CACHE를 삭제하세요. 디스크에 이미 있는 CACHE는 덮어쓰지 않습니다.
DOWNLOAD_CACHE = True
if DOWNLOAD_CACHE:
    download(f"{EXAMPLES}/{CACHE.name}", CACHE)
json_cache = JsonCache(CACHE)


if not FRAMES.exists():
    with zipfile.ZipFile(download(f"{EXAMPLES}/usfws_frames.zip", Path("usfws_frames.zip"))) as zf:
        zf.extractall()  # zip 안에는 usfws_frames/u001.jpg부터 u101.jpg까지 들어 있습니다
NAMES = [f"u{second:03d}.jpg" for second in range(1, 102)]  # 1초에 한 프레임
paths = [FRAMES / n for n in NAMES]
labels = json.loads(download(f"{EXAMPLES}/usfws_labels.json", LABELS).read_text())

birds_per_frame = [len(labels[n]["species"]) for n in NAMES]
print(f"{len(paths)} frames, {PILImage.open(paths[0]).size[0]} x {PILImage.open(paths[0]).size[1]}")
print(f"frames by birds in full view: {[birds_per_frame.count(k) for k in (0, 1, 2)]} (0, 1, 2)")
print("u010.jpg is labeled", labels["u010.jpg"])
```

```
101 frames, 1024 x 576
frames by birds in full view: [11, 27, 63] (0, 1, 2)
u010.jpg is labeled {'species': ['cardinal_male', 'downy_woodpecker'], 'partial': False}
```

## 질문 작성하기

요청마다 프레임 한 장을 `frame`이라는 키로 state(상태)에 담아 보내고, 그 프레임에 대해 다섯 가지를
묻습니다. 다섯 질문이 모두 한 요청에 들어가므로 이미지는 한 번만 전송됩니다.

* `bird_present` 질문은 `Noul`입니다. 0.5 미만이면 빈 프레임으로 보고
  코드는 나머지 답을 무시합니다. Box 질문은 찾을 것이 없어도 항상 상자를 반환하므로,
  상자를 쓸지 말지는 이 답을 보고 코드가 판단합니다.
* `how_many` 질문은 `none`, `one`, `several` 중에서 고르는 `Choice`입니다. 코드는
  답이 `several`일 때만 두 번째 상자를 요청합니다.
* `bird` 질문은 가장 눈에 띄는 새를 둘러싸는 `Box`입니다. 새가 한 마리인 프레임에서는
  이 상자를 씁니다.
* `species_left`와 `species_right` 질문은 종 열 개에 `other`를 더한 선택지를 두고
  가장 왼쪽 새와 가장 오른쪽 새에 대해 묻는 같은 `Choice`입니다. 프레임에 새가 한 마리이면
  두 답이 모두 그 새를 설명합니다. 두 마리이면 답 하나가 새 한 마리씩을 설명합니다.

각 종 선택지에는 도감에 실린 설명이 아니라 모이통 카메라에 그 새가 어떻게 보이는지를
적습니다. 예를 들어 grackle 선택지에는 새가 프레임을 가득 채워서
머리만 보일 수 있다고 적혀 있습니다.

```python
SPECIES = {
    "blue_jay": "A blue bird with a crest, a pale grey-white face and chest, a black necklace, "
    "and black bars on the blue wings and tail.",
    "cardinal_male": "A bright red bird with a pointed crest, a black mask, and an orange beak.",
    "cardinal_female": "A warm brown or buff bird with a pointed crest, reddish tinges on the "
    "wings, tail and crest, and an orange beak.",
    "grackle": "A common grackle: a glossy black bird with a pale yellow eye, a long heavy dark "
    "beak, and an iridescent blue, purple or bronze sheen on the head and body. On a close-up "
    "feeder camera it often fills the frame, so only the glossy dark head, pale eye and beak "
    "may be visible.",
    "starling": "A European starling: a chunky short-tailed dark bird with glossy plumage "
    "speckled with pale spots, a dark eye, and a pointed yellow or dark beak. Juveniles are "
    "plain grey-brown with a dark beak.",
    "mourning_dove": "A plump grey-brown dove with a small head, a long pointed tail, and black "
    "spots on the wings.",
    "house_sparrow": "A small streaky brown sparrow. Males have a grey crown and a black bib.",
    "house_finch": "A small streaky brown finch. Males have a rosy red head and chest.",
    "red_winged_blackbird": "A glossy black bird with a red shoulder patch edged in yellow.",
    "downy_woodpecker": "A small black-and-white woodpecker with a white stripe down the back "
    "and spotted wings. Males have a red patch on the back of the head.",
    "other": "A bird that matches none of the other descriptions.",
}

QUESTIONS = {
    "bird_present": Noul(instructions="Is there at least one bird in `frame`?"),
    "how_many": Choice(
        instructions="How many birds are in `frame`?",
        criteria={
            "none": "No bird at all.",
            "one": "Exactly one bird.",
            "several": "Two or more birds.",
        },
    ),
    "bird": Box(instructions="The most prominent bird in `frame`"),
    "species_left": Choice(
        instructions="What species is the leftmost bird in `frame`?", criteria=SPECIES
    ),
    "species_right": Choice(
        instructions="What species is the rightmost bird in `frame`?", criteria=SPECIES
    ),
}
```

## 프레임 한 장에 대해 묻기

요청마다 프레임 한 장과 질문 묶음을 보내며, 모델 버전과 이미지 헤더는 위에서 설정한 값을
씁니다. 아래 세 프레임에는 각각 새가 두 마리, 한 마리, 없습니다.

```python
@json_cache
def ask_cached(frame: str, questions_json: str, model: str) -> dict:
    started = perf_counter()
    response = client.system_one(
        state={"frame": Image(FRAMES / frame)},
        questions=json.loads(questions_json),
        model=model,
        extra_headers=IMAGE_HEADERS,
    )
    seconds = round(perf_counter() - started, 2)
    return {"response": response.model_dump(mode="json", exclude={"model"}), "seconds": seconds}


def ask(path: Path, questions: dict):
    as_json = json.dumps(
        {key: q.model_dump(mode="json", exclude_none=True) for key, q in questions.items()},
        sort_keys=True,
    )
    cached = ask_cached(path.name, as_json, model=TYPESAFE_MODEL)
    # 캐시된 답에는 model이 없지만 응답 타입에는 필요하므로, 캐시 키로 쓴 모델을 넣습니다.
    response = SystemOneResponse.model_validate({**cached["response"], "model": TYPESAFE_MODEL})
    return response, cached["seconds"]


def edges(box) -> str:
    return f"left {box.left:.2f}  right {box.right:.2f}  top {box.top:.2f}  bottom {box.bottom:.2f}"


def show_first_request(name: str) -> None:
    response, seconds = ask(FRAMES / name, QUESTIONS)
    how_many = response.choices["how_many"]
    print(f"{name}  labeled {labels[name]['species'] or 'empty'}  ({seconds:.2f} s)")
    print(f"  bird_present   {response.nouls['bird_present'].noul:.2f}")
    print(f"  how_many       {how_many.choice} {how_many.probabilities[how_many.choice]:.2f}")
    bird = response.boxes["bird"]
    print(f"  bird           {edges(bird.box)}  confidence {bird.confidence:.2f}")
    for key in ("species_left", "species_right"):
        answer = response.choices[key]
        print(f"  {key:<15}{answer.choice} {answer.probabilities[answer.choice]:.2f}")
    print()


for name in ("u010.jpg", "u090.jpg", "u097.jpg"):
    show_first_request(name)
```

```
u010.jpg  labeled ['cardinal_male', 'downy_woodpecker']  (0.35 s)
  bird_present   0.98
  how_many       several 1.00
  bird           left 0.13  right 0.42  top 0.58  bottom 0.90  confidence 0.71
  species_left   cardinal_male 1.00
  species_right  downy_woodpecker 1.00

u090.jpg  labeled ['downy_woodpecker']  (0.29 s)
  bird_present   0.95
  how_many       one 1.00
  bird           left 0.33  right 0.44  top 0.43  bottom 0.94  confidence 0.85
  species_left   downy_woodpecker 0.99
  species_right  downy_woodpecker 0.99

u097.jpg  labeled empty  (0.28 s)
  bird_present   0.07
  how_many       none 0.99
  bird           left 0.04  right 0.13  top 0.47  bottom 0.80  confidence 0.14
  species_left   other 0.38
  species_right  other 0.61
```

새가 두 마리인 프레임은 `several`로 나왔고, 홍관조가 왼쪽, 딱따구리가
오른쪽이며, 두 종 답 모두 1.00에 가깝습니다. 새가 한 마리인 프레임에서는 왼쪽 답과 오른쪽
답이 모두 딱따구리를 가리킵니다. 빈 프레임에서는 `bird_present`가 낮으므로 코드가
상자와 종 답을 무시합니다. 이때에도 종 답은 선택지 하나를 고르며, 여기서는 `other`입니다.

새가 두 마리인 프레임에서 `bird` 상자는 홍관조만 덮습니다. Box 질문 하나는 상자를
하나만 반환하므로, 딱따구리에게는 별도의 질문이 필요합니다.

## 종 이름으로 새마다 상자 치기

Box 질문이 두 개 필요합니다. 하나는 "the leftmost bird"(가장 왼쪽 새), 다른 하나는 "the
rightmost bird"(가장 오른쪽 새)를 찾는 질문입니다. 이 모이통에서는 두 상자가 모두 홍관조 위에 놓였습니다.

```python
POSITION_BOXES = {
    "leftmost": Box(instructions="The leftmost bird in `frame`"),
    "rightmost": Box(instructions="The rightmost bird in `frame`"),
}
by_position, _ = ask(FRAMES / "u010.jpg", POSITION_BOXES)
for key in POSITION_BOXES:
    answer = by_position.boxes[key]
    print(f"{key:<11}{edges(answer.box)}  confidence {answer.confidence:.2f}")
```

```
leftmost   left 0.13  right 0.42  top 0.58  bottom 0.91  confidence 0.76
rightmost  left 0.13  right 0.41  top 0.40  bottom 0.88  confidence 0.32
```

두 상자가 프레임의 같은 부분을 덮습니다. 그러니 새마다 종 이름을 지정해서 요청해야 합니다.
어떤 두 종이 있는지는 첫 번째 요청에서 이미 알았으므로, 두 번째 요청에서는
"the male northern cardinal"과 "the downy woodpecker"를 요청합니다. 종마다 쓰는
문구는 `BOX_TARGETS`에 있습니다. 이름만으로는 다른 새에도 들어맞을 수 있는 경우에는 문구에
그 새의 생김새를 덧붙였습니다.

```python
BOX_TARGETS = {
    "blue_jay": "the blue jay",
    "cardinal_male": "the male northern cardinal (the bright red bird)",
    "cardinal_female": "the female northern cardinal (the brown bird with an orange beak)",
    "grackle": "the common grackle (the glossy black bird)",
    "starling": "the European starling (the speckled dark bird)",
    "mourning_dove": "the mourning dove",
    "house_sparrow": "the house sparrow",
    "house_finch": "the house finch",
    "red_winged_blackbird": "the red-winged blackbird",
    "downy_woodpecker": "the downy woodpecker (the black and white bird)",
    "other": "the bird that is none of the listed species",
}


def box_questions(species: list[str]) -> dict:
    return {sp: Box(instructions=f"{BOX_TARGETS[sp]} in `frame`") for sp in species}


by_species, _ = ask(FRAMES / "u010.jpg", box_questions(["cardinal_male", "downy_woodpecker"]))
for key, answer in by_species.boxes.items():
    print(f"{key:<17}{edges(answer.box)}  confidence {answer.confidence:.2f}")
```

```
cardinal_male    left 0.13  right 0.42  top 0.58  bottom 0.92  confidence 0.84
downy_woodpecker left 0.59  right 0.73  top 0.40  bottom 0.88  confidence 0.36
```

이름을 지정한 상자를 쓰면 딱따구리가 홍관조 상자 오른쪽에 자기 상자를 따로 받습니다. 딱따구리 상자의
confidence(신뢰도)는 0.36으로, 홍관조의 0.84보다 낮습니다. 아래 그림은
프레임 위에 두 쌍의 상자를 모두 보여 줍니다.

```python
SURFACE, INK, INK2, MUTED = "#fcfcfb", "#0b0b0b", "#52514e", "#898781"
BLUE, ORANGE = "#2a78d6", "#eb6834"
COLOURS = {"cardinal_male": ORANGE, "downy_woodpecker": BLUE}  # 그 밖의 종은 모두 INK2
FONT = ImageFont.load_default(size=26)


def draw(ax, name: str, boxes: list[tuple[str, dict, str]], title: str) -> None:
    """Draw (label, edges, colour) boxes on a frame and put it on a matplotlib axis."""
    image = PILImage.open(FRAMES / name).convert("RGB")
    width, height = image.size
    pen = ImageDraw.Draw(image)
    for label, e, colour in boxes:
        x0, y0 = e["left"] * width, e["top"] * height
        pen.rectangle([x0, y0, e["right"] * width, e["bottom"] * height], outline=colour, width=6)
        text_box = pen.textbbox((x0, y0), label, font=FONT)
        y = y0 - (text_box[3] - text_box[1]) - 12 if y0 > 40 else y0 + 6
        pen.rectangle([x0, y, x0 + text_box[2] - text_box[0] + 12, y + 36], fill=colour)
        pen.text((x0 + 6, y + 2), label, fill="white", font=FONT)
    ax.imshow(image)
    ax.set_axis_off()
    ax.set_title(title, loc="left", fontsize=10, color=INK2)


def show(fig, filename: str) -> None:
    """Show a figure in a notebook; save it as a PNG when run as a script."""
    if __name__ == "__main__" and "__file__" in globals():
        fig.savefig(filename, dpi=120)
        print(f"saved {filename}")
    else:
        display(fig)
    plt.close(fig)


fig, axes = plt.subplots(1, 2, figsize=(11, 3.6), facecolor=SURFACE)
draw(
    axes[0],
    "u010.jpg",
    [(key, by_position.boxes[key].box.model_dump(), MUTED) for key in POSITION_BOXES],
    "Box by position: both boxes on the cardinal",
)
draw(
    axes[1],
    "u010.jpg",
    [(key.replace("_", " "), a.box.model_dump(), COLOURS[key]) for key, a in by_species.boxes.items()],
    "Box by species name: one box per bird",
)
fig.suptitle("The second request on frame u010 (0:09)", x=0.01, ha="left", color=INK, fontsize=12)
fig.tight_layout()
show(fig, "second-request.png")
```

![출력](https://mintcdn.com/ts-docs/l_j5jNYNp6JTBESY/cookbooks/feeder_watch/feeder_watch.executed.1.png?fit=max\&auto=format\&n=l_j5jNYNp6JTBESY\&q=85\&s=4958af057b8d6595f79deba1a373e01a)

Box 질문은 항상 상자를 반환합니다. 모이통 뒤에 숨은 새처럼 프레임에 거의 보이지 않는 종을
첫 번째 요청이 지목하더라도 두 번째 요청은 그 종의 상자를 반환하며,
이 상자가 다른 새 위에 놓일 수 있습니다. 이름을 지정한 두 상자가
0.6 이상 겹치면 같은 새를 두 번 찾은 것으로 보고, confidence가 더 높은 상자만
남깁니다. 여기서 겹침 정도는 intersection over union(합집합 대비 교집합 비율)입니다. 두 상자가 공유하는 면적을
두 상자가 덮는 전체 면적으로 나눈 값입니다.

```python
OVERLAP_AT = 0.6  # 이름을 지정한 두 상자가 이만큼 겹치면 같은 새를 두 번 찾은 것입니다


def overlap(a: dict, b: dict) -> float:
    """Intersection over union of two boxes given as edge fractions."""
    w = max(0.0, min(a["right"], b["right"]) - max(a["left"], b["left"]))
    h = max(0.0, min(a["bottom"], b["bottom"]) - max(a["top"], b["top"]))
    area = lambda e: (e["right"] - e["left"]) * (e["bottom"] - e["top"])  # noqa: E731
    union = area(a) + area(b) - w * h
    return w * h / union if union > 0 else 0.0


def dedupe(birds: list[dict]) -> list[dict]:
    kept: list[dict] = []
    for bird in sorted(birds, key=lambda b: -b["box_confidence"]):
        if all(overlap(bird["box"], k["box"]) < OVERLAP_AT for k in kept):
            kept.append(bird)
    return kept
```

## 모든 프레임에 대해 묻기

프레임마다 다음과 같이 처리합니다.

1. 질문 다섯 개를 보냅니다. `bird_present`가 0.5 미만이거나 두 종 답 중
   어느 것도 0.5에 미치지 못하면 여기서 멈춥니다.
2. `how_many`가 `several`이고 두 종 답이 서로 다른 종을 가리키면,
   종마다 상자 하나씩을 요청하는 두 번째 요청을 보낸 다음 상자들에 `dedupe()`를 실행합니다. 한쪽
   답만 0.5에 이를 때도 똑같이 합니다. `bird` 상자가 다른 새를
   둘러싸고 있을 수 있기 때문입니다.
3. 그 밖의 경우에는 가장 확률이 높은 종과 첫 번째 요청의 `bird` 상자를 그대로 씁니다.

프레임 하나의 결과에는 새마다의 종, 종 확률, 상자, 상자 confidence가 들어가고,
여기에 요청 수와 그 프레임에 걸린 시간(초)이 더해집니다.

```python
PRESENT_AT = 0.5  # 이 값 미만이면 빈 프레임으로 보고 나머지 답을 무시합니다
SPECIES_AT = 0.5  # 이 값 미만인 종 답은 상자에 이름을 지정하기에는 너무 불확실합니다


def ask_frame(path: Path) -> dict:
    first, seconds = ask(path, QUESTIONS)
    result = {
        "frame": path.name,
        "bird_present": first.nouls["bird_present"].noul,
        "how_many": first.choices["how_many"].choice,
        "birds": [],
        "requests": 1,
        "seconds": seconds,
    }
    # 가장 왼쪽 답과 가장 오른쪽 답이 가리킨 종과, 종마다의 가장 높은 확률
    named: dict[str, float] = {}
    sure_sides = 0
    for key in ("species_left", "species_right"):
        answer = first.choices[key]
        p = answer.probabilities[answer.choice]
        if p >= SPECIES_AT:
            named[answer.choice] = max(p, named.get(answer.choice, 0.0))
            sure_sides += 1
    if result["bird_present"] < PRESENT_AT or not named:
        return result

    # 새가 여러 마리일 때 `bird` 상자는 양쪽이 같은 종을 가리킬 때만 맞습니다
    if result["how_many"] == "several" and (len(named) == 2 or sure_sides < 2):
        second, more_seconds = ask(path, box_questions(list(named)))
        result["requests"] = 2
        result["seconds"] += more_seconds
        boxes = {sp: second.boxes[sp] for sp in named}
    else:
        top = max(named, key=named.get)
        named, boxes = {top: named[top]}, {top: first.boxes["bird"]}

    birds = [
        {"species": sp, "p": p, "box": boxes[sp].box.model_dump(), "box_confidence": boxes[sp].confidence}
        for sp, p in named.items()
    ]
    result["birds"] = dedupe(birds)
    return result
```

프레임은 서로 의존하지 않으므로 스레드 네 개가 병렬로 질문합니다. 그래도
결과는 프레임 순서대로 돌아옵니다. `pool.map`이 입력 순서를 유지하기 때문입니다. 실시간
카메라에서는 그 대신 프레임을 한 번에 한 장씩, 앞선 답이 돌아오는 즉시 다음 프레임을
보내세요.

```python
with ThreadPoolExecutor(max_workers=4) as pool:  # 프레임 101개, 요청 약 180개, 한 번에 4개씩
    results = list(pool.map(ask_frame, paths))

two_requests = [r for r in results if r["requests"] == 2]
print(f"{len(results)} frames, {sum(r['requests'] for r in results)} requests")
print(f"frames that needed the second request: {len(two_requests)}")
print(f"frames that ended with two boxes:      {sum(len(r['birds']) == 2 for r in results)}")
print(f"seconds per frame, median: {statistics.median(r['seconds'] for r in results):.2f}")
```

```
101 frames, 178 requests
frames that needed the second request: 77
frames that ended with two boxes:      66
seconds per frame, median: 0.54
```

일부 프레임은 두 번째 요청을 보내고도 상자 하나로 끝났습니다. 이런 프레임에서는
종 답 하나만 0.5에 이르렀거나, 이름을 지정한 두 상자가 겹쳐서 `dedupe()`가 하나만
남겼습니다.

## 답을 방문으로 바꾸기

방문은 프레임 순서대로 놓인 결과에서 종별로 하나씩 만들어집니다. 그래서 새 두 마리가
모이통에 동시에 있으면 서로 겹치는 방문 두 개가 생깁니다. 무엇을 방문으로 볼지는 세 가지
설정으로 정합니다.

* 어떤 종의 확률이 `species_at` 이상이면 그 프레임은 그 종의 방문에 포함됩니다.
* 새가 잠깐 화면 밖으로 뛰어나갈 때처럼 `max_gap` 프레임 이하의 공백이 있어도 방문은
  계속 열려 있습니다.
* 방문이 되려면 그 종을 가리키는 프레임이 `min_frames`개 이상 있어야 합니다. 기본값이 2이므로
  프레임 하나에서 나온 틀린 답은 방문이 되지 않습니다.

이 설정들은 애플리케이션에 속한 값이며, Jev는 이 값을 전혀 보지 못합니다. 다른 값을 시험하려면
저장된 결과에 `visits()`를 다시 실행하면 됩니다. 요청을 새로 보낼 필요가 없습니다.

```python
def visits(results: list[dict], species_at=0.6, max_gap=2, min_frames=2) -> list[dict]:
    done, open_visits, last_seen = [], {}, {}
    for second, result in enumerate(results):  # 1초에 한 프레임
        seen = {b["species"] for b in result["birds"] if b["p"] >= species_at}
        for sp in list(open_visits):
            if sp not in seen and second - last_seen[sp] > max_gap:
                done.append(open_visits.pop(sp))
        for sp in sorted(seen):
            if sp in open_visits:
                open_visits[sp]["end"] = second
                open_visits[sp]["frames"] += 1
            else:
                open_visits[sp] = {"species": sp, "start": second, "end": second, "frames": 1}
            last_seen[sp] = second
    done.extend(open_visits.values())
    kept = [v for v in done if v["frames"] >= min_frames]
    return sorted(kept, key=lambda v: (v["start"], v["species"]))


def clock(second: int) -> str:
    return f"{second // 60}:{second % 60:02d}"


def print_visits(found: list[dict]) -> None:
    for v in found:
        seconds = v["end"] - v["start"] + 1
        print(f"  {v['species']:<18}{clock(v['start'])} to {clock(v['end'])}  ({seconds} s)")


found = visits(results)
print("Visits:")
print_visits(found)

per_species: dict[str, list[int]] = {}
for v in found:
    count = per_species.setdefault(v["species"], [0, 0])
    count[0] += 1
    count[1] += v["end"] - v["start"] + 1
print("\nPer species:")
for sp, (n, seconds) in sorted(per_species.items(), key=lambda kv: -kv[1][1]):
    print(f"  {sp:<18}{n} visit{'s' if n > 1 else ''}, {seconds} s at the feeder")
print("\nVisits in the hand labels:")
print_visits(visits([{"birds": [{"species": s, "p": 1.0} for s in labels[n]["species"]]} for n in NAMES]))
```

```
Visits:
  cardinal_male     0:00 to 1:02  (63 s)
  downy_woodpecker  0:00 to 1:29  (90 s)
  cardinal_male     1:12 to 1:15  (4 s)

Per species:
  downy_woodpecker  1 visit, 90 s at the feeder
  cardinal_male     2 visits, 67 s at the feeder

Visits in the hand labels:
  cardinal_male     0:00 to 1:02  (63 s)
  downy_woodpecker  0:00 to 1:29  (90 s)
```

타임라인으로 그리면 방문 기록의 각 행은 종 하나, 각 막대는 방문 하나이며,
행 옆에는 모이통에 머문 총 시간이 표시됩니다.

```python
def draw_visits(found: list[dict], seconds: int, title: str) -> None:
    by_species: dict[str, list[dict]] = {}
    for v in found:
        by_species.setdefault(v["species"], []).append(v)
    total = {sp: sum(v["end"] - v["start"] + 1 for v in vs) for sp, vs in by_species.items()}
    order = sorted(by_species, key=lambda sp: -total[sp])

    height = 0.45 * len(order) + 1.4
    fig, (ax, bx) = plt.subplots(
        1, 2, figsize=(11, height), facecolor=SURFACE,
        gridspec_kw={"width_ratios": [3.6, 1], "wspace": 0.04},
    )
    for a in (ax, bx):
        a.set_facecolor(SURFACE)
        a.spines[["top", "right", "left"]].set_visible(False)
        a.tick_params(colors=INK2, labelsize=9, length=0)
    for row, sp in enumerate(order):
        colour = MUTED if sp == "other" else BLUE
        spans = [(v["start"], v["end"] - v["start"] + 1) for v in by_species[sp]]
        ax.broken_barh(spans, (row - 0.32, 0.64), facecolors=colour)
        bx.barh(row, total[sp], height=0.64, color=colour)
        n = len(by_species[sp])
        label = f"{total[sp]} s, {n} visit{'s' if n > 1 else ''}"
        bx.text(total[sp] + seconds * 0.03, row, label, va="center", fontsize=9, color=INK2)
    ax.set_yticks(range(len(order)), [sp.replace("_", " ") for sp in order], color=INK, fontsize=10)
    ax.set_xlim(0, seconds)
    ax.set_xticks(range(0, seconds + 1, 10), [clock(s) for s in range(0, seconds + 1, 10)])
    ax.grid(axis="x", color="#e4e3df")
    ax.set_axisbelow(True)
    ax.set_title("Each visit, by time of footage (m:ss)", loc="left", fontsize=10, color=INK2)
    bx.set_xlim(0, seconds * 1.6)
    bx.set_xticks([])
    bx.set_yticks([])
    bx.spines["bottom"].set_visible(False)
    bx.set_title("Time at the feeder", loc="left", fontsize=10, color=INK2)
    for a in (ax, bx):
        a.set_ylim(len(order) - 0.5, -0.5)
    fig.suptitle(title, x=0.01, ha="left", fontsize=12, color=INK)
    # 여백(인치 단위): 왼쪽에는 종 이름, 위에는 제목, 아래에는 시각
    fig.subplots_adjust(left=1.6 / 11, right=0.99, top=1 - 0.75 / height, bottom=0.4 / height)
    show(fig, "visits.png")


draw_visits(found, len(results), f"{len(found)} visits in {clock(len(results))} of footage")
```

![출력](https://mintcdn.com/ts-docs/l_j5jNYNp6JTBESY/cookbooks/feeder_watch/feeder_watch.executed.2.png?fit=max\&auto=format\&n=l_j5jNYNp6JTBESY\&q=85\&s=aeeedf9c6c369a3f23c09abb4d961cb1)

긴 방문 두 개는 저희가 직접 붙인 레이블로 같은 방식으로 만든 방문 기록과 일치하며,
시작 초와 끝 초도 같습니다. 이 클립으로 실행할 때마다 긴 방문 두 개는 똑같이
나왔지만, 짧은 방문은 실행마다 달라졌습니다.

여기 나온 짧은 방문은 틀렸습니다. 1:12부터 1:15까지의 수컷 홍관조 방문은 사실 모이통 뒤에 있던 암컷
홍관조이며, 붉은 부리만 보였을 때 종을 잘못 짚은 것입니다.

`min_frames`를 1로 낮추면 기본값 2일 때 기록에서 빠지는 단일 프레임 관찰이
드러납니다.

```python
logged = {(v["species"], v["start"]) for v in visits(results)}
print("Single-frame sightings that min_frames=2 leaves out:")
print_visits([v for v in visits(results, min_frames=1) if (v["species"], v["start"]) not in logged])
```

```
Single-frame sightings that min_frames=2 leaves out:
  cardinal_female   1:04 to 1:04  (1 s)
  cardinal_female   1:08 to 1:08  (1 s)
  other             1:16 to 1:16  (1 s)
  cardinal_male     1:34 to 1:34  (1 s)
```

이 가운데에는 1:04와 1:08의 암컷 홍관조처럼 실제로 있던 새도 있고,
1:34의 수컷 홍관조처럼 잘못 읽은 것도 있습니다. `min_frames`를 높이면 틀린 답이 기록에 덜
들어가지만, 1초나 2초만 나타나는 새도 더 많이 빠집니다.

## 답을 레이블과 대조하기

두 가지 검사로 답을 직접 붙인 레이블과 비교합니다.

* 프레임에 새가 있는지에 대해 `bird_present`가 레이블과 일치하는지
* 레이블에 있는 모든 새의 종이 `ask_frame()`이 반환한 종에 포함되는지

`partial`로 표시된 프레임은 첫 번째 검사에서 새가 있는 것으로 셉니다. 두 번째 검사에서는
제외합니다. 레이블에 그 프레임의 모든 새가 적혀 있지는 않기 때문입니다.

```python
present_ok, found, total, misses = 0, 0, 0, []
for r in results:
    label = labels[r["frame"]]
    has_bird = bool(label["species"]) or label["partial"]  # 일부만 보이는 새도 새입니다
    if (r["bird_present"] >= PRESENT_AT) == has_bird:
        present_ok += 1
    else:
        misses.append(f"{r['frame']}  bird_present {r['bird_present']:.2f}, labeled {label['species'] or 'empty'}")
    if label["partial"]:
        continue
    returned = {b["species"] for b in r["birds"]}
    for sp in label["species"]:
        total += 1
        if sp in returned:
            found += 1
        else:
            misses.append(f"{r['frame']}  labeled {sp}, returned {sorted(returned) or 'nothing'}")

print(f"bird_present agrees with the labels: {present_ok}/{len(results)} frames")
print(f"labeled birds returned:              {found}/{total}")
print("\nDisagreements:")
print("\n".join(misses) or "none")
```

```
bird_present agrees with the labels: 101/101 frames
labeled birds returned:              132/133

Disagreements:
u063.jpg  labeled downy_woodpecker, returned ['cardinal_male']
```

`bird_present`에 가장 어려운 프레임은 홍관조의 일부만 보이는 1:33과 1:34이며,
아래 그림의 왼쪽 절반이 1:33입니다. Jev는 둘 다 약 0.8로 점수를 매겨 새가 있는 것으로
셌고, 레이블은 둘 다 `partial`로 표시되어 있습니다.

레이블에 있는데 Jev가 놓친 새는 1:02의 딱따구리 하나이며, 수컷 홍관조와 같은 프레임에
있습니다. 이 프레임에서는 수컷 홍관조만 반환되었습니다.

그림의 오른쪽 절반은 `dedupe()`가 잡아내지 못하는 실패를 보여 줍니다. 1:15에
암컷 홍관조의 부리가 케이지 오른쪽 가장자리에 보이는데, Jev는 이를 수컷 홍관조라고 답했습니다.
수컷 홍관조의 상자는 딱따구리 위에 놓였습니다. 이 상자가 딱따구리 자신의 상자와 겹치는
정도는 0.6 미만이어서 두 상자가 모두 남습니다. 상자마다 검사를 하나씩 두면 이런 경우를 잡을 수 있습니다. 상자대로
잘라 낸 다음, 잘라 낸 이미지 속 새가 수컷 홍관조인지 `Noul` 질문으로 묻는 것입니다.

```python
by_frame = {r["frame"]: r for r in results}


def answer_boxes(name: str) -> list[tuple[str, dict, str]]:
    return [
        (f"{b['species'].replace('_', ' ')} {b['p']:.2f}", b["box"], COLOURS.get(b["species"], INK2))
        for b in by_frame[name]["birds"]
    ]


fig, axes = plt.subplots(1, 2, figsize=(11, 3.6), facecolor=SURFACE)
for ax, name in zip(axes, ("u094.jpg", "u076.jpg")):
    r = by_frame[name]
    second = int(name[1:4]) - 1
    draw(ax, name, answer_boxes(name), f"{clock(second)}  bird_present {r['bird_present']:.2f}")
fig.suptitle("Two hard frames from the run", x=0.01, ha="left", color=INK, fontsize=12)
fig.tight_layout()
show(fig, "failures.png")
```

![출력](https://mintcdn.com/ts-docs/l_j5jNYNp6JTBESY/cookbooks/feeder_watch/feeder_watch.executed.3.png?fit=max\&auto=format\&n=l_j5jNYNp6JTBESY\&q=85\&s=deac8de512d669ca99426e3192c0b361)

## 실시간으로 동작하는 모습 보기

같은 질문을 실시간 영상에도 쓸 수 있습니다. 아래 영상은 이 질문들로 만든 작은 웹 앱을
보여 줍니다. 앱은 웹캠이나 영상 파일에서 프레임을 가져와 Jev에 묻고,
상자를 그리고, 방문 기록을 계속 갱신합니다. 영상에서는 질문을 조정할 때 쓰지 않은
Ostdrossel 모이통 카메라 영상의 한 구간을 재생하며, 1초에 답을 두 개 정도 받습니다.

[새 모이통 관찰 웹 앱 데모](https://player.vimeo.com/video/1237184729)

새 영상: Ostdrossel의 [Ostdrossel feeder cam](https://www.youtube.com/watch?v=h6iNvPiyPbI),
[CC BY](https://creativecommons.org/licenses/by/3.0/) 라이선스.
상자와 레이블은 앱이 덧붙인 것입니다.

## 여러분의 카메라에 적용하기 전에

* **Box 질문 하나는 상자를 하나만 반환합니다.** 이 코드는 프레임당 최대 두 마리까지만,
  그것도 서로 다른 종일 때만 상자를 칩니다. 종이 더 많다면 종마다
  "Is there a blue jay in `frame`?" 같은 `Noul` 질문을 하나씩 던지고, 답이 높게 나온 종마다
  이름을 지정한 Box를 요청하세요.
* **같은 종의 새 두 마리는 이름으로 구별할 수 없습니다.** 무리를 다룰 때는 Meta의 SAM 3 같은
  zero-shot detector(제로샷 탐지기)를 쓰면 "bird"라는 텍스트 prompt(프롬프트)만으로 학습 없이 모든 새를
  찾을 수 있습니다. 그런 다음 잘라 낸 새마다 Jev가 종 이름을 붙이면 됩니다.
* **작은 프레임은 거부됩니다.** [이미지](../images.md) 페이지에 따르면 각 변은
  280에서 1568 픽셀 사이여야 하고, 전체 픽셀 수는 78,400개(280 × 280) 이상이어야 합니다.
* **필요에 따라 threshold(임계값)를 조정하세요.** `PRESENT_AT`, `SPECIES_AT`,
  `OVERLAP_AT`의 값과 `visits()`의 기본값은 이 영상에서 잘 맞았던 값입니다. 여러분의 영상
  몇 분 분량에 레이블을 붙이고 이 값들을 대조해 보세요.
* **여러분의 카메라에 맞게 종 목록을 작성하세요.** 클로즈업을 포함해 카메라에 각 새가 어떻게
  찍히는지 설명하세요. `other` 선택지는 남겨 두세요. 그래야 목록에 없는 새가
  목록에 있는 이름 중 하나로 억지로 분류되지 않습니다.


이 문서는 개발자 문서 플랫폼인 [Mintlify](https://mintlify.com)로 제작하고 호스팅합니다.
