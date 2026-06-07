# Options Trading Pipeline — Architecture Design

## 전체 데이터 플로우

```
[09:00 AM ET — 장 시작 30분 전]

TRIGGER
  │
  ▼
┌─────────────────────────────────────────┐
│  PHASE 1: TECHNICAL SCREENING           │
│  Input: 30 master tickers               │
│                                         │
│  MCP: get_historical_stock_prices()     │
│  - period="3mo", interval="1d"          │
│  - 계산: MA1, MA2, MA5, MA20, MACD      │
│                                         │
│  MCP: get_historical_stock_prices()     │
│  - period="1d", interval="5m"           │
│  - 계산: VWAP, 당일 거래량              │
│                                         │
│  OUTPUT: 시그널 점수 (0-5) per ticker   │
│  → 점수 ≥ 3인 티커만 PHASE 2로         │
└─────────────────────────────────────────┘
  │ (예: 8-12개 티커 통과)
  ▼
┌─────────────────────────────────────────┐
│  PHASE 2: RISK FILTER                   │
│                                         │
│  MCP: get_finance_news()                │
│  - 실적 발표 10일 이내 → EXCLUDE        │
│  - 주요 이벤트(FDA, M&A) 플래그         │
│                                         │
│  MCP: get_stock_info()                  │
│  - earningsDate 확인                    │
│  - beta, shortPercentOfFloat 확인       │
│  - 52주 레인지 위치 확인                 │
│                                         │
│  MCP: get_recommendations()             │
│  - 최근 3일 업그레이드/다운그레이드      │
│                                         │
│  OUTPUT: 클린 후보 5-8개 티커           │
└─────────────────────────────────────────┘
  │
  ▼
┌─────────────────────────────────────────┐
│  PHASE 3: OPTIONS CHAIN ANALYSIS        │
│                                         │
│  MCP: get_option_expiration_dates()     │
│  - W+2, W+3, W+4 만기일 추출            │
│  - (오늘 기준 10~28일 이내 금요일)       │
│                                         │
│  MCP: get_option_chain()                │
│  - option_type: "calls" or "puts"       │
│    (시그널 방향에 따라)                  │
│  - ATM ± 5% 범위 행사가 필터링          │
│                                         │
│  계산:                                  │
│  - 델타 범위 0.30-0.55 필터             │
│  - 스프레드율 = (ask-bid)/mid × 100     │
│  - Vol/OI 비율                          │
│  - IV vs 30일 평균 IV 비교              │
│                                         │
│  OUTPUT: 만기별 추천 계약 1-2개         │
└─────────────────────────────────────────┘
  │
  ▼
┌─────────────────────────────────────────┐
│  PHASE 4: REPORT ASSEMBLY & DELIVERY    │
│                                         │
│  - 점수 기반 Top 5 픽 선정              │
│  - 전체 리포트 마크다운 생성             │
│                                         │
│  MCP: KakaotalkChat-MemoChat()          │
│  - 요약본 (Top 5 테이블 + 핵심 메시지)  │
│                                         │
│  OUTPUT: 대화창 full report             │
└─────────────────────────────────────────┘
```

---

## 기술적 지표 계산 로직

### MA 계산 (Python-style pseudo code)

```python
# 입력: get_historical_stock_prices() 결과
closes = [row['Close'] for row in price_data]  # 최신순 정렬

MA1 = closes[0]                    # 당일 종가
MA2 = sum(closes[0:2]) / 2         # 2일 이동평균
MA5 = sum(closes[0:5]) / 5         # 5일 이동평균
MA20 = sum(closes[0:20]) / 20      # 20일 이동평균

# 불리시 정렬 확인
bullish_ma = MA1 > MA2 > MA5       # 단기 정렬
trend_up = MA5 > MA20              # 중기 추세
```

### MACD 계산

```python
def ema(prices, period):
    k = 2 / (period + 1)
    ema_val = prices[0]
    for price in prices[1:]:
        ema_val = price * k + ema_val * (1 - k)
    return ema_val

ema12 = ema(closes[:12], 12)
ema26 = ema(closes[:26], 26)
macd_line = ema12 - ema26

# Signal line: EMA(9) of MACD
# 교차 확인: 현재 MACD > Signal AND 전일 MACD < Signal → 상향돌파
```

