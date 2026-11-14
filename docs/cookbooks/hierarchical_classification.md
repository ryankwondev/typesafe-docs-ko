# 계층적 분류

> Jev의 Choice 확률에 병렬 beam search를 적용해, 깊은 특허, 소매 상품, 생의학 계층 구조에서 문서를 분류합니다.

계층 구조를 이루는 데이터는 많습니다. taxonomy(분류 체계), 파일 시스템, 웹사이트, 코드베이스, 조직도, 생물학 온톨로지, LLM 스킬, 모더레이션 정책이 모두 그렇습니다. 계층적 분류는 계층 구조를 root(루트)에서부터 올바른 leaf node(리프 노드)까지 따라 내려가는 작업이고, 도착한 leaf가 최종 분류입니다. root에서 시작해 자식 노드들을 선택지로 하는 System One `Choice` 질문을 던지고, 확률이 가장 높은 자식을 따라가며, leaf에 도달할 때까지 이 과정을 반복합니다(**Greedy Search**, 탐욕 탐색).

System One 질문은 병렬로 실행할 수 있으므로 **Beam Search**(빔 서치)로 여러 경로를 동시에 따라갈 수도 있습니다. 그러면 초반에 한 번 잘못 고른 것이 결과를 좌우하지 않습니다. beam search는 단계마다 모든 경로에 edge(엣지) 확률의 기하 평균인 `product(edge_probabilities) ** (1 / decisions)`로 점수를 매기고, 상위 `K`개만 남긴 뒤 나머지를 가지치기합니다.
단순 곱과 달리 기하 평균은 결정 횟수가 더 많다는 이유만으로 깊은 leaf에 불이익을 주지 않습니다.

노드마다 질문을 하나씩 던지면 분류기를 관찰하고 테스트하기도 쉬워집니다.

* 관측 가능성
  * 오분류가 어느 노드에서 가장 많이 일어나는지 파악할 수 있습니다
  * 각 노드와 edge를 몇 번 지나갔는지 측정할 수 있습니다
* 테스트 용이성
  * 계층 구조를 수정했을 때 분류 성능에 미치는 영향을 단위 테스트로 측정할 수 있습니다

### 이 cookbook에서 사용하는 계층 구조

