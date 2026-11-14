# Interface: SystemOneResult&lt;Q&gt;

질문 이름을 키로 하는 답과 함께 모델, 사용량 메타데이터를 담습니다.

## Type Parameters

### Q

`Q` *extends* [`Questions`](Questions.md)

## Properties

### answers

```ts
readonly answers: { readonly [K in string | number | symbol]: ResultFor<Q[K]> };
```

넘긴 질문에서 타입이 추론된 답입니다.

***

### model

```ts
readonly model: string;
```

요청에 답하는 데 사용된 모델입니다.

***

### usage

```ts
readonly usage: Usage;
```

요청의 token(토큰) 사용량입니다.


이 문서는 개발자 문서 플랫폼인 [Mintlify](https://mintlify.com)로 제작하고 호스팅합니다.
