# DISCOVERY — organvm/lido-v3-audit-2026-05-04

**Date:** 2026-06-22 · **Verdict:** PROMOTE (real latent value) · **Tier:** ranked (value-repos.json)

## Value Thesis

On its face this repo is a single time-boxed (90-minute) security-audit log of Lido's
V3 Stakers Vaults that found **no critical bug** — the author's own ledger records a
realized bounty EV of `$0`. Read only as a bounty submission it is near-archival. But
that reading misses where the value actually sits. This repo is a **credibility-and-lead-
generation asset wired to a live revenue funnel**: it is a public, concrete demonstration
that the author can reason about a top-tier, multi-billion-TVL DeFi protocol's contracts
(VaultHub, StakingVault, EIP-7251 consolidation) using a 1M-context model for cross-file
invariant analysis — and the README closes with a services table ($2,000+ smart-contract
audits, $500 architecture reviews) and a direct **"Book via the consult page"** CTA. The
audit's quality is real: it surfaces six well-reasoned observations including a *conditional
medium* (`F3` fee-underflow DoS) and two findings (`F3`, `F4`) explicitly scoped as live,
unresolved follow-ups against Lido's **$2M Immunefi ceiling**. So the repo holds two
distinct, honest forms of latent value: (1) a **reusable audit methodology** — the EV-driven
target-selection framework ("non-trivial Solidity LoC pushed in the last 14 days, by a
protocol with an active bounty program, no major firm engagement in 6 months") plus a
seven-point vulnerability pattern checklist and a documented 1M-context workflow — that the
estate can templatize to repeatably mint *more* such artifacts, each one simultaneously a
portfolio proof-point feeding the consult funnel **and** a bounty lottery ticket; and (2) a
**named, scoped revenue continuation** — finishing the LazyOracle/OperatorGrid pass to
resolve F3/F4 from "conditional" to confirmed-or-exploitable. The capability the rest of the
estate can use is the methodology: turning this one-off into a repeatable engine converts an
ORGAN-III (Commerce) income experiment into a compounding service line. This is not archival.

## What's actually here (verified)

- `AUDIT.md` (~17.6 KB) — the full working audit log: target-selection rationale (Lido V3 vs
  Uniswap V4 EV table), a seven-point pattern checklist, surface map, six findings (F1–F6)
  with severities and exploitability verdicts, an explicit TODO list of unresolved invariants,
  a files-reviewed-with-depth table, and an honest "wager outcome" retrospective.
- `README.md` (~3.7 KB) — TL;DR findings table, files-reviewed table, methodology, honest
  assessment, **and the monetization layer**: a services/pricing table + consult-page CTA
  (added in commit `dceaad4`).
- `LICENSE` — MIT (the log is explicitly reusable).
- No code, no build, no CI — the asset is the methodology and the credibility/funnel, not a
  program.

## Highest latent value (specific & honest)

1. **Reusable audit methodology (estate-wide capability).** The EV target-selection framework
   + pattern checklist + 1M-context cross-file reasoning workflow is the transferable asset.
   Extracted into a repo-agnostic playbook, it lets the estate produce additional audit
   artifacts cheaply (~90 min of model time each), each compounding the consult funnel and
   carrying real bounty upside. *This is the value "the rest of the estate could use."*
2. **Live consult/lead-gen funnel.** A recognizable-protocol audit + direct CTA is a working
   top-of-funnel for $2K+ engagements; the marginal cost of pointing more traffic at it
   (portfolio cross-link, audit index) is near zero.
3. **Scoped bounty EV (unfinished).** F3/F4 remain "conditional medium" pending a
   `LazyOracle.sol` + `OperatorGrid.sol` read against a $2M ceiling — the author already
   scoped the files and the exact invariants to check.

## Single best concrete first task

**Extract the methodology into a reusable, repo-agnostic `PLAYBOOK.md`** — lift the
EV-driven target-selection framework, the seven-point vulnerability pattern checklist, the
pivot/time-box discipline, and the 1M-context cross-file workflow out of `AUDIT.md`'s prose
into a standalone template any ORGAN-III income session can run to generate a new audit
artifact. This is the keystone move: it converts a one-off log into a **repeatable
audit-generation engine**, turning the repo from a single proof-point into a reusable estate
capability and a compounding lead-gen + bounty pipeline. It is self-contained (no external
dependencies), keeps the build green (docs-only), and unblocks build-out. The named revenue
continuation — finishing the LazyOracle/OperatorGrid pass to resolve F3/F4 against the $2M
ceiling — is the natural second task once the playbook makes such passes repeatable.

## Build status

Green by construction — docs-only repo, no build or CI. This discovery adds two Markdown/JSON
files and changes no code path.
