# 공통 타입

> TypeSafe API SDK에서 공통으로 쓰는 타입입니다.

## typesafe\_sdk.JSONValue

`module-attribute`

시그니처 복사
<pre tabIndex="0" aria-label="SDK signature">`JSONValue = [TypeAliasType](https://typing-extensions.readthedocs.io/en/latest/index.html#typing_extensions.TypeAliasType)(
&#x20;   "JSONValue",
&#x20;   "str | int | float | bool | Sequence\[JSONValue | None] | Mapping\[str, JSONValue | None]",
)`</pre>

JSON과 비슷한 형태의 값입니다. 중첩할 수 있고 `None`을 포함할 수 있습니다.

## typesafe\_sdk.JSONContent

`module-attribute`

시그니처 복사
<pre tabIndex="0" aria-label="SDK signature">`JSONContent = [TypeAliasType](https://typing-extensions.readthedocs.io/en/latest/index.html#typing_extensions.TypeAliasType)(
&#x20;   "JSONContent",
&#x20;   "str | Mapping\[str, JSONValue | None] | Sequence\[JSONValue | None]",
)`</pre>

일반 문자열이거나, [`JSONValue`](common.md#typesafe_sdk.JSONValue) 항목으로 이루어진 mapping(매핑) 또는 sequence(시퀀스)입니다.

## typesafe\_sdk.ContentValue

`module-attribute`

시그니처 복사
<pre tabIndex="0" aria-label="SDK signature">`ContentValue = [TypeAliasType](https://typing-extensions.readthedocs.io/en/latest/index.html#typing_extensions.TypeAliasType)(
&#x20;   "ContentValue",
&#x20;   "str | int | float | bool | Sequence\[ContentValue | None] | Mapping\[str, ContentValue | None] | Asset",
)`</pre>

JSON과 비슷한 형태의 값이며, [asset(첨부 자산)](assets.md#typesafe_sdk.Asset)도 담을 수 있습니다. 중첩할 수 있고 `None`을 포함할 수 있습니다.

## typesafe\_sdk.Content

`module-attribute`

시그니처 복사
<pre tabIndex="0" aria-label="SDK signature">`Content = [TypeAliasType](https://typing-extensions.readthedocs.io/en/latest/index.html#typing_extensions.TypeAliasType)(
&#x20;   "Content",
&#x20;   "str | Mapping\[str, ContentValue | None] | Sequence\[ContentValue | None] | Asset",
)`</pre>

텍스트, [asset](assets.md#typesafe_sdk.Asset), 또는 [`ContentValue`](common.md#typesafe_sdk.ContentValue) 항목으로 이루어진 mapping이나 sequence입니다.

각 asset은 요청의 assets에 담겨 전송되고, 원래 자리에는 해당 asset의 고유 참조가 들어갑니다.


이 문서는 개발자 문서 플랫폼인 [Mintlify](https://mintlify.com)로 제작하고 호스팅합니다.
