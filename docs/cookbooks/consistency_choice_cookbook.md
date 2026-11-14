# Self-consistency: Choice 질문

> 모델에게 같은 moderation(콘텐츠 검수) 질문을 두 번 던지면 결정이 얼마나 자주 바뀔까요? 정답표 없이도 할 수 있는 신뢰성 테스트로, 경계선에 놓인 게시물을 두고 LLM과 Jev를 비교합니다.

모델에게 결정을 맡기는 pipeline(파이프라인)에서는 같은 게시물에 매번 같은 결정이 나와야
합니다. 여기서는 게시물 하나를 바꾸지 않고 반복해서 보내면서, 삭제할지 그대로 둘지,
escalate(상위 단계로 넘기기)할지 자동으로 처리할지, 어느 큐가 맡을지 같은 결정이 얼마나 자주
바뀌는지 셉니다.

이 테스트는 결정이 옳은지에 대해서는 아무것도 말해 주지 않습니다. 그보다 앞선 문제를 다룹니다.
게시물은 그대로인데 결정이 움직인다면 이의 제기를 재현할 수 없고, 똑같은 게시물이 서로 다른
처분을 받게 됩니다. 정답표는 필요 없고 반복 호출만 있으면 되므로 모든 모델을 같은 조건에서
측정할 수 있습니다.

`Choice` 질문 8개는 각각 선택지별 확률로 돌아옵니다. 선택지는 모두 37개이며, 예를 들어 올바른
조치가 Strike일 확률이 0.62라는 식입니다. pipeline은 cutoff(기준값)를 두고 이 확률 중 하나를
결정으로 바꿉니다. P(Strike)가 0.6을 넘으면 strike를 주는 식입니다. 다음 요청에서 0.55가
돌아오면 같은 게시물인데도 strike를 받지 않습니다. 이런 흔들림이 결정을 바꾸는지는 cutoff가
어디에 있느냐에 달려 있는데, 여러분의 cutoff가 어디인지는 알 수 없으므로 0에서 1까지 모든
cutoff를 똑같은 비중으로 셉니다. 두 답이 0.10만큼 떨어져 있으면 cutoff의 10%에서 서로 어긋납니다.
이 값을 반복 호출의 모든 쌍에 대해 평균하면 선택지마다 숫자 하나가 나옵니다. 다시 물었을 때
결정이 바뀔 확률입니다.

뚜렷한 게시물은 어느 모델에게나 쉬우므로 이 테스트에서는 경계선에 놓인 게시물을 씁니다.
게시물 하나의 40가지 버전이며, 버전마다 moderation 모델이 저울질해야 하는 사실들의 조합이
다릅니다. pipeline은 가장 불안한 결정만큼만 믿을 수 있으므로, 각 게시물은 가장 불안정한
선택지를 기준으로 점수를 매깁니다.

조건은 다음과 같으며, 조건 하나는 모델 하나와 설정 하나의 조합입니다.

* Claude 모델 `claude-fable-5-1`, `claude-opus-5-5`, `claude-sonnet-5-5`,
  `claude-haiku-5-5`와 GPT 모델 `gpt-6-astra`, `gpt-6.1-sol`, `gpt-6.1-terra`,
  `gpt-6-luna`. 모두 reasoning(추론)을 high effort로 실행합니다.
* reasoning을 끈 `gpt-6.1-terra`와 `gpt-6-luna`. temperature(온도)는 `0`과 API 기본값
  두 가지입니다. 여기서 temperature를 받는 모델은 이 둘뿐입니다.
* Jev 1.14(`jev-1.14.0`): `Choice` 질문 8개를 담은 `system_one` 호출 한 번입니다.

Jev의 경우 게시물에서 가장 불안정한 결정이 바뀌는 비율은 평균 4.0%, 즉 25번에 1번 정도이며
6.7%를 넘는 일이 없습니다. LLM 조건은 어느 것이든 결정이 10% 넘게 바뀌는 게시물이 있고,
조건에 따라 게시물 40개 중 2개에서 36개까지 결정이 10번에 1번 넘게 바뀝니다. temperature
`0`은 격차를 줄여 주지만 없애지는 못합니다. Jev가 어느 LLM 조건에 대해서든 보인 우위만큼 큰
격차가 순전히 우연으로 나올 가능성은 천 번 실행에 한 번도 되지 않습니다.

## 준비

```bash
pip install anthropic openai matplotlib ipython 'cooksafe>=0.3.0,<0.4.0'
```

그런 다음 `TYPESAFE_API_KEY`, `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`를 설정합니다. 아래 수치는
2026-11-02에 `jev-1.14.0`으로 얻은 것입니다. 40가지 버전에 대한 LLM 답은 2026-11-07부터
2026-11-09까지, 예시 게시물에 대한 답은 2026-11-10에 샘플링했습니다.

```python
import copy
import itertools
import json
import os
import random
import textwrap
import urllib.request
from concurrent.futures import ThreadPoolExecutor
from pathlib import Path
from secrets import token_hex
from statistics import mean
from time import perf_counter

import anthropic
import matplotlib
import matplotlib.pyplot as plt
import numpy as np
from cooksafe import JsonCache, make_playground_link
from IPython.display import Markdown, display
from matplotlib.colors import ListedColormap
from matplotlib.ticker import PercentFormatter
from openai import OpenAI
from typesafe_sdk import Choice, TypeSafeClient

matplotlib.use("Agg")  # 화면 없이 렌더링

LLM_MODELS = {  # 모두 high effort로 reasoning 실행
    "claude-fable-5-1": "Claude Fable 5.1",
    "claude-opus-5-5": "Claude Opus 5.5",
    "claude-sonnet-5-5": "Claude Sonnet 5.5",
    "claude-haiku-5-5": "Claude Haiku 5.5",
    "gpt-6-astra": "GPT-6 Astra",
    "gpt-6.1-sol": "GPT-6.1 Sol",
    "gpt-6.1-terra": "GPT-6.1 Terra",
    "gpt-6-luna": "GPT-6 Luna",
}
TEMPERATURE_MODELS = ["gpt-6.1-terra", "gpt-6-luna"]  # reasoning을 끈 조건도 실행: temperature 0 + 기본값
TYPESAFE_MODEL = "jev-1.14.0"  # 이 페이지에서 쓴 모델. "jev-latest"는 최신 모델을 실시간으로 실행
NUM_SAMPLES = 15  # 조건당 게시물+rubric 반복 호출 횟수
NUM_POSTS = 40  # 게시물 버전 수. 경계선 사실들의 조합에서 추출

LLM_PRICES = {  # token 1M개당 $ (입력, 출력), 2026-11 기준 정가
    "claude-fable-5-1": (10.00, 50.00),
    "claude-opus-5-5": (4.00, 20.00),
    "claude-sonnet-5-5": (2.00, 10.00),
    "claude-haiku-5-5": (0.10, 0.50),
    "gpt-6-astra": (10.00, 50.00),
    "gpt-6.1-sol": (2.00, 10.00),
    "gpt-6.1-terra": (1.00, 5.00),
    "gpt-6-luna": (0.10, 0.50),
}
TYPESAFE_PRICE = (0.042, 0.00)  # token 1M개당 $ (입력, 출력), 2026-08 기준

anthropic_client = anthropic.Anthropic(
    api_key=os.environ.get("ANTHROPIC_API_KEY", "cache-only")
)
openai_client = OpenAI(api_key=os.environ.get("OPENAI_API_KEY", "cache-only"))
typesafe_client = TypeSafeClient(
    api_key=os.environ.get("TYPESAFE_API_KEY", "cache-only"),
    base_url="https://api.typesafe.ai",
    timeout=30.0,
)
```

