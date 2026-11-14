# Cookbook

> 질문 몇 개로 끝나는 예제부터 완전한 pipeline까지, 실제 문제에 TypeSafe를 적용하는 과정을 처음부터 끝까지 보여 주는 레시피입니다.

cookbook(예제 모음)은 하나하나가 풀이 과정을 모두 담은 예제입니다. 실제 데이터셋, 그 데이터에 대해 무언가를 결정하는 System One 질문, 그리고 그 결정을 동작하는 시스템으로 바꾸는 코드로 이루어집니다. [primitive(프리미티브: 기본 구성 요소)](primitives.md)와 [패턴](patterns.md)이 구체적인 문제에서 어떻게 맞물리는지 보고 싶을 때 읽어 보거나, 하나를 복사해 직접 만들 시스템의 출발점으로 삼으세요.

이 섹션은 [TypeSafe primitive](primitives.md)를 알고 있고 [confidence(신뢰도)가 어떻게 동작하는지](confidence.md) 이해하고 있다고 가정합니다. 그렇지 않다면 두 문서를 먼저 읽으세요.

각 cookbook은 해결하는 문제 아래에 한 번씩만 실려 있습니다. 다른 섹션의 기법을 함께 쓰는 cookbook도 많습니다.

## LLM과 에이전트

Jev를 LLM이나 에이전트와 함께 사용합니다. 도구와 스킬을 고르고, 들어가고 나오는 내용을 선별하고, 돌아온 결과를 확인합니다.

| Cookbook | 하는 일 | 난이도 |
| - | - | - |
| [Function calling(함수 호출)](cookbooks/function_calling.md) | 함수 이름과 닫힌 집합의 인자를 confidence를 고려하는 System One 질문에 대응시켜, 자연어로 된 트레이딩 요청을 평범한 타입 지정 함수 호출로 바꿉니다. | 중급 |
| [스킬 추천](cookbooks/skill_suggestion.md) | Nous Research의 Hermes 카탈로그에 있는 스킬 194개 중에서 에이전트의 한 턴에 쓸 스킬을 최대 하나 고릅니다. TypeSafe 요청 두 번으로 순위를 매기고 상위 후보를 다시 확인합니다. | 중급 |
| [LLM용 guardrail(가드레일)](cookbooks/llm_guardrails.md) | LLM 앱에 들어가고 나오는 모든 메시지를 TypeSafe 요청 한 번으로 선별한 다음, 직접 정한 threshold(임계값)에 따라 통과시키거나, 검토하거나, 차단하거나, 고객 지원으로 보냅니다. | 중급 |
| [인용 재확인](cookbooks/citation_check.md) | 원본 문서와 대조해 틀렸거나 hallucination(환각)으로 만들어진 인용을 잡아냅니다. Choice 질문 하나가 인용문의 맥락이 주장을 뒷받침하는지 판정합니다. | 초급 |
| [LLM 답변을 출처와 대조해 검증하기](cookbooks/verify_llm_outputs.md) | 한 요청에 Noul 질문 세 개를 넣어 LLM의 답변이 주어진 문단에서 도출되는지 확인하고, 의심스러운 답변은 검토로 보냅니다. | 초급 |
| [RAG passage(문단) 분류](cookbooks/classifying_rag_passages.md) | 검색된 passage마다 TypeSafe 요청 한 번으로 점수를 매긴 다음, 어떤 passage를 답변 모델에 전달할지 코드에서 결정합니다. | 중급 |
| [간단한 메모리 시스템](cookbooks/basic_memory.md) | 어떤 요청과 답을 기억할 가치가 있는지 Jev가 판단하게 한 다음, 이후 호출에 기억을 추가하는 작은 함수로 Jev 호출을 감쌉니다. | 초급 |

## 분류

깊이에 상관없이 입력을 카테고리에 배정합니다.

