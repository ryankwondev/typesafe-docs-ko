# 답변과 응답

> TypeSafe API가 반환하는 답변, confidence(신뢰도) 점수, token(토큰) 사용량, 사용 가능한 모델을 읽는 방법입니다.

## 응답

## typesafe\_sdk.SystemOneResponse

`pydantic-model`

Bases(기반 클래스): `Response`

질문 유형별로 묶은 답변과 함께 모델, 사용량 메타데이터를 담습니다.

자세한 내용은 [System One](https://docs.typesafe.ai/go/system-one.md)을 참고하세요.

> [!NOTE]
>
> **JSON schema 보기:**
>
> > [accordion] 세부 정보
> >
> > ```json theme={null}
> > {
> >   "$defs": {
> >     "BoxAnswer": {
> >       "description": "A located box with its confidence.",
> >       "properties": {
> >         "type": {
> >           "const": "box",
> >           "default": "box",
> >           "title": "Type",
> >           "type": "string"
> >         },
> >         "box": {
> >           "$ref": "#/$defs/BoxCoordinates"
> >         },
> >         "confidence": {
> >           "anyOf": [
> >             {
> >               "type": "number"
> >             },
> >             {
> >               "type": "null"
> >             }
> >           ],
> >           "title": "Confidence"
> >         }
> >       },
> >       "required": [
> >         "box",
> >         "confidence"
> >       ],
> >       "title": "BoxAnswer",
> >       "type": "object"
> >     },
> >     "BoxCoordinates": {
> >       "description": "A box's edges as fractions of the image's height (`top`, `bottom`) and width (`left`, `right`).",
> >       "properties": {
> >         "top": {
> >           "description": "Top edge of the box, as a fraction of image height from the top (0-1).",
> >           "examples": [
> >             0.2
> >           ],
> >           "title": "Top",
> >           "type": "number"
> >         },
> >         "bottom": {
> >           "description": "Bottom edge of the box, as a fraction of image height from the top (0-1).",
> >           "examples": [
> >             0.66
> >           ],
> >           "title": "Bottom",
> >           "type": "number"
> >         },
> >         "left": {
> >           "description": "Left edge of the box, as a fraction of image width from the left (0-1).",
> >           "examples": [
> >             0.19
> >           ],
> >           "title": "Left",
> >           "type": "number"
> >         },
> >         "right": {
> >           "description": "Right edge of the box, as a fraction of image width from the left (0-1).",
> >           "examples": [
> >             0.68
> >           ],
> >           "title": "Right",
> >           "type": "number"
> >         }
> >       },
> >       "required": [
> >         "top",
> >         "bottom",
> >         "left",
> >         "right"
> >       ],
> >       "title": "BoxCoordinates",
> >       "type": "object"
> >     },
> >     "ChoiceAnswer": {
> >       "description": "A selected label and its probabilities.\n\nSee the [choice primitive](https://docs.typesafe.ai/go/choice) for details.",
> >       "properties": {
> >         "type": {
> >           "const": "choice",
> >           "default": "choice",
> >           "title": "Type",
> >           "type": "string"
> >         },
> >         "choice": {
> >           "description": "The name of the choice with the highest probability among the question's criteria.",
> >           "examples": [
> >             "angry"
> >           ],
> >           "title": "Choice",
> >           "type": "string"
> >         },
> >         "confidence": {
> >           "description": "Confidence in the selected choice, from 0 to 1. Higher values indicate greater certainty; use lower values to flag uncertain selections for review.",
> >           "examples": [
> >             0.9
> >           ],
> >           "title": "Confidence",
> >           "type": "number"
> >         },
> >         "probabilities": {
> >           "additionalProperties": {
> >             "type": "number"
> >           },
> >           "description": "Probability of each choice in criteria, keyed by choice name, from 0 to 1. Shows how likely the alternatives are; values sum to approximately 1.",
> >           "examples": [
> >             {
> >               "angry": 0.8,
> >               "calm": 0.1,
> >               "excited": 0.1
> >             }
> >           ],
> >           "title": "Probabilities",
> >           "type": "object"
> >         }
> >       },
> >       "required": [
> >         "choice",
> >         "confidence",
> >         "probabilities"
> >       ],
> >       "title": "ChoiceAnswer",
> >       "type": "object"
> >     },
> >     "NoulAnswer": {
> >       "description": "A yes/no answer.\n\nSee the [noul primitive](https://docs.typesafe.ai/go/noul) for details.",
> >       "properties": {
> >         "type": {
> >           "const": "noul",
> >           "default": "noul",
> >           "title": "Type",
> >           "type": "string"
> >         },
> >         "noul": {
> >           "description": "Probability of a yes answer or a true statement, from 0 to 1. Values near 1 favor yes or true, values near 0 favor no or false, and values near 0.5 indicate uncertainty.",
> >           "examples": [
> >             0.98
> >           ],
> >           "title": "Noul",
> >           "type": "number"
> >         }
> >       },
> >       "required": [
> >         "noul"
> >       ],
> >       "title": "NoulAnswer",
> >       "type": "object"
> >     },
> >     "ScoreAnswer": {
> >       "description": "An expected score with its rubric and probabilities.\n\nSee the [score primitive](https://docs.typesafe.ai/go/score) for details.",
> >       "properties": {
> >         "type": {
> >           "const": "score",
> >           "default": "score",
> >           "title": "Type",
> >           "type": "string"
> >         },
> >         "score": {
> >           "description": "Expected score: the probability-weighted average of the rubric levels. May fall between integer levels.",
> >           "examples": [
> >             1.7
> >           ],
> >           "title": "Score",
> >           "type": "number"
> >         },
> >         "confidence": {
> >           "description": "Confidence in the score, from 0 to 1. Higher values indicate greater certainty; use lower values to flag uncertain ratings for review.",
> >           "examples": [
> >             0.9
> >           ],
> >           "title": "Confidence",
> >           "type": "number"
> >         },
> >         "legend": {
> >           "additionalProperties": {
> >             "anyOf": [
> >               {
> >                 "type": "string"
> >               },
> >               {
> >                 "additionalProperties": true,
> >                 "type": "object"
> >               },
> >               {
> >                 "items": {},
> >                 "type": "array"
> >               }
> >             ]
> >           },
> >           "title": "Legend",
> >           "type": "object"
> >         },
> >         "probabilities": {
> >           "additionalProperties": {
> >             "type": "number"
> >           },
> >           "title": "Probabilities",
> >           "type": "object"
> >         }
> >       },
> >       "required": [
> >         "score",
> >         "confidence",
> >         "legend",
> >         "probabilities"
> >       ],
> >       "title": "ScoreAnswer",
> >       "type": "object"
> >     },
> >     "Usage": {
> >       "description": "Token counts for a request, when reported by the API.",
> >       "properties": {
> >         "input_tokens": {
> >           "anyOf": [
> >             {
> >               "type": "integer"
> >             },
> >             {
> >               "type": "null"
> >             }
> >           ],
> >           "default": null,
> >           "title": "Input Tokens"
> >         },
> >         "output_tokens": {
> >           "anyOf": [
> >             {
> >               "type": "integer"
> >             },
> >             {
> >               "type": "null"
> >             }
> >           ],
> >           "default": null,
> >           "title": "Output Tokens"
> >         }
> >       },
> >       "title": "Usage",
> >       "type": "object"
> >     }
> >   },
> >   "description": "Answers grouped by question type with model and usage metadata.\n\nSee [System One](https://docs.typesafe.ai/go/system-one) for details.",
> >   "properties": {
> >     "model": {
> >       "title": "Model",
> >       "type": "string"
> >     },
> >     "usage": {
> >       "$ref": "#/$defs/Usage"
> >     },
> >     "answers": {
> >       "additionalProperties": {
> >         "discriminator": {
> >           "mapping": {
> >             "box": "#/$defs/BoxAnswer",
> >             "choice": "#/$defs/ChoiceAnswer",
> >             "noul": "#/$defs/NoulAnswer",
> >             "score": "#/$defs/ScoreAnswer"
> >           },
> >           "propertyName": "type"
> >         },
> >         "oneOf": [
> >           {
> >             "$ref": "#/$defs/NoulAnswer"
> >           },
> >           {
> >             "$ref": "#/$defs/ChoiceAnswer"
> >           },
> >           {
> >             "$ref": "#/$defs/ScoreAnswer"
> >           },
> >           {
> >             "$ref": "#/$defs/BoxAnswer"
> >           }
> >         ]
> >       },
> >       "title": "Answers",
> >       "type": "object"
> >     }
> >   },
> >   "required": [
> >     "model",
> >     "usage"
> >   ],
> >   "title": "SystemOneResponse",
> >   "type": "object"
> > }
> > ```

Config(설정):

* `extra`: `ignore`
* `frozen`: `True`
* `strict`: `True`

Fields(필드):

* `model` (`str`)
* `usage` (`Usage`)
* `answers` (`dict[str, Answer]`)

### request\_id

`cached` `property`

시그니처 복사

<pre tabIndex="0" aria-label="SDK signature">
  \`request\_id

  :

  &#x20;

  [str](https://docs.python.org/3/builtins/stdtypes.html#str)\`
</pre>

`x-typesafe-request-id` 응답 헤더입니다.

### raw\_http\_response

`property`

```python
raw_http_response: httpx2.Response
```

내부의 `httpx2.Response` 객체이며, 상태 코드, 헤더, 본문을 확인할 수 있습니다.

### model\_config

`class-attribute` `instance-attribute`

```python
model_config = ConfigDict(
    extra="ignore", frozen=True, strict=True
)
```

### model

`pydantic-field`

시그니처 복사

<pre tabIndex="0" aria-label="SDK signature">
  \`model

  :

  &#x20;

  [str](https://docs.python.org/3/builtins/stdtypes.html#str)\`
</pre>

요청에 답하는 데 사용된 모델입니다.

### usage

`pydantic-field`

시그니처 복사

<pre tabIndex="0" aria-label="SDK signature">
  \`usage

  :

  &#x20;

  [Usage](responses.md#typesafe_sdk.Usage)\`
</pre>

요청의 token 사용량입니다.

### answers

`pydantic-field`

시그니처 복사

<pre tabIndex="0" aria-label="SDK signature">
  \`answers

  :

  &#x20;

  [dict](https://docs.python.org/3/builtins/stdtypes.html#dict)

  \[

  [str](https://docs.python.org/3/builtins/stdtypes.html#str)

  ,

  &#x20;

  [Answer](responses.md#typesafe_sdk.Answer)

  ]\`
</pre>

질문 이름을 키로 하는 모든 답변 객체입니다.

### nouls

`cached` `property`

시그니처 복사

<pre tabIndex="0" aria-label="SDK signature">
  \`nouls

  :

  &#x20;

  [dict](https://docs.python.org/3/builtins/stdtypes.html#dict)

  \[

  [str](https://docs.python.org/3/builtins/stdtypes.html#str)

  ,

  &#x20;

  [NoulAnswer](responses.md#typesafe_sdk.NoulAnswer)

  ]\`
</pre>

질문 이름을 키로 하는 예/아니요 답변입니다.

### choices

`cached` `property`

시그니처 복사

<pre tabIndex="0" aria-label="SDK signature">
  \`choices

  :

  &#x20;

  [dict](https://docs.python.org/3/builtins/stdtypes.html#dict)

  \[

  [str](https://docs.python.org/3/builtins/stdtypes.html#str)

  ,

  &#x20;

  [ChoiceAnswer](responses.md#typesafe_sdk.ChoiceAnswer)

  ]\`
</pre>

질문 이름을 키로 하는 choice 답변입니다.

### scores

`cached` `property`

시그니처 복사

<pre tabIndex="0" aria-label="SDK signature">
  \`scores

  :

  &#x20;

  [dict](https://docs.python.org/3/builtins/stdtypes.html#dict)

  \[

  [str](https://docs.python.org/3/builtins/stdtypes.html#str)

  ,

  &#x20;

  [ScoreAnswer](responses.md#typesafe_sdk.ScoreAnswer)

  ]\`
</pre>

질문 이름을 키로 하는 score 답변입니다.

### boxes

`cached` `property`

시그니처 복사

<pre tabIndex="0" aria-label="SDK signature">
  \`boxes

  :

  &#x20;

  [dict](https://docs.python.org/3/builtins/stdtypes.html#dict)

  \[

  [str](https://docs.python.org/3/builtins/stdtypes.html#str)

  ,

  &#x20;

  [BoxAnswer](responses.md#typesafe_sdk.BoxAnswer)

  ]\`
</pre>

질문 이름을 키로 하는 box 답변입니다.

## typesafe\_sdk.Usage

`pydantic-model`

Bases(기반 클래스): `wire.Usage`

요청의 token 수이며, API가 보고한 경우에만 값이 있습니다.

> [!NOTE]
>
> **JSON schema 보기:**
>
> > [accordion] 세부 정보
> >
> > ```json theme={null}
> > {
> >   "description": "Token counts for a request, when reported by the API.",
> >   "properties": {
> >     "input_tokens": {
> >       "anyOf": [
> >         {
> >           "type": "integer"
> >         },
> >         {
> >           "type": "null"
> >         }
> >       ],
> >       "default": null,
> >       "title": "Input Tokens"
> >     },
> >     "output_tokens": {
> >       "anyOf": [
> >         {
> >           "type": "integer"
> >         },
> >         {
> >           "type": "null"
> >         }
> >       ],
> >       "default": null,
> >       "title": "Output Tokens"
> >     }
> >   },
> >   "title": "Usage",
> >   "type": "object"
> > }
> > ```

Config(설정):

* `extra`: `ignore`
* `frozen`: `True`
* `strict`: `True`

Fields(필드):

* `input_tokens` (`int | None`)
* `output_tokens` (`int | None`)

### model\_config

`class-attribute` `instance-attribute`

```python
model_config = ConfigDict(
    extra="ignore", frozen=True, strict=True
)
```

### input\_tokens

`pydantic-field`

시그니처 복사

<pre tabIndex="0" aria-label="SDK signature">
  \`input\_tokens

  :

  &#x20;

  [int](https://docs.python.org/3/builtins/functions.html#int)

  &#x20;

  |

  &#x20;

  None

  &#x20;

  \=

  &#x20;

  None\`
</pre>

사용된 입력 token 수입니다. API가 보고하지 않았으면 `None`입니다.

### output\_tokens

`pydantic-field`

시그니처 복사

<pre tabIndex="0" aria-label="SDK signature">
  \`output\_tokens

  :

  &#x20;

  [int](https://docs.python.org/3/builtins/functions.html#int)

  &#x20;

  |

  &#x20;

  None

  &#x20;

  \=

  &#x20;

  None\`
</pre>

사용된 출력 token 수입니다. API가 보고하지 않았으면 `None`입니다.

## 답변

## typesafe\_sdk.NoulAnswer

`pydantic-model`

Bases(기반 클래스): `wire.NoulAnswer`

예/아니요 답변입니다.

자세한 내용은 [noul primitive(프리미티브: 기본 구성 요소)](https://docs.typesafe.ai/go/noul.md)를 참고하세요.

> [!NOTE]
>
> **JSON schema 보기:**
>
> > [accordion] 세부 정보
> >
> > ```json theme={null}
> > {
> >   "description": "A yes/no answer.\n\nSee the [noul primitive](https://docs.typesafe.ai/go/noul) for details.",
> >   "properties": {
> >     "type": {
> >       "const": "noul",
> >       "default": "noul",
> >       "title": "Type",
> >       "type": "string"
> >     },
> >     "noul": {
> >       "description": "Probability of a yes answer or a true statement, from 0 to 1. Values near 1 favor yes or true, values near 0 favor no or false, and values near 0.5 indicate uncertainty.",
> >       "examples": [
> >         0.98
> >       ],
> >       "title": "Noul",
> >       "type": "number"
> >     }
> >   },
> >   "required": [
> >     "noul"
> >   ],
> >   "title": "NoulAnswer",
> >   "type": "object"
> > }
> > ```

Config(설정):

* `extra`: `ignore`
* `frozen`: `True`
* `strict`: `True`

Fields(필드):

* `noul` (`float`)
* `type` (`Literal['noul']`)

### noul

`pydantic-field`

시그니처 복사

<pre tabIndex="0" aria-label="SDK signature">
  \`noul

  :

  &#x20;

  [float](https://docs.python.org/3/builtins/functions.html#float)\`
</pre>

답이 예이거나 진술이 참일 확률이며, 0에서 1 사이의 값입니다. 1에 가까우면 예 또는 참, 0에 가까우면 아니요 또는 거짓 쪽이고, 0.5에 가까우면 불확실하다는 뜻입니다.

### model\_config

`class-attribute` `instance-attribute`

```python
model_config = ConfigDict(
    extra="ignore", frozen=True, strict=True
)
```

## typesafe\_sdk.ChoiceAnswer

`pydantic-model`

Bases(기반 클래스): `wire.ChoiceAnswer`

선택된 label(레이블)과 각 label의 확률입니다.

자세한 내용은 [choice primitive](https://docs.typesafe.ai/go/choice.md)를 참고하세요.

> [!NOTE]
>
> **JSON schema 보기:**
>
> > [accordion] 세부 정보
> >
> > ```json theme={null}
> > {
> >   "description": "A selected label and its probabilities.\n\nSee the [choice primitive](https://docs.typesafe.ai/go/choice) for details.",
> >   "properties": {
> >     "type": {
> >       "const": "choice",
> >       "default": "choice",
> >       "title": "Type",
> >       "type": "string"
> >     },
> >     "choice": {
> >       "description": "The name of the choice with the highest probability among the question's criteria.",
> >       "examples": [
> >         "angry"
> >       ],
> >       "title": "Choice",
> >       "type": "string"
> >     },
> >     "confidence": {
> >       "description": "Confidence in the selected choice, from 0 to 1. Higher values indicate greater certainty; use lower values to flag uncertain selections for review.",
> >       "examples": [
> >         0.9
> >       ],
> >       "title": "Confidence",
> >       "type": "number"
> >     },
> >     "probabilities": {
> >       "additionalProperties": {
> >         "type": "number"
> >       },
> >       "description": "Probability of each choice in criteria, keyed by choice name, from 0 to 1. Shows how likely the alternatives are; values sum to approximately 1.",
> >       "examples": [
> >         {
> >           "angry": 0.8,
> >           "calm": 0.1,
> >           "excited": 0.1
> >         }
> >       ],
> >       "title": "Probabilities",
> >       "type": "object"
> >     }
> >   },
> >   "required": [
> >     "choice",
> >     "confidence",
> >     "probabilities"
> >   ],
> >   "title": "ChoiceAnswer",
> >   "type": "object"
> > }
> > ```

Config(설정):

* `extra`: `ignore`
* `frozen`: `True`
* `strict`: `True`

Fields(필드):

* `choice` (`str`)
* `confidence` (`float`)
* `probabilities` (`dict[str, float]`)
* `type` (`Literal['choice']`)

### choice

`pydantic-field`

시그니처 복사

<pre tabIndex="0" aria-label="SDK signature">
  \`choice

  :

  &#x20;

  [str](https://docs.python.org/3/builtins/stdtypes.html#str)\`
</pre>

질문의 criteria(판단 기준)에 있는 선택지 가운데 확률이 가장 높은 선택지의 이름입니다.

### confidence

`pydantic-field`

시그니처 복사

<pre tabIndex="0" aria-label="SDK signature">
  \`confidence

  :

  &#x20;

  [float](https://docs.python.org/3/builtins/functions.html#float)\`
</pre>

선택된 선택지에 대한 confidence이며, 0에서 1 사이의 값입니다. 값이 높을수록 확실하다는 뜻이므로, 값이 낮은 불확실한 선택은 검토 대상으로 표시하는 데 활용하세요.

### probabilities

`pydantic-field`

시그니처 복사

<pre tabIndex="0" aria-label="SDK signature">
  \`probabilities

  :

  &#x20;

  [dict](https://docs.python.org/3/builtins/stdtypes.html#dict)

  \[

  [str](https://docs.python.org/3/builtins/stdtypes.html#str)

  ,

  &#x20;

  [float](https://docs.python.org/3/builtins/functions.html#float)

  ]\`
</pre>

criteria에 있는 각 선택지의 확률이며, 선택지 이름을 키로 하고 값은 0에서 1 사이입니다. 다른 선택지의 가능성이 어느 정도인지 보여 주며, 값을 모두 더하면 약 1이 됩니다.

### model\_config

`class-attribute` `instance-attribute`

```python
model_config = ConfigDict(
    extra="ignore", frozen=True, strict=True
)
```

## typesafe\_sdk.ScoreAnswer

`pydantic-model`

Bases(기반 클래스): `wire.ScoreAnswer`

기대 점수와 그 rubric(루브릭: 채점 기준표), 확률입니다.

자세한 내용은 [score primitive](https://docs.typesafe.ai/go/score.md)를 참고하세요.

> [!NOTE]
>
> **JSON schema 보기:**
>
> > [accordion] 세부 정보
> >
> > ```json theme={null}
> > {
> >   "description": "An expected score with its rubric and probabilities.\n\nSee the [score primitive](https://docs.typesafe.ai/go/score) for details.",
> >   "properties": {
> >     "type": {
> >       "const": "score",
> >       "default": "score",
> >       "title": "Type",
> >       "type": "string"
> >     },
> >     "score": {
> >       "description": "Expected score: the probability-weighted average of the rubric levels. May fall between integer levels.",
> >       "examples": [
> >         1.7
> >       ],
> >       "title": "Score",
> >       "type": "number"
> >     },
> >     "confidence": {
> >       "description": "Confidence in the score, from 0 to 1. Higher values indicate greater certainty; use lower values to flag uncertain ratings for review.",
> >       "examples": [
> >         0.9
> >       ],
> >       "title": "Confidence",
> >       "type": "number"
> >     },
> >     "legend": {
> >       "additionalProperties": {
> >         "anyOf": [
> >           {
> >             "type": "string"
> >           },
> >           {
> >             "additionalProperties": true,
> >             "type": "object"
> >           },
> >           {
> >             "items": {},
> >             "type": "array"
> >           }
> >         ]
> >       },
> >       "title": "Legend",
> >       "type": "object"
> >     },
> >     "probabilities": {
> >       "additionalProperties": {
> >         "type": "number"
> >       },
> >       "title": "Probabilities",
> >       "type": "object"
> >     }
> >   },
> >   "required": [
> >     "score",
> >     "confidence",
> >     "legend",
> >     "probabilities"
> >   ],
> >   "title": "ScoreAnswer",
> >   "type": "object"
> > }
> > ```

Config(설정):

* `extra`: `ignore`
* `frozen`: `True`
* `strict`: `True`

Fields(필드):

* `score` (`float`)
* `confidence` (`float`)
* `type` (`Literal['score']`)
* `legend` (`dict[int, str | dict[str, Any] | list[Any]]`)
* `probabilities` (`dict[int, float]`)

### score

`pydantic-field`

시그니처 복사

<pre tabIndex="0" aria-label="SDK signature">
  \`score

  :

  &#x20;

  [float](https://docs.python.org/3/builtins/functions.html#float)\`
</pre>

기대 점수입니다. rubric의 각 단계를 확률로 가중 평균한 값이며, 정수 단계 사이의 값이 나올 수 있습니다.

### confidence

`pydantic-field`

시그니처 복사

<pre tabIndex="0" aria-label="SDK signature">
  \`confidence

  :

  &#x20;

  [float](https://docs.python.org/3/builtins/functions.html#float)\`
</pre>

점수에 대한 confidence이며, 0에서 1 사이의 값입니다. 값이 높을수록 확실하다는 뜻이므로, 값이 낮은 불확실한 평가는 검토 대상으로 표시하는 데 활용하세요.

### model\_config

`class-attribute` `instance-attribute`

```python
model_config = ConfigDict(
    extra="ignore", frozen=True, strict=True
)
```

### legend

`pydantic-field`

시그니처 복사
<pre tabIndex="0" aria-label="SDK signature">`legend: [dict](https://docs.python.org/3/builtins/stdtypes.html#dict)\[
&#x20;   [int](https://docs.python.org/3/builtins/functions.html#int), [str](https://docs.python.org/3/builtins/stdtypes.html#str) | [dict](https://docs.python.org/3/builtins/stdtypes.html#dict)\[[str](https://docs.python.org/3/builtins/stdtypes.html#str), [Any](https://docs.python.org/3/library/typing.html#typing.Any)] | [list](https://docs.python.org/3/builtins/stdtypes.html#list)\[[Any](https://docs.python.org/3/library/typing.html#typing.Any)]
]`</pre>

정수 점수를 키로 하는 rubric 설명입니다.

### probabilities

`pydantic-field`

시그니처 복사

<pre tabIndex="0" aria-label="SDK signature">
  \`probabilities

  :

  &#x20;

  [dict](https://docs.python.org/3/builtins/stdtypes.html#dict)

  \[

  [int](https://docs.python.org/3/builtins/functions.html#int)

  ,

  &#x20;

  [float](https://docs.python.org/3/builtins/functions.html#float)

  ]\`
</pre>

정수 점수를 키로 하는 확률입니다.

## typesafe\_sdk.BoxAnswer

`pydantic-model`

Bases(기반 클래스): `wire.BoxAnswer`

찾아낸 box와 그 confidence입니다.

> [!NOTE]
>
> **JSON schema 보기:**
>
> > [accordion] 세부 정보
> >
> > ```json theme={null}
> > {
> >   "$defs": {
> >     "BoxCoordinates": {
> >       "description": "A box's edges as fractions of the image's height (`top`, `bottom`) and width (`left`, `right`).",
> >       "properties": {
> >         "top": {
> >           "description": "Top edge of the box, as a fraction of image height from the top (0-1).",
> >           "examples": [
> >             0.2
> >           ],
> >           "title": "Top",
> >           "type": "number"
> >         },
> >         "bottom": {
> >           "description": "Bottom edge of the box, as a fraction of image height from the top (0-1).",
> >           "examples": [
> >             0.66
> >           ],
> >           "title": "Bottom",
> >           "type": "number"
> >         },
> >         "left": {
> >           "description": "Left edge of the box, as a fraction of image width from the left (0-1).",
> >           "examples": [
> >             0.19
> >           ],
> >           "title": "Left",
> >           "type": "number"
> >         },
> >         "right": {
> >           "description": "Right edge of the box, as a fraction of image width from the left (0-1).",
> >           "examples": [
> >             0.68
> >           ],
> >           "title": "Right",
> >           "type": "number"
> >         }
> >       },
> >       "required": [
> >         "top",
> >         "bottom",
> >         "left",
> >         "right"
> >       ],
> >       "title": "BoxCoordinates",
> >       "type": "object"
> >     }
> >   },
> >   "description": "A located box with its confidence.",
> >   "properties": {
> >     "type": {
> >       "const": "box",
> >       "default": "box",
> >       "title": "Type",
> >       "type": "string"
> >     },
> >     "box": {
> >       "$ref": "#/$defs/BoxCoordinates"
> >     },
> >     "confidence": {
> >       "anyOf": [
> >         {
> >           "type": "number"
> >         },
> >         {
> >           "type": "null"
> >         }
> >       ],
> >       "title": "Confidence"
> >     }
> >   },
> >   "required": [
> >     "box",
> >     "confidence"
> >   ],
> >   "title": "BoxAnswer",
> >   "type": "object"
> > }
> > ```

Config(설정):

* `extra`: `ignore`
* `frozen`: `True`
* `strict`: `True`

Fields(필드):

* `type` (`Literal['box']`)
* `box` (`BoxCoordinates`)
* `confidence` (`float | None`)

### model\_config

`class-attribute` `instance-attribute`

```python
model_config = ConfigDict(
    extra="ignore", frozen=True, strict=True
)
```

### box

`pydantic-field`

시그니처 복사

<pre tabIndex="0" aria-label="SDK signature">
  \`box

  :

  &#x20;

  [BoxCoordinates](responses.md#typesafe_sdk.BoxCoordinates)\`
</pre>

box의 네 변 위치이며, 이미지 높이와 너비에 대한 비율(0에서 1 사이)로 나타냅니다.

### confidence

`pydantic-field`

시그니처 복사

<pre tabIndex="0" aria-label="SDK signature">
  \`confidence

  :

  &#x20;

  [float](https://docs.python.org/3/builtins/functions.html#float)

  &#x20;

  |

  &#x20;

  None\`
</pre>

box에 대한 confidence이며, 0에서 1 사이의 값입니다. 모델이 제공하지 않으면 `None`입니다.

## typesafe\_sdk.BoxCoordinates

`pydantic-model`

Bases(기반 클래스): `wire.BoxCoordinates`

box의 네 변 위치이며, 이미지 높이에 대한 비율(`top`, `bottom`)과 너비에 대한 비율(`left`, `right`)로 나타냅니다.

> [!NOTE]
>
> **JSON schema 보기:**
>
> > [accordion] 세부 정보
> >
> > ```json theme={null}
> > {
> >   "description": "A box's edges as fractions of the image's height (`top`, `bottom`) and width (`left`, `right`).",
> >   "properties": {
> >     "top": {
> >       "description": "Top edge of the box, as a fraction of image height from the top (0-1).",
> >       "examples": [
> >         0.2
> >       ],
> >       "title": "Top",
> >       "type": "number"
> >     },
> >     "bottom": {
> >       "description": "Bottom edge of the box, as a fraction of image height from the top (0-1).",
> >       "examples": [
> >         0.66
> >       ],
> >       "title": "Bottom",
> >       "type": "number"
> >     },
> >     "left": {
> >       "description": "Left edge of the box, as a fraction of image width from the left (0-1).",
> >       "examples": [
> >         0.19
> >       ],
> >       "title": "Left",
> >       "type": "number"
> >     },
> >     "right": {
> >       "description": "Right edge of the box, as a fraction of image width from the left (0-1).",
> >       "examples": [
> >         0.68
> >       ],
> >       "title": "Right",
> >       "type": "number"
> >     }
> >   },
> >   "required": [
> >     "top",
> >     "bottom",
> >     "left",
> >     "right"
> >   ],
> >   "title": "BoxCoordinates",
> >   "type": "object"
> > }
> > ```

Config(설정):

* `extra`: `ignore`
* `frozen`: `True`
* `strict`: `True`

Fields(필드):

* `top` (`float`)
* `bottom` (`float`)
* `left` (`float`)
* `right` (`float`)

### top

`pydantic-field`

시그니처 복사

<pre tabIndex="0" aria-label="SDK signature">
  \`top

  :

  &#x20;

  [float](https://docs.python.org/3/builtins/functions.html#float)\`
</pre>

box의 위쪽 변이며, 이미지 위쪽 끝에서부터 잰 거리를 이미지 높이에 대한 비율(0-1)로 나타냅니다.

### bottom

`pydantic-field`

시그니처 복사

<pre tabIndex="0" aria-label="SDK signature">
  \`bottom

  :

  &#x20;

  [float](https://docs.python.org/3/builtins/functions.html#float)\`
</pre>

box의 아래쪽 변이며, 이미지 위쪽 끝에서부터 잰 거리를 이미지 높이에 대한 비율(0-1)로 나타냅니다.

### left

`pydantic-field`

시그니처 복사

<pre tabIndex="0" aria-label="SDK signature">
  \`left

  :

  &#x20;

  [float](https://docs.python.org/3/builtins/functions.html#float)\`
</pre>

box의 왼쪽 변이며, 이미지 왼쪽 끝에서부터 잰 거리를 이미지 너비에 대한 비율(0-1)로 나타냅니다.

### right

`pydantic-field`

시그니처 복사

<pre tabIndex="0" aria-label="SDK signature">
  \`right

  :

  &#x20;

  [float](https://docs.python.org/3/builtins/functions.html#float)\`
</pre>

box의 오른쪽 변이며, 이미지 왼쪽 끝에서부터 잰 거리를 이미지 너비에 대한 비율(0-1)로 나타냅니다.

### model\_config

`class-attribute` `instance-attribute`

```python
model_config = ConfigDict(
    extra="ignore", frozen=True, strict=True
)
```

## typesafe\_sdk.Answer

`module-attribute`

시그니처 복사
<pre tabIndex="0" aria-label="SDK signature">`Answer: [TypeAlias](https://docs.python.org/3/library/typing.html#typing.TypeAlias) = [Annotated](https://docs.python.org/3/library/typing.html#typing.Annotated)\[
&#x20;   [NoulAnswer](responses.md#typesafe_sdk.NoulAnswer)
&#x20;   | [ChoiceAnswer](responses.md#typesafe_sdk.ChoiceAnswer)
&#x20;   | [ScoreAnswer](responses.md#typesafe_sdk.ScoreAnswer)
&#x20;   | [BoxAnswer](responses.md#typesafe_sdk.BoxAnswer),
&#x20;   Field(discriminator="type"),
]`</pre>

질문 하나에 대한 답변이며, `type`으로 구분됩니다.

## 사용 가능한 모델

## typesafe\_sdk.ListModelsResponse

`pydantic-model`

Bases(기반 클래스): `Response`

계정에서 사용할 수 있는 모델입니다.

> [!NOTE]
>
> **JSON schema 보기:**
>
> > [accordion] 세부 정보
> >
> > ```json theme={null}
> > {
> >   "$defs": {
> >     "ModelMetadata": {
> >       "description": "Metadata describing a single available model.",
> >       "properties": {
> >         "name": {
> >           "title": "Name",
> >           "type": "string"
> >         },
> >         "description": {
> >           "title": "Description",
> >           "type": "string"
> >         },
> >         "release_date": {
> >           "title": "Release Date",
> >           "type": "string"
> >         }
> >       },
> >       "required": [
> >         "name",
> >         "description",
> >         "release_date"
> >       ],
> >       "title": "ModelMetadata",
> >       "type": "object"
> >     }
> >   },
> >   "description": "The models available to the account.",
> >   "properties": {
> >     "models": {
> >       "items": {
> >         "$ref": "#/$defs/ModelMetadata"
> >       },
> >       "title": "Models",
> >       "type": "array"
> >     }
> >   },
> >   "required": [
> >     "models"
> >   ],
> >   "title": "ListModelsResponse",
> >   "type": "object"
> > }
> > ```

Fields(필드):

* `models` (`tuple[ModelMetadata, ...]`)

### request\_id

`cached` `property`

시그니처 복사

<pre tabIndex="0" aria-label="SDK signature">
  \`request\_id

  :

  &#x20;

  [str](https://docs.python.org/3/builtins/stdtypes.html#str)\`
</pre>

`x-typesafe-request-id` 응답 헤더입니다.

### raw\_http\_response

`property`

```python
raw_http_response: httpx2.Response
```

내부의 `httpx2.Response` 객체이며, 상태 코드, 헤더, 본문을 확인할 수 있습니다.

### model\_config

`class-attribute` `instance-attribute`

```python
model_config = ConfigDict(
    extra="ignore", frozen=True, strict=True
)
```

### models

`pydantic-field`

시그니처 복사

<pre tabIndex="0" aria-label="SDK signature">
  \`models

  :

  &#x20;

  [tuple](https://docs.python.org/3/builtins/stdtypes.html#tuple)

  \[

  [ModelMetadata](responses.md#typesafe_sdk.ModelMetadata)

  ,

  &#x20;

  ...

  ]\`
</pre>

사용 가능한 모델 목록입니다.

## typesafe\_sdk.ModelMetadata

`pydantic-model`

Bases(기반 클래스): `Schema`

사용 가능한 모델 하나를 설명하는 메타데이터입니다.

> [!NOTE]
>
> **JSON schema 보기:**
>
> > [accordion] 세부 정보
> >
> > ```json theme={null}
> > {
> >   "description": "Metadata describing a single available model.",
> >   "properties": {
> >     "name": {
> >       "title": "Name",
> >       "type": "string"
> >     },
> >     "description": {
> >       "title": "Description",
> >       "type": "string"
> >     },
> >     "release_date": {
> >       "title": "Release Date",
> >       "type": "string"
> >     }
> >   },
> >   "required": [
> >     "name",
> >     "description",
> >     "release_date"
> >   ],
> >   "title": "ModelMetadata",
> >   "type": "object"
> > }
> > ```

Fields(필드):

* `name` (`str`)
* `description` (`str`)
* `release_date` (`str`)

### name

`pydantic-field`

시그니처 복사

<pre tabIndex="0" aria-label="SDK signature">
  \`name

  :

  &#x20;

  [str](https://docs.python.org/3/builtins/stdtypes.html#str)\`
</pre>

요청의 model 필드에 넣을 수 있는 모델 이름 또는 별칭입니다.

### description

`pydantic-field`

시그니처 복사

<pre tabIndex="0" aria-label="SDK signature">
  \`description

  :

  &#x20;

  [str](https://docs.python.org/3/builtins/stdtypes.html#str)\`
</pre>

모델과 그 기능을 사람이 읽기 쉽게 풀어 쓴 설명입니다.

### release\_date

`pydantic-field`

시그니처 복사

<pre tabIndex="0" aria-label="SDK signature">
  \`release\_date

  :

  &#x20;

  [str](https://docs.python.org/3/builtins/stdtypes.html#str)\`
</pre>

모델 출시일이며, YYYY-MM-DD 형식입니다.


이 문서는 개발자 문서 플랫폼인 [Mintlify](https://mintlify.com)로 제작하고 호스팅합니다.