## state: 경계선에 놓인 사용자 게시물(JSON)

아래 게시물은 어느 쪽으로도 판단할 수 있도록 만들었습니다. 표현은 격하고 모욕적이며, 일부는
특정인을, 일부는 주장과 커뮤니티를 겨냥합니다. 플랫폼 밖으로 부르는 초대(사람들을 다른 사이트로
끌어가는 링크)가 들어 있고, 위협을 명시하지는 않지만 위협처럼 들리는 문구도 있습니다. 계정에는
이전 strike가 한 번 있고, 사용자 네 명이 이 게시물을 신고했습니다.

이것이 예시 게시물입니다.

LLM은 prompt(프롬프트) 안에서 `json.dumps(post)`를 받습니다. Jev는 구조를 그대로 받습니다.

```python
POST = {
    "post_id": "P-88213",
    "author": {
        "user_id": "user_4471",
        "account_age_days": 38,
        "prior_strikes": 1,
        "followers": 210,
    },
    "context": {
        "surface": "public reply",
        "in_reply_to": "another user defending a game patch",
        "community": "example-game-debates",
    },
    "content": {
        "text": (
            "Are you seriously this dense? Anyone who defends that patch is a complete clown and "
            "should be embarrassed to even post here. People like you are what's ruining this "
            "community and honestly you need to be dealt with. Come say it to my face, invite's "
            "right here. Keep it up and I'll end your whole channel."
        ),
        "has_link": True,
        "link_domain": "chat.example",
        "language": "en",
    },
    "reports": {"user_reports": 4, "report_reasons": ["harassment", "spam", "threat"]},
}
```

## rubric: `Choice` 질문 8개

rubric(루브릭: 채점 기준)의 질문에는 각각 `key`, 한 줄짜리 instructions(지시문), 고정된 선택지
집합이 있습니다. 한 질문 안의 선택지는 상호 배타적이고(정확히 하나만 해당합니다) 선택지마다
짧은 설명이 붙습니다. Jev는 고른 `choice`와 함께 선택지에 대한 `probabilities` 분포를
반환합니다. LLM에게도 같은 선택지를 쓰도록 요청하므로 LLM의 답과 Jev의 답을 서로 비교할 수
있습니다.

```python
QUESTIONS = {
    "category": (
        "What is the single most applicable content-policy category for this post?",
        {
            "None": "No policy violation of any kind.",
            "Harass": "Insults or demeans a person, with no threat of harm and no protected-class attack.",
            "Hate": "Attacks a person or group over a protected characteristic (race, religion, gender, ...).",
            "Violence": "Makes a credible threat of harm or incites violence against someone.",
            "Spam": "Unsolicited promotion or link spam, with no personal attack.",
            "Sexual": "Sexual or adult content.",
        },
    ),
    "primary_risk": (
        "What is the primary moderation risk that should drive triage for this post?",
        {
            "Harassment": "Personal attack or targeted abuse is the main risk.",
            "Violence": "A threat of harm or intimidation is the main risk.",
            "LinkAbuse": "External-link or off-platform coordination risk is the main risk.",
            "AccountHistory": "Prior account history or repeat behavior is the main risk.",
            "LowRisk": "No meaningful moderation risk is present.",
        },
    ),
    "target": (
        "Who or what is the content primarily directed at?",
        {
            "None": "Not directed at anyone in particular.",
            "Person": "Directed at one specific individual.",
            "Group": "Directed at a protected group or class.",
            "Platform": "Directed at the community or platform itself, not a person.",
        },
    ),
    "action": (
        "What enforcement action should be taken on this post?",
        {
            "Allow": "Leave the post up with no action.",
            "Warn": "Leave the post up but attach a warning label.",
            "Remove": "Remove the post, but do not penalize the account.",
            "Strike": "Remove the post and add a strike to the account.",
            "Escalate": "Take no automated action; hold for a human decision.",
        },
    ),
    "queue": (
        "Which single moderation queue should own this post?",
        {
            "Auto": "Auto-resolve; no human queue needed.",
            "General": "General moderation queue.",
            "Threat": "Threat / violence response queue.",
            "Spam": "Spam and platform-abuse queue.",
            "TSLead": "Trust-and-safety lead / senior queue.",
        },
    ),
    "link_handling": (
        "How should any external link or off-platform invite in the post be handled?",
        {
            "Allow": "Leave the link in place.",
            "RmLink": "Strip or disable the link but keep the post.",
            "Brigade": "Treat the link as coordinated brigading and action it as abuse.",
            "Escalate": "Send the link to a specialist to assess before acting.",
        },
    ),
    "review_path": (
        "Who should make the final call on this post?",
        {
            "Auto": "Automated action; no human review.",
            "Human": "A frontline human moderator makes the call.",
            "Senior": "A senior or specialist reviewer is required.",
            "Legal": "Route to legal or law-enforcement escalation.",
        },
    ),
    "severity": (
        "What is the overall severity of this post?",
        {
            "None": "No violation.",
            "Low": "Rude or dismissive, but essentially harmless.",
            "Medium": "Personal harassment with no clearly credible threat.",
            "High": "Harassment together with a threat that could be read as credible.",
        },
    ),
}
QUESTIONS_JSON = json.dumps(QUESTIONS)
```

## 질문하는 방법

LLM 호출 한 번은 `json.dumps(post)`, 질문 8개 전부, 허용되는 모든 선택지를 담은 prompt
하나입니다. 모델은 structured output(구조화된 출력)으로 답합니다. 질문마다 모든 선택지에 대한
확률을 요구하는 JSON schema(스키마)입니다. 호출은 모델 이름에 따라 Anthropic 또는 OpenAI로
routing(라우팅)됩니다. reasoning은 `high` effort로 실행하고, reasoning을 끈 `gpt-6.1-terra`와
`gpt-6-luna`는 `temperature`를 받으며 값은 `0` 또는 API 기본값입니다.

TypeSafe 호출은 같은 게시물과 같은 `Choice` 질문 8개를 담은 `system_one` 요청 한 번이며,
질문마다 분포 하나를 반환합니다.

모든 쿼리에는 새 `uid`도 붙습니다. 게시물과 rubric은 그대로 둔 채 실행할 때마다 바뀌는 일회용
고유 값입니다. LLM prompt에 들어가고, TypeSafe state에는 추가 필드로 들어갑니다. 이 값 때문에
요청이 매번 달라지므로, 중간에 있는 어떤 캐시도 이전 답을 그대로 돌려줄 수 없습니다.

호출마다 비용과 왕복 시간도 함께 기록합니다.