* **[CPC 2026.05](https://www.cooperativepatentclassification.org/):** 특허의 기술 주제를 넓은 기술 섹션에서 좁은 발명까지 분류합니다. 유럽 특허청과 미국 특허상표청이 발행합니다.
* **[Shopify 2026-02](https://github.com/Shopify/product-taxonomy):** 소매 상품 카테고리를 매장의 상품군에서 구체적인 상품 유형까지 분류합니다. Shopify의 Product Taxonomy(MIT)에서 가져왔습니다.
* **[MeSH 2026](https://www.nlm.nih.gov/mesh/):** 생의학 주제를 넓은 영역에서 구체적인 질환까지 분류합니다. MeSH는 DAG(방향성 비순환 그래프)이므로 descriptor(디스크립터: 주제 표목) 하나가 여러 부모 아래에 나타날 수 있습니다. 여기서는 공식 tree number(트리 번호)마다 별도의 경로를 부여합니다. 출처: U.S. National Library of Medicine. 이 데이터는 MeSH 2026의 2026년 1월 빌드이며 최신 MeSH 데이터가 아닐 수 있습니다.

세 계층 구조의 저작자 표시와 라이선스는 [`DATA_LICENSE.md`](https://github.com/typesafe-ai/typesafe-public-examples/blob/main/cookbooks/hierarchical_classification/DATA_LICENSE.md)에 있습니다.

### 방법

* **Greedy search:** 확률이 가장 높은 자식을 고르고 나머지 대안은 모두 버립니다. 초반에 한 번 실수하면 되돌릴 수 없습니다.
* **Beam search:** 가능성 있는 경로 `K`개를 유지하면서 frontier(탐색 경계)의 모든 노드를 병렬로 분류합니다. 초반 선택이 근소한 차이로 갈렸더라도, 더 아래에서 분명한 결정이 나오면 올바른 경로가 다시 1위로 올라올 수 있습니다. 기하 평균 확률이 가장 높은 경로의 leaf가 최종 분류입니다.
* **System One Choice 질문:** 각 노드는 자식들을 선택지로 하는 `Choice` 질문을 하나 던지고, 답의 확률이 edge 확률이 됩니다. beam의 질문 `K`개는 단계마다 병렬로 실행되므로, beam search도 greedy search와 마찬가지로 단계당 호출 한 라운드면 됩니다.
* **공식:**
  * `path_score = product(edge_probabilities) ** (1 / decisions)`
    * 경로를 가지치기하고 비교하는 데 씁니다
    * 약 10단계보다 깊은 계층 구조에서는 부동소수점 정밀도 손실을 피하도록 `exp(mean(log(probs)))`로 계산합니다
  * `separation = top_path_score / second_path_score`
    * 보고용으로만 쓰고 가지치기에는 쓰지 않습니다
    * `1×`에 가까우면 상위 두 경로가 거의 동률이고, `1×`보다 클수록 승자가 분명합니다

## 준비

Python 3.13 이상이 필요합니다.

```bash
pip install typesafe-sdk ipython 'cooksafe>=0.3.0,<0.4.0'
```

그런 다음 `TYPESAFE_API_KEY`를 설정합니다. 처음 실행하면 세 계층 구조(약 28 MB)를 `datasets/`에 내려받습니다.

아래 수치는 2026-11-03에 `jev-1.14.0`으로 얻은 결과입니다.

## 예제 계층 구조 불러오고 시각화하기

각 taxonomy의 고정된 버전을 내려받아 직계 자식으로 이루어진 트리로 바꾸고, 각 탐색 과정을 정적 SVG로 그립니다.

```python
import html
import os
import shutil
import textwrap
import urllib.request
from collections import defaultdict
from concurrent.futures import ThreadPoolExecutor
from pathlib import Path
from typing import NamedTuple, TypeAlias
from xml.etree import ElementTree
from zipfile import ZipFile

from cooksafe import JsonCache
from IPython.display import Markdown, display
from typesafe_sdk import Choice, RetryPolicy, TypeSafeClient

Tree: TypeAlias = dict[str, "Tree"]


class Report(Markdown):
    """Markdown that renders in a notebook and prints as plain text from a terminal."""

    def __str__(self) -> str:
        return self.data


class Hierarchy(NamedTuple):
    """One query and a complete hierarchy.

    :param slug: filename-safe taxonomy name.
    :param name: display name.
    :param source_url: hierarchy source.
    :param node_count: number of loaded hierarchy nodes.
    :param document: unstructured text classified by Jev.
    :param expected_leaf: expected final classification.
    :param tree: nested direct-child menus.
    """

    slug: str
    name: str
    source_url: str
    node_count: int
    document: str
    expected_leaf: str
    tree: Tree


EXAMPLES_URL = (
    "https://raw.githubusercontent.com/typesafe-ai/typesafe-public-examples/"
    "main/cookbooks/hierarchical_classification"
)
CPC_URL = (
    "https://www.cooperativepatentclassification.org/sites/default/files/"
    "cpc/archive/CPCSchemeXML202605.zip"
)
SHOPIFY_URL = (
    "https://raw.githubusercontent.com/Shopify/product-taxonomy/"
    "v2026-02/dist/en/categories.txt"
)
# NLM은 최신 MeSH 파일을 같은 위치에서 덮어써 갱신하므로, 고정된 2026년 1월 빌드를 따로 호스팅합니다.
MESH_URL = f"{EXAMPLES_URL}/datasets/mesh_descriptors_2026.zip"
MESH_CATEGORIES = {
    "A": "Anatomy",
    "B": "Organisms",
    "C": "Diseases",
    "D": "Chemicals and Drugs",
    "E": "Analytical, Diagnostic and Therapeutic Techniques, and Equipment",
    "F": "Psychiatry and Psychology",
    "G": "Phenomena and Processes",
    "H": "Disciplines and Occupations",
    "I": "Anthropology, Education, Sociology, and Social Phenomena",
    "J": "Technology, Industry, and Agriculture",
    "K": "Humanities",
    "L": "Information Science",
    "M": "Named Groups",
    "N": "Health Care",
    "V": "Publication Characteristics",
    "Z": "Geographicals",
}


def _download(url: str, path: Path) -> Path:
    """Download a pinned dataset once.

    :param url: pinned dataset URL.
    :param path: local cache path.
    :returns: local dataset path.
    """

    if path.exists():
        return path
    path.parent.mkdir(parents=True, exist_ok=True)
    temporary_path: Path = path.with_suffix(path.suffix + ".tmp")
    request = urllib.request.Request(
        url, headers={"User-Agent": "typesafe-taxonomy/1.0"}
    )
    with urllib.request.urlopen(request, timeout=120) as response:
        with temporary_path.open("wb") as file:
            shutil.copyfileobj(response, file)
    temporary_path.replace(path)
    return path


def _insert(tree: Tree, path: tuple[str, ...]) -> None:
    subtree_value: Tree = tree
    for label in path:
        subtree_value = subtree_value.setdefault(label, {})


def _cpc_title(item: ElementTree.Element) -> str:
    class_title: ElementTree.Element | None = item.find("class-title")
    if class_title is None:
        return ""
    return " ".join(" ".join(class_title.itertext()).split())


def _load_cpc(path: Path) -> tuple[Tree, int]:
    titles: dict[str, str] = {}
    levels: dict[str, int] = {}
    parent_by_symbol: dict[str, str] = {}
    children_by_symbol: defaultdict[str, list[str]] = defaultdict(list)

    def visit(item: ElementTree.Element, parent_symbol: str | None) -> None:
        symbol: str | None = item.findtext("classification-symbol")
        next_parent: str | None = parent_symbol
        if symbol:
            title: str = _cpc_title(item)
            if title:
                titles[symbol] = title
            levels[symbol] = min(levels.get(symbol, 99), int(item.attrib["level"]))
            if (
                parent_symbol
                and parent_symbol != symbol
                and symbol not in parent_by_symbol
            ):
                parent_by_symbol[symbol] = parent_symbol
                children_by_symbol[parent_symbol].append(symbol)
            next_parent = symbol
        for child in item.findall("classification-item"):
            visit(child, next_parent)

    with ZipFile(path) as zip_file:
        names = sorted(
            name
            for name in zip_file.namelist()
            if name.startswith("cpc-scheme-") and name.endswith(".xml")
        )
        for name in names:
            root = ElementTree.fromstring(zip_file.read(name))
            for item in root.findall("classification-item"):
                visit(item, None)

    labels: dict[str, str] = {
        symbol: f"{symbol} {titles.get(symbol, '')}".strip() for symbol in levels
    }

    def build(symbol: str) -> Tree:
        return {
            labels[child]: build(child) for child in children_by_symbol.get(symbol, [])
        }

    root_symbols: list[str] = sorted(
        symbol for symbol, level in levels.items() if level == 2
    )
    tree: Tree = {labels[symbol]: build(symbol) for symbol in root_symbols}
    return tree, len(labels)


def _load_shopify(path: Path) -> tuple[Tree, int]:
    tree: Tree = {}
    category_count: int = 0
    for line in path.read_text().splitlines():
        if not line or line.startswith("#"):
            continue
        _, path_text = line.split(" : ", maxsplit=1)
        category_path: tuple[str, ...] = tuple(path_text.strip().split(" > "))
        _insert(tree, category_path)
        category_count += 1
    return tree, category_count


def _load_mesh(path: Path) -> tuple[Tree, int]:
    """Load every official MeSH tree-number path.

    A descriptor may have multiple tree numbers because MeSH is a DAG. Expanding
    those positions into paths makes it usable by the tree-oriented beam search.

    :param path: MeSH descriptor XML ZIP.
    :returns: expanded tree and position count.
    """

    with ZipFile(path) as zip_file:
        root: ElementTree.Element = ElementTree.fromstring(
            zip_file.read("desc2026.xml")
        )
    names_by_tree_number: dict[str, str] = {
        tree_number.text: descriptor_record.findtext("DescriptorName/String", "")
        for descriptor_record in root.findall("DescriptorRecord")
        for tree_number in descriptor_record.findall("TreeNumberList/TreeNumber")
        if tree_number.text
    }
    tree: Tree = {}
    for tree_number in sorted(names_by_tree_number):
        parts: list[str] = tree_number.split(".")
        prefixes: list[str] = [
            ".".join(parts[:index]) for index in range(1, len(parts) + 1)
        ]
        category_code: str = tree_number[0]
        category_path: tuple[str, ...] = (
            f"{category_code} {MESH_CATEGORIES[category_code]}",
            *(f"{prefix} {names_by_tree_number[prefix]}" for prefix in prefixes),
        )
        _insert(tree, category_path)
    position_count: int = len(names_by_tree_number) + len(tree)
    return tree, position_count


def load_hierarchies(data_directory: Path = Path("datasets")) -> tuple[Hierarchy, ...]:
    """Load the three public taxonomies.

    :param data_directory: cache directory for the pinned raw files.
    :returns: CPC, Shopify, and MeSH examples.
    """

    cpc_tree, cpc_nodes = _load_cpc(
        _download(CPC_URL, data_directory / "CPCSchemeXML202605.zip")
    )
    shopify_tree, shopify_nodes = _load_shopify(
        _download(SHOPIFY_URL, data_directory / "shopify_categories_2026-02.txt")
    )
    mesh_tree, mesh_nodes = _load_mesh(
        _download(MESH_URL, data_directory / "mesh_descriptors_2026.zip")
    )
    return (
        Hierarchy(
            slug="cpc",
            name="CPC patents",
            source_url=CPC_URL,
            node_count=cpc_nodes,
            document=(
                "Patent abstract: a freestanding structural wooden perch for poultry or "
                "pet birds. The elevated roost has crossbars sized for bird feet and mounts "
                "inside an aviary."
            ),
            expected_leaf="A01K31/12 Perches for poultry or birds, e.g. roosts",
            tree=cpc_tree,
        ),
        Hierarchy(
            slug="shopify",
            name="Shopify products",
            source_url=SHOPIFY_URL,
            node_count=shopify_nodes,
            document=(
                "Furniture listing: a wall-mounted window shelf bed. This padded floating shelf "
                "uses suction cups and a washable cushion as a sunny perch for one cat."
            ),
            expected_leaf="Cat Window Beds & Perches",
            tree=shopify_tree,
        ),
        Hierarchy(
            slug="mesh",
            name="MeSH biomedical subjects",
            source_url=MESH_URL,
            node_count=mesh_nodes,
            document=(
                "Clinical abstract: Crohn disease with transmural ileocolonic inflammation, "
                "skip lesions, abdominal pain, and chronic diarrhea. Colonoscopy showed "
                "cobblestoning and biopsy found noncaseating granulomas; treatment with "
                "infliximab produced remission."
            ),
            expected_leaf="C06.405.469.432.500 Crohn Disease",
            tree=mesh_tree,
        ),
    )


NODE_W, NODE_H = 300, 38
COL_W, ROW_H = 360, 50
PAD_X = 28
EDGE_TOP_K = 5


def subtree(tree: Tree, path: tuple[str, ...]) -> Tree:
    """Return the direct-child menu below ``path``.

    :param tree: taxonomy root.
    :param path: path from the taxonomy root.
    :returns: child mapping at the path.
    """

    subtree_value: Tree = tree
    for label in path:
        subtree_value = subtree_value[label]
    return subtree_value


def _escape(value: object) -> str:
    return html.escape(str(value), quote=True)


def _truncate(value: str, length: int = 33) -> str:
    return value if len(value) <= length else value[: length - 1] + "…"


def _build_nodes(hierarchy: Hierarchy, result: dict) -> dict:
    records: dict[tuple[str, ...], dict] = {
        tuple(record["parent"]): record for record in result["records"]
    }
    best_path: tuple[str, ...] = tuple(result["beam"][0]["path"])
    greedy_path: tuple[str, ...] = tuple(result["greedy"]["path"])
    retained: set[tuple[str, ...]] = {tuple(path) for path in result["retained_paths"]}

    def grow(path: tuple[str, ...]) -> list[dict]:
        record: dict | None = records.get(path)
        if record is None:
            return []
        children: list[dict] = []
        probabilities: dict[str, float] = record["probabilities"]
        ranked: list[tuple[str, float]] = sorted(
            probabilities.items(), key=lambda item: item[1], reverse=True
        )
        shown_labels: set[str] = {
            label for label, p in ranked[:EDGE_TOP_K] if round(p, 2) > 0
        }
        shown_labels.update(
            label
            for label, _ in ranked
            if path + (label,) in retained
            or path + (label,) == best_path[: len(path) + 1]
            or path + (label,) == greedy_path[: len(path) + 1]
        )
        for label, probability in ranked:
            if label not in shown_labels:
                continue
            child_path: tuple[str, ...] = path + (label,)
            on_best_path: bool = child_path == best_path[: len(child_path)]
            on_greedy_path: bool = child_path == greedy_path[: len(child_path)]
            kind: str = (
                "winner"
                if on_best_path
                else "greedy"
                if on_greedy_path
                else "beam"
                if child_path in retained
                else "alt"
            )
            children.append(
                {
                    "label": label,
                    "probability": probability,
                    "decision": len(probabilities) > 1,
                    "kind": kind,
                    "children": grow(child_path),
                }
            )
        return children

    return {
        "label": hierarchy.name,
        "probability": None,
        "kind": "root",
        "children": grow(()),
    }


def _layout(root: dict) -> tuple[int, int]:
    rows: list[int] = [0]
    maximum_depth: list[int] = [0]

    def walk(node: dict, depth: int) -> None:
        node["depth"] = depth
        maximum_depth[0] = max(maximum_depth[0], depth)
        if node["children"]:
            for child in node["children"]:
                walk(child, depth + 1)
            node["row"] = (node["children"][0]["row"] + node["children"][-1]["row"]) / 2
        else:
            node["row"] = rows[0]
            rows[0] += 1

    walk(root, 0)
    return maximum_depth[0], rows[0]


def render_svg(hierarchy: Hierarchy, result: dict, path: Path) -> None:
    """Write a standalone SVG of the beam-search traversal, drawn left to right.

    :param hierarchy: taxonomy demonstration.
    :param result: beam-search result from the notebook.
    :param path: output SVG path.
    """

    root: dict = _build_nodes(hierarchy, result)
    maximum_depth, row_count = _layout(root)
    document_lines: list[str] = textwrap.wrap(
        hierarchy.document,
        width=105,
        break_long_words=False,
        break_on_hyphens=False,
    ) or [""]
    document_y: int = 124
    greedy_y: int = document_y + (len(document_lines) - 1) * 21 + 34
    beam_y: int = greedy_y + 25
    method_y: int = beam_y + 29
    legend_y: int = method_y + 23
    header_height: int = legend_y + 32
    width: int = PAD_X * 2 + maximum_depth * COL_W + NODE_W
    height: int = header_height + max(row_count, 1) * ROW_H + 34
    edges: list[str] = []
    nodes: list[str] = []

    def node_x(node: dict) -> float:
        return PAD_X + node["depth"] * COL_W

    def node_y(node: dict) -> float:
        return header_height + node["row"] * ROW_H

    def walk(node: dict) -> None:
        x_value, y_value = node_x(node), node_y(node)
        for child in node["children"]:
            child_x, child_y = node_x(child), node_y(child)
            x1, y1 = x_value + NODE_W, y_value + NODE_H / 2
            x2, y2 = child_x, child_y + NODE_H / 2
            bend: float = COL_W * 0.38
            edges.append(
                f'<path class="edge {child["kind"]}" '
                f'd="M{x1:.0f},{y1:.0f} C{x1 + bend:.0f},{y1:.0f} '
                f'{x2 - bend:.0f},{y2:.0f} {x2:.0f},{y2:.0f}"/>'
            )
            if child["decision"]:
                edges.append(
                    f'<text class="prob" x="{x2 - 7:.0f}" y="{y2 - 5:.0f}" '
                    f'text-anchor="end">{child["probability"]:.2f}</text>'
                )
            walk(child)

        kind: str = node["kind"]
        label: str = _truncate(node["label"], 36)
        nodes.append(
            f'<g class="node {kind}"><title>{_escape(node["label"])}</title>'
            f'<rect x="{x_value:.0f}" y="{y_value:.0f}" width="{NODE_W}" '
            f'height="{NODE_H}" rx="7"/>'
            f'<text x="{x_value + 11:.0f}" y="{y_value + 24:.0f}">'
            f"{_escape(label)}</text></g>"
        )

    walk(root)
    best: dict = result["beam"][0]
    best_path: tuple[str, ...] = tuple(best["path"])
    greedy_path: tuple[str, ...] = tuple(result["greedy"]["path"])
    beam_leaf: str = best_path[-1] if best_path else "no leaf"
    greedy_leaf: str = greedy_path[-1] if greedy_path else "no leaf"
    beam_width: int = result["beam_width"]
    separation_ratio: float = result["separation_ratio"]
    document_text: str = "".join(
        f'<text class="document" x="24" y="{document_y + index * 21}">'
        f"{_escape(line)}</text>"
        for index, line in enumerate(document_lines)
    )
    legend_text: str = (
        "green = greedy and beam (same route)"
        if greedy_path == best_path
        else "orange = greedy · green = beam winner"
    ) + " · purple = retained beam · dashed = pruned"
    separation_text: str = (
        ">999×" if separation_ratio > 999 else f"{separation_ratio:.2f}×"
    )

    svg: str = f'''<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 {width} {height}"
  width="{width}" height="{height}" role="img" aria-label="{_escape(hierarchy.name)} taxonomy beam search">
<style>
  .bg {{ fill:#f6f7fb }}
  text {{ font-family:ui-monospace,"SF Mono",Menlo,Consolas,monospace }}
  .eyebrow {{ font-size:14px; font-weight:700; letter-spacing:1.2px; fill:#4f46e5 }}
  .title {{ font:700 28px system-ui,-apple-system,"Segoe UI",sans-serif; fill:#181b28 }}
  .document-label {{ font:700 12px system-ui,-apple-system,"Segoe UI",sans-serif; letter-spacing:1px; fill:#777c91 }}
  .document {{ font:500 17px system-ui,-apple-system,"Segoe UI",sans-serif; fill:#303449 }}
  .copy {{ font-size:14px; fill:#5c6178 }}
  .result {{ font-size:14px; font-weight:700 }}
  .greedy-result {{ fill:#c2410c }}
  .beam-result {{ fill:#15803d }}
  .edge {{ fill:none; stroke:#c9cee0; stroke-width:2 }}
  .edge.winner {{ stroke:#15803d; stroke-width:2.5 }}
  .edge.greedy {{ stroke:#ea580c; stroke-width:2.5 }}
  .edge.beam {{ stroke:#4f46e5; stroke-width:2.2 }}
  .edge.alt {{ opacity:.48 }}
  .prob {{ font-size:12px; font-weight:600; fill:#5c6178 }}
  .node rect {{ stroke-width:1.7 }}
  .node text {{ font-size:13px }}
  .node.root rect {{ fill:#f0f2f8; stroke:#e2e5f0 }}
  .node.root text {{ fill:#5c6178 }}
  .node.winner rect {{ fill:#e4f5ea; stroke:#15803d }}
  .node.winner text {{ fill:#15803d; font-weight:700 }}
  .node.greedy rect {{ fill:#fff0e8; stroke:#ea580c }}
  .node.greedy text {{ fill:#c2410c; font-weight:700 }}
  .node.beam rect {{ fill:#ecebfd; stroke:#4f46e5 }}
  .node.beam text {{ fill:#181b28 }}
  .node.alt rect {{ fill:#fff; stroke:#e2e5f0; stroke-dasharray:3 3 }}
  .node.alt text {{ fill:#5c6178 }}
</style>
<rect class="bg" width="{width}" height="{height}" rx="14"/>
<text class="eyebrow" x="24" y="32">{hierarchy.node_count:,} NODES</text>
<text class="title" x="24" y="69">Greedy vs parallel beam search</text>
<text class="document-label" x="24" y="99">DOCUMENT</text>
{document_text}
<text class="result greedy-result" x="24" y="{greedy_y}">GREEDY TOP-1 → {_escape(_truncate(greedy_leaf, 105))}</text>
<text class="result beam-result" x="24" y="{beam_y}">BEAM K={beam_width} → {_escape(_truncate(beam_leaf, 105))}</text>
<text class="copy" x="24" y="{method_y}">separation (top ÷ second path score) = {separation_text}</text>
<text class="copy" x="24" y="{legend_y}">{legend_text}</text>
{"".join(edges)}{"".join(nodes)}
</svg>'''
    path.write_text(svg)
```

## Greedy search와 beam search 구현하기

각 노드의 자식들이 `Choice` 질문 하나가 됩니다. 두 전략 모두 받은 분포를 전부 기록하므로, 다이어그램에서 edge마다 확률을 표시할 수 있습니다.

```python
HIERARCHIES = load_hierarchies()
TYPESAFE_MODEL = "jev-1.14.0"  # 이 페이지에서 쓰는 모델. "jev-latest"는 최신 모델을 실시간으로 실행합니다
BEAM_WIDTH, MAX_DEPTH, EPSILON = 3, 12, 1e-9
client = TypeSafeClient(
    api_key=os.environ.get("TYPESAFE_API_KEY", "cache-only"),
    retry=RetryPolicy(max_retries=5, backoff_initial=1.0, backoff_max=20.0),
)
CACHE = Path("hierarchical_classification_cache.json")
# True이면 공개된 답으로 CACHE를 채우므로 실행 중에 요청을 보내지 않습니다. 실시간으로
# 실행하려면 False로 바꾸고 CACHE를 삭제하세요. 디스크에 이미 있는 CACHE는 덮어쓰지 않습니다.
DOWNLOAD_CACHE = True
if DOWNLOAD_CACHE:
    _download(f"{EXAMPLES_URL}/{CACHE.name}", CACHE)
json_cache = JsonCache(CACHE)


@json_cache
def choose(state: str, question: Choice, model: str) -> dict[str, float]:
    """Ask one atomic direct-child question and return its distribution."""
    keys = question.criteria
    if len(keys) == 1:
        return {label: 1.0 for label in keys.values()}
    response = client.system_one(
        state=state, questions={"child": question}, model=model
    )
    probabilities = response.answers["child"].probabilities
    return {label: probabilities[key] for key, label in keys.items()}


def child_question(labels: tuple[str, ...]) -> Choice:
    """Build the direct-child Choice; its option keys map back to the labels."""
    keys = {f"c{i}": label for i, label in enumerate(labels)}
    return Choice(
        instructions="Which direct child category best matches this document?",
        criteria=keys,
    )


def extend_candidate(
    candidate: dict, label: str, probabilities: dict[str, float]
) -> dict:
    """Append one edge and recompute its geometric-mean path score."""
    is_decision: bool = len(probabilities) > 1
    # 아주 깊은 트리에서는 부동소수점 정밀도 손실을 피하도록 로그 공간에서 계산하세요.
    probability_product: float = candidate["probability_product"] * (
        max(probabilities[label], EPSILON) if is_decision else 1.0
    )
    decision_count: int = candidate["decision_count"] + is_decision
    return {
        "path": candidate["path"] + (label,),
        "probability_product": probability_product,
        "decision_count": decision_count,
        "score": probability_product ** (1 / decision_count) if decision_count else 1.0,
    }


def choice_record(path: tuple[str, ...], probabilities: dict[str, float]) -> dict:
    """Package one sibling decision for the traversal diagram."""
    return {"parent": path, "probabilities": probabilities}


def beam_search(hierarchy: Hierarchy) -> dict:
    """Parallel width-three beam search using geometric-mean probability."""
    beam = [{"path": (), "probability_product": 1.0, "decision_count": 0, "score": 1.0}]
    records, retained_paths = [], {()}

    for _ in range(MAX_DEPTH):
        expandable = [
            candidate
            for candidate in beam
            if subtree(hierarchy.tree, candidate["path"])
        ]
        finished = [
            candidate
            for candidate in beam
            if not subtree(hierarchy.tree, candidate["path"])
        ]
        if not expandable:
            break
        with ThreadPoolExecutor(max_workers=BEAM_WIDTH) as executor:
            distributions = list(
                executor.map(
                    lambda candidate: choose(
                        hierarchy.document,
                        child_question(tuple(subtree(hierarchy.tree, candidate["path"]))),
                        model=TYPESAFE_MODEL,
                    ),
                    expandable,
                )
            )

        expanded = []
        round_records = []
        for candidate, probabilities in zip(expandable, distributions, strict=True):
            round_records.append(choice_record(candidate["path"], probabilities))
            candidate_expanded = []
            for label in probabilities:
                candidate_expanded.append(
                    extend_candidate(candidate, label, probabilities)
                )
            expanded.extend(candidate_expanded)
        beam = sorted(
            finished + expanded,
            key=lambda candidate: candidate["score"],
            reverse=True,
        )[:BEAM_WIDTH]
        retained_paths.update(candidate["path"] for candidate in beam)
        records.extend(round_records)

    beam = sorted(beam, key=lambda candidate: candidate["score"], reverse=True)
    return {
        "beam": beam,
        "records": records,
        "retained_paths": sorted(retained_paths, key=lambda path: (len(path), path)),
    }


def greedy_search(hierarchy: Hierarchy) -> dict:
    """Follow only the locally highest-probability child."""
    path, probability_product, decision_count, records = (), 1.0, 0, []
    for _ in range(MAX_DEPTH):
        labels = tuple(subtree(hierarchy.tree, path))
        if not labels:
            break
        probabilities = choose(hierarchy.document, child_question(labels), model=TYPESAFE_MODEL)
        records.append(choice_record(path, probabilities))
        label = max(probabilities, key=probabilities.get)
        if len(probabilities) > 1:
            probability_product *= max(probabilities[label], EPSILON)
            decision_count += 1
        path += (label,)
    score: float = (
        probability_product ** (1 / decision_count) if decision_count else 1.0
    )
    return {"path": path, "score": score, "records": records}


def compare_searches(hierarchy: Hierarchy) -> dict:
    """Run beam and greedy, then merge their queried nodes for rendering."""
    result = beam_search(hierarchy)
    greedy = greedy_search(hierarchy)
    recorded_paths = {tuple(record["parent"]) for record in result["records"]}
    result["records"].extend(
        record
        for record in greedy["records"]
        if tuple(record["parent"]) not in recorded_paths
    )
    result["greedy"] = greedy
    result["beam_width"] = BEAM_WIDTH
    top_score: float = result["beam"][0]["score"]
    second_score: float = result["beam"][1]["score"]
    result["separation_ratio"] = top_score / max(second_score, EPSILON)
    return result
```

## 두 방법 비교하기

라벨이 붙은 예제 세 개에 두 전략을 모두 실행하고, 각 전략이 도달한 leaf를 기대한 분류와 비교한 다음, 탐색한 경로를 시각화합니다.

```python
with ThreadPoolExecutor(max_workers=len(HIERARCHIES)) as executor:
    results = list(executor.map(compare_searches, HIERARCHIES))

rows: list[dict[str, str | int | bool]] = []
for hierarchy, result in zip(HIERARCHIES, results, strict=True):
    svg_path: Path = Path(f"{hierarchy.slug}_tree.svg")
    render_svg(hierarchy, result, svg_path)
    beam_path: tuple[str, ...] = tuple(result["beam"][0]["path"])
    greedy_path: tuple[str, ...] = tuple(result["greedy"]["path"])
    beam_leaf: str = beam_path[-1]
    greedy_leaf: str = greedy_path[-1]
    rows.append(
        {
            "hierarchy": hierarchy.name,
            "nodes": hierarchy.node_count,
            "expected leaf": hierarchy.expected_leaf,
            "greedy leaf": greedy_leaf,
            "beam K=3 leaf": beam_leaf,
            "greedy correct": greedy_leaf == hierarchy.expected_leaf,
            "beam correct": beam_leaf == hierarchy.expected_leaf,
            "mean p": f"{result['beam'][0]['score']:.2f}",
            "top/second": f"{result['separation_ratio']:.2f}×",
        }
    )

greedy_correct_count: int = sum(bool(row["greedy correct"]) for row in rows)
beam_correct_count: int = sum(bool(row["beam correct"]) for row in rows)
recovered_names: str = ", ".join(
    str(row["hierarchy"])
    for row in rows
    if not row["greedy correct"] and row["beam correct"]
)
table_lines: list[str] = [
    "| Hierarchy | Expected leaf | Greedy leaf | Beam K=3 leaf | Greedy correct | Beam correct |",
    "| --- | --- | --- | --- | --- | --- |",
]
table_lines.extend(
    "| "
    + " | ".join(
        (
            str(row["hierarchy"]),
            str(row["expected leaf"]),
            str(row["greedy leaf"]),
            str(row["beam K=3 leaf"]),
            "yes" if row["greedy correct"] else "no",
            "yes" if row["beam correct"] else "no",
        )
    )
    + " |"
    for row in rows
)
display(
    Report(
        "## Results\n\n"
        f"Beam search matched {beam_correct_count} of {len(rows)} expected leaves; "
        f"greedy search matched {greedy_correct_count} of {len(rows)}. "
        f"Keeping three paths recovered the expected classification for {recovered_names}.\n\n"
        + "\n".join(table_lines)
        + "\n\nThe diagrams show why the methods differ.\n\n"
        + "\n\n".join(
            f"### {hierarchy.name}\n\n![]({hierarchy.slug}_tree.svg)"
            for hierarchy in HIERARCHIES
        )
    )
)
```

## 결과

Beam search는 기대한 leaf 3개 중 3개를 맞혔고, greedy search는 3개 중 2개를 맞혔습니다. 경로 세 개를 유지한 덕분에 Shopify products에서 기대한 분류를 되찾았습니다.

| 계층 구조 | 기대한 leaf | Greedy leaf | Beam K=3 leaf | Greedy 정답 여부 | Beam 정답 여부 |
| - | - | - | - | - | - |
| CPC patents | A01K31/12 Perches for poultry or birds, e.g. roosts | A01K31/12 Perches for poultry or birds, e.g. roosts | A01K31/12 Perches for poultry or birds, e.g. roosts | yes | yes |
| Shopify products | Cat Window Beds & Perches | Pet Chairs | Cat Window Beds & Perches | no | yes |
| MeSH biomedical subjects | C06.405.469.432.500 Crohn Disease | C06.405.469.432.500 Crohn Disease | C06.405.469.432.500 Crohn Disease | yes | yes |

다이어그램을 보면 두 방법이 왜 다른 결과를 내는지 알 수 있습니다.

### CPC patents(CPC 특허)

![](https://mintcdn.com/ts-docs/SMRVJ3B8F9ZiCWm4/cookbooks/hierarchical_classification/cpc_tree.svg?fit=max\&auto=format\&n=SMRVJ3B8F9ZiCWm4\&q=85\&s=fc414461a508f682a2466c1fc1def0a5)

### Shopify products(Shopify 상품)

![](https://mintcdn.com/ts-docs/SMRVJ3B8F9ZiCWm4/cookbooks/hierarchical_classification/shopify_tree.svg?fit=max\&auto=format\&n=SMRVJ3B8F9ZiCWm4\&q=85\&s=f0d83cdd85ec85d1645b96885b38a310)

### MeSH biomedical subjects(MeSH 생의학 주제)

![](https://mintcdn.com/ts-docs/SMRVJ3B8F9ZiCWm4/cookbooks/hierarchical_classification/mesh_tree.svg?fit=max\&auto=format\&n=SMRVJ3B8F9ZiCWm4\&q=85\&s=1df35485dd5a41a8541d016417a66e57)


이 문서는 개발자 문서 플랫폼인 [Mintlify](https://mintlify.com)로 제작하고 호스팅합니다.
