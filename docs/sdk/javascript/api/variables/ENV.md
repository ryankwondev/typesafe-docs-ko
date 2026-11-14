# Variable: ENV

```ts
const ENV: object;
```

클라이언트 설정에 쓰는 환경 변수 이름입니다. 명시적으로 넘긴 옵션이 우선합니다.

## Type Declaration

### apiKey

```ts
readonly apiKey: "TYPESAFE_API_KEY" = "TYPESAFE_API_KEY";
```

필수 API 키입니다. `apiKey`를 생략했을 때 사용됩니다.

### baseURL

```ts
readonly baseURL: "TYPESAFE_BASE_URL" = "TYPESAFE_BASE_URL";
```

API 루트입니다. 기본값은 `https://api.typesafe.ai`입니다.

### defaultModel

```ts
readonly defaultModel: "TYPESAFE_DEFAULT_MODEL" = "TYPESAFE_DEFAULT_MODEL";
```

기본 모델 이름입니다. 기본값은 `jev-latest`입니다.

### logLevel

```ts
readonly logLevel: "TYPESAFE_LOG_LEVEL" = "TYPESAFE_LOG_LEVEL";
```

로그 레벨입니다. 기본값은 `warn`입니다.


이 문서는 개발자 문서 플랫폼인 [Mintlify](https://mintlify.com)로 제작하고 호스팅합니다.