```python
def argmax_label(values: list, labels: list[str]) -> str | None:
    """The label with the most probability mass, or ``None`` if any value is missing or
    non-numeric -- a partially parsed distribution never yields a confident-looking pick."""
    numeric = [_numeric_value(value) for value in values]
    if any(value is None for value in numeric):
        return None
    return labels[int(np.argmax(numeric))]


def _numeric_value(value: object) -> float | None:
    """A finite numeric value, or ``None`` if the model emitted something unusable."""
    try:
        numeric = float(value)
    except (TypeError, ValueError):
        return None
    return numeric if np.isfinite(numeric) else None


def parse_distribution(raw: object, labels: list[str]) -> list[float]:
    """Map a model's per-question reply to per-label probabilities, in label order.

    Anything that doesn't match a known label or isn't a finite number is left NaN -- we report
    the gap rather than massaging the reply (e.g. stripping an echoed description) to make it
    fit."""
    if not isinstance(raw, dict):
        return [float("nan")] * len(labels)
    return [
        value if (value := _numeric_value(raw.get(label))) is not None else float("nan")
        for label in labels
    ]


def rubric_prompt(sample_index: int, post: dict, questions: dict) -> str:
    """The post + all questions (with their label sets) in one prompt, asking for a
    probability distribution per question. ``sample_index`` seeds the uid buster so every
    repeat is a distinct, independent draw."""
    lines = []
    for key, (instructions, choices) in questions.items():
        labels = "\n".join(f"     {label}: {desc}" for label, desc in choices.items())
        lines.append(f"- {key}: {instructions}\n   labels:\n{labels}")
    return (
        f"uid: {sample_index}:{token_hex(4)}\n\n"
        f"Document (a reported user post):\n{json.dumps(post, indent=2)}\n\nQuestions:\n"
        + "\n".join(lines)
        + "\n\nEach question's labels are mutually exclusive: exactly one applies. If a post could "
        "arguably fit more than one, pick the single most severe / most specific label per the "
        "label descriptions."
        + "\n\nFor each question, give a probability distribution over that question's labels "
        "(values 0.00-1.00 that sum to 1).\nRespond with ONLY a JSON object mapping each "
        "question's key to an object mapping that question's bare labels (the label only, "
        "not its description) to probabilities, with one entry per question."
    )


def _cost(prices: tuple[float, float], input_tokens: int, output_tokens: int) -> float:
    return input_tokens / 1e6 * prices[0] + output_tokens / 1e6 * prices[1]


def _answer_schema(questions: dict) -> dict:
    """Structured output: for each question, exactly one number per option."""
    properties = {
        key: {
            "type": "object",
            "properties": {label: {"type": "number"} for label in choices},
            "required": list(choices),
            "additionalProperties": False,
        }
        for key, (_instructions, choices) in questions.items()
    }
    return {
        "type": "object",
        "properties": properties,
        "required": list(properties),
        "additionalProperties": False,
    }


def _call_llm(model: str, prompt: str, schema: dict, reasoning: str,
              temperature: float | None):
    """One LLM call -> (text, cost_usd, latency_s), routed by model name. ``reasoning`` is the
    effort; ``"none"`` turns reasoning off, which only the GPT models allow."""
    started = perf_counter()
    if model.startswith("claude"):
        with anthropic_client.messages.stream(
            model=model,
            max_tokens=32000,
            messages=[{"role": "user", "content": prompt}],
            thinking={"type": "adaptive"},
            output_config={"format": {"type": "json_schema", "schema": schema},
                           "effort": reasoning},
        ) as stream:
            response = stream.get_final_message()
        text = next((b.text for b in response.content if b.type == "text"), "")
        usage = (response.usage.input_tokens, response.usage.output_tokens)
    else:
        kwargs = {
            "model": model,
            "messages": [{"role": "user", "content": prompt}],
            "response_format": {
                "type": "json_schema",
                "json_schema": {"name": "rubric", "schema": schema, "strict": True},
            },
            "reasoning_effort": reasoning,
        }
        if temperature is not None:
            kwargs["temperature"] = temperature
        response = openai_client.chat.completions.create(**kwargs)
        text = response.choices[0].message.content or ""
        usage = (response.usage.prompt_tokens, response.usage.completion_tokens)
    return text, _cost(LLM_PRICES[model], *usage), perf_counter() - started


def _query_llm(model, reasoning, temperature, sample_index, post_json, questions_json):
    """One LLM rubric query -> (label distributions keyed by question key, cost_usd,
    latency_s); NaNs where the reply doesn't parse."""
    questions = json.loads(questions_json)
    prompt = rubric_prompt(sample_index, json.loads(post_json), questions)
    text, cost, latency = _call_llm(model, prompt, _answer_schema(questions), reasoning,
                                    temperature)
    try:
        raw = json.loads(text)
    except (ValueError, json.JSONDecodeError):
        raw = {}
    if not isinstance(raw, dict):
        raw = {}
    distributions = {
        key: parse_distribution(raw.get(key), list(choices))
        for key, (_instructions, choices) in questions.items()
    }
    return distributions, cost, latency


def _query_typesafe(sample_index, post_json, questions_json, model):
    """One TypeSafe call -> (label distributions keyed by question key, input tokens,
    output tokens, latency_s)."""
    questions = json.loads(questions_json)
    started = perf_counter()
    response = typesafe_client.system_one(
        model=model,
        state={"uid": f"{sample_index}:{token_hex(4)}", "post": json.loads(post_json)},
        questions={
            key: Choice(instructions=instructions, criteria=choices)
            for key, (instructions, choices) in questions.items()
        },
    )
    distributions = {}
    for key, (_instructions, choices) in questions.items():
        probabilities = dict(response.answers[key].probabilities)
        distributions[key] = [probabilities.get(label, float("nan")) for label in choices]
    return (
        distributions,
        response.usage.input_tokens,
        response.usage.output_tokens,
        perf_counter() - started,
    )


# 모든 샘플은 캐시 파일에 저장되므로 두 번째 실행에서는 요청을 보내지 않습니다. 게시물과
# 질문은 모든 키에 포함되므로 둘 중 하나를 수정하면 다시 묻습니다. ``sample_index``도
# 키에 포함되므로 NUM_SAMPLES번의 반복은 각각 독립적인 추출입니다.

EXAMPLES_URL = (
    "https://raw.githubusercontent.com/typesafe-ai/typesafe-public-examples/main/"
    "cookbooks/consistency_choice_cookbook"
)


def download(name: str) -> Path:
    """Fetch one of this cookbook's files from typesafe-public-examples unless it is on disk."""
    path = Path(name)
    if not path.exists():
        urllib.request.urlretrieve(f"{EXAMPLES_URL}/{name}", path)
    return path


CACHE = Path("consistency_choice_cookbook_cache.json")
# True이면 공개된 답으로 CACHE를 채우므로 실행 중에 요청을 보내지 않습니다. 실시간으로 실행하려면
# False로 바꾸고 CACHE를 삭제하세요. 디스크에 이미 있는 CACHE는 절대 덮어쓰지 않습니다.
DOWNLOAD_CACHE = True
if DOWNLOAD_CACHE:
    download(CACHE.name)
json_cache = JsonCache(CACHE)


@json_cache
def ask_llm(model: str, reasoning: str, temperature: float | None, sample_index: int,
            post_json: str, questions_json: str):
    """The example post's LLM answers, with cost and latency."""
    return _query_llm(model, reasoning, temperature, sample_index, post_json, questions_json)


@json_cache
def ask_typesafe(sample_index: int, post_json: str, questions_json: str, model: str):
    """The example post's Jev answers, with token usage and latency."""
    return _query_typesafe(sample_index, post_json, questions_json, model)


@json_cache
def sweep_llm(model: str, reasoning: str, temperature: float | None, sample_index: int,
              post_json: str, questions_json: str):
    """One LLM answer set for one version of the post."""
    return _query_llm(model, reasoning, temperature, sample_index, post_json, questions_json)[0]


@json_cache
def sweep_typesafe(sample_index: int, post_json: str, questions_json: str, model: str):
    """One Jev answer set for one version of the post."""
    return _query_typesafe(sample_index, post_json, questions_json, model)[0]
```

## 조건

