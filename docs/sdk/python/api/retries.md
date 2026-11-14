# Retries

> RetryPolicy로 retry(재시도)를 설정합니다. 시도 횟수, 재시도할 상태 코드, backoff(백오프), retry 관련 헤더 처리 방식을 지정할 수 있습니다.

## typesafe\_sdk.RetryPolicy

`dataclass`

시그니처 복사
<pre tabIndex="0" aria-label="SDK signature">`RetryPolicy(
&#x20;   max\_retries: [int](https://docs.python.org/3/builtins/functions.html#int) = 2,
&#x20;   backoff\_initial: [float](https://docs.python.org/3/builtins/functions.html#float) = 0.5,
&#x20;   backoff\_max: [float](https://docs.python.org/3/builtins/functions.html#float) = 5.0,
&#x20;   backoff\_jitter: [float](https://docs.python.org/3/builtins/functions.html#float) = 0.25,
&#x20;   http\_statuses: [set](https://docs.python.org/3/builtins/stdtypes.html#set)\[[int](https://docs.python.org/3/builtins/functions.html#int)] = (
&#x20;       lambda: \{408, 429, \*[range](https://docs.python.org/3/builtins/stdtypes.html#range)(500, 600)}
&#x20;   )(),
&#x20;   respect\_retry\_after: [bool](https://docs.python.org/3/builtins/functions.html#bool) = True,
&#x20;   api\_connection\_error: [bool](https://docs.python.org/3/builtins/functions.html#bool) = True,
&#x20;   api\_timeout\_error: [bool](https://docs.python.org/3/builtins/functions.html#bool) = True,
&#x20;   exceptions: [set](https://docs.python.org/3/builtins/stdtypes.html#set)\[
&#x20;       [type](https://docs.python.org/3/builtins/functions.html#type)\[[BaseException](https://docs.python.org/3/builtins/exceptions.html#BaseException)]
&#x20;   ] = [set](https://docs.python.org/3/builtins/stdtypes.html#set)(),
&#x20;   predicate: [Callable](https://docs.python.org/3/library/collections.abc.html#collections.abc.Callable)\[\[[BaseException](https://docs.python.org/3/builtins/exceptions.html#BaseException)], [bool](https://docs.python.org/3/builtins/functions.html#bool)]
&#x20;   | None = None,
&#x20;   timeout: [float](https://docs.python.org/3/builtins/functions.html#float) | None = 30.0,
)`</pre>

SDK의 retry 동작을 정하는 설정입니다.

Examples(예제):

```python
from typesafe_sdk import RetryPolicy, TypeSafeClient

client = TypeSafeClient(
    retry=RetryPolicy(
        max_retries=3, timeout=10.0, http_statuses={429, 500, 502, 503, 504}
    )
)
```

### max\_retries

`class-attribute` `instance-attribute`

시그니처 복사

