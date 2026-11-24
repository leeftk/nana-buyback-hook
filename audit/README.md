# Nana Buyback Hook v6 Audit — Possible Valid Findings PoCs

Foundry regression tests for Medium findings identified in the audit of the
JBBuybackHook / JBBuybackHookRegistry codebase. Tests live in the project's
`test/regression/` directory and run against
`/Users/welcome1/Coding/uni-v4/nana-buyback-hook-v6`.

## Audit Reports

| ID | File | Date | Model | Pipeline |
|----|------|------|-------|----------|
| R1 | `2026-04-06_light.md` | 2026-04-06 | Sonnet | light — 10 skill agents + adversarial |
| R2 | `2026-04-06_light-2.md` | 2026-04-06 | Sonnet | light — 10 skill agents + adversarial |

---

## M01_CashOutCountValidation.t.sol

**Source**: R1 M-01 (`2026-04-06_light.md`)
**Canonical location**: `test/regression/CashOutCountValidation.t.sol`
**Severity**: Medium | **Confidence**: Low | **Class**: HookData

### Issue

`afterCashOutRecordedWith` does not validate `cashOutCountToSell` against
`context.cashOutCount` (the number of tokens the user actually burned).

Two paths lead to the same invariant violation:

**Path A — empty hookMetadata (line 212):** when `hookMetadata.length == 0`,
the hook uses `context.cashOutCount` directly as `cashOutCountToSell`. A
registered terminal that controls `context.cashOutCount` can pass any value
here without restriction.

**Path B — non-empty hookMetadata (line 214, also covered by M-03 below):**
when metadata is present the hook decodes `cashOutCountToSell` from it:

```solidity
(minimumSwapAmountOut, cashOutCountToSell) = abi.decode(context.hookMetadata, (uint256, uint256));
```

No check enforces `cashOutCountToSell <= context.cashOutCount` in either path.

In both cases `controller.mintTokensOf` is called with the attacker-supplied
count, minting project tokens to the hook out of thin air. Those tokens are
then sold into the Uniswap V4 pool, draining terminal tokens to the attacker's
`beneficiary` address.

**Precondition**: attacker must control a terminal registered in
`DIRECTORY.isTerminalOf(projectId)` — the sole on-chain guard (line 196–198).
Severity is Medium because that registration requires a privileged operation.

### Tests

| Test | What it proves |
|------|---------------|
| `test_baseline_metadataCountEqualsBurned_mintsCorrectAmount` | Correct behaviour — metadata count equals actual burned; `mintTokensOf` receives the right amount |
| `test_M01_emptyMetadata_inflatedContextCount_mintsInflatedAmount` | Path A: empty metadata, terminal supplies inflated `context.cashOutCount`; hook passes it to `mintTokensOf` without validation |
| `test_M01_nonEmptyMetadata_inflatedMetadataCount_mintsInflatedAmount` | Path B: non-empty metadata, terminal encodes inflated `cashOutCountToSell`; hook decodes and uses it verbatim |
| `test_callerNotTerminal_reverts` | Terminal guard works — unregistered callers are rejected; attack requires registered terminal |

---

## M02_FakeControllerInjection.t.sol

**Source**: R1 M-02 (`2026-04-06_light.md`)
**Canonical location**: `test/regression/FakeControllerInjection.t.sol`
**Severity**: Medium | **Confidence**: Medium | **Class**: HookData

### Issue

`afterPayRecordedWith` decodes the controller address from `hookMetadata` at
line 284 without integrity verification:

```solidity
(projectTokenIs0, amountToMintWith, minimumSwapAmountOut, controller, tokenCountWithoutHook)
    = abi.decode(context.hookMetadata, (bool, uint256, uint256, IJBController, uint256));
```

`beforePayRecordedWith` encodes the real `DIRECTORY.controllerOf()` into
`hookMetadata` at line 840–851, but the terminal is trusted to forward that
metadata unchanged. A registered terminal can replace the controller field
with any address before calling `afterPayRecordedWith`.

The injected address is then used for `burnTokensOf` (line 1012),
`currentRulesetOf` (line 319), and `mintTokensOf` (line 398). A fake controller
that is a no-op for `mintTokensOf` causes:

- The payer's terminal tokens are spent (swap executes).
- The project tokens received from the pool are burned via `burnTokensOf` on
  the fake contract (no-op — project tokens remain in the hook, permanently
  stranded).
- `mintTokensOf` on the fake contract is a no-op — beneficiary receives zero
  project tokens.

Additionally, the fake controller can return an inflated `weight` from
`currentRulesetOf`, causing `partialMintTokenCount` to grow — though with
`reservedPercent = 0` this particular lever has no effect on the final
beneficiary mint.

**Precondition**: attacker must control a terminal registered in
`DIRECTORY.isTerminalOf(projectId)`.

### Tests

