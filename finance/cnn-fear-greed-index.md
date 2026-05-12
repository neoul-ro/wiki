---
title: "CNN Fear & Greed Index를 Google Sheets로 가져오기"
created: 2026-05-12
updated: 2026-05-12
tags: [how-to, google-sheets, apps-script, finance, cnn]
author: "neoul"
status: "published"
---

# CNN Fear & Greed Index를 Google Sheets로 가져오기

CNN Fear & Greed Index를 Google Sheets에서 쓰려면 CNN 엔드포인트를 시트에서 직접 호출하지 말고, 얇은 proxy를 하나 둔 뒤 Google Sheets는 그 proxy를 호출하게 만드는 편이 안정적이다. CNN이 Google Apps Script 요청을 bot으로 판단하면 `CNN request failed: 418`이 발생하기 때문이다.

## 결론

먼저 Cloudflare Worker 같은 간단한 proxy를 만든다. Worker는 CNN JSON을 가져와 그대로 돌려주는 역할만 한다.

Cloudflare Worker가 뭔지 모르겠다면 먼저 [[web-infrastructure/cloudflare-worker-proxy|Cloudflare Worker로 간단한 proxy 만들기]]를 읽고, Worker URL을 만든 뒤 이 글로 돌아온다.

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

Worker를 배포한 뒤, Google Sheets에서 `Extensions` → `Apps Script`를 열고 아래 코드를 붙여넣는다. `CNN_FEAR_GREED_PROXY_URL`은 본인의 Worker URL로 바꾼다.

```javascript
const CNN_FEAR_GREED_PROXY_URL =
  'https://YOUR_WORKER.YOUR_SUBDOMAIN.workers.dev';

function fetchCnnFearGreed_() {
  const cache = CacheService.getScriptCache();
  const cached = cache.get('cnn_fear_greed');
  if (cached) return JSON.parse(cached);

  const response = UrlFetchApp.fetch(CNN_FEAR_GREED_PROXY_URL, {
    muteHttpExceptions: true,
  });

  const status = response.getResponseCode();
  if (status !== 200) {
    throw new Error(`CNN proxy request failed: ${status}`);
  }

  const data = JSON.parse(response.getContentText());
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

## 왜 proxy가 필요한가

CNN Fear & Greed 데이터는 CSV가 아니라 JSON이다. Google Sheets의 기본 함수만으로 JSON 내부 필드를 안정적으로 뽑기 어렵기 때문에 Apps Script에서 JSON을 가져오고, 필요한 필드만 2차원 배열로 반환하는 편이 낫다.

문제는 Google Apps Script가 CNN 엔드포인트를 직접 호출하면 CNN이 요청을 bot으로 판단할 수 있다는 점이다. 이때 다음 오류가 난다.

```text
Error: CNN request failed: 418
```

`418`은 Google Sheets 수식이나 JSON parsing 문제가 아니라 CNN 쪽 차단 응답이다. proxy를 두면 Sheets는 proxy만 호출하고, proxy가 CNN에 브라우저에 가까운 header로 요청한다.

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

- `CNN request failed: 403`: CNN이 요청을 차단했거나 header가 부족하다.
- `CNN request failed: 418`: Google Apps Script 직접 호출이 CNN에 막힌 것이다. proxy 방식으로 바꾼다.
- `CNN proxy request failed: 418`: proxy도 CNN에 막힌 것이다. Worker header를 확인하거나 다른 배포 위치를 쓴다.
- `Cannot read properties of undefined`: CNN JSON 구조가 바뀌었다.
- 시트에서 값이 늦게 바뀜: custom function 재계산이 아직 일어나지 않았다.
- 너무 잦은 호출 실패: cache 시간을 늘리거나 trigger 방식으로 바꾼다.

## 주의할 점

이 방식은 CNN의 공식 공개 API 계약에 의존하는 것이 아니라, CNN 페이지에서 사용하는 공개 JSON 응답에 의존한다. CNN이 엔드포인트, 차단 정책, JSON 구조를 바꾸면 proxy나 스크립트를 수정해야 한다.

투자 판단에는 단독으로 쓰지 말고, CNN이 설명하는 것처럼 시장 심리를 확인하는 보조 지표로만 사용한다.

## References

- [CNN Fear & Greed Index](https://edition.cnn.com/markets/fear-and-greed)
- [CNN Fear & Greed JSON endpoint](https://production.dataviz.cnn.io/index/fearandgreed/graphdata)
- [fear-greed Python package](https://pypi.org/project/fear-greed/)

## Related

- [[finance/_index|Finance]]
- [[web-infrastructure/cloudflare-worker-proxy|Cloudflare Worker로 간단한 proxy 만들기]]
