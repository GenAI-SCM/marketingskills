# Options Trading Pipeline — 최종 아키텍처 설계

## MCP 소스 2개 통합

| MCP | 강점 | 역할 |
|-----|------|------|
| **Unusual Whales** (95 tools) | 옵션 플로우, 다크풀, GEX, 스마트머니, 기술지표 직접 제공 | 주력 데이터 소스 |
| **UsStockInfo** (yfinance) | 재무제표, 인사이더, 뉴스, 추천 | 보조 데이터 소스 |

---

## 전체 데이터 파이프라인

```
[장 시작 30분 전 — 09:00 AM ET]

══════════════════════════════════════════════════════
 PHASE 0: 시장 컨텍스트 수집 (한번만)
══════════════════════════════════════════════════════

UW: get_market_state()
  → 현재 시장 상태 (bullish/bearish/neutral)
  → VIX 레벨 확인

UW: get_market_tide()
  → 전체 풋/콜 플로우 방향 (시장 센티먼트)
  → 콜 > 풋 비율이면 시장 전체 불리시

UW: get_futures_indices()
  → 선물 갭업/갭다운 확인

※ 시장이 극단적 공포 상태 (VIX > 35) → 리포트에 경고 표시

══════════════════════════════════════════════════════
 PHASE 1: 기술적 스크리닝 (30 티커 전체)
══════════════════════════════════════════════════════

[Unusual Whales로 기술 지표 직접 수집]

UW: get_ticker_indicator_events(ticker)
  → MA 크로스오버 이벤트 감지 (MA5 > MA20 돌파 등)
  → MACD 골든크로스 / 데드크로스 이벤트

UW: get_extended_technical_indicator(ticker)
  → VWAP 현재값
  → 추가 고급 지표

UW: get_ticker_indicator_series(ticker, indicator="MA", period=5/20)
  → MA5, MA20 시리즈
  → RSI, Bollinger Band 등

UW: get_ticker_performances(ticker)
  → 1d, 5d, 1mo 수익률
  → 모멘텀 점수

※ 기술 시그널 점수 계산 (0-5점):
  +1: MA 단기 정렬 (MA1 > MA2 > MA5)
  +1: MA 중기 추세 (MA5 > MA20)
  +1: MACD 상향 돌파
  +1: 가격 > VWAP
  +1: 모멘텀 양호 (5d return > 0)

  → 점수 ≥ 3 이면 다음 단계로

══════════════════════════════════════════════════════
 PHASE 2: 옵션 플로우 확인 (통과 티커, ~10개)
══════════════════════════════════════════════════════
        ★ Unusual Whales 핵심 기능 ★

UW: get_ticker_lit_flow(ticker)
  → 해당 티커의 콜/풋 플로우 방향
  → 대형 주문 (10만달러 이상) 있는지 확인
  → "이상 플로우"(unusual) 플래그 여부

UW: get_interval_flow(ticker)
  → 최근 시간대별 플로우 집계
  → 플로우 가속도 측정 (최근 1시간 급등 여부)

UW: get_flow_per_expiry(ticker)
  → W+2, W+3, W+4 만기별 플로우 집중도
  → 어느 만기에 큰 돈이 몰리는지

UW: get_flow_per_strike(ticker)
  → 행사가별 플로우
  → 어느 행사가에 베팅이 집중되는지

UW: get_open_interest_changes(ticker)
  → OI 급증 행사가 파악
  → 기관이 새로 포지션 잡는 레벨

플로우 확인 점수 (0-3점):
  +1: 콜(불리시) 플로우 > 풋 플로우
  +1: 대형 주문 (프리미엄 $50k+) 존재
  +1: OI 급증 (전일 대비 20% 이상)

══════════════════════════════════════════════════════
 PHASE 3: 다크풀 & 스마트머니 확인
══════════════════════════════════════════════════════

UW: get_dark_pool_trades(ticker)
  → 최근 다크풀 대형 거래
  → 기관 평단가 추정
  → 다크풀 거래 방향 (매수/매도)

UW: get_dark_pool_volume_price_group(ticker)
  → 가격대별 다크풀 볼륨 분포
  → 기관이 가장 많이 산 가격 = 지지선

UW: get_prediction_smart_money(ticker)
  → UW 스마트머니 방향 예측
  → Bull/Bear/Neutral

다크풀 점수 (0-2점):
  +1: 다크풀 매수 우세
  +1: 스마트머니 예측 일치

══════════════════════════════════════════════════════
 PHASE 4: GEX & 맥스페인 분석
══════════════════════════════════════════════════════
        ★ 옵션 진입 레벨 결정에 핵심 ★

UW: get_greek_exposure_by_ticker(ticker)
  → 총 GEX 값 (양수 = 시장안정, 음수 = 변동성 확대)
  → GEX 절벽 레벨 파악

UW: get_greek_exposure_by_strike(ticker, expiry)
  → 행사가별 GEX
  → 가장 큰 GEX 행사가 = 주가 자석 레벨
  → 그 위/아래를 깨면 가속 구간

UW: get_max_pain(ticker, expiry)
  → 만기일의 맥스페인 레벨
  → 주가가 맥스페인 위면 Put 매도 유리
  → 아래면 Call 매도 유리
  ※ 프리미엄 매수 관점: 맥스페인 반대방향 베팅

GEX 활용:
  - GEX 절벽 아래/위 = 가속 구간 진입 타겟
  - 맥스페인 대비 현재가 위치로 방향 확인

══════════════════════════════════════════════════════
 PHASE 5: 리스크 필터 (이벤트/실적)
══════════════════════════════════════════════════════

UW: get_upcoming_earnings()
  → 10일 이내 실적 발표 티커 → 제외

UsStockInfo: get_finance_news(ticker)
  → 주요 뉴스, M&A, FDA 이슈 확인

UW: get_short_data_by_ticker(ticker)
  → 공매도 비율 확인
  → Short squeeze 가능성 있는 종목 플래그

══════════════════════════════════════════════════════
 PHASE 6: 옵션 계약 선정
══════════════════════════════════════════════════════

UW: get_options_chain(ticker, expiry)  [W+2, W+3, W+4]
  → 계약 목록 + 그릭스 + 플로우 컨텍스트 포함
  → Delta, IV, Volume, OI, Bid/Ask

UW: get_options_screener(filters)
  → 조건: 델타 0.30-0.55, 스프레드 < 10%,
         Vol/OI > 0.1, IV < 40%

선택 기준:
  1. 플로우 집중 만기 우선
  2. OI 급증 행사가 근처
  3. 스프레드율 < 10%
  4. 델타 0.35-0.50

══════════════════════════════════════════════════════
 PHASE 7: 리포트 생성 & 발송
══════════════════════════════════════════════════════

  Full report → 대화창 출력
  KakaoTalk → KakaotalkChat-MemoChat (Top 5 요약)
```