| Test | What it proves |
|------|---------------|
| `test_baseline_realController_mintCalledOnRealController` | Correct behaviour — real controller in metadata triggers `mintTokensOf` on the real controller and the beneficiary receives project tokens |
| `test_M02_fakeController_beneficiaryReceivesZeroTokens` | Fake controller injected via hookMetadata: swap executes (ETH consumed), `burnTokensOf` and `mintTokensOf` land on the fake contract, beneficiary receives 0 project tokens; pool-received tokens are stuck in the hook |
| `test_M02_fakeController_withInflatedWeight_stillZeroTokensTobeneficiary` | Fake controller returns inflated weight; beneficiary still receives 0 tokens — `mintTokensOf` goes to the fake regardless of weight |

---

## M03_InflatedCashOutCountSell.t.sol

**Source**: R1 M-03 (`2026-04-06_light.md`)
**Canonical location**: `test/regression/InflatedCashOutCountSell.t.sol`
**Severity**: Medium | **Confidence**: Medium | **Class**: HookData

### Issue

This is the non-empty hookMetadata variant of M-01, isolated as a distinct
finding because the exploitation mechanism and preconditions differ slightly.

`afterCashOutRecordedWith` at line 214 decodes `cashOutCountToSell` from
`context.hookMetadata` without validating it against `context.cashOutCount`:

```solidity
(minimumSwapAmountOut, cashOutCountToSell) = abi.decode(context.hookMetadata, (uint256, uint256));
```

A registered terminal encodes an inflated `cashOutCountToSell` in the metadata.
The hook calls `controller.mintTokensOf(cashOutCountToSell, address(this))`
(line 221–227), minting that inflated count to itself. Those tokens are sold
via `_swapExactInput` (line 231–236) and the proceeds forwarded to
`context.beneficiary` (the attacker).

The attacker also sets `minimumSwapAmountOut = 0` in the same metadata
encoding, ensuring the slippage guard at line 239 is trivially satisfied even
if pool conditions are unfavourable.

Key difference from `CashOutCountValidation.t.sol`: the test in this file uses
`ICOS_MintingController` — a controller that actually calls `ERC20._mint` when
`mintTokensOf` is invoked — making `totalSupply` assertions meaningful and
eliminating the need to pre-fund the hook manually.

**Precondition**: attacker must control a terminal registered in
`DIRECTORY.isTerminalOf(projectId)`.

### Tests

| Test | What it proves |
|------|---------------|
| `test_baseline_metadataCountEqualsActualBurned` | Correct behaviour — metadata count equals actual burned; controller mints the right amount, attacker receives fair proceeds |
| `test_M03_inflatedMetadataCount_mintsAndSellsMoreThanBurned` | Non-empty metadata with `cashOutCountToSell = 10_000 ether` vs `actualBurned = 100 ether`; `mintTokensOf` receives the inflated count; attacker extracts 100× the ETH |
| `test_M03_zeroMinimumBypassesSlippageGuard` | `minimumSwapAmountOut = 0` in metadata disables the slippage revert; `totalSupply` equals the inflated count, confirming unbacked supply inflation |
| `test_callerNotTerminal_reverts` | Terminal guard is the sole defense; confirms attack surface is limited to registered terminals |

---

---

## M01_ReservedRateSlippage.t.sol

**Source**: R2 M-01 (`2026-04-06_light-2.md`)
**Severity**: Medium | **Confidence**: High | **Class**: TokenEdgeCases
**Status**: `[POC-PASS]`

### Issue

`afterPayRecordedWith` checks the slippage guard against `exactSwapAmountOut` — the gross number of tokens returned by the pool — before `mintTokensOf` applies the reserved rate split. When `reservedPercent > 0`, the beneficiary receives only `(1 - reservedPercent)` of that gross amount. The guard therefore passes even though the beneficiary ends up with fewer tokens than `minimumSwapAmountOut`.

```solidity
// JBBuybackHook.sol:383
if (!swapFailed && exactSwapAmountOut + partialMintTokenCount < minimumSwapAmountOut)
    revert ...;
```

`exactSwapAmountOut` is the pool delta; `mintTokensOf` later splits it with `reservedPercent`. The beneficiary's actual receipt is `exactSwapAmountOut * (1 - reservedPercent)`.

### Tests

| Test | What it proves |
|------|---------------|
| `test_exploit_M01_beneficiaryReceivesLessThanMinimum` | Pool returns exactly `minimumSwapAmountOut`; 20% reserved rate causes beneficiary to receive only 80% of that — below the stated minimum with no revert |

---

## M04_PartialFillStrandedTokens.t.sol

**Source**: R2 M-04 (`2026-04-06_light-2.md`)
**Severity**: Medium | **Confidence**: High | **Class**: TokenEdgeCases
**Status**: `[POC-PASS]`

### Issue