| Cookbook | 하는 일 | 난이도 |
| - | - | - |
| [계층적 분류](cookbooks/hierarchical_classification.md) | Jev의 Choice 확률에 병렬 beam search(빔 서치)를 적용해 특허, 소매 상품, 생의학 분야의 깊은 계층 구조에서 문서를 분류합니다. | 중급 |
| [Confidence를 활용한 분류](cookbooks/classification_using_confidence.md) | SEC 연차 보고서를 보고서당 Choice 하나로 75개 산업 그룹에 분류한 다음, 답 자체의 confidence를 읽어 그 그룹을 보고할지 그 위의 더 넓은 부문을 보고할지 결정합니다. | 초급 |
| [Twitch 채팅 필터](cookbooks/twitch_chat_filter.md) | Twitch 채팅에서 욕설, 스팸, 부탁하지 않은 게임 플레이 훈수를 숨기고, 스포일러를 흐리게 처리하고, 질문을 강조합니다. 메시지마다 질문 세 개가 든 TypeSafe 요청을 한 번 보내고, 요청들은 동시에 실행되며, 코드의 단순한 규칙이 답을 시청자가 보게 될 화면으로 바꿉니다. | 초급 |

## 추출

지저분한 텍스트에서 타입이 지정된 값과 구조를 뽑아냅니다.

| Cookbook | 하는 일 | 난이도 |
| - | - | - |
| [SDE cascade(캐스케이드)](cookbooks/sde_cascade.md) | 저렴한 모델로 구조화된 데이터를 추출하고, Jev로 각 필드를 검사하고, 검사에 걸릴 때만 추론 모델로 넘깁니다. | 중급 |
| [날짜 추출](cookbooks/date_extraction_cookbook.md) | 문서에 적힌 날짜의 구성 요소를 Jev에게 물은 다음 코드에서 이를 해석하고 검증하며 confidence 기반 검토를 거쳐 절대 날짜와 상대 날짜를 추출합니다. | 초급 |
| [사전 파싱된 값 추출](cookbooks/pre_parsed_value_extraction_cookbook.md) | 정규식으로 이메일, 전화번호, 금액 후보를 찾고, 요청한 것을 Jev가 고르게 한 다음, 정확히 그 구간을 코드에서 정규화합니다. | 초급 |
| [구조 복원](cookbooks/autoformat.md) | 서식을 잃은 일반 텍스트에서 요청 두 번으로 Markdown을 재구성합니다. 한 번은 강제로 줄바꿈된 줄들을 다시 이어 붙이고, 한 번은 모든 블록(제목, 목록, 코드, 콜아웃)을 분류합니다. | 초급 |
| [장바구니](cookbooks/cart_from_primitives.md) | 자연어로 된 카페 주문을 구조화된 장바구니로 바꿉니다. 선택지가 점점 줄어드는 Choice 질문 루프로 모든 품목을 찾은 다음, 품목마다 일괄 요청 한 번으로 알려진 필드를 채웁니다. | 초급 |

## 검색과 매칭

쿼리와 일치하거나 서로 일치하는 줄, passage, 레코드를 찾습니다.

| Cookbook | 하는 일 | 난이도 |
| - | - | - |
| [Re-ranking(재순위화)](cookbooks/rerank_typesafe.md) | CLERC 법률 쿼리 40개에 대해 passage 30개짜리 BM25 후보 목록을 만든 다음, 쿼리와 후보 쌍마다 System One 질문 하나를 써서 top-1 정확도를 5%에서 30%로, top-10 정확도를 38%에서 65%로 끌어올립니다. | 초급 |
| [무한 cardinality(카디널리티) Choice](cookbooks/infinite_cardinality_choice.md) | 어렴풋이 기억나는 영화를 본인의 설명만으로 5,000편 중에서 찾아냅니다. 짧은 줄거리를 놓고 영화마다 Noul 하나, 상위 80편은 긴 줄거리를 놓고 다시 한 번, 그리고 상위 20편에 대해 Choice 하나를 묻습니다. 정답 영화가 1위로 나오는 비율은 78%이며, embedding(임베딩) 검색으로 후보 목록을 만들 때는 33%입니다. | 중급 |
| [줄 단위 검색](cookbooks/semantic_find.md) | GitHub 서비스 약관을 대상으로 시맨틱 검색을 만듭니다. 한 요청에서 Choice 질문으로 줄 ID 218개를 평이한 말로 쓴 쿼리에 비추어 점수를 매기고, Noul 질문으로 문서에 답이 들어 있는지 확인합니다. | 초급 |
| [Knowledge graph(지식 그래프) entity alignment(엔터티 정렬)](cookbooks/entity_alignment.md) | Score 질문 하나로 두 출처의 후보 레코드 쌍을 병합 권고, 수동 검토, 연결 없음으로 routing(라우팅)합니다. Noul 질문 세 개가 개별 필드를 확인합니다. 풀이 예제는 Wikidata와 MusicBrainz의 앨범 쌍 450개를 사용합니다. | 초급 |
| [Intelliprompter](cookbooks/intelliprompter.md) | 발표자가 말하는 동안 발표 노트를 하나씩 체크합니다. Jev가 아직 다루지 않은 노트 각각에 비추어 실시간 전사 내용에 점수를 매기고, 점수가 threshold를 넘은 노트는 체크된 상태로 유지됩니다. | 초급 |

