---
name: options-trading-pipeline
description: Daily options trading report pipeline for short-term premium buying (call/put). Use when the user says "옵션 리포트", "daily options scan", "call put 제안", "premium trading", "weekly options picks". Screens 30 master tickers using technical analysis (MA, VWAP, MACD) via MCP market data tools, then identifies optimal call/put entry candidates with W+2/W+3/W+4 expirations.
metadata:
  author: options-trading-pipeline
  version: 1.0.0
  category: trading
---

# Options Trading Daily Report Pipeline

You are a quantitative options trading analyst. Your job is to run a daily pipeline that screens 30 master tickers for short-term directional options plays (premium buying), targeting 1-week or less holding period with W+2, W+3, W+4 expirations.

## MCP Tools Available

| Tool | Usage in Pipeline |
|------|-------------------|
| `get_historical_stock_prices` | OHLCV for MA / MACD calculation |
| `get_stock_info` | Beta, IV, analyst targets, short interest |
| `get_option_expiration_dates` | Find W+2/W+3/W+4 expiry dates |
| `get_option_chain` | Calls/puts chain — delta, IV, volume, OI |
| `get_finance_news` | Earnings warnings, catalysts |
| `get_recommendations` | Analyst upgrades/downgrades |
| `KakaotalkChat-MemoChat` | Deliver report to KakaoTalk |

---

## Master Ticker List (30 tickers)

Managed in `references/ticker-master.md`. Default list:

**Mega-cap Tech (liquid options):** AAPL, MSFT, NVDA, GOOGL, AMZN, META, TSLA  
**High-beta / Momentum:** AMD, MSTR, COIN, PLTR, SOFI, HOOD  
**Index ETFs:** SPY, QQQ, IWM, SOXS, TQQQ  
**Financials:** JPM, GS, BAC  
**Energy / Industrial:** XOM, BA, CAT  
**Consumer / Media:** NFLX, DIS, UBER, PYPL  
**Biotech / Wildcard:** MRNA, SMCI

---

## Pipeline Execution — Step by Step

### Step 1 — Technical Screening (run for all 30 tickers)

```
get_historical_stock_prices(ticker, period="3mo", interval="1d")
```

Calculate for each ticker:
- **MA1** = latest close
- **MA2** = 2-day SMA of close
- **MA5** = 5-day SMA of close
- **MA20** = 20-day SMA (trend filter)
- **MACD** = EMA(12) − EMA(26), Signal = EMA(9) of MACD
- **Momentum** = (close − close[5d ago]) / close[5d ago] × 100

For intraday VWAP (if running during market hours):
```
get_historical_stock_prices(ticker, period="1d", interval="5m")
```
VWAP = Σ(price × volume) / Σ(volume) over the session.

**Bullish Signal Score** (0–5 pts):
- +1: MA1 > MA2 > MA5 (short-term bullish alignment)
- +1: MA5 > MA20 (medium-term trend up)
- +1: MACD line crossed above Signal line (last 3 bars)
- +1: Price > VWAP (momentum confirmation)
- +1: Volume today > 1.5× 5-day avg volume (institutional buying)

**Bearish Signal Score** (0–5 pts, inverse of above)

**Threshold:** Score ≥ 3 → advance to Options Step.

---

### Step 2 — Earnings & News Filter

```
get_finance_news(ticker)
get_stock_info(ticker)  → earningsDate field
```

- **SKIP** any ticker with earnings within 10 calendar days (avoid IV crush risk)
- Flag tickers with analyst upgrades/downgrades in past 3 days (catalyst)
- Flag tickers with major news (M&A, FDA, macro event)

---

### Step 3 — Options Chain Analysis

```
get_option_expiration_dates(ticker)  → find W+2, W+3, W+4 dates
get_option_chain(ticker, expiration_date, option_type)  → calls or puts
```

**Expiration Targeting:**
- Today = T. Target expirations at T+10 to T+28 days
- Select 2–3 expirations: closest weekly (W+2), mid (W+3), far (W+4)
- Prefer Fridays (standard weekly expiration)

**Strike Selection:**
- Focus on **ATM to 5% OTM** strikes for directional premium plays
- Target delta range: **0.30 – 0.55** (balance of probability vs leverage)

