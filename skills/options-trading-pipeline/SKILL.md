---
name: options-trading-pipeline
description: Daily options trading report for short-term premium buying (call/put). Use when the user says "옵션 리포트", "daily options scan", "call put 제안", "플로우 스캔", "weekly options picks". Uses Unusual Whales MCP (flow-first approach) + yfinance MCP to identify high-conviction directional plays with W+2/W+3/W+4 expirations. Target holding: under 1 week.
metadata:
  author: bokangkim
  version: 2.0.0
  category: trading
---

# Options Trading Daily Pipeline

## 설계 원칙

**기술분석 먼저 (X)** → **플로우 먼저 (O)**

시장은 스마트머니가 지금 어디 베팅하는지 스스로 알려준다.
`get_flow_alerts` 한 번이 30종목 차트 분석 전부를 대체한다.
GEX(델타 헷지 노출)가 주가의 실질 인력과 척력 레벨을 만든다.

## MCP 구성

| MCP | 역할 |
|-----|------|
| **Unusual Whales** (`api.unusualwhales.com/api/mcp`) | 주력: 플로우, GEX, 다크풀, 기술지표 |
| **UsStockInfo** (yfinance) | 보조: 뉴스, 재무, 추천 |
| **KakaotalkChat-MemoChat** | 리포트 발송 |

---

## 실행 순서 (7 Layers)

### LAYER 0 — 시장 레짐 판단 (항상 먼저, 4 calls)

```
get_market_state()
get_market_tide()
get_greek_exposure_by_ticker("SPY")
get_greek_exposure_by_ticker("QQQ")
```

판정표:

| GEX | Market Tide | 판정 |
|-----|------------|------|
| 음수 | BULLISH | CALL 풀사이즈 진입 |
| 음수 | BEARISH | PUT 풀사이즈 진입 |
| 양수 | 어느쪽이든 | 사이즈 50% 축소 |
| — | VIX > 35 | 전체 보류 |

GEX 음수 = 마켓메이커 같은 방향 헷지 = 방향성 폭발 구간 = 프리미엄 매수 최적


### LAYER 1 — 플로우 디스커버리 (1 call)

```
get_flow_alerts()
```

→ 오늘 이상 옵션 플로우 발생 종목 전체 수신  
→ 마스터 30종목과 교집합 추출  
→ 프리미엄 $50k 이상 대형 주문 포함 종목 우선  

**이 한 번의 호출이 30종목 기술분석을 대체한다.**  
결과: 오늘 집중할 종목 5~10개 도출


### LAYER 2 — 확신 스태킹 (관심 종목당 4 calls)

```
get_ticker_lit_flow(ticker)       → 콜/풋 방향 확인
get_interval_flow(ticker)         → 플로우 가속 여부
get_dark_pool_trades(ticker)      → 기관 블록 매수 존재?
get_open_interest_changes(ticker) → 신규 포지션(확신) vs 청산
```

확신 점수 (0~4점):
- lit_flow 방향 일치: +1
- 플로우 가속 중: +1
- 다크풀 같은 방향: +1
- OI 급증 (+20%↑): +1

**3점 이상만 다음 레이어로 진행**


### LAYER 3 — 가격 레벨 인텔리전스 (확신 3점+ 종목당 3 calls)

```
get_greek_exposure_by_strike(ticker, expiry)
get_max_pain(ticker, expiry)
get_dark_pool_volume_price_group(ticker)
```

도출 정보:
- **GEX 절벽**: 이 레벨 돌파 시 가격 가속 → 1차 타겟
- **GEX 벽**: 저항/지지로 작동하는 레벨
- **맥스페인**: 현재가 > 맥스페인 → CALL 유리 / 아래 → PUT 유리
- **다크풀 집중 레벨**: 기관 평단가 = 실질 지지선 → 진입 기준점


### LAYER 4 — 기술적 확인 (종목당 2 calls)

```
get_ticker_indicator_events(ticker)        → MA/MACD 크로스오버 이벤트
get_extended_technical_indicator(ticker)   → VWAP, 볼린저밴드
```

플로우 방향과 기술적 방향이 일치 → 확신 추가  
불일치 → 경고 플래그 (진입 보류 고려)


### LAYER 5 — 리스크 게이트 (global 1 call + 종목당 1 call)

```
get_upcoming_earnings()               → 10일 이내 실적 → 즉시 제외
get_short_data_by_ticker(ticker)      → 공매도 비율 확인
```

공매도 비율 > 20% + 불리시 플로우 = 숏 스퀴즈 잠재력 → 콜 매수 추가 근거


### LAYER 6 — 계약 선정 (최종 후보 종목당 최대 6 calls)

```
get_flow_per_expiry(ticker)           → 스마트머니가 선택한 만기
get_flow_per_strike(ticker)           → 스마트머니가 베팅한 행사가
get_options_chain(ticker, W+2_expiry) → 체인 상세
get_options_chain(ticker, W+3_expiry)
get_options_chain(ticker, W+4_expiry)
```

**행사가 선택 우선순위:**
1. 플로우 집중 행사가 = GEX 절벽 행사가 (최강)
2. 플로우 집중 행사가 단독
3. GEX 절벽 기반 ATM+5%