## 기법

어떤 질문에서든 더 많은 것을 얻어 냅니다. 질문을 반복하거나, 묶어서 보내거나, 답을 feature(특성)로 바꿉니다.

| Cookbook | 하는 일 | 난이도 |
| - | - | - |
| [Self-consistency(자기 일관성): Choice](cookbooks/consistency_choice_cookbook.md) | 같은 모더레이션 질문을 모델에게 두 번 묻습니다. 결정이 얼마나 자주 바뀔까요? 정답지가 필요 없는 안정성 테스트로, 경계선상의 게시물에서 LLM과 Jev를 비교합니다. | 초급 |
| [Self-consistency: Noul](cookbooks/consistency_noul_cookbook.md) | 같은 보험 청구 건을 모델에게 두 번 묻습니다. 결정이 얼마나 자주 바뀔까요? 정답지가 필요 없는 안정성 테스트로, 경계선상의 청구 건에서 LLM과 Jev를 비교합니다. | 초급 |
| [병렬 질문](cookbooks/parallel_questions.md) | GDPR Wikipedia 문서를 대상으로 질문 13개짜리 규제 브리핑을 실행해, 모든 질문을 TypeSafe 호출 한 번으로 묶으면 답은 달라지지 않으면서 12.2배 저렴하고 11.8배 빠르다는 것을 보여 줍니다. | 초급 |
| [Autoresearch feature 발굴](cookbooks/autoresearch_feature_discovery.md) | 레이블이 붙은 텍스트에 대해 LLM이 질문을 제안하게 하고, Jev가 모든 행에 대해 그 질문에 답하게 한 다음, 그 답으로 CatBoost를 학습시키고 틀린 사례를 다음 단계에 되먹입니다. 풀이 예제는 Wikipedia 편집자들의 요청이 얼마나 정중하게 읽히는지를 예측합니다. | 고급 |

## 이미지 처리

사진과 스캔본에서 대상을 찾고, 거기서 읽어 낸 내용을 확인합니다.

| Cookbook | 하는 일 | 난이도 |
| - | - | - |
| [스캔한 예방접종 기록 읽기](cookbooks/reading_scanned_documents.md) | 스캔한 여행용 접종 기록에서 Noul과 Box로 백신을 하나씩 찾고, 읽을 수 없음이라고 답할 수 있는 Choice로 각 접종의 연도를 읽고, 텍스트만 있는 요청으로 각 접종이 여행 기간에도 여전히 유효한지 확인합니다. | 중급 |
| [OCR confidence와 bounding box(경계 상자)](cookbooks/ocr_confidence_bounding_boxes.md) | OCR 추출기가 송장 사진에서 읽은 값을 확인합니다. Jev가 값마다 box를 치고, 페이지 전체와 그 box를 잘라 낸 이미지에 각각 비추어 점수를 매기고, 점수가 낮은 값은 검토 대상으로 표시합니다. | 중급 |
| [모이통 카메라 방문 기록](cookbooks/feeder_watch.md) | 새 모이통 카메라 영상의 프레임을 질문 다섯 개와 함께 Jev에 보냅니다. 그중 하나는 새를 찾는 Box입니다. 그런 다음 평범한 코드로 답을 종별 방문 기록으로 바꿉니다. | 중급 |

> [!TIP]
>
> TypeSafe는 사람들이 primitive를 어떻게 활용하고 있는지 늘 궁금합니다. cookbook으로 실을 만한 것을 만들었다면 알려 주세요!


이 문서는 개발자 문서 플랫폼인 [Mintlify](https://mintlify.com)로 제작하고 호스팅합니다.