| 모델 그룹 | 모델 | Reasoning (high) | Temperature 0 | 기본 temperature |
| - | - | :-: | :-: | :-: |
| Claude | `claude-fable-5-1` | ✓ | - | - |
| Claude | `claude-opus-5-5` | ✓ | - | - |
| Claude | `claude-sonnet-5-5` | ✓ | - | - |
| Claude | `claude-haiku-5-5` | ✓ | - | - |
| GPT | `gpt-6-astra` | ✓ | - | - |
| GPT | `gpt-6.1-sol` | ✓ | - | - |
| GPT | `gpt-6.1-terra` | ✓ | ✓ | ✓ |
| GPT | `gpt-6-luna` | ✓ | ✓ | ✓ |
| System One | `jev-1.14.0` | - | - | ✓ |

* 체크 표시 하나가 조건 하나이며, 게시물당 15번 실행합니다. 하이픈(-)은 테스트하지 않은
  조합입니다.
* temperature 열은 reasoning을 끈 상태입니다. Claude 모델은 temperature를 받지 않고,
  GPT 모델도 reasoning을 하는 동안에는 받지 않습니다.
* 기본값 열에서는 temperature 인자를 보내지 않습니다.
* 반복 가능성을 높이려면 temperature `0`을 쓰라는 권장이 흔하므로, 모델이 temperature를 받는
  경우 API 기본값과 비교합니다.

```python
CONDITIONS = [
    {"label": f"{name} (reasoning)", "model": model, "reasoning": "high", "temp": None}
    for model, name in LLM_MODELS.items()
] + [
    {"label": f"{LLM_MODELS[model]} ({setting})", "model": model, "reasoning": "none",
     "temp": temp}
    for model in TEMPERATURE_MODELS
    for temp, setting in ((0, "temperature 0"), (None, "default temperature"))
]
LABELS = [condition["label"] for condition in CONDITIONS]
TYPESAFE_LABEL = "Jev 1.14"
ALL_LABELS = [*LABELS, TYPESAFE_LABEL]
# 예시 그래프의 행으로, 두 self-consistency 페이지에서 동일합니다. Jev와 조건 몇 개만 넣습니다.
# 조건을 전부 넣으면 너무 길어져 읽기 어렵기 때문입니다.
PLOT_ORDER = [
    TYPESAFE_LABEL,
    "Claude Opus 5.5 (reasoning)",
    "Claude Haiku 5.5 (reasoning)",
    "GPT-6.1 Sol (reasoning)",
    "GPT-6 Luna (temperature 0)",
    "GPT-6 Luna (default temperature)",
]
STRIP_ORDER = [TYPESAFE_LABEL, *LABELS]  # 마지막 그래프의 행: 모든 조건


POST_JSON = json.dumps(POST)

runs: dict[str, list] = {}  # 레이블 -> {질문 key: 분포} 샘플 NUM_SAMPLES개
stats: dict[str, list] = {}  # 레이블 -> (cost_usd, latency_s) 쌍 NUM_SAMPLES개
with ThreadPoolExecutor(max_workers=16) as pool:
    futures = {
        condition["label"]: [
            pool.submit(ask_llm, condition["model"], condition["reasoning"],
                        condition["temp"], sample_index, POST_JSON, QUESTIONS_JSON)
            for sample_index in range(NUM_SAMPLES)
        ]
        for condition in CONDITIONS
    }
    for label, sample_futures in futures.items():
        results = [future.result() for future in sample_futures]
        runs[label] = [result[0] for result in results]
        stats[label] = [(result[1], result[2]) for result in results]

# Jev 샘플은 LLM 호출이 끝난 뒤 순차적으로 뽑습니다. 그래야 지연 시간이 16개 LLM 스레드가
# 경합하는 상황에서 잰 값이 아니라 깨끗한 왕복 시간이 됩니다.
typesafe_usage_results = [
    ask_typesafe(sample_index, POST_JSON, QUESTIONS_JSON, model=TYPESAFE_MODEL)
    for sample_index in range(NUM_SAMPLES)
]
# 가격은 캐시에서 꺼낸 뒤에 적용하므로, 가격이 바뀌어도 새로 샘플링할 필요가 없습니다.
runs[TYPESAFE_LABEL] = [result[0] for result in typesafe_usage_results]
stats[TYPESAFE_LABEL] = [
    (_cost(TYPESAFE_PRICE, input_tokens, output_tokens), latency)
    for _distributions, input_tokens, output_tokens, latency in typesafe_usage_results
]
```

### 비용 + 속도 (rubric 쿼리당)

비용은 준비 절에 나온 token(토큰)당 가격으로 계산합니다. LLM은 2026-11 기준 정가이고,
TypeSafe는 2026-08 기준 요율입니다.

행 하나는 예시 게시물에 대한 질문 8개짜리 rubric 호출 한 번 전체입니다. `time/call`과
`cost/call`은 15번 호출의 평균이고, `vs Jev 1.14` 열은 Jev 1.14의 수치로 나눈
값입니다.

```python
typesafe_cost = mean([cost for cost, _latency in stats[TYPESAFE_LABEL]])
typesafe_latency = mean([latency for _cost, latency in stats[TYPESAFE_LABEL]])
name_w = max(len(name) for name in ALL_LABELS) + 2  # 가장 긴 조건 레이블에 맞춤
# 비교 열 머리글을 두 줄로 쌓아서 상대 속도 열과 상대 비용 열을 좁게 유지합니다.
print(
    f"{'':<{name_w + 31}}{'speed vs':>11}{'cost vs':>11}\n"
    f"{'condition':<{name_w}}{'calls':>7}{'time/call':>11}{'cost/call':>13}"
    f"{TYPESAFE_LABEL:>11}{TYPESAFE_LABEL:>11}"
)
for name in ALL_LABELS:
    costs, latencies = zip(*stats[name])
    cost = mean(costs)
    latency = mean(latencies)
    print(
        f"{name:<{name_w}}{len(costs):>7}{latency * 1000:>9.0f}ms"
        f"{'$' + format(cost, '.6f'):>13}"
        f"{format(latency / typesafe_latency, '.1f') + 'x':>11}"
        f"{format(cost / typesafe_cost, '.1f') + 'x':>11}"
    )
```

```
                                                                       speed vs    cost vs
condition                              calls  time/call    cost/call   Jev 1.14   Jev 1.14
Claude Fable 5.1 (reasoning)              15    12793ms    $0.077985     124.5x    1189.0x
Claude Opus 5.5 (reasoning)               15     7841ms    $0.027427      76.3x     418.2x
Claude Sonnet 5.5 (reasoning)             15     3627ms    $0.009907      35.3x     151.0x
Claude Haiku 5.5 (reasoning)              15     6110ms    $0.001019      59.5x      15.5x
GPT-6 Astra (reasoning)                   15    18171ms    $0.055399     176.9x     844.6x
GPT-6.1 Sol (reasoning)                   15    12335ms    $0.010125     120.1x     154.4x
GPT-6.1 Terra (reasoning)                 15    11829ms    $0.005616     115.2x      85.6x
GPT-6 Luna (reasoning)                    15    14467ms    $0.000685     140.8x      10.4x
GPT-6.1 Terra (temperature 0)             15     4274ms    $0.002694      41.6x      41.1x
GPT-6.1 Terra (default temperature)       15     4344ms    $0.002745      42.3x      41.8x
GPT-6 Luna (temperature 0)                15     2753ms    $0.000269      26.8x       4.1x
GPT-6 Luna (default temperature)          15     2836ms    $0.000269      27.6x       4.1x
Jev 1.14                                  15      103ms    $0.000066       1.0x       1.0x
```

이번 실행에서 Jev의 평균 왕복 지연 시간은 103ms입니다. LLM 조건은 위의 동시 실행 설정에서
호출당 2.8초에서 18.2초 사이입니다.

