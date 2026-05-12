---
title: "CNN Fear & Greed Index를 Google Sheets로 가져오기"
created: 2026-05-12
updated: 2026-05-12
tags: [how-to, google-sheets, apps-script, finance, cnn]
author: "neoul"
status: "published"
---

# CNN Fear & Greed Index를 Google Sheets로 가져오기

CNN Fear & Greed Index는 Google Sheets에서 CNN을 직접 호출하면 `418`로 막힐 수 있다. 가장 간단한 우회는 Jina Reader를 통해 CNN JSON endpoint를 읽고, Apps Script에서 응답 안의 JSON 부분만 파싱하는 것이다. 단순 `IMPORTXML`로 HTML을 긁는 방식은 숫자가 HTML에 직접 들어있지 않아 맞지 않는다.

## 결론

Google Sheets에서 `Extensions` → `Apps Script`를 열고 아래 코드를 붙여넣는다.

```javascript
const CNN_FEAR_GREED_READER_URL =
  'https://r.jina.ai/https://production.dataviz.cnn.io/index/fearandgreed/graphdata';

function parseJsonFromReader_(text) {
  const start = text.indexOf('{');
  const end = text.lastIndexOf('}');
  if (start === -1 || end === -1 || end <= start) {
    throw new Error('CNN Reader response did not contain JSON');
  }
  return JSON.parse(text.slice(start, end + 1));
}

function fetchCnnFearGreed_() {
  const cache = CacheService.getScriptCache();
  const cached = cache.get('cnn_fear_greed');
  if (cached) return JSON.parse(cached);

  const response = UrlFetchApp.fetch(CNN_FEAR_GREED_READER_URL, {
    muteHttpExceptions: true,
    headers: {
      Accept: 'text/plain',
    },
  });

  const status = response.getResponseCode();
  if (status !== 200) {
    throw new Error(`CNN Reader request failed: ${status}`);
  }

  const data = parseJsonFromReader_(response.getContentText());
  cache.put('cnn_fear_greed', JSON.stringify(data), 300);
  return data;
}

function CNN_FEAR_GREED() {
  const data = fetchCnnFearGreed_();
  const fg = data.fear_and_greed;

  return [
    ['metric', 'value'],
    ['score', fg.score],
    ['rating', fg.rating],
    ['timestamp', fg.timestamp],
    ['previous_close', fg.previous_close],
    ['previous_1_week', fg.previous_1_week],
    ['previous_1_month', fg.previous_1_month],
    ['previous_1_year', fg.previous_1_year],
  ];
}

function CNN_FEAR_GREED_INDICATORS() {
  const data = fetchCnnFearGreed_();
  const keys = [
    'market_momentum_sp500',
    'stock_price_strength',
    'stock_price_breadth',
    'put_call_options',
    'market_volatility_vix',
    'junk_bond_demand',
    'safe_haven_demand',
  ];

  const rows = [['indicator', 'score', 'rating', 'timestamp']];
  keys.forEach((key) => {
    if (!data[key]) return;
    rows.push([
      key,
      data[key].score,
      data[key].rating,
      data[key].timestamp || '',
    ]);
  });
  return rows;
}
```

시트에서는 아래처럼 사용한다.

```text
=CNN_FEAR_GREED()
```

하위 지표까지 보고 싶으면 다른 영역에 아래 함수를 쓴다.

```text
=CNN_FEAR_GREED_INDICATORS()
```

## 왜 이렇게 해야 하나

CNN Fear & Greed 페이지의 HTML에는 최종 숫자가 직접 들어있지 않다. HTML 안에는 데이터 위치만 들어있다.

```html
data-data-url="https://production.dataviz.cnn.io/index/fearandgreed/graphdata"
```

그래서 `IMPORTXML`로 페이지를 파싱해도 현재 점수를 바로 얻기 어렵다.

JSON endpoint를 Apps Script에서 직접 호출하면 CNN이 Google 서버 요청을 bot 요청으로 보고 다음처럼 막을 수 있다.

```text
Error: CNN request failed: 418
```

로컬 터미널에서는 쿠키를 받아 재요청하는 방식이 될 수 있다.

```bash
curl -L -s -c /tmp/cnn-cookies.txt \
  'https://edition.cnn.com/markets/fear-and-greed' \
  -o /tmp/cnn-fng.html

curl -s -b /tmp/cnn-cookies.txt \
  -H 'Accept: application/json,text/plain,*/*' \
  -H 'Referer: https://edition.cnn.com/markets/fear-and-greed' \
  -H 'Origin: https://edition.cnn.com' \
  -H 'User-Agent: Mozilla/5.0' \
  'https://production.dataviz.cnn.io/index/fearandgreed/graphdata'
```

