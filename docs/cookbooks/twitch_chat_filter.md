# Twitch 채팅 필터

> Twitch 채팅에서 모욕, 스팸, 요청하지 않은 게임 플레이 조언은 숨기고, 스포일러는 흐리게 처리하고, 질문은 강조합니다. 메시지마다 질문 세 개를 담은 TypeSafe 요청을 하나씩 보내고, 요청은 동시에 실행하며, 코드에 적은 단순한 규칙이 답을 시청자에게 보이는 화면으로 바꿉니다.

Twitch 방송을 보고 있는데 채팅이 읽는 속도보다 빠르게 올라갑니다. 이 cookbook(예제 모음)에서는 메시지가 도착하는 대로 분류합니다. 질문은 강조하고, 스팸과 모욕, 요청하지 않은 게임 플레이 조언은 숨기고, 스포일러는 흐리게 처리합니다. 키워드 목록으로 스팸 링크는 잡을 수 있겠지만, "go left, there's a site of grace behind the tree"(왼쪽으로 가, 나무 뒤에 축복의 장소가 있어)에는 나쁜 단어가 하나도 없고, 줄거리 반전을 말하는 메시지도 마찬가지입니다.

메시지마다 방송 제목과 게임을 함께 담아 별도의 요청으로 Jev에 보냅니다. 메시지에 대해 세 가지를 묻습니다. 어떤 종류의 메시지인지, 유해한지(모욕, 혐오, 괴롭힘), 게임 내용을 스포일러하는지입니다. 그런 다음 단순한 규칙 몇 개가 이 답을 시청자에게 보이는 화면으로 바꿉니다. [LLM을 위한 guardrail](llm_guardrails.md)에서도 같은 패턴으로 LLM에 들어가고 나오는 메시지를 걸러 냅니다.

다음 데모에서는 Jev 기반 Chrome 확장 프로그램이 실제 Twitch 실시간 채팅을 필터링합니다.

