# Google Sheets에서 Airbnb 숙소 검색하기

> 원하는 숙소를 평범한 영어 문장으로 설명하면, Google Sheets 수식 하나가 케이프타운 Airbnb 숙소 1,000곳 중 가장 잘 맞는 10곳을 10점 만점 점수와 함께 약 1초 만에 찾아 줍니다.

## 문제

머물 곳을 제대로 고르려면 숙소 설명을 읽어야 합니다. 필터로는 가격, 침실 수, 인원을 걸러 낼 수 있지만 "반려견 동반 가능", "해변까지 걸어서 갈 수 있음", "순환 정전에 대비한 예비 전원 있음" 같은 조건은 걸러 낼 수 없습니다. 이런 정보는 설명 안에 있고 호스트마다 다르게 쓰기 때문에, 확인하려면 하나하나 읽는 수밖에 없습니다. 스프레드시트에 숙소가 1,000곳 있다면 오후 내내 스크롤해야 합니다.

## Jev가 적합한 이유

숙소가 요청에 맞는지는 읽고 금방 내릴 수 있는 판단입니다. Jev는 이 판단을 1초도 안 되는 시간에 내리고, 자유 형식 텍스트가 아닌 확률이나 점수를 반환하므로 스프레드시트 수식에서 그 값으로 정렬할 수 있습니다.

## 숙소당 요청 한 번은 너무 느립니다

가장 먼저 떠오르는 방법은 숙소마다 요청을 하나씩 보내며 "이 숙소가 요청에 얼마나 잘 맞는가?"라는 Score 질문을 묻는 것입니다. 요청 하나하나는 빠르지만, 1,000건을 차례로 보내면 30초가 넘게 걸리고 Google Sheets는 사용자 정의 수식을 30초에서 중단시킵니다.

## 후보를 추린 다음 점수 매기기

요청 하나에 숙소를 여러 곳 담을 수 있습니다. Choice 질문은 숙소 ID 100개를 선택지로 가질 수 있고, 답에는 선택지마다 확률이 나오므로 한 그룹 전체의 순위가 한 번에 매겨집니다. 검색을 두 단계로 나눴습니다.

| 단계 | 요청 수 | 질문 | 답으로 하는 일 |
| - | - | - | - |
| 후보 추리기 | 10건, 병렬 | 어느 숙소가 요청에 가장 잘 맞는가? 숙소 ID 100개를 선택지로 하는 Choice 질문이며, 각 숙소에는 설명의 앞부분이 붙습니다. | 그룹마다 확률이 가장 높은 숙소 5곳이 통과해 모두 50곳이 됩니다. |
| 점수 매기기 | 1건 | 이 숙소가 요청에 얼마나 잘 맞는가? 후보로 추린 숙소마다 Score 질문 하나씩이며, 설명 전문이 붙습니다. | 각 점수를 10점 만점 점수로 바꾸고 가장 높은 10곳을 반환합니다. |

Score 질문에는 "Not a match at all"부터 "Matches everything asked for"까지 다섯 단계가 있으므로, 점수는 숙소가 요청을 얼마나 충족하는지를 나타냅니다. "6 people and a dog on the beach"라는 요청에서는 최상위 결과가 8.8점을 받습니다. 설명에 10명이 잘 수 있고, 해변에서 몇 분 거리이며, 반려동물을 환영한다고 적혀 있기 때문입니다.

## 요청 1,000건이 11건으로

두 단계에 드는 요청은 1,000건이 아니라 11건이고, 검색은 30초 제한을 넘기는 대신 약 1초 만에 끝납니다.

## 어떤 요청이든 같은 시트에서

수식은 `=JEV_TOP10(B2, Listings!A2:H1001)`이므로 `B2`에 새 요청을 입력하면 새로운 검색이 됩니다. 시트는 요청별 결과를 캐시하므로 다른 셀을 편집해도 검색이 다시 실행되지 않습니다.

## 직접 만들기

