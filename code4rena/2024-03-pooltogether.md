## [H-01] `PrizeVault.claimYieldFeeShares()` resets yieldFeeBalance on every call

## Relevant GitHub Links

https://github.com/code-423n4/2024-03-pooltogether/blob/480d58b9e8611c13587f28811864aea138a0021a/pt-v5-vault/src/PrizeVault.sol#L617

## Summary

Calling `PrizeVault.claimYieldFeeShares(uint256 _shares)` with any amount of `_shares` (that passes function conditions) will result in loss of yield (loss of `yieldFeeBalance` inside vault) because it resets `yieldFeeBalance`. When yieldFeeRecipient calls this function with `_shares` that is less then `_yieldFeeBalance` and not equal to 0, it will lead to `_yieldFeeBalance - shares` amount of yield being stuck in the `PrizeVault` contact.

## Impact

Fees are stuck in `PrizeVault` contract.

## Proof of Concept

Add this test to `PrizeVault.t.sol` and run with

```bash
    forge test --match-contract PrizeVaultTest --match-test testClaimYieldFeeShares_LossOfFundsIfClaimNotAllYieldFeeBalance
```

```solidity
    function testClaimYieldFeeShares_LossOfFundsIfClaimNotAllYieldFeeBalance() public {
        vault.setYieldFeePercentage(1e8); // 10%
        vault.setYieldFeeRecipient(bob);
        assertEq(vault.totalDebt(), 0);

        // make an initial deposit
        underlyingAsset.mint(alice, 1e18);
        vm.startPrank(alice);
        underlyingAsset.approve(address(vault), 1e18);
        vault.deposit(1e18, alice);
        vm.stopPrank();

        underlyingAsset.mint(address(vault), 1e18);
        vault.setLiquidationPair(address(this));
        uint256 maxLiquidation = vault.liquidatableBalanceOf(address(underlyingAsset));
        uint256 amountOut = maxLiquidation / 2;
        uint256 yieldFee = (1e18 - vault.yieldBuffer()) / (2 * 10);
        vault.transferTokensOut(address(0), bob, address(underlyingAsset), amountOut);

        vm.prank(bob);
        // claim half of yieldFee
        vault.claimYieldFeeShares(yieldFee / 2);

        // because claimYieldFeeShares resets yieldFeeBalance it will always be 0 not depending on the amount of _shares
        assertEq(vault.yieldFeeBalance(), 0);
    }
```

## Recommended Mitigation Steps

Substruct `_shares` instead of `_yieldFeeBalance` in `PrizeVault.claimYieldFeeShares()`.

```diff
    function claimYieldFeeShares(uint256 _shares) external onlyYieldFeeRecipient {
        if (_shares == 0) revert MintZeroShares();

        uint256 _yieldFeeBalance = yieldFeeBalance;
        if (_shares > _yieldFeeBalance) revert SharesExceedsYieldFeeBalance(_shares, _yieldFeeBalance);

-       yieldFeeBalance -= _yieldFeeBalance;
+       yieldFeeBalance -= _shares;

        _mint(msg.sender, _shares);

        emit ClaimYieldFeeShares(msg.sender, _shares);
    }
```

## [M-01] `PrizeVault.maxDeposit()` doesn't take into account produced fees

## Relevant GitHub Links

https://github.com/code-423n4/2024-03-pooltogether/blob/480d58b9e8611c13587f28811864aea138a0021a/pt-v5-vault/src/PrizeVault.sol#L368-L392

https://github.com/code-423n4/2024-03-pooltogether/blob/480d58b9e8611c13587f28811864aea138a0021a/pt-v5-vault/src/PrizeVault.sol#L685

https://github.com/code-423n4/2024-03-pooltogether/blob/480d58b9e8611c13587f28811864aea138a0021a/pt-v5-vault/src/PrizeVault.sol#L619

## Summary

