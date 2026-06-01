# NRX OTC Platform: Post-Audit Frontend & Backend Integration Guide

This guide contains only the application-layer changes required for the OTC contracts after reviewing:

- `contracts/NRXFund.sol`
- `contracts/NRXCrowdsale.sol`
- `artifacts/contracts/NRXFund.sol/NRXFund.json`
- `artifacts/contracts/NRXCrowdsale.sol/NRXCrowdsale.json`

---

## 1. Contract Changes That Affect Integration

| Area | Current contract behavior | Required frontend/backend update |
| :--- | :--- | :--- |
| **Role-based operations** | `NRXFund` and `NRXCrowdsale` use `accessControl.hasActiveRole(role, account)` for operational actions. | Gate admin screens and backend jobs by active role, not by old whitelist assumptions. |
| **Two-step ownership** | Both contracts inherit `Ownable2StepUpgradeable`; UUPS upgrades are still owner-authorized. | Support `transferOwnership` and `acceptOwnership`; production owner should be a multisig. |
| **Pull-based yield** | `setMonthlyYield` stores total monthly USDC yield; `claimYields` and `settleClaim` compute investor shares lazily. | Do not display `Investment.totalYieldAccumulated` as pending yield. Compute previews from monthly yield and `fundActiveNRXPerMonth`. |
| **Crowdsale slippage protection** | `NRXCrowdsale.buyTokens(usdcAmount, minTokensOut)` reverts if output is below `minTokensOut`. | Quote with `getTokenAmountForUSDC`, apply user slippage tolerance, and pass `minTokensOut`. |
| **Oracle staleness checks** | Both contracts reject stale Chainlink EUR/USD data with `Price too old` / `Stale price`. | Surface stale oracle errors and monitor feed freshness against `oracleHeartbeat`. |
| **Token address immutability** | NRX/USDC setter functions are disabled/commented out. | Remove token swap UI. Token migration requires upgrade/redeploy planning. |
| **Fee wallet separation** | `NRXFund.adminWallet` receives subscription/exit fees and is updated by `PARAM_ADMIN_ROLE`. | Add separate fee-wallet rotation UI; do not tie fee wallet to ownership transfer. |
| **Fee caps** | `createFund` caps `subscriptionFee` and `exitFee` at `5000` bps. | Enforce max 50% in fund creation forms before sending transactions. |
| **Emergency pause** | `pause()` requires `EMERGENCY_ROLE`; `unpause()` requires `DEFAULT_ADMIN_ROLE`. | Read `paused()` and disable user flows when paused; add ops controls for pause/unpause. |
| **Treasury withdrawals** | `withdrawTokens`, `withdrawUSDC`, and `withdrawNRX` require `TREASURY_ROLE`. | Restrict treasury UI/actions to treasury-role accounts and monitor withdrawal events. |

---

## 2. Frontend Updates

### 2.1 Role-Gated Admin UI

Use the shared `AccessControl` contract for all admin visibility and actions.

```typescript
const OTC_ROLES = {
  DEFAULT_ADMIN_ROLE: ethers.ZeroHash,
  ASSET_LISTING_ROLE: ethers.keccak256(ethers.toUtf8Bytes("ASSET_LISTING_ROLE")),
  INCOME_POSTER_ROLE: ethers.keccak256(ethers.toUtf8Bytes("INCOME_POSTER_ROLE")),
  TREASURY_ROLE: ethers.keccak256(ethers.toUtf8Bytes("TREASURY_ROLE")),
  EMERGENCY_ROLE: ethers.keccak256(ethers.toUtf8Bytes("EMERGENCY_ROLE")),
  PARAM_ADMIN_ROLE: ethers.keccak256(ethers.toUtf8Bytes("PARAM_ADMIN_ROLE")),
};

const canCreateFund = await accessControl.hasActiveRole(OTC_ROLES.ASSET_LISTING_ROLE, user);
const canPostYield = await accessControl.hasActiveRole(OTC_ROLES.INCOME_POSTER_ROLE, user);
```