> [card] [Google Sheet 복사하기](https://docs.google.com/spreadsheets/d/1OeHSq-B354XT7ZoUIZU31kXKdgb7CHQnh43EpOSRAKo/copy)
>
> 숙소 1,000곳과 `JEV_TOP10` 수식이 들어 있습니다. Extensions > Apps Script > Project Settings > Script properties에서 TypeSafe API 키를 `TYPESAFE_API_KEY`라는 이름으로 추가하세요. 그런 다음 `B2`에 요청을 입력합니다. 복사본에 편집 권한이 있는 사람은 누구나 이 키를 볼 수 있습니다.

> [card] [API 키 발급받기](https://console.typesafe.ai/keys)
>
> TypeSafe 콘솔에서 키를 만듭니다.

> [accordion] JEV\_TOP10을 구현한 Apps Script
>
> ```javascript Code.gs theme={null}
> /**
>  * JEV_TOP10: TypeSafe Jev를 사용해, 일상 언어로 쓴 요청에 가장 잘 맞는 숙소 10곳을 찾습니다.
>  *
>  * 설정: Extensions > Apps Script에서 이 파일을 붙여 넣고 저장합니다. 그런 다음 Project Settings >
>  * Script properties에서 TYPESAFE_API_KEY에 TypeSafe 키를 넣어 추가합니다. 권한 요청 창은 뜨지 않습니다.
>  *
>  * 사용법: =JEV_TOP10(B2, Listings!A2:H1001)
>  *   숙소 열은 반드시 다음 순서여야 합니다: Name, Type, Price per night (R), Bedrooms, Guests, Rating,
>  *   Description, Link.
>  *
>  * 동작 방식(숙소당 요청 한 번 대신 요청 11건):
>  *   1. 후보 추리기: 숙소를 100개씩 그룹으로 나눕니다. 그룹마다 요청 하나를 모두
>  *      병렬로 보내 그룹의 숙소 ID를 선택지로 하는 Choice 질문을 묻습니다. Jev가 모든 숙소의
>  *      확률을 반환하고, 가능성이 가장 높은 5곳이 통과합니다.
>  *   2. 점수 매기기: 요청 하나에 후보 숙소의 전체 내용을 담고, 숙소마다 Score 질문을
>  *      하나씩 묻습니다. 가장 높은 점수 10개를 0에서 10 사이 값으로 반환합니다.
>  * 결과는 요청별로 6시간 동안 캐시됩니다.
>  */
>
> const GROUP_SIZE = 100;
> const SHORTLIST_PER_GROUP = 5;
> const TOP = 10;
> const SHORT_DESCRIPTION = 300;
> const FULL_DESCRIPTION = 1500;
> const LEVELS = [
>   'Not a match at all',
>   'Matches little of what was asked for',
>   'Matches some of what was asked for',
>   'Matches most of what was asked for',
>   'Matches everything asked for',
> ];
>
> /**
>  * 요청에 가장 잘 맞는 숙소 10곳을 0-10 일치 점수와 함께 반환합니다.
>  *
>  * @param {string} request 찾고 있는 것을 일상 언어로 쓴 문장.
>  * @param {Listings!A2:H1001} listings Name, Type, Price, Bedrooms, Guests, Rating, Description, Link.
>  * @return 상위 10곳의 Match, Name, Type, Price, Bedrooms, Guests, Rating, Link.
>  * @customfunction
>  */
> function JEV_TOP10(request, listings) {
>   const key = PropertiesService.getScriptProperties().getProperty('TYPESAFE_API_KEY');
>   if (!key) throw new Error('Add TYPESAFE_API_KEY under Project Settings > Script properties.');
>   request = String(request || '').trim();
>   if (!request) return [['Type what you are looking for in B2.']];
>
>   const rows = listings.filter((r) => r[0] !== '');
>   const cache = CacheService.getScriptCache();
>   const cacheKey = digest(`${request}\n${rows.length}\n${rows.map((r) => r[7]).join('|')}`);
>   const hit = cache.get(cacheKey);
>   if (hit) return JSON.parse(hit);
>
>   // 1단계: 숙소 ID 100개 그룹마다 Choice 질문 하나, 모든 그룹을 병렬로.
>   const groups = [];
>   for (let start = 0; start < rows.length; start += GROUP_SIZE) {
>     groups.push(rows.slice(start, start + GROUP_SIZE).map((_, j) => start + j));
>   }
>   const shortlistResponses = UrlFetchApp.fetchAll(groups.map((ids) => jevRequest(key, {
>     state: ids.map((i) => line(i, rows[i], SHORT_DESCRIPTION)).join('\n'),
>     questions: {
>       best: {
>         type: 'choice',
>         instructions: `Which listing best matches what the guest is looking for: ${request}`,
>         criteria: Object.fromEntries(ids.map((i) => [`L${i}`, null])),
>       },
>     },
>   })));
>   const shortlist = shortlistResponses.flatMap((res) => {
>     const p = parse(res).answers.best.probabilities;
>     return Object.keys(p).sort((a, b) => p[b] - p[a]).slice(0, SHORTLIST_PER_GROUP).map((id) => Number(id.slice(1)));
>   });
>
>   // 2단계: 후보로 추린 모든 숙소의 점수를 요청 하나로 매깁니다.
>   const scored = parse(UrlFetchApp.fetchAll([jevRequest(key, {
>     state: shortlist.map((i) => line(i, rows[i], FULL_DESCRIPTION)).join('\n\n'),
>     questions: Object.fromEntries(shortlist.map((i) => [`L${i}`, {
>       type: 'score',
>       instructions: `How well does listing [L${i}] match what the guest is looking for: ${request}`,
>       criteria: LEVELS,
>     }])),
>   })])[0]).answers;
>
>   const top = shortlist
>     .map((i) => ({i, score: Math.round((scored[`L${i}`].score / (LEVELS.length - 1)) * 100) / 10}))
>     .sort((a, b) => b.score - a.score)
>     .slice(0, TOP)
>     .map(({i, score}) => {
>       const [name, type, price, beds, guests, rating, , link] = rows[i];
>       return [score, name, type, price, beds, guests, rating, link];
>     });
>   cache.put(cacheKey, JSON.stringify(top), 6 * 60 * 60);
>   return top;
> }
>
> function line(i, r, descriptionChars) {
>   const [name, type, price, beds, guests, , description] = r;
>   return `[L${i}] ${name} | ${type} | R${price}/night | ${beds || '?'} bedrooms | sleeps ${guests} | ${String(description).slice(0, descriptionChars)}`;
> }
>
> function jevRequest(key, body) {
>   return {
>     url: 'https://api.typesafe.ai/v1/systemone',
>     method: 'post',
>     contentType: 'application/json',
>     headers: {Authorization: `Bearer ${key}`},
>     muteHttpExceptions: true,
>     payload: JSON.stringify(Object.assign({model: 'jev-latest'}, body)),
>   };
> }
>
> function parse(res) {
>   if (res.getResponseCode() !== 200) {
>     throw new Error(`Jev returned HTTP ${res.getResponseCode()}: ${res.getContentText().slice(0, 200)}`);
>   }
>   return JSON.parse(res.getContentText());
> }
>
> function digest(text) {
>   return Utilities.base64EncodeWebSafe(Utilities.computeDigest(Utilities.DigestAlgorithm.SHA_256, text));
> }
> ```

숙소 데이터는 [Inside Airbnb](https://insideairbnb.com/get-the-data/)가 2026년 9월 23일에 수집한 케이프타운 데이터에서 가져왔으며, [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) 라이선스를 따릅니다. 가격이 있고 설명이 200자 이상인 숙소 중 1,000곳을 무작위로 추출한 뒤, HTML 줄바꿈과 반복되는 공백을 제거하고 편의 시설을 쉼표로 구분한 목록으로 펼쳤습니다.


이 문서는 개발자 문서 플랫폼인 [Mintlify](https://mintlify.com)로 제작하고 호스팅합니다.
