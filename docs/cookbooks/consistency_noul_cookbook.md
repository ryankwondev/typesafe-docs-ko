# Self-consistency(자기 일관성): noul

> 같은 보험금 청구 건을 모델에 두 번 물으면 결정이 얼마나 자주 바뀔까요? 정답지 없이도 할 수 있는 신뢰성 테스트로, 경계선상에 있는 청구 건을 놓고 LLM과 Jev를 비교합니다.

모델에 결정을 맡기는 pipeline(파이프라인)에서는 같은 청구 건에 매번 같은 결정이 나와야 합니다. 이 cookbook(예제 모음)에서는 청구 건 하나를 바꾸지 않고 여러 번 반복해서 보내면서, 지급할지, 거절할지, 사람에게 넘길지의 결정이 얼마나 자주 바뀌는지 셉니다.

이 테스트는 결정이 옳은지에 대해서는 아무것도 말해 주지 않습니다. 그보다 앞선 문제를 봅니다. 청구 건은 그대로인데 결정이 움직인다면, 이의가 제기된 결정을 재현할 수 없고 똑같은 청구 건이 한 번은 지급되고 한 번은 거절될 수 있습니다. 정답지 없이 반복 호출만 있으면 되므로 모든 모델을 같은 조건에서 측정할 수 있습니다.

`Noul` 질문 14개는 각각 확률로 돌아옵니다. 예를 들어 손해가 보장 대상일 확률이 0.62라는 식입니다. pipeline은 cutoff(기준값)를 두어 이 확률을 결정으로 바꿉니다. 0.6보다 높으면 지급하는 식입니다. 다음 요청에서 0.55가 돌아오면 같은 청구 건인데도 지급되지 않습니다. 이런 흔들림이 결정을 바꾸는지는 cutoff가 어디에 있느냐에 달려 있는데, 여러분의 cutoff가 어디인지는 알 수 없으므로 0부터 1까지 모든 cutoff를 똑같은 비중으로 셉니다. 두 답이 0.10만큼 떨어져 있으면 전체 cutoff의 10%에서 결정이 갈립니다. 이 값을 반복 호출의 모든 쌍에 대해 평균하면 질문마다 숫자 하나가 나옵니다. 다시 물었을 때 결정이 바뀔 확률입니다.

명확한 청구 건은 어느 모델에나 쉬우므로 이 테스트에서는 경계선상에 있는 청구 건을 씁니다. 청구 건 하나를 40가지 버전으로 만들었고, 버전마다 청구 심사 모델이 저울질해야 하는 사실의 조합이 다릅니다. pipeline의 신뢰도는 가장 불안정한 결정이 좌우하므로, 각 청구 건의 점수는 가장 불안정한 질문을 기준으로 매깁니다.

조건은 다음과 같고, 조건 하나는 모델 하나와 설정 하나의 조합입니다.

* Claude 모델 `claude-fable-5-1`, `claude-opus-5-5`, `claude-sonnet-5-5`, `claude-haiku-5-5`와 GPT 모델 `gpt-6-astra`, `gpt-6.1-sol`, `gpt-6.1-terra`, `gpt-6-luna`. 모두 높은 effort로 reasoning(추론)을 수행합니다.
* reasoning을 끈 `gpt-6.1-terra`와 `gpt-6-luna`. temperature(온도)는 `0`과 API 기본값 두 가지입니다. 여기서 temperature를 받는 모델은 이 둘뿐입니다.
* Jev 1.14(`jev-1.14.0`): `Noul` 질문 14개를 담은 `system_one` 호출 한 번.

Jev에서는 청구 건의 가장 불안정한 결정이 평균 3.8%의 확률로 바뀝니다. 약 25번에 1번꼴이고, 가장 높은 경우에도 8.4%를 넘지 않습니다. 가장 안정적인 LLM 조건은 평균적인 청구 건에서 이에 근접합니다. 둘 다 reasoning을 켠 GPT-6.1 Terra가 6.2%, Claude Opus 5.5가 6.4%입니다. 하지만 모든 LLM 조건에는 결정이 10번에 1번보다 자주 바뀌는 청구 건이 있고, 나머지 조건의 평균은 7.7%에서 34.8% 사이입니다. temperature `0`은 격차를 줄이지만 없애지는 못합니다. 가장 불안정한 질문을 기준으로 보면, Jev가 어느 LLM 조건에 대해서든 보인 만큼의 우위가 순전히 우연으로 나올 가능성은 100번 실행에 1번 미만입니다.

## 준비

```bash
pip install anthropic openai matplotlib ipython 'cooksafe>=0.3.0,<0.4.0'
```

그런 다음 `TYPESAFE_API_KEY`, `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`를 설정합니다. 아래 수치는 2026-11-02에 `jev-1.14.0`으로 얻은 결과입니다. 40가지 버전에 대한 LLM 답은 2026-11-07부터 2026-11-09까지, 예제 청구 건에 대한 답은 2026-11-10에 수집했습니다.

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
from matplotlib.ticker import PercentFormatter
from cooksafe import JsonCache, make_playground_link
from IPython.display import Markdown, display
from openai import OpenAI
from typesafe_sdk import Noul, TypeSafeClient

matplotlib.use("Agg")  # 화면 없이 렌더링