Current `PrizeVault.maxDeposit()` calculates the maximum possible amount of deposit without taking into account produced fees. That means if there is already maxed deposited amount of asset that is calculated by the current implementation if `PrizeVault.maxDeposit()` `yieldFeeRecipient` can't withdraw shares with `PrizeVault.claimYieldFeeShares()` because in that case `_mint()` will revert because of overflow. A lot of low-price tokens can exceed the limit of `type(uint96).max` with ease. For example to make a deposit with `maxDeposit()` value with LADYS token it's needed only $13568 (as of 08.03.2024).

## Impact

If a user makes a maximum allowed deposit that is calculated by the current implementation of `PrizeVault.maxDeposit()` `yieldFeeRecipient` can't withdraw fees if they are available.

## Proof of Concept

Add this test to `PrizeVault.t.sol` and run with

```bash
    forge test --match-contract PrizeVaultTest --match-test testMaxDeposit_CalculatesWithoutTakingIntoAccountGeneratedFees
```

```solidity
    function _deposit(address account, uint256 amount) private {
        underlyingAsset.mint(account, amount);
        vm.startPrank(account);
        underlyingAsset.approve(address(vault), amount);
        vault.deposit(amount, account);
        vm.stopPrank();
    }

    function testMaxDeposit_CalculatesWithoutTakingIntoAccountGeneratedFees() public {
        vault.setYieldFeePercentage(1e8); // 10%
        vault.setYieldFeeRecipient(bob);

        // alice make initial deposit
        _deposit(alice, 1e18);

        // mint yield to the vault and liquidate
        underlyingAsset.mint(address(vault), 1e18);
        vault.setLiquidationPair(address(this));
        uint256 maxLiquidation = vault.liquidatableBalanceOf(address(underlyingAsset));
        uint256 amountOut = maxLiquidation / 2;
        uint256 yieldFee = (1e18 - vault.yieldBuffer()) / (2 * 10); // 10% yield fee + 90% amountOut = 100%

        // bob transfers tokens out and increase fee
        vault.transferTokensOut(address(0), bob, address(underlyingAsset), amountOut);

        // alice make deposit with maximum available value for deposit
        uint256 maxDeposit = vault.maxDeposit(address(this));
        _deposit(alice, maxDeposit);

        // then bob want to withdraw earned fee but he can't do that
        vm.prank(bob);
        vm.expectRevert();
        vault.claimYieldFeeShares(yieldFee);
    }
```

## Recommended Mitigation Steps

Add function to withdraw fees in `asset`.
Or change function `PrizeVault.maxDeposit()` to calculate max deposit amount with taking into account produced fees:

```diff
    function maxDeposit(address) public view returns (uint256) {
        uint256 _totalSupply = totalSupply();
        uint256 totalDebt_ = _totalDebt(_totalSupply);
        if (totalAssets() < totalDebt_) return 0;

        // the vault will never mint more than 1 share per asset, so no need to convert supply limit to assets
        uint256 twabSupplyLimit_ = _twabSupplyLimit(_totalSupply);
        uint256 _maxDeposit;
        uint256 _latentBalance = _asset.balanceOf(address(this));
        uint256 _maxYieldVaultDeposit = yieldVault.maxDeposit(address(this));
        if (_latentBalance >= _maxYieldVaultDeposit) {
            return 0;
        } else {
            unchecked {
                _maxDeposit = _maxYieldVaultDeposit - _latentBalance;
            }
-           return twabSupplyLimit_ < _maxDeposit ? twabSupplyLimit_ : _maxDeposit;
+           return twabSupplyLimit_ < _maxDeposit ? twabSupplyLimit_ - yieldFeeBalance : _maxDeposit - yieldFeeBalance;
        }
    }
```

## [QA-01] No need to inherit `ERC20` by `TwabERC20` because `ERC20Permit` already inherits `ERC20`

## Relevant GitHub Links

https://github.com/code-423n4/2024-03-pooltogether/blob/480d58b9e8611c13587f28811864aea138a0021a/pt-v5-vault/src/TwabERC20.sol#L19

