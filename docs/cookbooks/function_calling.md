# Function calling

> 함수 이름과 closed set(닫힌 집합) 인자를 confidence(신뢰도)가 함께 나오는 System One 질문에 대응시켜, 자연어로 된 트레이딩 요청을 타입이 지정된 평범한 함수 호출로 바꿉니다.

"오트 밀크 아이스 라테 라지 사이즈, 시럽은 빼고요"라고 주문하면 바리스타는 그 문장을 받아 적지 않습니다.
컵에 인쇄된 선택지 네 개에 표시할 뿐입니다. 이 cookbook(예제 모음)은 트레이딩 API를 대상으로
같은 일을 합니다. 문장 하나를 넣으면 함수 이름과 인자가 평가를 마친 enum(열거형) 값으로 나오고,
값마다 confidence가 붙습니다.

```text
"plot rolling correlation between nvda and spy for the past month"
    rolling_correlation(symbol='NVDA', benchmark='SPY', window='1mo')   confidence 0.93

"compare nvda amd and msft over the past three months"
    compare_returns(symbols=['NVDA', 'AMD', 'MSFT'], window='3mo')      confidence 0.93

"show me apple daily with volume"
    plot_price(symbol='AAPL', resolution='1d', include_volume=True)     confidence 0.69

"what tickers do you have"
    list_symbols()                                                     confidence 1.00
```

이 호출들은 트레이딩 어시스턴트에 들어 있는 평범한 함수 열 개로 전달됩니다. 인자가 받는 값은
고정된 목록에서 나오므로, 인자는 이미 `Literal`로 선언되어 있습니다.

```python title="Sketch"
def plot_price(
    symbol: Literal["SPY", "NVDA", "AMD", "AAPL", "MSFT", "TSLA"],
    style: Literal["line", "candles"] = "line",
    resolution: Literal["1m", "5m", "15m", "1h", "1d"] = "15m",
    window: Literal["1d", "1w", "1mo", "3mo"] = "1w",
    include_volume: bool = False,
    moving_average: Literal["9", "20", "50"] | None = None,
    log_scale: bool = False,
): ...
```

값이 고정된 목록에서 나오는 인자를 closed set이라고 합니다. 그 목록에서 값 하나를 받는 인자에는
정확히 그 값들만 선택지로 둔 `Choice` 질문이 하나 붙으므로, 함수에 도달하는 값은 언제나 함수가
받아들이는 값입니다. 함수는 손대지 않습니다. 추가하는 것은 각 인자의 의미를 평이한 말로 적은
spec(명세)뿐입니다. 이 문서를 끝까지 따라가면 여러분의 함수에 바로 연결할 수 있는 `Dispatcher`가
완성됩니다.

## 준비

```bash
pip install ipython polars matplotlib numpy 'cooksafe>=0.3.0,<0.4.0'
```

