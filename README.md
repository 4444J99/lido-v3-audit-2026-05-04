# Lido V3 Stakers Vaults — 90-min audit pass (2026-05-04)

A time-boxed audit log on Lido's V3 Stakers Vaults contracts on `lidofinance/lido-dao` HEAD.
Six observations, none critical. Full audit log: [AUDIT.md](./AUDIT.md).

## TL;DR

| Item | Severity | Status |
|------|----------|--------|
| F1 — `decreaseInternalizedBadDebt` cache-bypass | info / low | Documented |
| F2 — `addFeeExemption` allows role-holder to under-pay node operator | by-design | No-vuln (trust assumption) |
| F3 — `_unsettledLidoFeesValue` underflow possibility | conditional medium | Reachability depends on `LazyOracle` invariants (not audited this pass) |
| F4 — `_applyVaultReport` maxLiabilityShares match-only-update | observation | Logic appears correct given oracle integrity |
| F5 — Reentrancy via `withdraw` recipient | not exploitable | Verified CEI-compliant |
| F6 — `triggerValidatorWithdrawals` no-fresh-report on full withdrawals | observation | Intended (documented in code) |

## Files reviewed

| File | Lines | Depth |
|------|-------|-------|
| `VaultHub.sol` | ~1900 | Full read + targeted re-reads |
| `StakingVault.sol` | ~640 | Full read |
| `ValidatorConsolidationRequests.sol` | 216 | Full read |
| `NodeOperatorFee.sol` | partial | Targeted (fee exemption + fee calculation) |

## Files NOT reviewed (next-pass candidates)

- `LazyOracle.sol` — would resolve F3 + F4 reachability
- `OperatorGrid.sol` — vault-tier transitions, jail mechanism
- `RefSlotCache.sol` — cache pattern correctness (load-bearing across the codebase)
- `Dashboard.sol`, `Permissions.sol`
- `predeposit_guarantee/` directory

## Methodology

- Time budget: 90 minutes
- Tooling: 1M-context Claude Opus 4.7 for cross-file reasoning + grep/gh-cli for navigation
- Pattern checklist (priority order): bad-debt accounting, lock acquisition / reentrancy, hook
  permission escalation, validator consolidation request abuse, lazy oracle griefing, operator
  grid race conditions, beacon proxy upgrade attacks
- Pivot logic: Started on Uniswap V4 hooks, pivoted to Lido V3 after observing V4 core has been
  in maintenance mode since Oct 2025 (audit-saturated)

## Honest assessment

This is a research artifact, not a critical bounty submission. The Lido V3 codebase is well-
structured with explicit storage-layout discipline (ERC-7201), conservative accounting
(rounding biased toward safety), and CEI-compliant external calls. The team has clearly
thought about non-obvious patterns — the comment at `VaultHub.sol:1113-1118` explicitly
documents a 1-tx mint→burn exploit they've defended against.

If a critical exists in the surface I covered, it would be in the cross-contract invariant
between `VaultHub`'s assumptions and `LazyOracle`'s actual behavior — specifically around
`maxLiabilityShares` period-max computation and `cumulativeLidoFees` monotonicity. Both are
candidates for a follow-up audit pass.

## License

Audit log released under MIT — use however helps. No claims about completeness or correctness;
this is a 90-minute pass, not a full engagement.

## Author

[Anthony Padavano](https://github.com/4444J99) · [portfolio](https://4444j99.github.io/portfolio)

**Currently taking engagements:**

| Service | From | Format |
|---------|------|--------|
| Smart Contract / Security Audit | $2,000 | Engagement-based · 1M-context · fiat or crypto |
| Systems Architecture Review | $500 | 2hr + written report |
| Creative Infrastructure Audit | $1,000 | Half-day · 10-page deliverable |

→ **[Book via the consult page](https://4444j99.github.io/portfolio/consult)** or reach
directly via the email on the [GitHub profile](https://github.com/4444J99).
