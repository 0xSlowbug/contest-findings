## Title: ETH Stuck in Contract Due to Excess Handling in _buyQuote

**Severity:** Medium

## Description

The contract includes a `receive()` function that allows it to accept ETH directly, even when no specific function is invoked. While this feature enables flexibility, it introduces a vulnerability where 1 wei (or potentially more) of ETH can become permanently stuck in the contract due to how the `_buyQuote` function handles excess ETH.

Here's a link to the relevant section of the code: [LamboVEthRouter.sol](https://github.com/code-423n4/2024-12-lambowin/blob/874fafc7b27042c59bdd765073f5e412a3b79192/src/LamboVEthRouter.sol#L148-L188).

When a user calls the `buyQuote` function, the following logic in `_buyQuote` is responsible for returning excess ETH to the user:

```solidity
if (msg.value > (amountXIn + fee + 1)) {
    (bool success, ) = payable(msg.sender).call{value: msg.value - amountXIn - fee - 1}("");
    require(success, "ETH transfer failed");
}
```

If the `msg.value` provided by the user is exactly `amountXIn + fee + 1`, the surplus of 1 wei will not be returned. Instead, it remains stuck in the contract. The `receive()` function further exacerbates this issue by allowing the contract to passively accumulate unintended ETH deposits, which cannot be retrieved due to the absence of a withdrawal mechanism for the owner or users.

## Impact

### Stuck Funds

- Over time, small amounts of ETH (like 1 wei) can accumulate, resulting in unrecoverable funds.

### Reputation Risk

- Users may lose trust in the contract if even negligible amounts of funds become inaccessible, especially in scenarios involving precise accounting.

## Proof of Concept

### Explanation

The vulnerability arises because `_buyQuote` checks for excess ETH but does not handle the exact scenario where `msg.value` equals `amountXIn + fee + 1`. This leaves 1 wei stuck in the contract.

### Steps to Reproduce

1. Deploy the contract.
2. Call the `buyQuote` function with `msg.value` set to `amountXIn + fee + 1`.
3. Observe that 1 wei remains stuck in the contract.

### Code

Below is the PoC test. Add this code to the `GeneralTest.t.sol` file:

```solidity
function test_funds_stuck_in_contract() public {
    // Create a launch pad
    (address quoteToken, ) = factory.createLaunchPad(
        "TestToken",
        "TEST",
        10 ether,
        address(vETH)
    );

    // Calculate exact amount that would cause funds to be stuck
    uint256 amountXIn = 10 ether;
    uint256 feeRate = lamboRouter.feeRate();
    uint256 feeDenominator = lamboRouter.feeDenominator();
    
    // Calculate the fee
    uint256 fee = (amountXIn * feeRate) / feeDenominator;
    
    // Prepare exact amount that would cause funds to be stuck
    uint256 msgValue = amountXIn + fee + 1;

    // Check contract balance before
    uint256 initialContractBalance = address(this).balance;
    uint256 initialRouterBalance = address(lamboRouter).balance;

    // Perform the buy
    vm.deal(address(this), msgValue);
    IERC20(quoteToken).approve(address(lamboRouter), type(uint256).max);
    
    lamboRouter.buyQuote{value: msgValue}(quoteToken, amountXIn, 0);

    // Check contract balance after
    uint256 finalContractBalance = address(this).balance;
    uint256 finalRouterBalance = address(lamboRouter).balance;

    // 1 wei should be stuck in the contract
    assertEq(finalRouterBalance - initialRouterBalance, 1, "1 wei should be stuck in the contract");
    
    // The contract should have 1 wei
    console2.log("Router contract balance after: ", finalRouterBalance);
    console2.log("Initial router balance: ", initialRouterBalance);
    console2.log("msg.value", msgValue);
    console2.log("amountXIn", amountXIn);
    console2.log("fee", fee);
}
```

### Observations

- After executing the `buyQuote` function with the calculated `msg.value`, 1 wei remains in the contract balance.
- Repeated calls or other ETH deposits through `receive()` will accumulate over time without a recovery mechanism.

## Recommended Mitigation Steps

### Suggested Fixes

#### Handle Exact Excess ETH

Modify the `_buyQuote` logic to ensure no wei is left in the contract, even when `msg.value` is exactly `amountXIn + fee + 1`:

```solidity
if (msg.value > (amountXIn + fee)) {
    (bool success, ) = payable(msg.sender).call{value: msg.value - amountXIn - fee}("");
    require(success, "ETH transfer failed");
}
```

#### Add an Owner Withdrawal Function

Allow the owner to withdraw stuck ETH:

```solidity
function withdrawStuckETH() external onlyOwner {
    uint256 balance = address(this).balance;
    require(balance > 0, "No ETH to withdraw");
    payable(owner()).transfer(balance);
}
```

