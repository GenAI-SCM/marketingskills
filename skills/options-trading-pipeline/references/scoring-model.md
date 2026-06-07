# 확신 점수 모델 — Unusual Whales 단독

## 종목 확신 점수 (MAX 10점)

### Layer 2 — 확신 스태킹 (4점)

| UW 툴 | 신호 | 점수 |
|-------|------|------|
| `get_ticker_lit_flow` | 콜(풋) 방향 플로우 우세 | +1 |
| `get_interval_flow` | 최근 1시간 플로우 가속 중 | +1 |
| `get_dark_pool_trades` | 기관 블록이 같은 방향 | +1 |
| `get_open_interest_changes` | OI +20% 이상 신규 오픈 | +1 |

### Layer 3 — 가격 레벨 일치 (4점)

| UW 툴 | 신호 | 점수 |
|-------|------|------|
| `get_greek_exposure_by_strike` | GEX 방향 플로우와 일치 | +1 |
| `get_greek_exposure_by_strike` | GEX 절벽 레벨 명확 | +1 |
| `get_max_pain` | 맥스페인 대비 현재가 위치 유리 | +1 |
| `get_dark_pool_volume_price_group` | 다크풀 지지 현재가 근처 | +1 |

### Layer 4 — 기술 확인 (2점)

| UW 툴 | 신호 | 점수 |
|-------|------|------|
| `get_ticker_indicator_events` | MACD 크로스 방향 일치 | +1 |
| `get_extended_technical_indicator` | VWAP 위(콜)/아래(풋) | +1 |

---

## 점수별 판정

| 점수 | 등급 | 액션 |
|------|------|------|
| 9~10 | ●●●● TIER 1 | 최대 사이즈 진입 |
| 7~8 | ●●●○ TIER 2 | 보통 사이즈 진입 |
| 5~6 | ●●○○ TIER 3 | 소규모 or 관망 |
| < 5 | ●○○○ SKIP | 진입 금지 |

---

## 계약 우선순위

```
1순위: 플로우 집중 행사가 = GEX 절벽 행사가
       (get_flow_per_strike + get_greek_exposure_by_strike 일치)

2순위: 플로우 집중 행사가
       (get_flow_per_strike 기준)

3순위: GEX 절벽 직전 행사가
       (get_greek_exposure_by_strike 기준)
```

## 만기 우선순위

```
1순위: get_flow_per_expiry 집중 만기
2순위: W+3 기본값 (균형)
3순위: W+4 (이벤트 대기 or 확신 낮을 때)
W+2: 플로우 매우 강할 때만 (빠른 시간가치 손실 주의)
```
