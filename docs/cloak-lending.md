# CLOAK Lending

*Private Credit, Dynamic Risk Pricing, and On-Chain Insurance*

**Technical and Economic Guide — July 2026**

Here, **private credit** means credit conducted through CLOAK’s shielded privacy layer—not the corporate private-credit asset class.

CLOAK Lending is a private, overcollateralized lending protocol built around a simple but unusual idea: lending, liquidity provision, risk pricing, insurance, recapitalization, and systemic backstops should work as one financial system rather than as separate products.

A user can borrow the protocol’s low-volatility token, **VIGOR**, against crypto collateral. The same user can also go the other way and borrow supported crypto assets against VIGOR collateral. Active insurers provide the crypto inventory that can be borrowed, earn a share of protocol fees, and stand ready to absorb distressed positions. A final reserve sits behind the insurer pool, while the VIGOR Savings pool earns fee revenue and acts as a narrow last-resort backstop.

All of this is integrated with the CLOAK Shielded Protocol. Users can interact through shielded identities and private action flows rather than exposing their financial activity through an ordinary public account.

This document explains how the system works, why the incentives exist, where the mathematics enters, and what the design does—and does not—guarantee.

> **Reading guide**
>
> - New to the protocol? Read Sections 1–7, 11, and 16.
> - Interested in finance and risk? Focus on Sections 4–12 and Appendix A.
> - Interested in implementation? Focus on Sections 8–15 and Appendices B and C.

> **Status note**
>
> This is technical and economic documentation, not an audit, a guarantee of mainnet readiness, or investment advice.

> **Implementation basis**
>
> This guide reflects the CLOAK Lending v1 development state confirmed on July 22, 2026, including side-specific bailout selection and participation by debtor-owned active insurance. Before public release, the document should be pinned to a canonical repository commit and the configuration of the deployment it describes.

---

## Contents