### VWAP 계산 (장중 5분봉 기준)

```python
# 입력: period="1d", interval="5m"
vwap_num = sum(row['Typical'] * row['Volume'] for row in intraday)
# Typical = (High + Low + Close) / 3
vwap_denom = sum(row['Volume'] for row in intraday)
vwap = vwap_num / vwap_denom
```

---

## 옵션 계약 스코어링 공식

```
계약 점수 (100점 만점) =

  델타 점수    (25점): 0.40-0.50 범위 = 25점, 벗어날수록 감점
+ 유동성 점수  (25점): Vol/OI > 1.0 = 25점, 스프레드 < 5% = bonus
+ IV 점수      (25점): IV < 25% = 25점 (저렴), IV > 45% = 0점 (비쌈)
+ 만기 점수    (25점): W+2 = 25점 (최대 레버리지),
                       W+3 = 20점, W+4 = 15점
```

**최적 진입 조건:**
- 델타: 0.35–0.50
- IV: 장기 IV 평균 대비 낮을수록 유리 (프리미엄 매수)
- 스프레드: mid 대비 10% 이내
- Vol/OI: 0.1 이상 (당일 거래 활성)

---

## 만기일 계산 로직

```
오늘 = T
W+2 = 다음 다음 금요일 (T+10 ~ T+14)
W+3 = 3번째 금요일 (T+15 ~ T+21)
W+4 = 4번째 금요일 (T+22 ~ T+28)

get_option_expiration_dates() 결과에서
위 범위에 해당하는 날짜를 필터링
```

---

## MCP 호출 최적화

### 배치 전략 (API 부하 최소화)

```
Round 1 (동시): SPY, QQQ, NVDA, TSLA, AAPL  → 주요 5개 먼저
Round 2 (동시): MSFT, META, AMD, COIN, PLTR
Round 3 (동시): GOOGL, AMZN, MSTR, SOFI, HOOD
...

각 라운드 후 점수 계산 → 점수 높은 순으로 옵션 체인 조회
옵션 체인 조회는 통과 티커만 (보통 5-8개)
```

### 예상 MCP 호출 수 (일일)

| 단계 | 도구 | 호출수 |
|------|------|--------|
| 일봉 가격 | get_historical_stock_prices | 30회 |
| 분봉 VWAP | get_historical_stock_prices | 30회 |
| 뉴스 필터 | get_finance_news | 10회 (통과 티커) |
| 종목 정보 | get_stock_info | 10회 |
| 만기일 조회 | get_option_expiration_dates | 8회 |
| 옵션 체인 | get_option_chain | 24회 (8종목 × W+2/3/4) |
| **합계** | | **~112회** |

---

## 리포트 생성 일정

| 시간 (ET) | 작업 | 출력 |
|-----------|------|------|
| 09:00 | 장전 스캔 실행 | 기술 스코어보드 |
| 09:15 | 옵션 체인 분석 | 후보 리스트 |
| 09:30 | 리포트 완성 | 카카오톡 발송 |
| 14:00 | 장중 업데이트 (선택) | 포지션 재검토 |
| 15:45 | 청산 알림 (선택) | 일일 마감 점검 |

---

## 확장 가능한 보고서 유형

| 리포트 | 설명 | 추가 MCP 활용 |
|--------|------|--------------|
| **일일 옵션 픽** | 핵심 (현재 설계) | — |
| **주간 전략 리뷰** | 지난 주 픽 성과 분석 | get_historical_stock_prices |
| **IV 스크리너** | 전체 티커 IV 히트맵 | get_option_chain |
| **실적 이벤트 레이더** | 2주 앞 실적 캘린더 | get_finance_news + get_stock_info |
| **기관 포지션 추적** | 인사이더/기관 동향 | get_holder_info |
| **애널리스트 모멘텀** | 최근 업/다운그레이드 급증 | get_recommendations |
| **섹터 로테이션 맵** | ETF 상대강도 분석 | get_historical_stock_prices(ETFs) |