| UI action | Contract call | Required role |
| :--- | :--- | :--- |
| Create fund | `NRXFund.createFund` | `ASSET_LISTING_ROLE` |
| Approve/reject investment | `approveInvestment`, `rejectInvestment` | `ASSET_LISTING_ROLE` |
| Close fund | `closeFund` | `ASSET_LISTING_ROLE` |
| Post monthly yield | `setMonthlyYield` | `INCOME_POSTER_ROLE` |
| Settle matured claim | `settleClaim` | `INCOME_POSTER_ROLE` |
| Rotate fee wallet | `updateAdminWallet` | `PARAM_ADMIN_ROLE` |
| Update oracle/feed/rate | `setOracleHeartbeat`, `updatePriceFeeds`, `setRate` | `PARAM_ADMIN_ROLE` |
| Treasury withdrawals | `withdrawTokens`, `withdrawUSDC`, `withdrawNRX` | `TREASURY_ROLE` |
| Pause | `pause` | `EMERGENCY_ROLE` |
| Unpause | `unpause` | `DEFAULT_ADMIN_ROLE` |

### 2.2 Crowdsale Purchase Flow

`NRXCrowdsale.buyTokens` requires `usdcAmount` and `minTokensOut`.

```typescript
async function buyNrxWithUsdc(usdcAmountInput: string, slippageBps = 50n) {
  const usdcAmount = ethers.parseUnits(usdcAmountInput, 6);
  const expectedTokensOut = await crowdsale.getTokenAmountForUSDC(usdcAmount);
  const minTokensOut = (expectedTokensOut * (10_000n - slippageBps)) / 10_000n;

  const allowance = await usdc.allowance(userAddress, crowdsaleAddress);
  if (allowance < usdcAmount) {
    await (await usdc.approve(crowdsaleAddress, usdcAmount)).wait();
  }

  return crowdsale.buyTokens(usdcAmount, minTokensOut);
}
```

Frontend requirements:

- USDC uses 6 decimals; NRX uses 18 decimals.
- Show quote expiry because the EUR/USD oracle price and `rate` can change before mining.
- Catch and explain `Slippage too high`, `Price too old`, `Invalid price feed`, `Round not complete`, and `Stale price`.
- Read `crowdsale.paused()` and block purchases when paused.
- Check `nrxToken.balanceOf(crowdsaleAddress)` before large buys to detect insufficient sale inventory.

### 2.3 Investment Request and Approval Flow

Investor flow:

1. Read `getFundInfo(fundId)`.
2. Validate launch status, active/closed status, min/max investment, subscription type, and target capacity.
3. Ask investor to approve gross NRX amount to `NRXFund`.
4. Call `invest(fundId, grossAmount)`.
5. Display the emitted `investmentId` as `Requested` until an operator approves it.

Operator flow:

1. Verify caller has `ASSET_LISTING_ROLE`.
2. Call `approveInvestment(investmentId)` or `rejectInvestment(investmentId, reason)`.
3. On approval, contract pulls gross NRX from investor, sends subscription fee to `adminWallet`, and activates the net investment.
4. Refresh `getInvestment`, `getNextYieldClaimTime`, `getMaturityEndTime`, and `getAllYieldClaimTimes`.

### 2.4 Pull-Based Yield Display and Claims

`Investment.totalYieldAccumulated` is not the live pending-yield value. Calculate pending USDC yield per investment/month.

```typescript
async function previewInvestmentYield(fundId: bigint, investmentId: bigint, year: bigint, month: bigint) {
  const monthYearKey = (year * 100n) + month;

  const alreadyClaimed = await nrxFund.monthYearClaimed(investmentId, monthYearKey);
  if (alreadyClaimed) return 0n;

  const investment = await nrxFund.getInvestment(fundId, investmentId);
  if (!investment.active) return 0n;

  const monthlyYield = await nrxFund.getMonthlyYield(fundId, year, month);
  if (!monthlyYield.isSet || monthlyYield.totalYieldUSDC === 0n) return 0n;

  const activeNRX = await nrxFund.fundActiveNRXPerMonth(fundId, monthYearKey);
  if (activeNRX === 0n) return 0n;

  return (investment.amount * monthlyYield.totalYieldUSDC) / activeNRX;
}
```

Claim flow:

