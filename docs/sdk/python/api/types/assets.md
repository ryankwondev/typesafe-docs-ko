# Assets

> System One 요청에 asset(첨부 자산)을 첨부하고 요청 내용에서 참조합니다.

> [!NOTE]
>
> **참고**
>
> API는 각 이미지의 바이트를 보고 형식을 감지합니다.

## typesafe\_sdk.Asset

`module-attribute`

시그니처 복사

<pre tabIndex="0" aria-label="SDK signature">
  \`Asset

  :

  &#x20;

  [TypeAlias](https://docs.python.org/3/library/typing.html#typing.TypeAlias)

  &#x20;

  \=

  &#x20;

  [Image](assets.md#typesafe_sdk.Image)\`
</pre>

요청 내용에서 참조하는 asset입니다. 현재는 이미지만 지원합니다.

## typesafe\_sdk.Image

`dataclass`

시그니처 복사

<pre tabIndex="0" aria-label="SDK signature">
  \`Image

  (

  data

  :

  &#x20;

  [Buffer](https://typing-extensions.readthedocs.io/en/latest/index.html#typing_extensions.Buffer)

  &#x20;

  |

  &#x20;

  [PathLike](https://docs.python.org/3/library/os.html#os.PathLike)

  \[

  [str](https://docs.python.org/3/builtins/stdtypes.html#str)

  ])\`
</pre>

이미지 파일의 바이트 또는 이미지 파일 경로로 만드는 이미지 asset입니다.

> [!TIP]
>
> **이미지 재사용**
>
> `Image`는 처음 사용할 때 한 번만 로드되며, 같은 `Image` 객체를 보내는 모든 요청에서 재사용됩니다.

> [!TIP]
>
> **`Buffer` 객체 다루기**
>
> `Image`는 `Buffer` 객체도 받습니다. `bytearray`, `memoryview`, `mmap`, NumPy 배열처럼 [buffer protocol(버퍼 프로토콜)](https://docs.python.org/3/c-api/buffer.html)로 메모리를 노출하는 객체라면 무엇이든 됩니다. `bytes`가 아닌 버퍼는 처음 사용할 때 한 번 메모리로 복사되므로, 그 뒤에는 원본을 변경하거나 닫아도 됩니다.

Examples(예시):

파일 경로로 만들기:

```python
from pathlib import Path

image = Image(Path("photos/product.png"))
```

메모리에 있는 바이트로 만들기:

```python
def from_bytes(data: bytes) -> Image:
    return Image(data)
```

업로드된 데이터처럼 base64 텍스트로 만들기:

```python
def from_upload(data_base64: str) -> Image:
    return Image.from_base64(data_base64)
```

Pillow 이미지로 만들기(`pil` extra 필요):

```python
import PIL.Image

image = Image.from_pil(PIL.Image.new("RGB", (64, 64)), format="jpeg")
```

큰 버퍼의 일부를 먼저 `bytes`로 복사하지 않고 그대로 잘라서 만들기:

```python
def embedded_photo(container: bytes, offset: int, length: int) -> Image:
    return Image(memoryview(container)[offset : offset + length])
```

메모리 맵 파일로 만들기:

```python
import mmap


def map_photo(path: Path) -> Image:
    with path.open("rb") as file:
        return Image(mmap.mmap(file.fileno(), 0, access=mmap.ACCESS_READ))
```

### data

`instance-attribute`

시그니처 복사

<pre tabIndex="0" aria-label="SDK signature">
  \`data

  :

  &#x20;

  [Buffer](https://typing-extensions.readthedocs.io/en/latest/index.html#typing_extensions.Buffer)

  &#x20;

  |

  &#x20;

  [PathLike](https://docs.python.org/3/library/os.html#os.PathLike)

  \[

  [str](https://docs.python.org/3/builtins/stdtypes.html#str)

  ]\`
</pre>

이미지 내용입니다. 다음 중 하나로 표현합니다.

* 원시 바이트([`Buffer` protocol](https://docs.python.org/3/library/stdtypes.html#buffer-objects)을 지원하는 모든 객체)
* 파일 시스템에 있는 이미지를 가리키는 [path-like](https://docs.python.org/3/library/os.html#os.PathLike) 객체

### from\_base64

`classmethod`

시그니처 복사

<pre tabIndex="0" aria-label="SDK signature">
  \`from\_base64

  (

  data

  :

  &#x20;

  [str](https://docs.python.org/3/builtins/stdtypes.html#str)

  &#x20;

  |

  &#x20;

  [Buffer](https://typing-extensions.readthedocs.io/en/latest/index.html#typing_extensions.Buffer)

  )

  &#x20;

  ->

  &#x20;

  [Self](https://docs.python.org/3/library/typing.html#typing.Self)\`
</pre>

base64로 인코딩된 이미지 바이트에서 `Image` 인스턴스를 만듭니다.

Parameters(매개변수):

* **`data`** (`str | Buffer`) –

  base64로 인코딩된 이미지 파일 바이트이며, 텍스트나 ASCII 버퍼로 전달합니다. 줄바꿈된 base64의 개행 같은 공백 문자는 무시됩니다.

Raises(예외):

* `TypeSafeError` –

  데이터가 올바른 base64 텍스트가 아닙니다.

### from\_pil

`classmethod`

시그니처 복사
<pre tabIndex="0" aria-label="SDK signature">`from\_pil(
&#x20;   image: [PIL](https://pillow.readthedocs.io/en/stable/PIL.html#module-PIL).[Image](https://pillow.readthedocs.io/en/stable/reference/Image.html#module-PIL.Image).[Image](https://pillow.readthedocs.io/en/stable/reference/Image.html#PIL.Image.Image), format: [str](https://docs.python.org/3/builtins/stdtypes.html#str) = "jpeg"
) -> [Self](https://docs.python.org/3/library/typing.html#typing.Self)`</pre>

[Pillow](https://pillow.readthedocs.io/) 이미지에서 `Image` 인스턴스를 만듭니다.

> [!WARNING]
>
> **인코더 설정**
>
> 이미지는 Pillow의 기본 인코더 설정으로 저장되며, JPEG만 예외로 quality 95를 사용합니다. JPEG는 투명도나 팔레트를 저장할 수 없으므로 RGBA 또는 P 모드 이미지에는 `format="png"`를 전달하세요. 다른 설정이 필요하면 직접 인코딩하세요.

Parameters(매개변수):

* **`image`** (`PIL.Image.Image`) –

  인코딩할 Pillow 이미지입니다.
* **`format`** (`str`, 기본값: `'jpeg'` ) –

  저장할 이미지 형식이며, [Pillow에서 쓰는 이름](https://pillow.readthedocs.io/en/stable/handbook/image-file-formats.html)으로 지정합니다.

Raises(예외):

* `TypeSafeError` –

  Pillow가 이미지를 이 형식으로 인코딩할 수 없습니다.


이 문서는 개발자 문서 플랫폼인 [Mintlify](https://mintlify.com)로 제작하고 호스팅합니다.