LLM_MODELS = {  # 모두 높은 effort로 reasoning을 수행합니다
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
TYPESAFE_MODEL = "jev-1.14.0"  # 이 페이지에서 쓰는 모델. "jev-latest"는 최신 모델을 실시간으로 실행합니다
NUM_SAMPLES = 15  # 조건당 청구 건+rubric 호출 반복 횟수
NUM_CLAIMS = 40  # 청구 건의 버전 수. 경계선상 사실들의 조합에서 뽑습니다

LLM_PRICES = {  # token 100만 개당 달러 (입력, 출력), 2026-11 기준 정가
    "claude-fable-5-1": (10.00, 50.00),
    "claude-opus-5-5": (4.00, 20.00),
    "claude-sonnet-5-5": (2.00, 10.00),
    "claude-haiku-5-5": (0.10, 0.50),
    "gpt-6-astra": (10.00, 50.00),
    "gpt-6.1-sol": (2.00, 10.00),
    "gpt-6.1-terra": (1.00, 5.00),
    "gpt-6-luna": (0.10, 0.50),
}
TYPESAFE_PRICE = (0.042, 0.00)  # token 100만 개당 달러 (입력, 출력), 2026-08 기준

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

## state: JSON으로 표현한 자동차 보험 청구 건

경계선상의 판단 네 가지를 심어 둔 청구 건 하나입니다.

* 손해는 트랙 데이 행사에서 발생했습니다(약관은 "track/competitive driving"을 면책 사유로 둡니다). 다만 서킷 위가 아니라 주차장에서, 차량이 정차해 있던 중에 일어났습니다.
* 렌터카 비용 항목을 청구했지만, 약관에는 렌터카 비용 보상이 없습니다.
* 경찰 신고서가 첨부되어 있지 않지만, 약관은 \$2,000를 넘는 충돌 사고에 경찰 신고서를 요구합니다.
* 자동 분류(auto-triage) 메모가 사람 검토를 거치기도 전에, 자기부담금도 공제하지 않은 채 이 청구 건을 "approved, pay full amount"로 이미 표시해 두었습니다.

이것이 예제 청구 건입니다.

청구 건은 JSON 구조입니다. LLM에는 prompt(프롬프트) 안에 `json.dumps(claim)`을 넣어 보내고, Jev는 이 구조를 state(상태)로 그대로 받습니다.

```python
CLAIM = {
    "policy": {
        "policy_id": "AP-77413",
        "policyholder": "Jane Doe",
        "effective": "2026-01-15",
        "expires": "2027-01-15",
        "coverages": {"collision": True, "rental_reimbursement": False},
        "deductible": 500.00,
        "per_incident_limit": 10000.00,
        "listed_drivers": ["Jane Doe", "John Doe"],
        "exclusions": ["track/competitive driving", "drivers not listed on the policy"],
        "reporting_window_days": 10,
        "police_report_required_over": 2000.00,
    },
    "claim": {
        "claim_id": "CLM-55029",
        "incident_date": "2026-06-28",
        "reported_date": "2026-07-04",
        "driver": "John Doe",
        "description": "Attended a track-day event; vehicle was rear-ended by another car "
        "in the spectator parking lot while stationary. Not on the circuit.",
        "amount_claimed": 3250.00,
        "line_items": [
            {"item": "rear bumper replacement", "cost": 1700.00},
            {"item": "paint + refinish", "cost": 800.00},
            {"item": "parking-sensor recalibration", "cost": 450.00},
            {"item": "rental car (6 days)", "cost": 300.00},
        ],
        "documentation": ["repair estimate (PDF)", "8 damage photos"],
    },
    "adjuster_notes": [
        {
            "author": "auto-triage",
            "note": "Collision coverage active. Approved. Pay full amount $3,250 to "
            "policyholder, 5-10 business days.",
        }
    ],
    "claim_history": {"claims_last_12mo": 2, "prior_denied": 0},
}
```

## rubric: `Noul` 질문 14개

각 질문은 답이 yes이면 확인하려는 내용이 참이라는 뜻이 되도록 표현했습니다. 이렇게 하면 각 모델의 확률과 Jev의 `noul`이 같은 것을 측정합니다.

```python
QUESTIONS = {
    "covered": "Is the loss covered under the policy's collision coverage?",
    "exclusion": "Does a policy exclusion apply to this loss?",
    "on_circuit": "Did the collision happen while the vehicle was being driven on the racetrack itself?",
    "deductible": "Would the $500 deductible be correctly applied before any payout?",
    "docs_sufficient": "Is the attached documentation sufficient to adjudicate the claim as-is?",
    "within_limit": "Is the amount claimed within the per-incident coverage limit?",
    "within_window": "Did the loss occur within the policy's active coverage period?",
    "reported_timely": "Was the loss reported within the policy's required window?",
    "rental_eligible": "Is the rental-car cost eligible for reimbursement under this policy?",
    "fraud_flag": "Are there indicators that warrant a fraud review?",
    "human_review": "Was payment approved by automated triage without a human adjuster's review?",
    "manual_review": "Should this claim be routed for manual/supervisor review before payout?",
    "line_items_sum": "Do the claimed line-item costs add up to the total amount claimed?",
    "subrogation": "Is there a potentially at-fault third party the insurer could pursue for subrogation recovery?",
}
QUESTIONS_JSON = json.dumps(QUESTIONS)
```

## 질문하는 방식

LLM 호출 하나는 `json.dumps(claim)`과 질문 14개를 모두 담은 prompt 하나입니다. 모델은 structured output(구조화된 출력)으로 답합니다. 질문의 키마다 숫자 하나를 요구하는 JSON 스키마입니다. 호출은 모델 이름에 따라 Anthropic이나 OpenAI로 보냅니다. reasoning은 `high` effort로 실행하고, reasoning을 끈 `gpt-6.1-terra`와 `gpt-6-luna`는 `temperature`를 `0` 또는 API 기본값으로 받습니다.

TypeSafe 호출은 같은 청구 건과 같은 `Noul` 질문 14개를 담은 `system_one` 요청 하나입니다. 각 답의 `noul`은 P(true)입니다.

모든 요청에는 새 `uid`도 붙습니다. 한 번 쓰고 버리는 고유한 값으로, 실행할 때마다 바뀌지만 청구 건과 rubric(루브릭: 채점 기준)은 그대로 둡니다. LLM prompt 안에 들어가고, TypeSafe state에는 추가 필드로 들어갑니다. 이 값 때문에 모든 요청이 서로 달라지므로, 중간의 어떤 캐시도 이전 답을 돌려줄 수 없습니다.

호출마다 비용과 왕복 시간도 기록합니다.

```python
def rubric_prompt(sample_index: int, claim: dict, questions: dict) -> str:
    """The claim + all questions in one prompt, asking for a probability per question.
    ``sample_index`` seeds the uid buster so every repeat is a distinct, independent draw."""
    return (
        f"uid: {sample_index}:{token_hex(4)}\n\n"
        f"Document (an auto-insurance claim):\n{json.dumps(claim, indent=2)}\n\nQuestions:\n"
        + "\n".join(f"- {key}: {question}" for key, question in questions.items())
        + "\n\nFor each question, give your probability that the answer is yes.\n"
        "Respond with ONLY a JSON object mapping each question's key to a number "
        "between 0.00 and 1.00, with one entry per question."
    )


def _cost(prices: tuple[float, float], input_tokens: int, output_tokens: int) -> float:
    return input_tokens / 1e6 * prices[0] + output_tokens / 1e6 * prices[1]


def _answer_schema(questions: dict) -> dict:
    """Structured output: exactly one number per question key."""
    return {
        "type": "object",
        "properties": {key: {"type": "number"} for key in questions},
        "required": list(questions),
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


def _parse_answer(answer: object) -> float:
    """One raw per-question answer -> a probability; NaN if missing or not a number."""
    try:
        return float(answer)
    except (TypeError, ValueError):
        return float("nan")


def _query_llm(model, reasoning, temperature, sample_index, claim_json, questions_json):
    """One LLM rubric query -> (probabilities keyed by question key, cost_usd, latency_s)."""
    questions = json.loads(questions_json)
    prompt = rubric_prompt(sample_index, json.loads(claim_json), questions)
    text, cost, latency = _call_llm(model, prompt, _answer_schema(questions), reasoning,
                                    temperature)
    try:
        raw = json.loads(text)
    except (ValueError, json.JSONDecodeError):
        raw = {}
    raw = raw if isinstance(raw, dict) else {}
    return {key: _parse_answer(raw.get(key)) for key in questions}, cost, latency


def _query_typesafe(sample_index, claim_json, questions_json, model):
    """One TypeSafe call -> (nouls keyed by question key, input tokens, output tokens, latency)."""
    questions = json.loads(questions_json)
    started = perf_counter()
    response = typesafe_client.system_one(
        model=model,
        state={"uid": f"{sample_index}:{token_hex(4)}", "claim": json.loads(claim_json)},
        questions={key: Noul(instructions=question) for key, question in questions.items()},
    )
    nouls = {key: response.answers[key].noul for key in questions}
    return (
        nouls,
        response.usage.input_tokens,
        response.usage.output_tokens,
        perf_counter() - started,
    )


# 모든 샘플이 캐시 파일에 저장되므로 두 번째 실행부터는 요청을 보내지 않습니다. 청구 건과
# 질문이 모든 키에 포함되므로 둘 중 하나를 수정하면 다시 묻습니다. ``sample_index``도
# 키에 포함되므로 NUM_SAMPLES번의 반복은 각각 독립적인 추출입니다.

EXAMPLES_URL = (
    "https://raw.githubusercontent.com/typesafe-ai/typesafe-public-examples/main/"
    "cookbooks/consistency_noul_cookbook"
)


def download(name: str) -> Path:
    """Fetch one of this cookbook's files from typesafe-public-examples unless it is on disk."""
    path = Path(name)
    if not path.exists():
        urllib.request.urlretrieve(f"{EXAMPLES_URL}/{name}", path)
    return path


CACHE = Path("consistency_noul_cookbook_cache.json")
# True이면 공개된 답으로 CACHE를 채우므로 실행 중에 요청을 보내지 않습니다. 실시간으로
# 실행하려면 False로 바꾸고 CACHE를 삭제하세요. 디스크에 이미 있는 CACHE는 덮어쓰지 않습니다.
DOWNLOAD_CACHE = True
if DOWNLOAD_CACHE:
    download(CACHE.name)
json_cache = JsonCache(CACHE)


@json_cache
def ask_llm(model: str, reasoning: str, temperature: float | None, sample_index: int,
            claim_json: str, questions_json: str):
    """The example claim's LLM answers, with cost and latency."""
    return _query_llm(model, reasoning, temperature, sample_index, claim_json, questions_json)


@json_cache
def ask_typesafe(sample_index: int, claim_json: str, questions_json: str, model: str):
    """The example claim's Jev answers, with token usage and latency."""
    return _query_typesafe(sample_index, claim_json, questions_json, model)


@json_cache
def sweep_llm(model: str, reasoning: str, temperature: float | None, sample_index: int,
              claim_json: str, questions_json: str):
    """One LLM answer set for one version of the claim."""
    return _query_llm(model, reasoning, temperature, sample_index, claim_json, questions_json)[0]


@json_cache
def sweep_typesafe(sample_index: int, claim_json: str, questions_json: str, model: str):
    """One Jev answer set for one version of the claim."""
    return _query_typesafe(sample_index, claim_json, questions_json, model)[0]
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

* 체크 표시 하나가 조건 하나이고, 청구 건마다 15번씩 실행했습니다. 하이픈(-)은 테스트하지 않은 조합입니다.
* temperature 열은 reasoning을 끈 상태입니다. Claude 모델은 temperature를 받지 않고, GPT 모델도 reasoning을 수행하는 동안에는 받지 않습니다.
* 기본값 열에서는 temperature 인자를 보내지 않습니다.
* 반복 가능성을 높이려면 temperature `0`을 쓰라는 것이 일반적인 조언이므로, 모델이 temperature를 받는 경우에는 이를 API 기본값과 비교합니다.

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
# 예제 그래프의 행으로, self-consistency 페이지 두 곳에서 같습니다. Jev와 조건 몇 개만
# 넣었습니다. 조건을 모두 넣으면 그래프가 너무 길어져 읽기 어렵기 때문입니다.
PLOT_ORDER = [
    TYPESAFE_LABEL,
    "Claude Opus 5.5 (reasoning)",
    "Claude Haiku 5.5 (reasoning)",
    "GPT-6.1 Sol (reasoning)",
    "GPT-6 Luna (temperature 0)",
    "GPT-6 Luna (default temperature)",
]
STRIP_ORDER = [TYPESAFE_LABEL, *LABELS]  # 마지막 그래프의 행: 모든 조건


CLAIM_JSON = json.dumps(CLAIM)

runs: dict[str, list] = {}  # 라벨 -> {질문 키: 확률} 샘플 NUM_SAMPLES개
stats: dict[str, list] = {}  # 라벨 -> (cost_usd, latency_s) 쌍 NUM_SAMPLES개
with ThreadPoolExecutor(max_workers=16) as pool:
    futures = {
        condition["label"]: [
            pool.submit(ask_llm, condition["model"], condition["reasoning"],
                        condition["temp"], sample_index, CLAIM_JSON, QUESTIONS_JSON)
            for sample_index in range(NUM_SAMPLES)
        ]
        for condition in CONDITIONS
    }
    for label, sample_futures in futures.items():
        results = [future.result() for future in sample_futures]
        runs[label] = [result[0] for result in results]
        stats[label] = [(result[1], result[2]) for result in results]

# Jev 샘플은 LLM 호출이 끝난 뒤 순차적으로 뽑으므로, 각 지연 시간은 다른 호출의 영향을
# 받지 않은 왕복 시간입니다.
typesafe_usage_results = [
    ask_typesafe(sample_index, CLAIM_JSON, QUESTIONS_JSON, model=TYPESAFE_MODEL)
    for sample_index in range(NUM_SAMPLES)
]
# 가격은 캐시에서 값을 꺼낸 뒤에 적용하므로, 가격이 바뀌어도 샘플을 새로 뽑을 필요가 없습니다.
runs[TYPESAFE_LABEL] = [result[0] for result in typesafe_usage_results]
stats[TYPESAFE_LABEL] = [
    (_cost(TYPESAFE_PRICE, input_tokens, output_tokens), latency)
    for _nouls, input_tokens, output_tokens, latency in typesafe_usage_results
]
```

### 비용과 속도 (rubric 요청 한 번 기준)

비용은 준비 섹션의 token(토큰)당 가격으로 계산합니다. LLM은 2026-11 기준 정가, TypeSafe는 2026-08 기준 요금입니다.

한 행은 예제 청구 건에 대해 질문 14개짜리 rubric 전체를 호출한 것 하나입니다. `time/call`과 `cost/call`은 15번 호출의 평균이고, `vs Jev 1.14` 열은 Jev 1.14의 수치로 나눈 값입니다.

```python
typesafe_cost = mean([cost for cost, _latency in stats[TYPESAFE_LABEL]])
typesafe_latency = mean([latency for _cost, latency in stats[TYPESAFE_LABEL]])
name_w = max(len(name) for name in ALL_LABELS) + 2
# 비교 헤더를 두 줄로 쌓아서 상대 속도와 비용 열을 좁게 유지합니다.
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
Claude Fable 5.1 (reasoning)              15    10065ms    $0.055738     104.3x    1231.8x
Claude Opus 5.5 (reasoning)               15     6048ms    $0.017619      62.7x     389.4x
Claude Sonnet 5.5 (reasoning)             15     5531ms    $0.010128      57.3x     223.8x
Claude Haiku 5.5 (reasoning)              15     4464ms    $0.000621      46.2x      13.7x
GPT-6 Astra (reasoning)                   15    16440ms    $0.038681     170.3x     854.8x
GPT-6.1 Sol (reasoning)                   15     8311ms    $0.005624      86.1x     124.3x
GPT-6.1 Terra (reasoning)                 15     4275ms    $0.002307      44.3x      51.0x
GPT-6 Luna (reasoning)                    15     6133ms    $0.000330      63.5x       7.3x
GPT-6.1 Terra (temperature 0)             15     2464ms    $0.001483      25.5x      32.8x
GPT-6.1 Terra (default temperature)       15     2527ms    $0.001483      26.2x      32.8x
GPT-6 Luna (temperature 0)                15     1656ms    $0.000148      17.2x       3.3x
GPT-6 Luna (default temperature)          15     2092ms    $0.000148      21.7x       3.3x
Jev 1.14                                  15       97ms    $0.000045       1.0x       1.0x
```

이번 실행에서 Jev의 평균 왕복 지연 시간은 97ms입니다. LLM 조건은 위의 동시 실행 설정에서 호출당 1.7초에서 16.4초 사이입니다.

## 그래프: 예제 청구 건의 모든 샘플

읽는 법은 다음과 같습니다.

* 바깥쪽 행 그룹: 질문.
* 안쪽 행: 조건.
* 열: rubric 전체 호출 한 번.
* 셀 색: 빨강은 P(yes)가 1 근처, 노랑은 0.5 근처, 초록은 0 근처입니다.

```python
rows_per_block = len(PLOT_ORDER)  # 질문 블록당 행 수
GAP = 1  # 질문 블록 사이에 넣는 빈 간격 행 수
row_values, row_labels, blocks = [], [], []
for question_index, (question_key, question_text) in enumerate(QUESTIONS.items()):
    if question_index:  # 빈 간격 행(NaN -> 흰색으로 렌더링)으로 블록을 구분합니다
        row_values.extend([np.nan] * NUM_SAMPLES for _ in range(GAP))
        row_labels.extend([""] * GAP)
    blocks.append(
        (len(row_values), question_key, question_text)
    )  # (이 블록의 첫 행, 질문 키, 질문 텍스트)
    for label in PLOT_ORDER:
        row_values.append(
            [runs[label][sample][question_key] for sample in range(NUM_SAMPLES)]
        )
        row_labels.append(label)
heatmap_matrix = np.array(row_values)
cmap = plt.get_cmap("RdYlGn_r").copy()  # 빨강 = P(yes)가 높음, 초록 = P(yes)가 낮음
cmap.set_bad("white")  # 간격(NaN) 행은 빈칸으로 렌더링됩니다

fig, ax = plt.subplots(figsize=(15, 0.26 * len(row_values) + 1))
ax.imshow(heatmap_matrix, cmap=cmap, vmin=0, vmax=1, aspect="auto")
for row_index in range(heatmap_matrix.shape[0]):
    for col_index in range(heatmap_matrix.shape[1]):
        value = heatmap_matrix[row_index, col_index]
        if np.isnan(value):
            continue
        ax.text(
            col_index,
            row_index,
            f"{value:.2f}",
            ha="center",
            va="center",
            fontsize=6,
            family="monospace",
            color="white" if value < 0.22 or value > 0.78 else "black",
        )

ax.set_xticks(range(NUM_SAMPLES))
ax.set_xticklabels(range(1, NUM_SAMPLES + 1), fontsize=7)
ax.set_xlabel("rubric query")
ax.set_yticks(range(len(row_labels)))
ax.set_yticklabels(row_labels, fontsize=7)
ax.tick_params(length=0)
for edge in ("top", "right", "left", "bottom"):
    ax.spines[edge].set_visible(False)

# 다중 인덱스의 바깥쪽 단계: 질문 키를 블록마다 한 번씩 가운데에 출력하고, 바로 아래에
# 질문 텍스트를 줄바꿈해서 붙입니다
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
    "Every sample on the example claim",
    pad=12,
)
fig.tight_layout()
display(fig)
```

![출력](https://mintcdn.com/ts-docs/orb6EiXzZ12Fo-od/cookbooks/consistency_noul_cookbook/consistency_noul_cookbook.executed.1.png?fit=max\&auto=format\&n=orb6EiXzZ12Fo-od\&q=85\&s=e5d016c80052dd4745a848de7c5154ca)

Jev는 모든 질문을 좁은 범위 안에 유지하고, 가장 많이 변하는 질문은 `exclusion`(0.37에서 0.53)입니다. 다른 조건에서는 사실 확인 질문조차 움직입니다. GPT-6 Luna는 temperature `0`에서도 기본값에서와 마찬가지로 `line_items_sum`을 0부터 거의 1까지 아무 데나 둡니다. 판단이 필요한 질문에서는 GPT-6.1 Sol이 `deductible`을 0.03부터 1.00까지 아무 데나 둡니다.

## 청구 건의 40가지 버전

모델은 어떤 청구 건에서는 안정적이다가 다음 청구 건에서는 흔들릴 수 있으므로, 청구 건 하나만으로는 모델을 판단하기에 부족합니다. 청구 건의 각 버전은 청구 심사 모델이 저울질해야 하는 사실 여섯 가지를 정합니다.

* 충돌이 일어난 장소: 기록 측정 세션 중 서킷 위, 걷는 속도로 움직이던 피트 레인, 관람객 주차장(예제 청구 건과 같음), 식료품점 주차장 중 하나
* 경찰 신고서가 첨부되어 있는지
* 약관이 렌터카를 보장하는지
* 운전자가 약관에 등재되어 있는지
* 손해를 10일 기한 안에 신고했는지
* 청구 금액이 \$10,000 한도를 넘는지

이렇게 하면 조합이 128개 나옵니다. 고정된 시드로 그중 40개를 뽑고, 각각을 모든 조건에서 15번씩 보냅니다.

```python
LOCATIONS = {
    "circuit": "Attended a track-day event; vehicle was rear-ended by another car during a "
    "timed session on the circuit.",
    "pit-lane": "Attended a track-day event; vehicle was rear-ended by another car in the pit "
    "lane, moving at walking pace towards the circuit entry.",
    "spectator-lot": CLAIM["claim"]["description"],
    "grocery-lot": "Vehicle was rear-ended by another car in a grocery-store parking lot while "
    "stationary.",
}
OVER_LIMIT_ITEMS = [
    {"item": "rear bumper + frame repair", "cost": 11200.00},
    {"item": "paint + refinish", "cost": 800.00},
    {"item": "parking-sensor recalibration", "cost": 450.00},
    {"item": "rental car (6 days)", "cost": 300.00},
]
SWITCHES = ("police", "rental", "unlisted", "late", "over")


def build_claim(location: str, police: bool, rental: bool, unlisted: bool, late: bool,
                over: bool) -> dict:
    """The example claim with the six borderline facts set one way."""
    claim = copy.deepcopy(CLAIM)
    claim["claim"]["description"] = LOCATIONS[location]
    if police:
        claim["claim"]["documentation"] = [*claim["claim"]["documentation"], "police report"]
    claim["policy"]["coverages"]["rental_reimbursement"] = rental
    if unlisted:
        claim["claim"]["driver"] = "Alex Roe"
    if late:
        claim["claim"]["incident_date"] = "2026-06-10"  # 신고일보다 24일 전
    if over:
        claim["claim"]["line_items"] = OVER_LIMIT_ITEMS
        claim["claim"]["amount_claimed"] = 12750.00
    return claim


combinations = list(itertools.product(LOCATIONS, *[(False, True)] * len(SWITCHES)))
CLAIMS = {
    "-".join([combo[0], *(name for name, on in zip(SWITCHES, combo[1:]) if on)]): build_claim(*combo)
    for combo in random.Random(0).sample(combinations, NUM_CLAIMS)
}

# 청구 건 40개 x 조건 13개 x 반복 15번. 실시간으로 실행하면 몇 시간이 걸리고, 대부분은 모델이
# 높은 effort로 reasoning을 수행하는 시간입니다.
with ThreadPoolExecutor(max_workers=16) as pool:
    sweep = {
        claim_id: {
            **{
                condition["label"]: [
                    pool.submit(sweep_llm, condition["model"], condition["reasoning"],
                                condition["temp"], index, json.dumps(claim), QUESTIONS_JSON)
                    for index in range(NUM_SAMPLES)
                ]
                for condition in CONDITIONS
            },
            TYPESAFE_LABEL: [
                pool.submit(
                    sweep_typesafe, index, json.dumps(claim), QUESTIONS_JSON, model=TYPESAFE_MODEL
                )
                for index in range(NUM_SAMPLES)
            ],
        }
        for claim_id, claim in CLAIMS.items()
    }
    sweep = {
        claim_id: {label: [future.result() for future in futures] for label, futures in by_label.items()}
        for claim_id, by_label in sweep.items()
    }
```

## 다시 물으면 결정이 얼마나 자주 바뀌는가

청구 건의 질문마다 답 15개를 가능한 모든 쌍, 즉 105쌍으로 비교하고, 두 확률이 얼마나 떨어져 있는지를 평균합니다. 이 평균이 모든 cutoff를 똑같은 비중으로 셌을 때, 다시 물으면 그 질문의 결정이 바뀔 확률입니다. 그런 다음 각 청구 건은 가장 불안정한 질문의 점수를 자기 점수로 가져갑니다. 표에는 이 점수를 평균적인 청구 건과 최악의 청구 건에 대해 싣고, 10번에 1번을 넘는 청구 건의 수를 세고, 좀 더 너그러운 척도로 질문 14개 전체의 평균도 함께 싣습니다.

```python
def change_chances(samples: list) -> dict:
    """Per question: the mean distance between the probabilities of every pair of repeats,
    which is the chance that asking again changes the decision (parse failures left out)."""
    chances = {}
    for question_key in QUESTIONS:
        values = np.array([sample[question_key] for sample in samples], dtype=float)
        values = values[~np.isnan(values)]
        first, second = np.triu_indices(len(values), k=1)
        chances[question_key] = float(np.abs(values[first] - values[second]).mean())
    return chances


average_change = {label: [] for label in ALL_LABELS}  # 라벨 -> 청구 건별 질문 전체 평균
least_stable = {label: [] for label in ALL_LABELS}  # 라벨 -> 청구 건별 가장 불안정한 질문
for by_label in sweep.values():
    for label in ALL_LABELS:
        chances = change_chances(by_label[label])
        average_change[label].append(mean(chances.values()))
        least_stable[label].append(max(chances.values()))
ranked = sorted(ALL_LABELS, key=lambda label: mean(least_stable[label]))

print(
    f"{'':<{name_w}}{'average over':>14}{'least stable question':>26}{'claims above':>14}\n"
    f"{'condition':<{name_w}}{'questions':>14}{'average claim':>15}{'worst claim':>13}"
    f"{'1 in 10':>12}"
)
for label in ranked:
    print(
        f"{label:<{name_w}}{mean(average_change[label]):>14.1%}"
        f"{mean(least_stable[label]):>15.1%}{max(least_stable[label]):>13.1%}"
        f"{sum(value > 0.10 for value in least_stable[label]):>9}/{NUM_CLAIMS}"
    )
```

```
                                       average over     least stable question  claims above
condition                                 questions  average claim  worst claim     1 in 10
Jev 1.14                                       1.2%           3.8%         8.4%        0/40
GPT-6.1 Terra (reasoning)                      1.2%           6.2%        25.9%        6/40
Claude Opus 5.5 (reasoning)                    1.7%           6.4%        10.4%        2/40
Claude Fable 5.1 (reasoning)                   2.0%           7.7%        14.9%        6/40
GPT-6.1 Sol (reasoning)                        1.3%          11.2%        32.9%       22/40
GPT-6.1 Terra (temperature 0)                  1.7%          13.8%        50.3%       20/40
Claude Haiku 5.5 (reasoning)                   4.0%          17.6%        36.2%       33/40
GPT-6 Astra (reasoning)                        1.8%          18.6%        51.1%       29/40
GPT-6.1 Terra (default temperature)            2.7%          19.7%        50.2%       28/40
GPT-6 Luna (reasoning)                         4.7%          27.6%        49.0%       40/40
Claude Sonnet 5.5 (reasoning)                  6.1%          29.3%        52.0%       40/40
GPT-6 Luna (temperature 0)                     4.7%          31.1%        51.4%       39/40
GPT-6 Luna (default temperature)               6.5%          34.8%        53.3%       40/40
```

Jev에서는 가장 불안정한 결정이 평균적인 청구 건에서 3.8%, 최악의 청구 건에서 8.4%의 확률로 바뀌고, 10번에 1번을 넘는 청구 건은 하나도 없습니다. LLM 조건에서는 평균적인 청구 건에서 6.2%에서 34.8%의 확률로 바뀌고, 10번에 1번을 넘는 청구 건이 적게는 2개에서 많게는 40개 전부입니다. 질문 14개 전체로 평균하면 격차가 사라집니다. reasoning을 켠 GPT-6.1 Terra가 Jev의 1.2%와 같습니다.

이런 격차가 순전히 우연으로 생길 수 있을까요? LLM 조건마다 청구 건 40개의 점수를 Jev의 점수와 짝지어 놓고, 각 쌍 안에서 두 점수를 무작위로 맞바꾸는 일을 100,000번 반복한 다음, 맞바꾼 결과의 격차가 실제 격차만큼 큰 경우가 얼마나 자주 나오는지 셉니다.

```python
signs = np.random.default_rng(0).choice([-1.0, 1.0], size=(100_000, NUM_CLAIMS))


def permutation_p_value(typesafe_values: list, other_values: list) -> float:
    """One-sided paired sign-flip permutation test: the share of random swaps at least as
    favourable to Jev as the measured gap."""
    gaps = np.array(other_values) - np.array(typesafe_values)
    null = (signs * gaps).mean(axis=1)
    return float(((null >= gaps.mean()).sum() + 1) / (len(null) + 1))


def p_text(p: float) -> str:
    return "p < 0.001" if p < 0.001 else f"p = {p:.3f}"


print(f"{'condition':<{name_w}}{'least stable question':>23}{'average':>12}")
for label in LABELS:
    # 비교 횟수를 곱해서(Bonferroni) 여러 번 검정한 탓에 우연이 쌓이는 일을 막습니다
    worst_p = min(1.0, permutation_p_value(least_stable[TYPESAFE_LABEL], least_stable[label]) * len(LABELS))
    average_p = min(1.0, permutation_p_value(average_change[TYPESAFE_LABEL], average_change[label]) * len(LABELS))
    print(f"{label:<{name_w}}{p_text(worst_p):>23}{p_text(average_p):>12}")
```

```
condition                              least stable question     average
Claude Fable 5.1 (reasoning)                       p < 0.001   p < 0.001
Claude Opus 5.5 (reasoning)                        p < 0.001   p < 0.001
Claude Sonnet 5.5 (reasoning)                      p < 0.001   p < 0.001
Claude Haiku 5.5 (reasoning)                       p < 0.001   p < 0.001
GPT-6 Astra (reasoning)                            p < 0.001   p = 0.018
GPT-6.1 Sol (reasoning)                            p < 0.001   p = 1.000
GPT-6.1 Terra (reasoning)                          p = 0.008   p = 1.000
GPT-6 Luna (reasoning)                             p < 0.001   p < 0.001
GPT-6.1 Terra (temperature 0)                      p < 0.001   p = 0.152
GPT-6.1 Terra (default temperature)                p < 0.001   p < 0.001
GPT-6 Luna (temperature 0)                         p < 0.001   p < 0.001
GPT-6 Luna (default temperature)                   p < 0.001   p < 0.001
```

가장 불안정한 질문 기준으로는, 비교 12번에 맞춰 12를 곱한 뒤에도 모든 p-value(p값)가 0.01 미만이고, reasoning을 켠 GPT-6.1 Terra(p = 0.008)를 제외한 모든 조건에서 0.001 미만입니다. 질문 전체 평균 기준으로는, reasoning을 켠 GPT-6.1 Sol과 GPT-6.1 Terra, 그리고 temperature `0`의 GPT-6.1 Terra에 대한 Jev의 우위가 우연일 수 있습니다. temperature `0`은 LLM의 수치를 낮추지만, temperature `0`의 GPT-6.1 Terra에서도 청구 건 40개 중 20개에는 10번에 1번보다 자주 바뀌는 결정이 여전히 있습니다.

그래프의 각 행은 조건 하나이고 각 점은 청구 건 40개 중 하나입니다. 점의 위치는 다시 물었을 때 그 청구 건의 가장 불안정한 결정이 바뀔 확률이므로, 왼쪽에 있을수록 일관성이 높습니다. 한 행 안에서 점들을 위아래로 흩어 놓은 것은 겹치지 않게 하기 위해서일 뿐이고, 높이에는 아무 의미가 없습니다. 점선은 10번에 1번을 나타냅니다.

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
        # 값이 비슷한 점도 셀 수 있도록 청구 건을 세로로 흩어 놓습니다
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
    strip_ax.invert_yaxis()  # 첫 행을 맨 위에 둡니다
    strip_ax.set_xlim(0, 0.6)
    strip_ax.xaxis.set_major_formatter(PercentFormatter(1.0))
    strip_ax.set_xlabel("chance that asking again changes the decision", fontsize=9, labelpad=16)
    for x, text, align in ((0, "← more consistent", "left"), (1, "less consistent →", "right")):
        # 축 아래로 고정된 오프셋을 주어, 그래프 높이와 상관없이 라벨이 x축 라벨 위에 놓이게 합니다
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
        f"One dot per claim ({NUM_CLAIMS} per row), at its least stable of {len(QUESTIONS)} questions",
        loc="left",
        fontsize=10,
        color=p["ink"],
    )
    fig_strip.tight_layout()
    return fig_strip


# 문서 빌드에서는 COOKSAFE_RENDER를 설정하고 어두운 테마용 사본도 함께 그립니다.
for theme in ("light", "dark") if os.environ.get("COOKSAFE_RENDER") else ("light",):
    fig_strip = draw_strip(STRIP_PALETTES[theme])
    display(fig_strip, metadata={"theme": theme})
    plt.close(fig_strip)
```

![출력](https://mintcdn.com/ts-docs/orb6EiXzZ12Fo-od/cookbooks/consistency_noul_cookbook/consistency_noul_cookbook.executed.2.png?fit=max\&auto=format\&n=orb6EiXzZ12Fo-od\&q=85\&s=7d4d5a588b02dd539f247ea1e133a66e)

## TypeSafe 플레이그라운드에서 열어 보기

아래 링크를 누르면 예제 청구 건과 rubric이 플레이그라운드에서 열립니다. 청구 건 하나, 같은 `Noul` 질문 14개, 그리고 `jev-latest`입니다. 위에서 쓴, 매번 바뀌는 `uid` 필드는 빠져 있습니다.

```python
playground_link = make_playground_link(
    {"claim": CLAIM},
    {key: Noul(instructions=question) for key, question in QUESTIONS.items()},
)
display(
    Markdown(
        f"🔗 [Open this claim + rubric in the TypeSafe playground]({playground_link})"
    )
)
```

[이 청구 건과 rubric을 TypeSafe 플레이그라운드에서 열기 →](https://console.typesafe.ai/playground#share/N4IgJg9gxgrgtgUwHYBcAqCAeKQC4AEIwAOiFADYCGAlnKQSSAA4TnVQCe9+jLbnAfWphupAIIAFALQB2GQBYAjAGZSAGnyk+7DgAtWYBACdRIAFKUkCfABEICdZpAIAZi4RQU1AG4O8TgCYABgCANikgxSlFAFZHUiwmaiMEAGdTYICZCKjY+LIIXyNKAHM07kYoVjZU6ggkbhQjGAQNUhTUSnIBFNoAIxgjVIREVG4XLuGAXzbwBDAYT2o+8j8CGKCggDog2aZjISQoYWQUATY4ahRuRU3NndmalHmBMCMfY3T-AG1SCytbPZ8mYILokIC-ABdWZYCgwWr1L4EX4gJqUKAAawA9FU4PsvF5fPg3j5qEgSvkSUVUvgkBAUPgnvN8PV8ChdNZtJxSNCnCkWEYvOSBAB3MmQEWvSgcJH4W57VjsBA9BACs4pACOMGSL0Kxm4wXuQRmTgoNDo-kqVFoQhE-lIAGEADIAWSkMQ2AQAnPkycdDKgpc8MiFwkFwgEABz5fkQQUvMCUYP2kCZMPZILySnvIqmEFgiGUtJQd5MLz1UxiFDPJCGMD4Shs4qYqSJjj4BC+VAAbnwvl07FW+BFlBpKUoRikyDr+D67cs9I5RnwUAn+DJbI5+FS+08Sbj+CYE4xZJKjPpw4HQ9SKCTdSQE44W3wADkL6z2dZjkZYFctvlKDgCAYEDM1aHmbhlACDYHicNgrCEZ44Flb5GCuEZTHHZcBjxYx8H5KgoBGU58iqG8bhkI0TTQpDTCPMkGQAanw1wyWoVJdFIiByP8SMqI0GiMJTI8jBPckpGGJBUgPFJVzYPpinLBoQFmMjrn8eQYONATSHQi0CHaU4uhXNcAApQmJaVUgASi4niCGUKjeVISBYFGW8lO4FF+RoZc0i8OAk2sUyJBsAAxWyVKcSNLMCspD30FBuJ5E1SEoMAACt4WeIwBDpZ4UMYSgYHZONTGKpKpCaahSj8WZ8rWJwHWqdj7xXPVini9FCQQZ8xCYJgjD1MBnwkaV8BcGByHIBsgJAhkABJlDUaCgjZCBD0VTh9HIQwjA0GJojWgZaisVIaTbVJ-xAKZnLIa04AEAcbzjLhLVIMDkPOUczkUAIgINPZ3jjV5kGoCD-GNKYpii5ghrxFBUgwbB-CID6OohhhSBQDh9lMOkpt9KSmkWJTZVIABJGlP3Pc72qKZkQL2zdOS2jgAHIaSqabWtZKoilqgB+UhUucTA4QRZSsdRXHGtIAnyCJm9miWRFTDsNIG02-h21hch4TaygBvIdsks3djadSYWbtmeoBG-X91OlnG8ZTBWlZJ1WpPV4QWfannJfwXQjf2cERSvawaf7QdrBHGk+gQU9iRzZAWXBGnmwQNFMXXRGEHIFxrdFutSeWVYKmx2X8eAxWot04mVbJ0wAHUa-rGmFo2NaS6WFZrAT9qjFklATYbY3wfrBOXDjaxLHbI8OGAlAi9mVzUgEVIYDcdhwbGd6ZddgyQHduuQDJZXS7VlMqb9pNbygDl61c+AjKU7ct5cHfTnWhtMpgMB2BBT9p9BsqQpDsRXk4MU7IyTnFoFcCuB85bHxrh7Ru95yYgBvjTQCwFUArgesyaBA505bn2JOP0Jx8H82MLVRk8Dl4i1mMQ2BYpawQBFIgl2yCT6zHPp7JuKYbC+xpuQbiNJoCwGXCw0hrMdacwbEsIkNDOqcmMHUMAkDDJqheAFfOb1nZVzdqg0+-D0FXyPs3UcfsxF01jPGesMi-Zcg5mOBAWodSOPFBwrRIAOi3m6PnagJQy6NUYNw6uhNTEN0vt7a+1Mtz+K6FIVcy41IdjYCEvuE0ZKJzgAMIYxF8FMzwjAmkLjfEuGKP-AQLgqAUn3hE4xUS+ExK9pgsQKQWZdPFIApKQxNxJmHBOYo+DGxVOKvWFI3hwYil8boeAlgVQzIQJwxpRij68KcGY2JmCrHlOlO5Meg1hqznnCVCAgVnjt3eHQ4hS8tYLMCuCdKWUbzGAUdM2ZvjnkwC6Ms2ZXCNlOC2fXC+7TTAAGV9BTXbs9Ah5pZzWCGiVZk09ly-K6FiTe5CZnSWXF81ZSL0WcmlEvXx8FlR6XXpvfSPBK6HxBSY1p4LBFHzsMAwh9ZKXgKQu1G8NJ0r1hgEwH+NMkoBNmnghkn15i+M3gpCAJQ7wVnWYy+WzLtltLZU4bBS5Z6bRrF4Loo8kxSAmFNBkMCjD1hEjjP259Bh4SqLCw8gxN7WHRe-RVyq36yQ6hwIuMMNAgCNtQAAap8e8qNvCKBukAA)


이 문서는 개발자 문서 플랫폼인 [Mintlify](https://mintlify.com)로 제작하고 호스팅합니다.
