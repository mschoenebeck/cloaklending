# CLOAK Lending

**Private credit, dynamic risk pricing, and on-chain insurance.**

CLOAK Lending is a privacy-enabled, overcollateralized lending protocol for Antelope-based blockchains. It combines two lending markets, risk-sensitive borrowing fees, active insurer capital, protocol-directed bailouts, a final reserve, and a narrow Savings backstop inside one integrated financial system.

> In this project, **private credit** means credit conducted through the CLOAK Shielded Protocol—not the corporate private-credit asset class.

## At a Glance

CLOAK Lending supports two distinct borrowing directions:

```text
Crypto collateral  →  borrow VIGOR
VIGOR collateral   →  borrow supported crypto
```

The two sides share protocol infrastructure but remain separate for collateral checks, risk pricing, fees, insolvency detection, and bailout handling.

The protocol also includes:

- **active insurers** that supply borrowable crypto inventory and absorb losses;
- **dynamic risk pricing** based on collateral composition, volatility, correlation, solvency, and liquidity utilization;
- **VIGOR Savings**, which earns a share of VIG fees while serving as a narrow final stable-side backstop;
- a **final reserve** behind the normal insurer pool;
- **side-specific bailouts** that resolve only the debt side that actually failed;
- **shielded identities and private action flows** through the CLOAK protocol;
- **deterministic fixed-point execution** using [`fp128`](https://github.com/mschoenebeck/fp128);
- a **staged, resumable epoch engine** designed for Antelope/EOSIO WASM execution constraints.

## Core Economic Structure

```text
                         Borrowing fees
                              │
          ┌───────────────────┼───────────────────┐
          ↓                   ↓                   ↓
    Active insurers        Savings         Final reserve
          │                   │                   │
          └──────────── Default waterfall ───────┘

Crypto collateral  ↔  VIGOR borrowing
VIGOR collateral   ↔  Crypto borrowing
```

Unlike a conventional lending pool, insurers do more than provide passive capital. Active insurer assets:

- create live crypto borrowing capacity, subject to each token's configured hard lending cap;
- earn risk-weighted and liquidity-weighted fee revenue;
- inherit distressed debt and collateral during bailouts;
- recapitalize inherited positions using their own insurance capital;
- remain exposed during delayed withdrawal periods.

The default waterfall is predefined:

```text
Distressed borrower position
          ↓
Active normal insurers
          ↓
Final reserve
          ↓
Savings backstop
(stable side only; narrow final case)
```

## VIGOR and VIG

| Token | Role in CLOAK Lending |
|---|---|
| **VIGOR** | USD-referenced synthetic low-volatility token, stable debt, crypto-borrowing collateral, Savings asset, and protocol unit of account |
| **VIG** | CLOAK token used for protocol fees and fee rewards |

VIGOR is not a fiat-backed dollar claim and does not have a guaranteed one-dollar redemption facility. Its intended stability depends on overcollateralization, repayment demand, borrowing incentives, market liquidity, insurer capital, reserve support, and confidence in the protocol.

## Dynamic Risk Pricing

Borrowing rates are not based on utilization alone.

The protocol considers:

- collateral and debt value;
- asset volatility;
- portfolio correlation;
- stressed loss estimates;
- global insurer solvency;
- token-level configured lending-cap utilization and scarcity;
- configured minimum and maximum rate bounds.

The resulting rates are stored separately for the two lending directions:

- `tesprice` for VIGOR debt backed by crypto;
- `l_tesprice` for crypto debt backed by VIGOR.

Risk-contribution weights also influence both insurer fee rewards and bailout allocation. Capital that contributes more to modeled system protection may earn more—and absorb more when losses materialize.

## Privacy

CLOAK Lending integrates directly with the [CLOAK / ZEOS Caterpillar Shielded Protocol](https://github.com/mschoenebeck/zeos-caterpillar).

The protocol supports ordinary public-account actions and shielded equivalents using CLOAK's `zidentity` authorization model. This allows users to manage collateral, debt, insurance, Savings, and repayments without exposing the same public account-level financial history as a conventional transparent lending application.

Privacy does not make the economic rules unverifiable. State transitions remain deterministic and enforced by the public smart-contract system.

## Deterministic Execution

Financial calculations use [`fp128`](https://github.com/mschoenebeck/fp128), a signed 128-bit decimal fixed-point arithmetic type.

This provides:

- deterministic prices and portfolio values;
- explicit precision and rounding;
- reproducible fee calculations;
- deterministic insurer and bailout allocations;
- bounded arithmetic suitable for integer token accounting.

Global repricing, fee collection, insolvency processing, and bailouts are executed through a staged and resumable epoch pipeline. This preserves the economic model while keeping individual Antelope/EOSIO transactions bounded.

## Live Interface

The current CLOAK Lending interface is available at:

**[Open CLOAK Lending](https://app.cloak.today/lending/dashboard)**

The application route may change as the CLOAK web application evolves.

## Repository Status

This is currently a **public documentation repository**.

The production smart-contract implementation and supporting protocol code are proprietary and are **not included** in this repository.

CLOAK Lending is in late-stage development and approaching launch readiness. This repository and its documentation are not:

- a security audit;
- a guarantee of mainnet readiness;
- financial or investment advice;
- a promise of solvency, liquidity, stable pricing, or loss-free operation.

The implementation may be published in whole or in part in the future, but no source-code release is promised by this repository.

## Documentation

The complete technical and economic guide covers:

- the two lending markets;
- VIGOR supply and stability incentives;
- protocol balance-sheet mechanics;
- Savings and insurer economics;
- dynamic risk pricing;
- fees and reward distribution;
- insolvency, bailouts, and recapitalization;
- the default waterfall;
- oracle and market dependencies;
- privacy and deterministic execution;
- protocol risks and limitations;
- mathematical formulas, terminology, actions, and operational dependencies.

**[Read the complete CLOAK Lending guide](docs/cloak-lending.md)**

## Important Risks and Dependencies

CLOAK Lending depends on more than deployed code.

A production deployment requires robust operation of:

- oracle publication and methodology;
- VIGOR market liquidity;
- VIG liquidity for fee payment;
- insurer recruitment and capital depth;
- final-reserve capitalization;
- reliable `tick()` execution;
- contract permissions, governance, and upgrade controls;
- testing, review, and security processes.

The protocol is designed to allocate and process losses. It cannot guarantee that sufficient capital, liquidity, or market confidence will always exist.

## Related Work

- [CLOAK](https://cloak.today)
- [CLOAK application](https://app.cloak.today)
- [ZEOS Caterpillar Shielded Protocol](https://github.com/mschoenebeck/zeos-caterpillar)
- [`fp128`](https://github.com/mschoenebeck/fp128)
- [Native Hybrid Exchange Engine](https://github.com/mschoenebeck/hybrid-exchange-engine)

## Author

**Matthias Schönebeck**

Blockchain and DeFi infrastructure engineer focused on deterministic trading systems, lending infrastructure, privacy protocols, and applied cryptography.

- [GitHub](https://github.com/mschoenebeck)
- [X / Twitter](https://x.com/mschoenebeck1)
- [Telegram](https://t.me/mschoenebeck)
- [Email](mailto:matthias.schoenebeck@gmail.com)

## Rights

Copyright © Matthias Schönebeck.

The repository currently publishes documentation only. No license to the proprietary CLOAK Lending implementation is granted or implied.
