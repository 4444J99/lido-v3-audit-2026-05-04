# Bug Bounty Audit — Lido V3 Stakers Vaults

**Date:** 2026-05-04
**Auditor:** Claude Opus 4.7 (1M context)
**Bounty program:** https://immunefi.com/bounty/lido/ — $2M ceiling for Critical
**Target repo:** github.com/lidofinance/lido-dao @ HEAD (default branch)
**Snapshot:** `/Users/4jp/Workspace/income-2026-05-04/lido-dao/`

---

## Why this target (vs Uniswap V4)

| Dimension                    | Uniswap V4 core                  | Lido V3 Vaults                         |
|------------------------------|----------------------------------|----------------------------------------|
| Last meaningful change       | Oct 2025 (CI/build only since)   | Multiple commits in last 5 days        |
| Audit saturation             | TOB, Spearbit, Certora, OZ — high | Less coverage (V3 is newer)            |
| Bounty ceiling               | $15.5M                           | $2M                                    |
| Novel primitives             | Hooks (audit-saturated since 2024) | EIP-7251 consolidation (post-Pectra, NEW) |
| New attack surface (LoC, Solidity, last 30 days) | ~0    | ~150K total in `0.8.25/vaults/`       |
| Fresh test failures requiring contract-aware fixes | None observed | Yes (PR #1793 — bad-debt math edge cases) |

**Decision:** Lido V3 Vaults > Uniswap V4 for EV. Probability of finding × payoff favors freshness over ceiling here.

---

## In-scope target files

```
contracts/0.8.25/vaults/
├── LazyOracle.sol                     32KB  — oracle reporting + share-rate accounting
├── OperatorGrid.sol                   38KB  — operator allocation + validator grid
├── PinnedBeaconProxy.sol              1.5KB
├── StakingVault.sol                   33KB  — per-vault contract (custodian of beacon validators)
├── ValidatorConsolidationRequests.sol 10KB  — EIP-7251 consolidation requests
├── VaultFactory.sol                   8KB   — deploys new vaults
├── VaultHub.sol                       79KB  — central hub: vault registry, share accounting, withdrawals
├── dashboard/                         (sub-contracts)
├── interfaces/
├── lib/
└── predeposit_guarantee/
```

Plus contracts/0.8.25/ValidatorExitDelayVerifier.sol (18KB) — exit delay validation, consolidation-adjacent.

---

## Surface map

(To be filled during reading.)

---

## Vulnerability hunt — pattern checklist

Going in the order most likely to be productive given the contracts:

### 1. Bad-debt accounting math (HIGH PRIORITY — flagged by recent test fix)

PR #1793 (merged 2026-04-28) fixed the integration test `withdrawals-bad-debt.integration.ts`. The PR body says:
> "Forked protocol state (non-empty Burner, non-zero EL/WV vault balances) leaks into oracle-report math and breaks assertions written for scratch mode"

The fix updated the test to handle real mainnet state. The question for the auditor: **does the contract math produce correct outcomes on real mainnet state, or is the test simply matched to a buggy contract?**

Specific things to check:
- `bad-debt` calculation in vault withdrawal flow — what's the share-rate at the moment of finalization?
- Does `excludeVaultsBalances: true` reflect a contract bug or a test convenience?
- The PR uses `batchShareRate from the WithdrawalsFinalized event for the prefinalize branch` — is the share rate at prefinalize different from finalize, and if so, can a withdrawer be advantaged/disadvantaged?

### 2. Lock acquisition / reentrancy

Vault withdrawal involves: vault → hub → withdrawal queue → external calls. Look for:
- Reentrancy via vault-controlled `withdraw` callbacks
- Missing nonReentrant on critical paths
- State changes after external calls (CEI violations)

### 3. Hook permission escalation in vault dashboard

The `dashboard/` subdirectory likely contains role-based access for vault operators. Check:
- Can a sub-role escalate? (e.g., a `DEPOSITOR` who can call `WITHDRAW`)
- Default admin role boundaries

### 4. Validator consolidation request abuse

`ValidatorConsolidationRequests.sol` handles EIP-7251 consolidation. Check:
- Can an attacker submit a consolidation request for someone else's validator?
- Are pubkey ownership checks correct?
- What happens if consolidation request fails on-chain after fees were paid?

### 5. Lazy oracle griefing

`LazyOracle.sol` — oracle reports lazily. Check:
- Can a caller force-trigger an oracle report at an unfavorable time?
- Stale-report exploitation: does the contract use a stale report when fresh data exists?

### 6. Operator grid race condition

`OperatorGrid.sol` — operator allocation. Check:
- Can two operators occupy the same grid slot?
- Validator-to-operator mapping: can it be corrupted?

### 7. PinnedBeaconProxy — upgrade attack

Beacon proxies have specific upgrade attack vectors. Check:
- Storage layout collision between implementation upgrades
- Initialization frontrunning
- Pinned beacon vs upgradeable beacon — is the "pinned" guarantee actually enforced?

---

## Findings (filled during audit)

### Observation 1 — `decreaseInternalizedBadDebt` bypasses cache (likely intentional, but worth flagging)

`VaultHub.sol:684-689`:
```solidity
function decreaseInternalizedBadDebt(uint256 _amountOfShares) external {
    _requireSender(LIDO_LOCATOR.accounting());
    // don't cache previous value, we don't need it for sure
    _storage().badDebtToInternalize.value -= uint104(_amountOfShares);
}
```

- **Cast truncation:** `uint104(_amountOfShares)` truncates `_amountOfShares > type(uint104).max` (~2.03e31). If accounting somehow passes a large value, the cast wraps. Since accounting is trusted, low priority — but a defense-in-depth `SafeCast.toUint104` would catch a contract-level bug.
- **Cache divergence:** `internalizeBadDebt` uses `withValueIncrease` (cache-aware). This decrease bypasses cache. After this, `getValueForLastRefSlot` returns potentially stale data. The semantics may be intentional ("snapshot at last refSlot is preserved"), but worth confirming via RefSlotCache library reading. → **TODO: read RefSlotCache.sol**

**Severity if exploitable:** info / low. Lido core code calls this with controlled values.

### Observation 2 — `_applyVaultReport` maxLiabilityShares match-only-update logic

`VaultHub.sol:1100-1133`. The `if (_record.maxLiabilityShares == _reportMaxLiabilityShares)` branch is the "unlock" gate. The intent is to prevent the 1-tx loop the comment describes.

Verified by walking scenarios:
- Mint + burn + apply-report (where mint happened AFTER refSlot): on-chain max diverges from report max → no update → lock stays high. ✓ (intended)
- Mint + burn + apply-report (where mint+burn both happened BEFORE refSlot): on-chain max matches report max → update with `max(currentLiab, reportLiab)` → can drop to 0. ✓ (intended)

**Question: can the oracle's `_reportMaxLiabilityShares` be deflated via an attack?** If `LazyOracle` only samples liabilityShares at refSlot rather than computing period-max from accounting events, it could miss intra-period mint+burn cycles. → **TODO: verify LazyOracle's max-liability computation**

### Observation 3 — Reentrancy via `_withdrawFromVault → IStakingVault.withdraw`

`VaultHub.sol:1147-1155, 1600-1602`. The withdraw flow:
1. Check totalValue
2. `_updateInOutDelta` (state update)
3. `IStakingVault(_vault).withdraw(_recipient, _amount)` (external call)

State is updated before external call (CEI-compliant). On reentry, `_withdrawableValue` re-reads state and would see the reduced balance.

`receive() external payable {}` has no logic. ✓

→ **NOT exploitable as far as the V4-style hook reentry analogy goes.** Lower priority; will re-check after reading StakingVault.

### Observation 4 — `_unsettledLidoFeesValue` underflow possibility

`VaultHub.sol:1536-1538`:
```solidity
return _record.cumulativeLidoFees - _record.settledLidoFees;
```

If `settledLidoFees > cumulativeLidoFees`, this underflow-reverts in 0.8.x. The comment at line 524 says "LazyOracle sanity checks already verify that the fee can only increase" — so cumulative is monotonic.

But: settledLidoFees is updated by `_settleLidoFees` calls. Could there be a flow where settled exceeds cumulative? E.g., accounting bug where the same fee is settled twice? → **TODO: trace `_settleLidoFees` to verify settled monotonicity is enforced**

If exploitable: vault would get stuck (every read of `_unsettledLidoFeesValue` reverts), denying user actions including disconnect. Severity: medium DoS at most.

### TODOs (to verify)

- [ ] Read `RefSlotCache.sol` and `DoubleRefSlotCache` to verify cache semantics match the documented intent
- [ ] Read `LazyOracle.sol` for max-liability-shares computation (does it sample or compute period-max?)
- [ ] Read `StakingVault.sol` withdraw path for reentrancy
- [ ] Read `_settleLidoFees` to confirm settled ≤ cumulative invariant
- [ ] Read `ValidatorConsolidationRequests.sol` for EIP-7251 consolidation primitive
- [ ] Check `triggerValidatorWithdrawals` no-fresh-report path for full withdrawals
- [ ] Check `_updateInOutDelta` cache update logic

---

## Files reviewed (with depth)

| File | Lines | Depth | Notes |
|------|-------|-------|-------|
| `VaultHub.sol` | ~1900 | Full read + targeted re-reads | Central hub, accounting, withdraw/mint/burn flows, bad-debt mechanism, vault lifecycle |
| `StakingVault.sol` | ~640 | Full read | Per-vault contract, validator deposits/exits/withdrawals, balance accounting |
| `ValidatorConsolidationRequests.sol` | 216 | Full read | EIP-7251 calldata generator, fee exemption flow |
| `NodeOperatorFee.sol` | partial | Targeted (fee exemption + fee calculation) | `addFeeExemption`, `_calculateFee` |
| `LazyOracle.sol` | (not read) | Skipped | Time-budget — flagged as TODO for follow-up |
| `OperatorGrid.sol` | (not read) | Skipped | Time-budget — flagged as TODO for follow-up |
| `RefSlotCache.sol` | (not read) | Skipped | Time-budget — flagged as TODO |

## Findings — concrete items

### F1. `decreaseInternalizedBadDebt` cache-bypass — **info / low**

`VaultHub.sol:684-689`. Direct mutation of `_storage().badDebtToInternalize.value` bypasses the `RefSlotCache` write-helper (`withValueIncrease`). After a decrease, `getValueForLastRefSlot` returns the cached snapshot value, not the post-decrease current value. May be intentional (snapshot semantics), but inconsistent with the increase path which properly updates the cache. Defense-in-depth fix: use a `withValueDecrease` helper or document the intent in the function header.

**Exploit path:** none — caller is the `accounting` contract, internal trust boundary. No external attacker can call this.

**Severity if no exploit path: info.**

### F2. `addFeeExemption` allows the role-holder to under-pay node operator fees — **medium-by-design / no-vuln**

`NodeOperatorFee.sol:268-269 + 341-345`. The `NODE_OPERATOR_FEE_EXEMPT_ROLE` holder can call `addFeeExemption(X)` with any X up to `MAX_SANE_SETTLED_GROWTH`. This increases `settledGrowth`, reducing the unsettled-growth-based fee calculation. The user is meant to use this to exempt consolidated balances from fee — but nothing prevents over-exemption.

**Mitigation:** the role is OPT-IN granted by the node operator to a trusted withdrawal-credentials account. If the operator grants to a malicious actor, fees are lost, but the trust is on the operator. Documented at `ValidatorConsolidationRequests.sol:12-22` and `:79-81` ("not a precise method").

**Verdict:** by-design trust assumption, not a contract bug.

### F3. `_unsettledLidoFeesValue` underflow possibility — **conditional medium**

`VaultHub.sol:1536-1538`:
```solidity
return _record.cumulativeLidoFees - _record.settledLidoFees;
```

If `settledLidoFees > cumulativeLidoFees`, this auto-reverts (Solidity 0.8.x). The comment at L524 says LazyOracle sanity-checks ensure cumulative is monotonic non-decreasing. BUT: `settledLidoFees` is updated by `_settleLidoFees` (not read in this audit) and could in principle race ahead if the report applies a smaller cumulative than what's already been settled.

**Impact if reachable:** every call that reads `_unsettledLidoFeesValue` would revert — including `withdrawableValue`, `_initiateDisconnection`, `obligations`. Vault becomes locked from owner's perspective until governance fix.

**Concrete attack path:** would require either an oracle bug (cumulativeLidoFees decreased) or a settledLidoFees increase that exceeded cumulative. Both shouldn't happen if invariants hold; but I did not verify the invariants by reading LazyOracle.

**Severity if reachable: medium DoS.** **Severity as documented: not exploitable by itself.**

### F4. `_applyVaultReport` maxLiabilityShares match-only-update — **observation, depends on LazyOracle**

`VaultHub.sol:1100-1133`. Only updates `maxLiabilityShares` from the report when current matches report's max. The intent: prevent a 1-tx exploit where mint→burn within the same tx then a stale report unlocks the lock.

**Soundness depends on:** LazyOracle correctly computing the period-max (i.e., the highest liabilityShares observed by the oracle between refSlot N and refSlot N+1, not just the value at refSlot N+1). I did not read LazyOracle to verify this. If the oracle samples at refSlot only, an attacker could exploit:
1. Period: 0 → mint X → operate → burn X → end-of-period
2. Oracle's reported max for the period = 0 (because oracle only sampled at endpoints)
3. On-chain max = X (set by _increaseLiability)
4. Match check fails → no update → max stays X → lock stays high.

Actually wait — that scenario keeps the lock HIGH, which is conservative (good for protocol). The attacker doesn't benefit. **Reverse** scenario:
1. Period: oldMax → no activity → end-of-period
2. Oracle's reported max = oldMax. on-chain max = oldMax. Match → reset to max(currentLiab, reportLiab).
3. If currentLiab=0 and reportLiab=0, max becomes 0. ✓ OK.

Could not construct a concrete exploit. **Verdict:** logic appears correct given oracle integrity. Flag for deeper LazyOracle audit.

### F5. Reentrancy via `withdraw` recipient — **not exploitable**

State updates precede external calls. Reentry sees reduced `availableBalance` and reduced `_totalValue`. Verified.

### F6. `triggerValidatorWithdrawals` no-fresh-report path for full withdrawals — **observation**

`VaultHub.sol:903-905`. `_requireFreshReport` only fires when `hasPartialWithdrawals == true`. For full-only withdrawals, no freshness check.

**Concrete impact:** a vault owner can trigger validator full withdrawals using a stale report. The trigger does not depend on report data — it just queues an EIP-7002 request. The ETH eventually flows back to the vault (post-validator-exit-delay). The accounting catches up via the next oracle report.

**Verdict:** intended; flagged in the function comment. Not a vuln.

---

## Conclusion

**Outcome:** No critical / high vulnerability found in the time-budgeted audit pass. Several observations and one conditional medium flagged, none exploitable in the form they appear.

**Quality of attack surface:** Lido V3 Vaults code is well-structured, with explicit storage-layout discipline (ERC-7201 namespace), careful CEI patterns, and conservative accounting (e.g., `_isThresholdBreached` rounds up liability and rounds down threshold, biasing safety). The team has clearly thought about the 1-tx mint→burn pattern (line 1113-1118 comment).

**What I did NOT audit:** `LazyOracle.sol` (32KB), `OperatorGrid.sol` (38KB), `RefSlotCache.sol`, `Dashboard.sol`, `Permissions.sol`, `PredepositGuarantee` directory, `lib/`, `interfaces/`. Each of these contains nontrivial logic that could host findings I missed.

**Highest-EV follow-up areas (next session):**
1. **`LazyOracle.sol`** — verify sanity checks on cumulativeLidoFees monotonicity, validate maxLiabilityShares period-max computation. ~2-3h read. Could resolve F3 and F4 to either confirmed-no-issue or exploitable.
2. **`OperatorGrid.sol`** — vault-tier transitions, share-limit math, jail mechanism. Recently-changed surface.
3. **`RefSlotCache.sol`** — the cache pattern is used everywhere; correctness here is load-bearing.
4. **Cross-contract invariants** — e.g., does any path let `settledLidoFees` exceed `cumulativeLidoFees` (F3 concrete construction)?
5. **Re-read `withdrawal-bad-debt.integration.ts`** carefully — the test contains assertions about the contract's actual numerical behavior that may reveal hidden invariants.

**Wager outcome (for handoff):**
- Predicted EV: $50K (10% × $500K conservative critical)
- Actual EV: $0 (no findings) + ~$300 reputational (research note quality is decent)
- Realized < predicted. The Lido pivot was the right call (higher base rate than V4) but the surface has indeed been audited to a high standard. Pivot to LazyOracle + OperatorGrid in a follow-up session has clean continuation value.

**Lesson for the methodology:**
- Recent commits ≠ unaudited surface. Lido V3 launched late 2025 and went through a multi-firm audit cycle; the recent commits are mostly polish + integration tests catching forked-state edge cases.
- Truly fresh surface (post-Pectra EIP-7251 work) is small (`ValidatorConsolidationRequests.sol` is only 216 lines and is mostly a calldata generator).
- **For next bounty hunt:** filter by "non-trivial Solidity LoC pushed in last 14 days, by a protocol with active bounty program" — and look for protocols that haven't had a major audit firm engagement in the last 6 months.

