---
name: options-trading-pipeline
description: Daily options trading report for short-term premium buying (call/put). Use when the user says "옵션 리포트", "daily options scan", "call put 제안", "플로우 스캔", "weekly options picks". Uses Unusual Whales MCP exclusively. Flow-first approach targeting W+2/W+3/W+4 expirations, holding under 1 week.
metadata:
  author: bokangkim
  version: 3.0.0
  category: trading
---

# Options Trading Daily Pipeline
## Unusual Whales MCP 단독 운용

MCP: `https://api.unusualwhales.com/api/mcp`  
배달: `KakaotalkChat-MemoChat`

---

## 실행 순서

### LAYER 0 — 시장 레짐 (4 calls)

```
get_market_state()
get_market_tide()
get_greek_exposure_by_ticker("SPY")
get_futures_indices()
```

| GEX SPY | Market Tide | 판정 |
|---------|-------------|------|
| 음수 | BULLISH | CALL 풀사이즈 |
| 음수 | BEARISH | PUT 풀사이즈 |
| 양수 | — | 사이즈 50% 축소 |
| — | VIX > 35 | 전체 보류 |

---

### LAYER 1 — 플로우 디스커버리 (1 call)

```
get_flow_alerts()
```

→ 이상 플로우 발생 종목 전체 수신  
→ 마스터 30종목 교집합 추출, $50k+ 대형 주문 우선  
→ 오늘 집중할 종목 5~10개 도출  

**이 1 call이 30종목 기술분석 전체를 대체한다**

---

### LAYER 2 — 확신 스태킹 (관심 종목당 4 calls)

```
get_ticker_lit_flow(ticker)
get_interval_flow(ticker)
get_dark_pool_trades(ticker)
get_open_interest_changes(ticker)
```

| 신호 | 조건 | 점수 |
|------|------|------|
| lit_flow | 방향 일치 | +1 |
| interval_flow | 가속 중 | +1 |
| dark_pool | 같은 방향 | +1 |
| OI | +20%↑ 신규 오픈 | +1 |

**3점 이상만 다음으로**

---

### LAYER 3 — 가격 레벨 (확신 3점+ 종목당 3 calls)

```
get_greek_exposure_by_strike(ticker, expiry)
get_max_pain(ticker, expiry)
get_dark_pool_volume_price_group(ticker)
```

- **GEX 절벽**: 돌파 시 가격 가속 → 1차 타겟
- **맥스페인**: 현재가 > 맥스페인 → CALL / 아래 → PUT
- **다크풀 집중 레벨**: 기관 평단가 = 진입 기준점

---

### LAYER 4 — 기술 확인 (종목당 2 calls)

```
get_ticker_indicator_events(ticker)
get_extended_technical_indicator(ticker)
```

- MA/MACD 크로스오버 이벤트
- VWAP 위/아래 포지션 확인
- 플로우와 방향 불일치 시 경고 플래그

---

### LAYER 5 — 리스크 게이트 (global 1 + 종목당 2 calls)

```
get_upcoming_earnings()              → 10일 이내 실적 → 제외
get_market_events()                  → 매크로 이벤트 확인
get_short_data_by_ticker(ticker)     → 공매도 비율 / 숏 스퀴즈 여부
```

---

### LAYER 6 — 계약 선정 (종목당 5 calls)

```
get_flow_per_expiry(ticker)
get_flow_per_strike(ticker)
get_options_chain(ticker, W+2_expiry)
get_options_chain(ticker, W+3_expiry)
get_options_chain(ticker, W+4_expiry)
```

행사가 선택 우선순위:
1. 플로우 집중 행사가 = GEX 절벽 행사가 (최강)
2. 플로우 집중 행사가 단독
3. GEX 절벽 기준 ATM+5%

만기 선택:
- 플로우 집중 만기 우선
- 기본값: W+3
- 이벤트 대기: W+4

계약 필터: 델타 0.30~0.55 | 스프레드 < 10% | Vol/OI > 0.1 | IV < 35%

---

## 5-Section 리포트

```
═══════════════════════════════════════════════════
 OPTIONS DAILY REPORT — {DATE}  {TIME} ET
═══════════════════════════════════════════════════

[SECTION 1] 시장 환경
GEX(SPY): ±$XXXm → [방향성 폭발 / 변동성 억제]
Market Tide: [BULLISH / BEARISH / NEUTRAL]
VIX: XX.X | ES: ±X% | NQ: ±X%
오늘 전략: [CALL 풀사이즈 / PUT 풀사이즈 / 50% 축소 / 보류]

[SECTION 2] Market Tide
콜: $XXXm | 풋: $XXXm | 비율: X.XX
방향: [BULLISH ↑ / BEARISH ↓ / NEUTRAL →]

[SECTION 3] 플로우 Top 20
순위 | 티커 | 방향  | 대형주문 | OI변화 | 다크풀 | 확신
  1  | NVDA | CALL↑ | $XXXk  | +XX%  | 매수  | ●●●●
...

[SECTION 4] 콜/풋 추천
### CALL #1 — NVDA [●●●●]
현재가 $XXX | GEX절벽 $XXX | 맥스페인 $XXX | 다크풀지지 $XXX

만기       | 행사가  | 프리미엄 | 델타 | IV  | 근거
W+2 MM/DD | $XXX C | $X.XX   | 0.45 | 24% | 플로우 집중
W+3 MM/DD | $XXX C | $X.XX   | 0.40 | 23% | [추천] GEX절벽
W+4 MM/DD | $XXX C | $X.XX   | 0.35 | 22% | 보수적

[SECTION 5] 손익분기 / 타겟 / 손절
BEP:   행사가 + 프리미엄 = $XXX.XX (+X.X%)
1차:   GEX절벽 $XXX → 프리미엄 +50~60%
2차:   절벽 돌파 후 $XXX → 프리미엄 +80~120%
손절:  프리미엄 -40% OR 다크풀지지 $XXX 이탈
시간:  만기 3일 전 잔량 전량 청산
R/R:   1차 1:1.4 / 2차 1:2.2
진입:  장 시작 15분 후 | VWAP 위 | 다크풀지지 확인

⚠️ 실적: [없음 / TICKER MM/DD — 진입 금지]
═══════════════════════════════════════════════════
```

---

## 호출 요약

| Layer | Calls |
|-------|-------|
| 0 시장 레짐 | 4 |
| 1 플로우 디스커버리 | 1 |
| 2 확신 스태킹 × 8종목 | 32 |
| 3 가격 레벨 × 5종목 | 15 |
| 4 기술 확인 × 5종목 | 10 |
| 5 리스크 게이트 | 11 |
| 6 계약 선정 × 5종목 | 25 |
| **총계** | **~98 calls** |
