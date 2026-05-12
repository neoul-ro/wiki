---
title: "Cloudflare Worker로 간단한 proxy 만들기"
created: 2026-05-12
updated: 2026-05-12
tags: [how-to, cloudflare, workers, proxy]
author: "neoul"
status: "published"
---

# Cloudflare Worker로 간단한 proxy 만들기

Cloudflare Worker는 서버를 직접 만들지 않고도 인터넷에 작은 JavaScript 함수를 배포하는 기능이다. 특정 사이트가 Google Sheets나 Apps Script 요청을 막을 때, Worker를 중간에 두면 `Google Sheets → Worker → 외부 API` 흐름으로 데이터를 가져올 수 있다. 이 글은 Cloudflare를 처음 쓰는 사람이 웹 화면에서 Worker를 만들고 URL을 얻는 최소 절차를 정리한다.

## 결론

Worker는 “내가 소유한 작은 중간 주소”라고 생각하면 된다.

```text
Google Sheets
  ↓
https://my-worker.my-subdomain.workers.dev
  ↓
외부 API
```

Google Sheets는 외부 API를 직접 부르지 않고 Worker URL만 부른다. Worker가 대신 외부 API를 호출하고 결과를 돌려준다.

## 준비물

필요한 것은 Cloudflare 계정 하나다. 도메인을 Cloudflare로 옮길 필요는 없다. Worker는 기본적으로 `workers.dev` 주소를 받을 수 있다.

예상 결과는 이런 형태의 URL이다.

```text
https://my-worker.my-subdomain.workers.dev
```

이 URL을 나중에 Google Sheets Apps Script에 넣는다.

## Worker 만들기

Cloudflare에 로그인한 뒤 Dashboard에서 진행한다.

1. Cloudflare Dashboard에 로그인한다.
2. 왼쪽 메뉴에서 `Workers & Pages`로 이동한다.
3. `Create application` 또는 `Create`를 선택한다.
4. Worker를 만드는 선택지를 고른다.
5. 이름을 정한다. 예를 들어 `cnn-fear-greed-proxy`처럼 쓴다.
6. 기본 코드가 보이면 일단 배포하거나, 바로 코드를 수정한다.

Cloudflare 화면 문구는 조금씩 바뀔 수 있지만 핵심은 `Workers & Pages`에서 새 Worker application을 만드는 것이다.

## proxy 코드 붙여넣기

Worker 편집 화면에서 아래 코드를 붙여넣는다. 이 예시는 CNN Fear & Greed Index JSON을 대신 가져오는 proxy다.

```javascript
export default {
  async fetch() {
    const response = await fetch(
      'https://production.dataviz.cnn.io/index/fearandgreed/graphdata',
      {
        headers: {
          Accept: 'application/json',
          Referer: 'https://edition.cnn.com/markets/fear-and-greed',
          'User-Agent':
            'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36',
        },
      },
    );

    return new Response(await response.text(), {
      status: response.status,
      headers: {
        'content-type': 'application/json; charset=utf-8',
        'cache-control': 'public, max-age=300',
      },
    });
  },
};
```

코드를 저장하고 `Deploy`를 누른다.

## URL 확인하기

배포가 끝나면 Cloudflare가 Worker URL을 보여준다. 보통 다음처럼 생겼다.

```text
https://cnn-fear-greed-proxy.MY_SUBDOMAIN.workers.dev
```

이 주소를 브라우저에서 열었을 때 JSON처럼 보이는 텍스트가 나오면 성공이다.

대략 이런 형태여야 한다.

```json
{
  "fear_and_greed": {
    "score": 50,
    "rating": "neutral"
  }
}
```

실제 값과 필드는 CNN 응답에 따라 달라질 수 있다.

## Google Sheets에서 쓰기

Google Sheets Apps Script에는 CNN 원본 주소가 아니라 Worker URL을 넣는다.

```javascript
const CNN_FEAR_GREED_PROXY_URL =
  'https://cnn-fear-greed-proxy.MY_SUBDOMAIN.workers.dev';
```

이렇게 하면 Sheets는 Worker만 호출한다. Worker가 CNN에 대신 요청하고 결과를 돌려준다.

## 왜 이렇게 하는가

Google Apps Script가 CNN 같은 사이트를 직접 호출하면 요청이 자동화된 bot처럼 보여 차단될 수 있다. 이때 `418`, `403` 같은 오류가 난다.

Worker를 중간에 두면 다음 장점이 있다.

- Google Sheets 코드가 단순해진다.
- 외부 사이트 요청 header를 Worker에서 조정할 수 있다.
- cache header를 넣어 호출 부담을 줄일 수 있다.
- 나중에 다른 API proxy로 바꾸기도 쉽다.

## 실패할 때 확인할 것

Worker URL을 브라우저에서 먼저 열어본다.

- JSON이 보이면 Worker는 정상이다.
- Cloudflare 에러 페이지가 보이면 배포가 안 됐거나 코드가 실패한 것이다.
- CNN의 `418` 문구가 보이면 CNN이 Worker 요청도 막은 것이다.
- Google Sheets에서만 실패하면 Apps Script의 proxy URL이 잘못됐을 가능성이 높다.

Apps Script에는 반드시 Worker URL을 넣어야 한다. CNN 원본 URL을 넣으면 다시 같은 차단 문제가 생길 수 있다.

## 주의할 점

Worker는 중간 서버 역할을 하므로 공개 URL이 된다. API key나 비밀번호를 코드에 직접 넣는 방식은 피해야 한다. 공개 데이터 proxy처럼 민감정보가 없는 요청에 먼저 쓰는 것이 안전하다.

또한 대상 사이트가 공식 API로 제공하지 않는 데이터를 가져오는 경우, 사이트 구조나 차단 정책이 바뀌면 Worker도 수정해야 한다.

## References

- [Cloudflare Workers Dashboard guide](https://developers.cloudflare.com/workers/get-started/dashboard/)
- [Cloudflare Workers getting started](https://developers.cloudflare.com/workers/get-started/)

## Related

- [[finance/cnn-fear-greed-index|CNN Fear & Greed Index를 Google Sheets로 가져오기]]