## Summary

OpenZeppelin's `ERC20Permit` inherits `ERC20` so there is no need to import and inherit `ERC20` in `TwabERC20`

## Recommended Mitigation Steps

Remove `ERC20` when declaring `TwabERC20`

```diff
    /// @dev    This contract is designed to be used as an accounting layer when building a vault
    ///         for PoolTogether V5.
    /// @dev    The TwabController limits all balances including total token supply to uint96 for
    ///         gas savings. Any mints that increase a balance past this limit will fail.
-   contract TwabERC20 is ERC20, ERC20Permit {
+   contract TwabERC20 is ERC20Permit {

    ////////////////////////////////////////////////////////////////////////////////
    // Public Variables
    ////////////////////////////////////////////////////////////////////////////////

    /// @notice Address of the TwabController used to keep track of balances.
    TwabController public immutable twabController;
```

## [QA-02] No need to use signature to approve assets if user has already approved more than enough assets in `PrizeVault.depositWithPermit()`

## Relevant GitHub Links

https://github.com/code-423n4/2024-03-pooltogether/blob/480d58b9e8611c13587f28811864aea138a0021a/pt-v5-vault/src/PrizeVault.sol#L514-L546

## Summary

## Recommended Mitigation Steps

Call `permit()` only if doesn't have enough approved tokens.

```diff
    // Skip the permit call if the allowance has already been set to exactly what is needed. This prevents
    // griefing attacks where the signature is used by another actor to complete the permit before this
    // function is executed.
-   if (_asset.allowance(_owner, address(this)) != _assets) {
+   if (_asset.allowance(_owner, address(this)) < _assets) {
        IERC20Permit(address(_asset)).permit(_owner, address(this), _assets, _deadline, _v, _r, _s);
    }
```

## [QA-03] Use `safePermit()` instead of `permit` in `PrizeVault.depositWithPermit()`

## Relevant GitHub Links

https://github.com/code-423n4/2024-03-pooltogether/blob/480d58b9e8611c13587f28811864aea138a0021a/pt-v5-vault/src/PrizeVault.sol#L540

## Summary

Some tokens can wrongly implement `ERC20Permit`. SafeERC20 library from openzeppelin have `safePermit()` function that provides stronger security that might help in some edge cases with tokens that wrongly implement `ERC20Permit`.

## Recommended Mitigation Steps

```diff
    // Skip the permit call if the allowance has already been set to exactly what is needed. This prevents
    // griefing attacks where the signature is used by another actor to complete the permit before this
    // function is executed.
    if (_asset.allowance(_owner, address(this)) != _assets) {
-       IERC20Permit(address(_asset)).permit(_owner, address(this), _assets, _deadline, _v, _r, _s);
+       SafeERC20.safePermit(IERC20Permit(address(_asset)), _owner, address(this), _assets, _deadline, _v, _r, _s);
    }
```

## [QA-04] Frequently repeated `_maxWithdraw` calculation doesn't have separate function.

## Relevant GitHub Links

https://github.com/code-423n4/2024-03-pooltogether/blob/480d58b9e8611c13587f28811864aea138a0021a/pt-v5-vault/src/PrizeVault.sol#L405

https://github.com/code-423n4/2024-03-pooltogether/blob/480d58b9e8611c13587f28811864aea138a0021a/pt-v5-vault/src/PrizeVault.sol#L416

https://github.com/code-423n4/2024-03-pooltogether/blob/480d58b9e8611c13587f28811864aea138a0021a/pt-v5-vault/src/PrizeVault.sol#L639

## Summary

3 functions in `PrizeVault` contract calculates `_maxWithdraw` by `_maxYieldVaultWithdraw() + _asset.balanceOf(address(this))`. But this operation does not have separate function even though it is used in few places.

## Recommended Mitigation Steps

Create function for `_maxYieldVaultWithdraw() + _asset.balanceOf(address(this))` operation and call it instead.
