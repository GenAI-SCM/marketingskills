# Unusual Whales MCP — 95개 툴 전체 분류

MCP Endpoint: `https://api.unusualwhales.com/api/mcp`

---

## 카테고리별 툴 분류

### 1. 옵션 플로우 (Options Flow) — 핵심 신호

| 툴 | 용도 | 파이프라인 활용 |
|----|------|--------------|
| `get_flow_alerts` | 실시간 이상 옵션 플로우 알림 | ★★★ 1차 트리거 |
| `get_ticker_lit_flow` | 티커별 장내(lit) 옵션 플로우 | ★★★ 방향 확인 |
| `get_interval_flow` | 시간대별 플로우 집계 | ★★★ 장중 모멘텀 |
| `get_flow_per_expiry` | 만기별 플로우 분포 | ★★☆ 만기 선택 근거 |
| `get_flow_per_strike` | 행사가별 플로우 분포 | ★★☆ 행사가 선택 근거 |
| `get_flow_alerts` | 커스텀 알림 기반 플로우 | ★★☆ |
| `get_flow_alert_rules` | 알림 규칙 조회 | ★☆☆ |
| `get_flow_watchlist` | 워치리스트 플로우 | ★★☆ |
| `get_options_chain` | 옵션 체인 (플로우 컨텍스트 포함) | ★★★ |
| `get_chains_for_expiry` | 특정 만기 체인 | ★★☆ |
| `get_historic_chains` | 과거 체인 데이터 | ★☆☆ |
| `search_option_contracts` | 계약 검색 | ★★☆ |
| `get_option_trades` | 개별 옵션 거래 내역 | ★★☆ |

### 2. 그릭 익스포저 (Greek Exposure / GEX)

| 툴 | 용도 | 파이프라인 활용 |
|----|------|--------------|
| `get_greek_exposure_by_ticker` | 티커별 GEX 총량 | ★★★ 주요 레벨 파악 |
| `get_greek_exposure_by_expiry` | 만기별 GEX | ★★☆ 만기 선택 |
| `get_greek_exposure_by_strike` | 행사가별 GEX | ★★★ 저항/지지 레벨 |
| `get_greek_exposure_by_strike_expiry` | 행사가×만기 GEX 매트릭스 | ★★☆ |
| `get_greek_flow` | 그릭 플로우 방향성 | ★★☆ |
| `get_greek_flow_by_expiry` | 만기별 그릭 플로우 | ★☆☆ |
| `get_max_pain` | 맥스페인 레벨 | ★★★ 만기 자석 레벨 |
| `get_open_interest_changes` | OI 변화량 | ★★★ 기관 포지션 추적 |

### 3. 다크풀 (Dark Pool)

| 툴 | 용도 | 파이프라인 활용 |
|----|------|--------------|
| `get_dark_pool_trades` | 다크풀 거래 내역 | ★★★ 기관 매집 신호 |
| `get_dark_pool_volume_price_group` | 가격대별 다크풀 볼륨 | ★★★ 기관 평단가 파악 |

### 4. 기술 지표 (Technical Indicators)

| 툴 | 용도 | 파이프라인 활용 |
|----|------|--------------|
| `get_ticker_indicator_series` | MA, MACD, RSI 등 지표 시리즈 | ★★★ 기술 분석 |
| `get_ticker_indicator_events` | 지표 이벤트 (MA 크로스, MACD 크로스) | ★★★ 크로스오버 감지 |
| `get_extended_technical_indicator` | VWAP 등 고급 지표 | ★★★ VWAP 계산 |
| `get_ticker_candles_by_range` | OHLCV 캔들 | ★★☆ 가격 데이터 |
| `get_ticker_ohlc_latest_or_date` | 최신/특정일 OHLC | ★★☆ |
| `get_ticker_close_prices` | 종가 시리즈 | ★★☆ |
| `get_ticker_performances` | 가격 퍼포먼스 | ★★☆ |

### 5. 시장 전체 현황

| 툴 | 용도 | 파이프라인 활용 |
|----|------|--------------|
| `get_market_tide` | 시장 전체 풋/콜 플로우 비율 | ★★★ 시장 방향성 |
| `get_market_state` | 현재 시장 상태 | ★★★ 장전 컨텍스트 |
| `get_market_events` | 시장 이벤트 캘린더 | ★★☆ |
| `get_futures_indices` | 선물/인덱스 현황 | ★★☆ |
| `get_trading_states` | 거래 상태 | ★☆☆ |
| `get_yield_curve` | 국채 수익률 곡선 | ★☆☆ |
| `get_central_bank_rates` | 중앙은행 금리 | ★☆☆ |

### 6. 실적 & 이벤트 (Earnings / Events)

| 툴 | 용도 | 파이프라인 활용 |
|----|------|--------------|
| `get_upcoming_earnings` | 예정 실적 캘린더 | ★★★ 리스크 필터 |
| `get_earnings_history` | 과거 실적 반응 | ★★☆ |
| `get_earnings_report` | 실적 상세 | ★☆☆ |
| `get_earnings_screener` | 실적 스크리너 | ★☆☆ |

### 7. 스크리너 (Screeners)

