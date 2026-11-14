# 구조 복원

> 서식이 사라진 일반 텍스트에서 Markdown을 요청 두 번으로 복원합니다. 첫 요청은 강제 줄바꿈으로 끊긴 줄을 다시 이어 붙이고, 두 번째 요청은 모든 블록을 제목, 목록, 코드, callout 등으로 분류합니다.

이 cookbook(예제 모음)은 마크업이 모두 벗겨진 일반 텍스트를 받아 구조를 Markdown으로
복원합니다. 입력 텍스트는 문장 중간에서 줄이 강제로 바뀌어 있고, 제목 표시도 목록 기호도
없습니다. 복원 대상은 제목, 문단, 목록, 인용, 코드, callout(콜아웃: 강조 상자)입니다.
예제로 쓰는 입력은 정확히 그런 상태가 된 팀 메모입니다.

텍스트 생성 모델에 맡겨 Markdown으로 다시 쓰게 할 수도 있지만, 다시 쓰는 과정에서 단어가
바뀔 수 있습니다. 여기서는 모델이 텍스트를 전혀 생성하지 않습니다. 모델은 문서에 대한 좁은
질문(*이 줄은 문장 중간에서 이어지는가? 이 블록은 어떤 종류의 내용인가?*)에만 답하고,
렌더링은 코드가 맡습니다. 그래서 출력의 모든 글자는 입력에서 그대로 오고, 모든 판단에는
확률이 붙습니다.

pipeline(파이프라인) 전체는 문서 하나당 API 요청 두 번이며, 순서대로 실행합니다.

* **1단계, 이어 붙이기:** 인접한 두 줄마다 `Noul` 질문(예/아니요 질문이며, 답은 "예"가
  맞을 확률입니다)을 하나씩 만들어, 줄바꿈이 한 문장을 두 줄로 갈라놓았는지 묻습니다. 모든
  쌍을 요청 하나에 담아 보내고, 끊긴 문장을 이어받는 줄은 앞줄과 합쳐 블록으로 만듭니다.
* **2단계, 분류:** 합쳐진 블록마다 `Choice` 질문(목록에서 선택지 하나를 고르며, 모든
  선택지에 확률이 붙습니다)을 하나씩 만들어 제목, 문단, 목록 항목, 인용, 코드, callout(본문과
  구분해 보여 주는 참고, 팁, 경고) 중 하나를 고릅니다. 블록은 1단계의 답이 나와야 생기므로
  이 단계는 두 번째 요청이 됩니다. 이 요청에는 블록마다 딸린 동반 질문(제목 단계, 단계 순서,
  callout 종류)도 함께 담기며, 그 답은 블록의 유형상 필요할 때만 읽습니다.
* **직접 확인할 수 있는 근거는 코드에서 처리합니다.** 빈 줄과 명시적인 표시(`- `, `1.`, `#`)는
  코드가 직접 읽고, 모델에 다시 판단해 달라고 보내지 않습니다. 이 메모에는 빈 줄은 남아
  있지만 표시는 모두 사라졌습니다. 모델에는 코드가 텍스트만 보고 답할 수 없는 질문만
  보냅니다.

## 준비

```bash
pip install ipython 'cooksafe>=0.3.0,<0.4.0'
```

그런 다음 `TYPESAFE_API_KEY`를 설정합니다.

```python
import json
import os
import re
import urllib.request
from pathlib import Path
from time import perf_counter

from cooksafe import JsonCache, make_playground_link
from IPython.display import Markdown, display
from typesafe_sdk import Choice, Noul, NoulCriteria, TypeSafeClient

TYPESAFE_MODEL = "jev-1.14.0"  # 이 페이지에서 쓴 모델입니다. "jev-latest"로 바꾸면 최신 모델을 실시간으로 호출합니다
client = TypeSafeClient(
    api_key=os.environ.get("TYPESAFE_API_KEY", "cache-only"), timeout=120.0
)
```

```python
EXAMPLES_URL = (
    "https://raw.githubusercontent.com/typesafe-ai/typesafe-public-examples/main/"
    "cookbooks/autoformat"
)


def download(name: str) -> Path:
    """Fetch one of this cookbook's files from typesafe-public-examples unless it is on disk."""
    path = Path(name)
    if not path.exists():
        urllib.request.urlretrieve(f"{EXAMPLES_URL}/{name}", path)
    return path


CACHE = Path("autoformat_cache.json")
# True이면 공개된 답변으로 CACHE를 채우므로 실행 중에 요청을 보내지 않습니다. 실시간으로
# 실행하려면 False로 바꾸고 CACHE를 삭제하세요. 디스크에 이미 있는 CACHE는 덮어쓰지 않습니다.
DOWNLOAD_CACHE = True
if DOWNLOAD_CACHE:
    download(CACHE.name)
json_cache = JsonCache(CACHE)
```

## 문서: 서식이 사라진 팀 메모

테스트 문서는 빌드 시스템 마이그레이션을 알리는 메모이며, 일반 텍스트 수신함에 도착했을
때의 상태 그대로입니다. 문단은 문장 중간에서 줄이 강제로 바뀌어 있고, 셸 명령은 아무 표시
없이 한 줄에 놓여 있으며, 목록 두 개에는 기호도 번호도 없고, 경고문에는 경고임을 알리는
표시가 없습니다.

```python
DOCUMENT_URL = (
    "https://raw.githubusercontent.com/typesafe-ai/typesafe-public-examples/main/cookbooks/autoformat/build-memo.txt"
)


@json_cache
def fetch_document(url: str) -> str:
    request = urllib.request.Request(url, headers={"User-Agent": "typesafe-cookbook/1.0"})
    with urllib.request.urlopen(request) as response:
        return response.read().decode()


RAW = fetch_document(DOCUMENT_URL)
print(RAW[:560])
```

```
Migration to the new build system

Hi everyone, quick heads up about the build system migration that is
happening next week. We have been running the new pipeline in shadow
mode for three weeks and the results look solid, so it is time to
make the switch for real.

What changes for you

The old make targets keep working until the end of the month. The new
entrypoint is a single command that wraps everything, including the
docs build that used to be separate.

bun run build

Generated artifacts no longer need to be committed. The new pipeline
uploads them
```

줄 나누기, 빈 줄 추적, id 붙이기는 모두 요청을 보내기 전에 끝나며, 여기에는 모델이 관여하지
않습니다. 각 줄에는 짧은 id(`L014| `)가 붙습니다. 모델은 이 id를 state(상태) 안의 평범한
텍스트로 읽고, 질문과 답은 id로 줄을 가리킵니다.
[시맨틱 검색 cookbook](semantic_find.md)도
같은 방식을 씁니다.

```python
def to_lines(text: str) -> list[dict]:
    lines, gap = [], False
    for raw in text.split("\n"):
        stripped = re.sub(r"[\t ]+", " ", raw).strip()
        if not stripped:
            gap = bool(lines)  # 맨 앞의 빈 줄은 구분으로 치지 않습니다
            continue
        lines.append({"text": stripped, "gap": gap})
        gap = False
    return lines


def tag(items: list[dict], prefix: str) -> str:
    return "\n".join(
        f"{chr(10) if item['gap'] else ''}{prefix}{i:03d}| {item['text']}"
        for i, item in enumerate(items)
    )


def line_id(i: int) -> str:
    return f"L{i:03d}"


def block_id(i: int) -> str:
    return f"B{i:03d}"


LINES = to_lines(RAW)
print(f"{len(LINES)} non-blank lines. The model sees, e.g.:")
print("\n".join(tag(LINES, "L").splitlines()[19:24]))
```

```
28 non-blank lines. The model sees, e.g.:
L013| The cutover touches three teams, so check whether you are on this
L014| list before you plan anything for Monday:
L015| The platform team
L016| The web client team
L017| Whoever still owns the release tooling
```