`TYPESAFE_API_KEY`를 설정합니다. 아래 코드는 모듈 두 개와 `spec.json`, 가격 봉 데이터를
[typesafe-public-examples](https://github.com/typesafe-ai/typesafe-public-examples/tree/main/cookbooks/function_calling)에서
내려받습니다. 이미 디스크에 있으면 내려받지 않습니다. `trader.py`에는 함수 열 개가 들어 있고,
`dispatch.py`에는 시그니처와 spec을 읽어 호출을 만들어 내는 코드가 들어 있습니다.

가격은 고정된 난수 시드로 시뮬레이션한 값이며 실제 시장 데이터가 아닙니다. 티커는 실제로 있는
것이지만, 아래에 나오는 가격과 수익률, 차트는 모두 생성한 것입니다.

```python
import json
import os
import urllib.request
from pathlib import Path

import matplotlib.pyplot as plt
from cooksafe import JsonCache, make_playground_link
from IPython.display import Markdown, display
from typesafe_sdk import SystemOneResponse, TypeSafeClient

EXAMPLES = (
    "https://raw.githubusercontent.com/typesafe-ai/typesafe-public-examples/main/"
    "cookbooks/function_calling"
)
for name in ["dispatch.py", "trader.py", "spec.json", "bars.parquet"]:
    if not Path(name).exists():
        urllib.request.urlretrieve(f"{EXAMPLES}/{name}", name)

from dispatch import ROUTE, Dispatcher, closed_sets  # noqa: E402  (위에서 내려받음)
from trader import TOOLS, chart_theme, load  # noqa: E402


def show(draw) -> None:
    """Display the chart ``draw()`` makes. The docs build sets COOKSAFE_RENDER and also gets
    a dark copy."""
    for theme in ("light", "dark") if os.environ.get("COOKSAFE_RENDER") else ("light",):
        with chart_theme(theme):
            figure = draw()
        display(figure, metadata={"theme": theme})
        plt.close(figure)


print(f"{len(TOOLS)} functions over {load().height:,} one-minute bars")
```

```
10 functions over 156,780 one-minute bars
```

```python
TYPESAFE_MODEL = "jev-1.14.0"  # 이 페이지에서 쓴 모델. "jev-latest"는 최신 모델을 실시간으로 실행
typesafe = TypeSafeClient(
    api_key=os.environ.get("TYPESAFE_API_KEY", "cache-only"),  # 키 없이 실행하면 캐시를 재생
    base_url=os.environ.get("TYPESAFE_ENDPOINT"),
    timeout=120.0,
)

CACHE = Path("function_calling_cache.json")
# True이면 공개된 답변으로 CACHE를 채우므로 실행 중에 요청을 보내지 않습니다. 실시간으로
# 실행하려면 False로 바꾸고 CACHE를 삭제하세요. 이미 디스크에 있는 CACHE는 덮어쓰지 않습니다.
DOWNLOAD_CACHE = True
if DOWNLOAD_CACHE and not CACHE.exists():
    urllib.request.urlretrieve(f"{EXAMPLES}/{CACHE.name}", CACHE)
json_cache = JsonCache(CACHE)


@json_cache
def _system_one(state: str, questions: str, model: str) -> dict:
    response = typesafe.system_one(state=state, questions=json.loads(questions), model=model)
    return {
        "answers": {k: a.model_dump(mode="json") for k, a in response.answers.items()},
        "usage": response.usage.model_dump(mode="json"),
    }


class Client:
    """The `system_one` the dispatcher calls, with every answer cached in CACHE."""

    def system_one(self, state, questions: dict, model: str) -> SystemOneResponse:
        raw = _system_one(str(state), json.dumps(questions, sort_keys=True), model=model)
        return SystemOneResponse.model_validate({**raw, "model": model})


client = Client()
```

## 시그니처에서 closed set 찾기

어떤 인자가 고정된 목록에서 값을 받는지, 그 목록에 무엇이 들어 있는지는 type hint(타입 힌트)에
이미 적혀 있습니다. `closed_sets`는 시그니처를 읽어 그런 인자를 세 가지 형태로 나눕니다.
**choice**(`Literal`이므로 목록에서 값 하나), **set**(`list[Literal[...]]`이므로 목록에서
임의 개수), **flag**(`bool`이므로 켜짐 또는 꺼짐)입니다. 함수 열 개는 모두 `trader.py`에
정의되어 있습니다.

```python
for name, fn in TOOLS.items():
    shapes = closed_sets(fn)
    print(
        f"  {name:<20}{len(shapes)}  "
        + ", ".join(f"{a}:{s}" for a, (s, _) in shapes.items())
    )
print(
    f"\n{sum(len(closed_sets(fn)) for fn in TOOLS.values())} fillable arguments in total"
)
```

```
  list_symbols        0  
  market_summary      1  window:choice
  plot_price          7  symbol:choice, style:choice, resolution:choice, window:choice, include_volume:flag, moving_average:choice, log_scale:flag
  intraday_pattern    3  symbol:choice, window:choice, metric:choice
  compare_returns     3  symbols:set, window:choice, normalize:flag
  rolling_correlation 4  symbol:choice, benchmark:choice, window:choice, resolution:choice
  summary_stats       2  symbol:choice, window:choice
  volatility          3  symbol:choice, window:choice, annualized:flag
  top_movers          2  window:choice, direction:choice
  drawdown            3  symbol:choice, window:choice, plot:flag

28 fillable arguments in total
```

무엇이 제외되는지는 `top_movers`에서 볼 수 있습니다. 인자 세 개 가운데 두 개가 closed set입니다.
세 번째인 `limit`은 `int`이므로 질문이 붙지 않고 기본값 3을 그대로 씁니다. 자유 텍스트, 숫자,
날짜도 마찬가지입니다. 질문이 없고, 함수의 기본값이 그대로 적용됩니다.

## spec 작성하기

`Literal`에서 얻을 수 있는 것은 `"1mo"`와 `"3mo"`라는 문자열입니다. 사용자가 "this quarter"라고
입력하면 두 번째를 뜻한다는 사실은 거기에 없습니다. 그 내용을 담는 것이 spec입니다. spec에는
인자마다 질문 하나, 선택지마다 설명 한 줄, 함수마다 설명 하나가 들어가고, 여기에 함수들 가운데
하나를 고르는 질문이 하나 더 들어갑니다. spec은 `spec.json`에 들어 있으며, 시그니처를 주고
LLM에게 작성하게 할 수도 있습니다.

```python
SPEC = json.loads(Path("spec.json").read_text())
for argument in ("style", "moving_average"):
    print(
        json.dumps(
            {argument: SPEC["functions"]["plot_price"]["arguments"][argument]}, indent=2
        )
    )
```

```
{
  "style": {
    "question": "Does the user want a plain line or candles?",
    "stated": "Does the user say how the chart should be drawn, such as a line, candles, or OHLC bars?",
    "options": {
      "line": "a simple line through the closing prices",
      "candles": "a candlestick or OHLC chart, showing each bar's open, high, low and close"
    }
  }
}
{
  "moving_average": {
    "question": "How many bars should the moving average cover - nine, twenty, or fifty?",
    "stated": "Does the user ask for a moving average or a smoothed line over the candles?",
    "options": {
      "9": "a nine-bar moving average, a fast one",
      "20": "a twenty-bar moving average",
      "50": "a fifty-bar moving average, a slow one"
    }
  }
}
```

선택지의 키가 곧 함수가 받는 문자열이므로, 나중에 라벨을 인자 값으로 되돌려 매핑할 필요가
없습니다. `stated`는 인자를 선택 사항으로 만듭니다. 명령이 그 인자에 대해 무엇이든 언급하는지를
묻는 두 번째 예/아니요 질문입니다. 답이 "아니요"이면 호출에서 그 인자를 빼고 함수 자체의 기본값을
적용합니다.

set 인자에는 구성원마다 질문이 하나씩 붙고, 구성원 이름이 들어갈 자리는 `{}`로 표시합니다.
`"Does the user want {} in the comparison?"`는 티커마다 질문 하나가 됩니다.

질문은 사용자가 고를 법한 단어가 아니라 개념을 기준으로 작성하세요. 매칭은 의미를 기준으로
이루어지기 때문입니다. "is amd tracking nvidia lately"는 `spec.json` 어디에도 *tracking*이나
*lately*라는 단어가 없는데도 `rolling_correlation`에 도달합니다. 질문에 매개변수 이름을 그대로
붙이는 것은 피하세요. `"Which resolution?"` 같은 질문에는 명령과 맞춰 볼 단서가 없습니다.

## spec을 질문으로 바꾸기

`Dispatcher`는 spec으로 질문을 한 번만 만들어 둡니다. 그 뒤로는 명령 하나가 요청 하나가 되고,
이 요청에 함수 선택과 모든 함수의 인자가 함께 실립니다. dispatcher는 선택된 함수의 답만 읽습니다.

```python
assistant = Dispatcher(SPEC, TOOLS, client, model=TYPESAFE_MODEL)
print(f"{len(assistant.questions)} questions per command, among them:")
for qid in (
    "__tool__",
    "plot_price.style",
    "plot_price.style?",
    "compare_returns.symbols.NVDA",
):
    question = assistant.questions[qid]
    print(f"  {qid:<30}{question['type']:<8}{str(question['instructions'])[:64]}")
```

```
54 questions per command, among them:
  __tool__                      choice  What is the user asking the trading assistant to do?
  plot_price.style              choice  Does the user want a plain line or candles?
  plot_price.style?             noul    Does the user say how the chart should be drawn, such as a line,
  compare_returns.symbols.NVDA  noul    Does the user want NVDA in the comparison?
```

## 명령 열네 개 실행하기

요청 하나가 한 줄을 차지하며, `confidence`는 그 호출을 이루는 판단 가운데 가장 확신이 낮은
판단의 값입니다.

```python
COMMANDS = [
    "show nvda 1h",
    "plot rolling correlation between nvda and spy for the past month",
    "when during the day does nvda trade the most",
    "what moved today",
    "what tickers do you have",
    "how did the market do this week",
    "candles for tesla with a 20 period moving average",
    "compare nvda amd and msft over the past three months",
    "how volatile is tsla",
    "biggest losers today",
    "worst drawdown for nvda this quarter, and chart it please",
    "spy stats for the last month",
    "show me apple daily with volume",
    "is amd tracking nvidia lately",
]

CALLS = {command: assistant(command) for command in COMMANDS}
for command, call in CALLS.items():
    print(f'  "{command}"')
    print(
        f"      {str(call):<66}confidence {call.confidence:.2f}"
        f"   tool {call.tool.probability:.2f}"
    )
```

```
  "show nvda 1h"
      plot_price(symbol='NVDA', resolution='1h')                        confidence 0.59   tool 0.99
  "plot rolling correlation between nvda and spy for the past month"
      rolling_correlation(symbol='NVDA', benchmark='SPY', window='1mo') confidence 0.93   tool 1.00
  "when during the day does nvda trade the most"
      intraday_pattern(symbol='NVDA')                                   confidence 0.53   tool 1.00
  "what moved today"
      top_movers(window='1d', direction='gainers')                      confidence 0.90   tool 0.94
  "what tickers do you have"
      list_symbols()                                                    confidence 1.00   tool 1.00
  "how did the market do this week"
      market_summary(window='1w')                                       confidence 0.94   tool 0.98
  "candles for tesla with a 20 period moving average"
      plot_price(symbol='TSLA', style='candles', moving_average='20')   confidence 0.88   tool 1.00
  "compare nvda amd and msft over the past three months"
      compare_returns(symbols=['NVDA', 'AMD', 'MSFT'], window='3mo')    confidence 0.93   tool 1.00
  "how volatile is tsla"
      volatility(symbol='TSLA')                                         confidence 0.97   tool 1.00
  "biggest losers today"
      top_movers(window='1d', direction='losers')                       confidence 0.94   tool 1.00
  "worst drawdown for nvda this quarter, and chart it please"
      drawdown(symbol='NVDA', window='3mo', plot=True)                  confidence 0.92   tool 0.98
  "spy stats for the last month"
      summary_stats(symbol='SPY', window='1mo')                         confidence 0.97   tool 0.98
  "show me apple daily with volume"
      plot_price(symbol='AAPL', resolution='1d', include_volume=True)   confidence 0.69   tool 0.98
  "is amd tracking nvidia lately"
      rolling_correlation(symbol='AMD', benchmark='NVDA')               confidence 0.69   tool 0.85
```

긴 명령 두 개는 모두 요청한 대로 나왔습니다. "plot rolling correlation between nvda and spy for
the past month"는 문장 하나로 인자 네 개 중 세 개를 채웠고 `resolution`은 기본값으로 남겼습니다.
그중 `symbol`과 `benchmark`는 같은 티커 여섯 개에서 값을 고르는데도 각 티커가 올바른 인자에
들어갔습니다. 질문이 두 인자의 역할을 풀어서 적고 있기 때문입니다. 하나는 *the one being
measured, named first*(측정 대상이며 먼저 언급된 쪽)이고, 다른 하나는 *the second one named,
the yardstick*(두 번째로 언급된 쪽, 비교 기준)입니다. "compare nvda amd and msft over the past
three months"는 티커 세 개를 set에 넣고 나머지 세 개는 뺐습니다.

그중 세 개를 실행해 봅니다.

```python
for command in (
    "plot rolling correlation between nvda and spy for the past month",
    "compare nvda amd and msft over the past three months",
    "when during the day does nvda trade the most",
):
    print(f'"{command}"  ->  {CALLS[command]}')
    show(CALLS[command].run)
```

```
"plot rolling correlation between nvda and spy for the past month"  ->  rolling_correlation(symbol='NVDA', benchmark='SPY', window='1mo')
"compare nvda amd and msft over the past three months"  ->  compare_returns(symbols=['NVDA', 'AMD', 'MSFT'], window='3mo')
"when during the day does nvda trade the most"  ->  intraday_pattern(symbol='NVDA')
```

![output](https://mintcdn.com/ts-docs/2NirYCl-v96cw05F/cookbooks/function_calling/function_calling.executed.1.png?fit=max\&auto=format\&n=2NirYCl-v96cw05F\&q=85\&s=be89e89d17fcf30851d5bc2def0efeb1)

![output](https://mintcdn.com/ts-docs/Jekhcl_2QPf6976X/cookbooks/function_calling/function_calling.executed.3.png?fit=max\&auto=format\&n=Jekhcl_2QPf6976X\&q=85\&s=a572852c52d68583e1a349933061294a)

![output](https://mintcdn.com/ts-docs/Jekhcl_2QPf6976X/cookbooks/function_calling/function_calling.executed.5.png?fit=max\&auto=format\&n=Jekhcl_2QPf6976X\&q=85\&s=9ad8aae9a8a36770e414fb3c7adc6657)

다음은 텍스트로 답하는 명령입니다.

```python
for command in ("how did the market do this week", "biggest losers today"):
    print(f'"{command}"  ->  {CALLS[command]}')
    print(CALLS[command].run(), "\n")
```

```
"how did the market do this week"  ->  market_summary(window='1w')
the board over 1w
  NVDA     254.12    9.62%    389,465,563
  AMD      184.20    1.51%    182,740,497
  AAPL     258.71    0.97%    223,818,998
  SPY      664.86    0.40%    138,617,365
  MSFT     451.35    0.26%    113,427,173
  TSLA     320.22   -0.97%    266,317,023 

"biggest losers today"  ->  top_movers(window='1d', direction='losers')
top 3 losers over 1d
  AMD      -0.57%  ->  184.20
  MSFT      0.67%  ->  451.35
  AAPL      1.40%  ->  258.71 
```

## confidence 읽기

`confidence`는 모든 판단을 곱한 값이 아니라 호출 안에서 가장 확신이 낮은 판단을 나타냅니다.
인자 하나만 틀려도 결과를 망치기에 충분하기 때문입니다. 곱은 다른 질문("모든 부분이 맞는가")에
답하는 값이고, 개별 판단이 불안하든 아니든 함수의 인자가 많아질수록 낮아집니다.

이 숫자가 어디서 나왔는지 인자별로 살펴보면 다음과 같습니다.

```python
call = CALLS["is amd tracking nvidia lately"]
print(f'"is amd tracking nvidia lately"  ->  {call}   confidence {call.confidence:.2f}')
for name, argument in call.arguments.items():
    top = sorted(argument.distribution.items(), key=lambda kv: -kv[1])[:3]
    shown = "omitted, default stands" if argument.omitted else repr(argument.value)
    print(
        f"  {name:<12}{shown:<26}p {argument.probability:.2f}   "
        + "  ".join(f"{k} {v:.2f}" for k, v in top)
    )
print(f"  weakest argument: {call.weakest().name}")
```

```
"is amd tracking nvidia lately"  ->  rolling_correlation(symbol='AMD', benchmark='NVDA')   confidence 0.69
  symbol      'AMD'                     p 0.93   AMD 0.93  NVDA 0.07  AAPL 0.00
  benchmark   'NVDA'                    p 0.69   NVDA 1.00  AAPL 0.00  AMD 0.00
  window      omitted, default stands   p 0.88   
  resolution  omitted, default stands   p 0.95   
  weakest argument: benchmark
```

여기서는 `window`와 `resolution`이 모두 빠졌습니다. "lately"라는 말로는 얼마나 과거까지 볼지,
어떤 봉을 쓸지 알 수 없기 때문입니다. 그래서 `rolling_correlation`은 자체 기본값인 한 달과
1시간 봉으로 실행됩니다. `stated` 질문은 바로 이럴 때 쓰려고 있는 것입니다. 이 질문이 없으면
명령이 기간을 전혀 언급하지 않았는데도 Choice가 어떤 window든 하나를 골라야 합니다.

## playground에서 열어 보기

아래 링크에는 명령 하나와, 그 명령이 고른 함수에 해당하는 질문이 들어 있습니다. 함수 설명 열 개를
선택지로 둔 Choice 하나와 `rolling_correlation`의 인자 네 개입니다. playground(플레이그라운드)에서
명령을 고치면 인자도 따라 바뀝니다.

```python
COMMAND = "plot rolling correlation between nvda and spy for the past month"
picked = CALLS[COMMAND]
playground_link = make_playground_link(
    COMMAND,
    {ROUTE: assistant.questions[ROUTE]}
    | {q: v for q, v in assistant.questions.items() if q.startswith(f"{picked.name}.")},
)
display(
    Markdown(
        f"🔗 [Open the command and its questions in the TypeSafe playground]({playground_link})"
    )
)
```

[TypeSafe playground에서 명령과 질문 열기 →](https://console.typesafe.ai/playground#share/N4IgJg9gxgrgtgUwHYBcAqCAeKQC4AEIADgDYQr4BOEJJAlkgOb5QSWUIkCGKdES+AEYIUAdwTJ8SAG5gu+LkjD4AzkQCe+AGZt8KABYJ8RLiopx+BkABpCRanCIoVGbHkLAAOiAD6PlBA0ft4EXiAo6kQIIfjeUPoQdFDRNrEgDGaUMFC8-Cox3gDq+jz4dCp6hvgwKgiUCioA1gzMBkYolFxgLQ0q5SiKFAH4kAD83rZxlHQodXRcMWH0Zj4q6nCCNPnu3mhVNXX4ooMVw41IEKJH+kn6quubJBW6CVdw2Xc3Zmya5fhkXQQYFsFyGEFUEgUSE0JkovFg3Hq8S4cPwuiQ8GElAmaTgKMaIlW8DxlHUBRAeyMB3qx1QzyQRnoDOMhzWGxoqlePVelSMogSJCMmxRygs0iBtlEMzuF1ULUF93ZJDlTEFyggMBQOO8pHIPnsSRSBF2+1qNJOenBZAgjQUFH4RjZjwA5BUDck0eL6rxEA0FCwSnDtelUJ05Op9TxZpQkOTKdUzQ1GhUefIIkQklxlR0uj1w-hGBAEBUdPUHYrHvgALTXUolIhRJAVUptNGN2ylOB0MDh2wMYatqBkWrB1iOFEIHwcFAwGPbY0U02HWnOPSicG6CwcCtbNECVsqLi+5Go4a1Pk3eIjbtCETR4PUWgtHysdicHh8WM7RdUxMr07gue+A8kOEC1CQmhiIBDy7iU4q3pIYo9AEjAiIYlAdkowGXJUdamAGiioWAwYqMSKIRmYPDzmk8bUkcFpplwggKhAWhSJidQlro5ZOhyNY3Iw+gqLYZCiMJChelwqH4GKCC2AEAzKtOs5fpMIDSDQH70BEcZLvUpjJthVwadwvCCrY8QQA26i2Lo0xNJoPEwcqJQVMIyDBgERA+LJlDUSav7LgxVCKASyjLPabH8rcO5PFQYFGLoWicNmVQWGYwZgJ0oiQKIX4LrRiYGSmOFaCie6Os52gpdoDj+lEXCNH2q7rn5FBgAgQ4MCkAC+PVqY+TKMC+bAcKZn4AHS8SQizeOmRppJZhrBhkHTZLkTbksUMXfFAtp-K2dEGT0TEahQNatuWwg9IgpizhKUhHkC2h0G14ypFMMxzAs7hhAAygACgAmuSgNA-JVR-QAZAD+AAKwAAwI2UShYPgACiaAAGIQ0YJIEhQ+HyPyNApGpAByABqAAiACC5Lk9I3bzLZWizAIojTCg7P4FTdPBrTACy1PkkL1O2LTYDSIoyTKILSTUPg1MIEzyTbGptO0wDAAyosNuZaJs5InMzDzms68Ggt-VjaDkvLUDUCorEoKzPMm9zkhWzbwZoH92v09+GAqNwrvG1zPO+-73h9QNNBDSNb7jfwE3CEg8T47N4SRAtcQJMtH0hpk62fv5IDbVeu37RUMy3j0Y6ws9UlcKt1a8hCrBYeWSBPcCbfqCKZhJI071qQ7X3TD9oTeGDoPA7j+DQ7DiPIwwHWYBj2Pz-jIh+sTApk2kfMBwujPM1wocc+HkhHwLwui8LEtSzLz324ryuq8WAta7r360-rcmGzdlfAQ5sf5qS9rbb8r8wLOwvkcYB+AIE+z9sfGixYQ6ALDqbSQkcA4xzSINZ8r4xofmTqndO+J3pTyzlEckFwYAzQLqtLIOQS7kmpkWU4elHq+nkLUDuyhpqWhYBAcc24m6rXev1AhcciGjXfBtCaUolCXEzvNckS1kgrSbGtVheRyQAAlSrlUEFwPaZQuGBX0k0E6mxNStwCL2NuJgzBHAkE1ZxphzCWH0LZb0VQXFDH0BwPGPiVAj0Wlzb6mcACMxFvyOK4DZNuplixDDDD0WoKg+j8D3BBYMMTRDklbIEtxCBGgFIsMUgJXiZI+ODAAZiqQkmpriDAhLqagISHZ8AAEcYAonvCAfB3hCFMATiQxRyjcpUPwGEdR356GMLUsw4u+jvwcOLG3Oih5NA8jKvUUx5jhjWg8aRK8+FEnJIMH8cQ5SIZ-AsF0vxlQ-j9MGXUKRscnzjOIQoyaHAnYkE1J+NR2cNF5y0UwnRLCNql2KKUUx9Q+gAC9HQJAYcoVsyk5y3hkggO6HB1QCBrOWLsGJZi2C0HQeC5LNTFipXQI2iEGD0vEuWDFGE0RlmZOGCJn1ozzFiXAckDoqx0tmFQEQKl1ZpDhiK781LxTitZZKnFm0C4xPleSalzKkAqopUYdVsrvAxP0OSTlEEpUzjnAU+JC45B0Ctca6O0jRmyN+fIpOSAJqApoCC-gsz5ngsWRqZZaRVl6I1QuTZliEysiSbWCgSK5Rou5SjaM0tszgluqRbcxq9xSJ6qkEAXAMyU04p+dw6kYnDKAA)


이 문서는 개발자 문서 플랫폼인 [Mintlify](https://mintlify.com)로 제작하고 호스팅합니다.