| 툴 | 용도 | 파이프라인 활용 |
|----|------|--------------|
| `get_options_screener` | 옵션 조건 스크리닝 | ★★★ 계약 필터링 |
| `get_stock_screener` | 종목 스크리닝 | ★★☆ |
| `get_short_data_by_ticker` | 공매도 데이터 | ★★☆ |
| `get_short_screener` | 공매도 스크리너 | ★☆☆ |
| `get_short_volume_ratio_by_ticker` | 티커별 공매도 비율 | ★★☆ |
| `get_short_volume_ratio_by_exchange` | 거래소별 공매도 비율 | ★☆☆ |

### 8. 스마트머니 / 예측 (Smart Money / Predictions)

| 툴 | 용도 | 파이프라인 활용 |
|----|------|--------------|
| `get_prediction_smart_money` | 스마트머니 방향 예측 | ★★★ 보조 신호 |
| `get_prediction_whales` | 고래 포지션 예측 | ★★☆ |
| `get_prediction_unusual_markets` | UW 모델 시장 예측 | ★★☆ |
| `get_prediction_insiders` | 인사이더 기반 예측 | ★☆☆ |
| `get_prediction_user` | 유저 예측 | ★☆☆ |
| `get_prediction_market` | 전체 시장 예측 | ★★☆ |
| `get_prediction_watchlist` | 워치리스트 예측 | ★☆☆ |
| `get_prediction_watchlists` | 복수 워치리스트 예측 | ★☆☆ |

### 9. 기관/내부자/의회 (Institutional / Insider / Congress)

| 툴 | 용도 | 파이프라인 활용 |
|----|------|--------------|
| `get_institution_holdings` | 기관 보유 현황 | ★★☆ |
| `get_institutions` | 기관 리스트 | ★☆☆ |
| `get_insider_transactions` | 내부자 거래 내역 | ★★☆ |
| `get_insider_ticker_flow` | 티커별 내부자 플로우 | ★★☆ |
| `get_insider_activity_by_ticker` | 티커별 내부자 활동 | ★★☆ |
| `get_insider_sector_flow` | 섹터별 내부자 플로우 | ★☆☆ |
| `get_congress_trades` | 의회 거래 내역 | ★★☆ |
| `get_politics_overview` | 정치인 거래 개요 | ★☆☆ |
| `get_midterms_ratings` | 중간선거 관련 | ★☆☆ |
| `get_trump_activity` | 트럼프 트레이드 활동 | ★★☆ |
| `get_prediction_insiders` | 인사이더 예측 | ★☆☆ |

### 10. 재무제표 (Fundamentals)

| 툴 | 용도 | 파이프라인 활용 |
|----|------|--------------|
| `get_income_statements` | 손익계산서 | ★☆☆ |
| `get_balance_sheets` | 대차대조표 | ★☆☆ |
| `get_cash_flows` | 현금흐름표 | ★☆☆ |
| `get_fundamental_breakdown` | 펀더멘털 종합 | ★☆☆ |
| `get_income_statement_screener` | 손익 스크리너 | ★☆☆ |
| `get_balance_sheet_screener` | 재무상태 스크리너 | ★☆☆ |
| `get_cash_flow_screener` | 현금흐름 스크리너 | ★☆☆ |
| `get_analyst_ratings` | 애널리스트 레이팅 | ★★☆ |
| `get_company_info` | 기업 기본 정보 | ★☆☆ |
| `get_correlations` | 티커 간 상관관계 | ★★☆ |
| `get_seasonality_month` | 월별 계절성 | ★★☆ |

### 11. 크립토 (Crypto)

| 툴 | 용도 |
|----|------|
| `get_crypto_ohlc_candles` | 크립토 OHLC |
| `get_crypto_pair_state` | 크립토 페어 상태 |
| `get_crypto_whale_transactions` | 크립토 고래 거래 |
| `get_recent_crypto_whale_trades` | 최근 크립토 고래 거래 |

### 12. 워치리스트 / 사용자 도구

| 툴 | 용도 |
|----|------|
| `get_stock_watchlist` | 주식 워치리스트 |
| `get_options_watchlist` | 옵션 워치리스트 |
| `get_users_flow_watchlists` | 플로우 워치리스트들 |
| `get_users_dark_pool_watchlists` | 다크풀 워치리스트들 |
| `get_users_interval_flow_watchlists` | 인터벌 플로우 워치리스트 |
| `get_users_oi_changes_watchlists` | OI 변화 워치리스트 |
| `get_users_options_screener_watchlists` | 옵션 스크리너 워치리스트 |
| `get_users_options_watchlists` | 옵션 워치리스트들 |
| `get_users_stock_watchlists` | 주식 워치리스트들 |
| `get_user_super_flow_dashboards` | 슈퍼 플로우 대시보드 |
| `get_flow_alert_rules` | 플로우 알림 규칙 |
| `get_custom_alerts` | 커스텀 알림 |
| `get_custom_alert_args` | 커스텀 알림 인자 |

### 13. 기타

| 툴 | 용도 |
|----|------|
| `get_public_api_docs` | API 문서 조회 |
| `get_support_info` | 지원 정보 |
| `search_tickers` | 티커 검색 |
