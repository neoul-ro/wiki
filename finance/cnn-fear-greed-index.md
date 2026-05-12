---
title: "CNN Fear & Greed Index를 Google Sheets로 가져오기"
created: 2026-05-12
updated: 2026-05-12
tags: [how-to, google-sheets, apps-script, finance, cnn]
author: "neoul"
status: "published"
---

# CNN Fear & Greed Index를 Google Sheets로 가져오기

CNN Fear & Greed Index를 Google Sheets에서 쓰려면 `IMPORTDATA`보다 Apps Script로 JSON을 가져오는 방식이 안정적이다. CNN 페이지가 사용하는 공개 JSON 엔드포인트에서 현재 score, rating, 과거 비교값, 7개 하위 지표를 받을 수 있고, Sheets에서는 custom function으로 표 형태를 반환하면 된다.

## 결론

Google Sheets에서 `Extensions` → `Apps Script`를 열고 아래 코드를 붙여넣는다.

```javascript
const CNN_FEAR_GREED_URL =
  'https://production.dataviz.cnn.io/index/fearandgreed/graphdata';

function fetchCnnFearGreed_() {
  const cache = CacheService.getScriptCache();
  const cached = cache.get('cnn_fear_greed');
  if (cached) return JSON.parse(cached);

  const response = UrlFetchApp.fetch(CNN_FEAR_GREED_URL, {
    muteHttpExceptions: true,
    headers: {
      Accept: 'application/json',
      Referer: 'https://edition.cnn.com/',
      'User-Agent': 'Mozilla/5.0',
    },
  });

  const status = response.getResponseCode();
  if (status !== 200) {
    throw new Error(`CNN request failed: ${status}`);
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

## 왜 IMPORTDATA가 아니라 Apps Script인가

CNN Fear & Greed 데이터는 CSV가 아니라 JSON이다. Google Sheets의 기본 함수만으로 JSON 내부 필드를 안정적으로 뽑기 어렵기 때문에 Apps Script에서 `UrlFetchApp.fetch()`로 JSON을 가져오고, 필요한 필드만 2차원 배열로 반환하는 편이 낫다.

또한 CNN 엔드포인트는 일반 브라우저 요청을 기대할 수 있다. 그래서 Apps Script 요청에 `Accept`, `Referer`, `User-Agent` header를 넣어두면 실패 가능성을 줄일 수 있다.

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
- `Cannot read properties of undefined`: CNN JSON 구조가 바뀌었다.
- 시트에서 값이 늦게 바뀜: custom function 재계산이 아직 일어나지 않았다.
- 너무 잦은 호출 실패: cache 시간을 늘리거나 trigger 방식으로 바꾼다.

## 주의할 점

이 방식은 CNN의 공식 공개 API 계약에 의존하는 것이 아니라, CNN 페이지에서 사용하는 공개 JSON 응답에 의존한다. CNN이 엔드포인트나 JSON 구조를 바꾸면 스크립트를 수정해야 한다.

투자 판단에는 단독으로 쓰지 말고, CNN이 설명하는 것처럼 시장 심리를 확인하는 보조 지표로만 사용한다.

## References

- [CNN Fear & Greed Index](https://edition.cnn.com/markets/fear-and-greed)
- [CNN Fear & Greed JSON endpoint](https://production.dataviz.cnn.io/index/fearandgreed/graphdata)
- [fear-greed Python package](https://pypi.org/project/fear-greed/)

## Related

- [[finance/_index|Finance]]
