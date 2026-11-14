# 요청 규칙과 오류

> 요청 제한, 압축, HTTP 상태 코드

## 요청 규칙

다음 규칙은 모든 요청에 적용됩니다.

| 규칙 | 제한 |
| - | - |
| 본문 크기 | 최대 32 MiB. 헤더는 포함하지 않습니다. |
| 인코딩 | byte order mark(BOM, 바이트 순서 표시)가 없는 UTF-8 JSON. |
| 중첩 | `state`, `instructions`, `criteria` 각각에서 객체와 배열의 중첩은 최대 64단계. 더 깊은 내용은 평탄하게 펼치세요. |
| 숫자 | 유한한 숫자만 허용됩니다. `NaN`, `Infinity`, 그리고 `1e400`처럼 표현할 수 없을 만큼 큰 숫자는 거부됩니다. 식별자와 정확해야 하는 큰 수량은 문자열로 보내세요. |

본문이 32 MiB를 넘으면 `413`을 반환하고, 나머지 규칙을 어기면 `400`을 반환합니다.

### 압축

본문은 `Content-Encoding: gzip`(또는 `x-gzip`)이나 `zstd`로 압축할 수 있습니다. 압축은 한 겹만 사용하고, zstd의 window는 최대 8 MiB로 유지하세요. 32 MiB 제한은 압축된 본문과 압축을 푼 본문 모두에 적용됩니다. 지원하지 않는 `Content-Encoding`을 쓰거나 압축을 두 겹 이상 적용하면 `415`를 반환하고, 압축된 본문의 형식이 잘못되었으면 `400`을 반환합니다.

## 오류

오류는 표준 HTTP 상태 코드를 사용하며, 무엇이 잘못되었는지 설명하는 JSON 본문이 함께 옵니다.

| 상태 | 의미 |
| - | - |
| `400 Bad Request` | 본문이 [요청 규칙](#request-rules)을 어겼습니다. 유효하지 않은 UTF-8, 지나친 중첩, 유한하지 않은 숫자, 형식이 잘못된 압축 본문 등이 해당합니다. `max_tokens_exceeded`가 함께 오면 요청이 모델의 [context length(컨텍스트 길이)](../models.md)보다 깁니다. |
| `401 Unauthorized` | API 키가 없거나 유효하지 않습니다. `Authorization` 헤더를 확인하세요. |
| `413 Payload Too Large` | 본문이 32 MiB를 넘습니다. [요청 규칙](#request-rules)을 참고하세요. |
| `415 Unsupported Media Type` | 본문이 지원하지 않는 `Content-Encoding`을 쓰거나 압축을 두 겹 이상 적용했습니다. [압축](#compression)을 참고하세요. |
| `422 Unprocessable Entity` | 요청 본문이 검증을 통과하지 못했습니다. 필수 필드가 빠졌거나 질문의 형식이 잘못된 경우 등입니다. 문제가 된 필드는 본문에 자세히 나옵니다. |
| `429 Too Many Requests` | rate limit(요청 한도)을 초과했습니다. 요청을 늦추고 잠시 후 retry(재시도)하세요. |
| `529 Overloaded` | TypeSafe가 일시적으로 과부하 상태입니다. 잠시 후 retry하세요. |

## Rate limit 처리하기

`429 Too Many Requests` 또는 `529 Overloaded` 응답을 받으면 즉시 retry하지 말고 exponential backoff(지수 백오프)를 적용해 요청을 retry하세요. 클라이언트 SDK는 이를 자동으로 처리하므로, SDK를 기본 retry 정책 그대로 사용한다면 따로 처리할 필요가 없습니다.


이 문서는 개발자 문서 플랫폼인 [Mintlify](https://mintlify.com)로 제작하고 호스팅합니다.