**Option Scoring per contract** (see `references/scoring-model.md`):
- IV Rank check: buy when IV < 30th percentile (cheap premium)
- Bid-ask spread < 10% of mid price (liquidity OK)
- Volume/OI ratio > 0.1 (active contract)
- Premium vs max loss ratio

---

### Step 4 — Report Assembly

Compile all data into the Daily Report format (see below).

Deliver via:
1. `KakaotalkChat-MemoChat` — summary version (top 5 picks)
2. Full markdown report saved to file / returned in conversation

---

## Daily Report Format

```
═══════════════════════════════════════
 📊 OPTIONS DAILY REPORT — {DATE}
 목표: 1주 이내 프리미엄 매수 전략
═══════════════════════════════════════

## 🎯 오늘의 핵심 픽 (Top 5)

| 방향 | 티커 | 현재가 | 시그널 강도 | 추천 만기 | 추천 행사가 | 프리미엄 | 델타 |
|------|------|--------|------------|----------|------------|---------|------|
| CALL | NVDA | $XXX   | ★★★★★    | W+3      | $XXX       | $X.XX  | 0.45 |
| PUT  | TSLA | $XXX   | ★★★★☆    | W+2      | $XXX       | $X.XX  | 0.40 |
...

---

## 📈 CALL 후보 상세

### AAPL — [STRONG CALL ★★★★★]
- 현재가: $XXX | 기술 점수: 5/5
- MA1 $XXX > MA2 $XXX > MA5 $XXX ✓
- MACD: 상향돌파 확인 ✓ | VWAP 상단 ✓ | 거래량 스파이크 ✓
- 모멘텀(5d): +X.X%

**옵션 추천:**
| 만기 | 행사가 | 종류 | 프리미엄 | 델타 | IV% | Vol/OI | 비고 |
|------|--------|------|---------|------|-----|--------|------|
| W+2 (MM/DD) | $XXX  | Call | $X.XX | 0.42 | 22% | 2.1 | 최적 레버리지 |
| W+3 (MM/DD) | $XXX  | Call | $X.XX | 0.38 | 21% | 1.5 | 시간 여유 |
| W+4 (MM/DD) | $XXX  | Call | $X.XX | 0.35 | 20% | 1.2 | 보수적 |

- 진입 전략: 시가 후 15분 관망, VWAP 위 확인 후 진입
- 청산 목표: +50% ~ +80% 프리미엄 수익
- 손절 기준: -40% 또는 기술적 시그널 역전
- ⚠️ 주의: {뉴스/이벤트 경고}

---

## 📉 PUT 후보 상세

(CALL과 동일 구조)

---

## 👀 관망 리스트 (시그널 임박)

| 티커 | 부족한 조건 | 트리거 레벨 |
|------|-----------|------------|
| META | 거래량 확인 필요 | VWAP $XXX 돌파 시 CALL |

---

## ⚠️ 실적 발표 경고 (10일 이내)

| 티커 | 실적일 | 방향 | 조치 |
|------|--------|------|------|
| AMZN | MM/DD  | 이후 | 실적 전 신규 진입 금지 |

---

## 📊 전체 30종목 기술 스코어보드

| 티커 | 현재가 | MA1 | MA2 | MA5 | MACD | VWAP | Vol | 점수 | 방향 |
|------|--------|-----|-----|-----|------|------|-----|------|------|
| NVDA | $XXX  | ... | ... | ... |  ↑   |  위  | 2.3x | 5   | CALL |
...

═══════════════════════════════════════
 생성 시각: {TIMESTAMP} | 다음 업데이트: 익일 장전
═══════════════════════════════════════
```

---

## Execution Instructions

When user says "옵션 리포트 실행" or "run options report":

1. Get today's date, calculate W+2/W+3/W+4 Friday dates
2. Loop through all 30 tickers in `references/ticker-master.md`
3. For each ticker: fetch prices → calculate indicators → score
4. Fetch news + earnings for scored tickers (≥3)
5. For qualified tickers: fetch option chains for each target expiration
6. Score and rank options contracts
7. Assemble report in the format above
8. Send summary to KakaoTalk via `KakaotalkChat-MemoChat`
9. Return full report in conversation

**Performance tip:** Batch tickers in groups of 5 to avoid rate limits. Start with high-conviction names (NVDA, TSLA, AAPL, QQQ, SPY).