## 1단계: 끊긴 문장 이어 붙이기

인접한 두 줄마다 Noul 질문을 하나씩 만들어 모두 요청 하나에 담습니다. 빈 줄로 나뉜 쌍은
건너뜁니다. 질문은 일부러 좁게 잡았습니다("이 줄은 문장 중간에서 이어지는가?"). 이 정도면
텍스트에 대한 객관적 사실에 가깝습니다. 질문에는 두 줄의 내용도 그대로 인용해 넣습니다.
그래야 모델이 28줄 가운데서 id로 해당 줄을 찾지 않고, 문제가 되는 텍스트 자체를 보고
판단합니다.

```python
def join_question(i: int) -> Noul:
    return Noul(
        instructions=(
            f'Line {line_id(i - 1)} is "{LINES[i - 1]["text"]}" and line {line_id(i)} is '
            f'"{LINES[i]["text"]}". Does line {line_id(i)} pick up mid-sentence, continuing a '
            f"sentence left unfinished at the end of line {line_id(i - 1)}?"
        ),
        criteria=NoulCriteria(
            true="The line starts in the middle of a sentence that began on the previous line - the line break tore the sentence apart",
            false="The line begins a new sentence, item, heading, or thought of its own",
        ),
    )


def as_json(questions: dict) -> str:
    return json.dumps(
        {k: q.model_dump(mode="json", exclude_none=True) for k, q in questions.items()}
    )


def stitch_questions(make) -> dict:
    return {line_id(i): make(i) for i in range(1, len(LINES)) if not LINES[i]["gap"]}


@json_cache
def stitch(state: str, questions_json: str, model: str) -> dict:
    started = perf_counter()
    response = client.system_one(
        state=state, questions=json.loads(questions_json), model=model
    )
    return {
        "joins": [
            response.answers[line_id(i)].noul if line_id(i) in response.answers else 0.0
            for i in range(len(LINES))
        ],
        "seconds": round(perf_counter() - started, 2),
        "usage": [response.usage.input_tokens, response.usage.output_tokens],
    }


result = stitch(
    tag(LINES, "L"), as_json(stitch_questions(join_question)), model=TYPESAFE_MODEL
)
print(f"{sum(1 for l in LINES if not l['gap']) - 1} pair questions, one request, "
      f"{result['seconds']}s")
```

```
16 pair questions, one request, 0.24s
```

두 줄을 합치는 기준값은 앞줄이 어떻게 끝나는지에 따라 달라집니다. 앞줄이 끝맺지 않은
줄(문장을 끝내는 문장 부호가 없는 줄)이면 이어짐 확률이 0.2 이상일 때 두 줄을 합칩니다.
앞줄이 종결 문장 부호(`.` `!` `?` `:` `;`)로 끝나면 기준값이 0.5로 올라갑니다.

```python
JOIN_AFTER_DANGLING, JOIN_AFTER_TERMINAL = 0.2, 0.5


def ends_terminal(text: str) -> bool:
    return re.search(r'[.!?:;…]["\')\]]*$', text) is not None


def merge(joins: list[float]) -> list[dict]:
    blocks = []
    for i, line in enumerate(LINES):
        bar = (
            JOIN_AFTER_TERMINAL
            if i and ends_terminal(LINES[i - 1]["text"])
            else JOIN_AFTER_DANGLING
        )
        if blocks and not line["gap"] and joins[i] >= bar:
            blocks[-1]["text"] += " " + line["text"]
            blocks[-1]["lines"].append(i)
        else:
            blocks.append({"text": line["text"], "lines": [i], "gap": line["gap"]})
    return blocks


blocks = merge(result["joins"])
healed = len(LINES) - len(blocks)
print(f"{len(LINES)} lines -> {len(blocks)} blocks ({healed} line breaks healed)")
for i, block in enumerate(blocks):
    n = len(block["lines"])
    print(f"{block_id(i)}  {n} line{'s' if n > 1 else ' '}  {block['text'][:62]}")
```

```
28 lines -> 17 blocks (11 line breaks healed)
B000  1 line   Migration to the new build system
B001  4 lines  Hi everyone, quick heads up about the build system migration t
B002  1 line   What changes for you
B003  3 lines  The old make targets keep working until the end of the month. 
B004  1 line   bun run build
B005  3 lines  Generated artifacts no longer need to be committed. The new pi
B006  2 lines  The cutover touches three teams, so check whether you are on t
B007  1 line   The platform team
B008  1 line   The web client team
B009  1 line   Whoever still owns the release tooling
B010  1 line   Things to do before Monday
B011  1 line   Update your local toolchain to version 2.4 or later
B012  1 line   Delete the old build cache directory
B013  1 line   Run the doctor script and fix anything it flags
B014  3 lines  If the doctor script reports a red result on the toolchain che
B015  2 lines  As Dana put it in the kickoff, "a migration nobody notices is 
B016  1 line   Thanks, and shout if anything looks off.
```

## 2단계: 블록 분류

이어 붙인 블록마다 `Choice` 질문을 하나씩 던집니다. *이 블록은 어떤 종류의 내용인가?* 아래
dict 세 개와, 뒤에 나오는 `classify_questions` 안에 있는 단계 질문의 true/false
criteria(판단 기준)가 분류기 명세의 전부입니다. 이 pipeline을 여러분의 문서에 맞추려면 이
설명문을 고치면 됩니다.

```python
TYPE_CRITERIA = {
    "heading": "A short label or title that names the document or the section that follows it - not a full sentence of content",
    "paragraph": "Running prose: one or more complete sentences of explanatory or narrative text",
    "list_item": "One entry in a list of parallel items - an ingredient, a feature, a task, an attendee; reads as one of several sibling entries",
    "quote": "Words attributed to a person or source - quoted speech, a citation, an excerpt someone else wrote",
    "code": "Computer code, a shell command, terminal output, or a config snippet meant to be read verbatim",
    "callout": "A warning, tip, or important note that interrupts the flow to flag something the reader must not miss",
}
HLEVEL_CRITERIA = {
    "title": "The title of the whole document",
    "section": "A major section heading within the document",
    "subsection": "A minor heading nested under a section",
}
CALLOUT_CRITERIA = {
    "note": "Neutral extra information the reader should be aware of",
    "tip": "A helpful suggestion or shortcut that makes things easier",
    "warning": "A caution about something that can go wrong or cause harm",
}
```

유형이 `heading`으로 나오면 렌더러는 제목 단계를 알아야 하고, `list_item`이면 순서가
중요한지, `callout`이면 어떤 종류인지 알아야 합니다. 하지만 유형은 아직 알 수 없고, 유형이
나올 때까지 기다리면 왕복이 한 번 더 생깁니다. 그래서 동반 질문을 같은 요청에 미리 담아
보냅니다. 이 답의 대부분은 끝내 읽지 않습니다. 예를 들어 문단의 단계 확률은 쓰이지
않습니다. 질문을 하나 더 넣어도 늘어나는 비용은 작습니다. token(토큰)의 대부분은 state가
차지하고 state는 어차피 한 번만 보내기 때문입니다. 반면 왕복이 한 번 늘면 요청 하나만큼의
지연 시간이 고스란히 더해집니다.