## 그래프: 예시 게시물에서 모든 샘플이 고른 1순위 선택지

읽는 법은 다음과 같습니다.

* 바깥쪽 행 그룹: 질문입니다.
* 안쪽 행: 조건입니다.
* 열: rubric 호출 한 번 전체입니다.
* 셀 텍스트: 1순위 선택지와 그 확률입니다.
* 셀 색: 해당 질문 안에서 그 선택지가 놓인 위치입니다. 따라서 한 행이 끝까지 같은 색이면
  매번 같은 선택지가 1순위였다는 뜻입니다.
* 빗금 친 `n/a`: 응답을 쓸 수 있는 선택지로 파싱하지 못한 경우(파싱 실패)입니다.

```python
GAP = 1  # 질문 블록 사이에 넣는 빈 간격 행 수
rows_per_block = len(PLOT_ORDER)  # 질문 블록당 행 수


def top_label_annotation(values: list, labels: list[str]) -> str:
    """The top label and its probability, for a heatmap cell."""
    label = argmax_label(values, labels)
    if label is None:
        return ""
    probability_text = f"{max(float(value) for value in values):.2f}".removeprefix("0")
    return f"{label} {probability_text}"


row_index_values, row_text, row_labels, blocks = [], [], [], []
for question_index, (question_key, (question_text, choices)) in enumerate(
    QUESTIONS.items()
):
    labels = list(choices)
    if question_index:  # 빈 간격 행(NaN -> 흰색으로 렌더링)이 블록을 구분
        row_index_values.extend([np.nan] * NUM_SAMPLES for _ in range(GAP))
        row_text.extend([[""] * NUM_SAMPLES for _ in range(GAP)])
        row_labels.extend([""] * GAP)
    blocks.append((len(row_index_values), question_key, question_text))
    for label in PLOT_ORDER:
        values_by_sample = [runs[label][sample][question_key] for sample in range(NUM_SAMPLES)]
        picks = [argmax_label(values, labels) for values in values_by_sample]
        row_index_values.append(
            [labels.index(pick) if pick in labels else np.nan for pick in picks]
        )
        row_text.append(
            [top_label_annotation(values, labels) for values in values_by_sample]
        )
        row_labels.append(label)

heatmap_matrix = np.array(row_index_values, dtype=float)
cmap = ListedColormap(plt.get_cmap("tab10").colors)
cmap.set_bad(
    "white"
)  # NaN 셀(간격 행과 파싱할 수 없는 응답 모두)은 여기서 흰색으로 렌더링되지만...

fig, ax = plt.subplots(figsize=(15, 0.33 * len(row_index_values) + 1))
ax.imshow(heatmap_matrix, cmap=cmap, vmin=0, vmax=9, aspect="auto")
for row in range(heatmap_matrix.shape[0]):
    is_spacer_row = row_labels[row] == ""  # 질문 블록 사이의 빈 구분 행
    for col in range(heatmap_matrix.shape[1]):
        label_text = row_text[row][col]
        if label_text:
            ax.text(
                col,
                row,
                label_text,
                ha="center",
                va="center",
                fontsize=5.7,
                family="monospace",
                color="black",
            )
        elif (
            not is_spacer_row
        ):  # ...파싱할 수 없는 응답은 빈 흰색이 아니라 빗금 친 "n/a"로 표시
            ax.add_patch(
                plt.Rectangle(
                    (col - 0.5, row - 0.5),
                    1,
                    1,
                    facecolor="#e8e8e8",
                    edgecolor="#b0b0b0",
                    hatch="////",
                    linewidth=0,
                )
            )
            ax.text(
                col,
                row,
                "n/a",
                ha="center",
                va="center",
                fontsize=5,
                family="monospace",
                color="#b30000",
            )

ax.set_xticks(range(NUM_SAMPLES))
ax.set_xticklabels(range(1, NUM_SAMPLES + 1), fontsize=7)
ax.set_xlabel("rubric query")
ax.set_yticks(range(len(row_labels)))
ax.set_yticklabels(row_labels, fontsize=7)
ax.tick_params(length=0)
for edge in ("top", "right", "left", "bottom"):
    ax.spines[edge].set_visible(False)

# 다중 인덱스의 바깥 단계: 질문 key를 블록마다 한 번씩 가운데에 찍고, 바로 아래에
# 질문 텍스트를 줄바꿈해서 넣습니다
y_axis_transform = ax.get_yaxis_transform()
for start, question_key, question_text in blocks:
    center = start + (rows_per_block - 1) / 2
    ax.text(
        -0.2,
        center - 0.7,
        question_key,
        transform=y_axis_transform,
        ha="right",
        va="center",
        fontsize=8,
        fontweight="bold",
    )
    ax.text(
        -0.2,
        center + 0.1,
        textwrap.fill(question_text, 34),
        transform=y_axis_transform,
        ha="right",
        va="top",
        fontsize=6,
        style="italic",
        color="gray",
    )

ax.set_title(
    "Every sample's top option + its probability on the example post",
    pad=12,
)
fig.tight_layout()
display(fig)
```