- Use `canClaimYields(fundId, investmentId)` and `getAllYieldClaimTimes(fundId, investmentId)` for scheduling UX.
- Build grouped arrays for `claimYields(fundIds, monthsByFund, yearsByFund)`.
- The contract skips already-claimed, incomplete, inactive, and unset-yield months.
- The transaction reverts if total claimable yield is zero.
- Display actual claimed USDC from `YieldsClaimedDetail`.

### 2.5 Mature Claim Settlement

`settleClaim(investmentId, preferredToken, recipient)` is operator-driven and requires `INCOME_POSTER_ROLE`.

| Value | Withdrawal token | Behavior |
| :--- | :--- | :--- |
| `0` | `NRX` | Transfers remaining NRX principal plus USDC yield converted to NRX. |
| `1` | `USDC` | Converts remaining NRX principal to USDC and adds unclaimed USDC yield. |
| `2` | `EURO` | Emits calculated EURO amount only; no on-chain EURO transfer. |

EURO settlement must be handled as an off-chain payment workflow. Use `ClaimCompleted` for reconciliation.

### 2.6 Fund Creation Validation

Validate before `createFund`:

- `name` and `category` are required.
- `launchTimestamp >= current block timestamp`.
- `targetAmount > 0`.
- `minInvestment > 0`.
- `maturityPeriodMonths > 0`.
- `subscriptionType` is `0` or `1`.
- `distributionFrequency` is `0`, `1`, `2`, or `3`.
- `subscriptionFee <= 5000` bps.
- `exitFee <= 5000` bps.
- `maxInvestment` should be `0` or at least `minInvestment`. This is not enforced in `createFund`, so enforce it in frontend/backend validation.

### 2.7 Ownership, Pause, and Token Controls

Ownership is two-step:

```typescript
await nrxFund.transferOwnership(newOwner);
await nrxFund.connect(newOwnerSigner).acceptOwnership();
```

Pause controls:

```typescript
await nrxFund.pause();       // EMERGENCY_ROLE
await crowdsale.pause();     // EMERGENCY_ROLE
await nrxFund.unpause();     // DEFAULT_ADMIN_ROLE
await crowdsale.unpause();   // DEFAULT_ADMIN_ROLE
```

Fee wallet rotation:

```typescript
await nrxFund.updateAdminWallet(newFeeWallet); // PARAM_ADMIN_ROLE
```

Remove any UI for changing `nrxToken` or `usdcToken`; token setters are disabled.

---

## 3. Backend & DevOps Updates

### 3.1 Deployment Configuration

Current initializers:

```typescript
await upgrades.deployProxy(NRXFund, [
  nrxTokenAddress,
  usdcTokenAddress,
  initialOwner,
  eurUsdPriceFeedAddress,
  accessControlAddress,
], { initializer: "initialize", kind: "uups" });

await upgrades.deployProxy(NRXCrowdsale, [
  nrxTokenAddress,
  usdcTokenAddress,
  eurUsdPriceFeedAddress,
  accessControlAddress,
], { initializer: "initialize", kind: "uups" });
```

Post-deploy checklist:

1. Grant production roles in `AccessControl`.
2. Transfer ownership to upgrade multisig with `transferOwnership` and `acceptOwnership`.
3. Confirm `adminWallet` is the fee recipient wallet.
4. Confirm `oracleHeartbeat` matches the target Chainlink feed heartbeat.
5. Confirm `testMode == false` and `secondsPerMonth == 2629743` for production.
6. Fund `NRXCrowdsale` with enough NRX inventory before enabling purchases.

### 3.2 Production Month Configuration

`NRXFund` production defaults:

- `testMode = false`
- `secondsPerMonth = 2629743`
- `BASE_YEAR = 2026`
- `BASE_MONTH = 1`

Only use `setTestMode(true, secondsPerMonth)` in local/staging. Production scripts should skip `setTestMode` or call `setTestMode(false, 0)`.

### 3.3 Monthly Yield Posting Job

Before `setMonthlyYield(fundId, year, month, totalYieldUSDC)`, backend should verify:

- Caller has `INCOME_POSTER_ROLE`.
- `totalYieldUSDC > 0`.
- `month` is `1..12`.
- Fund exists and has investments.
- Target month is not before `fundFirstInvestmentMonthKey`.
- Target month is not in the future relative to `getCurrentMonthYear()`.
- There is at least one active investment for the target month.
- Poster has USDC balance and allowance for the required top-up.

