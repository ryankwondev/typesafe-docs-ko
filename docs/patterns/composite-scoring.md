# Composite scoring

> 복잡한 판단을 더 쪼갤 수 없는 개별 점수로 나누고, 코드에서 직접 정한 가중치로 합칩니다.

여러 criteria(판단 기준)를 한꺼번에 놓고 항목들의 순위를 매겨야 할 때가 많습니다. composite scoring(복합 점수화)은 이 문제를 쉽게 풀어 가는 방법입니다. 판단을 서로 독립적인 차원으로 나누고, 차원마다 따로 점수를 매긴 다음, 코드에서 직접 정한 가중치로 합칩니다.

## 예제: 이력서 선별하기

엔지니어링 직무 지원자의 이력서를 처리한다고 가정해 보겠습니다. 여러 criteria로 지원자의 순위를 매기고, 최종적으로 상위 X명을 골라 추가 검토를 진행하려고 합니다.

```mermaid
%%{init: {"fontFamily": "Inter, sans-serif", "flowchart": {"rankSpacing": 35, "wrappingWidth": 300, "subGraphTitleMargin": {"top": 12, "bottom": 36}}}}%%
flowchart LR
    resume["지원자 이력서"]

    subgraph req["Jev가 질문을<br/>병렬로 평가"]
        direction TB
        py["<b>Score:</b> Python 숙련도"]
        lead["<b>Score:</b> 팀 리더십"]
        arch["<b>Score:</b> 시스템 설계"]
        general["<b>Score:</b> 제너럴리스트"]
        %% 보이지 않는 링크로 질문을 세로로 쌓습니다. 질문은 병렬로 답변됩니다.
        py ~~~ lead ~~~ arch ~~~ general
    end

    resume -- "요청 1건<br/>이력서 + 질문 4개" --> req
    req -- "응답 1건<br/>Score 답 4개" --> normalize["<b>점수를 0–1로 정규화</b><br/>코드에서 각 점수를 4로 나눔"]
    normalize ic_weights@--> ic["<b>시니어 IC 가중치</b><br/>Python 40% + 리더십 10%<br/>설계 40% + 제너럴리스트 10%"]
    normalize em_weights@--> em["<b>엔지니어링 매니저 가중치</b><br/>Python 15% + 리더십 40%<br/>설계 20% + 제너럴리스트 25%"]
    ic --> rank["직무별로<br/>지원자 순위 매기기"]
    em --> rank
    %% 두 가중치 경로에 흐름 애니메이션을 적용합니다(prefers-reduced-motion에서는 styles.css가 멈춥니다).
    ic_weights@{ animate: true }
    em_weights@{ animate: true }
```

### 차원마다 따로 점수 매기기

이력서를 보내면서 차원마다 Score 질문을 하나씩 붙입니다. 질문 하나가 판단 하나만 요구하므로, 나중에 점수마다 가중치를 따로 줄 수 있습니다.

Playground에서 실행해 보기 →

### 가중치로 점수 합치기

각 차원의 점수를 0–1 범위로 정규화한 뒤 가중치를 적용합니다. 가중치를 쓰면 개별 점수에 담긴 세밀한 차이를 잃지 않으면서 각 차원의 상대적 중요도를 손쉽게 조정할 수 있습니다.

```python title="scoring.py"
py      = response.answers["python_depth"].score / 4
lead    = response.answers["team_leadership"].score / 4
arch    = response.answers["system_design"].score / 4
general = response.answers["generalist"].score / 4

# 시니어 IC
ic_score = (0.40 * py) + (0.10 * lead) + (0.40 * arch) + (0.10 * general)

# 엔지니어링 매니저
em_score = (0.15 * py) + (0.40 * lead) + (0.20 * arch) + (0.25 * general)
```

이제 복합 점수로 지원자의 순위를 매길 수 있습니다. 더 중요한 점은 최종 점수가 정확히 어떻게 계산되는지 들여다볼 수 있다는 것입니다. 상위에 오른 지원자가 기대와 다르다면 가중치를 조정해 알맞은 균형을 찾으면 됩니다.


이 문서는 개발자 문서 플랫폼인 [Mintlify](https://mintlify.com)로 제작하고 호스팅합니다.
