# Inlet

## What This Is

Injective 체인 위에서 동작하는 합성 파생상품(Synthetic Derivatives) 엔진. 오더북 DEX가 아니라, 외부 가격 인덱스를 기반으로 합성 포지션(롱/숏)을 열고 관리하는 어플리케이션 레벨 엔진이다. CosmWasm 스마트 컨트랙트 + TypeScript 오프체인 릴레이어로 구성되며, Injective 테스트넷 배포를 목표로 한다.

## Core Value

유저가 INJ 담보를 예치하고, 외부 가격 피드 기반으로 합성 롱/숏 포지션을 열어 레버리지 노출을 얻을 수 있어야 한다.

## Requirements

### Validated

(None yet — ship to validate)

### Active

- [ ] 오라클 가격 피드 (injective-price-oracle 활용)
- [ ] 팩토리 패턴으로 멀티마켓 지원 (BTC/USD, ETH/USD 등)
- [ ] INJ 담보 예치/인출
- [ ] 롱/숏 포지션 오픈 (최대 10x 레버리지)
- [ ] 포지션 수동 클로즈
- [ ] PnL 실시간 계산
- [ ] MMR 기반 청산 메커니즘
- [ ] TypeScript 오프체인 릴레이어 (가격 업데이트 전송)
- [ ] injective-test-tube 기반 통합 테스트

### Out of Scope

- 매칭 엔진 — 오더북 DEX가 아님, 합성 노출 엔진
- LP/AMM 풀 — 유동성 제공 구조 불필요
- 펀딩 레이트 — MVP에서는 0으로 스텁
- 프론트엔드 UI — 컨트랙트 + 릴레이어만
- 메인넷 배포 — 테스트넷 프로토타입 단계

## Context

**아키텍처: 팩토리 패턴**
- 팩토리 컨트랙트: 마켓 컨트랙트 생성/관리
- 마켓 컨트랙트: 개별 거래쌍의 포지션, 담보, 청산 처리
- 유저당 마켓당 여러 포지션 가능

**오픈소스 레퍼런스:**
- `InjectiveLabs/cw-injective` — CosmWasm 바인딩 + 예제 컨트랙트 (베이스 템플릿)
- `InjectiveLabs/injective-ts` — Node/TS 클라이언트 SDK (릴레이어용)
- `InjectiveLabs/test-tube` (injective-test-tube) — Rust 통합 테스트
- `InjectiveLabs/injective-price-oracle` — 가격 오라클 (가격 피드 소스)

**포지션 구조:**
```
owner, side (long/short), size, entry_price, collateral, leverage, opened_at
```

**PnL 계산:**
```
long_pnl  = (current_price - entry_price) * size
short_pnl = (entry_price - current_price) * size
```

**청산 조건:**
```
collateral + pnl <= MMR threshold → 누구나 liquidate(position_id) 호출 가능
청산자에게 liquidation_fee 보상
```

**레포 구조 (예상):**
```
/contracts
  /factory    — 팩토리 컨트랙트
  /market     — 마켓 컨트랙트 (오라클, 포지션, 청산)
/relayer      — TypeScript 릴레이어
/tests        — injective-test-tube 통합 테스트
```

## Constraints

- **체인**: Injective (CosmWasm) — 테스트넷 배포
- **담보 토큰**: INJ native만 지원
- **레버리지**: 최대 10x
- **가격 피드**: injective-price-oracle 활용, 릴레이어가 주기적 업데이트
- **설계 철학**: clarity over completeness — 최소한이지만 현실적인 안전 장치 (stale price 거부, 청산)

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| 팩토리 패턴 (마켓별 컨트랙트) | 마켓 간 격리, 확장성, 독립적 관리 가능 | — Pending |
| INJ native 담보만 | MVP 단순화, CW20 핸들링 불필요 | — Pending |
| injective-price-oracle 활용 | Injective Labs 공식 오라클, 생태계 표준 활용 | — Pending |
| 최대 10x 레버리지 | 보수적 한도, 청산 로직 검증 용이 | — Pending |
| 마켓당 다중 포지션 | 유연한 전략 구성 가능, 실제 프로토콜과 유사 | — Pending |

---
*Last updated: 2026-02-16 after initialization*