<pre tabIndex="0" aria-label="SDK signature">
  \`max\_retries

  :

  &#x20;

  [int](https://docs.python.org/3/builtins/functions.html#int)

  &#x20;

  \=

  &#x20;

  2\`
</pre>

첫 시도 이후에 retry하는 최대 횟수입니다. `0`으로 지정하면 retry하지 않습니다.

### backoff\_initial

`class-attribute` `instance-attribute`

시그니처 복사

<pre tabIndex="0" aria-label="SDK signature">
  \`backoff\_initial

  :

  &#x20;

  [float](https://docs.python.org/3/builtins/functions.html#float)

  &#x20;

  \=

  &#x20;

  0.5\`
</pre>

첫 backoff 지연 시간(초)입니다. 시도할 때마다 두 배로 늘어나며 `backoff_max`를 넘지 않습니다. 0으로 지정하면 backoff를 쓰지 않습니다.

### backoff\_max

`class-attribute` `instance-attribute`

시그니처 복사

<pre tabIndex="0" aria-label="SDK signature">
  \`backoff\_max

  :

  &#x20;

  [float](https://docs.python.org/3/builtins/functions.html#float)

  &#x20;

  \=

  &#x20;

  5.0\`
</pre>

backoff 지연 시간의 최댓값(초)입니다. 0으로 지정하면 backoff를 쓰지 않습니다.

### backoff\_jitter

`class-attribute` `instance-attribute`

시그니처 복사

<pre tabIndex="0" aria-label="SDK signature">
  \`backoff\_jitter

  :

  &#x20;

  [float](https://docs.python.org/3/builtins/functions.html#float)

  &#x20;

  \=

  &#x20;

  0.25\`
</pre>

각 backoff 지연 시간에서 무작위로 빼는 비율입니다. 0과 1 사이의 값입니다.

### http\_statuses

`class-attribute` `instance-attribute`

시그니처 복사
<pre tabIndex="0" aria-label="SDK signature">`http\_statuses: [set](https://docs.python.org/3/builtins/stdtypes.html#set)\[[int](https://docs.python.org/3/builtins/functions.html#int)] = [field](https://docs.python.org/3/library/dataclasses.html#dataclasses.field)(
&#x20;   default\_factory=lambda: \{
&#x20;       408,
&#x20;       429,
&#x20;       \*[range](https://docs.python.org/3/builtins/stdtypes.html#range)(500, 600),
&#x20;   }
)`</pre>

retry 대상이 되는 HTTP 상태 코드입니다.

### respect\_retry\_after

`class-attribute` `instance-attribute`

시그니처 복사

<pre tabIndex="0" aria-label="SDK signature">
  \`respect\_retry\_after

  :

  &#x20;

  [bool](https://docs.python.org/3/builtins/functions.html#bool)

  &#x20;

  \=

  &#x20;

  True\`
</pre>

응답 헤더 `Retry-After`와 `retry-after-ms`를 따를지 여부입니다.

### api\_connection\_error

`class-attribute` `instance-attribute`

시그니처 복사

<pre tabIndex="0" aria-label="SDK signature">
  \`api\_connection\_error

  :

  &#x20;

  [bool](https://docs.python.org/3/builtins/functions.html#bool)

  &#x20;

  \=

  &#x20;

  True\`
</pre>

`TypeSafeAPIConnectionError`가 발생했을 때 retry할지 여부입니다. 이 예외는 요청이 서버에 닿지 못하거나 서버에서 응답을 읽지 못할 때 발생합니다.

### api\_timeout\_error

`class-attribute` `instance-attribute`

시그니처 복사

<pre tabIndex="0" aria-label="SDK signature">
  \`api\_timeout\_error

  :

  &#x20;

  [bool](https://docs.python.org/3/builtins/functions.html#bool)

  &#x20;

  \=

  &#x20;

  True\`
</pre>

`TypeSafeAPITimeoutError`가 발생했을 때 retry할지 여부입니다. 이 예외는 요청이 타임아웃을 넘길 때 발생합니다.

### exceptions

`class-attribute` `instance-attribute`

시그니처 복사
<pre tabIndex="0" aria-label="SDK signature">`exceptions: [set](https://docs.python.org/3/builtins/stdtypes.html#set)\[[type](https://docs.python.org/3/builtins/functions.html#type)\[[BaseException](https://docs.python.org/3/builtins/exceptions.html#BaseException)]] = [field](https://docs.python.org/3/library/dataclasses.html#dataclasses.field)(
&#x20;   default\_factory=[set](https://docs.python.org/3/builtins/stdtypes.html#set)
)`</pre>

기본 규칙에 더해 retry를 일으킬 예외 타입을 추가로 지정합니다.

### predicate

`class-attribute` `instance-attribute`

시그니처 복사
<pre tabIndex="0" aria-label="SDK signature">`predicate: (
&#x20;   [Callable](https://docs.python.org/3/library/collections.abc.html#collections.abc.Callable)\[\[[BaseException](https://docs.python.org/3/builtins/exceptions.html#BaseException)], [bool](https://docs.python.org/3/builtins/functions.html#bool)] | None
) = None`</pre>

발생한 예외를 인자로 받아 호출되는 선택적 predicate(판별 함수)입니다. `True`를 반환하면 다른 규칙과 별개로 retry합니다.

### timeout

`class-attribute` `instance-attribute`

시그니처 복사

<pre tabIndex="0" aria-label="SDK signature">
  \`timeout

  :

  &#x20;

  [float](https://docs.python.org/3/builtins/functions.html#float)

  &#x20;

  |

  &#x20;

  None

  &#x20;

  \=

  &#x20;

  30.0\`
</pre>

SDK 호출 한 번에 쓸 수 있는 전체 retry 시간 예산(초)입니다. 첫 시도와 지연 시간을 모두 포함합니다. `None`으로 지정하면 제한이 없습니다.

지연 시간이 남은 예산과 같거나 그보다 긴 retry는 시작하지 않고, 마지막 오류를 다시 발생시킵니다.


이 문서는 개발자 문서 플랫폼인 [Mintlify](https://mintlify.com)로 제작하고 호스팅합니다.
