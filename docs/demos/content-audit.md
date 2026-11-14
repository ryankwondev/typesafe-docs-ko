# Jev로 하는 콘텐츠 감사

> Django 문서 전체를 5초 안에, $0.10로 감사합니다. 어떤 종류의 콘텐츠가 있는지, 무엇을 고쳐야 하는지, 어디에 노력을 들여야 하는지 한눈에 파악할 수 있습니다.

## 문제

대부분의 회사와 프로젝트는 시간이 지나면서 콘텐츠가 쌓이고, 결국 어느 한 사람이 전체를 이해하거나 어디를 개선해야 할지 알 수 없는 지경에 이릅니다. [Django 문서](https://docs.djangoproject.com/en/dev/)는 릴리스 노트를 빼고도 279페이지, 584,234단어입니다. 한 사람이 한 번 읽는 데 약 39시간이 걸립니다. 콘텐츠 종류도 여러 가지이고 오류 종류도 여러 가지이며, 어떤 페이지는 다른 페이지보다 훨씬 중요합니다. 그런데 어느 페이지가 그럴까요?

## Jev가 적합한 이유

Jev는 많은 양의 텍스트를 숫자로 바꿉니다. 수백 개의 규칙을 손으로 작성하는 대신 자연어로 질문하면 됩니다. 이 페이지는 오래되었는가? 튜토리얼인가, 레퍼런스 페이지인가? 초보자용인가, 전문가용인가? Jev는 한 페이지에 대한 이런 질문 수십 개에 요청 한 번으로 답하고 질문마다 확률이나 점수를 반환하므로, 결과를 정렬하고 걸러 내고 그래프로 그릴 수 있습니다.

## 페이지당 요청 한 번

요청 하나에는 페이지 하나와 질문 45개가 들어갑니다.

| 질문 수 | 유형 | 측정하는 것 |
| - | - | - |
| 8 | Score | 품질: 전반, 정확성, 최신성, 명확성, 완결성, 구조, 예제, 맞춤법과 문법. |
| 5 | Score, Choice, Noul | 중요도: 페이지가 얼마나 중요한지, 예상 트래픽, 입문자나 AI 어시스턴트가 읽는지 여부. |
| 7 | Choice | 분류: [Divio](https://docs.divio.com/documentation-system/) 유형, 단계, 복잡도, 길이, 수정에 드는 노력, 누가 고칠 수 있는지, 가장 효과적인 수정 한 가지. |
| 21 | Noul | 문제: 오래된 내용, 틀린 사실, deprecated(지원 중단 예정) API, 오타, 깨진 마크업, 안전하지 않은 예제, 누락된 예제 등. |
| 4 | Noul | quick win(빠른 개선): 작은 수정 하나로 나아지는지, 삭제할 내용이 있는지, 교차 링크가 필요한지. |

Jev는 한 요청의 모든 질문에 병렬로 답하므로 질문이 늘어도 시간은 거의 늘지 않습니다. 테스트 페이지 하나로 측정했을 때 요청 시간의 중앙값은 질문 1개일 때 144 ms, 45개 전부일 때 151 ms였습니다. 다만 질문 텍스트가 입력 token(토큰)으로 계산되므로 비용은 늘어납니다. 여기서는 질문 45개가 \$0.10 가운데 약 40%를 차지합니다. 이 감사에서는 몇 안 되는 페이지에만 해당하는 질문까지 포함해 쓸모 있을 만한 질문을 모두 묻습니다. 질문 수에는 제한이 없고, [context length(컨텍스트 길이)](../models.md#context-length)에만 제한이 있습니다.

## 결과 시각화하기

Jev의 답으로 어떤 대시보드나 시각화든 만들 수 있습니다. 여기서는 각 페이지를 중요도와 품질 격자 위에 찍어, 사람의 손길이 가장 필요한 페이지가 쉽게 보이도록 했습니다. 품질은 낮고 중요도는 높은 왼쪽 위 사분면으로, "Fix first"(먼저 고칠 것)라는 레이블이 붙어 있습니다. 대시보드에는 quick win도 나열됩니다. Jev가 고치기에 trivial 또는 small로 평가한 중요한 페이지들입니다. 무엇을 중요하게 보느냐에 따라 격자에 수정에 드는 노력이나 페이지 복잡도, 그 밖의 어떤 것이든 대신 표시할 수 있습니다.

![감사 대시보드: Django 문서 279페이지 전체의 중요도를 품질과 대비해 표시하고, 옆에 먼저 고칠 페이지 목록을 둔 화면.](https://mintcdn.com/ts-docs/tSHMUPy1QMGjBb5r/images/demos/content-audit/dashboard.webp?fit=max\&auto=format\&n=tSHMUPy1QMGjBb5r\&q=85\&s=ae55a8799b263f5521ba257c73aca191)

## 질문으로 찾아낸 것

문제 질문과 quick win 질문은 모두 예/아니오로 표시되는 플래그입니다. 차트 왼쪽은 각 질문이 279페이지 중 몇 페이지에 플래그를 세웠는지 보여 줍니다. 오른쪽은 분류 질문 네 개에서 페이지가 어떻게 나뉘었는지 보여 줍니다.

Django 페이지 대부분은 중급 개발자를 위한 레퍼런스 또는 설명 페이지이고, 금방 고칠 수 있습니다. 흔한 플래그는 교차 링크나 다이어그램을 추가하거나 전문 용어를 풀어 설명하는 것 같은 작은 일입니다. 심각한 플래그는 훨씬 적은 페이지에서만 나옵니다. 28페이지가 deprecated API를 보여 주고, 18페이지에는 틀린 사실이 있을 수 있으며, 13페이지에는 오래된 내용이 있습니다. 직접 확인해 볼 만큼 짧은 목록입니다.

## 정확한 문장 찾기

감사 화면의 각 페이지에는 deep scan(정밀 스캔) 버튼이 있습니다. 이 버튼은 각 문제가 어느 문장에 있는지 Jev에게 묻고 그 문장을 강조 표시합니다. 여기서는 Django admin 페이지에서 오타가 두 개 있는 문장을 짚어 냅니다. "it's"는 "its"여야 하고, "as"는 "are"여야 합니다.

![Django admin 페이지의 한 단락을 deep scan한 화면. 'At it's simplest list\_filter takes a list or tuple of field names to activate filtering upon, but several more advanced options as available.'라는 문장에 밑줄이 그어져 있고, 수정이 trivial한 오타로 태그되어 있습니다.](https://mintcdn.com/ts-docs/tSHMUPy1QMGjBb5r/images/demos/content-audit/deep-scan.webp?fit=max\&auto=format\&n=tSHMUPy1QMGjBb5r\&q=85\&s=c37c21b23f9c4def1021acab76035ccf)

발견된 항목마다 GitHub의 해당 줄로 가는 링크가 있어서, pull request로 오타를 고칠 수 있습니다.

## 직접 만들기

> [card] [API 키 발급받기](https://console.typesafe.ai/keys)
>
> TypeSafe 콘솔에서 키를 만듭니다.

> [card] [질문 유형](../primitives.md)
>
> Score, Choice, Noul과 각각을 언제 쓰는지 설명합니다.

이 스크립트는 데모의 점수 매기기 단계를 줄인 버전입니다. 질문 45개 중 7개를 묻고, 나머지 38개가 들어갈 자리는 `# ...` 주석으로 표시해 두었습니다. 한 번에 12페이지씩 보내고, 우선순위가 가장 높은 20페이지를 나열합니다. 우선순위 계산은 대시보드보다 단순해서 (1 − 전반 품질) × 중요도이므로, 목록이 대시보드의 "Fix first" 목록과 다를 수 있습니다.

> [accordion] Python으로 작성한 점수 매기기 단계
>
> ```python audit.py theme={null}
> """Score every page of the Django docs with Jev, 12 requests at a time.
>
> git clone --depth 1 https://github.com/django/django.git
> TYPESAFE_API_KEY=... python audit.py
> """
> import json
> import os
> from concurrent.futures import ThreadPoolExecutor
> from pathlib import Path
>
> import requests
>
> API = "https://api.typesafe.ai/v1/systemone"
> KEY = os.environ["TYPESAFE_API_KEY"]
> LEVELS = ["0: Serious problems; urgent editorial attention.", "1: Significant problems; revise soon.",
>           "2: Mixed; targeted edits needed.", "3: Mostly sound; minor edits.", "4: Excellent; leave as is."]
> # 데모의 사이트 컨텍스트에는 최근 Django 릴리스와 기능 목록, 그리고 문서 구성에 대한 설명도 들어 있습니다.
> SITE = ("docs.djangoproject.com: the official documentation for Django, the Python web framework, development "
>         "version. These docs are the source of truth for APIs newer than your knowledge: never call an API, setting, "
>         "date or version scheme incorrect or outdated just because you do not recognize it. Sphinx reStructuredText "
>         "markup (roles, directives, underlined headings) is source markup, NOT an error.")
>
>
> def score(q, levels=LEVELS):
>     return {"type": "score", "instructions": q, "criteria": levels}
>
>
> def noul(q):
>     return {"type": "noul", "instructions": q + " Answer yes only when the page content gives concrete evidence."}
>
>
> def choice(q, options):
>     return {"type": "choice", "instructions": q, "criteria": options}
>
>
> QUESTIONS = {
>     "quality": score("Overall, how high quality is this page as official framework documentation?"),
>     "importance": score(
>         "How important is this page to Django developers? The tutorial, installation, models, views, URLs, "
>         "templates, forms, the ORM, settings, deployment and security are critical; niche contrib apps, "
>         "internals and rarely used backends are minor.",
>         ["0: Negligible; niche or archival.", "1: Minor.", "2: Moderate.",
>          "3: High; many developers rely on it.", "4: Critical; core page most developers read."]),
>     "divio": choice("Which Divio documentation type does this page mainly belong to?", {
>         "tutorial": "Tutorial: a lesson that takes a beginner by the hand through building something.",
>         "how_to": "How-to guide: steps to solve a specific real-world problem for someone who already knows the basics.",
>         "reference": "Reference: dry, complete technical description of APIs, settings, commands or fields.",
>         "explanation": "Explanation: discussion that clarifies concepts, background, design decisions and alternatives."}),
>     "level": choice("What level of Django experience does this page assume?", {
>         "beginner": "New to Django; may be new to web development.",
>         "intermediate": "Has built a Django app and knows models, views, templates and forms.",
>         "advanced": "Experienced; customizes internals, writes backends, tunes performance, or contributes to Django."}),
>     "effort": choice("An editor is going to bring this page to excellent quality, fixing its mistakes and problems. "
>                      "How much work is that?", {
>         "trivial": "Minutes: a few wording, typo or link fixes.", "small": "About an hour: a handful of targeted edits.",
>         "medium": "Half a day: rework sections, refresh examples.", "large": "Days: substantial rewrite or restructuring."}),
>     "outdated": noul("Is part of this page out of date for current Django (6.x) or current Python?"),
>     "typos": noul("Does the prose contain spelling, grammar or punctuation errors? Ignore code and markup."),
>     # ... 데모에서는 38개를 더 묻습니다: 품질 점수 7개, 중요도 질문 4개,
>     # 분류 4개, 문제 플래그 19개, quick win 플래그 4개.
> }
>
>
> def grade(path):
>     body = {"model": "jev-latest", "questions": QUESTIONS,
>             "state": {"site": SITE, "page": str(path), "content": path.read_text()[:105_000]}}
>     r = requests.post(API, headers={"Authorization": f"Bearer {KEY}"}, json=body, timeout=120)
>     r.raise_for_status()
>     return str(path), r.json()["answers"]
>
>
> docs = Path("django/docs")
> pages = sorted(p for p in docs.rglob("*.txt")
>                if p.relative_to(docs).parts[0] != "releases" and p.name != "requirements.txt")
> with ThreadPoolExecutor(12) as pool:
>     results = dict(pool.map(grade, pages))
>
> # 우선순위: 페이지에 손볼 곳이 얼마나 많은지에 페이지가 얼마나 중요한지를 곱한 값. 대시보드 버전은
> # 다른 품질 점수, 예상 트래픽, 들어오는 링크, 문제 플래그도 함께 반영합니다.
> rows = []
> for path, a in results.items():
>     quality, importance = a["quality"]["score"] / 4, a["importance"]["score"] / 4
>     rows.append((100 * (1 - quality) * importance, path, a["divio"]["choice"], a["effort"]["choice"]))
> for priority, path, divio, effort in sorted(rows, reverse=True)[:20]:
>     print(f"{priority:5.1f}  {path}  {divio}  {effort} effort")
> Path("audit.json").write_text(json.dumps(results, indent=1))
> ```

페이지는 [django/django](https://github.com/django/django)의 `ca3f31b` 커밋(2026년 11월 4일) 시점 `docs/` 폴더에서 가져왔으며, Django의 [BSD 라이선스](https://github.com/django/django/blob/main/LICENSE)를 따릅니다. 릴리스 노트는 제외했습니다.


이 문서는 개발자 문서 플랫폼인 [Mintlify](https://mintlify.com)로 제작하고 호스팅합니다.