```python
HEADING_MAX_CHARS = 90  # 이보다 긴 블록은 제목으로 렌더링할 수 없으므로 묻지 않습니다


def classify_questions(texts: list[str]) -> dict:
    questions = {}
    for i, text in enumerate(texts):
        bid = block_id(i)
        questions[f"type_{bid}"] = Choice(
            instructions=f"What kind of content is block {bid}?", criteria=TYPE_CRITERIA
        )
        if len(text) <= HEADING_MAX_CHARS:
            questions[f"hlevel_{bid}"] = Choice(
                instructions=f"As a heading, what level would block {bid} occupy in this document's structure?",
                criteria=HLEVEL_CRITERIA,
            )
        questions[f"step_{bid}"] = Noul(
            instructions=f"Is block {bid} an instruction in a sequence where the order of the items matters?",
            criteria=NoulCriteria(
                true="It is one step of a procedure - the items around it must happen in order",
                false="Order is irrelevant - it is a loose collection, or not a list item at all",
            ),
        )
        questions[f"callout_{bid}"] = Choice(
            instructions=f"What kind of aside is block {bid}?", criteria=CALLOUT_CRITERIA
        )
    return questions


@json_cache
def classify(texts: list[str], gaps: list[bool], questions_json: str, model: str) -> dict:
    tagged = tag([{"text": t, "gap": g} for t, g in zip(texts, gaps)], "B")
    questions = json.loads(questions_json)
    started = perf_counter()
    response = client.system_one(state=tagged, questions=questions, model=model)
    judgments = []
    for i in range(len(texts)):
        bid = block_id(i)
        type_answer = response.answers[f"type_{bid}"]
        hlevel = response.answers.get(f"hlevel_{bid}")
        judgments.append(
            {
                "type": type_answer.choice,
                "confidence": type_answer.confidence,
                "probabilities": type_answer.probabilities,
                "hlevel": hlevel.choice if hlevel else "section",
                "step": response.answers[f"step_{bid}"].noul,
                "callout": response.answers[f"callout_{bid}"].choice,
            }
        )
    return {
        "judgments": judgments,
        "n_questions": len(questions),
        "seconds": round(perf_counter() - started, 2),
        "usage": [response.usage.input_tokens, response.usage.output_tokens],
    }


texts = [b["text"] for b in blocks]
classified = classify(
    texts, [b["gap"] for b in blocks], as_json(classify_questions(texts)), model=TYPESAFE_MODEL
)
for block, judgment in zip(blocks, classified["judgments"]):
    block.update(judgment)
print(f"{classified['n_questions']} questions about {len(blocks)} blocks, one request, "
      f"{classified['seconds']}s\n")
print(f"{'block':<6}{'type':<11}{'conf':<6}{'companion used':<18}text")
for i, b in enumerate(blocks):
    companion = {
        "heading": f"level={b['hlevel']}",
        "list_item": f"step={b['step']:.2f}",
        "callout": f"kind={b['callout']}",
    }.get(b["type"], "-")
    print(f"{block_id(i):<6}{b['type']:<11}{b['confidence']:.2f}  {companion:<18}"
          f"{b['text'][:46]}")
```

```
62 questions about 17 blocks, one request, 0.22s

block type       conf  companion used    text
B000  heading    0.98  level=title       Migration to the new build system
B001  paragraph  0.89  -                 Hi everyone, quick heads up about the build sy
B002  heading    1.00  level=section     What changes for you
B003  paragraph  1.00  -                 The old make targets keep working until the en
B004  code       1.00  -                 bun run build
B005  paragraph  0.92  -                 Generated artifacts no longer need to be commi
B006  paragraph  0.65  -                 The cutover touches three teams, so check whet
B007  list_item  1.00  step=0.07         The platform team
B008  list_item  1.00  step=0.08         The web client team
B009  list_item  1.00  step=0.08         Whoever still owns the release tooling
B010  heading    1.00  level=section     Things to do before Monday
B011  list_item  1.00  step=0.56         Update your local toolchain to version 2.4 or 
B012  list_item  1.00  step=0.55         Delete the old build cache directory
B013  list_item  1.00  step=0.68         Run the doctor script and fix anything it flag
B014  callout    0.84  kind=warning      If the doctor script reports a red result on t
B015  quote      0.99  -                 As Dana put it in the kickoff, "a migration no
B016  paragraph  0.99  -                 Thanks, and shout if anything looks off.
```

companion 열을 보면 미리 받아 둔 답이 어떻게 쓰였는지 알 수 있습니다. "Things to do before
Monday" 아래 세 줄은 단계 확률이 0.55에서 0.68이므로 번호 목록으로 렌더링되고, 팀 이름 세
줄은 0.07에서 0.08이므로 글머리 기호 목록이 됩니다. 그리고 doctor 스크립트에 관한 세 줄짜리
경고인 B014는 `warning` 종류의 callout으로 분류되었습니다.

## 렌더링

코드가 이 판단들을 모아 페이지를 조립합니다. 연속된 목록 항목은 목록 하나로 묶고, 항목들의
단계 확률 평균이 0.5 이상이면 번호를 붙입니다. 이 threshold(임계값)는 묶음 전체에 대한
결정이며, 어느 질문도 이것을 직접 묻지는 않았습니다.

````python
STEP_THRESHOLD = 0.5
HEADING_MARK = {"title": "#", "section": "##", "subsection": "###"}
CALLOUT_MARK = {"note": "NOTE", "tip": "TIP", "warning": "WARNING"}


def to_markdown(blocks: list[dict]) -> str:
    groups = []
    for b in blocks:
        if b["type"] in ("list_item", "code") and groups and groups[-1][0] == b["type"]:
            groups[-1][1].append(b)
        else:
            groups.append((b["type"], [b]))
    parts = []
    for kind, items in groups:
        if kind == "list_item":
            ordered = sum(b["step"] for b in items) / len(items) >= STEP_THRESHOLD
            parts.append("\n".join(
                f"{n + 1}. {b['text']}" if ordered else f"- {b['text']}"
                for n, b in enumerate(items)
            ))
        elif kind == "code":
            parts.append("```\n" + "\n".join(b["text"] for b in items) + "\n```")
        elif kind == "heading":
            parts.append(f"{HEADING_MARK[items[0]['hlevel']]} {items[0]['text']}")
        elif kind == "quote":
            parts.append(f"> {items[0]['text']}")
        elif kind == "callout":
            parts.append(f"> [!{CALLOUT_MARK[items[0]['callout']]}]\n> {items[0]['text']}")
        else:
            parts.append(items[0]["text"])
    return "\n\n".join(parts) + "\n"


markdown = to_markdown(blocks)
print(markdown)
````

````text
# Migration to the new build system

Hi everyone, quick heads up about the build system migration that is happening next week. We have been running the new pipeline in shadow mode for three weeks and the results look solid, so it is time to make the switch for real.

## What changes for you

The old make targets keep working until the end of the month. The new entrypoint is a single command that wraps everything, including the docs build that used to be separate.

```
bun run build
```

Generated artifacts no longer need to be committed. The new pipeline uploads them to the registry automatically, and checking them in just creates merge conflicts.

The cutover touches three teams, so check whether you are on this list before you plan anything for Monday:

- The platform team
- The web client team
- Whoever still owns the release tooling

## Things to do before Monday

1. Update your local toolchain to version 2.4 or later
2. Delete the old build cache directory
3. Run the doctor script and fix anything it flags

> [!WARNING]
> If the doctor script reports a red result on the toolchain check, do not proceed with the migration. Ping the infra channel first and we will sort it out together.

> As Dana put it in the kickoff, "a migration nobody notices is the only kind worth shipping."

Thanks, and shout if anything looks off.
````

위 출력의 모든 단어는 입력에서 왔습니다. pipeline이 정한 것은 경계, 유형, 마크업뿐입니다.

## Playground에서 열어 보기

이 공유 링크에는 이어 붙인 블록과 2단계 질문 전체가 들어 있습니다. 링크를 열면 분류를
실시간으로 다시 실행해 볼 수 있습니다.

```python
playground_link = make_playground_link(
    tag(blocks, "B"),
    classify_questions([b["text"] for b in blocks]),
)
display(Markdown(f"🔗 [Open the stitched memo + questions in the TypeSafe playground]({playground_link})"))
```

