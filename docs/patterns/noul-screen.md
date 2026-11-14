# Noul로 먼저 선별하기

> 예/아니오 질문 하나를 먼저 묻고, 답이 분명한 yes일 때만 비용이 큰 단계를 실행합니다.

모든 입력에 비용이 큰 단계가 필요한 것은 아닙니다. 입력만 담은 작은 state(상태)로 Noul 질문 하나를 먼저 묻고, 답이 분명한 yes일 때만 비용이 큰 단계를 실행하세요. 분명한 no는 더 저렴하게 처리하고, 판단이 서지 않는 답에는 별도의 규칙을 적용합니다. 이렇게 하면 모든 입력이 빠짐없이 처리됩니다.

![수신함의 메시지들이 Noul 질문 하나를 향해 흘러갑니다. 각 메시지는 질문에 답이 나올 때까지 대기한 뒤 세 출구 중 하나로 나갑니다. 분명한 no는 보관 처리되고, 판단이 서지 않는 답은 잠깐 확인할 대상으로 따로 빼 두며, 분명한 yes는 고객 기록이 첨부된 전체 triage 요청으로 이어집니다.](https://mintcdn.com/ts-docs/0mdJYbuyjcLi9rbO/patterns/noul-screen-hero-light.gif?s=156a517da1ccc828140f49adf6c59fcd)

## 먼저 선별해야 하는 경우

Jev는 한 요청에 담긴 모든 질문에 병렬로 답하고 state는 한 번만 읽으므로, state 비용도 한 번만 냅니다. 뒤따르는 질문들이 같은 state를 사용한다면 따로 떼어 내지 마세요. 모두 한 요청에 담아 보내고, 해당하지 않는 답은 코드에서 무시하면 됩니다. 이 패턴을 [speculative fan-out(추측성 팬아웃)](fan-out.md)이라고 부릅니다. 먼저 묻느라 요청을 한 번 더 보낼 가치가 있는 것은, 첫 질문에는 들지 않는 비용이 다음 단계에 들 때입니다.

* **더 큰 state.** 다음 단계에 고객 기록, 전체 스레드, 또는 코드로 먼저 가져와야 하는 첨부 파일이 필요합니다.
* **더 느린 모델.** 다음 단계가 LLM 호출입니다.
* **사람.** 다음 단계에서 누군가가 해당 항목을 직접 살펴봐야 합니다.

## 예제: 공유 수신함 triage하기

공유 수신함 도구를 만든다고 가정해 보겠습니다. 주문을 문의하는 고객, 뉴스레터, 결제 영수증, 감사 인사, 그리고 가끔 오는 "참고로 알려 드립니다" 같은 메시지까지 모든 것이 하나의 스트림으로 들어옵니다. 전체 triage(우선순위 분류)를 하려면 보낸 사람의 계정과 주문 내역을 가져온 다음 메시지에 대해 여러 질문을 해야 합니다. 도착한 그대로의 메시지만 보는 Noul 질문 하나로 어떤 메시지를 전체 triage에 넘길지 정할 수 있습니다.

### 보낸 사람이 답장을 기다리는지 묻기

메시지만으로 판단할 수 있는 한 가지를 묻습니다. 보낸 사람이 답장을 기다리고 있는가 하는 것입니다. `false` criterion(판단 기준)에는 "기다리지 않음"이라고 쓰는 대신 답할 것이 없는 메시지 종류를 나열합니다. 사람이 보냈지만 답장이 필요 없는 메모도 여기에 포함됩니다.

Playground에서 실행해 보기 →

Tom의 메시지는 문제를 알리지만 아무것도 요청하지 않습니다. 책상은 이미 조립했고, Tom이 답장을 원하는지는 메시지에 나와 있지 않습니다. 답은 분명한 yes도, 분명한 no도 아닙니다.

```json
{
  "model": "jev-1.14.0",
  "answers": {
    "expects_reply": {
      "type": "noul",
      "noul": 0.44
    }
  },
  "usage": {
    "input_tokens": 414,
    "output_tokens": 21
  }
}
```

서로 다른 메시지 다섯 개에 같은 질문을 했을 때 Jev의 답은 다음과 같습니다.

| 메시지 | `expects_reply` |
| - | - |
| "I ordered a standing desk two weeks ago and the tracking hasn't updated... I'm moving on the 30th and need it before then." | 0.98 |
| "Just so you know, the desk arrived with one of the cross-bar screws missing. I had a spare so it's built now." | 0.44 |
| "Thanks, the desk arrived yesterday and it's great." | 0.10 |
| "20% off every desk and chair until Sunday. Unsubscribe \| View in browser" | 0.02 |
| "Your payment of \$129.00 was successful... replies to this address are not monitored." | 0.02 |

### yes, no, 불확실을 각각 처리하기

Noul을 두 개의 threshold(임계값)와 비교합니다. 0.7을 넘으면 yes, 0.3 미만이면 no로 봅니다. 두 값 모두 출발점으로 삼기에 좋습니다. 그 사이의 답은 메시지만으로는 질문의 답이 정해지지 않는다는 뜻이므로, yes나 no로 반올림하지 말고 별도의 결과로 처리하세요.

```python title="screen.py"
YES_ABOVE = 0.7
NO_BELOW = 0.3


def handle(message_id, screen_response):
    expects_reply = screen_response.answers["expects_reply"].noul

    if expects_reply < NO_BELOW:
        # 뉴스레터, 영수증, 알림. 답할 것이 없습니다.
        file_away(message_id)

    elif expects_reply > YES_ABOVE:
        # 사람이 기다리고 있습니다. triage에 필요한 것을 가져와 전체 요청을 실행합니다.
        customer = fetch_customer(message_id)
        triage(message_id, customer)

    else:
        # 살펴볼 것은 있지만 답할 것은 없을 수도 있습니다. 한 번 훑어보면 정리됩니다.
        queue_for_quick_look(message_id)
```

위의 다섯 메시지 중 triage 요청으로 이어지는 것은 하나, 아직 도착하지 않은 스탠딩 데스크 건입니다. 감사 인사와 자동 발송 메시지 두 건은 보관 처리되고, 나사가 빠졌다는 메모는 사람이 잠깐 확인하도록 넘어갑니다.

### 답장이 필요한 메시지만 triage하기

고객 기록은 분명한 yes가 나온 메시지에 대해서만 가져옵니다. triage 요청은 기록을 메시지와 함께 보내고, 각각 한 가지만 확인하는 질문들을 묻습니다. `repeat_contact`는 `customer.previous_contacts`를 읽고, `urgency`는 `message.received`를 기준으로 날짜를 셉니다. 실제 triage 단계라면 이보다 더 많은 질문을 묻게 됩니다.

Playground에서 실행해 보기 →

Jev의 응답:

```json
{
  "model": "jev-1.14.0",
  "answers": {
    "request": {
      "type": "choice",
      "choice": "order_status",
      "confidence": 1.0,
      "probabilities": {
        "order_status": 1.0,
        "return_or_refund": 0.0,
        "product_question": 0.0,
        "complaint": 0.0
      }
    },
    "urgency": {
      "type": "score",
      "score": 1.99,
      "confidence": 0.98,
      "legend": {
        "0": "No deadline is mentioned",
        "1": "The sender wants an answer soon but gives no date",
        "2": "The sender names a date or event within a week"
      },
      "probabilities": {
        "0": 0.0,
        "1": 0.01,
        "2": 0.99
      }
    },
    "repeat_contact": {
      "type": "noul",
      "noul": 0.99
    }
  },
  "usage": {
    "input_tokens": 803,
    "output_tokens": 84
  }
}
```

`repeat_contact`의 답은 고객 기록에서 나옵니다. 이전 문의 내역이 없는 기록으로 같은 메시지를 보냈을 때는 0.05가 나왔습니다.

여기서부터는 routing(라우팅) 코드가 답에 따라 동작합니다. 긴급하고 반복된 Dana의 주문 상태 문의는 대기열 맨 앞으로 보낼 수 있습니다.

## 그 밖의 선별 방법

* **저렴한 단계를 먼저 검사하기.** 저렴한 단계를 실행하고, 그 출력이 올바른지 Noul로 물은 다음, 검사를 통과하지 못한 출력만 비용이 큰 모델로 보냅니다.
* **여러 단계의 선별.** 콘텐츠 pipeline(파이프라인)이라면 먼저 "영어로 작성되었는가"를, 그다음 "우리 제품에 관한 것인가"를 확인할 수 있습니다. 각 검사는 더 비용이 큰 다음 단계에 앞서 입력을 걸러 냅니다.


이 문서는 개발자 문서 플랫폼인 [Mintlify](https://mintlify.com)로 제작하고 호스팅합니다.
