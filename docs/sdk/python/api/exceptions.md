# 예외

> TypeSafe API 오류, rate limit(요청 한도), 연결 실패, 타임아웃을 처리합니다.

## 기반 예외

## typesafe\_sdk.TypeSafeError

Bases(기반 클래스): `Exception`

SDK에서 발생하는 모든 실패의 기반 예외입니다.

## HTTP 오류

## typesafe\_sdk.TypeSafeAPIError

Bases: `TypeSafeError`

실패한 HTTP 응답을 나타내며, 응답 본문과 요청 메타데이터를 함께 담습니다.

### status

`instance-attribute`

```python
status = status
```

HTTP 응답 상태 코드입니다.

### body

`instance-attribute`

```python
body = body
```

서버가 보낸 JSON 오류 본문 또는 일반 텍스트 응답입니다. 본문이 비어 있으면 `None`입니다.

### headers

`instance-attribute`

```python
headers = headers
```

HTTP 응답 헤더입니다.

### endpoint

`instance-attribute`

```python
endpoint = endpoint
```

요청 메서드와 URL입니다(확인할 수 있는 경우). 자격 증명, 쿼리 매개변수, 프래그먼트는 포함하지 않습니다.

### request\_id

`property`

시그니처 복사

<pre tabIndex="0" aria-label="SDK signature">
  \`request\_id

  :

  &#x20;

  [str](https://docs.python.org/3/builtins/stdtypes.html#str)

  &#x20;

  |

  &#x20;

  None\`
</pre>

`x-typesafe-request-id` 응답 헤더 값입니다. 헤더가 없으면 `None`입니다.

## typesafe\_sdk.TypeSafeBadRequestError

Bases: `TypeSafeAPIError`

요청이 유효하지 않습니다(400).

## typesafe\_sdk.TypeSafeAuthenticationError

Bases: `TypeSafeAPIError`

인증에 실패했습니다(401).

## typesafe\_sdk.TypeSafePermissionDeniedError

Bases: `TypeSafeAPIError`

접근이 거부되었습니다(403).

## typesafe\_sdk.TypeSafeNotFoundError

Bases: `TypeSafeAPIError`

리소스를 찾을 수 없습니다(404).

## typesafe\_sdk.TypeSafeUnprocessableEntityError

Bases: `TypeSafeAPIError`

요청이 서버의 유효성 검사를 통과하지 못했습니다(422).

## typesafe\_sdk.TypeSafeRateLimitError

Bases: `TypeSafeAPIError`

rate limit을 초과했습니다(429).

### retry\_after\_ms

`instance-attribute`

```python
retry_after_ms = parse_retry_after(headers)
```

서버가 요청한 대기 시간(밀리초)입니다. 알 수 없으면 `None`입니다.

## typesafe\_sdk.TypeSafeInternalServerError

Bases: `TypeSafeAPIError`

서버가 요청을 처리하지 못했습니다(5xx).

## 연결 오류

## typesafe\_sdk.TypeSafeAPIConnectionError

Bases: `TypeSafeError`, `ConnectionError`

HTTP 응답을 받지 못한 채 요청이 실패했습니다.

## typesafe\_sdk.TypeSafeAPITimeoutError

Bases: `TypeSafeAPIConnectionError`, `TimeoutError`

요청이 설정된 타임아웃을 넘겼습니다.

### timeout

`instance-attribute`

```python
timeout = timeout
```

요청에 사용된 타임아웃 설정입니다. 초 단위 숫자이거나 `httpx2.Timeout`입니다.

## 응답 검증

## typesafe\_sdk.TypeSafeAPIResponseValidationError

Bases: `TypeSafeAPIError`

HTTP 응답 자체는 성공했지만, 본문에 필수 데이터가 없거나 그 구조가 올바르지 않습니다.

### field\_path

`instance-attribute`

```python
field_path = field_path
```

문제가 된 필드까지의 경로를 점으로 이어 나타낸 값입니다(예: `answers.tone.confidence`).

### args

`instance-attribute`

```python
args = (
    status,
    body,
    headers,
    field_path,
    endpoint,
)
```


이 문서는 개발자 문서 플랫폼인 [Mintlify](https://mintlify.com)로 제작하고 호스팅합니다.