[TypeSafe playground에서 이어 붙인 메모와 질문 열기 →](https://console.typesafe.ai/playground#share/N4IgJg9gxgrgtgUwHYBcAqCAeKQC4AEIAQgAxkA++AsgJYDmATgIYo0RL4oScAWC+SBAHd8AIxg0ANmHwBnAJ6yUCOAB0k60iQCMlABI18CAG4IG89ggA0+AI4SoAa3x8mYWfhgAHfE1EQYFF5+cSkZBSUVfDh6ZlZ2XhZ8Gg8eJi8vZBokOgEsIKEEBEcAOnwAdX400zEijgYYJCRs3JQ+PJEvGkzJbP5suTTIETgIMH4AMwgGXgYi-ELijyYkGTb+OdkYSRQPSQgIZ1kIXrAbY+SglM4aRE5uOCZHfnW5IRoUKB58KZm5pkkJXUmjIACZKOU0kEvis6AgPL98BYYMCkFoAMyUNDtE4yR7PThMBhw3b4Z4IHxCaaOFqeVBSYJGVb4CATRmjVA8MrY-iCETIFDmLwQbJXZZyFqSfhQCBwR7MtpJITMLweExmeRtFo2bJQSQwMC016QKAeULSRJBGCyBBrbiifg2rxElgIIEaNFkAAslHE9UaYgk0lRWgArJQAOLIMyumRE1gTJhQUlIbj7HJmPK2+61fAyuUfZRgbntPn4Lo9PqeLz7NwedZwHOvOZ0FKC+S+QKylg0KAAyTyGwrGRfBBOI18RsDABW1uh-2UHkQxOl7AmvWTsndIJIADYse1YFxTDMuDBR-WeHMXggmHBZOduKOnAs+OsZsjfHMWRwtXs27Uvz8J+NYrL4SCajwtKIlQ7BgEw8i4DuADsB78KBKC-I2yh3juAAcaELAgoh5r0AqcLeaieiQACcEI8BA6ozEoUiSCyQhIJeGwIFKTA2vcJwtCGOgkAeLT1twkCAdM-CwasCHCdouj4AAql48HKEiAQzPsfZsVwJwwgMXD4CeshsBwoIlF6LI6a6DAgto4L4AAIjxCCaa8uKBmEeZJu0hpzMm0zyI5mL4AASgGxrQFwzFQAw3RBMOPw0Jg4GQbSHw-JITB0LIik+vgACSbIxcF8WJV4QRzMKDCkkw+BzDImzbEECSvAZkhGRwz6ODYUmpkEXgMNARQyO8bTsrEPbsGUAAKE79EgEzMHmaRNDxqUMEo4ETfw7ySGxxz1ZcLKBPcJJ8Aw26eto4b4AAgh4LkrI1XgXdlxntDSTishMNiqCAjUxIws0cKm-hgB2Q29vCyRcT+A5ktkE3TFNshQRkLRAiAin7vg2IrI4D57YMARXGyKyZTk+D7IcHj-SUIA2CAI2ytVsgYNgeCEMAQMoPImQAPpaCQQMEPzICC5kEv4EDXwilACBA4DIDZEoDTJhZBW80DkJJDSzKsnm7DKKgCNiLpzhiwA-Kr8sgAlHxmDQTBy1LriGjkctA495OnblDpsdMNwoFKloCHe8PlfA5Gh68Nrax1UI-Cc+xCB42UALQCBAyU-Nsx0CsgyssmyMqoAKDtA86zBg14PC+yAUVNLS7M2gQli2dEMmm3ANYeY6JdIMrjNslgoFICwIU99PDBxDQNTKNgNcgL0SjCy7ah6yAADyghMu2yQcI1G-tWydf9lt28eLnYEtM1NACkOPy3igMBzK-KB8f14G+Cgc24wEAAG4mq3ncL4Rmh8TY2hPACCUohei0wFIleEa97D5xVrvco0xIEsEFDQcQRYczvTMMcDgodjifzLrnTBJDZCZDHDwV+UAPjgyHBwLAysGDVTkLKBA3ceL8WVFgteMpxjNwAMIc0CJmCR1hfCDB4mxfM8ozgUQYDEaeIdAgfRQDYUOjVK4THoHIZoGQPLRFvBbEyDpwFuFMmYUQPYd5qz0vsQIzd-ZCCJM0HINhWBeEMTMW4dUf4WyGi8VOoozANE5oydcEARAmXXHlfhiAtS02bBAzMcA5x5yCDEWQusAC+pS1Y8ClKYSQosyDi15lLGW2CCAKwYnDNeGtBTnniJxbx4ovbalfEkapW0qTbBkEg6ANt6ksigLALwHZvrXBNHHVAAByDwmsemfwQPbFmjtnbKESu7RpAsPhSmbjyMOEcTavCEAxCOqzECoDXknXp3johMGnFQscvSXAQNpJNKCv4ArQDWSgN5MBRDvIsp87RodBm00EJEGQjRxgzEarC9gQNylq0iF4OpZAPYCyFi0x2qZtidM4t05OfTd7FTNNbfAYt-5dK1v8gYWKED2FLgda60T+B4MzHc9ot8vmAPIfs9xiVjluxJdLBo5KgbFTFD+R0ygfAm3eqNZWYBdn4Fzq8cVRIAjMmyvk3aaRLEcAGMKhyBygaJkkDaZue8GAYstjQBe7ljArCCLnL64p6b8RlEdP5FkQmFKUefS4UQkj9lxRUw5-YKZEoaZLUlstd6Kw6Y69WNKOU62bgbIIRsZDavMuMS2UyXx23EbK12pzM0gCic3AAcggQIzA2L5DWtkLC4NGT-E9ZjAIFp7FMF8d+Vka8gmfL4JILwExthyBgHQOELEEhUIYvVI8kd8QxxBflIwfFn4OrVtO-xdBPl9kCBZXw-gLrHEyceyOfYOB0G4KI2moc738TSFopNatmlEu0Aq5pzdc3K2pdsulutWkgFLSjY2FczbkWuLWmZOhpWHMbSchVSKb2739mOwOfgtoJwudEpI09ECI2efHU87RsWgqSFMI6ySs4BujY1FdR05AjzLibSu5tIX5qvg3Juu9W7XvLKNTu6qe6jG-PmQemkbRV1HvDE2k9crTzih2UO89F7L3yGvc+W9lA70QwffgqClmnzpgBE2V9w1sXFffW1OQn4vyURMd+uzv6-04QAoBRQwEjuWNAoVbI4ExmOkQ5BuRUHP11mrehyqkN4OWIAxKxDswmTITtbdzFtK0LsDALB4QmFfFYew3poXuFmD4S+wRh9hEHVGsocRYwssyIHnImYCjX6YxUf3dRgSzDaIQRTfRUbjFrjMbICxmQik2KCHY7ijiTwuNYG4lNnGvEkYWH4oZQSo1hPRv6wpNGrhVwXt4UkrwkkpO4Gk3IbWsmtHaCOvJBShrRBSGU5NQMCVgYg2S5ulLJCwdpb0hDjtGVW2mSysg2g2WFp6Q+rlgneXadfGYQVtlPWiv6NZpcBCpUNpdgRs5iqYBZdVZbbuBLy5KPZnqg1RqxUU6-GamQFqCnWsyN5knZg17Otdbvd1nrrjermNUm7ga1VnwOKG9OEb2BRsB2fAC28AG+COsBw7niUAQ-p5BnN7SYP5vZdj9giP9ap3Lezs91bMPMq0NoXDCt8Pyvp223enbu0IL7Y1Ad0xHj-JyW4TMY6Jm5inUSWLc7ugLp4su1dWwN3wn+Tu9G+7FRFKeEe8Sp7zIS-zVeoSJ273-L8BTDJHk33F78p+79o1f3DaYNaKoRId54sdqBrQoJIfZsQ9B7Bat7fwZLS71G7PRMYaZaj0fvunb++bfgT2gKfYnbI0EIOlHTzUcjnRo9-BGMWwTixrXbGggcYztxw1vHC4Cc0+bYTaGtPibVpJlUaTRDWTduBTBALuGBGYFTVcAeKUDTITHTCeTAKeGecwOeIkUzG8VefNSzbeN1DrVANA3HWNVzF0dzONe8V-B+HzW0Z+VAV+ALFgILJRH+WQP+MCKnVYCLBxfBGLdneLHtRBZLI+NBdLR2TLEtHLMLfLORO0DnchUrfhGhfgOhKrBhWrFhJRNhH+Rrf+ZrXhIINrIRF1LrMRfNBRaRWRY5U2cYUbRdVRWUSbTRGbXRFAebHuRbFaZbVbKxRAG7LbHgpxBgPbW4cRVNY7RDHxM7AJG4YJHuK7eqG7KJSOWJR7BJF7DOHMD7Zvb7YdXJKAgHfOIHEpE3IGKpdUWpUfcfLLKfOHItR3fpJRIjGwB5EZCohYcdSZL3MEOZBZRzXgFZcFF5FATZOQeHXZDfI5JtCDajK5doVgcOWLRkB5E4K-IY6ufNVjeFb5X5OlAFNwIFD4EFRka-P-R2LYGFe-eFbIRFPfXIFFEhdFTMblOlUokAcHKoy3KHXeGHOoh3elRDZHLDNHEgUETHODTlJzG0fHMuB5InRke1dnY1PnaPY5WQSYrfCDJVZuZna4VnTVN3eTMafVb8HncnFQZYUadFM6S1IIEXZAE+cXC9R2KXLLWXTMeXH1JXC2FXS2NXCADXcNOlHXIovXXaA3BNY3EAIfBWcI83T4ltK3SfG3afR2WfBHefQ2RfStGgD3VfOtMEDE2nAPFtIPRDEPQUMPbAftFaKPIdWPUdBiRPSdadVPfNedE7RdLPY6ddTdfPZiXdT4C6NvQ9S8cvW8SvZkoGGvffSIvye9BIRvZ9ARXItvD9fAL9BYLvXIP9XvADAfN4kfMgdEaoqDFUv4ufHBBfVDU2X-GtbokgdEI0uVbfXfA42Mx2UjQMumCjEOU-RY8-aOBjdYm-ZjYePYtvJ-LjM6XOXXd-YuLTb-WssTNeAA9IIAx2EA2mDucApTUOaA-udTYeRcxAowZA-TVAozGYEzHsMzbAtWXA6zfA+zQg-osUi+csMgqUDzPnLzE+RgWgvzPjQLL+FgkLf+Tg4BSLCBaLPcuLJiBBcyJBWkVLdBfNCQnBKQghGQkhIrcsBQyhMrZQ1-TLGrIoOrLQhrSNPQzAHhVrARYwkRbrVUhWPrSwwbawkbJRMbATNRYcKbLRbIWbPRQIBbWs0xT7Hw9bfw+0bbGQXbVxMIo7cTOMmMugQJboS7Aea7SJLBFIh7eJZ7doV7LI3KT7FM1vX7fI6IQoopYHN4j44s0sn48dCsjUhlfU7DdEcE+HHHKEnlRnAnOE78LyD1EVMqXnCkiVNE5s6Yy3bEhlNVfEikQkznW0bnRkE1Kk81IpYXdIUXRk+1SXAEaXWzMK0JLOLkkwZXM6a4fkwUqUYUueUU5zcU6zQ3RNaU0HJ2OUolEsr4ifQ5csu3LHSsxDZDV3HUvUlHA0xs2KunU0sw80rtS03ta0iPW0rRe0qyuPAMzopPV08uNPLwDPJdfjNdXPLdQigOIMzbVOUMgYnINUM9KvS9aI4jOM+vB9JMwwiyicJIdMzMn9HMnvPvFwAsrqkDMlIlL0Zy5UpWFigtCE4tKsrUms5fC2T3Nfb0eak0nfMou4z5Q-Hs4OHuBYiONvC-Yc2AYYsmu-Cc1OKczOGct-fjBcr-JYjGs42uF0KTZuLc3IHciAoVKAvuNTOA48jm8eM8lAwzdAheW8rA7m9eNsKzFQZ8kQt81qj8tzb8igu+THACw0ICt+Jg0CxqVg9g0+SVLg0BHg2C7uWBBCxLZClBQgtLDBNQrLXBD1XLQhArOQ4rChHuahBgCrUiuQDQ+rHQ6isCfQ+ixARi0wnrcwti3eAbfReRPrOw8bPi1YASlw86Nw0Sjw8S7w7oNbaxGS3MEdIIkIg7WU5Sz5NSjSuI0OBIiJIIZItvVIwyxGEy1JMynIyyuS-7XaQHYpEHSpUZSo70OGoahGtylGyIgZO4lo1OGejo50hsmyaAPoxkv8fAU40YiEiYmnFsmYxYuYl4M-MnV8VYo+kc5WrYk7R4H5Zie-fY72XIYFb6NYmmjY-FaFF+uMhFGYIjPIVFOkT1F4j5SG84zVGG+eoGX4ka5Ghojyma7DGyag9BsXblGE-leE0K0nCK8kyg1E6ncwzE+KxnHEpKw+NnbVIkrnUkzKvnU1akoXK1fKhku1cq4qkwt1cqr1Kqv1Hk2q4NdXVcIU3Q4zFq2NCU5KKUmUnq5SpBgamo4amfUa9y8a6sitKmKtfoTykEr0XG1slBpax2C0wQ8PE+QdGPHax0-al0lPI6909PT0zPc6nPP0h9AvPdYM+60vMMp6ivc9NeNS29XvBvJ9X619f66EMCIG7Mnuf9fvIDeBrNBAIlUMZBp2bRtU3R5ex2Ca7Un-MTes7GkgUMCxwjQmg-bs4-Psm5O7KOejE4p+um8cpx9jdOacnOVmouQTE8pfdDV5CTXmwA-mxoOTIWvc0W1TDmCW0ZqW9nPTN6OW4zDAxWiie8x2R89WmXAg4+YglzS+L8m+X8w23zeg-zECxRC28Cjgm2qC+2qBOCwTeBF24Q1CsQoGDC8arCvLIhWQ0hfCkra60O8Or2si5haOjhGiuihJ9rezEwrMsw9xNOxDDOobGwp55RXixw-i5woS1w9woxMuySiu3wjbHMexWuhS-bJSs3Zu961urS8JJIvSnugyp7fuzIwe9JL7EengseruooyehyxBsMAp1BnRvBp3EAIEhs0MHy+o-BvHQK2EgVBEkRu+8VShnaep2hpnBhjVFK5htKkklQ9hqKzhnKmynhm1QqgR-NVk4RuXSqxXaqiRoNGNaR02WR6i+RguRR9qyU2HbJtRs3PJgp2otB3yjB-RtGwxqBXUkxrBkEup8+uKxalO5a0PNay0hxu0px0eva50-gZPGdCYY60670i6-xxQw-IvEJ54MJk9CMyJ6vd6mJhM0+eJ4epJ9vDMzvdgEG+M-MrJ1RosvcBNopoGdU0p53NN8ZusrG2a3cU1ltIjIm5p3ssms-SmocrpgB0cxkVjSOJml-WclqtmtZvlDdlcqZ+uGZmTOZ0AgU3cx2pZmAo8p97TaWzZgzWeHZhW1gO85Wo5mzR2OzTWxk989nXW65qKv8x+QC+54Cs2wly20LSC7gqLT5x2+Cn5oQlC92tCjLL2yQ326QsF3C7gIO1t8rO1iOxhcizQ4xKi7XZFlrVFpOzFlO7FyRdOqwrO2w7i+wibUl45QuubEuqlkxcuyxaS2xWSwI5l0I8wuU9lhga9Tl+I7SxI3SzyGJfl9I4yoV97Ie0VpacVgo8eqV+ymNxyhdzR6HVypNzV5V1Vmp3cDV-4pD7V594K4nREw1lEqnE1vNha-Ghnc1lnRhgk613VdKth5Eh17KwXXKl1gq-hjFQR0q+DkRzk318RnjAN+qmRxquR68hR-XSN5R6N1RjxNNLQXcRdxenz-45V8pms93LN4Ezr3dhLs0mxlaux9astraitxz8mat3wQ62dTxk67xs67PX0vPAJvaoJu6pIB677Z6yMqJ-tuvWJ764d+z7JVOQGid7vadzJwfbq+d5CbrvNRV5NgEspgxl9lfbNrQZCMbts7+g99GEmk-Npwczp2OWm2-Xph9ScgZ5moZucx9z-Z9kTCZ5WtcxuWZtubcsA4W5TMWlZoeIDseDZ88rZ8D683ZqDpWizVWvAk5l8s5pzEgy5ntPWzzW5rDgxB53D4LNggjt5ojmCkjmBMjhLCjt2whaj8Q2jzC+j7CxjwrZjyF4OqhNjkiuFyOrjxF3QuO2igT5vIT0RETw5HFx2PFzi7O6T3Okl-OslnRIuylzFal8xWl9TzbTTpl5xRS3Tpuk7Fu2IrlnSyV8zpIXugVxJGznKEVv627yt51qP4oqex2compIld7zz63Hrr73zxoxqZo4ZI-do8ZCdBs5CXo7wfow+4+rZcYuYEH85K+3ea5cmpY+5R5f+iFN5K41+nYj+vYiB3+0Ffv4YqFS4141+m48Bu4yBx4rgz3l+1R9zvPxU74xDBV4ppVnE0xoHoLvY3HaEnVohkKnEA1sh-W6KqhmVY0yxxL+h5Ly1rVKmFhjLu1rLygx13LtPuDVdaFdXqLJEqmyTK4+tfUNVarnTCDZho6uobBruGya7xoWubxdroEFz4fdbcRfPrpqTLQVMM201EbmQGB5xc8aUsCbkDFsZWlS2keObkj2cbx4nSE6Gtit3rZrdG2vjbbldRDqBl22h3UJo9W7YvUoyIAaJhd0HaPom8N3H7ADRSYPcp2GTcGrO1e7Q0tAeEHAYjRXYptfu67bHpuyP5kA8IbfEAPuyaYQ8Wmx7Acqe1h5goL27UMcoJgZr9NOMqPHjOjxGaY8Cchg19v-mmbrkCe8zYnos17jLNYCFPHwVT10w08wOaBCDpgX2YwcWeT5Nnoh3Oa7RSCPPNDpQQw40Fja2HU2h-HNqEhReEFcXnbWI58QvmAhRCklko4K8AWIAIFmUxBb+1wWeFTIFCxDq69VC1WA3gi0oox0+OJvFFubw6wYtLeiNCwuJw4qSdCWPFBwnKDk7TZyW7vJTp7xU40s1OVdDTjXQgR10g+7iPTqHw5bh9jO3LMzu01j5WdJgCfbInILyK7VABE9Vzqo2z48QiUWg-PvDU+779vuyrZ6E0TXrl86YlfVxg2Twh19FkB9KCB4Cb5jEOUZ9ahk-0vqXJO+8xW+rfxWJPIn6g-OfqAxH4uD-k4-I4n-UfqOCZ+IDTskDlTCL92y9xPPNmCeJr8h+G-WViYPlbec8BY1JHMYJIDQjcG33ELufzC56sSG4Ve1hQxi7okKBz-bpElzxIpcrWn-G1hlV-6UkBcNJPKsAMIpFcPW4Ar1hySgHckququOAT+2DaIC+OYbGNKgMbBRsMBvVTQdoKXp6C12hAwbsY2qazVTB8ohVNQJAC0CS2NpRxkwNT4J42By3dxqtxAxeM4yXpHgZdX9I3VBBJeTtiIJO69s3qBnWvJ9Uu6JlruyfeQckw7yYtHuKgwDC9yhoiwtANEN0b1z5GeiUM6bLmr6Oww0QzBFguMsTWsFUZbBqcKmuewhQ9MSRTAtwc-hZpeCP8CBf7pMwCHvsghn7QnoLVCF-twhAHVZtENPKgdLy8tJISvBSGbxWetmU5kQU54XNPyOQn8uh356FDBeOHEoXhxebW1wsVQyXjUNI7fNZeSFP5lR2aGtD9Y7QnChr3kI9CdexFfoeoUN7DCkWYws3kYUmFMUsW1vMTriwk7DYHeWKGTnnQ0Tyd1hinQXspyWw7DK6fhfYYy0OHacG6sbCmPp0M4XD26JnTurdn0rHI+68fZJKZST6JMU+C3WktGmlYxtPhs9WiI2N5F6NOyq9RkevTaI1It61fGpjRBhEN94RlIiFCfRb57IAxluWYpiJvoDk76uIqfoA3OJD8iR79ccQkDJFZIRx0-TYsAwsm0iwGX9WkA8RZGr9uK7I7qu5wbG-CKUPIgEcX0wakDaIJ-SEt5MIaE4r+ItUhtKMpySpYuqIi+ma1f7Kj3+qVdLra1fyaj+cXDPLnSV4Zi4iqhooRjLkgHJAxGMAi0SGlq734RSKAtqmgKNytduqmA+UmQH8nb9BqbSQvsFPwGo0vR6bIbh2JBJdjdJBbLLCGLPL0DNq0eCMQtyjGTJ2BsYzgfGPW6JifGW3FMbtzTHBMhBmY47hE1AHRlzuBY6QT9RHa3cFB5Y4GukzzLPdCyGgkSBJIGnNikMf3PwQD1IHaASA3Yxpr2MPak0BxFNIcWezh5MYr2n9ZHu4LvbDNZxYzb6QuMdh48NyQMAWkSUUwbiDy4tKIQgRA5xD9xiQvZkeOZ4ni0hZ49nhePtFZDue18W8XkPvF0FHxxQ5gs83KGvN3x0FOsFLyWJ1DfmjQ0Qp7WsbATVeoLAOhC26Ha8iKYddjvr045DCeOIwpAE1lN4GEJh6LFCVb1YroTbemEgljnWJYrCXeBEt3kRLErbDveuwiiX7wOE7ZA+LLYPmyzOF5iYiF2S4ZHzYl8sOJcfDItxOFbmU+JP2VPoJLeElERJM9OpH9LenLsSmHokAMCNL6gjWiFfBSVXy6LY0-pqkuEYMUcFaTkRrfSaQl277X1oexkvvhpIcn4pnJfsL5FZOva2TjikMlGWDicmEiXJC-NycimZFoovJsDOFG505ExyApVjKlE2KkkqpjB2c4UZq1FEBVxRxDa-vFLynGs5RKU-NiXISqAkLWYxVURzmykajIqf-HLjqPy58N9Rp0kAJ6wqnesqpFXGqXyUtENUGpzVJqVcGa6tTnR6jUgCPJ6laN+pccg-kNNbFu4fRW7GZH9LMFBiZp9jBgQtJTiRjWBK0mMXWwbYbcm2fjHbq2wEEHSMxZecJj2yvmSCLpcTWQSWPfSKCKxygx6aoJrHD4XpSkWOUjUBEEDQFyMtVL9J9zFzQe+Y2kX2KPagz2mw4luU4OhmuDH8KPeGTOPZpY9KmZknmkuPx4riQhP7EnvuTJ6RD4CJ5QmbLTp5RxIOS8JnjgVSHHMqZGQy8XTOvEMy7++Qo2izIYKPMReVtMLMgHebVC+CTtcjn+KFke10KyvYFuLI6FMdwJMspQnLL14DDFZFFZWfBK4TqyE6aLIwFMOYq9Y9ZQMO3gsKNnLCnCZs4SsXWIlbDSJ1s8ifSwCIB9gixw03PRNdmMSPZzEq4VHxuGWcjK9wgObZ14kt4HOf2Jzun2EkfCo5v88DKPMKaAKWFIUleiCNklgjN6GcwHjoAxx716+uchEU-QLk7Ii5G8+Lk0n0mIYu+2I5YpXNOIEi4Glk3YqSKX4T97JCi94u3NOWdz6R3cpkVA1ZHeS5+HIikNHOGX-yvO48ySauxVbTyMcs84LmfwXlBUJRy8qUavNlFmDFR6UvgkwzVGHzMux8rUQVMAH0kSp7rNWDfLKp3yFc0A-1rVPgGa4mqdo5Dkoy-kxsOpXy5hboJ+4tjJqRjTNmNKGXQLRZwYqbnQLDHltFpPSxbtGNrZukNp3AnaS22ha4KDu+CrttmOIXnTaRX1IseQuDmUK7paTXMmDWrHPS6xOgMfCMsTb-KE5A3NsTjzZV6qAZjI8HuRhBn9kwZtGCGQ4NHEI9rJD+NOHDOnEPtvBc4jhauUCHKLgCX7InmorCG4zye2i9ZrEL0UJD6ehi6DuTPNynj4O54rWlz2sXkE+e1BexSbUYLPjnFYvbmR8y-HS8fxghHxfL2Fn+LOVPtfBBLM6Ga9pZrHKCZViiVR04JxveJeMKQlazk6Mwm3hkoNlcUcJTvE2fhLWHmyRKhSrQsUpWw+89hdsqiQ7MqVOyThIfVSucPqWhIWJPLaPvdl9l3Cco7SxPkHK6X8SBVYclzhHIGUVFo5+qn5QX3+FALWFJGGSd-TklpyxkkIrOWCSWWwjlkqy-Oc30Lk6StllA9vhiL2VYijJOIo5fiM2K1zE59c85Q+ibkUjjljk2fvcrrmuSIGHkvuTA1dUytPlv8u9SXJ36BS-l70yeYCvmVOQIpflKKRfxikRcb+CU+-slMf6pTFS28pHLvKRUHziSR88huiqdaCSsVbrA0biqNG3yTR98oleaKfl1TrRr8ildrQoIdUVG7Ul0XqvpXxzGVn0gwSypIHdEnIHKwtpN2LazTeVjAxBUtOQUHU1p6CraZtx9K7ScFhePBV8iOnHo5V4gkhYqsLFDsVVp60sWO1SaTsHpWqiGnO0YX9V71fw3AVRoBUmr5xnCkzU2R4UE0rVlgm1VD274w9L8VcqGYnBhmM0pFnqguBjx9XyLW5bMf1ejJbhBq1xIanGZosA47jdFF5bZrGsPHmYTFFMsxcmupmpqrxqHRmQbSzV3NWZua9mWUJcWEcPxvM4tfzOdpy8UsAEkWeZrFm1rglYEljtCz6EtqYJSsvMLx1Vn8cNZ3a5JdrL7XpKQAmSrCVJ2HXGzcl46-JR72nVeEyJdLauouvkqOydOq6l2eurdnqUmJW6xpd7Is77rWlh6t7MeuukhyBJtlDPm8VEnRy4tZG3qaMsfXjLBpky5OdMtTngj05X6g0toG8q-q1JeczSUBo2UgbONm8nZR30g2GTbkMGh+uhprkdzsNxIxuZcvJGT8ittWi4jSOw1dzcNvc6Bs8UI1DziNOgTHU0nI1jzo2Rq-Tf53J3eUQVp-fytFPC76sV5aK9jevMZ3bKBYPGlVHxtS7IrBNqK4TflNE26iCul88QXitK4ErqpxKxTaSpDa2jkBtMj+S1M6ptdtNFO3TcAtTbDSwFrKiBWjgp1mbpp3K0MRtXDG2aBVy0hzWgq4EYLkxEq-gR5ulVeaCFog07n21B0DsyFyZVVWmSoX3TNVM7ehTk2jmw0DVS7PHR9JS0cLzV2gcxplvMGAz+FwMvLSe3Bn2DTJl7ErRIvdVTi0eXqxGRzVS1+qlFDWzGQs1a0RD2tBM6ntGqvIGLetBzIGLBw1oOYQuaasbbYuZk5qnFYFTmW+LcUS8ltnimXmWoaEVq-FNHatSBPV6B0teTaiJdBOzDRLuOp2lWWrK7UMVkJvatJf1kHXYSiWOS1YYJQnUFLLZM6qSvOoZZyUjhK66pREVpFh9N1yQbddcPYlxI-Z1nI9Y8IoUOkJWQk94b5OHkt74tFG1XUloTka7IFODbzHg3nl67IVcU6FUbrXlwqLdKrK3fvJ1S26f+Ru--mfKKl6imSxXCAR7ofle66qz8+qeSv92UrP5werTT-J0BMGsdAC3HQyv65fSjNw3Ezb3tA3P8YFSeqzSnr5Vp7rKGetxlntFU57xV2CyVQXoPTCDjpRCvzQqrrlKqgtVekLWqvHbULItDenVbk1-n5NW9Yysw2wuZXLkfpJm3NrYYabZagZVgwRXauEWOrx9YiyfX00kUerZ9lW71UjJq2496twQ79tjMgKbjDy24rfVGq636KbyjPZIQmrVpwcgYCHE-ZkJ1pXNxtVBbzNmqKEzbSh+HCoYWo8W1DVt5a9bU0M23e0v9ksroQRV6HNqOOba2JR2rPLgHE6kB4TrdpgPzDHtiw3Cc7zHVIH3tmwz7RJRKU-bKJWBmiayxqUg66lmlT2aZyaWkG0isOgeh0pPWplmBvSug1eoYPy77o3IyjU+omX8jaN6rbXZFIIZMb9dkomYFFyirCG+98KxKm-z3kf8BNrDaQ-btkPcN5DzuxQ2VJK7DHKphKs0a-lgFKaEBKm7Q2pqpV6GgdHXHQEkeYN9TTDem8w4ZuIFWGs52R03WBtbScrYFM3eBdtSQWuNVpHh4fAmNpFJjvDfAwJrdX8PebwyYgs7uXqkGV7UWUJ26dEbr2g04jMbIstoC67JHxTEe-QVHq72x7f5O7PvT2MH0FHbV0POwYVtOJjjr2sMmfZ4Ln2yLfB9RpfXlA-aBrVxWM39q0bDVaLJafKTrbTxjV77SZfWh8qYqGP7wU1p+0bRMYv2TaBeji4Xjfvm2VCeZvBFY94tf3rHK1H+rbdliCWgSf9jag7QcYVlHGQDcS044hIgM9rLj0B9ipnVuPZLZOpst7RSxeOeE3js6m2WUv97USAdtEjqQxPOwAmGlXs7utDrIMHrwTCOp4TQZhPhzM+ZRQZToFdOimcdiWlE-juklTK31MyiEdvSzmBcqdKy4XSMTp3JgUR8phUbssdj7LoNhyznXBu51YbENb9ZDTZIF12TRF1IhDf7Bw1L88N0utke8vhOEpfTSJ1g++Y+kcG49gXTEwxuxOLzYpTJJEkIdhXEnRDuJRFdbspPf9cpMh0+XSaAEMnSpUm8qfitk1sm-WCm9Q1ybJX1c38EbIPZpsFNYDSLbpt8+3uo2d7LD3ev0zkcDxKmHDcC+aWqbs0anUFIq7U5tN1PbTXNeew0+mKL2yqTpwRi06Qqu7BabTZYu0xqodNPSuqpSA5MDC6AAA1chBZF5ggBjA2gaUkAA)

***

# 부록

## 이어 붙이기 threshold의 근거

1단계에서 얻은 줄별 이어짐 확률은 다음과 같습니다.

```python
print("join  line")
for i, line in enumerate(LINES):
    join = "    " if i == 0 or line["gap"] else f"{result['joins'][i]:.2f}"
    print(f"{join}  {line_id(i)}| {line['text'][:66]}")
```

```
join  line
      L000| Migration to the new build system
      L001| Hi everyone, quick heads up about the build system migration that 
0.97  L002| happening next week. We have been running the new pipeline in shad
0.79  L003| mode for three weeks and the results look solid, so it is time to
0.95  L004| make the switch for real.
      L005| What changes for you
      L006| The old make targets keep working until the end of the month. The 
0.61  L007| entrypoint is a single command that wraps everything, including th
0.95  L008| docs build that used to be separate.
      L009| bun run build
      L010| Generated artifacts no longer need to be committed. The new pipeli
0.82  L011| uploads them to the registry automatically, and checking them in
0.92  L012| just creates merge conflicts.
      L013| The cutover touches three teams, so check whether you are on this
0.92  L014| list before you plan anything for Monday:
0.16  L015| The platform team
0.04  L016| The web client team
0.06  L017| Whoever still owns the release tooling
      L018| Things to do before Monday
      L019| Update your local toolchain to version 2.4 or later
0.02  L020| Delete the old build cache directory
0.02  L021| Run the doctor script and fix anything it flags
      L022| If the doctor script reports a red result on the toolchain check, 
0.89  L023| not proceed with the migration. Ping the infra channel first and w
0.93  L024| will sort it out together.
      L025| As Dana put it in the kickoff, "a migration nobody notices is the
0.94  L026| only kind worth shipping."
      L027| Thanks, and shout if anything looks off.
```

이 메모에서는 확률이 서로 떨어진 두 구간으로 갈립니다. 문장을 갈라놓은 줄바꿈은 0.61
이상이고, 작성자가 의도한 줄바꿈은 0.16 이하입니다. 따라서 여기서는 그 사이 어느 값을
기준으로 잡아도 결과가 같습니다. 그래도 코드는 **앞줄이 어떻게 끝나는지**에 따라 기준값을
정합니다. 이것은 코드가 직접 읽을 수 있는 사실입니다.

* *끝맺지 않은* 줄(문장을 끝내는 문장 부호가 없는 줄) 뒤에서는 문장이 아직 끝나지 않았을 수
  있으므로, 확률이 0.2 이상이면 두 줄을 합치기에 충분합니다.
* *종결* 문장 부호(문장이나 절을 끝내는 문자: `.` `!` `?` `:` `;`) 뒤에서는 기준값이 0.5로
  올라갑니다. `L015| The platform team`은 콜론 뒤에 오는데, 콜론 뒤에는 목록이 올 수도 있고
  문장의 나머지가 올 수도 있습니다. 여기서는 0.16이 나왔습니다. 점수가 더 높게 나왔더라도
  0.5를 넘어야만 목록이 그것을 소개하는 문장에 합쳐집니다.

## 질문이 "같은 문단"이 아니라 "문장 중간"인 이유

이 pipeline의 첫 버전은 누구나 떠올릴 만한 질문을 던졌습니다. "이 두 줄은 같은 문단에
속하는가?" 그런데 제목 아래에 짧은 줄이 이어지는 경우(기호 없이 입력한 목록)는 느슨한
의미에서 문단이 *맞습니다*. 줄들이 한데 모여 있고 주제도 같기 때문입니다. 문단을 기준으로
물으면 모델은 모든 쌍에 "예"라고 답하고, 이어 붙이기 단계는 목록 전체를 긴 블록 하나로
합쳐 버립니다.

```python
def naive_join_question(i: int) -> Noul:
    return Noul(
        instructions=(
            f'Line {line_id(i - 1)} is "{LINES[i - 1]["text"]}" and line {line_id(i)} is '
            f'"{LINES[i]["text"]}". Are lines {line_id(i - 1)} and {line_id(i)} part of the '
            "same paragraph?"
        ),
        criteria=NoulCriteria(
            true="The two lines belong to the same paragraph of running text",
            false="The two lines belong to different paragraphs or different pieces of content",
        ),
    )


naive = stitch(
    tag(LINES, "L"), as_json(stitch_questions(naive_join_question)), model=TYPESAFE_MODEL
)
print(f"{'':14}{'mid-sentence':>13}{'same paragraph':>16}")
for i in (15, 16, 17, 20, 21):
    print(f"{line_id(i)}{'':2}{LINES[i]['text'][:36]:<38}"
          f"{result['joins'][i]:>7.2f}{naive['joins'][i]:>13.2f}")
print(f"\nblocks after merge: {len(blocks)} (mid-sentence) vs "
      f"{len(merge(naive['joins']))} (same paragraph)")
```

```
               mid-sentence  same paragraph
L015  The platform team                        0.16         0.41
L016  The web client team                      0.04         0.61
L017  Whoever still owns the release tooli     0.06         0.63
L020  Delete the old build cache directory     0.02         0.62
L021  Run the doctor script and fix anythi     0.02         0.72

blocks after merge: 17 (mid-sentence) vs 13 (same paragraph)
```

문단을 기준으로 물으면 각 목록에서 첫 항목을 뺀 모든 항목이 0.61에서 0.72를 받습니다.
끝맺지 않은 줄 뒤에 적용하는 기준값 0.2를 훌쩍 넘는 값이어서, 목록마다 블록 하나로
뭉개집니다. "같은
문단"은 주제가 이어지는지를 모델에게 판단하게 하는 질문이고, 목록 항목 사이에서는 주제가
이어집니다. "문장 중간에서 이어지는가"는 텍스트 자체를 묻습니다. 판단 결과가 threshold에
쓰인다면, 질문은 그 판단을 가르는 가장 좁은 사실을 짚어야 합니다. 여기서는 질문 문구에
따라 블록이 17개가 되기도 하고 13개가 되기도 합니다.

## confidence가 가장 낮은 블록

```python
uncertain = min(blocks, key=lambda b: b["confidence"])
print(f'"{uncertain["text"]}"')
print(f"confidence {uncertain['confidence']:.2f}: ", end="")
print(", ".join(f"{k} {v:.2f}" for k, v in
                sorted(uncertain["probabilities"].items(), key=lambda kv: -kv[1])[:3]))
```

```
"The cutover touches three teams, so check whether you are on this list before you plan anything for Monday:"
confidence 0.65: paragraph 0.71, callout 0.25, heading 0.03
```

팀 목록을 소개하는 문장입니다. 문단으로 읽히지만, 행동하기 전에 무언가를 확인하라고 독자에게
알리기도 하는데, 이것은 callout이 하는 일입니다. 확률도 그에 맞게 나뉩니다(paragraph 0.71,
callout 0.25, heading 0.03). UI에서 이 점을 드러낼 수도 있습니다. 예를 들어 유형
confidence(신뢰도)가 0.7 미만인 블록에 밑줄을 쳐서 검토하도록 표시하면, 이 페이지에서는 이
블록 하나만 걸립니다.


이 문서는 개발자 문서 플랫폼인 [Mintlify](https://mintlify.com)로 제작하고 호스팅합니다.
