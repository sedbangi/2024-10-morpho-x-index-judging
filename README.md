# Issue M-1: _calculateMaxBorrowCollateral calculates repay incorrectly and can lead to set token liquidation 

Source: https://github.com/sherlock-audit/2024-10-morpho-x-index-judging/issues/26 

## Found by 
0x52, Kirkeelee
### Summary

When calculating the amount to repay, `_calculateMaxBorrowCollateral` incorrectly applies `unutilizedLeveragePercentage` when calculating `netRepayLimit`. The result is that if the `borrowBalance` ever exceeds `liquidationThreshold * (1 - unutilizedLeveragPercentage)` then all attempts to repay will revert. This is nearly identical to the [valid issue](https://github.com/sherlock-audit/2023-05-Index-judging/issues/254) reported in the last contest that was fixed in the Aave leverage extension.

### Root Cause

In [MorphoLeverageStrategyExtension.sol:L1124](https://github.com/sherlock-audit/2024-10-morpho-x-index/blob/main/index-coop-smart-contracts/contracts/adapters/MorphoLeverageStrategyExtension.sol#L1124-L1127) the borrow limit is reduced by `unutilizedLeveragePercentage` which will cause [L1134](https://github.com/sherlock-audit/2024-10-morpho-x-index/blob/main/index-coop-smart-contracts/contracts/adapters/MorphoLeverageStrategyExtension.sol#L1134) to underflow and revert.

### Internal pre-conditions

`unutilizedLeveragePercentage` must be a nonzero value. For context this is nonzero for every existing leveraged token currently deployed by Index Coop.

### External pre-conditions

The underlying collateral value decreases rapidly in price pushing the set towards liquidation

### Attack Path

1. The price of the underlying collateral decreases rapidly causing `liquidationThreshold` to drop
2. `borrowBalance` exceeds `liquidationThreshold * (1 - unutilizedLeveragPercentage)`
3. Calls to `MorphoLeverageStrategyExtension#ripcord` will revert due to underflow
4. Set token is liquidated

### Impact

Set token suffers losses due to liquidation fee. For most Morpho markets this is at least 5%. Due to the leveraged nature of the set the loss will be multiplicative. This means a 3x leverage token will lose 15% NAV (5% * 3), 5x leverage will lose 25% NAV (5% * 5), etc.

### PoC

_No response_

### Mitigation

Don't adjust the max value by `unutilizedLeveragPercentage` when deleveraging