1. [CLOAK Lending at a Glance](#1-cloak-lending-at-a-glance)
2. [The Tokens and the People Using Them](#2-the-tokens-and-the-people-using-them)
3. [Two Lending Markets, One Protocol](#3-two-lending-markets-one-protocol)
4. [The Protocol Balance Sheet](#4-the-protocol-balance-sheet)
5. [How VIGOR Aims to Stay Near One Dollar](#5-how-vigor-aims-to-stay-near-one-dollar)
6. [Why External Market Liquidity Matters](#6-why-external-market-liquidity-matters)
7. [Savings: Demand, Revenue, and Tail Risk](#7-savings-demand-revenue-and-tail-risk)
8. [Active Insurers and Borrowable Liquidity](#8-active-insurers-and-borrowable-liquidity)
9. [Dynamic Risk Pricing](#9-dynamic-risk-pricing)
10. [Fees and Incentives](#10-fees-and-incentives)
11. [Insolvency, Bailouts, and Recapitalization](#11-insolvency-bailouts-and-recapitalization)
12. [The Default Waterfall](#12-the-default-waterfall)
13. [Oracles, Markets, and Trust Assumptions](#13-oracles-markets-and-trust-assumptions)
14. [Privacy and Deterministic Execution](#14-privacy-and-deterministic-execution)
15. [How CLOAK Lending Compares](#15-how-cloak-lending-compares)
16. [Risks, Limits, and What CLOAK Lending Is Not](#16-risks-limits-and-what-cloak-lending-is-not)
17. [A Complete Worked Example](#17-a-complete-worked-example)
18. [Appendix A — Mathematical Reference](#appendix-a--mathematical-reference)
19. [Appendix B — Terminology and Code Mapping](#appendix-b--terminology-and-code-mapping)
20. [Appendix C — Actions and Operational Dependencies](#appendix-c--actions-and-operational-dependencies)
21. [References](#references)

---

## 1. CLOAK Lending at a Glance

Most lending protocols are easy to describe at first: lenders deposit assets, borrowers post collateral, and liquidators step in when a position becomes unsafe.

CLOAK Lending is broader than that.

It has two lending directions:

```text
Crypto collateral  →  borrow VIGOR
VIGOR collateral   →  borrow crypto
```

It also has a dedicated class of **active insurers**. These insurers do more than passively lend assets. Their capital:

- supplies the crypto inventory that borrowers can draw;
- earns a share of borrowing fees;
- is measured by how much risk it removes from the system;
- absorbs distressed debt and collateral when a borrower fails;
- recapitalizes inherited positions after a bailout.

Behind the normal insurer pool sits a **final reserve**. Behind that, in one narrow stable-side failure case, sits the **Savings pool**.

The result looks less like a simple lending pool and more like a small on-chain credit system:

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

The protocol is also private by design. Native public-account actions exist, but equivalent shielded actions use CLOAK’s `zidentity` authorization model. Borrowing, repayment, collateral management, insurance, and Savings can therefore be used through the CLOAK Shielded Protocol.

The key idea is simple:

> **CLOAK Lending turns lending, risk underwriting, and loss absorption into one integrated market.**

---

## 2. The Tokens and the People Using Them

Before looking at formulas, it helps to know the cast.

### 2.1 VIGOR: the low-volatility token

**VIGOR** is the protocol’s USD-referenced low-volatility token.

It is best understood as a synthetic dollar:

- its intended market target is approximately one US dollar;
- it is created when users borrow it against crypto collateral;
- it is retired when VIGOR debt is repaid or canceled with VIGOR;
- it is used as collateral for borrowing crypto;
- it can be deposited into Savings;
- it may also be supplied as insurer capital;
- it is the protocol’s internal unit of account.

VIGOR is not described here as a dollar in a bank account. It is not a legal claim on fiat reserves and it does not come with a guaranteed one-dollar redemption facility. Its stability depends on collateral, insurance capital, demand, supply contraction, market liquidity, and confidence in the protocol.

### 2.2 VIG: the CLOAK token and protocol fee token

**VIG is the CLOAK token used by CLOAK Lending as its fee token.**

Borrowing fees are economically calculated in VIGOR-value terms, then converted into VIG using the VIG oracle price. Borrowers therefore pay protocol fees in VIG, and fee recipients—including insurers and savers—receive VIG.

This distinction matters throughout the document:

| Token | Main role in CLOAK Lending |
|---|---|
| **VIGOR** | USD-referenced low-volatility token, stable debt, crypto-borrowing collateral, Savings asset, possible insurer capital |
| **VIG** | CLOAK token, protocol fee token, fee-reward asset |

A borrower can owe VIGOR while paying the borrowing fee in VIG. A saver deposits VIGOR but earns a share of fees in VIG.

VIG is special because it is the fee and reward token, but it is not excluded from the wider asset model. When configured as a supported token, VIG can also be held in collateral or insurance and can participate in ordinary crypto borrowing and repayment flows.

### 2.3 Borrowers

A borrower can take either side of the protocol:

- deposit crypto and borrow VIGOR;
- deposit VIGOR and borrow supported crypto.

The same account may hold both kinds of debt at once. The two sides remain separate for collateral checks, pricing, fees, insolvency, and bailout handling.

### 2.4 Active insurers

An active insurer places assets into the `insurance` bucket and enters the active insurer set.

Active insurer capital has two jobs:

1. provide borrowable crypto liquidity;
2. absorb protocol risk.

Insurers earn fees because their capital is exposed. When a bailout happens, they can inherit a proportional share of debt and collateral and may need to move their own insurance assets into collateral to restore the inherited position.

### 2.5 Savers

A saver deposits VIGOR into the Savings pool.

In normal operation, the saver earns a proportional share of the configured Savings allocation from protocol fees. Those rewards are paid in VIG.

In an extreme event, Savings is also a narrow final backstop for stable-side reserve insolvency. Savings is therefore not risk-free yield.

### 2.6 Final reserve

The final reserve is a special protocol backstop.

It:

- holds capital in its own `insurance` bucket;
- receives a configured share of fees;
- may receive bailout collateral cuts;
- receives redirected insurer fees if no normal insurer capital exists;
- remains outside the normal active-insurer weighting;
- absorbs debt only after normal insurer capital has been exhausted.

### 2.7 Tick callers

CLOAK Lending updates risk, fees, and insolvencies in epochs. Anyone can call `tick()` to advance an overdue epoch.

The caller whose tick actually completes the epoch can receive a configured VIG reward. That helps keep the system moving, but a production deployment should still run reliable tick automation.

### 2.8 One user can wear several hats

These are economic roles, not exclusive identities.

Alice can:

- borrow VIGOR;
- keep some VIGOR in Savings;
- supply other assets as active insurance;
- later absorb part of a bailout, including part of her own bailout if she is still an active insurer.

That overlap is intentional. The protocol accounts for each bucket separately.

---

## 3. Two Lending Markets, One Protocol

CLOAK Lending supports two mirror-image debt modes.

They share an account system, oracle infrastructure, risk engine, and default waterfall, but they should not be mentally collapsed into one generic loan.

### 3.1 Mode A: borrow VIGOR against crypto

Alice deposits BTC, EOS, or another supported crypto asset into her `collateral` bucket.

She then borrows VIGOR:

```text
BTC collateral
      ↓
VIGOR debt is created
      ↓
New VIGOR is issued to Alice
```

The basic borrow-time requirement is:

```math
\text{crypto collateral value}
\ge
\text{new VIGOR debt}
\times
\text{required collateral ratio}
```

If Alice later repays, she sends VIGOR back to the contract. The debt is reduced and the repaid VIGOR is retired.

This is the stable-debt side of the system.

### 3.2 Mode B: borrow crypto against VIGOR

Bob deposits VIGOR into his `stable_collateral` bucket.

He then borrows a supported crypto asset from the live inventory supplied by active insurers:

```text
VIGOR collateral
      ↓
Crypto debt is created
      ↓
Borrowed crypto leaves active insurer inventory
```

The basic borrow-time requirement is:

```math
\text{VIGOR collateral}
\ge
\text{new crypto debt value}
\times
\text{required collateral ratio}
```

Bob does not receive newly minted crypto. The borrowed asset must already exist inside active insurer capital and be available after subtracting currently lent amounts.

When Bob repays the crypto, the returned tokens replenish protocol-held liquidity.

### 3.3 Why the distinction matters

The two sides behave differently:

| Feature | Stable-debt side | Crypto-debt side |
|---|---|---|
| Borrowed asset | VIGOR | Whitelisted crypto |
| Collateral | Crypto | VIGOR |
| Supply source | New VIGOR issuance | Active insurer inventory |
| Risk price | `tesprice` | `l_tesprice` |
| Bailout weight | `pcts` | `l_pcts` |
| Main market shock | Collateral falls | Borrowed crypto rises |
| Recapitalization | Crypto insurance can move to collateral | VIGOR and crypto insurance can support stable collateral |

An account with both debts can be healthy on one side and unhealthy on the other. CLOAK Lending therefore selects bailout sides independently. It does not liquidate a healthy debt side just because the other side failed.

---

## 4. The Protocol Balance Sheet

A useful way to understand CLOAK Lending is to stop thinking in terms of buttons and start thinking in terms of assets, liabilities, and contingent risk.

### 4.1 Borrower view

For a stable borrower:

| Alice’s position | Economic meaning |
|---|---|
| Crypto in `collateral` | Asset pledged to the protocol |
| `stable_debt` | VIGOR liability |
| VIG in `vigfees` | Balance available to pay protocol fees |

For a crypto borrower:

| Bob’s position | Economic meaning |
|---|---|
| VIGOR in `stable_collateral` | Asset pledged to the protocol |
| `crypto_debt` | Liability in the borrowed token |
| VIG in `vigfees` | Balance available to pay protocol fees |

### 4.2 Insurer view

An insurer’s `insurance` bucket is both an asset portfolio and a risk-capital account.

A crypto token held by an active insurer can be:

- available to borrow;
- already lent to a crypto borrower;
- used to cancel matching inherited debt in a bailout;
- moved into collateral during recapitalization.

VIGOR held in insurance can cancel inherited stable debt directly or support crypto-side recapitalization.

### 4.3 VIGOR creation and destruction

Within CLOAK Lending’s own issuance lifecycle, VIGOR creation and retirement follow stable debt.

#### Creation

When a user successfully borrows VIGOR:

```text
new stable debt recorded
        +
equivalent VIGOR issued
```

#### Destruction

VIGOR is retired when:

- a borrower repays stable debt;
- an insurer uses VIGOR insurance to cancel inherited stable debt;
- Savings VIGOR is used to cancel final-reserve stable debt.

A bailout does not automatically destroy all debt. It can first transfer debt and collateral from the distressed borrower to absorbing insurers. Supply contracts only where VIGOR is actually used and retired against that debt.

At system level, outstanding VIGOR equals CLOAK Lending stable debt only if the deployment makes CLOAK Lending the exclusive net issuer—or separately accounts for every other issuance and retirement path. The lending contract’s debt accounting cannot, by itself, prove that no other token authority can change supply.

### 4.4 Crypto lending is inventory-based

Crypto borrowing is different.

The contract does not create BTC, EOS, or another borrowed asset. It tracks live active-insurer inventory:

```math
\text{available}_j
=
\max(
\text{active insurance}_j
-
\text{live lent}_j,
0
)
```

for token $j$.

That creates a hard limit: a user cannot borrow more of a token than active insurers have made available.

### 4.5 The economic loop

At a high level:

```text
Borrowers receive capital
        ↓
Borrowers pay VIG fees
        ↓
Insurers, savers, reserve, and tick callers earn VIG
        ↓
Insurer and reserve capital absorb failures
        ↓
Solvency and pricing are recomputed
```

The system is not free of losses. It decides in advance who is paid for bearing them and in what order capital is used when they occur.

---

## 5. How VIGOR Aims to Stay Near One Dollar

VIGOR is designed to be a low-volatility token that tracks the US dollar.

It is important to describe this honestly:

> **The protocol creates incentives that can pull VIGOR toward one dollar. It does not provide a guaranteed one-dollar redemption.**

Four separate ideas are often mixed together in stable-token discussions. CLOAK Lending keeps them conceptually distinct.

### 5.1 The target

The economic target is approximately:

```math
1\ \text{VIGOR} \approx 1\ \text{USD}
```

This is the public meaning of “low-volatility token” or “stable token.”

### 5.2 The internal unit of account

Inside CLOAK Lending:

```math
1\ \text{VIGOR} = 1\ \text{VIGOR}
```

VIGOR is the numeraire. Other supported assets are valued in VIGOR terms, and VIGOR itself does not require an oracle row.

This keeps internal accounting simple and deterministic. It does not prove that the external market price is one dollar.

### 5.3 Balance-sheet support

The protocol supports VIGOR’s credibility through:

- overcollateralized issuance;
- market-sensitive borrowing fees;
- active insurer capital;
- bailout and recapitalization;
- a final reserve;
- a narrow Savings backstop;
- a hard rule that unresolved insolvency must not silently survive an epoch.

These mechanisms are primarily about **solvency and backing**. A solvent system has a stronger foundation for a stable token than an undercapitalized one.

But solvency and market price are not the same thing. A token can be economically backed and still trade below target if liquidity is poor or confidence is weak.

### 5.4 Supply expansion above the target

Suppose VIGOR trades at **\$1.05**.

A market participant may see an opportunity:

1. deposit crypto collateral;
2. borrow newly issued VIGOR;
3. sell that VIGOR for more than one dollar in the market.

If borrowing costs and transaction friction are lower than the premium, this expands supply and adds selling pressure.

```text
VIGOR above $1
    ↓
Borrowing and issuance become more attractive
    ↓
Supply expands
    ↓
Selling pressure may push VIGOR down
```

The trade has limits. Expansion is constrained by:

- collateral availability;
- collateral-ratio requirements;
- borrowing fees;
- confidence in the protocol;
- available market depth.

### 5.5 Supply contraction and demand below the target

Suppose VIGOR trades at **\$0.95**.

A stable borrower with 10,000 VIGOR of debt can buy VIGOR at a discount, repay 10,000 units of nominal debt, and cause that VIGOR to be retired.

That creates both buying pressure and supply contraction.

Other possible sources of VIGOR demand include:

- posting VIGOR as collateral to borrow crypto;
- depositing VIGOR into Savings to earn VIG;
- supplying VIGOR as insurance capital when insurers choose to hold it;
- external use of VIGOR as a low-volatility settlement asset, if that use develops.

None of these creates a guaranteed bid. Their strength depends on real users, attractive economics, and liquid markets.

```text
VIGOR below $1
    ↓
Repayment, collateral use, Savings, and insurance become attractive
    ↓
VIGOR is bought, locked, or retired
    ↓
Demand rises and liquid supply may contract
```

Again, “may” matters. The incentive works only if participants believe the discount is temporary and can trade with acceptable cost and risk.

The mechanism is asymmetric. Above target, new issuance creates a relatively direct expansion trade. Below target, there is no hard redemption floor: convergence depends on outstanding borrowers choosing to repay, users wanting VIGOR for collateral or Savings, and markets remaining liquid enough to execute those trades.

### 5.6 The peg loop in one picture

```text
                    VIGOR trades above target
                              │
                    More attractive to issue
                              │
                        Supply expands
                              │
                        Downward pressure
                              │
                              ▼
                    Target around one dollar
                              ▲
                              │
                 Buying, locking, and repayment
                              │
                    Demand and supply contraction
                              │
                     VIGOR trades below target
```

### 5.7 What VIGOR does not promise

CLOAK Lending does not currently provide:

- a fiat reserve redeemable at one dollar;
- a hard one-dollar conversion window;
- automatic open-market purchases by the lending contract;
- a direct VIGOR/USD price input used to revalue VIGOR inside the protocol;
- certainty that arbitrage remains profitable during stress.

VIGOR should therefore be described as a **USD-referenced, market-supported synthetic low-volatility token**.

That is still a meaningful design. It is simply different from a token backed by redeemable dollars in a bank.

---

## 6. Why External Market Liquidity Matters

The lending contract creates incentives. A market turns those incentives into trades.

For the peg mechanism to work well, VIGOR needs at least one deep, reliable market—commonly a pair such as:

```text
blockchain system token / VIGOR
```

A single pair may provide a route, but its quality depends on the system token’s own liquidity and its connection to broader USD markets.

### 6.1 What the market does

A liquid VIGOR market provides:

- price discovery;
- a place for borrowers to sell newly borrowed VIGOR;
- a place for debtors to buy VIGOR for repayment;
- entry and exit for savers;
- entry and exit for insurers holding VIGOR;
- executable peg arbitrage;
- an external signal of market confidence.

### 6.2 Why “a market exists” is not enough

A market can technically exist and still be economically useless.

What matters is:

- depth near the target price;
- narrow spreads;
- manageable slippage;
- resilient liquidity during volatility;
- reliable access to the system token’s USD markets;
- enough market-maker capital to handle issuance and repayment flows;
- resistance to manipulation.

Imagine that the VIGOR pair has only \$20,000 of usable liquidity. A borrower selling 100,000 newly issued VIGOR could move the price sharply even if the protocol is perfectly solvent.

The reverse is also true. During a depeg, borrowers may want to buy VIGOR to repay, but shallow liquidity can prevent the demand from translating into smooth price convergence.

### 6.3 Four different system states

These should not be confused:

| State | What it means |
|---|---|
| Solvent protocol, liquid VIGOR | Balance sheet is healthy and markets function well |
| Solvent protocol, illiquid VIGOR | Backing may be sound, but users cannot trade efficiently |
| Solvent protocol, depegged VIGOR | Internal accounting is intact, but market confidence or liquidity is weak |
| Insolvent protocol | Losses exceed the available capital waterfall |

A strong stable-token design needs both a credible balance sheet and credible market infrastructure.

### 6.4 The oracle-market relationship

CLOAK Lending consumes oracle prices denominated in VIGOR. This makes the oracle methodology important during a depeg.

A production specification must choose—and publicly document—what “price in VIGOR” means when VIGOR is not worth one dollar. Two broad policies are possible:

1. **Market-relative pricing.** External assets are converted at their actual market exchange rate against VIGOR. If VIGOR falls to \$0.80, a \$60,000 BTC is approximately 75,000 VIGOR. This keeps real relative values aligned, but the protocol’s supposedly low-volatility numeraire now moves through every collateral and debt calculation.
2. **Target-relative pricing.** External assets continue to use their USD price as though one VIGOR were one dollar. This preserves target-based accounting, but can misstate real economic collateralization. Below target it overvalues VIGOR posted against crypto debt; above target it understates the real dollar burden of VIGOR debt.

Neither policy is a harmless implementation detail. The first imports the depeg into all VIGOR-denominated prices. The second can make one debt side look safer than it is in real market value.

The contract validates freshness and data structure, but it cannot determine the economic meaning of the publisher’s number. Oracle methodology, source markets, depeg handling, and emergency behavior therefore need to be treated as protocol law and tested before production deployment.

---

## 7. Savings: Demand, Revenue, and Tail Risk

Savings gives VIGOR a productive use beyond borrowing and repayment.

A user deposits VIGOR into the `savings` bucket. In return, the user receives a proportional share of the fee revenue allocated to Savings.

The important token flow is:

```text
Saver deposits VIGOR
        ↓
Protocol activity produces fees in VIG
        ↓
Configured Savings share is distributed
        ↓
Saver receives VIG in the `vigfees` balance
```

### 7.1 Normal Savings rewards

Let:

- $F_S$ be the amount of VIG allocated to Savings for an epoch;
- $S_i$ be saver $i$’s VIGOR balance;
- $S$ be total VIGOR Savings.

Ignoring integer rounding for the moment:

```math
\text{VIG reward}_i
=
F_S
\frac{S_i}{S}
```

#### Example

Maria deposits 10,000 VIGOR into a Savings pool containing 100,000 VIGOR.

She owns 10% of the pool.

If the epoch allocates 500 VIG to Savings:

```math
500 \times 10\% = 50\ \text{VIG}
```

Maria receives approximately 50 VIG in her `vigfees` balance.

The implementation uses deterministic integer allocation. Floor rounding is used for ordinary shares, and any remaining dust is assigned deterministically rather than lost or distributed by arbitrary table order.

### 7.2 What determines the yield

Savings does not promise a fixed interest rate.

The realized return depends on:

- total borrowing fees collected;
- the configured Savings fee cut;
- the market value of VIG;
- the amount of VIGOR deposited by all savers;
- how long the user remains in Savings;
- whether an extreme backstop event occurs.

A larger Savings pool creates stronger VIGOR demand, but it also divides the same fee allocation among more deposited VIGOR.

### 7.3 Why Savings supports VIGOR demand

Savings creates demand in three ways:

1. users must acquire VIGOR to participate;
2. deposited VIGOR sits inside the protocol rather than on an exchange order book;
3. expected VIG rewards give VIGOR an income-producing use.

That can support the peg when protocol activity is healthy, but the effect is not permanent. Under normal operating conditions, savers can withdraw their VIGOR without the delayed-claim process imposed on active insurers.

### 7.4 Savers can leave—and that matters

Savings is liquid in ordinary operation. A saver can withdraw while the protocol is ready, without waiting through the active-insurer withdrawal delay. Once an epoch is overdue or an update is running, ordinary withdrawals are blocked until processing finishes.

That creates an uncomfortable but important incentive. If stress becomes visible early enough, rational savers may leave before the backstop is needed. The pool can therefore shrink at exactly the time the system would most value it. A sustainable Savings pool needs enough fee income and confidence to compensate users for that tail risk; calling it a backstop does not guarantee the capital will still be there.

There is also a market-liquidity trade-off. VIGOR held in Savings is not offered on an exchange order book, so a large Savings pool can reduce sell pressure while making dedicated market-maker liquidity more important.

### 7.5 Savings is not risk-free

Savings is also the final narrow stable-side backstop.

It is used only after:

1. a distressed borrower has been processed;
2. normal insurer capital has been exhausted;
3. the final reserve has absorbed losses and attempted self-recapitalization;
4. the final reserve still remains insolvent on the stable-debt side.

At that point, Savings can cancel at most the smaller of:

- the final reserve’s remaining VIGOR debt;
- total available VIGOR Savings.

Savers lose a proportional amount of VIGOR Savings, that VIGOR is retired against reserve debt, and savers receive a proportional share of the corresponding reserve collateral.

#### Example

Assume:

- total Savings: 100,000 VIGOR;
- Maria’s Savings: 10,000 VIGOR;
- reserve debt requiring backstop: 20,000 VIGOR;
- matching reserve collateral being socialized: \$22,000 worth of crypto.

Maria owns 10% of Savings, so she would absorb approximately:

```math
20{,}000 \times 10\% = 2{,}000\ \text{VIGOR}
```

and receive approximately 10% of the collateral slice.

This may be profitable or unprofitable depending on market prices, execution, and the severity of the event.

A better way to think about it is:

> **Savings is a VIGOR deposit facility that earns VIG fee revenue while serving as a junior, last-resort stable-side backstop.**

It is closer to a yield-bearing stability pool than a bank savings account.

---

## 8. Active Insurers and Borrowable Liquidity

Insurers are the economic center of CLOAK Lending.

They are paid because their capital does real work.

### 8.1 Entering the active insurer set

A user first deposits assets into `insurance`, then enters the active insurer set.

Merely having an insurance balance is not enough. The `active_insurer` status determines whether the capital:

- counts toward live crypto borrowing capacity;
- participates in ordinary insurer fee distribution;
- participates in normal bailout allocation;
- is subject to delayed withdrawal.

The final reserve is intentionally excluded from this set.

### 8.2 Insurers provide crypto liquidity

For each supported token, the protocol tracks:

- total insurance;
- active insurance;
- live lent amount;
- currently available amount;
- repriced lendable and utilization measures.

Only active normal insurer inventory counts toward live crypto borrowing.

If active insurers collectively hold 10,000 EOS and 6,000 EOS are already lent:

```math
\text{available EOS} = 10{,}000 - 6{,}000 = 4{,}000
```

A new borrower cannot draw more than that live amount.

### 8.3 Insurers earn for risk and liquidity

Insurer rewards are not based only on raw deposit size.

Stable-side fees are distributed using `pcts`, which measures an insurer’s contribution to reducing stable-side system risk.

Crypto-side insurer fees are split between:

- `l_pcts`, the crypto-side risk contribution;
- `l_rmliq`, the insurer’s liquidity-risk contribution.

This reflects two different services:

1. capital that makes the system safer;
2. inventory that supports borrowed-token liquidity.

If one of the normal risk or liquidity denominators collapses to zero while active insurer capital still exists, the implementation falls back to deterministic value-weighted distribution across active insurers. That is a degenerate safety rule, not the normal reward model.

### 8.4 Withdrawal is deliberately delayed

An active insurer can request a withdrawal, but the capital remains inside `insurance` until the delay expires and the claim is executed. This is deliberately stricter than Savings, where ordinary withdrawals have no comparable delay.

During the pending period, the capital:

- remains active;
- remains fee-earning;
- remains borrowable where applicable;
- remains exposed to bailouts and recapitalization.

This prevents an insurer from escaping the exact risk for which it has been paid.

Any bailout cancels pending insurer withdrawal requests. The insurer can request withdrawal again later based on the post-bailout balance.

### 8.5 The debtor can also be an insurer

An account is not excluded from the insurer pool merely because it is the distressed borrower.

If Alice is both:

- a borrower with an insolvent position; and
- an active insurer with eligible capital,

her insurance participates according to the same `pcts` or `l_pcts` weighting as other active insurers.

Economically, that is the clean result. Alice’s insurer capital was earning fees and contributing to the risk pool before the failure. It should not become protected simply because another bucket in Alice’s account is now distressed.

### 8.6 What happens to an insurer during bailout

An absorbing insurer can receive:

- a share of the distressed debt;
- a matching share of the distressed collateral.

The protocol then tries to restore the inherited position.

Depending on the debt side, it may:

- cancel debt using matching insurance inventory;
- move VIGOR insurance into stable collateral;
- move crypto insurance into collateral;
- borrow the minimum required VIGOR against moved crypto collateral;
- recheck the insurer after the bailout.

If absorbing the loss makes the insurer insolvent, that account can itself enter the ordinary insolvency queue.

Losses can therefore propagate, but they do not disappear.

---

## 9. Dynamic Risk Pricing

A simple lending pool often sets rates mostly from utilization.

CLOAK Lending tries to price something richer: the expected risk contributed by a specific collateral portfolio under current system conditions.

The central intuition is:

> A volatile, concentrated, poorly collateralized position should pay more than a diversified, well-collateralized position—and rates should also respond when the insurer pool itself becomes stressed.

### 9.1 Asset valuation

For a non-VIGOR asset $j$:

```math
V_j = Q_j P_j
```

where:

- $Q_j$ is the token quantity;
- $P_j$ is its oracle price in VIGOR;
- $V_j$ is its VIGOR-denominated value.

For a bucket of assets:

```math
V_{\text{bucket}}
=
\sum_j Q_j P_j
```

VIGOR is already the numeraire, so its value is its amount.

### 9.2 Portfolio volatility

For a portfolio with value weights $w_i$, volatilities $\sigma_i$, and correlations $\rho_{ij}$, the protocol follows the familiar portfolio-variance structure:

```math
\sigma_p^2
=
\sum_i w_i^2\sigma_i^2
+
2\sum_{i\lt j}w_iw_j\sigma_i\sigma_j\rho_{ij}
```

and:

```math
\sigma_p = \sqrt{\sigma_p^2}
```

This is why two accounts holding the same total collateral value may receive different pricing.

#### Example

Alice holds only one volatile token.

Daniel holds a portfolio split across two assets with imperfect correlation.

Even if both portfolios are worth 20,000 VIGOR today, Daniel’s portfolio may have lower modeled variance. Lower modeled tail risk can translate into a lower borrowing price.

Diversification is not assumed blindly. If two assets become highly correlated, the benefit falls.

A missing correlation entry currently defaults to zero. That keeps calculation deterministic, but it can overstate diversification when the data set is incomplete. Oracle operations must therefore treat correlation coverage—not only freshness—as a risk control.

### 9.3 Tail stress

The protocol includes normal-distribution helpers and a conditional-value-at-risk style tail multiplier.

For tail confidence $\alpha$:

```math
z_\alpha = \Phi^{-1}(\alpha)
```

```math
m_{\text{tail}}
=
\frac{\phi(z_\alpha)}{1-\alpha}
```

where:

- $\Phi^{-1}$ is the inverse standard normal CDF;
- $\phi$ is the standard normal density.

The multiplier is applied to portfolio volatility to estimate a stressed downside move for collateral and a stressed upside move for borrowed crypto.

Crypto returns are not perfectly normal. The model uses a normal-distribution framework because it provides a consistent, deterministic way to turn volatility and correlation inputs into a stress measure—not because it captures every jump or regime change.

### 9.4 Stable-side risk

For a VIGOR borrower, the dangerous direction is down: crypto collateral loses value.

The contract constructs a stressed collateral loss, scales it by global stable-side conditions, and estimates a shortfall payoff:

```math
\text{payoff}_{stable}
=
\max(
D_s - C_{\text{stressed}},
0
)
```

where:

- $D_s$ is VIGOR debt;
- $C_{\text{stressed}}$ is stressed crypto collateral value.

The user’s stable risk price is derived from:

- expected stressed shortfall;
- modeled probability of the shortfall;
- outstanding VIGOR debt;
- minimum and maximum configured rate bounds.

At a high level:

```math
r_s
\approx
\frac{
\text{expected stable-side loss}
}{
\text{VIGOR debt}
}
```

clamped between the configured floor and cap.

The resulting rate is stored as `tesprice`.

### 9.5 Crypto-side risk

For a crypto borrower, the dangerous direction is up: the borrowed asset rises relative to fixed VIGOR collateral.

The stressed payoff is conceptually:

```math
\text{payoff}_{crypto}
=
\max(
D_{c,\text{stressed}} - C_s,
0
)
```

where:

- $D_{c,\text{stressed}}$ is stressed crypto debt value;
- $C_s$ is VIGOR collateral.

Crypto pricing also includes liquidity scarcity.

For borrower $u$, a simplified representation of the liquidity component is:

```math
L_u
=
\sum_j
\left(
\frac{V_{u,j}}{V_{u,\text{crypto debt}}}
\right)
U_j
```

where $U_j$ is the token’s utilization measure.

The more the borrower depends on scarce, heavily utilized tokens, the larger the liquidity-risk adjustment.

The resulting crypto-side rate is stored as `l_tesprice`.

### 9.6 Global solvency scales

User risk is not priced in isolation.

The protocol also asks:

- how much insurer capital exists;
- how that capital behaves under stress;
- how much distressed borrower exposure exists;
- how far the system is from configured solvency targets.

Those conditions produce global scales used in user pricing.

When insurer solvency deteriorates, the pricing stress scale can rise. Borrowers then pay more for creating additional risk in a weaker system.

That is closer to traditional risk-based credit pricing than a utilization curve alone: the rate responds to both the loan and the balance sheet standing behind it.

### 9.7 Risk contribution weights

The protocol measures how much worse stressed system insurance would look if a particular insurer’s capital were removed.

For insurer $i$, the stable-side contribution is conceptually:

```math
c_i
=
\max(
S - S_{-i},
0
)
```

where:

- $S$ is the stressed value of global insurance;
- $S_{-i}$ is stressed insurance after excluding insurer $i$’s effective contribution.

Normalized across the system:

```math
pcts_i
=
\frac{c_i}{\sum_k c_k}
```

The crypto-side equivalent produces `l_pcts`.

These weights serve two purposes:

- allocate insurer fee rewards;
- allocate bailout exposure.

That symmetry matters. The same risk contribution that earns an insurer more can also make it absorb more when the modeled risk materializes.

---

## 10. Fees and Incentives

CLOAK Lending charges fees by epoch.

The economic fee is calculated in VIGOR-value terms. Payment is made in VIG, the CLOAK token.

### 10.1 Fee accrual

For debt value $D$, annualized risk price $r$, and epoch year fraction $T$:

```math
F
=
D\left((1+r)^T - 1\right)
```

The implementation evaluates the power deterministically as:

```math
(1+r)^T
=
\exp\left(T\ln(1+r)\right)
```

For the stable side:

```math
F_s
=
D_s\left((1+\texttt{tesprice})^T - 1\right)
```

For the crypto side:

```math
F_c
=
V_c\left((1+\texttt{l\_tesprice})^T - 1\right)
```

The year convention used by the implementation is:

```math
T
=
\frac{\text{epoch seconds}}
{360 \times 24 \times 60 \times 60}
```

### 10.2 Conversion into VIG

Let:

- $F$ be the fee value in VIGOR;
- $P_{VIG}$ be the oracle price of one VIG in VIGOR.

Then the required VIG amount is:

```math
\text{VIG due}
=
\left\lceil
\frac{F}{P_{VIG}}
\right\rceil
```

after normalization to VIG precision.

The protocol rounds in its own favor when converting the economic fee into payable VIG units. This avoids systematic fee leakage through truncation.

### 10.3 Same-epoch repayment

A borrower cannot open and close a loan inside the same epoch for free.

Repayment first charges the full current epoch fee. A configurable repayment-only minimum fee also prevents tiny or zero-cost borrow-repay cycles caused by integer rounding.

Repayment is all-or-nothing:

- the full fee must be payable;
- overpayment of debt is rejected rather than silently retained or refunded;
- only after fees are settled is principal reduced.

### 10.4 VIG lifeline

Borrowers maintain a VIG balance in `vigfees`.

If the balance is insufficient during epoch finalization, the protocol tries a bounded rescue sequence:

1. move VIG already present in insurance into `vigfees`;
2. move free VIG from collateral, but only if doing so does not break collateral requirements;
3. during finalization only, attempt an assisted rescue:
   - move free VIGOR insurance into stable collateral;
   - if necessary, create the minimum additional VIGOR support against crypto collateral;
   - borrow only the VIG needed for the current fee event, provided VIG is whitelisted, lendable, and available.

If the fee still cannot be paid, the shortfall enters a fee-default queue and is revalidated before bailout.

The protocol does not trigger a bailout merely because an earlier scan saw a shortfall. It gives fresh balances and rescue transfers a chance to cure the problem.

### 10.5 Where fees go

At epoch completion, collected VIG fees are allocated in layers:

```text
Collected VIG fees
        │
        ├── final tick reward
        ├── final reserve share
        ├── Savings share
        ├── optional fee-account share
        └── active insurer share
```

Insurer fees are risk-weighted. Savings fees are VIG distributed in proportion to VIGOR Savings balances.

If no normal insurer capital exists, the insurer share is redirected to the final reserve. If active insurers exist but a normal `pcts`, `l_pcts`, or liquidity-contribution denominator is zero, the corresponding insurer allocation falls back to deterministic weighting by current insurance value.

The incentive map is straightforward:

| Participant | Why they are paid |
|---|---|
| Tick caller | Advances and completes protocol processing |
| Saver | Locks VIGOR and accepts narrow tail risk |
| Insurer | Supplies liquidity and absorbs defaults |
| Final reserve | Builds last-resort capital |
| Optional fee account | Receives a configured passive share |

---

## 11. Insolvency, Bailouts, and Recapitalization

Liquidation is where CLOAK Lending differs most clearly from a conventional DeFi money market.

Instead of asking an external liquidator to repay a borrower’s debt in exchange for discounted collateral, CLOAK Lending transfers the failed position into its own risk-capital network.

The protocol calls this a **bailout**.

### 11.1 Insolvency is side-specific

#### Stable side

A stable-debt position is insolvent when:

```math
C_c \times 10{,}000
\lt 
D_s \times B_s
```

where:

- $C_c$ is crypto collateral value;
- $D_s$ is VIGOR debt;
- $B_s$ is `bailout_cr_bps`.

Equivalently:

```math
\frac{C_c}{D_s}
\lt 
\frac{B_s}{10{,}000}
```

#### Crypto side

A crypto-debt position is insolvent when:

```math
C_s \times 10{,}000
\lt 
D_c \times B_c
```

where:

- $C_s$ is VIGOR collateral;
- $D_c$ is crypto debt value;
- $B_c$ is `bailoutup_cr_bps`.

The comparison is strict. A position exactly at the configured ratio is not below it.

### 11.2 Why a bailout can start

There are two distinct triggers:

- **Ordinary insolvency:** a live collateral-ratio test fails on the stable side, the crypto side, or both.
- **Fee default:** after collection attempts and live revalidation, an unresolved stable-fee shortfall selects the stable side and an unresolved crypto-fee shortfall selects the crypto side.

A fee-default bailout is therefore not evidence that the selected side already failed its collateral-ratio test. It is the protocol’s consequence for an uncured fee obligation on that debt side.

### 11.3 Live revalidation

An account can improve its position after an earlier scan by:

- repaying debt;
- adding collateral;
- adding VIG to cover fees.

Before an ordinary bailout starts, the protocol rechecks fresh live state. A resolved or healthy side is not processed based on stale detection.

Fee defaults are also revalidated before bailout.

### 11.4 Only the selected side is resolved

Suppose Alice has:

- VIGOR debt backed by BTC;
- EOS debt backed by VIGOR.

If BTC falls and only the VIGOR-debt side becomes insolvent, the bailout touches that side only.

Alice’s healthy EOS debt and its VIGOR collateral remain intact.

If only the crypto side fails, the stable side remains intact.

If both sides are selected:

1. resolve the stable side;
2. run a repricing barrier;
3. resolve the crypto side;
4. clean up and recheck.

The repricing barrier matters because stable-side loss absorption changes insurer portfolios and therefore can change crypto-side risk weights.

### 11.5 Stable-side bailout

Assume Alice owes 10,000 VIGOR and has distressed crypto collateral worth 11,000 VIGOR.

The protocol builds a set of active normal insurers and assigns each a weight $w_i$. Stable bailout normally uses `pcts`; crypto bailout normally uses `l_pcts`. If the relevant denominator is unusable while active insurance still exists, participant collection falls back to current insurance value rather than arbitrary table order.

For insurer $i$:

```math
D_i
=
D
\frac{w_i}{\sum_k w_k}
```

```math
C_i
=
C
\frac{w_i}{\sum_k w_k}
```

with deterministic integer handling for the final remainder.

Each insurer receives:

- a share of Alice’s VIGOR debt;
- the same proportional share of Alice’s crypto collateral.

Then recapitalization begins.

#### Step 1: matching VIGOR cancellation

If the insurer already holds VIGOR inside `insurance`, that VIGOR is used to cancel inherited stable debt and is retired.

#### Step 2: crypto insurance moves into collateral

If debt remains, the protocol calculates how much collateral the inherited position requires. The required factor includes:

- the configured bailout collateral ratio;
- a volatility buffer based on the insurer’s remaining insurance portfolio.

Enough crypto insurance is moved into `collateral` to support the inherited debt.

The insurer has not received a free gift. It has received both an asset and a liability and used its own risk capital to make the position safe.

### 11.6 Crypto-side bailout

Now assume Bob owes 5,000 EOS and has insufficient VIGOR collateral.

Each active insurer receives:

- a proportional share of the EOS debt using `l_pcts`;
- the same share of Bob’s VIGOR collateral.

Recapitalization proceeds in order.

#### Step 1: matching-token cancellation

If an insurer already holds EOS in `insurance`, matching EOS cancels inherited EOS debt directly.

#### Step 2: VIGOR insurance becomes collateral

The insurer’s VIGOR insurance can move into `stable_collateral`.

#### Step 3: crypto insurance supports new VIGOR collateral

If the inherited position is still undercollateralized, the protocol can:

1. move crypto insurance into collateral;
2. create the minimum VIGOR support required against it;
3. move that VIGOR into stable collateral.

This lets a diversified insurer portfolio recapitalize a crypto-debt position even when it does not hold enough matching borrowed tokens.

### 11.7 Debtor-owned insurance participates

If the debtor is also an active insurer, the debtor’s insurance remains in the participant set.

Consider Alice with:

- an insolvent VIGOR loan;
- 10% of total stable-side insurer weight.

Alice’s insurer bucket absorbs approximately 10% of the bailout like any other insurer. That share then follows the same debt inheritance, matching cancellation, recapitalization, and recheck rules.

Otherwise, insurer capital would become protected at the exact moment its owner defaults elsewhere in the account.

### 11.8 Reserve bailout cut

A configurable portion of bailout collateral can be redirected into final-reserve insurance.

If the gross collateral share for participant $i$ is $C_i$ and the configured cut is $q$:

```math
C_{i,\text{net}}
=
C_i(1-q)
```

```math
C_{i,\text{reserve}}
=
C_i q
```

The cut applies to normal participants, not when the final reserve itself is the absorbing participant.

Over time, bailout activity can therefore add capital to the final backstop.

### 11.9 Touched-participant rechecks

A bailout can make an insurer weaker.

After completion, all absorbing participants are rechecked. Any newly insolvent non-reserve account is appended to the ordinary insolvency queue.

This can create a chain:

```text
Alice fails
    ↓
Bob absorbs loss and becomes insolvent
    ↓
Bob enters the bailout queue
    ↓
Remaining insurers absorb Bob
```

The protocol does not assume that the first allocation ended the risk. It follows the loss until the system is healthy or the capital waterfall is exhausted.

---

## 12. The Default Waterfall

CLOAK Lending resolves losses through a predefined hierarchy.

```text
Distressed borrower position
              ↓
Active normal insurers
              ↓
Final reserve
              ↓
Savings backstop
(stable side only; narrow final case)
              ↓
No unresolved insolvency may survive
```

### 12.1 Layer 1: borrower position

The borrower’s debt and available collateral are the starting point.

The borrower loses control of the affected distressed side. Debt and collateral are transferred into the bailout process.

### 12.2 Layer 2: normal insurer capital

Active insurers absorb first.

They are compensated during ordinary operation precisely because their capital is exposed here.

Allocation is risk-weighted rather than first-come-first-served.

### 12.3 Layer 3: final reserve

The final reserve stays outside normal insurer weights and borrowable active liquidity.

It enters only when normal insurer capital has been exhausted.

The reserve can inherit debt and collateral and recapitalize itself using reserve insurance.

### 12.4 Layer 4: Savings

Savings applies only in the narrow final stable-side case described earlier.

It is not a general bailout pool for every borrower and it does not directly back crypto-side debt.

### 12.5 Final invariant

The protocol does not intentionally carry unresolved insolvency forward as an invisible accounting hole.

After the full waterfall, the final reserve must be solvent. In non-mainnet validation builds, the contract can also perform a full user-table assertion. Mainnet execution avoids that expensive scan, but the staged queue and local reserve checks preserve the economic law:

> **The epoch must not complete with known unresolved insolvency.**

### 12.6 Traditional-finance intuition

The structure resembles a clearinghouse default waterfall more than an ordinary peer-to-peer loan:

- the failed position contributes its resources;
- mutualized risk capital absorbs the next layer;
- a dedicated reserve stands behind it;
- a junior final backstop exists for a narrow class of loss.

The analogy is the use of precommitted loss layers, not identical seniority. A conventional clearinghouse may place its own capital ahead of mutualized member resources; CLOAK uses normal insurer capital before the final reserve.

It also resembles mutual insurance:

- members contribute capital;
- members earn premiums;
- members absorb covered losses according to defined rules.

CLOAK Lending is not legally a clearinghouse or insurance company, but those analogies explain the economic structure better than “lending pool” alone.

---

## 13. Oracles, Markets, and Trust Assumptions

A smart contract cannot see the outside world by itself.

CLOAK Lending depends on an external oracle contract, `zoracle`, for the market data used by valuation and risk.

### 13.1 What the oracle provides

For each non-VIGOR asset, the lending contract consumes:

- symbol;
- current price in VIGOR;
- volatility;
- correlations;
- update timestamp.

The oracle contract accepts spot prices and maintains bounded history from which volatility and correlations are derived.

The current sampling model includes:

- 15-minute observations;
- 4-hour observations;
- 1-day observations.

The longer interval carries the largest weight in the public volatility and correlation output. Missing correlation lookup inside the lending contract currently defaults to zero, so complete and correctly paired correlation data is part of safe oracle operation.

### 13.2 Fail-closed validation

The lending contract rejects oracle data when:

- the row is missing;
- the symbol does not match;
- the price is zero or negative;
- the price is not denominated in VIGOR;
- the timestamp is uninitialized;
- the timestamp is too old;
- volatility is invalid;
- correlation values are malformed, duplicated, or out of range.

An oracle outage can therefore stop normal operation. That is inconvenient, but safer than knowingly valuing collateral with invalid data.

### 13.3 Oracle concentration

The v1 oracle uses a single configured publisher.

That makes oracle operation a critical trust and availability dependency.

A bad publisher, compromised key, faulty market feed, or methodological error can affect:

- borrowing capacity;
- collateral ratios;
- risk prices;
- fee conversion;
- insolvency detection;
- bailout recapitalization.

Future oracle decentralization should preserve the interface but reduce reliance on one publisher.

### 13.4 VIGOR as numeraire

Because VIGOR has no oracle row, every other price is expressed relative to it.

This is internally consistent but creates the external-price question discussed earlier.

A production oracle policy must define:

- which markets determine each asset’s VIGOR price;
- how VIGOR/USD deviations are handled;
- whether direct VIGOR pairs or USD cross-rates are preferred;
- how thin or manipulated venues are excluded;
- how outliers and rapid moves are handled;
- how oracle publication continues when the main VIGOR pair is disrupted.

### 13.5 Market makers are infrastructure

Market makers are not contract participants, but the stable-token economy depends on them.

They provide:

- two-sided VIGOR liquidity;
- inventory for arbitrage;
- smoother borrower entry and exit;
- a practical route between VIGOR, the system token, and external dollar markets.

A healthy protocol launch therefore needs more than deployed contracts. It needs an operating plan for:

- oracle publication;
- VIGOR market depth;
- system-token liquidity;
- VIG liquidity for fee payment;
- tick execution;
- reserve capitalization;
- insurer recruitment.

### 13.6 Epoch liveness and recovery

`tick()` is permissionless, but permissionless does not mean automatic. Once an epoch is due, ordinary actions are blocked except for narrow rescue transfers, and completing one epoch may require several tick calls across persisted stages.

In practice, a deployment needs reliable automation or operators willing to keep calling `tick()`, even when fees are low or markets are stressed. The current v1 failure model is intentionally harsh: if epoch processing wedges badly, the protocol can halt, and there is no general administrative recovery path.

### 13.7 Administrative trust

The contract also has configuration and operational control points.

Material authorities include the ability to:

- configure tokens and risk parameters;
- set the oracle account;
- set the final reserve and optional fee account;
- enable or disable deposits and borrows;
- control the VIGOR token permissions required for issuance and retirement;
- update contract code if the deployment remains upgradeable.

A public deployment should document who controls these permissions, whether they are multisignature or timelocked, and how emergency changes are governed.

The mathematics can be decentralized while administration remains concentrated. Users need to know both.

---

## 14. Privacy and Deterministic Execution

CLOAK Lending combines private financial access with deterministic public-chain execution.

### 14.1 CLOAK Shielded Protocol integration

The contract exposes ordinary account actions and shielded equivalents.

Private actions use CLOAK’s `zidentity` model and `REQUIRE_ZAUTH` rather than ordinary account authorization.

This allows shielded users to:

- open and close lending accounts;
- deposit and withdraw;
- borrow VIGOR;
- borrow crypto;
- enter the insurer set;
- manage insurer withdrawals;
- use Savings;
- repay through the integrated token flow.

Privacy is part of the protocol architecture, not a separate web interface layered over public lending actions.

### 14.2 Why private credit matters

Public lending positions can reveal:

- asset holdings;
- leverage;
- liquidity needs;
- liquidation risk;
- trading strategy;
- links between wallets and financial behavior.

For individuals, that is a privacy concern. For funds, market makers, businesses, and professional traders, it can also be a competitive and security concern.

CLOAK Lending allows the economic rules to remain verifiable while user identity and activity can operate through the shielded protocol.

### 14.3 Deterministic fixed-point arithmetic

Financial settlement cannot depend on platform-specific floating-point behavior.

CLOAK Lending uses `fp128`, a signed 128-bit decimal fixed-point type.

Conceptually, a value is stored as:

```math
x = a \times 10^{-p}
```

where:

- $a$ is a signed integer amount;
- $p$ is an explicit decimal precision.

For example:

```math
123.45
=
12345 \times 10^{-2}
```

The implementation supports:

- addition and subtraction;
- multiplication and division;
- comparisons;
- normalization;
- square root;
- logarithm;
- exponential;
- power;
- deterministic formatting and conversion.

`fp128` is bounded rather than infinitely precise. Signed 128-bit limits apply, some operations may normalize or reduce precision, and transcendental functions use deterministic approximations. Those are acceptable engineering choices only when ranges, error behavior, and rounding are covered by tests.

### 14.4 Why this matters

The same initial state and action sequence should produce the same:

- balances;
- prices;
- portfolio values;
- fees;
- insurer weights;
- bailout shares;
- rounding results.

`fp128` makes rounding explicit and intermediate precision bounded.

Final token accounting still returns to integer asset units. High-precision math does not eliminate rounding; it makes the policy visible and reproducible.

### 14.5 Protocol-favoring rounding

Different flows use deliberate rounding directions.

Examples include:

- fees converted into VIG with ceiling behavior;
- debt allocation that must not leave unpaid dust;
- collateral movement that must not underfund recapitalization;
- final participant allocation that receives deterministic remainder;
- Savings and fee distributions that assign dust predictably.

Rounding is economic policy. A one-unit difference repeated across millions of operations is not “just implementation detail.”

### 14.6 Epochs and staged processing

Global portfolio risk cannot be safely recomputed inside every user action.

CLOAK Lending therefore processes time-based epochs:

1. scan debtors;
2. reprice before bailouts;
3. process ordinary insolvencies;
4. reprice before fees;
5. collect fees and process fee defaults;
6. perform final repricing;
7. complete the epoch and distribute rewards.

The process is chunked and resumable. A tick does bounded work in one persisted stage rather than attempting the entire system update in one transaction.

This is essential on Antelope/EOSIO WASM, where a mathematically correct monolithic function can still fail because of execution limits or WASM stack behavior.

The implementation law is:

> **Preserve economics. Change execution shape only when necessary.**

---

## 15. How CLOAK Lending Compares

No comparison is exact. CLOAK Lending combines components that other systems usually keep separate.

### 15.1 MakerDAO / Sky

The closest similarity is collateralized stable-token issuance.

In the Maker/Sky lineage, an undercollateralized Vault transfers debt and collateral into the protocol’s liquidation machinery, where collateral is sold through Dutch auctions to raise DAI. The current Sky system also includes USDS and direct liquidity infrastructure such as the LitePSM, which can support stablecoin conversion and peg liquidity.

CLOAK Lending takes a different route:

- it does not use external collateral auctions as its normal loss-resolution mechanism;
- debt and collateral are allocated into an internal insurer network;
- insurers cancel matching debt and recapitalize inherited positions;
- the final reserve and Savings form additional predefined layers;
- CLOAK Lending currently has no equivalent hard one-dollar redemption or PSM-style conversion facility in the lending contract.

The comparison is therefore not simply “VIGOR versus DAI/USDS.” It is auction- and liquidity-module-based resolution versus integrated insurer capital and recapitalization.

### 15.2 Aave and Morpho

Aave and Morpho connect suppliers and borrowers through on-chain lending markets and rely on permissionless external liquidators to resolve unhealthy positions. A liquidator repays debt and receives collateral plus an incentive.

As of July 2026, Aave V4 is live on Ethereum and uses a more adaptive liquidation design than Aave V3, including a target health factor and variable liquidation bonus. Morpho uses market-specific LLTV thresholds and direct liquidator incentives. The mechanics differ, but both still depend on outside actors executing profitable liquidations.

CLOAK’s active insurer is not merely a lender or liquidator:

- it supplies liquidity;
- earns risk-based fees;
- absorbs bailout exposure;
- recapitalizes inherited debt;
- remains exposed during a pending withdrawal.

Aave’s Umbrella is a closer comparison to CLOAK’s loss-absorbing capital because staked aTokens can be burned against corresponding protocol deficits. CLOAK differs by integrating insurer capital directly into ordinary crypto liquidity, borrower-level bailout allocation, risk weights, and recapitalization.

### 15.3 Liquity

Liquity’s Stability Pools remain the closest single DeFi analogy to CLOAK’s loss absorption. In Liquity V2, each collateral market has its own Stability Pool: deposited BOLD is burned against liquidated debt and the corresponding collateral flows to Stability Providers. If the Stability Pool is insufficient, Liquity V2 can use just-in-time liquidation or redistribute debt and collateral across borrowers in that market.

CLOAK shares the idea of risk capital that earns compensation and is consumed against distressed debt, but extends it:

- insurance can contain multiple assets;
- active insurers also provide borrowable crypto;
- separate stable and crypto risk weights are used;
- matching-token insurance can cancel inherited crypto debt;
- remaining insurer assets can recapitalize the position;
- a final reserve sits behind normal insurers;
- Savings is a separate, narrower final layer.

### 15.4 Traditional secured lending

At the borrower level, CLOAK resembles:

- securities-backed lending;
- margin lending;
- repo-style collateral finance.

A borrower posts collateral, collateral is marked to market, and insufficient coverage triggers forced resolution.

The difference is that the rules are encoded in smart contracts rather than enforced through a broker, bank, or bilateral agreement.

### 15.5 Mutual insurance

The active insurer pool resembles a mutual insurance arrangement.

Members contribute capital, receive premiums, and absorb losses.

The analogy is especially strong because risk contribution influences both compensation and bailout allocation.

### 15.6 Credit default swaps

CLOAK has CDS-like risk-transfer economics:

- borrowers create risk;
- insurers are paid to bear it;
- a defined default condition activates loss absorption.

But CLOAK Lending is not a credit default swap.

There is:

- no separate reference entity;
- no bilateral protection contract;
- no fixed CDS notional or maturity;
- no contractual credit-event committee;
- no isolated transferable derivative on an external borrower.

So the fair comparison is:

> **CLOAK Lending contains CDS-like risk transfer, but it is an integrated lending and insurance protocol rather than a CDS market.**

### 15.7 Clearinghouse default waterfalls

The full system resembles a clearinghouse more than any individual DeFi feature because both define loss layers before a default occurs.

CLOAK’s order is:

- distressed borrower position;
- mutualized insurer capital;
- dedicated reserve;
- narrow junior backstop.

That ordering is not the same as every clearinghouse waterfall. The comparison is about precommitted resources and deterministic loss allocation, not legal status or identical seniority. CLOAK Lending is not a regulated central counterparty.

### 15.8 The “best of both worlds” claim

The most honest summary is:

> **CLOAK Lending combines DeFi’s programmable, permissionless, privacy-preserving execution with the risk-capital discipline of insurance and clearing systems.**

What stands out is not that CLOAK invented collateralized lending, stable tokens, insurance, or default waterfalls individually.

It is the integration:

> **Most protocols separate lending, liquidation, insurance, and systemic backstops. CLOAK Lending treats them as one financial system.**

---

## 16. Risks, Limits, and What CLOAK Lending Is Not

No amount of elegant math removes the system’s failure modes.

### 16.1 VIGOR depeg risk

VIGOR’s price is market-supported, not guaranteed.

It can trade away from one dollar because of:

- shallow liquidity;
- loss of confidence;
- weak borrowing demand;
- mass exits;
- market-maker withdrawal;
- oracle ambiguity;
- broader blockchain stress.

Internal solvency does not guarantee market-price stability.

A depeg can also feed back into lending risk. If oracle prices use actual VIGOR cross-rates, the depeg changes every non-VIGOR valuation. If oracle prices instead preserve the one-dollar target, real collateralization can be misstated: below target, VIGOR collateral can be overvalued against crypto debt; above target, the real burden of VIGOR debt can be understated. The oracle policy therefore changes who is protected and who is exposed during a depeg.

### 16.2 Oracle risk

The protocol depends on correct and timely prices, volatility, and correlations.

Risks include:

- publisher compromise;
- stale data;
- bad source markets;
- manipulation;
- implementation errors;
- incomplete correlation coverage, because missing correlations default to zero;
- unclear VIGOR-denomination policy during a depeg.

Fail-closed behavior limits incorrect execution but can halt normal operation.

### 16.3 Model risk

The risk engine is sophisticated, but it remains a model.

Volatility and correlation can change faster than historical estimates. Crypto returns have jumps, fat tails, and regime changes that a normal-distribution framework cannot fully capture.

Parameters such as:

- tail confidence;
- calibration;
- solvency targets;
- collateral thresholds;
- minimum and maximum rates;

can materially change incentives and safety.

### 16.4 Correlated market crashes

Diversification helps only when assets do not all fall together.

In a systemic crypto crash:

- collateral can lose value rapidly;
- borrowed crypto can become scarce;
- insurer portfolios can deteriorate at the same time;
- market liquidity can disappear;
- reserve capital may be stressed immediately after insurer capital.

### 16.5 Insurer-capital risk

Internal bailouts reduce dependence on external liquidators but increase dependence on sufficient insurer capital.

If insurer participation is too small:

- crypto borrowing capacity is limited;
- risk becomes concentrated;
- normal insurers can be exhausted quickly;
- the final reserve is reached more often.

### 16.6 Savings risk

Savings users earn VIG because they accept a contingent loss role.

In a severe stable-side reserve failure, Savings VIGOR can be consumed and replaced by distressed collateral exposure. The collateral may later recover—or continue falling.

Savings also has run risk. Ordinary withdrawals are not delayed like active-insurer withdrawals, so informed savers may exit as conditions worsen. The remaining pool can become smaller and more concentrated just before a backstop event.

### 16.7 VIG fee-token risk

VIG is required for fees.

The protocol therefore depends on:

- VIG market liquidity;
- a reliable VIG oracle;
- borrowers being able to obtain VIG;
- sufficient active insurer inventory if the assisted VIG lifeline needs to borrow it.

A sharp VIG price movement changes the number of VIG units required for a given VIGOR-value fee.

### 16.8 Smart-contract and execution risk

The protocol contains:

- fixed-point transcendental math;
- staged state machines;
- multi-asset accounting;
- oracle integration;
- token issuance and retirement;
- bailout recursion through requeued participants.

Bugs can affect funds even when the economic design is sound.

Real WASM execution must be tested, not only native code. The deterministic mathematical functions also need range and approximation-error tests; determinism means nodes agree on a result, not that the result is automatically economically accurate.

### 16.9 Epoch-liveness and recovery risk

When an epoch becomes due, most ordinary actions stop until staged processing completes. Permissionless ticking helps, but someone still has to submit the required transactions, and a wedged state machine can halt the protocol. The current v1 design has no general administrative recovery path.

### 16.10 Administrative and upgrade risk

Configuration, token permissions, oracle publishing, reserve management, and contract upgrades create control surfaces.

Their safety depends on deployment governance, key management, multisignature policy, timelocks, monitoring, and operational discipline.

### 16.11 What CLOAK Lending is not

CLOAK Lending is not:

- a fiat-backed stablecoin issuer;
- a guaranteed one-dollar redemption facility;
- a bank account;
- risk-free Savings;
- unsecured consumer or corporate credit;
- a literal CDS exchange;
- a system independent of external markets;
- a system independent of oracles;
- a promise that every backstop will always be sufficient.

It is:

- overcollateralized;
- risk-priced;
- insurer-backed;
- market-dependent;
- oracle-dependent;
- privacy-integrated;
- deterministic;
- designed to make losses explicit and process them through known capital.

---

## 17. A Complete Worked Example

One story ties the pieces together better than another diagram.

### 17.1 Alice borrows VIGOR

Alice opens a shielded CLOAK Lending account.

She deposits BTC worth 20,000 VIGOR into `collateral`.

Assume the required borrow-time collateral ratio is 150%.

The maximum VIGOR debt supported by that collateral is:

```math
\frac{20{,}000}{1.5}
=
13{,}333.33\ \text{VIGOR}
```

Alice borrows 10,000 VIGOR.

The protocol:

- records 10,000 VIGOR of `stable_debt`;
- issues 10,000 VIGOR;
- sends the VIGOR through the integrated user flow.

Alice’s initial collateral ratio is:

```math
\frac{20{,}000}{10{,}000}
=
200\%
```

### 17.2 Alice uses the market

Alice sells 4,000 VIGOR for the blockchain’s system token and keeps 6,000 VIGOR for payments and Savings.

Her sale adds VIGOR supply to the market. If VIGOR was trading above one dollar, this is part of the normal expansion incentive.

### 17.3 Bob and Carla provide insurance

Bob supplies EOS and VIGOR as active insurance.

Carla supplies BTC and EOS as active insurance.

Their crypto inventory becomes part of live borrowable liquidity.

They begin earning VIG fee shares based on:

- stable-side risk contribution;
- crypto-side risk contribution;
- crypto liquidity contribution.

### 17.4 Maria uses Savings

Maria buys 10,000 VIGOR and deposits it into Savings.

Suppose total Savings becomes 100,000 VIGOR.

Maria owns 10% of the pool and receives approximately 10% of future Savings fee allocations in VIG.

### 17.5 Alice accrues fees

The risk engine evaluates Alice’s BTC collateral.

Assume her annualized `tesprice` is 8%.

For a 30-day illustration:

```math
T = \frac{30}{360} = 0.08333
```

```math
F
=
10{,}000
\left(
1.08^{0.08333}-1
\right)
\approx
64.3\ \text{VIGOR-value}
```

If one VIG is worth 0.50 VIGOR:

```math
\text{VIG due}
\approx
\frac{64.3}{0.50}
=
128.6\ \text{VIG}
```

The contract rounds upward to the smallest VIG unit at the configured token precision—not necessarily to a whole VIG. The actual contract also charges per configured epoch, not once per month; this is only an intuitive aggregation.

### 17.6 BTC falls

BTC drops and Alice’s collateral is now worth 10,500 VIGOR.

Assume the bailout threshold is 110%.

Required collateral is:

```math
10{,}000 \times 1.10
=
11{,}000\ \text{VIGOR}
```

Alice has only 10,500 VIGOR worth of collateral.

Her stable side is insolvent.

If Alice also has a healthy crypto loan backed by VIGOR, that separate side remains untouched.

### 17.7 Insurers absorb the position

Assume stable-side insurer weights are:

| Insurer | `pcts` share |
|---|---:|
| Bob | 60% |
| Carla | 30% |
| Alice’s own active insurance | 10% |

The bailout allocates:

| Participant | VIGOR debt | Gross collateral value |
|---|---:|---:|
| Bob | 6,000 | 6,300 |
| Carla | 3,000 | 3,150 |
| Alice | 1,000 | 1,050 |

A configured reserve cut may reduce the collateral received by each normal participant and credit the cut into final-reserve insurance.

### 17.8 Matching VIGOR cancels debt

Suppose Bob already holds 1,000 VIGOR in insurance.

That VIGOR cancels 1,000 of Bob’s inherited debt and is retired.

Bob now has:

- 5,000 VIGOR inherited debt;
- his share of Alice’s BTC collateral;
- a recapitalization requirement.

Carla and Alice go through the same matching-cancellation process based on their own insurance portfolios.

### 17.9 Insurers recapitalize

The protocol calculates the required collateral for each inherited position using:

- bailout collateral ratio;
- volatility buffer from the insurer’s insurance portfolio.

If Bob needs more collateral, part of his crypto insurance moves into `collateral`.

The result is a solvent inherited position rather than an unresolved hole.

### 17.10 Recheck

After the bailout:

- Alice’s affected stable side has been removed from her original position;
- Bob, Carla, and Alice’s insurer position are rechecked;
- any participant made insolvent by the absorption enters the queue;
- the final reserve remains unused if normal insurers were sufficient;
- Savings remains untouched.

### 17.11 Economic result

The loss did not disappear.

It was processed through capital that had already been earning fees for bearing it.

The result combines:

- borrower collateral;
- insurer mutualization;
- debt cancellation;
- recapitalization;
- deterministic loss allocation;
- a reserve waiting behind the process.

That is CLOAK Lending’s core design.

---

## Appendix A — Mathematical Reference

This appendix gathers the core formulas in one place.

The implementation uses deterministic `fp128` arithmetic and integer asset amounts. The equations below describe the economic model in real-number notation.

### A.1 Asset valuation

For asset $i$:

```math
V_i = Q_iP_i
```

Portfolio or bucket value:

```math
V = \sum_iQ_iP_i
```

VIGOR is the numeraire:

```math
P_{\text{VIGOR}} = 1
```

inside protocol accounting.

### A.2 Collateral ratios

Stable side:

```math
CR_s
=
\frac{V_{\text{crypto collateral}}}
{D_{\text{VIGOR}}}
```

Crypto side:

```math
CR_c
=
\frac{C_{\text{VIGOR}}}
{V_{\text{crypto debt}}}
```

### A.3 Insolvency

Stable side:

```math
V_{\text{crypto collateral}}
\times 10{,}000
\lt 
D_{\text{VIGOR}}
\times \texttt{bailout\_cr\_bps}
```

Crypto side:

```math
C_{\text{VIGOR}}
\times 10{,}000
\lt 
V_{\text{crypto debt}}
\times \texttt{bailoutup\_cr\_bps}
```

### A.4 Portfolio weights

```math
w_i
=
\frac{V_i}{\sum_jV_j}
```

### A.5 Portfolio variance

```math
\sigma_p^2
=
\sum_i w_i^2\sigma_i^2
+
2\sum_{i\lt j}w_iw_j\sigma_i\sigma_j\rho_{ij}
```

```math
\sigma_p = \sqrt{\sigma_p^2}
```

VIGOR contributes value but zero volatility and covariance in the mixed-bucket risk helpers.

### A.6 Tail multiplier

```math
z_\alpha = \Phi^{-1}(\alpha)
```

```math
m_{\text{tail}}
=
\frac{\phi(z_\alpha)}{1-\alpha}
```

### A.7 Simplified stable stress

Let:

- $\sigma_c$ be collateral portfolio volatility;
- $k_s$ be the global stable stress scale.

A simplified form of the downside stress is:

```math
s_c
=
1-\exp(-m_{\text{tail}}\sigma_c k_s)
```

Stressed collateral:

```math
C_{\text{stressed}}
=
C(1-s_c)
```

Stable payoff:

```math
L_s
=
\max(D_s-C_{\text{stressed}},0)
```

The implementation additionally computes a CDF term from collateralization, volatility, and the configured horizon before converting the payoff into `tesprice`.

### A.8 Simplified crypto stress

Let:

- $\sigma_d$ be crypto debt portfolio volatility;
- $k_c$ include global crypto stress and liquidity scarcity.

A simplified upside stress is:

```math
s_d
=
\exp(m_{\text{tail}}\sigma_d k_c)-1
```

Stressed debt:

```math
D_{\text{stressed}}
=
D_c(1+s_d)
```

Crypto payoff:

```math
L_c
=
\max(D_{\text{stressed}}-C_s,0)
```

The implementation again applies a modeled probability term and clamps the resulting `l_tesprice`.

### A.9 Liquidity scarcity

For crypto borrower $u$:

```math
\ell_u
=
\sum_j
\frac{V_{u,j}}{V_{u,\text{crypto debt}}}
U_j
```

where $U_j$ is token utilization.

The implementation uses a nonlinear liquidity adjustment based on this weighted value.

### A.10 Risk contribution weights

Stable-side contribution:

```math
c_i
=
\max(S-S_{-i},0)
```

```math
pcts_i
=
\frac{c_i}{\sum_kc_k}
```

Crypto-side contribution:

```math
c_i^{(c)}
=
\max(S_c-S_{c,-i},0)
```

```math
l\_pcts_i
=
\frac{c_i^{(c)}}{\sum_kc_k^{(c)}}
```

If the relevant normalized denominator is zero while active insurance exists, current bailout and fee-distribution paths use deterministic current-insurance-value weighting as a fallback.

### A.11 Fee accrual

```math
F
=
D\left((1+r)^T-1\right)
```

```math
T
=
\frac{\text{epoch seconds}}
{360\times24\times60\times60}
```

Stable:

```math
F_s
=
D_s\left((1+\texttt{tesprice})^T-1\right)
```

Crypto:

```math
F_c
=
V_c\left((1+\texttt{l\_tesprice})^T-1\right)
```

### A.12 VIG conversion

```math
\text{VIG due}
=
\operatorname{ceil}_{\text{VIG precision}}
\left(
\frac{F_{\text{VIGOR-value}}}
{P_{\text{VIG in VIGOR}}}
\right)
```

### A.13 Savings rewards

```math
R_i
=
F_S
\frac{S_i}{\sum_kS_k}
```

where:

- $F_S$ is the Savings allocation in VIG;
- $S_i$ is saver $i$’s VIGOR Savings balance.

### A.14 Available crypto liquidity

```math
A_j
=
\max(I_j-L_j,0)
```

where:

- $I_j$ is active insurer inventory;
- $L_j$ is live lent amount.

### A.15 Bailout allocation

For participant weight $w_i$:

```math
D_i
=
D
\frac{w_i}{\sum_kw_k}
```

```math
C_i
=
C
\frac{w_i}{\sum_kw_k}
```

Integer remainders are assigned deterministically so the full debt and collateral are accounted for.

### A.16 Reserve collateral cut

For cut $q$:

```math
C_{i,\text{net}}
=
C_i(1-q)
```

```math
C_{\text{reserve}}
=
\sum_iC_iq
```

### A.17 Recapitalization factor

A simplified stable-side recap factor is:

```math
R_s
=
CR_{\text{bailout}}
+
2\sigma_{\text{insurance,monthly}}
```

Required collateral value:

```math
C_{\text{required}}
=
D_{\text{remaining}}R_s
```

The crypto-side recap follows the corresponding VIGOR-collateral requirement with its own threshold and volatility buffer.

---

## Appendix B — Terminology and Code Mapping

### B.1 Account buckets

| Document term | Contract field | Meaning |
|---|---|---|
| Crypto collateral | `collateral` | Crypto backing VIGOR debt |
| Borrowed crypto | `crypto_debt` | Outstanding non-VIGOR debt |
| VIGOR debt | `stable_debt` | Issued VIGOR owed by borrower |
| VIGOR collateral | `stable_collateral` | VIGOR backing crypto debt |
| Savings | `savings` | VIGOR Savings balance |
| Fee balance | `vigfees` | VIG available for fees and rewards |
| Insurance capital | `insurance` | Multi-asset insurer portfolio |
| Active insurer status | `active_insurer` | Membership in normal insurer set |

### B.2 Risk fields

| Document term | Contract field | Meaning |
|---|---|---|
| Stable-side risk price | `tesprice` | User VIGOR borrowing rate |
| Crypto-side risk price | `l_tesprice` | User crypto borrowing rate |
| Stable insurer weight | `pcts` | Stable risk contribution |
| Crypto insurer weight | `l_pcts` | Crypto risk contribution |
| Crypto liquidity contribution | `l_rmliq` | Liquidity-risk reward basis |
| Stable collateral value | `valueofcol` | Crypto collateral value |
| Crypto debt value | `l_valueofcol` | Borrowed crypto value |
| Insurance value | `valueofins` | User insurance portfolio value |
| Stable collateral volatility | `volcol` | Volatility of crypto collateral |
| Crypto debt volatility | `l_volcol` | Volatility of crypto debt |
| Stable system scale | `scale` | Stable pricing stress multiplier |
| Crypto system scale | `l_scale` | Crypto pricing stress multiplier |
| Stable solvency | `solvency` | Stable-side system condition |
| Crypto solvency | `l_solvency` | Crypto-side system condition |

### B.3 Core tables and state

| Concept | Contract object |
|---|---|
| User ledger | `users` |
| Configuration | `config` |
| Global epoch state | `global` |
| Supported tokens | `whitelist` |
| Live token liquidity | `tokenliq` |
| Repricing progress | `reprice` |
| Bailout progress | `bailout` |
| Fee progress | `fee` |
| Insurer withdrawals | `pending` |
| Protocol event history | `eventlog` / archive state |

### B.4 Two debt modes

| Mode | Borrow action | Collateral | Debt |
|---|---|---|---|
| Stable borrow | `borrow` / `borrowp` | `collateral` | `stable_debt` |
| Crypto borrow | `borrowcrypto` / `borrowcryptp` | `stable_collateral` | `crypto_debt` |

---

## Appendix C — Actions and Operational Dependencies

### C.1 Main user actions

| Action | Purpose |
|---|---|
| `openacct` / `openacctp` | Open native or shielded lending account |
| `closeacct` / `closeacctp` | Close eligible account |
| `borrow` / `borrowp` | Borrow VIGOR |
| `borrowcrypto` / `borrowcryptp` | Borrow supported crypto |
| `withdraw` / `withdrawp` | Withdraw from an eligible bucket |
| `enterins` / `enterinsp` | Enter active insurer set |
| `claimins` / `claiminsp` | Claim matured insurer withdrawals |
| `cancelins` / `cancelinsp` | Cancel pending insurer withdrawal |
| `tick` | Advance epoch processing |
| `kick` / `kickauth` | Clean up a small inactive non-debtor account |

Repayment is transfer-driven rather than a separate action:

| Transfer memo | Purpose |
|---|---|
| `repay` | Repay VIGOR debt |
| `repaycrypto` | Repay crypto debt |
| `collateral` | Deposit crypto collateral |
| `stablecol` | Deposit VIGOR collateral |
| `insurance` | Deposit insurer capital |
| `savings` | Deposit VIGOR Savings |
| `vigfees` | Deposit VIG fee balance |

### C.2 Important configuration groups

#### Tokens and accounts

- VIG token and contract;
- VIGOR token and contract;
- oracle account;
- final reserve account;
- optional fee account.

#### Operation

- epoch duration;
- deposits enabled;
- borrows enabled;
- maximum oracle age;
- insurer withdrawal delay;
- maximum active insurers.

#### Collateral and bailout

- stable-side bailout ratio;
- crypto-side bailout ratio;
- reserve bailout collateral cut.

#### Fees

- final tick reward;
- reserve fee cut;
- Savings fee cut;
- optional fee-account cut;
- minimum repayment fee;
- minimum event-log value.

#### Risk engine

- minimum and maximum borrowing price;
- tail probability;
- pricing horizon;
- calibration;
- stable and crypto solvency targets;
- minimum and maximum stress scales.

### C.3 Required external infrastructure

A production deployment needs:

1. **VIGOR token permissions**
   The lending contract must be able to issue and retire VIGOR according to debt changes.

2. **VIG token liquidity**
   Borrowers need a practical way to obtain the CLOAK fee token.

3. **VIGOR market liquidity**
   Borrowers, repayers, savers, insurers, and arbitrageurs need deep trading routes.

4. **Oracle operations**
   Spot prices must be published reliably and transformed into bounded volatility and correlation history.

5. **Tick operations and recovery planning**
   Permissionless callers or automation must advance overdue epochs. Operators also need monitoring and a documented response for stalled processing, because v1 has no general administrative recovery path.

6. **Insurer capital**
   Crypto borrowing and normal bailout absorption depend on active insurer participation.

7. **Final reserve capitalization**
   The reserve must hold meaningful capital before the system can credibly operate under stress.

8. **Monitoring**
   Operators should track:
   - oracle freshness;
   - VIGOR market price and depth;
   - VIG liquidity;
   - insurer concentration;
   - token utilization;
   - reserve health;
   - Savings exposure;
   - overdue epochs;
   - bailout and fee-default events.

9. **Governance and key management**
   Configuration, code, oracle, reserve, and token permissions should use transparent and resilient control policies.

---

## References

### CLOAK Lending source material

- `cloaklending.hpp` — contract declarations, tables, configuration, actions, and staged state used for this guide.
- `cloaklending.cpp` — economic logic, risk engine, token flows, fee distribution, bailout, reserve, and Savings implementation used for this guide.
- CLOAK Lending project brief and the accepted July 2026 development-thread updates — protocol-law and implementation-state context, including the final side-specific bailout and debtor-insurance participation behavior.
- `fp128.md` — deterministic decimal fixed-point arithmetic used by the lending protocol.

A public release should replace this development-state description with a repository URL, immutable commit hash, release tag, deployed contract accounts, and configuration snapshot.

### External comparison references

- [Sky Protocol — Collateral Liquidation](https://developers.skyeco.com/protocol/vaults/collateral-liquidation/)
- [Sky Protocol — LitePSM](https://developers.skyeco.com/protocol/liquidity/litepsm/)
- [Aave — V4 is Live on Ethereum](https://aave.com/blog/aave-v4-live-ethereum)
- [Aave — V4 Liquidation Engine](https://aave.com/blog/aave-v4-liquidations)
- [Aave — Umbrella](https://aave.com/help/umbrella/umbrella)
- [Liquity V2 — Borrowing and Liquidations](https://docs.liquity.org/v2-faq/borrowing-and-liquidations)
- [Morpho — Liquidation](https://docs.morpho.org/learn/concepts/liquidation/)
- [CME Clearing — Financial Safeguards](https://www.cmegroup.com/solutions/risk-management/financial-safeguards.html)

