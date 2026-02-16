# Requirements: Inlet

**Defined:** 2026-02-16
**Core Value:** 유저가 INJ 담보를 예치하고, 외부 가격 피드 기반으로 합성 롱/숏 포지션을 열어 레버리지 노출을 얻을 수 있어야 한다.

## v1 Requirements

Requirements for initial release. Each maps to roadmap phases.

### Oracle & Price Feed

- [ ] **ORAC-01**: 마켓별 가격 피드 저장 (latest_price, last_updated_timestamp, market_id)
- [ ] **ORAC-02**: Staleness 체크 — 60초 이상 오래된 가격 업데이트 거부
- [ ] **ORAC-03**: injective-price-oracle 연동 — Injective Labs 공식 오라클 활용
- [ ] **ORAC-04**: TWAP 가격 계산 — 청산 시 가격 조작 방지용 시간가중평균가격

### Position Management

- [ ] **POSN-01**: User can deposit INJ as collateral to a market contract
- [ ] **POSN-02**: User can open long/short position with up to 10x leverage
- [ ] **POSN-03**: User can manually close a full position and receive collateral +/- PnL
- [ ] **POSN-04**: PnL is calculated in real-time: long=(current-entry)*size, short=(entry-current)*size
- [ ] **POSN-05**: User can partially close a position (reduce size proportionally)
- [ ] **POSN-06**: User can add or withdraw collateral from an existing position
- [ ] **POSN-07**: User can adjust leverage on an existing position

### Risk & Liquidation

- [ ] **RISK-01**: MMR(Maintenance Margin Ratio) 기반 청산 조건 — collateral+pnl <= MMR threshold
- [ ] **RISK-02**: 누구나 liquidate(position_id) 호출 가능 — permissionless liquidation
- [ ] **RISK-03**: 청산자에게 liquidation_fee 보상 지급
- [ ] **RISK-04**: Bad debt 추적 — collateral+pnl < 0인 경우 프로토콜 손실 기록
- [ ] **RISK-05**: Insurance fund — bad debt 커버용 보험 풀 운영
- [ ] **RISK-06**: 부분 청산 — 포지션 일부만 청산하여 margin ratio 회복

### Multi-Market Factory

- [ ] **FACT-01**: 팩토리 컨트랙트가 마켓 컨트랙트를 생성 (SubMsg+Reply 패턴)
- [ ] **FACT-02**: 마켓별 설정 가능 (MMR, max_leverage, collateral_denom, oracle_config)
- [ ] **FACT-03**: 마켓별 독립 컨트랙트 — 마켓 간 격리
- [ ] **FACT-04**: 마켓 pause/resume — 긴급 시 거래 일시정지
- [ ] **FACT-05**: Admin 마이그레이션 — 컨트랙트 업그레이드 지원

### Off-chain Relayer

- [ ] **RELY-01**: injective-ts SDK로 update_price tx를 온체인에 전송
- [ ] **RELY-02**: 주기적 가격 릴레이 (configurable interval)
- [ ] **RELY-03**: 실제 가격 피드 연동 (CoinGecko/Binance API)

### Testing

- [ ] **TEST-01**: injective-test-tube 기반 Rust 통합 테스트 환경 구축
- [ ] **TEST-02**: E2E 시나리오 테스트 — 포지션 오픈 → 가격 변동 → PnL 변화 → 청산 트리거

## v2 Requirements

Deferred to future release. Tracked but not in current roadmap.

### Oracle Enhancements

- **ORAC-V2-01**: Deviation guard — 비정상 가격 점프 거부 (circuit breaker)
- **ORAC-V2-02**: 다중 오라클 소스 집계 — 여러 소스의 가격 합산
- **ORAC-V2-03**: WasmX 자동 실행 — BeginBlocker로 오라클/청산 자동화

### Risk Enhancements

- **RISK-V2-01**: ADL (Auto-Deleveraging) — insurance fund 소진 시 이익 포지션 자동 감소
- **RISK-V2-02**: Cross-margin — 마켓 간 담보 공유

### Relayer Enhancements

- **RELY-V2-01**: 다중 릴레이어 — 단일 장애점 제거
- **RELY-V2-02**: Mock price generator — 테스트/데모용 가짜 가격 생성기

## Out of Scope

| Feature | Reason |
|---------|--------|
| 매칭 엔진 / 오더북 | 오더북 DEX가 아님 — 합성 노출 엔진 |
| LP/AMM 풀 | 유동성 제공 구조 불필요, 프로토콜이 카운터파티 |
| 펀딩 레이트 | MVP에서 0으로 스텁 — mark/index 스프레드 없음 |
| 프론트엔드 UI | 컨트랙트 + 릴레이어만 스코프 |
| 메인넷 배포 | 테스트넷 프로토타입 단계 |
| 오더 타입 (limit, stop-loss) | 합성 포지션 엔진이므로 불필요 |
| Multi-collateral | INJ native만 — MVP 단순화 |

## Traceability

Which phases cover which requirements. Updated during roadmap creation.

| Requirement | Phase | Status |
|-------------|-------|--------|
| ORAC-01 | — | Pending |
| ORAC-02 | — | Pending |
| ORAC-03 | — | Pending |
| ORAC-04 | — | Pending |
| POSN-01 | — | Pending |
| POSN-02 | — | Pending |
| POSN-03 | — | Pending |
| POSN-04 | — | Pending |
| POSN-05 | — | Pending |
| POSN-06 | — | Pending |
| POSN-07 | — | Pending |
| RISK-01 | — | Pending |
| RISK-02 | — | Pending |
| RISK-03 | — | Pending |
| RISK-04 | — | Pending |
| RISK-05 | — | Pending |
| RISK-06 | — | Pending |
| FACT-01 | — | Pending |
| FACT-02 | — | Pending |
| FACT-03 | — | Pending |
| FACT-04 | — | Pending |
| FACT-05 | — | Pending |
| RELY-01 | — | Pending |
| RELY-02 | — | Pending |
| RELY-03 | — | Pending |
| TEST-01 | — | Pending |
| TEST-02 | — | Pending |

**Coverage:**
- v1 requirements: 27 total
- Mapped to phases: 0
- Unmapped: 27 ⚠️

---
*Requirements defined: 2026-02-16*
*Last updated: 2026-02-16 after initial definition*
