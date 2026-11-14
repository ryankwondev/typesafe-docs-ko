# Agent skill

> Claude Code, Codex를 비롯한 에이전트 환경에 바로 넣어 쓸 수 있는 skill입니다.

TypeSafe agent skill(에이전트 스킬)은 AI 코딩 에이전트에게 TypeSafe API에 관한 전체 맥락을 제공합니다. 질문 [유형](primitives.md), 아키텍처 [패턴](patterns.md), 요청을 구조화하는 모범 사례가 포함됩니다.

## 설치

> [tab] Claude Code
>
> 터미널에서 다음 두 명령을 실행하세요.
>
> ```bash theme={null}
> claude plugin marketplace add typesafe-ai/skills
> claude plugin install typesafe@typesafe-ai
> ```

> [tab] 다른 에이전트
>
> ```bash theme={null}
> npx skills add typesafe-ai/skills --skill typesafe-ai
> ```
>
> 안내가 나오면 사용하는 에이전트를 선택하세요. 기본적으로 프로젝트 로컬에 설치되며, 전역으로 설치하려면 `-g`를 추가하세요.

> [tab] 에이전트에 복사하기
>
> 다음 prompt(프롬프트)를 코딩 에이전트에 붙여 넣으세요.
>
> ```text wrap theme={null}
> Install the TypeSafe skill. If you're in Claude Code, run `claude plugin marketplace add typesafe-ai/skills`, then `claude plugin install typesafe@typesafe-ai`. If you're in another agent, run `npx skills add typesafe-ai/skills --skill typesafe-ai` and select your agent. Use one installation method. You can read the skill directly at https://github.com/typesafe-ai/skills/blob/main/skills/typesafe-ai/SKILL.md (raw: https://raw.githubusercontent.com/typesafe-ai/skills/main/skills/typesafe-ai/SKILL.md). Then use the TypeSafe skill when working on this project.
> ```