**만기 선택:**
- 플로우가 W+2에 집중 → W+2 (모멘텀 강할 때)
- 기본값 → W+3 (균형)
- 이벤트 대기 → W+4 (시간 여유)

**계약 필터:**
- 델타: 0.30 ~ 0.55
- 스프레드율: < 10%
- Vol/OI: > 0.1
- IV: < 35% (프리미엄 매수이므로 저렴할 때)


### LAYER 7 — 리포트 생성 & 발송

5섹션 리포트 조립 후 `KakaotalkChat-MemoChat`으로 요약 발송

---

## 5-Section Daily Report

```
═══════════════════════════════════════════════════
 OPTIONS DAILY REPORT — {DATE}  {TIME} ET
 전략: W+2~W+4 프리미엄 매수 | 보유: 1주 이내
═══════════════════════════════════════════════════

SECTION 1 — 시장 환경
─────────────────────────────────────────────────
GEX (SPY): [+$XXXm 양수 | -$XXXm 음수]
환경 판정: [변동성 억제 구간 | 방향성 폭발 구간]
Market Maker 헷지 방향: [매수 | 매도]
VIX: XX.X | 선물: ES ±X% | NQ ±X%
오늘 전략: [CALL 풀사이즈 | PUT 풀사이즈 | 50% 축소 | 보류]

SECTION 2 — Market Tide 방향
─────────────────────────────────────────────────
콜 프리미엄: $XXXm | 풋 프리미엄: $XXXm
콜/풋 비율: X.XX
센티먼트: [BULLISH ↑ | BEARISH ↓ | NEUTRAL →]
오늘 전략 방향: [CALL 집중 | PUT 집중 | 선별적]

SECTION 3 — 옵션 플로우 Top 20
─────────────────────────────────────────────────
순위 | 티커 | 방향  | 대형주문 | OI변화 | 다크풀 | 확신
  1  | NVDA | CALL↑ | $XXXk   | +XX%   | 매수  | ●●●●
  2  | TSLA | PUT↓  | $XXXk   | +XX%   | 중립  | ●●●○
...
 20  | AAPL | CALL↑ | $XXk    | +X%    | 매수  | ●●○○

SECTION 4 — 콜/풋 추천 + 행사가격
─────────────────────────────────────────────────
### CALL PICK #1 — NVDA [확신 ●●●●]

현재가: $XXX.XX
GEX 절벽(1차 타겟): $XXX  [돌파시 $XXX까지 가속]
다크풀 지지선: $XXX  [기관 평단가, 진입 기준]
맥스페인 W+3: $XXX  [현재가 > 맥스페인 → CALL 유리]

추천 계약:
만기       | 행사가  | 프리미엄 | 델타 | IV  | 선택 근거
W+2 MM/DD | $XXX C | $X.XX   | 0.45 | 24% | 플로우 집중 만기
W+3 MM/DD | $XXX C | $X.XX   | 0.40 | 23% | [기본 추천] GEX 절벽 근처
W+4 MM/DD | $XXX C | $X.XX   | 0.35 | 22% | 보수적

### PUT PICK #1 — TSLA [확신 ●●●○]
(동일 포맷)

SECTION 5 — 손익분기 / 타겟 / 손절
─────────────────────────────────────────────────
NVDA W+3 $XXX Call @ $X.XX 프리미엄:

손익분기(BEP): $XXX + $X.XX = $XXX.XX (현재 대비 +X.X%)
1차 타겟:      $XXX (GEX 절벽) → 프리미엄 +50~60% 예상
2차 타겟:      $XXX (절벽 돌파 후) → 프리미엄 +80~120% 예상
손절:          프리미엄 -40% OR 다크풀 지지 $XXX 이탈
시간 손절:     만기 3일 전 잔량 전량 청산
R/R:           1차 기준 1:1.4 / 2차 기준 1:2.2

진입 조건:
  · 장 시작 15분 후 (가격 안정 확인)
  · 현재가 > VWAP 확인 (콜 기준)
  · 다크풀 지지 $XXX 위에서 진입

⚠️ 실적 경고: [해당 없음 | TICKER MM/DD — 진입 금지]

═══════════════════════════════════════════════════
생성: {TIMESTAMP} ET | 다음: 익일 09:00 ET
═══════════════════════════════════════════════════
```

---

## 총 MCP 호출 수

| 레이어 | 호출수 |
|--------|--------|
| Layer 0: 시장 레짐 | 4 |
| Layer 1: 플로우 디스커버리 | 1 |
| Layer 2: 확신 스태킹 × 8종목 | 32 |
| Layer 3: 가격 레벨 × 5종목 | 15 |
| Layer 4: 기술 확인 × 5종목 | 10 |
| Layer 5: 리스크 게이트 | 6 |
| Layer 6: 계약 선정 × 5종목 | 30 |
| **합계** | **~98 calls** |

---

## 트리거

"옵션 리포트 실행" 또는 "run options report" 또는 "daily scan" 입력 시:
1. Layer 0~7 순서대로 실행
2. 5섹션 리포트 전체 출력
3. KakaoTalk으로 섹션 1~3 요약 발송
