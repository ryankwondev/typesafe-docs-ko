# SDE cascade

> 저렴한 모델로 구조화된 데이터를 추출하고, 필드마다 Jev로 검사한 뒤, 검사가 발동할 때만 reasoning(추론) 모델로 escalate(상위 모델로 넘기기)합니다.

* 개요
  * 큰 reasoning 모델은 텍스트를 정확한 구조화된 데이터로 바꿔 주지만 느리고 비쌉니다
  * 작은 모델은 저렴하지만 실수를 합니다
  * *cascade*(캐스케이드: 단계적 처리)는 작은 모델을 먼저 실행하고, 검사에서 작은 모델의 답에
    문제가 있다고 표시될 때만 큰 모델 비용을 냅니다
  * 사용하는 모델은 다음과 같습니다:
    * rung(단계) 0 (저렴한 모델):
      [`gpt-6-luna`](https://developers.openai.com/api/docs/models/gpt-6-luna)
    * rung 1 (reasoning 모델):
      [`gpt-6.1-sol`](https://developers.openai.com/api/docs/models/gpt-6.1-sol),
      `reasoning_effort="high"` 설정
    * verifier(검증기): `jev-1.14.0`
* 알고리즘
  1. 저렴한 모델로 **답합니다**. 답은 schema(스키마)에 맞는 JSON 객체입니다.
  2. **Jev**로 **검증합니다**. 답이 지켜야 할 규칙마다 하나씩, 답에 대해 예/아니요 질문("Noul 질문")을
     던집니다. 예를 들면 "이 시간이 누군가의 회의와 겹치나요?" 같은 질문입니다. 질문마다
     P(규칙이 깨졌음)가 반환됩니다.
  3. 답이 schema를 통과하지 못하거나 규칙 검사가 발동하면 reasoning 모델로 **escalate합니다**.
     그렇지 않으면 저렴한 모델의 답을 그대로 씁니다.

## 준비

* 의존성을 설치합니다:

```bash
pip install openai jsonschema ipython matplotlib pandas 'cooksafe>=0.3.0,<0.4.0'
```

* 그런 다음 환경 변수에 `OPENAI_API_KEY`와 `TYPESAFE_API_KEY`를 설정합니다

```python
import json
import os
import re
import urllib.request
from pathlib import Path

import jsonschema
import matplotlib.pyplot as plt
import numpy as np
import pandas as pd
from cooksafe import JsonCache, make_playground_link
from IPython.display import Markdown, display
from openai import OpenAI
from typesafe_sdk import Noul, NoulCriteria, TypeSafeClient

CHEAP = "gpt-6-luna"  # rung 0: 저렴하고 빠름
STRONG = "gpt-6.1-sol-high"  # rung 1: reasoning_effort="high"로 설정한 gpt-6.1-sol
CONFIGS = {
    CHEAP: {"model": "gpt-6-luna"},
    STRONG: {"model": "gpt-6.1-sol", "reasoning_effort": "high"},
}
TYPESAFE_MODEL = "jev-1.14.0"  # 이 페이지에서 쓴 모델. "jev-latest"는 최신 모델을 실시간으로 실행
FIRE_T = 0.7  # 어느 규칙이든 P(broken)이 이 값을 넘으면 escalate. "<== FIRES" 표시 기준이기도 함

oai = OpenAI(api_key=os.environ.get("OPENAI_API_KEY", "cache-only"))

ts = TypeSafeClient(
    api_key=os.environ.get("TYPESAFE_API_KEY", "cache-only"), timeout=30.0
)
```

## 1단계: 데이터

* `example.json`에는 [NATURAL PLAN](https://github.com/google-deepmind/natural-plan)의
  캘린더 세트에서 가져온 과제 하나가 들어 있습니다. 어느 근무일에 세 사람이 각각 바쁜 시간과,
  그 사이에 끼워 넣어야 할 한 시간짜리 회의입니다
* 아래 코드는 이 파일이 디스크에 없으면 typesafe-public-examples에서 내려받습니다

```python
EXAMPLE = Path("example.json")
EXAMPLE_URL = (
    "https://raw.githubusercontent.com/typesafe-ai/typesafe-public-examples/main/"
    "cookbooks/sde_cascade/example.json"
)
if not EXAMPLE.exists():
    urllib.request.urlretrieve(EXAMPLE_URL, EXAMPLE)
task = json.loads(EXAMPLE.read_text())["prompt_0shot"]

print(task.split("TASK:")[-1].replace("SOLUTION:", "").strip())
```

```
You need to schedule a meeting for Joan, Theresa and Shirley for one hour between the work hours of 9:00 to 17:00 on Monday. 

Here are the existing schedules for everyone during the day: 
Joan has meetings on Monday during 11:00 to 11:30, 12:30 to 13:00; 
Theresa has meetings on Monday during 12:00 to 12:30, 15:00 to 15:30; 
Shirley has meetings on Monday during 9:30 to 10:30, 11:00 to 12:00, 13:00 to 14:00, 15:30 to 16:30; 

Find a time that works for everyone's schedule and constraints.
```

* 답을 내는 데 필요한 사실은 모두 텍스트에 있지만 답 자체는 없습니다. 모델이 바쁜 시간들을 보고
  비어 있는 한 시간을 직접 찾아내야 합니다
* Shirley에게 회의가 네 개 있어서, 세 사람 모두 비어 있는 온전한 한 시간은 하루 중 딱 한 번뿐입니다

## 2단계: 저렴한 모델로 답하기

* prompt(프롬프트)는 답의 JSON Schema로 끝납니다. 요일, 시작 시각, 종료 시각으로 이루어진 schema입니다
* 모델은 prompt에 있는 schema를 읽고 일반 텍스트로 응답하며, 이 텍스트를 우리가 직접 파싱합니다.
  structured outputs(구조화된 출력)는 쓰지 않습니다. 이 기능은 유효한 JSON을 강제하기 때문에,
  드물게 schema를 어기는 답이 나와도 가려져 버립니다

```python
TIME = {"type": "string", "pattern": "^([01][0-9]|2[0-3]):[0-5][0-9]$"}
SCHEMA = {
    "type": "object",
    "properties": {
        "day": {"type": "string", "enum": ["Monday", "Tuesday", "Wednesday", "Thursday", "Friday"], "description": "the day of the meeting"},
        "start_time": TIME | {"description": "when the meeting starts, 24-hour HH:MM"},
        "end_time": TIME | {"description": "when the meeting ends, 24-hour HH:MM"},
    },
    "required": ["day", "start_time", "end_time"],
    "additionalProperties": False,
}
FORMAT_LINE = "Return ONLY a JSON object matching this JSON Schema:\n" + json.dumps(SCHEMA, indent=2)


def prompt_for(task: str) -> str:
    """The task's prompt with the schema placed just before its closing SOLUTION:."""
    return re.sub(r"SOLUTION:\s*$", FORMAT_LINE + "\nSOLUTION: ", task.rstrip())


# LLM 호출과 TypeSafe 호출은 CACHE에 캐시되므로, 다시 실행하면 API를 또 호출하지 않고
# 이전 답을 재사용합니다.

EXAMPLES_URL = (
    "https://raw.githubusercontent.com/typesafe-ai/typesafe-public-examples/main/"
    "cookbooks/sde_cascade"
)


def download(name: str) -> Path:
    """Fetch one of this cookbook's files from typesafe-public-examples unless it is on disk."""
    path = Path(name)
    if not path.exists():
        urllib.request.urlretrieve(f"{EXAMPLES_URL}/{name}", path)
    return path


CACHE = Path("sde_cascade_cache.json")
# True이면 공개된 답으로 CACHE를 채우므로 실행 중에 요청을 보내지 않습니다. 실시간으로 실행하려면
# False로 바꾸고 CACHE를 삭제하세요. 디스크에 이미 있는 CACHE는 절대 덮어쓰지 않습니다.
DOWNLOAD_CACHE = True
if DOWNLOAD_CACHE:
    download(CACHE.name)
json_cache = JsonCache(CACHE)


@json_cache
def ask(prompt: str, **config) -> dict:
    """One reply from a rung, keyed on the prompt and the whole config: model and effort."""
    r = oai.chat.completions.create(messages=[{"role": "user", "content": prompt}], **config)
    return {"text": r.choices[0].message.content, "in": r.usage.prompt_tokens, "out": r.usage.completion_tokens}


def parse(text: str) -> dict | None:
    """The answer as a record, or None if it isn't a JSON object matching ``SCHEMA``. The prompt
    asks for ONLY a JSON object, so parse the reply as-is -- no regex fishing a substring out of
    a malformed reply. A reply that fails either check escalates: the safe direction."""
    try:
        record = json.loads(text)
    except ValueError:
        return None
    return record if jsonschema.Draft202012Validator(SCHEMA).is_valid(record) else None


cheap_text = ask(prompt_for(task), **CONFIGS[CHEAP])["text"]
cheap_record = parse(cheap_text)
print("cheap answer:", cheap_text)
print("\nschema-valid:", cheap_record is not None)
```

```
cheap answer: {"day":"Monday","start_time":"10:30","end_time":"11:30"}

schema-valid: True
```

* 이 답은 **schema에는 맞지만**(위 줄에 `True`가 출력됩니다) 틀렸습니다:
  * 월요일 10:30-11:30은 Joan의 11:00 회의, Shirley의 11:00 회의와 겹칩니다
  * JSON Schema 검사로는 이 문제를 알아챌 수 없습니다. 저렴한 모델은 이처럼 schema는 만족하면서
    자신 있게 틀리는 실수를 저지르며, 이런 실수를 잡아내는 것이 의미를 따지는 verifier의
    역할입니다

## 3단계: Jev로 검증하기

* verifier는 **Jev**입니다. 답이 지켜야 할 규칙마다 `Noul` 질문을 하나씩 던지며, 질문은 모두
  `true` = 규칙이 깨졌음(escalate)이 되도록 구성합니다
* Jev는 system\_one 호출 한 번으로 질문마다 calibrated(보정된) `noul` = `P(true)`를 반환합니다
* state(상태)에는 과제와 저렴한 모델이 낸 레코드가 들어갑니다. Jev는 정답을 한 번도 보지 않습니다
* **The TypeSafe Way: Decomposition(분해)**
  * 검사를 코드에서 규칙당 하나씩의 좁은 질문으로 나눕니다. 답 전체를 두고 질문 하나로 묻는 일은
    없습니다
  * 그러면 flag(플래그: 경고 표시)가 어떤 규칙이 깨졌는지 짚어 주고, 질문마다 따로 문구를 고치거나
    threshold(임계값)를 다시 정할 수 있습니다
  * "이 답이 맞나요?"라고 한 번만 물으면 점수 하나만 돌아오고, 그 점수로는 무엇이 잘못됐는지 알 수
    없습니다
  * 다음 문서를 참고하세요:
    [질문 분해하기](../concepts/how-to-build-with-system-one.md#decompose-the-questions)
  * ![이것이 길입니다](https://mintcdn.com/ts-docs/2NirYCl-v96cw05F/cookbooks/sde_cascade/this_is_the_way.jpg?fit=max\&auto=format\&n=2NirYCl-v96cw05F\&q=85\&s=10bd7d99dc5f679022bb6763dde57330)

```python
RULES = {
    "busy_conflict": "Does the meeting time in the `answer` overlap a time when any participant in the `task` is busy?",
    "outside_hours": "Is the meeting time in the `answer` outside the work hours or the days the `task` allows?",
    "breaks_preference": "Does the meeting time in the `answer` break a preference stated in the `task`, such as a participant who would rather not meet on a certain day or before or after a certain time?",
    "not_earliest": "Does the `task` ask for the earliest availability, and is there an earlier time that works for every participant?",
    "wrong_length": "Is the meeting in the `answer` a different length from the one the `task` asks for?",
    "no_single_time": "Does the `answer` fail to propose exactly one day and one start and end time?",
}
CHECKS = {
    rule: Noul(
        instructions=question,
        criteria=NoulCriteria(true="the answer breaks this rule", false="the answer follows this rule"),
    )
    for rule, question in RULES.items()
}


@json_cache
def verify(state_json: str, model: str) -> dict[str, float]:
    """Ask every rule question about one answer in one TypeSafe call; return ``{rule: P(broken)}``."""
    questions = {k: q.model_dump(mode="json", exclude_none=True) for k, q in CHECKS.items()}
    answers = ts.system_one(state=json.loads(state_json), questions=questions, model=model).answers
    return {rule: ans.noul for rule, ans in answers.items()}
```

```python
state = {"task": task, "answer": cheap_record}
checks = verify(json.dumps(state), model=TYPESAFE_MODEL)
playground_link = make_playground_link(state, {k: q.model_dump(mode="json", exclude_none=True) for k, q in CHECKS.items()})
display(Markdown(f"🔗 [Open this verification in the TypeSafe playground]({playground_link})"))

print(f"{'rule':<20}{'P(broken)':>10}")
print("-" * 30)
for rule, p in sorted(checks.items(), key=lambda c: -c[1]):
    flag = "  <== FIRES" if p > FIRE_T else ""
    print(f"{rule:<20}{p:>10.2f}{flag}")
```

```
rule                 P(broken)
------------------------------
busy_conflict             0.97  <== FIRES
not_earliest              0.25
wrong_length              0.17
breaks_preference         0.09
no_single_time            0.07
outside_hours             0.04
```

[TypeSafe playground에서 이 검증 열기 →](https://console.typesafe.ai/playground#share/N4IgJg9gxgrgtgUwHYBcAqCAeKQC4AEIwAOiCgIYDOA1qQaQJoQz7kBOCrS+WADgmxSshlKAAsEYGABsAlkgDm+RAhTyFlAHT4mLdpwWyAbslb4AZggDu+KBCSUUbcvJSV89-Cgk9Msx+r4ohJS0pwQ5jzk4vi87GpQsnGoADRePipqivhSzmr2aeRIYLEQlJSyAEbSAJ5BEIixHJYcSFAI7p7enJmBaojaAHIQKJzdHL7+bmaUENIw+dze5EJWEGzU7lay3pMB2cGSMuGRCCZsdXGCsonJKNoAEgKc+maWNljkcLxhXlSbXBKs3mi0ouGIxCQEKQaAAggBlADSBF0+CQCEkXggQXER1+5GUGKySnM63wACkIEU0mgJBxKASiiV4WJZGwwnVSWwPOj8GJmNzKqorBilj41hs+QLOpEAJy4AAMCqx+AAjAB2RXKzwAWXsYHINW00OhTwmr26e2JOJCx3cXJ45xq9k4uT6PgNNXokMpRT5VEJqnUnW4euKhpyMDYgVVqq1KtjuAAzAq0qqAEzJ5UobGqpNagDc+GhtOeDP97l6ihD+DDnsj0eyGfjObVmZTaYArC3c92U0XoSy2RyK4HiTW6xG3dl5SmEwqs2m40qE5mlWn8yvW6qACxartZhMANizA8h0IAYvISgT+mMxCt8BKAQ6zgJneiAOTuQ6hF7FWx7EcZxXC0YtIXhAB5AAZABVNAAElIMGegQDSUgikoEU2DofASHAQ1cNIScalIdCQEceIAH07yIkBVQXFMyOLEBkDAGjZEQOjEyYkAAF8+LQwheDYBpeDcDBsDwQh8MqGBKBqKi7CQcw5CgFBcPwlAan4OikGYaRmNIeRgJgdTZCAuiABEIA6dIeiJPpOM4eR7PwAADTDsPcjxzmkcheDMO8nwkbgikueIbiSIohFcy13IoGgfP8fA5IUgB+IyQCgaNRmjchNNIJwYAQOjLS8gRUo4cgAW8FK2GOLLzHIaRKFK6Sip8CruVJaRpAgKx3Dq9wGrCUgBPI5g3FkMAECo-ko0oQqyB09rUP0mQspM4rzMsjqQAQoaMkc7JgrinxPIcbyPAWCpZrc58pUWjxuUtT0js4BL-h8lr+sGzKhNIHKdgEWQCukrSGrWljyquyrKmq2rWRGxrAZAZrWuhzr-ywyrer+o76tRiaWIRhAasoKiRIQFpkHaZbtN0-aNsMtHtoa3aHGs2yPrHJzGnOz7up8smarManabaThKNGEpBY8xLqHctJKDMsRWHcAkrgSaLUBC7E1hkEo8jpNERjHHkzHaQQXG4esySFLlwm5chzDyq2BAoOLnIB8jgbysGGahsqurhwVEcJlGxrRjG2pDnHsIsOYCfSIno5J0h9JQKjyfZWQOg0iGitWvSDK2hwdtBbm7PixWfpoJPXp8XO5AL1gjBcfzKlkORtMKACUvGf8ojzyrguWVZ1hfMk3wuWJItuGLfZY-3QfBghIZK+OuFx8PyaRtP2vI2OsbIUPd6TvqBsj-BRvajOQCsUTFCosJFG8BmS+Zsu2Yrjmq-2odNyVYlDy0urvH6ORZDmElkIN+ChdjmFEnANyLo3JfSShraebBl5A1ymvIOW99qwwvqLA+Ucj4sRPtvbql8U7DVvsTPi5F9JUQqIoMIHEuJFxWkzdaP9yLszMgA1CNka4XWFhYTuKoRIQF4GUTgnx1K1B5K6CMTJVFBAoIIQEPAAJ3lwdlfB+VCGnxIYnMhN875NRanHYh59E742vqnCh40BJCRAAFWQAA1AQFR7DSRAEYVU-EgA)

* 깨진 규칙만 점수가 높게 나옵니다:
  * `busy_conflict`는 0.97로 발동합니다
  * 나머지 규칙은 모두 0.25 이하에 머뭅니다

## 4단계: escalate할지 결정하기

* 답이 schema 검사를 통과하지 못했거나, 규칙 중 *어느 하나라도* 확률이 `FIRE_T`를 넘으면
  escalate합니다
* 평균이 아니라 `max` 방식의 검사이므로, 확신에 찬 경고가 하나만 있어도 충분합니다. 평균에 묻혀
  조용히 사라지지 않습니다
* threshold 하나로 충분한 것은 질문들이 잘 갈릴 때뿐입니다. 3단계에서처럼 깨진 규칙에는 높은 점수가,
  지켜진 규칙에는 낮은 점수가 나와야 합니다
* verifier 비용은 reasoning rung보다 훨씬 낮아야 합니다. 그렇지 않으면 cascade로 아끼는 것이
  없습니다

```python
fired = {rule: p for rule, p in checks.items() if p > FIRE_T}
escalate = cheap_record is None or bool(fired)

print(f"check (threshold {FIRE_T}): {'ESCALATE' if escalate else 'ACCEPT cheap answer'}")
for rule, p in sorted(fired.items(), key=lambda c: -c[1]):
    print(f"  fired: {rule}  (P={p:.2f})")
```

```
check (threshold 0.7): ESCALATE
  fired: busy_conflict  (P=0.97)
```

## 5단계: reasoning 모델로 escalate하기

검사가 발동했으므로 강한 모델(`gpt-6.1-sol`,
`reasoning_effort="high"`) 비용을 냅니다

```python
final_record = parse(ask(prompt_for(task), **CONFIGS[STRONG])["text"]) if escalate else cheap_record

print("cheap     :", json.dumps(cheap_record))
print("reasoning :", json.dumps(final_record))
```

```
cheap     : {"day": "Monday", "start_time": "10:30", "end_time": "11:30"}
reasoning : {"day": "Monday", "start_time": "14:00", "end_time": "15:00"}
```

* **개선된 점**
  * reasoning 모델은 월요일 14:00-15:00을 제안하며, 이 시간에는 세 사람 모두 비어 있습니다
  * cascade가 이 과제에서 reasoning 모델 비용을 낸 이유는 verifier가 문제를 표시했기 때문일
    뿐입니다

## 6단계: 과제 1,000개에 cascade 적용하기

* 예시 하나로는 이 루프가 어떻게 도는지만 알 수 있습니다. 실제로 이득이 되는지 보려면 많은 과제에
  돌려 보고 비용 대비 품질을 그래프로 그려야 합니다
* cascade는 저렴한 모델이 대부분 맞히고, 틀릴 때는 검사로 알아챌 수 있는 규칙을 어기는 경우에
  이득이 됩니다. 회의 일정 잡기가 여기에 들어맞습니다:
  * **과제:** [NATURAL PLAN](https://github.com/google-deepmind/natural-plan)의 캘린더 일정
    잡기 과제 1,000개입니다. 과제마다 근무일 1~5일에 걸친 2~7명의 바쁜 시간, 회의 길이,
    그리고 때로는 "월요일에는 회의하고 싶지 않음" 같은 선호 사항이 주어집니다. 모델은 모두에게
    맞는 시간 하나를 제안합니다
  * **prompt:** 각 과제의 텍스트이며, 2단계의 JSON Schema로 끝납니다
  * **모델:** 기본 reasoning 설정의 `gpt-6-luna`가 저렴한 rung이고,
    `reasoning_effort="high"`로 설정한 `gpt-6.1-sol`이 강한 rung입니다. 두 모델 모두 모든 과제에
    답하므로, 각 모델이 단독으로 어떤 결과를 내는지 알 수 있습니다
  * **검사:** 3단계의 규칙 질문 여섯 개입니다. 과제의 flag 값은 여섯 개 중 가장 높은 값이며,
    luna의 답이 schema 검사를 통과하지 못하면 1입니다
  * **cascade:** flag 값이 threshold보다 높은 과제는 sol-high로 넘어갑니다. 비용은 luna와 Jev
    비용을 더한 값이고, 넘어간 경우에는 sol-high 비용이 추가됩니다
  * **채점:** 답이 schema에 맞는 JSON이고 과제의 유효한 시간 중 하나를 제안하면 통하는 답으로
    봅니다. 유효한 시간이 둘 이상인 과제도 있어서, 그런 시간을 NATURAL PLAN의 정답표에
    추가했습니다. 전체 정답표는 `calendar_answers.json`에 있습니다. 채점은 답에 점수를 매길
    뿐이고 cascade에는 전혀 관여하지 않습니다
* [`calendar_sweep.py`](https://github.com/typesafe-ai/typesafe-public-examples/blob/main/cookbooks/sde_cascade/calendar_sweep.py)가
  이 호출들을 실행하고 과제당 한 행씩 `calendar_results.csv`에 기록했습니다. 아래 코드는
  그 파일로 차트를 그리기만 하고 아무것도 호출하지 않습니다
* 아래 수치는 2026-11-05에 `jev-1.14.0`으로 얻은 것입니다

```python
RESULTS = Path("calendar_results.csv")
RESULTS_URL = (
    "https://raw.githubusercontent.com/typesafe-ai/typesafe-public-examples/main/"
    "cookbooks/sde_cascade/calendar_results.csv"
)
if not RESULTS.exists():
    urllib.request.urlretrieve(RESULTS_URL, RESULTS)
tasks = pd.read_csv(RESULTS)  # 과제당 한 행: 각 모델의 비용, 답이 통하는지 여부, Jev의 flag


def cascade_at(t: float) -> dict:
    """Mean cost per 1,000 tasks and share of answers that work when tasks flagged above t escalate."""
    escalate = tasks["flag"] > t
    cost = tasks["cheap_cost"] + tasks["check_cost"] + escalate * tasks["strong_cost"]
    works = tasks["strong_works"].where(escalate, tasks["cheap_works"])
    return {"threshold": t, "cost": 1000 * cost.mean(), "works": works.mean(), "escalated": escalate.mean()}


cascade = pd.DataFrame([cascade_at(t) for t in np.linspace(0, 1, 201)])
frontier = cascade.sort_values(["cost", "works"], ascending=[True, False])
frontier = frontier[frontier["works"] > frontier["works"].cummax().shift(fill_value=-1)]
singles = pd.DataFrame(
    {
        name: {
            "cost": 1000 * tasks[f"{rung}_cost"].mean(),
            "cost_sem": 1000 * tasks[f"{rung}_cost"].sem(),
            "works": tasks[f"{rung}_works"].mean(),
            "works_sem": tasks[f"{rung}_works"].sem(),
        }
        for name, rung in (("gpt-6-luna", "cheap"), ("gpt-6.1-sol-high", "strong"))
    }
).T

# 문서 페이지는 라이트 또는 다크 테마에 맞는 차트를 보여 주고, 노트북은 라이트 차트를 보여 줍니다.
PALETTES = {
    "light": {"surface": "#ffffff", "ink": "#34342f", "muted": "#7c7c77", "grid": "#d8d8cf", "blue": "#5d76a2", "cloud": 0.25},
    "dark": {"surface": "#0d0a0f", "ink": "#fafafa", "muted": "#a1a1aa", "grid": "#3f3f46", "blue": "#8fa6d6", "cloud": 0.35},
}


def draw_chart(p: dict) -> plt.Figure:
    fig, ax = plt.subplots(figsize=(8, 5.4), facecolor=p["surface"])
    ax.set_facecolor(p["surface"])
    for side in ("top", "right"):
        ax.spines[side].set_visible(False)
    for side in ("left", "bottom"):
        ax.spines[side].set_color(p["grid"])
    ax.tick_params(colors=p["muted"], labelcolor=p["ink"], labelsize=9)
    ax.grid(color=p["grid"], linewidth=0.6)
    ax.set_axisbelow(True)
    ax.scatter(cascade["cost"], cascade["works"], s=10, color=p["blue"], alpha=p["cloud"], lw=0,
               label="cascade, every threshold")
    ax.plot(frontier["cost"], frontier["works"], color=p["blue"], lw=2, marker="o", ms=4,
            label="cascade frontier (gpt-6-luna → gpt-6.1-sol-high)")
    first, last, luna = frontier.iloc[0], frontier.iloc[-1], singles.loc["gpt-6-luna"]
    if luna["cost"] < first["cost"]:  # luna 단독 지점이 frontier 왼쪽에 있으면 luna의 높이에서 이어 줌
        ax.plot([luna["cost"], first["cost"]], [luna["works"]] * 2, color=p["blue"], lw=2)
    ax.plot([last["cost"], cascade["cost"].max()], [last["works"]] * 2, color=p["blue"], lw=2)  # 이 지점부터는 돈을 더 써도 얻는 것이 없음
    ax.errorbar(singles["cost"], singles["works"], xerr=singles["cost_sem"], yerr=singles["works_sem"], fmt="D",
                ms=8, color=p["ink"], ecolor=p["muted"], elinewidth=1, capsize=3, label="single model ±1 SEM")
    for name, row in singles.iterrows():
        ax.annotate(name, (row["cost"], row["works"]), xytext=(7, -12), textcoords="offset points", fontsize=9,
                    color=p["ink"])
    ax.set_xlabel("mean $ per 1,000 tasks", color=p["ink"], fontsize=9)
    ax.set_ylabel("share of answers that work", color=p["ink"], fontsize=9)
    ax.set_title("1,000 meeting-scheduling tasks", color=p["ink"], fontsize=10)
    ax.legend(fontsize=8, frameon=False, loc="lower right", labelcolor=p["ink"])
    plt.tight_layout()
    return fig


# 문서 빌드는 COOKSAFE_RENDER를 설정하고 다크 버전도 함께 그립니다.
for theme in ("light", "dark") if os.environ.get("COOKSAFE_RENDER") else ("light",):
    fig = draw_chart(PALETTES[theme])
    display(fig, metadata={"theme": theme})
    plt.close(fig)
```

![출력](https://mintcdn.com/ts-docs/Jekhcl_2QPf6976X/cookbooks/sde_cascade/cascade_cookbook.executed.1.png?fit=max\&auto=format\&n=Jekhcl_2QPf6976X\&q=85\&s=cda7e0b86544a4d52fe1079ab3ba524b)

```python
print(singles[["cost", "works"]].round(3).to_string())
wrong = tasks["cheap_works"] == 0
print(f"\nluna is wrong on {wrong.sum()} tasks; {tasks.loc[wrong, 'schema_valid'].sum()} of those answers pass the schema check")
print()
print(pd.DataFrame([cascade_at(t) for t in (0.58, 0.29, 0.18, 0.16)]).round(3).to_string(index=False))
```

```
                   cost  works
gpt-6-luna        0.156  0.926
gpt-6.1-sol-high  1.890  0.981

luna is wrong on 74 tasks; 72 of those answers pass the schema check

 threshold  cost  works  escalated
      0.58 0.261  0.949      0.027
      0.29 0.490  0.965      0.133
      0.18 1.307  0.981      0.536
      0.16 1.500  0.983      0.641
```

* 차트 읽는 법:
  * **검은 마름모:** 각 모델을 단독으로 썼을 때의 결과이며 ±1 SEM 막대가 붙어 있습니다. 비용은
    과제마다 거의 차이가 없어서 비용 쪽 막대는 마커보다 좁습니다
  * **파란 점:** 모든 threshold에서의 cascade입니다. 선은 비용과 품질 양쪽 모두에서 다른 어떤
    점에도 뒤지지 않는 점들을 이은 것입니다
* 차트에서 알 수 있는 것:
  * luna 단독으로는 과제 1,000개당 \$0.16에 답의 92.6%를 맞히고, sol-high 단독으로는
    \$1.89에 98.1%를 맞힙니다
  * luna가 틀린 답 74개 중 72개가 schema 검사를 통과하므로, 이 답들은 Jev의 규칙 검사로만
    잡아낼 수 있습니다
  * Jev가 가장 강하게 flag를 세운 과제 3%를 sol-high로 보내면 cascade의 정답률이 \$0.26에
    94.9%까지 올라갑니다
  * 13%를 보내면 \$0.49에 96.5%에 이르며, 이는 sol-high 비용의 4분의 1 정도입니다
  * 54%를 보내면 \$1.31에 sol-high와 같은 98.1%가 됩니다. 64%(\$1.50에 98.3%)를 넘어서면 돈을
    더 써도 얻는 것이 없으며, 오른쪽의 평평한 선이 이를 보여 줍니다
  * Jev의 검사 비용은 과제 1,000개당 \$0.04입니다


이 문서는 개발자 문서 플랫폼인 [Mintlify](https://mintlify.com)로 제작하고 호스팅합니다.