[Twitch 채팅 필터링 Chrome 확장 프로그램 데모](https://player.vimeo.com/video/1230896366)

여기서는 Elden Ring 방송을 가정하고 작성한 채팅 메시지 29개를 필터에 통과시킵니다. 필터는 일곱 개를 숨기고, 두 개를 흐리게 처리하고, 여섯 개를 질문으로 강조합니다.

![채팅 메시지 다섯 개가 각각 방송 중인 게임 정보와 함께 별도의 요청으로 Jev에 한꺼번에 전달됩니다. category, toxic, spoiler 답이 돌아오면 decide()가 시청자의 채팅 창에서 각 메시지를 표시하거나, 강조하거나, 숨기거나, 흐리게 처리합니다](https://mintcdn.com/ts-docs/kfHWxZoHUEHHJ394/cookbooks/twitch_chat_filter/twitch-chat-filter-hero-light.gif?s=1a332412668d6e213c93b2112ad49dc1)

함수 두 개를 만듭니다. `classify()`는 메시지 하나에 대해 세 가지 질문을 던지고, `decide()`는 그 답을 표시, 강조, 흐리게 표시(dim), 숨김, 블러(blur) 중 하나로 바꿉니다. 스팸을 숨기는 규칙처럼 규칙 하나를 바꿀 때는 `decide()`만 다시 실행하면 되고, Jev에 새 요청을 보낼 필요가 없습니다.

## 준비

```bash
pip install 'typesafe-sdk>=0.7.1,<0.8.0' ipython 'cooksafe>=0.3.0,<0.4.0'
```

`twitch_chat_filter.py`라는 새 파일을 만듭니다. 이 cookbook을 따라가면서 Python 코드 블록을 순서대로 이 파일에 복사하세요. 실시간으로 실행하려면 환경 변수에 `TYPESAFE_API_KEY`를 설정합니다. 블록을 모두 추가했으면 파일이 있는 디렉터리에서 다음 명령을 실행합니다.

```bash
python twitch_chat_filter.py
```

아래 수치는 2026-10-29에 `jev-1.14.0`으로 얻은 결과입니다.

```python
from concurrent.futures import ThreadPoolExecutor
import os
import urllib.request
from pathlib import Path
from time import perf_counter

from cooksafe import JsonCache, make_playground_link
from IPython.display import Markdown, display
from typesafe_sdk import Choice, Noul, NoulCriteria, TypeSafeClient

TYPESAFE_MODEL = "jev-1.14.0"  # 이 페이지에서 쓰는 모델. "jev-latest"는 최신 모델을 실시간으로 실행합니다

client = TypeSafeClient(
    api_key=os.environ.get("TYPESAFE_API_KEY", "cache-only"),  # 캐시로 다시 렌더링할 때는 키가 필요 없습니다
    base_url=os.environ.get("TYPESAFE_ENDPOINT"),
    timeout=60.0,
)
CACHE_DIR = Path(__file__).resolve().parent if "__file__" in globals() else Path.cwd()

EXAMPLES_URL = (
    "https://raw.githubusercontent.com/typesafe-ai/typesafe-public-examples/main/"
    "cookbooks/twitch_chat_filter"
)
CACHE = CACHE_DIR / "twitch_chat_filter_cache.json"
# True이면 공개된 답으로 CACHE를 채우므로 실행 중에 요청을 보내지 않습니다. 실시간으로
# 실행하려면 False로 바꾸고 CACHE를 삭제하세요. 디스크에 이미 있는 CACHE는 덮어쓰지 않습니다.
DOWNLOAD_CACHE = True
if DOWNLOAD_CACHE and not CACHE.exists():
    urllib.request.urlretrieve(f"{EXAMPLES_URL}/{CACHE.name}", CACHE)
json_cache = JsonCache(CACHE)
```

## 채팅 일부 불러오기

모든 종류의 메시지가 적어도 두 번씩 나오도록 메시지는 직접 작성했습니다. 그중 두 개는 emote(이모트)만 있는 줄입니다. emote는 Twitch에서 쓰는 작은 채팅 이미지로, `PogChamp` 같은 단어로 입력합니다.

메시지가 스포일러인지, backseat advice(훈수: 스트리머가 요청하지 않은 게임 플레이 조언)인지는 어떤 게임을 하고 있는지에 따라 달라지므로, 모든 요청에 방송 제목과 게임을 넣습니다.

```python
STREAM = {"title": "Elden Ring first playthrough! No spoilers please", "game": "Elden Ring"}

CHAT = [
    ("night-owl-22", "hi chat, first time catching a stream live"),
    ("ggwp-andy", "PogChamp PogChamp PogChamp"),
    ("mira-plays", "how many hours have you played so far?"),
    ("tarnished-tom", "this area looks so good at night"),
    ("backseat-bert", "go left, there's a site of grace behind the tree"),
    ("follow4follow-bot", "Want more viewers? Buy 1000 followers for $5 at cheapviewz dot example"),
    ("lore-hound", "wait till you find out Ranni planned the Night of Black Knives"),
    ("kappa-kev", "LUL LUL LUL"),
    ("sunny-skies", "are you going to do a no-hit run after this?"),
    ("rage-ryan", "you're genuinely the worst player i've ever seen, uninstall trash"),
    ("cozy-cat", "the music in this area is so good"),
    ("backseat-bert", "use your flask NOW, why are you not healing"),
    ("mira-plays", "W streamer"),
    ("ggwp-andy", "GG that was insane"),
    ("quiet-quinn", "anyone else watching from Brazil?"),
    ("tarnished-tom", "what build are you going for?"),
    ("lore-hound", "the ending where you become Elden Lord with Ranni is the best one"),
    ("kappa-kev", "chat is wild today lol"),
    ("backseat-bert", "the boss is weak to fire, equip the torch before you go in"),
    ("rage-ryan", "nobody cares about your opinion, go back to your basement"),
    ("cozy-cat", "you're so bad at this boss lmao, love you tho"),
    ("sunny-skies", "who is the final boss?"),
    ("ggwp-andy", "LETS GOOOO"),
    ("night-owl-22", "i just got this game too, it's so hard"),
    ("promo-pete", "check out my channel, I stream every day, follow me!"),
    ("quiet-quinn", "dodge rolling through that was clean"),
    ("mira-plays", "is the DLC worth it?"),
    ("cozy-cat", "your character looks so cool"),
    ("night-owl-22", "this boss took me 40 tries lol"),
]

print(f"{len(CHAT)} messages. What one request sends as its state:")
print({"stream": STREAM, "message": {"user": CHAT[4][0], "text": CHAT[4][1]}})
```

```
29 messages. What one request sends as its state:
{'stream': {'title': 'Elden Ring first playthrough! No spoilers please', 'game': 'Elden Ring'}, 'message': {'user': 'backseat-bert', 'text': "go left, there's a site of grace behind the tree"}}
```

## 질문 작성하기

요청 하나가 같은 state(상태), 즉 방송 정보와 메시지 하나를 놓고 세 가지 질문을 던집니다.

* `category` 질문은 메시지 종류 다섯 가지를 대상으로 하는 `Choice`입니다. 여기서 고른 라벨이 메시지를 어떻게 처리할지 결정합니다. 표시, 강조, 흐리게 표시, 숨김 중 하나입니다.
* `toxic` 질문은 `Noul`입니다. 확률이 threshold(임계값) `TOXIC_AT`에 도달하면 category가 무엇이든 메시지를 숨깁니다.
* `spoiler` 질문은 `Noul`입니다. 확률이 threshold `SPOILER_AT`에 도달하면 메시지를 블러 처리합니다. 단, 이미 숨겨진 메시지는 제외합니다.

category는 메시지의 주된 목적을 나타냅니다. 반복되는 emote와 응원은 `hype`로, 요청하지 않은 홍보는 `spam`으로 봅니다. 링크가 있다는 것만으로는 스팸이 아닙니다.
`conversation`은 다른 category가 모두 맞지 않을 때 쓰는 기본값입니다.

```python
QUESTIONS = {
    "category": Choice(
        instructions="Which category best describes the main purpose of `message.text`?",
        criteria={
            "question": "A genuine request for information from the streamer or chat, "
            "not gameplay instructions phrased as a question or a promotional pitch",
            "hype": "Cheering, reactions, or repeated emotes (PogChamp, LUL, GG, W, "
            "lets gooo); repetition alone does not make cheering spam",
            "conversation": "Ordinary chatter, opinions, replies, useful shared links, "
            "or any message whose main purpose does not fit the other categories",
            "backseat": "Unrequested gameplay advice telling the streamer what to do, "
            "including instructions phrased as a question; not a request for information",
            "spam": "Unsolicited advertising, self-promotion, follower selling, or "
            "promotional links; not ordinary cheering, repeated emotes, or a useful link",
        },
    ),
    "toxic": Noul(
        instructions="Is this message insulting, hateful or harassing towards someone?",
        criteria=NoulCriteria(
            true="Insults, slurs, threats or personal attacks on the streamer or another viewer",
            false="Friendly, neutral, or light banter with no target",
        ),
    ),
    "spoiler": Noul(
        instructions="Does this message reveal a plot point or ending of the game being played?"
    ),
}
```

## 메시지 하나 분류하기

메시지 하나가 요청 하나가 됩니다. 메시지로 state를 만들고 세 가지 질문과 함께 보냅니다. 캐시 키는 state와 질문을 JSON으로 풀어 쓴 값이고, 각 답은 이 키 아래에 저장됩니다. 질문을 수정하면 키가 바뀌므로, 다음 실행에서는 이전 답을 재생하지 않고 새 요청을 보냅니다.

```python
def classify(user: str, text: str, stream: dict | None = None) -> dict:
    state = {
        "stream": STREAM if stream is None else stream,
        "message": {"user": user, "text": text},
    }
    questions = {key: q.model_dump(mode="json", exclude_none=True) for key, q in QUESTIONS.items()}
    return _ask(state, questions, model=TYPESAFE_MODEL)


@json_cache
def _ask(state: dict, questions: dict, model: str) -> dict:
    if not os.environ.get("TYPESAFE_API_KEY"):
        raise RuntimeError("Set TYPESAFE_API_KEY to send requests to TypeSafe.")
    started = perf_counter()
    response = client.system_one(state=state, questions=questions, model=model)
    category = response.answers["category"]
    return {
        "category": category.choice,
        "probabilities": dict(category.probabilities),
        "toxic": response.answers["toxic"].noul,
        "spoiler": response.answers["spoiler"].noul,
        "seconds": round(perf_counter() - started, 2),
    }


def show(user: str, text: str) -> None:
    result = classify(user, text)
    print(f"{user}: {text}")
    for label, p in sorted(result["probabilities"].items(), key=lambda kv: -kv[1]):
        print(f"  {label:<13}{p:5.2f}  {'#' * round(p * 30)}".rstrip())
    print(f"  {'toxic':<13}{result['toxic']:5.2f}")
    print(f"  {'spoiler':<13}{result['spoiler']:5.2f}\n")


show("backseat-bert", "go left, there's a site of grace behind the tree")
show("lore-hound", "wait till you find out Ranni planned the Night of Black Knives")
show("cozy-cat", "you're so bad at this boss lmao, love you tho")
```

```
backseat-bert: go left, there's a site of grace behind the tree
  backseat      1.00  ##############################
  question      0.00
  hype          0.00
  conversation  0.00
  spam          0.00
  toxic         0.03
  spoiler       0.07

lore-hound: wait till you find out Ranni planned the Night of Black Knives
  conversation  0.67  ####################
  backseat      0.32  ##########
  hype          0.01
  question      0.00
  spam          0.00
  toxic         0.04
  spoiler       0.95

cozy-cat: you're so bad at this boss lmao, love you tho
  conversation  0.79  ########################
  hype          0.21  ######
  question      0.00
  backseat      0.00
  spam          0.00
  toxic         0.28
  spoiler       0.02
```

훈수 메시지는 `backseat` 1.00으로 돌아오고 `toxic`과 `spoiler`는 낮습니다. Night of Black Knives 메시지는 0.67 대 0.32로 `backseat`보다 `conversation` 쪽으로 기울고, `spoiler` 질문이 이 메시지를 잡아냅니다. 마지막 메시지는 모욕처럼 읽히지만 `toxic`은 낮게 유지됩니다. 끝에 "love you tho"가 붙어 있어서 질문에 적어 둔 "light banter"(가벼운 농담) 설명에 들어맞기 때문입니다.

## 시청자에게 무엇을 보여 줄지 결정하기

시청자에게 무엇을 보여 줄지 결정할 때는 새 요청이 필요 없습니다. 답만 읽고 세 가지 검사를 순서대로 적용합니다.

1. `TOXIC_AT` 이상인 메시지는 숨깁니다.
2. category의 규칙이 `hide`인 메시지는 숨깁니다.
3. `SPOILER_AT` 이상인 메시지는 블러 처리합니다.

남은 메시지에는 해당 category의 규칙을 적용합니다.

샘플 메시지에서 toxic 확률은 모두 0.97 이상이거나 0.38 이하이고, spoiler 확률은 모두 0.95 이상이거나 0.38 이하입니다. 그래서 threshold를 0.38 초과 0.95 이하의 어떤 값으로 잡아도 여기서는 결과가 같습니다. 실제 채널의 채팅에서는 확률이 이 범위 안에 들어오는 경우가 있을 가능성이 높으니, 실제 방송에 적용하기 전에 두 threshold를 여러분 채널의 채팅으로 확인하세요.

```python
RULES = {
    "question": "highlight",
    "hype": "show",
    "conversation": "show",
    "backseat": "hide",
    "spam": "hide",
}
TOXIC_AT = 0.7  # toxic일 확률이 이 값 이상이면 숨깁니다
SPOILER_AT = 0.7  # 스포일러일 확률이 이 값 이상이면 블러 처리합니다


def decide(result: dict, rules: dict = RULES) -> str:
    if result["toxic"] >= TOXIC_AT:
        return "hide (toxic)"
    action = rules[result["category"]]
    if action == "hide":
        return f"hide ({result['category']})"
    if result["spoiler"] >= SPOILER_AT:
        return "blur (spoiler)"
    return action
```

## 채팅 전체 필터링하기

위의 메시지 세 개는 캐시에 들어가 있으므로, 처음 실시간으로 실행할 때 새로 보낼 요청은 26개입니다.
어떤 메시지도 다른 메시지에 의존하지 않으므로 요청은 병렬로 실행되고, `pool.map`은 답을 채팅 순서대로 반환합니다.

아래에는 채팅이 시청자에게 보이는 모습대로 출력됩니다. 숨긴 메시지는 숨긴 이유만 보이고, 블러 처리한 메시지는 작성자만 보이며, 강조한 질문에는 `>>` 표시가 붙습니다. 각 줄 오른쪽의 라벨은 `category` 답입니다.

```python
with ThreadPoolExecutor(max_workers=8) as pool:  # 요청 29개, 메시지당 하나씩, 한 번에 8개
    results = list(pool.map(lambda message: classify(*message), CHAT))

counts: dict[str, int] = {}
for (user, text), result in zip(CHAT, results):
    action = decide(result)
    counts[action] = counts.get(action, 0) + 1
    if action.startswith("hide"):
        line = f"   · hidden, {action[6:-1]}"
    elif action == "blur (spoiler)":
        line = f"   {user}: ░░░░░░░░ spoiler"
    else:
        mark = {"highlight": ">>", "dim": "..", "show": "  "}[action]
        line = f"{mark} {user}: {text}"
    print(f"{line[:78]:<80}{result['category']}")

print()
for action, n in sorted(counts.items(), key=lambda kv: -kv[1]):
    print(f"{action:<20}{n:>3}")
```

```text
   night-owl-22: hi chat, first time catching a stream live                     conversation
   ggwp-andy: PogChamp PogChamp PogChamp                                        hype
>> mira-plays: how many hours have you played so far?                           question
   tarnished-tom: this area looks so good at night                              conversation
   · hidden, backseat                                                           backseat
   · hidden, spam                                                               spam
   lore-hound: ░░░░░░░░ spoiler                                                 conversation
   kappa-kev: LUL LUL LUL                                                       hype
>> sunny-skies: are you going to do a no-hit run after this?                    question
   · hidden, toxic                                                              conversation
   cozy-cat: the music in this area is so good                                  conversation
   · hidden, backseat                                                           backseat
   mira-plays: W streamer                                                       hype
   ggwp-andy: GG that was insane                                                hype
>> quiet-quinn: anyone else watching from Brazil?                               question
>> tarnished-tom: what build are you going for?                                 question
   lore-hound: ░░░░░░░░ spoiler                                                 conversation
   kappa-kev: chat is wild today lol                                            conversation
   · hidden, backseat                                                           backseat
   · hidden, toxic                                                              conversation
   cozy-cat: you're so bad at this boss lmao, love you tho                      conversation
>> sunny-skies: who is the final boss?                                          question
   ggwp-andy: LETS GOOOO                                                        hype
   night-owl-22: i just got this game too, it's so hard                         conversation
   · hidden, spam                                                               spam
   quiet-quinn: dodge rolling through that was clean                            hype
>> mira-plays: is the DLC worth it?                                             question
   cozy-cat: your character looks so cool                                       conversation
   night-owl-22: this boss took me 40 tries lol                                 conversation

show                 14
highlight             6
hide (backseat)       3
hide (spam)           2
blur (spoiler)        2
hide (toxic)          2
```

여러분의 채팅에 필터를 적용하려면 `STREAM`과 `CHAT`을 바꾸세요. 채널이 여러 개라면 채널마다 해당 `stream`을 `classify()`에 넘깁니다.

rate limit(요청 한도)에 걸리면 `max_workers`를 낮추고 다시 실행하세요. 이미 받은 답은 캐시에 들어 있습니다. 실시간 연동에서는 요청마다 플랫폼의 메시지 ID나 메시지에 해당하는 페이지 요소의 참조를 함께 저장하세요. 답이 도착하면 이 값으로 올바른 메시지를 찾아 갱신합니다.

## 새 요청 없이 규칙 바꾸기

답은 메시지가 어떤 것인지만 설명하므로, 규칙을 바꾸더라도 앞 단계에서 얻은 결과 29개를 그대로 다시 쓸 수 있습니다. 여기서는 hype(응원과 emote만 있는 줄)를 숨기고 conversation을 흐리게 표시합니다.

```python
QUIET = RULES | {"hype": "hide", "conversation": "dim"}

changed = 0
for (user, text), result in zip(CHAT, results):
    before, after = decide(result), decide(result, QUIET)
    if before != after:
        changed += 1
        print(f"{user + ': ' + text:<62.62}  {before:<10} -> {after}")
print(f"\n{changed} messages changed, 0 new requests")
```

```
night-owl-22: hi chat, first time catching a stream live        show       -> dim
ggwp-andy: PogChamp PogChamp PogChamp                           show       -> hide (hype)
tarnished-tom: this area looks so good at night                 show       -> dim
kappa-kev: LUL LUL LUL                                          show       -> hide (hype)
cozy-cat: the music in this area is so good                     show       -> dim
mira-plays: W streamer                                          show       -> hide (hype)
ggwp-andy: GG that was insane                                   show       -> hide (hype)
kappa-kev: chat is wild today lol                               show       -> dim
cozy-cat: you're so bad at this boss lmao, love you tho         show       -> dim
ggwp-andy: LETS GOOOO                                           show       -> hide (hype)
night-owl-22: i just got this game too, it's so hard            show       -> dim
quiet-quinn: dodge rolling through that was clean               show       -> hide (hype)
cozy-cat: your character looks so cool                          show       -> dim
night-owl-22: this boss took me 40 tries lol                    show       -> dim

14 messages changed, 0 new requests
```

## 실제 채널에 쓰기 전에

* **메시지마다 답을 기다려야 합니다.** 이 cookbook의 요청은 중앙값 기준으로 약 0.1초가 걸렸으므로, 새 줄이 채팅에 나타난 뒤에야 답이 돌아옵니다. 답이 도착할 때까지 새 줄을 숨겨 두거나, 유해한 메시지가 그 시간만큼 화면에 보이는 것을 감수해야 합니다.
* **요청을 한 번에 하나씩 보내서는 바쁜 채팅을 따라갈 수 없습니다.** *채팅 전체 필터링하기*에서처럼 여러 요청을 동시에 실행하세요.
* **1위를 한 category도 불확실할 수 있습니다.** 여기서는 항상 1위 라벨의 규칙을 적용합니다. 실제로 쓰기 전에, 상위 두 category의 확률이 서로 비슷한 메시지와 Noul 값이 0.5 근처인 메시지를 어떻게 처리할지 정하세요. 이런 메시지는 자동으로 숨기는 대신 검토 대상으로 표시하거나 그대로 보이게 둘 수 있습니다. 이 정책을 여러분 채널의 채팅으로 테스트하세요.
* **스포일러 검사는 스트리머가 어디까지 진행했는지 알지 못합니다.** 게임 제목과 모델이 알고 있는 게임 지식을 바탕으로 줄거리를 드러낼 가능성이 높은 메시지를 표시할 뿐입니다. 해당 시청자나 스트리머가 그 내용을 이미 아는지는 판단할 수 없습니다. 아직 보지 못한 줄거리만 걸러 내려면 현재 진행 상황과 관련 줄거리 정보를 제공하고, 질문이 그 맥락을 활용하도록 고치세요.

## 플레이그라운드에서 열어 보기

아래 링크를 누르면 훈수 메시지의 state와 세 가지 질문이 채워진 상태로 플레이그라운드가 열립니다. 거기서 호출을 실시간으로 실행하거나, 질문을 수정해서 다시 실행해 보세요.

```python
playground_link = make_playground_link(
    {"stream": STREAM, "message": {"user": CHAT[4][0], "text": CHAT[4][1]}},
    QUESTIONS,
)
# 스크립트에서는 바로 쓸 수 있는 URL을 출력하고, 노트북에서는 클릭할 수 있는 링크를 렌더링합니다.
if __name__ == "__main__" and "__file__" in globals():
    print(f"Open in the TypeSafe playground:\n{playground_link}")
else:
    display(Markdown(f"🔗 [Open in the TypeSafe playground]({playground_link})"))
```

[TypeSafe 플레이그라운드에서 열기 →](https://console.typesafe.ai/playground#share/N4IgJg9gxgrgtgUwHYBcAqCAeKQC4AEIwAOiAM4oBOCAhnKQSSCgJYoA2CD+pAou2GT4ASiyQBzfADMWlCvgAO7GgE8UAC0oQY49QEJ8AOQj4yCiC05zFnGmS4gANDxDi6DgnwFDRE0gF9nUkQyMhpxD3wmGHtKblIAIxooAGt7GhQAWgSEShRSIOYsfLwXcRNOKRRnDVyEAHIyfBpTNgR8CCl8cUpk9pz1MTB8WpHqB39-J0IFLTgFFDIMbFKiUigMhHLKFW4mFBUFSPX1CygHQrEKShgoVggkMniQAHVBqHV8DZQtiB38HLyQRkKCUFiAkbqdpwGhiRQwSjmewdLoAAxCYQiADofthUQB+AouUFtME0PakACOMAQFBYD2eAEFusgYGJ2tRqbSUNI-vgxFI-jD7khpHNIe1rrREJQOrKPhlnEgIDy3IglKp+Y8qLcRU0FJo7AhhnZmvguXSHnKzbMIHAVfSkDR2Io2B8iaR1IdjiAAMJQ3JicTOajJPXOPnUI6bYYIe0-JoACgAChBxP66ApnAAZACq2ecAHFC84Xs5OItuhBqwBKADc+CjCFYIua7Ae7UgtPwyp5MJS7Q+CEDElMCjoHpAUAeADdcmERc8APKUMBiGj-BUoH6UCMKMSOsghhBKFi05wxBBSGAusjqDfG-DsMRpCOymhIFT4DHhdoAd1OZEYThBQESRTsIG7XtpDYCUOlqeVNm2M8nmmRJkjSWgSk8EBcyQTkaQoR81RPZQvxoMAZxYc4RgQdhn1HUYpXcWUAIyEYTEgZwxCgdgYDXUcrh1O5D0UQ17BNJoWgtEUG2gloCO5XlZQFIUMkdSczAnUpSDwsgIGfKA2hNSjclYMgg2cex2CkTJbXjR1nEFeiID-XJTDohjg2teyHQeZ0nxfMg5JVOUBI3L8hxHbymxjfA4xVc9rRaS9rxdBiUgCQIXBQCBMGoilmG9Z5lRvSchJuESHlQnCAEkmg0Fgmh-CItTIG9WAkZx7x+NLrXvXpQiDDi-w3MAmn0xAO0JNCpzBHcWHJUp9huH1aseDqj1MPi5BqTQsKaPkjjkfyXQyFAMMO0UmKoaV3L5D8VShWUqIQNy4lmqRnXsZ4ADEwWQMB2BUJUEBgKhnTfQLdB5JJUHcv82E+ZURg3CISkmQozAsKxCoOI4Su0dhyu1Sq9WeAARSCGsGZraUxDkEDnAKWiUULzDEHk+UB4bOjgkiAQQYaNRUY0ZsmaYQBofcADV50dVYZwARhAfwgA)


이 문서는 개발자 문서 플랫폼인 [Mintlify](https://mintlify.com)로 제작하고 호스팅합니다.
