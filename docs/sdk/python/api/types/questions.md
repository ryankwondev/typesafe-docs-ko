# 질문

> state(상태)를 전달하고, 객체나 딕셔너리로 예/아니요 질문, choice 질문, score 질문을 정의합니다.

## State

`state`는 질문의 대상이 되는 텍스트 또는 JSON 객체입니다. `None`일 수는 없지만, 객체 안의 값은 `None`이어도 됩니다.

## 질문 객체

`Noul`, `Choice`, `Score`, `Box`를 사용하면 이름 있는 인자로 질문을 정의할 수 있습니다.

## typesafe\_sdk.NoulCriteria

Bases(기반 클래스): `TypedDict`

예와 아니요 각 결과에 대한 설명이며, 선택 사항입니다.

자세한 내용은 [noul primitive(프리미티브: 기본 구성 요소)](https://docs.typesafe.ai/go/noul.md)를 참고하세요.

### true

`instance-attribute`

시그니처 복사

<pre tabIndex="0" aria-label="SDK signature">
  \`true

  :

  &#x20;

  [Content](common.md#typesafe_sdk.Content)

  &#x20;

  |

  &#x20;

  None\`
</pre>

예에 해당하는 결과의 설명이며, 텍스트, asset(첨부 자산), JSON 객체, 배열 중 하나로 작성합니다. `None`이면 설명 없이 둡니다.

### false

`instance-attribute`

시그니처 복사

<pre tabIndex="0" aria-label="SDK signature">
  \`false

  :

  &#x20;

  [Content](common.md#typesafe_sdk.Content)

  &#x20;

  |

  &#x20;

  None\`
</pre>

아니요에 해당하는 결과의 설명이며, 텍스트, asset, JSON 객체, 배열 중 하나로 작성합니다. `None`이면 설명 없이 둡니다.

## typesafe\_sdk.Noul

`pydantic-model`

> [tab] 구현
>
> 시그니처 복사
>
> <pre tabIndex="0" aria-label="SDK signature">
>   \`Noul
>
>   (
>
>   \*\*
>
>   data
>
>   :
>
>   &#x20;
>
>   [Any](https://docs.python.org/3/library/typing.html#typing.Any)
>
>   )\`
> </pre>

> [tab] 오버로드 1
>
> 시그니처 복사
> <pre tabIndex="0" aria-label="SDK signature">`Noul(
> &#x20;   \*,
> &#x20;   type: [Literal](https://docs.python.org/3/library/typing.html#typing.Literal)\["noul"] = ...,
> &#x20;   instructions: [Content](common.md#typesafe_sdk.Content),
> &#x20;   criteria: [NoulCriteria](questions.md#typesafe_sdk.NoulCriteria) | None = ...,
> )`</pre>

> [tab] 오버로드 2
>
> 시그니처 복사
> <pre tabIndex="0" aria-label="SDK signature">`Noul(
> &#x20;   \*,
> &#x20;   type: [Literal](https://docs.python.org/3/library/typing.html#typing.Literal)\["noul"] = ...,
> &#x20;   instructions: None = ...,
> &#x20;   criteria: \_DescribedTrue | \_DescribedFalse,
> )`</pre>

Bases(기반 클래스): `_Question`, `wire.NoulQuestion`

예/아니요 질문이며, 각 결과에 대한 설명을 선택적으로 붙일 수 있습니다.

자세한 내용은 [noul primitive](https://docs.typesafe.ai/go/noul.md)를 참고하세요.

> [!NOTE]
>
> **JSON schema 보기:**
>
> > [accordion] 세부 정보
> >
> > ```json theme={null}
> > {
> >   "$defs": {
> >     "Content": {
> >       "anyOf": [
> >         {
> >           "type": "string"
> >         },
> >         {
> >           "additionalProperties": {
> >             "anyOf": [
> >               {
> >                 "$ref": "#/$defs/ContentValue"
> >               },
> >               {
> >                 "type": "null"
> >               }
> >             ]
> >           },
> >           "type": "object"
> >         },
> >         {
> >           "items": {
> >             "anyOf": [
> >               {
> >                 "$ref": "#/$defs/ContentValue"
> >               },
> >               {
> >                 "type": "null"
> >               }
> >             ]
> >           },
> >           "type": "array"
> >         }
> >       ]
> >     },
> >     "ContentValue": {
> >       "anyOf": [
> >         {
> >           "type": "string"
> >         },
> >         {
> >           "type": "integer"
> >         },
> >         {
> >           "type": "number"
> >         },
> >         {
> >           "type": "boolean"
> >         },
> >         {
> >           "items": {
> >             "anyOf": [
> >               {
> >                 "$ref": "#/$defs/ContentValue"
> >               },
> >               {
> >                 "type": "null"
> >               }
> >             ]
> >           },
> >           "type": "array"
> >         },
> >         {
> >           "additionalProperties": {
> >             "anyOf": [
> >               {
> >                 "$ref": "#/$defs/ContentValue"
> >               },
> >               {
> >                 "type": "null"
> >               }
> >             ]
> >           },
> >           "type": "object"
> >         }
> >       ]
> >     },
> >     "NoulCriteria": {
> >       "additionalProperties": false,
> >       "description": "Optional descriptions of the yes and no outcomes.\n\nSee the [noul primitive](https://docs.typesafe.ai/go/noul) for details.",
> >       "properties": {
> >         "true": {
> >           "anyOf": [
> >             {
> >               "$ref": "#/$defs/Content"
> >             },
> >             {
> >               "type": "null"
> >             }
> >           ]
> >         },
> >         "false": {
> >           "anyOf": [
> >             {
> >               "$ref": "#/$defs/Content"
> >             },
> >             {
> >               "type": "null"
> >             }
> >           ]
> >         }
> >       },
> >       "title": "NoulCriteria",
> >       "type": "object"
> >     }
> >   },
> >   "additionalProperties": false,
> >   "description": "A yes/no question with optional descriptions for either outcome.\n\nSee the [noul primitive](https://docs.typesafe.ai/go/noul) for details.",
> >   "properties": {
> >     "type": {
> >       "const": "noul",
> >       "default": "noul",
> >       "title": "Type",
> >       "type": "string"
> >     },
> >     "instructions": {
> >       "anyOf": [
> >         {
> >           "$ref": "#/$defs/Content"
> >         },
> >         {
> >           "type": "null"
> >         }
> >       ],
> >       "default": null
> >     },
> >     "criteria": {
> >       "anyOf": [
> >         {
> >           "$ref": "#/$defs/NoulCriteria"
> >         },
> >         {
> >           "type": "null"
> >         }
> >       ],
> >       "default": null
> >     }
> >   },
> >   "title": "Noul",
> >   "type": "object"
> > }
> > ```

Fields(필드):

* `type` (`Literal['noul']`)
* `instructions` (`Content | None`)
* `criteria` (`NoulCriteria | None`)

### instructions

`pydantic-field`

시그니처 복사

<pre tabIndex="0" aria-label="SDK signature">
  \`instructions

  :

  &#x20;

  [Content](common.md#typesafe_sdk.Content)

  &#x20;

  |

  &#x20;

  None

  &#x20;

  \=

  &#x20;

  None\`
</pre>

물어볼 질문이며, 텍스트, asset, JSON 객체, 배열 중 하나로 작성합니다. `criteria`가 결과를 설명하고 있으면 생략할 수 있습니다.

### criteria

`pydantic-field`

시그니처 복사

<pre tabIndex="0" aria-label="SDK signature">
  \`criteria

  :

  &#x20;

  [NoulCriteria](questions.md#typesafe_sdk.NoulCriteria)

  &#x20;

  |

  &#x20;

  None

  &#x20;

  \=

  &#x20;

  None\`
</pre>

예와 아니요 각 결과에 대한 설명이며, 선택 사항입니다. `instructions`가 `None`이면 반드시 지정해야 합니다.

## typesafe\_sdk.Choice

`pydantic-model`

Bases(기반 클래스): `_Question`, `wire.ChoiceQuestion`

이름이 붙은 선택지 가운데 하나를 고르는 질문입니다.

자세한 내용은 [choice primitive](https://docs.typesafe.ai/go/choice.md)를 참고하세요.

> [!NOTE]
>
> **JSON schema 보기:**
>
> > [accordion] 세부 정보
> >
> > ```json theme={null}
> > {
> >   "$defs": {
> >     "Content": {
> >       "anyOf": [
> >         {
> >           "type": "string"
> >         },
> >         {
> >           "additionalProperties": {
> >             "anyOf": [
> >               {
> >                 "$ref": "#/$defs/ContentValue"
> >               },
> >               {
> >                 "type": "null"
> >               }
> >             ]
> >           },
> >           "type": "object"
> >         },
> >         {
> >           "items": {
> >             "anyOf": [
> >               {
> >                 "$ref": "#/$defs/ContentValue"
> >               },
> >               {
> >                 "type": "null"
> >               }
> >             ]
> >           },
> >           "type": "array"
> >         }
> >       ]
> >     },
> >     "ContentValue": {
> >       "anyOf": [
> >         {
> >           "type": "string"
> >         },
> >         {
> >           "type": "integer"
> >         },
> >         {
> >           "type": "number"
> >         },
> >         {
> >           "type": "boolean"
> >         },
> >         {
> >           "items": {
> >             "anyOf": [
> >               {
> >                 "$ref": "#/$defs/ContentValue"
> >               },
> >               {
> >                 "type": "null"
> >               }
> >             ]
> >           },
> >           "type": "array"
> >         },
> >         {
> >           "additionalProperties": {
> >             "anyOf": [
> >               {
> >                 "$ref": "#/$defs/ContentValue"
> >               },
> >               {
> >                 "type": "null"
> >               }
> >             ]
> >           },
> >           "type": "object"
> >         }
> >       ]
> >     }
> >   },
> >   "additionalProperties": false,
> >   "description": "A question that selects between named alternatives.\n\nSee the [choice primitive](https://docs.typesafe.ai/go/choice) for details.",
> >   "properties": {
> >     "type": {
> >       "const": "choice",
> >       "default": "choice",
> >       "title": "Type",
> >       "type": "string"
> >     },
> >     "instructions": {
> >       "anyOf": [
> >         {
> >           "$ref": "#/$defs/Content"
> >         },
> >         {
> >           "type": "null"
> >         }
> >       ],
> >       "default": null
> >     },
> >     "criteria": {
> >       "additionalProperties": {
> >         "anyOf": [
> >           {
> >             "$ref": "#/$defs/Content"
> >           },
> >           {
> >             "type": "null"
> >           }
> >         ]
> >       },
> >       "title": "Criteria",
> >       "type": "object"
> >     }
> >   },
> >   "required": [
> >     "criteria"
> >   ],
> >   "title": "Choice",
> >   "type": "object"
> > }
> > ```

Fields(필드):

* `type` (`Literal['choice']`)
* `criteria` (`Mapping[str, Content | None]`)
* `instructions` (`Content | None`)

### criteria

`pydantic-field`

시그니처 복사

<pre tabIndex="0" aria-label="SDK signature">
  \`criteria

  :

  &#x20;

  [Mapping](https://docs.python.org/3/library/collections.abc.html#collections.abc.Mapping)

  \[

  [str](https://docs.python.org/3/builtins/stdtypes.html#str)

  ,

  &#x20;

  [Content](common.md#typesafe_sdk.Content)

  &#x20;

  |

  &#x20;

  None

  ]\`
</pre>

label(레이블)을 설명에 대응시키는 비어 있지 않은 mapping(매핑)입니다. 설명은 텍스트, asset, 객체, 배열 중 하나로 작성하고, 설명 없는 label에는 `None`을 씁니다.

### instructions

`pydantic-field`

시그니처 복사

<pre tabIndex="0" aria-label="SDK signature">
  \`instructions

  :

  &#x20;

  [Content](common.md#typesafe_sdk.Content)

  &#x20;

  |

  &#x20;

  None

  &#x20;

  \=

  &#x20;

  None\`
</pre>

물어볼 질문이며, 텍스트, asset, JSON 객체, 배열 중 하나로 작성합니다. 선택 사항입니다.

## typesafe\_sdk.Score

`pydantic-model`

Bases(기반 클래스): `_Question`, `wire.ScoreQuestion`

순서가 있는 rubric(루브릭: 채점 기준표)에 따라 점수를 매기는 질문입니다.

자세한 내용은 [score primitive](https://docs.typesafe.ai/go/score.md)를 참고하세요.

> [!NOTE]
>
> **JSON schema 보기:**
>
> > [accordion] 세부 정보
> >
> > ```json theme={null}
> > {
> >   "$defs": {
> >     "Content": {
> >       "anyOf": [
> >         {
> >           "type": "string"
> >         },
> >         {
> >           "additionalProperties": {
> >             "anyOf": [
> >               {
> >                 "$ref": "#/$defs/ContentValue"
> >               },
> >               {
> >                 "type": "null"
> >               }
> >             ]
> >           },
> >           "type": "object"
> >         },
> >         {
> >           "items": {
> >             "anyOf": [
> >               {
> >                 "$ref": "#/$defs/ContentValue"
> >               },
> >               {
> >                 "type": "null"
> >               }
> >             ]
> >           },
> >           "type": "array"
> >         }
> >       ]
> >     },
> >     "ContentValue": {
> >       "anyOf": [
> >         {
> >           "type": "string"
> >         },
> >         {
> >           "type": "integer"
> >         },
> >         {
> >           "type": "number"
> >         },
> >         {
> >           "type": "boolean"
> >         },
> >         {
> >           "items": {
> >             "anyOf": [
> >               {
> >                 "$ref": "#/$defs/ContentValue"
> >               },
> >               {
> >                 "type": "null"
> >               }
> >             ]
> >           },
> >           "type": "array"
> >         },
> >         {
> >           "additionalProperties": {
> >             "anyOf": [
> >               {
> >                 "$ref": "#/$defs/ContentValue"
> >               },
> >               {
> >                 "type": "null"
> >               }
> >             ]
> >           },
> >           "type": "object"
> >         }
> >       ]
> >     }
> >   },
> >   "additionalProperties": false,
> >   "description": "A question that assigns a score using an ordered rubric.\n\nSee the [score primitive](https://docs.typesafe.ai/go/score) for details.",
> >   "properties": {
> >     "type": {
> >       "const": "score",
> >       "default": "score",
> >       "title": "Type",
> >       "type": "string"
> >     },
> >     "instructions": {
> >       "anyOf": [
> >         {
> >           "$ref": "#/$defs/Content"
> >         },
> >         {
> >           "type": "null"
> >         }
> >       ],
> >       "default": null
> >     },
> >     "criteria": {
> >       "items": {
> >         "$ref": "#/$defs/Content"
> >       },
> >       "minItems": 1,
> >       "title": "Criteria",
> >       "type": "array"
> >     }
> >   },
> >   "required": [
> >     "criteria"
> >   ],
> >   "title": "Score",
> >   "type": "object"
> > }
> > ```

Fields(필드):

* `type` (`Literal['score']`)
* `criteria` (`Sequence[Content]`)
* `instructions` (`Content | None`)

### criteria

`pydantic-field`

시그니처 복사

<pre tabIndex="0" aria-label="SDK signature">
  \`criteria

  :

  &#x20;

  [Sequence](https://docs.python.org/3/library/collections.abc.html#collections.abc.Sequence)

  \[

  [Content](common.md#typesafe_sdk.Content)

  ]\`
</pre>

순서가 있는 비어 있지 않은 설명 목록입니다. 0점부터 시작해 점수마다 하나씩, 텍스트, asset, 객체, 배열 중 하나로 작성합니다.

### instructions

`pydantic-field`

시그니처 복사

<pre tabIndex="0" aria-label="SDK signature">
  \`instructions

  :

  &#x20;

  [Content](common.md#typesafe_sdk.Content)

  &#x20;

  |

  &#x20;

  None

  &#x20;

  \=

  &#x20;

  None\`
</pre>

물어볼 질문이며, 텍스트, asset, JSON 객체, 배열 중 하나로 작성합니다. 선택 사항입니다.

## typesafe\_sdk.Box

`pydantic-model`

Bases(기반 클래스): `_Question`, `wire.BoxQuestion`

instructions(지시문)가 설명하는 대상을 요청의 이미지에서 찾아 위치를 알려 주는 질문입니다.

Examples(예시):

```python
from pathlib import Path

from typesafe_sdk import Box, BoxAnswer, Image, TypeSafeClient


def locate_dog(client: TypeSafeClient, photo: Path) -> BoxAnswer:
    result = client.system_one(
        state=Image(photo),
        questions={"dog": Box(instructions="The dog")},
        model="jev-preview",
    )
    return result.boxes["dog"]
```

> [!NOTE]
>
> **JSON schema 보기:**
>
> > [accordion] 세부 정보
> >
> > ````json theme={null}
> > {
> >   "$defs": {
> >     "Content": {
> >       "anyOf": [
> >         {
> >           "type": "string"
> >         },
> >         {
> >           "additionalProperties": {
> >             "anyOf": [
> >               {
> >                 "$ref": "#/$defs/ContentValue"
> >               },
> >               {
> >                 "type": "null"
> >               }
> >             ]
> >           },
> >           "type": "object"
> >         },
> >         {
> >           "items": {
> >             "anyOf": [
> >               {
> >                 "$ref": "#/$defs/ContentValue"
> >               },
> >               {
> >                 "type": "null"
> >               }
> >             ]
> >           },
> >           "type": "array"
> >         }
> >       ]
> >     },
> >     "ContentValue": {
> >       "anyOf": [
> >         {
> >           "type": "string"
> >         },
> >         {
> >           "type": "integer"
> >         },
> >         {
> >           "type": "number"
> >         },
> >         {
> >           "type": "boolean"
> >         },
> >         {
> >           "items": {
> >             "anyOf": [
> >               {
> >                 "$ref": "#/$defs/ContentValue"
> >               },
> >               {
> >                 "type": "null"
> >               }
> >             ]
> >           },
> >           "type": "array"
> >         },
> >         {
> >           "additionalProperties": {
> >             "anyOf": [
> >               {
> >                 "$ref": "#/$defs/ContentValue"
> >               },
> >               {
> >                 "type": "null"
> >               }
> >             ]
> >           },
> >           "type": "object"
> >         }
> >       ]
> >     }
> >   },
> >   "additionalProperties": false,
> >   "description": "A question that locates what its instructions describe in the request's image.\n\nExamples:\n    ```python\n    from pathlib import Path\n\n    from typesafe_sdk import Box, BoxAnswer, Image, TypeSafeClient\n\n\n    def locate_dog(client: TypeSafeClient, photo: Path) -> BoxAnswer:\n        result = client.system_one(\n            state=Image(photo),\n            questions={\"dog\": Box(instructions=\"The dog\")},\n            model=\"jev-preview\",\n        )\n        return result.boxes[\"dog\"]\n    ```",
> >   "properties": {
> >     "type": {
> >       "const": "box",
> >       "default": "box",
> >       "title": "Type",
> >       "type": "string"
> >     },
> >     "instructions": {
> >       "anyOf": [
> >         {
> >           "$ref": "#/$defs/Content"
> >         },
> >         {
> >           "type": "null"
> >         }
> >       ],
> >       "default": null
> >     }
> >   },
> >   "title": "Box",
> >   "type": "object"
> > }
> > ````

Fields(필드):

* `type` (`Literal['box']`)
* `instructions` (`Content | None`)

### instructions

`pydantic-field`

시그니처 복사

<pre tabIndex="0" aria-label="SDK signature">
  \`instructions

  :

  &#x20;

  [Content](common.md#typesafe_sdk.Content)

  &#x20;

  |

  &#x20;

  None

  &#x20;

  \=

  &#x20;

  None\`
</pre>

요청의 이미지에서 찾을 대상이며, 텍스트, asset, JSON 객체, 배열 중 하나로 작성합니다. 선택 사항입니다.

## typesafe\_sdk.Question

`module-attribute`

시그니처 복사
<pre tabIndex="0" aria-label="SDK signature">`Question: [TypeAlias](https://docs.python.org/3/library/typing.html#typing.TypeAlias) = (
&#x20;   [Noul](questions.md#typesafe_sdk.Noul) | [Choice](questions.md#typesafe_sdk.Choice) | [Score](questions.md#typesafe_sdk.Score) | [Box](questions.md#typesafe_sdk.Box) | [QuestionModel](questions.md#typesafe_sdk.QuestionModel)
)`</pre>

질문 객체 또는 질문 딕셔너리입니다.

## typesafe\_sdk.Questions

`module-attribute`

시그니처 복사

<pre tabIndex="0" aria-label="SDK signature">
  \`Questions

  :

  &#x20;

  [TypeAlias](https://docs.python.org/3/library/typing.html#typing.TypeAlias)

  &#x20;

  \=

  &#x20;

  [Mapping](https://docs.python.org/3/library/collections.abc.html#collections.abc.Mapping)

  \[

  [str](https://docs.python.org/3/builtins/stdtypes.html#str)

  ,

  &#x20;

  [Question](questions.md#typesafe_sdk.Question)

  ]\`
</pre>

질문 입력을 이름으로 묶은 mapping입니다. 이 이름은 각 질문의 답을 식별하는 키로 쓰입니다.

## 질문 딕셔너리

질문 딕셔너리에는 `type` 키가 들어가며, 값은 `"noul"`, `"choice"`, `"score"`, `"box"` 중 하나입니다. 한 요청 안에서 딕셔너리와 질문 객체를 섞어 쓸 수 있습니다.

## typesafe\_sdk.NoulModel

Bases(기반 클래스): `TypedDict`

`type="noul"`인 예/아니요 질문 딕셔너리입니다.

자세한 내용은 [noul primitive](https://docs.typesafe.ai/go/noul.md)를 참고하세요.

### type

`instance-attribute`

시그니처 복사

<pre tabIndex="0" aria-label="SDK signature">
  \`type

  :

  &#x20;

  [Literal](https://docs.python.org/3/library/typing.html#typing.Literal)

  \[

  'noul'

  ]\`
</pre>

### instructions

`instance-attribute`

시그니처 복사

<pre tabIndex="0" aria-label="SDK signature">
  \`instructions

  :

  &#x20;

  [NotRequired](https://docs.python.org/3/library/typing.html#typing.NotRequired)

  \[

  [Content](common.md#typesafe_sdk.Content)

  &#x20;

  |

  &#x20;

  None

  ]\`
</pre>

물어볼 질문이며, 텍스트, asset, JSON 객체, 배열 중 하나로 작성합니다. `criteria`가 결과를 설명하고 있으면 생략할 수 있습니다.

### criteria

`instance-attribute`

시그니처 복사

<pre tabIndex="0" aria-label="SDK signature">
  \`criteria

  :

  &#x20;

  [NotRequired](https://docs.python.org/3/library/typing.html#typing.NotRequired)

  \[

  [NoulCriteria](questions.md#typesafe_sdk.NoulCriteria)

  &#x20;

  |

  &#x20;

  None

  ]\`
</pre>

예와 아니요 각 결과에 대한 설명이며, 선택 사항입니다. `instructions`가 `None`이면 반드시 지정해야 합니다.

## typesafe\_sdk.ChoiceModel

Bases(기반 클래스): `TypedDict`

`type="choice"`인 choice 질문 딕셔너리입니다.

자세한 내용은 [choice primitive](https://docs.typesafe.ai/go/choice.md)를 참고하세요.

### type

`instance-attribute`

시그니처 복사

<pre tabIndex="0" aria-label="SDK signature">
  \`type

  :

  &#x20;

  [Literal](https://docs.python.org/3/library/typing.html#typing.Literal)

  \[

  'choice'

  ]\`
</pre>

### instructions

`instance-attribute`

시그니처 복사

<pre tabIndex="0" aria-label="SDK signature">
  \`instructions

  :

  &#x20;

  [NotRequired](https://docs.python.org/3/library/typing.html#typing.NotRequired)

  \[

  [Content](common.md#typesafe_sdk.Content)

  &#x20;

  |

  &#x20;

  None

  ]\`
</pre>

물어볼 질문이며, 텍스트, asset, JSON 객체, 배열 중 하나로 작성합니다. 선택 사항입니다.

### criteria

`instance-attribute`

시그니처 복사

<pre tabIndex="0" aria-label="SDK signature">
  \`criteria

  :

  &#x20;

  [Mapping](https://docs.python.org/3/library/collections.abc.html#collections.abc.Mapping)

  \[

  [str](https://docs.python.org/3/builtins/stdtypes.html#str)

  ,

  &#x20;

  [Content](common.md#typesafe_sdk.Content)

  &#x20;

  |

  &#x20;

  None

  ]\`
</pre>

label을 설명에 대응시키는 비어 있지 않은 mapping입니다. 설명은 텍스트, asset, 객체, 배열 중 하나로 작성하고, 설명 없는 label에는 `None`을 씁니다.

## typesafe\_sdk.ScoreModel

Bases(기반 클래스): `TypedDict`

`type="score"`인 score 질문 딕셔너리입니다.

자세한 내용은 [score primitive](https://docs.typesafe.ai/go/score.md)를 참고하세요.

### type

`instance-attribute`

시그니처 복사

<pre tabIndex="0" aria-label="SDK signature">
  \`type

  :

  &#x20;

  [Literal](https://docs.python.org/3/library/typing.html#typing.Literal)

  \[

  'score'

  ]\`
</pre>

### instructions

`instance-attribute`

시그니처 복사

<pre tabIndex="0" aria-label="SDK signature">
  \`instructions

  :

  &#x20;

  [NotRequired](https://docs.python.org/3/library/typing.html#typing.NotRequired)

  \[

  [Content](common.md#typesafe_sdk.Content)

  &#x20;

  |

  &#x20;

  None

  ]\`
</pre>

물어볼 질문이며, 텍스트, asset, JSON 객체, 배열 중 하나로 작성합니다. 선택 사항입니다.

### criteria

`instance-attribute`

시그니처 복사

<pre tabIndex="0" aria-label="SDK signature">
  \`criteria

  :

  &#x20;

  [Sequence](https://docs.python.org/3/library/collections.abc.html#collections.abc.Sequence)

  \[

  [Content](common.md#typesafe_sdk.Content)

  ]\`
</pre>

순서가 있는 비어 있지 않은 설명 목록입니다. 0점부터 시작해 점수마다 하나씩, 텍스트, asset, 객체, 배열 중 하나로 작성합니다.

## typesafe\_sdk.BoxModel

Bases(기반 클래스): `TypedDict`

`type="box"`인 box 질문 딕셔너리입니다.

### type

`instance-attribute`

시그니처 복사

<pre tabIndex="0" aria-label="SDK signature">
  \`type

  :

  &#x20;

  [Literal](https://docs.python.org/3/library/typing.html#typing.Literal)

  \[

  'box'

  ]\`
</pre>

### instructions

`instance-attribute`

시그니처 복사

<pre tabIndex="0" aria-label="SDK signature">
  \`instructions

  :

  &#x20;

  [NotRequired](https://docs.python.org/3/library/typing.html#typing.NotRequired)

  \[

  [Content](common.md#typesafe_sdk.Content)

  &#x20;

  |

  &#x20;

  None

  ]\`
</pre>

요청의 이미지에서 찾을 대상이며, 텍스트, asset, JSON 객체, 배열 중 하나로 작성합니다. 선택 사항입니다.

## typesafe\_sdk.QuestionModel

`module-attribute`

시그니처 복사
<pre tabIndex="0" aria-label="SDK signature">`QuestionModel: [TypeAlias](https://docs.python.org/3/library/typing.html#typing.TypeAlias) = (
&#x20;   [NoulModel](questions.md#typesafe_sdk.NoulModel)
&#x20;   | [ChoiceModel](questions.md#typesafe_sdk.ChoiceModel)
&#x20;   | [ScoreModel](questions.md#typesafe_sdk.ScoreModel)
&#x20;   | [BoxModel](questions.md#typesafe_sdk.BoxModel)
)`</pre>

`type` 키로 구분되는 질문 딕셔너리입니다.


이 문서는 개발자 문서 플랫폼인 [Mintlify](https://mintlify.com)로 제작하고 호스팅합니다.