Current-month updates replace the stored yield amount. If the new value is lower than the old value, the contract refunds the difference to the caller. Backend accounting must treat current-month updates as replacements, not additive deposits.

### 3.4 Claim Settlement Liquidity Checks

Before `settleClaim`, backend should verify payout liquidity:

- `WithdrawalToken.NRX`: `NRXFund` needs enough NRX for remaining principal plus converted yield, and enough NRX for exit fee.
- `WithdrawalToken.USDC`: `NRXFund` needs enough USDC for converted principal plus unclaimed yield, and enough NRX for exit fee.
- `WithdrawalToken.EURO`: no token transfer occurs; backend must track an off-chain payment obligation.

Use `ClaimCompleted` as the canonical settlement reconciliation event.

### 3.5 Oracle Monitoring

Both contracts depend on the EUR/USD Chainlink feed. Monitor `latestRoundData()` and alert before `block.timestamp - updatedAt > oracleHeartbeat`.

Operational adjustment:

```typescript
await nrxFund.setOracleHeartbeat(7200);      // PARAM_ADMIN_ROLE
await crowdsale.setOracleHeartbeat(7200);    // PARAM_ADMIN_ROLE
```

For L2 deployments, add a Chainlink Sequencer Uptime Feed check in deployment requirements. The current contracts do not implement sequencer checks internally.

### 3.6 Safe Upgrades

Before UUPS upgrades:

```typescript
await upgrades.validateUpgrade(proxyAddress, NewImplementation, { kind: "uups" });
await upgrades.upgradeProxy(proxyAddress, NewImplementation, {
  kind: "uups",
  unsafeSkipStorageCheck: false,
});
```

Do not use global `unsafeSkipStorageCheck`. Any per-call bypass must have a documented storage-layout review.

### 3.7 Deployment Manifest

Maintain a shared per-network manifest:

```json
{
  "network": "sepolia",
  "chainId": 11155111,
  "accessControl": "0x...",
  "nrxToken": "0x...",
  "usdcToken": "0x...",
  "eurUsdPriceFeed": "0x...",
  "nrxFund": "0x...",
  "nrxCrowdsale": "0x...",
  "ownerMultisig": "0x...",
  "adminWallet": "0x...",
  "treasury": "0x..."
}
```

Stop hardcoding proxy addresses in operational scripts.

---

## 4. Operations & QA Checklist

### 4.1 Emergency Pause

In the event of a frontend compromise, oracle incident, treasury issue, or accounting bug:

```typescript
await nrxFund.pause();
await crowdsale.pause();
```

Only `DEFAULT_ADMIN_ROLE` can unpause.

### 4.2 Treasury and Fee Controls

- `updateAdminWallet` controls where subscription and exit fees are sent.
- `withdrawTokens`, `withdrawUSDC`, and `withdrawNRX` require `TREASURY_ROLE`.
- Monitor all `TokensWithdrawn` events.
- Treasury withdrawals should be reconciled through backend accounting.

### 4.3 Minimum QA Scenarios

- Crowdsale buy succeeds when `minTokensOut` is below quoted output.
- Crowdsale buy reverts with `Slippage too high` when `minTokensOut` is above output.
- Crowdsale buy is blocked or reverts when sale NRX inventory is insufficient.
- Fund creation rejects invalid fee caps and invalid enum values before transaction submission.
- Investment request emits an ID and remains pending until approval.
- Investment approval pulls gross NRX, transfers subscription fee to `adminWallet`, and activates net investment.
- Past-month yield can be posted once; current-month yield can be replaced and reconciled.
- Yield claim ignores already-claimed months and transfers only available claimable USDC.
- Mature `settleClaim` works for NRX and USDC when contract liquidity is sufficient.
- EURO settlement emits calculated amounts and creates no on-chain token transfer.
- `pause()` blocks buy, invest, yield claim, and settlement flows.
- Ownership transfer requires both `transferOwnership` and `acceptOwnership`.
- Upgrade script runs `validateUpgrade` before `upgradeProxy`.