[GitHub의 SKILL.md](https://github.com/typesafe-ai/skills/blob/main/skills/typesafe-ai/SKILL.md)를 읽거나 [원본 Markdown](https://raw.githubusercontent.com/typesafe-ai/skills/main/skills/typesafe-ai/SKILL.md)을 직접 가져올 수도 있습니다. 수동으로 설치하려면 [skills/typesafe-ai 디렉토리](https://github.com/typesafe-ai/skills/tree/main/skills/typesafe-ai)를 에이전트의 skills 디렉토리에 복사하세요.

사본이 중복되지 않도록 설치 방법은 하나만 고르세요.

### 업데이트

Claude Code 플러그인이라면 다음을 실행하세요.

```bash
claude plugin marketplace update typesafe-ai
claude plugin update typesafe@typesafe-ai
```

업데이트를 불러오려면 Claude Code를 다시 시작하거나 `/reload-plugins`를 실행하세요. 자동 업데이트를 켜려면 `/plugin`을 열고 **Marketplaces → typesafe-ai → Enable auto-update**를 선택하세요.

skills.sh로 설치했다면 `npx skills update`를 실행하세요. 수동으로 복사했다면 skill 디렉토리 전체를 GitHub의 최신 버전으로 교체하세요.

## 예시 prompt

prompt에서 "use the TypeSafe skill"처럼 skill의 이름을 직접 언급하는 방법은 어떤 에이전트에서든 통하므로, 아래의 prompt는 모두 그렇게 합니다. Claude Code 플러그인을 쓴다면 `/typesafe:typesafe-ai`를 직접 호출할 수도 있습니다.

* 처음 써 보기에 좋은 것은 프로젝트의 어디에 TypeSafe를 쓰면 가장 좋을지 찾도록 도와주는 브레인스토밍 prompt입니다.

  ```text
  Using the TypeSafe skill, explore the project and find opportunities for using
  intelligent judgement to stand in for complex parsing or other fragile code.
  ```

* [API 키](https://console.typesafe.ai/keys)를 만들고, 저렴한 테스트 호출을 실행해 TypeSafe를 가장 잘 쓰는 방법을 찾아보도록 에이전트에게 권한을 줄 수도 있습니다.

  ```text
  Using the TypeSafe skill, run some experiments using the TypeSafe API key that I've
  exported to `TYPESAFE_API_KEY`. Propose changes based on the most promising results.
  ```

* 코드베이스에 있는 문제를 해결하는 [특정 cookbook(예제 모음)](cookbooks/consistency_noul_cookbook.md)을 에이전트에게 알려 주거나, [cookbook 목록](cookbooks.md)을 알려 주고 프로젝트에 있는 것과 비슷한 패턴이 있는지 물어보세요.

  ```text
  Using the TypeSafe skill, analyze my code and see if there are any applicable
  cookbooks (https://docs.typesafe.ai/cookbooks) that show how I could
  refactor my code to be less fragile or complex.
  ```

## 좋은 바이브 코딩 원칙

1. 위의 예시 prompt를 출발점으로 삼아 에이전트와 충분히 이야기를 나누세요.
2. 구현하기 전에 계획을 검토하고 말이 되는지 확인하세요.
3. 상수(질문과 threshold(임계값))는 검토하기 쉽도록 한곳에 모아 두세요. 에이전트는 질문을 잘 쓰지 못하므로 에이전트와 함께 고쳐 나가야 한다고 생각하세요.
4. 에이전트의 단언을 곧이곧대로 받아들이지 마세요. 에이전트가 자신의 가정을 검증하도록 유도하세요.

## 자주 생기는 문제

### 에이전트가 skill을 사용하지 않습니다

Claude Code 플러그인을 쓴다면 `/typesafe:typesafe-ai`를 호출하세요. 다른 에이전트에서는 "use the TypeSafe skill"이라고 요청하세요. 그래도 로드되지 않으면 설치 프로그램이 지금 쓰는 에이전트를 대상으로 설치했는지 확인한 다음 에이전트를 다시 시작하세요.

### Routing이 기대한 대로 동작하지 않습니다

질문과 threshold를 확인하세요. routing(라우팅)이 어긋난다면 threshold가 너무 높게 설정되어 false negative(거짓 음성)가 생기거나, 너무 낮게 설정되어 false positive(거짓 양성)가 생기는 것일 수 있습니다. 질문을 더 구체적으로 다듬어야 할 수도 있습니다.

### 모든 곳에 confidence threshold를 쓰고 있습니다

가장 좋은 선택지를 고르는 것만 중요하다면 답의 1위 선택지인 `choice`를 그대로 쓰고 threshold는 아예 걸지 마세요. 코드가 실행 여부를 결정해야 한다면 확률부터 보세요. Noul이라면 값 자체, Choice라면 `probabilities[choice]`입니다. Score의 위치 값이 갈라진 답을 가릴 수 있을 때는 단계별 확률을 읽으세요. `confidence`(신뢰도) 필드는 불확실성을 요약한 값이므로 행동의 근거로 삼을 숫자가 아닌 경우가 많습니다. Choice에서는 최고 확률로부터 계산되므로 두 번째 검증 수단이 되지 못합니다. [Confidence를 근거로 행동해야 할 때](confidence.md#when-to-act-on-confidence)를 참고하세요.

### TypeSafe 코드를 검토하기 어렵습니다

사람이 검토해야 할 가장 중요한 것은 TypeSafe 코드에서 쓰는 질문과 threshold 상수입니다. 깊이 파헤치지 않아도 쉽게 찾을 수 있도록 이들을 코드 파일 하나에 정의해 두세요.

### 에이전트가 요청이나 응답 필드를 지어냅니다

skill이 오래되면 이런 일이 생길 수 있습니다. 위에서 사용한 설치 방법에 맞게 업데이트한 뒤 다시 시도하세요.


이 문서는 개발자 문서 플랫폼인 [Mintlify](https://mintlify.com)로 제작하고 호스팅합니다.