`afterCashOutRecordedWith` calls `controller.mintTokensOf(cashOutCountToSell, address(this))` before the swap, then passes the full `cashOutCountToSell` as `amountIn` to `_swapExactInput`. If the pool hits `sqrtPriceLimit` before consuming all input (partial fill), only `partialFill` project tokens are spent by the pool but all `cashOutCountToSell` were minted. The unsold `cashOutCountToSell - partialFill` tokens remain in the hook with no recovery path, permanently inflating the project token supply.

The minimum check at line 239 does not catch this because it compares the ETH received (`amountReceived`) against the user's floor, not the stranded token count.

### Tests

| Test | What it proves |
|------|---------------|
| `test_exploit_M04_partialFillStrandsTokensInHook` | Pool consumes 700e18 of 1000e18 project tokens; beneficiary receives ETH above the minimum; 300e18 project tokens permanently stranded in hook; total supply inflated |

---

## M09_FoTSwapSilentFail.t.sol

**Source**: R2 M-09 (`2026-04-06_light-2.md`)
**Severity**: Medium | **Confidence**: High | **Class**: TokenEdgeCases
**Status**: `[POC-PASS]`

### Issue

`afterPayRecordedWith` reads `amountToSwapWith = context.forwardedAmount.value` (nominal, line 974). When the terminal token is fee-on-transfer, the hook receives `nominal * (1 - fee_rate)` after `safeTransferFrom`, but then passes the full `nominal` to `_swapExactInput`. Inside `unlockCallback`, `safeTransfer(poolManager, nominal)` reverts because the hook is short by the fee. The `try/catch` around the swap call catches this, sets `swapFailed = true`, and silently falls through to the mint path.

Consequence: the slippage check at line 383 is guarded by `!swapFailed` and is never reached. The user receives mint-rate tokens (far fewer than the pool-rate `minimumSwapAmountOut` they specified), and the FoT entry fee is permanently lost.

### Tests

| Test | What it proves |
|------|---------------|
| `test_exploit_M09_swapSilentlyFailsForFoTToken` | FoT input token causes swap to silently fail; beneficiary receives ~990e18 mint-rate tokens instead of 2000e18 pool-rate tokens; `minimumSwapAmountOut = 2000e18` silently bypassed; 0.1e18 FOT entry fee permanently lost |

---

## M10_FoTCashOutDoS.t.sol

**Source**: R2 M-10 (`2026-04-06_light-2.md`)
**Severity**: Medium | **Confidence**: High | **Class**: TokenEdgeCases
**Status**: `[POC-PASS]`

### Issue

`afterCashOutRecordedWith` → `_swapExactInput` → `unlockCallback` calls `POOL_MANAGER.take(outputCurrency, address(this), outputAmount)` where `outputAmount` is the pool's gross delta. For a fee-on-transfer terminal token, `take()` internally calls `FoT.transfer(hook, outputAmount)`, which burns a fee from the sender and delivers only `outputAmount * (1 - fee_rate)` to the hook.

`unlockCallback` returns `abi.encode(outputAmount)` — the gross delta, not what the hook holds. Back in `afterCashOutRecordedWith`, `safeTransfer(beneficiary, outputAmount)` reverts because the hook is short. There is no `try/catch` here, so the entire transaction reverts. The user cannot cash out via the pool path; the DoS is permanent for any FoT terminal token.

The slippage guard at line 239 does not help: it compares `amountReceived` (the gross delta) against `minimumSwapAmountOut`, so it passes even when the actual balance is insufficient.

### Tests

| Test | What it proves |
|------|---------------|
| `test_exploit_M10_fotTerminalTokenDoSCashOut` | 1% FoT terminal token; pool take() delivers 99e18 to hook; hook tries to forward 100e18 gross; reverts with `ERC20InsufficientBalance(hook, 98e18, 99e18)`; beneficiary receives 0; entire tx rolled back |

---

## Cross-Report Coverage

| PoC File | R1 (`2026-04-06_light.md`) | R2 (`2026-04-06_light-2.md`) |
|----------|---------------------------|------------------------------|
| M01_CashOutCountValidation | **M-01 (HookData variant)** | — |
| M01_ReservedRateSlippage | — | **M-01 (ReservedRate variant)** |
| M02_FakeControllerInjection | **M-02** | — |
| M03_InflatedCashOutCountSell | **M-03** | — |
| M04_PartialFillStrandedTokens | — | **M-04** |
| M09_FoTSwapSilentFail | — | **M-09** |
| M10_FoTCashOutDoS | — | **M-10** |

### False Positives (no PoC)

| Finding | Reason no PoC |
|---------|--------------|
| ~~H-01~~ | `context.weight` in `JBAfterPayRecordedContext` is the original ruleset weight, not the `0` returned by `beforePayRecordedWith` — invariant holds; regression test `PartialFillMinimumCheck.t.sol` confirms |
