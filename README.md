# Issue H-1: _calculateMaxBorrowCollateral calculates repay incorrectly and can lead to set token liquidation 

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

# Issue M-1: MorphoLeverageModule#removeModule is broken and cannot be used without trapping funds 

Source: https://github.com/sherlock-audit/2024-10-morpho-x-index-judging/issues/37 

## Found by 
0x52
### Summary

Removing a module for a set token is a core functionality of the protocol but MorphoLeverageModule cannot be removed without breaking set token redemption and trapping user funds. The standard methodology is that MorphoLeverageModule#removeModule would be called after the set has deleveraged to zero debt. However it fails to withdraw the collateral from Morpho, resulting in the funds becoming inaccessible after removal.

This also violates the expected behavior of the module based on comments in the [contract itself](https://github.com/sherlock-audit/2024-10-morpho-x-index/blob/main/index-protocol/contracts/protocol/modules/v1/MorphoLeverageModule.sol#L441-L445) and the [set token](https://github.com/sherlock-audit/2024-10-morpho-x-index/blob/main/index-protocol/contracts/protocol/SetToken.sol#L372-L375)

### Root Cause

MorphoLeverageModule#removeModule fails to withdraw collateral from Morpho

### Internal pre-conditions

None. Funds are trapped regardless of amount.

### External pre-conditions

None.

### Attack Path

[SetToken.sol#L376-L387](https://github.com/sherlock-audit/2024-10-morpho-x-index/blob/main/index-protocol/contracts/protocol/SetToken.sol#L376-L387)

    function removeModule(address _module) external onlyManager {
        require(!isLocked, "Only when unlocked");
        require(moduleStates[_module] == ISetToken.ModuleState.INITIALIZED, "Module must be added");

        IModule(_module).removeModule(); <-- @audit call to MorphoLeverageModule

        moduleStates[_module] = ISetToken.ModuleState.NONE;

        modules.removeStorage(_module);

        emit ModuleRemoved(_module);
    }

The removal process begins with the set token which calls MorphoLeverageModule#removeModule.

[MorphoLeverageModule.sol#L446-L462](https://github.com/sherlock-audit/2024-10-morpho-x-index/blob/main/index-protocol/contracts/protocol/modules/v1/MorphoLeverageModule.sol#L446-L462)

    function removeModule()
        external
        override
        onlyValidAndInitializedSet(ISetToken(msg.sender))
    {
        ISetToken setToken = ISetToken(msg.sender);

        sync(setToken);

        delete marketParams[setToken];

        // Try if unregister exists on any of the modules
        address[] memory modules = setToken.getModules();
        for(uint256 i = 0; i < modules.length; i++) {
            try IDebtIssuanceModule(modules[i]).unregisterFromIssuanceModule(setToken) {} catch {} <-- @audit unregisters but never withdraws funds from morpho
        }
    }

This unregisters the MorphoLeverageModule from the debtIssuanceModule and clears the marketParams.

[DebtIssuanceModule.sol#L288-L292](https://github.com/sherlock-audit/2024-10-morpho-x-index/blob/main/index-protocol/contracts/protocol/modules/v1/DebtIssuanceModule.sol#L288-L292)

    function unregisterFromIssuanceModule(ISetToken _setToken) external onlyModule(_setToken) onlyValidAndInitializedSet(_setToken) {
        require(issuanceSettings[_setToken].isModuleHook[msg.sender], "Module not registered.");
        issuanceSettings[_setToken].moduleIssuanceHooks.removeStorage(msg.sender);
        issuanceSettings[_setToken].isModuleHook[msg.sender] = false;
    }

This in turn clears the issuance/redemption hooks, preventing the assets from being accessed.

### Impact

When module is removed, set token redemption breaks and user funds are trapped.

### PoC

_No response_

### Mitigation

MorphoLeverageModule#removeModule should withdraw all funds from Morpho and update the default position for the collateral component.