![출력](https://mintcdn.com/ts-docs/orb6EiXzZ12Fo-od/cookbooks/consistency_choice_cookbook/consistency_choice_cookbook.executed.1.png?fit=max\&auto=format\&n=orb6EiXzZ12Fo-od\&q=85\&s=58026a966827f85ad53f034232fbe48c)

`target`은 모든 조건의 모든 반복에서 Person으로 나옵니다. Jev는 질문 하나, `primary_risk`에서만
1순위 선택지가 바뀝니다. Harassment가 11번, Violence가 4번이며 둘 다 0.5에 가깝습니다.
Claude Haiku 5.5는 질문 8개 중 7개에서 1순위 선택지가 바뀌고, `action`은 Strike, Escalate,
Remove 세 갈래로 나뉩니다. Claude Opus 5.5와 GPT-6.1 Sol은 각각 질문 4개에서 바뀝니다.
GPT-6 Luna는 temperature `0`에서 `action`이 한 번 바뀌고, 기본 temperature에서는 `action`과
`link_handling`이 갈립니다.

1순위 선택지의 변화로는 두 선택지의 순서가 뒤바뀌는 움직임만 드러납니다. 확률은 순서를 전혀
바꾸지 않고도 크게 움직일 수 있고, 그 움직임 구간 어디에든 cutoff가 놓여 있으면 결정이
바뀝니다.

## 게시물의 40가지 버전

모델은 어떤 게시물에서는 안정적이다가 다음 게시물에서는 흔들릴 수 있으므로, 게시물 하나만으로는
모델을 판단하기에 부족합니다. 게시물의 각 버전은 moderation 모델이 저울질해야 하는 사실 여섯
가지를 정합니다.

* 마지막 문장: "Keep it up and I'll end your whole channel."(예시 게시물과 같음),
  "Keep it up and I'll find out where you live.", "Keep it up and I'll report your whole
  channel.", 또는 마지막 문장 없음
* 모욕이 사람을 겨냥하는지 패치를 겨냥하는지
* 게시물에 플랫폼 밖 초대 링크가 있는지
* 계정의 이전 strike: 없음 또는 세 번
* 사용자 신고: 네 건 또는 없음
* 계정 나이: 38일 또는 2일

이렇게 하면 조합이 128가지 나옵니다. 고정된 시드로 그중 40개를 뽑고, 모든 조건에서 각각을
15번씩 보냅니다.

```python
PERSONAL = (
    "Are you seriously this dense? Anyone who defends that patch is a complete clown and "
    "should be embarrassed to even post here. People like you are what's ruining this "
    "community and honestly you need to be dealt with."
)
IMPERSONAL = (
    "Is this patch serious? Whoever signed off on it is a complete clown and should be "
    "embarrassed. This is what's ruining this community."
)
INVITE = " Come say it to my face, invite's right here."
CLOSINGS = {
    "end-channel": " Keep it up and I'll end your whole channel.",
    "find-you": " Keep it up and I'll find out where you live.",
    "report": " Keep it up and I'll report your whole channel.",
    "no-closing": "",
}
SWITCHES = ("personal", "invite", "strikes", "reports", "new_account")


def build_post(closing: str, personal: bool, invite: bool, strikes: bool, reports: bool,
               new_account: bool) -> dict:
    """The example post with the six borderline facts set one way."""
    post = copy.deepcopy(POST)
    post["content"]["text"] = (
        (PERSONAL if personal else IMPERSONAL) + (INVITE if invite else "") + CLOSINGS[closing]
    )
    post["content"]["has_link"] = invite
    post["content"]["link_domain"] = "chat.example" if invite else None
    post["author"]["prior_strikes"] = 3 if strikes else 0
    post["author"]["account_age_days"] = 2 if new_account else 38
    post["reports"] = (
        {"user_reports": 4, "report_reasons": ["harassment", "spam", "threat"]}
        if reports
        else {"user_reports": 0, "report_reasons": []}
    )
    return post


combinations = list(itertools.product(CLOSINGS, *[(False, True)] * len(SWITCHES)))
POSTS = {
    "-".join([combo[0], *(name for name, on in zip(SWITCHES, combo[1:]) if on)]): build_post(*combo)
    for combo in random.Random(0).sample(combinations, NUM_POSTS)
}

# 게시물 40개 x 조건 13개 x 반복 15번. 실시간으로 실행하면 몇 시간이 걸리며, 대부분은 모델이
# high effort로 reasoning하는 시간입니다.
with ThreadPoolExecutor(max_workers=16) as pool:
    sweep = {
        post_id: {
            **{
                condition["label"]: [
                    pool.submit(sweep_llm, condition["model"], condition["reasoning"],
                                condition["temp"], index, json.dumps(post), QUESTIONS_JSON)
                    for index in range(NUM_SAMPLES)
                ]
                for condition in CONDITIONS
            },
            TYPESAFE_LABEL: [
                pool.submit(
                    sweep_typesafe, index, json.dumps(post), QUESTIONS_JSON, model=TYPESAFE_MODEL
                )
                for index in range(NUM_SAMPLES)
            ],
        }
        for post_id, post in POSTS.items()
    }
    sweep = {
        post_id: {label: [future.result() for future in futures] for label, futures in by_label.items()}
        for post_id, by_label in sweep.items()
    }
```

## 다시 물으면 결정이 얼마나 자주 바뀌는가

게시물의 선택지마다 답 15개를 모든 쌍, 즉 105쌍으로 비교하고 두 확률이 얼마나 떨어져 있는지
평균을 냅니다. 이 평균이 모든 cutoff를 똑같은 비중으로 셌을 때, 다시 물으면 그 선택지에 대한
결정이 바뀔 확률입니다. 그런 다음 각 게시물은 가장 불안정한 선택지의 점수, 즉 37개 중 가장 높은
값을 자신의 점수로 삼습니다. 표에는 평균적인 게시물과 최악의 게시물에서의 이 점수, 10번에 1번을
넘는 게시물 수, 그리고 좀 더 너그러운 척도로 37개 선택지 전체의 평균이 나옵니다. 파싱되지 않은
답은 계산에서 빼고 마지막 열에 그 수를 셉니다.

```python
def change_chances(samples: list) -> tuple[dict, int]:
    """Per option: the mean distance between the probabilities of every pair of repeats,
    which is the chance that asking again changes the decision; plus the number of
    question answers that didn't parse (left out)."""
    chances, unparsed = {}, 0
    for question_key, (_instructions, choices) in QUESTIONS.items():
        values = np.array([sample[question_key] for sample in samples], dtype=float)
        parsed = ~np.isnan(values).any(axis=1)
        unparsed += int((~parsed).sum())
        values = values[parsed]
        if len(values) < 2:  # 비교할 수 있는 쓸 만한 답의 쌍이 없음
            continue
        first, second = np.triu_indices(len(values), k=1)
        gaps = np.abs(values[first] - values[second]).mean(axis=0)
        for label, gap in zip(choices, gaps):
            chances[f"{question_key}={label}"] = float(gap)
    return chances, unparsed


average_change = {label: [] for label in ALL_LABELS}  # 레이블 -> 게시물별 전체 레이블 평균
least_stable = {label: [] for label in ALL_LABELS}  # 레이블 -> 게시물별 가장 불안정한 레이블
unparsed = {label: 0 for label in ALL_LABELS}  # 레이블 -> 파싱되지 않은 질문 답의 수
for by_label in sweep.values():
    for label in ALL_LABELS:
        chances, failures = change_chances(by_label[label])
        average_change[label].append(mean(chances.values()))
        least_stable[label].append(max(chances.values()))
        unparsed[label] += failures
ranked = sorted(ALL_LABELS, key=lambda label: mean(least_stable[label]))

print(
    f"{'':<{name_w}}{'average over':>14}{'least stable option':>24}{'posts above':>13}"
    f"{'unparsed':>10}\n"
    f"{'condition':<{name_w}}{'all options':>14}{'average post':>14}{'worst post':>12}"
    f"{'1 in 10':>12}{'answers':>12}"
)
for label in ranked:
    print(
        f"{label:<{name_w}}{mean(average_change[label]):>14.1%}"
        f"{mean(least_stable[label]):>14.1%}{max(least_stable[label]):>12.1%}"
        f"{sum(value > 0.10 for value in least_stable[label]):>9}/{NUM_POSTS}"
        f"{unparsed[label]:>12}"
    )
```

```
                                       average over     least stable option  posts above  unparsed
condition                               all options  average post  worst post     1 in 10     answers
Jev 1.14                                       1.0%          4.0%        6.7%        0/40           0
Claude Opus 5.5 (reasoning)                    2.4%          6.6%       10.3%        2/40           0
Claude Fable 5.1 (reasoning)                   2.7%          7.5%       16.8%        5/40           0
Claude Sonnet 5.5 (reasoning)                  3.2%         10.3%       25.3%       13/40           0
GPT-6 Luna (temperature 0)                     2.2%         11.5%       28.4%       21/40           0
Claude Haiku 5.5 (reasoning)                   4.4%         12.8%       16.8%       35/40           0
GPT-6 Astra (reasoning)                        2.2%         12.9%       25.5%       25/40           0
GPT-6.1 Sol (reasoning)                        2.5%         14.9%       25.8%       30/40           0
GPT-6.1 Terra (reasoning)                      3.2%         16.0%       28.0%       32/40           0
GPT-6 Luna (default temperature)               3.6%         16.5%       29.8%       35/40           0
GPT-6.1 Terra (temperature 0)                  2.7%         16.7%       33.4%       30/40           0
GPT-6 Luna (reasoning)                         4.2%         17.1%       26.7%       34/40           0
GPT-6.1 Terra (default temperature)            3.4%         20.9%       36.6%       36/40           0
```

Jev의 경우 가장 불안정한 결정이 바뀌는 비율은 평균적인 게시물에서 4.0%, 최악의 게시물에서
6.7%이며, 10번에 1번을 넘는 게시물은 없습니다. LLM 조건에서는 평균적인 게시물에서 6.6%에서
20.9%까지 바뀌고, 게시물 40개 중 2개에서 36개가 10번에 1번을 넘습니다. 답은 모두
파싱되었습니다.

이런 격차가 순전히 우연으로 생길 수 있을까요? LLM 조건마다 게시물 40개의 점수를 Jev의 점수와
짝지어 놓고, 각 쌍 안에서 두 점수를 무작위로 맞바꾸기를 100,000번 반복한 다음, 맞바꾼 뒤의
격차가 실제 격차만큼 큰 경우가 얼마나 자주 나오는지 셉니다.

```python
signs = np.random.default_rng(0).choice([-1.0, 1.0], size=(100_000, NUM_POSTS))


def permutation_p_value(typesafe_values: list, other_values: list) -> float:
    """One-sided paired sign-flip permutation test: the share of random swaps at least as
    favourable to Jev as the measured gap."""
    gaps = np.array(other_values) - np.array(typesafe_values)
    null = (signs * gaps).mean(axis=1)
    return float(((null >= gaps.mean()).sum() + 1) / (len(null) + 1))


def p_text(p: float) -> str:
    return "p < 0.001" if p < 0.001 else f"p = {p:.3f}"


print(f"{'condition':<{name_w}}{'least stable option':>21}{'average':>12}")
for label in LABELS:
    # 비교 횟수를 곱해서(Bonferroni) 여러 번 검정하다 우연히 유의해지는 일을 막습니다
    worst_p = min(1.0, permutation_p_value(least_stable[TYPESAFE_LABEL], least_stable[label]) * len(LABELS))
    average_p = min(1.0, permutation_p_value(average_change[TYPESAFE_LABEL], average_change[label]) * len(LABELS))
    print(f"{label:<{name_w}}{p_text(worst_p):>21}{p_text(average_p):>12}")
```

```
condition                              least stable option     average
Claude Fable 5.1 (reasoning)                     p < 0.001   p < 0.001
Claude Opus 5.5 (reasoning)                      p < 0.001   p < 0.001
Claude Sonnet 5.5 (reasoning)                    p < 0.001   p < 0.001
Claude Haiku 5.5 (reasoning)                     p < 0.001   p < 0.001
GPT-6 Astra (reasoning)                          p < 0.001   p < 0.001
GPT-6.1 Sol (reasoning)                          p < 0.001   p < 0.001
GPT-6.1 Terra (reasoning)                        p < 0.001   p < 0.001
GPT-6 Luna (reasoning)                           p < 0.001   p < 0.001
GPT-6.1 Terra (temperature 0)                    p < 0.001   p < 0.001
GPT-6.1 Terra (default temperature)              p < 0.001   p < 0.001
GPT-6 Luna (temperature 0)                       p < 0.001   p < 0.001
GPT-6 Luna (default temperature)                 p < 0.001   p < 0.001
```

비교 12건을 감안해 12를 곱한 뒤에도 p-value는 모두 0.001 미만입니다. 이만큼 큰 격차가 우연으로
나올 가능성은 천 번 실행에 한 번도 되지 않습니다. temperature `0`은 LLM의 수치를 낮추지만,
temperature `0`의 GPT-6 Luna도 게시물 40개 중 21개에서 결정이 10번에 1번 넘게
바뀝니다.

그래프의 행 하나는 조건 하나이고 점 하나는 게시물 40개 중 하나입니다. 점은 다시 물었을 때 그
게시물의 가장 불안정한 결정이 바뀔 확률 위치에 찍히므로, 왼쪽에 있을수록 일관성이 높습니다.
한 행 안에서 점을 위아래로 흩어 놓은 것은 겹치지 않게 하려는 것일 뿐이며, 높이에는 아무 의미가
없습니다. 점선은 10번에 1번을 나타냅니다.

```python
STRIP_PALETTES = {
    "light": {"surface": "#ffffff", "ink": "#000000", "muted": "#808080", "jev": "#2b8cbe", "llm": "#fe9929", "alpha": 0.55},
    "dark": {"surface": "#0d0a0f", "ink": "#fafafa", "muted": "#a1a1aa", "jev": "#5fb0e0", "llm": "#fe9929", "alpha": 0.7},
}


def draw_strip(p: dict) -> plt.Figure:
    fig_strip, strip_ax = plt.subplots(figsize=(8, 0.5 * len(STRIP_ORDER) + 1), facecolor=p["surface"])
    strip_ax.set_facecolor(p["surface"])
    for position, label in enumerate(STRIP_ORDER):
        values = least_stable[label]
        # 값이 비슷한 점도 셀 수 있도록 게시물을 세로로 흩어 놓습니다
        strip_ax.scatter(
            values,
            position + np.linspace(-0.2, 0.2, len(values)),
            s=30,
            alpha=p["alpha"],
            color=p["jev"] if label == TYPESAFE_LABEL else p["llm"],
            edgecolors="none",
        )
    strip_ax.axvline(0.10, color=p["muted"], linewidth=0.8, linestyle="--")
    strip_ax.text(0.105, len(STRIP_ORDER) - 0.55, "1 in 10", fontsize=7, color=p["muted"], va="bottom")
    strip_ax.set_yticks(range(len(STRIP_ORDER)), STRIP_ORDER, fontsize=8)
    strip_ax.invert_yaxis()  # 첫 행이 맨 위에 오도록
    strip_ax.set_xlim(0, 0.6)
    strip_ax.xaxis.set_major_formatter(PercentFormatter(1.0))
    strip_ax.set_xlabel("chance that asking again changes the decision", fontsize=9, labelpad=16)
    for x, text, align in ((0, "← more consistent", "left"), (1, "less consistent →", "right")):
        # 축 아래로 고정된 간격을 두므로, 그림 높이가 어떻든 레이블이 x축 레이블 위에 놓입니다
        strip_ax.annotate(text, (x, 0), xycoords="axes fraction", xytext=(0, -18),
                          textcoords="offset points", ha=align, va="top", fontsize=8,
                          color=p["muted"])
    for edge in ("top", "right", "left"):
        strip_ax.spines[edge].set_visible(False)
    strip_ax.tick_params(length=0, labelcolor=p["ink"])
    strip_ax.spines["bottom"].set_color(p["ink"])
    strip_ax.xaxis.label.set_color(p["ink"])
    strip_ax.set_title(
        "Ask the same thing twice: how often does a decision change?\n"
        f"One dot per post ({NUM_POSTS} per row), at its least stable option out of "
        f"{sum(len(choices) for _instructions, choices in QUESTIONS.values())}",
        loc="left",
        fontsize=10,
        color=p["ink"],
    )
    fig_strip.tight_layout()
    return fig_strip


# 문서 빌드는 COOKSAFE_RENDER를 설정하고 다크 버전도 함께 그립니다.
for theme in ("light", "dark") if os.environ.get("COOKSAFE_RENDER") else ("light",):
    fig_strip = draw_strip(STRIP_PALETTES[theme])
    display(fig_strip, metadata={"theme": theme})
    plt.close(fig_strip)
```

![출력](https://mintcdn.com/ts-docs/orb6EiXzZ12Fo-od/cookbooks/consistency_choice_cookbook/consistency_choice_cookbook.executed.2.png?fit=max\&auto=format\&n=orb6EiXzZ12Fo-od\&q=85\&s=9ace390e28eeefd80b7d69fcbef27ebe)

## TypeSafe playground에서 열어 보기

아래 링크는 예시 게시물과 rubric을 playground에서 엽니다. 게시물 하나, 같은 `Choice` 질문
8개, 그리고 `jev-latest`입니다.

```python
playground_link = make_playground_link(
    {"post": POST},
    {
        key: Choice(instructions=instructions, criteria=choices)
        for key, (instructions, choices) in QUESTIONS.items()
    },
)
display(
    Markdown(
        f"🔗 [Open this post + rubric in the TypeSafe playground]({playground_link})"
    )
)
```

[TypeSafe playground에서 이 게시물 + rubric 열기 →](https://console.typesafe.ai/playground#share/N4IgJg9gxgrgtgUwHYBcAqCAeKQC4AEIwAOiAA4QDOKpBJ5VKA+gJZi36kAKAtABx8ATAEYAzKQA0nEAEMYKABYQATh3oxKCZa3Z5pGrUwAsRgOzDJ0mVCgQYqJjIDmCJmBkBPSh1F8ppMmUWFSZqIIBrBG89YX8QADMIABskiAB3LWiCEQAGAF8421QsGj16ShhleOsEDgCYACMklih8ZQQyJI9LUhYkJnbOjyYUCDrZJAhFLXwDZXwwBHjkMD6nfBl8JxlEfDIZFCgFHpBbODh7FhRuvVIsHc6EHm3EHkWGg6jSAukilGRSnRSP9sOMAILtfAeOz4TRBOyULr4RQsSgLZCaAD8+DBSGhSAQ+DSSnRyyQYDRigOewOR3wqI2+DOj3+TNSaSQG3JsKUMCSYHwDUJCDgH2UyhklE0AtG+AQADdkHtGPgFFoEAA6fBcBAQR74ZqRKEwmSQ4kHADkaOUMD6a2RCgZZwuSCuHi5AqUBOoSOhMHwBIQMoggsJixkSRQRKuCi1AGEILtKJ56VHZXB3dUoAgpH15VcEFa2iwnAoo2r2lqANIIDqp2ZkD34ACSFpScu5fvmxOShKOMiQBKSGpOCklTGaSHCHBQNpz0kn4TciZkfXG-ZQGvucEeJySA6cMGctVuIGQ3zigxUKCy+HUcIGHWvt6Ml6fymY7UlECQt4A2qQY4SlKiCoCclD7HAJyKF+pQALp5IhIBSOQyiJmQN4YNgehEKQUCfE4Kg3ECIDXGQJ4EHhSgtCecR9GEMBQCgwS-uMADqY5Rgy0ywmsSSEnAKoyGQnQtDITR9j+-yoDwFDNFA7r4f8hHKJmKgOgyFDUJiJxQEE-xBDIaikAAcj+FHSGZyrye6+bJAcLH4BA8Rcu64R9GAI7IdIAASpqSrepDNr+fI3k58yLIgA5ops5HKJQP5SGkMYBiGMEINSzmqqacBNpMexof8TFBjwUD7lKGwoCg1jhF5cR+f84JVTVMV7JkP7hVsaEwI2ECKvMsWFQgxUCv2ErFUE1AtPgAAU43zu0zROCxUguOSWhSBqW0AJR1dIABqwT8Ug2bjAAsjIkStXpQYsBJDqwU5LlAbl6l9FABZonZx3Zhs2z0VGCWIOZe2kAAypB4wAKq-skLQFgKgSJlMjnqYusKQUlKX5XFCVIBGlXVVAtUnGDWBHkk4xk5gFOdTIYChUyUkAl5iFxIELBwKawyTdOZTAh45HrtRp3eb0v6zoxzE-oFIAcdS3FqgVnPc-ggmLBK0ucrzDrUpQvL8gsQSKsihkuPgiTzCiaJaSgOli6c+laCwRn8yAfnAZQoGAtIOrxT+BMHET4SddVyguP8AriQY9KUkrXN9MWlAkw7h29idFmkGCD0ZVGWUvZ1fTMXAbAOR1isCau2uoincQADJ9OEYINAY4wAKLYFo+NJDw6Pqc58SyfuKCW7ltgqKs+Na0nIcV2rVcz6DIBgjYdioD5qKjKp4xcPCA2r-Y5ab0RnWDLnoZjnZ8xzwn1fJ0vdfpAASjX4xWVFrpIE48R8mrEAa2XO+s8bbtE0KgVmPxgSmgjsZUigtM6nBFrRaQAMbRMRYrLDiIZ1Lmi4nHSSxRUDKy5kEJEqx2gjUqvbQoTtDKwLMgSN+UwFgsAoZHSqrlzL0k5PsD8LQ+SmiXn7PG4wAAirDhrsMygSDGw0WDxGmh5Fg+YGYRiXgAcW6mQMREjKHUkGlMSRQYup2F6vMMqAUhHD1HjothxjqQ8WdJca4nVOgHFHqmTQSR4hSEmFGWK7UkAQLiNYLWsCyIIKOBAGiJxUFSwwexTiHZLbZm9hsdBHV9Z2ENkKZEl0lQdWtsqbSulaEu1gWCFI6Rxh1wyibHitsGzRkUKldJWsl5sVNEgGpdTCQNJVD1QU8hCbWAUIyNIXT7T7iFMOE4T8RR9QQfMwS9Sla2ykC3KMkBUpRnIt3FgAAvPpStrC2EPkvMGs4WCRHGMsxZDpCSNIHFHMAUdYRXKNLKHipy16bhOG3Sg+Fh4ILQPk1pchRhcykRkpAABuVUyQBSW0ZAoeAA50QfUoCxYJ0gACOMAEAEvCfA4W0TRZ0QlmgrWmDHR0ixV-fif8AHT3xYSwkWS+QCnSJyIpttqG-DKa7EiYJ5BjFPCK0YPBQHJEVPC-KqKuaclZQSgMtZFieROOo5AWgIzjC1QSCUSQmU6pZQSglS80AKFguMS1j0AD0+BvrIF+qAigv5CTKs1KTSGp4IY7CbG4keKg4A8GjpofAnqLVg1qfTG1NpqChvJDwZMywXH8XpvgB1YDgjzEjd8SBIBFxMDHOSScThiVC1PFEmJDs4kwtlj5dIPJslRzxHKTuyhu4Gkbp1AeQ93HBu4fmVkid+nUFDNlUtQZ+V4UFRUqpaQekyFWYSdGic3HZiXk-OADcpxUyuWYlhyZ7o8XRps-AkQ6xjr+Q7AAQkEbYiw43n1PT2yUTMJ59E+AKBoD76b2meW0xyVwNgxRbpoJeAKgWfCpisB53apzIhDJsCCciIybyQ6BzQFUhSW0JKEtYOLSDtHzAgNITB9iKArZEpBsTKXxJlokkMHLDZc0+UrBRXagVGsKY6G2jAZ2OwLHQt2EqxWUWXqKqF9iYVypDAq9FJGWBkaXj5NF3TxUWzQqgSchIFOcnVia9SbGojwe4xc5AObwSwks-3eYqGProfHUpsjMwGTtHxRIjVDtanbEpqeJ+dhWSyn4n5zq+40g8GQCkkUAI5SAojIAojIBND9TdNR0lNaKUMXrYkhW+CnL9QjEa1LzsXFZV5QJ0pwnyluwYQgqy30ksnEfougLMBFidVWF7VEWLFQbOGVEMBzFivuhevxKUS8zq3XgDvQJBMgIBTSclFp+UyoZWUEiG6qwT1WtzqpksxxTweyW3F0YEcKzNLGZsdKDiklnJyYSL8Uc0TbbuvxCBeRvKyDICwfamQWI4XlMIEAeQgA)


이 문서는 개발자 문서 플랫폼인 [Mintlify](https://mintlify.com)로 제작하고 호스팅합니다.