---

## 최종 종합 점수 체계

```
종목 종합 점수 (MAX 10점):

기술 점수   (5점): MA 정렬, MACD, VWAP, 모멘텀
플로우 점수 (3점): 콜/풋 플로우, 대형 주문, OI 급증
다크풀 점수 (2점): 기관 매수, 스마트머니 일치

─────────────────────
총점 8-10: TIER 1 ★★★★★ → 최우선 진입
총점 6-7:  TIER 2 ★★★★☆ → 진입 고려
총점 4-5:  TIER 3 ★★★☆☆ → 관망
총점 < 4:  SKIP
```

---

## 만기별 전략 매핑

| 만기 | 특성 | 적합 상황 | GEX/맥스페인 활용 |
|------|------|----------|-----------------|
| **W+2** (10-14일) | 최대 레버리지, 시간 손실 빠름 | 강한 모멘텀 + 높은 플로우 점수 | GEX 절벽 레벨 돌파 직후 |
| **W+3** (15-21일) | 균형, 기본 추천 | 중간 신호 | 맥스페인 방향 일치 시 |
| **W+4** (22-28일) | 보수적, 시간 여유 | 이벤트 (실적 직후 등) 대기 | 큰 OI 집중 만기 |

---

## Unusual Whales 독점 인사이트

### 의회/내부자 트레이드 모니터링

```
UW: get_congress_trades()
  → 의원들의 옵션 매수 패턴 추적
  → 특정 티커 집중 매수 = 선행 정보 가능성

UW: get_trump_activity()
  → 트럼프 관련 거래 동향

UsStockInfo: get_insider_ticker_flow(ticker)
  → 내부자 대량 매수 = 강한 불리시 신호
```

### 계절성 & 상관관계

```
UW: get_seasonality_month(ticker, month)
  → 현재 월의 역사적 평균 수익률
  → 예: NVDA는 11월 평균 +X%

UW: get_correlations(tickers)
  → SPY 상관성이 높은 종목 파악
  → 헷지 페어 찾기
```

---

## 예상 MCP 호출 수 (일일)

| 페이즈 | 툴 | 호출수 |
|--------|-----|--------|
| Phase 0: 시장 컨텍스트 | get_market_state, get_market_tide, get_futures_indices | 3회 |
| Phase 1: 기술 스크리닝 | get_ticker_indicator_events × 30 | 30회 |
| Phase 1: 추가 지표 | get_extended_technical_indicator × 30 | 30회 |
| Phase 2: 플로우 확인 | get_ticker_lit_flow + get_interval_flow × 10 | 20회 |
| Phase 2: 만기/행사가 | get_flow_per_expiry + per_strike × 10 | 20회 |
| Phase 3: 다크풀 | get_dark_pool_trades × 8 | 8회 |
| Phase 3: 스마트머니 | get_prediction_smart_money × 8 | 8회 |
| Phase 4: GEX | get_greek_exposure_by_strike × 6 | 6회 |
| Phase 4: 맥스페인 | get_max_pain × 6 × 3만기 | 18회 |
| Phase 5: 실적 필터 | get_upcoming_earnings (1회) | 1회 |
| Phase 6: 옵션 체인 | get_options_chain × 5종목 × 3만기 | 15회 |
| **합계** | | **~159회** |

---

## 가능한 리포트 유형 (Unusual Whales 기반 확장)

| 리포트 | 핵심 툴 | 주기 |
|--------|---------|------|
| **일일 옵션 픽** | flow_alerts + tech indicators | 매일 장전 |
| **이상 플로우 레이더** | get_flow_alerts (실시간) | 장중 알림 |
| **GEX 레벨 맵** | greek_exposure_by_strike | 매주 월요일 |
| **다크풀 매집 모니터** | dark_pool_trades | 매일 장후 |
| **의회/내부자 트레이드** | congress_trades + insider | 주간 |
| **실적 이벤트 레이더** | upcoming_earnings | 매주 일요일 |
| **스마트머니 방향 요약** | prediction_smart_money | 매일 |
| **OI 변화 추적** | open_interest_changes | 매일 장후 |
| **계절성 분석** | seasonality_month | 월초 |
| **공매도 압박 스캔** | short_screener + short_data | 매주 |