하지만 Apps Script 실행 환경에서는 같은 방식도 `418`로 막힐 수 있다. 그래서 위 코드는 Jina Reader를 통해 CNN JSON을 읽는다.

```text
Google Sheets → Jina Reader → CNN JSON endpoint
```

Jina Reader 응답은 앞에 `Title`, `URL Source`, `Markdown Content` 같은 설명이 붙고, 그 뒤에 JSON이 들어온다. 그래서 Apps Script에서 첫 `{`부터 마지막 `}`까지만 잘라 `JSON.parse()`한다.

## 가져오는 값

`CNN_FEAR_GREED()`는 현재 Fear & Greed Index 요약값을 반환한다.

| Field | Meaning |
|---|---|
| `score` | 현재 지수 점수 |
| `rating` | fear, neutral, greed 같은 현재 구간 |
| `timestamp` | CNN 데이터 timestamp |
| `previous_close` | 직전 종가 기준 값 |
| `previous_1_week` | 1주 전 값 |
| `previous_1_month` | 1개월 전 값 |
| `previous_1_year` | 1년 전 값 |

`CNN_FEAR_GREED_INDICATORS()`는 CNN이 쓰는 7개 하위 지표를 반환한다.

| Indicator key | Meaning |
|---|---|
| `market_momentum_sp500` | S&P 500과 125일 이동평균 |
| `stock_price_strength` | NYSE 52주 신고가/신저가 |
| `stock_price_breadth` | McClellan Volume Summation Index |
| `put_call_options` | 5일 평균 put/call ratio |
| `market_volatility_vix` | VIX와 50일 이동평균 |
| `junk_bond_demand` | junk bond와 investment grade bond 수익률 spread |
| `safe_haven_demand` | 주식과 채권의 20일 수익률 차이 |

## 자동 갱신

Custom function은 시트가 다시 계산될 때 갱신된다. 강제로 갱신하고 싶다면 빈 셀에 `=NOW()`를 두거나, Apps Script에서 time-driven trigger로 값을 쓰는 방식이 더 확실하다.

자주 호출하면 Google Apps Script quota나 CNN 측 차단에 걸릴 수 있으므로, 위 코드는 `CacheService`로 5분 동안 같은 응답을 재사용한다.

## 실패할 때 확인할 것

먼저 Apps Script editor에서 `CNN_FEAR_GREED` 함수를 직접 실행해 권한 승인과 에러 메시지를 확인한다.

자주 보는 문제는 다음과 같다.

- `CNN Reader request failed: 403`: Jina Reader 요청이 차단되었거나 일시적으로 실패했다.
- `CNN Reader request failed: 429`: Jina Reader rate limit에 걸렸다. 호출 빈도를 줄이거나 cache 시간을 늘린다.
- `CNN Reader response did not contain JSON`: Jina 응답 형식이 바뀌었거나 CNN endpoint 응답이 실패했다.
- `Cannot read properties of undefined`: CNN JSON 구조가 바뀌었다.
- 시트에서 값이 늦게 바뀜: custom function 재계산이 아직 일어나지 않았다.
- 너무 잦은 호출 실패: cache 시간을 늘리거나 trigger 방식으로 바꾼다.

## 그래도 막히면

위 방식은 Cloudflare Worker 없이 공개 Reader API를 쓰는 방법이다. 공개 proxy에 의존하고 싶지 않거나 rate limit이 문제라면 Cloudflare Worker 같은 proxy를 직접 둔다.

```text
Google Sheets → Worker proxy → CNN
```

proxy 방식은 [[web-infrastructure/cloudflare-worker-proxy|Cloudflare Worker로 간단한 proxy 만들기]]에 정리해 두었다.

## 주의할 점

이 방식은 CNN의 공식 공개 API 계약에 의존하는 것이 아니라, CNN 페이지에서 사용하는 공개 JSON 응답에 의존한다. CNN이 엔드포인트, 차단 정책, JSON 구조를 바꾸면 proxy나 스크립트를 수정해야 한다.

투자 판단에는 단독으로 쓰지 말고, CNN이 설명하는 것처럼 시장 심리를 확인하는 보조 지표로만 사용한다.

## References

- [CNN Fear & Greed Index](https://edition.cnn.com/markets/fear-and-greed)
- [CNN Fear & Greed JSON endpoint](https://production.dataviz.cnn.io/index/fearandgreed/graphdata)
- [Jina Reader API](https://jina.ai/reader/)
- [fear-greed Python package](https://pypi.org/project/fear-greed/)

## Related

- [[finance/_index|Finance]]
- [[web-infrastructure/cloudflare-worker-proxy|Cloudflare Worker로 간단한 proxy 만들기]]
