
# Bug in `cashIn` Function Prevents Minting of Virtual Tokens for Non-Native Token Deposits  

**Severity:** High  

## Finding Description and Impact  

The `cashIn` function in the smart contract contains a bug that results in the incorrect minting of virtual tokens when non-native tokens are used for deposits. The function relies on `msg.value` to mint tokens, which is only applicable for native tokens (ETH). However, when non-native tokens are deposited, `msg.value` remains zero, causing the function to mint zero tokens. Despite the successful transfer of assets to the contract, no virtual tokens are minted for the user.  

### Impact:  
- **Incorrect token minting:** Users depositing non-native tokens do not receive any virtual tokens, even though the asset transfer is successful.  
- **Inconsistent behavior:** The function works as expected for native token deposits but fails for non-native tokens, leading to confusion and potentially frustrated users.  
- **Loss of functionality:** Users who deposit non-native tokens will see no increase in their virtual token balance, despite the assets being properly transferred to the contract.  
- **Security risk:** This inconsistency could be exploited by malicious users to bypass the minting of tokens, undermining the system’s intended behavior.  

## Proof of Concept  

To illustrate the issue, the following test case demonstrates that no tokens are minted when a non-native token is used for deposit. Add this code to the `GeneralTest.t.sol` file:  

```solidity
function test_cashin_with_non_native_token_fails_for_whitelisted_user() public {
    // Step 1: Set up mock token and configure vETH
    MockToken nonNativeToken = new MockToken("MockToken", "MKT");
    nonNativeToken.mint(address(this), DEPOSIT_AMOUNT); // Mint tokens to test contract

    vm.startPrank(multiSigAdmin);
    vETH = new VirtualToken("vETH", "vETH", address(nonNativeToken));
    vm.stopPrank();

    nonNativeToken.approve(address(vETH), DEPOSIT_AMOUNT); // Approve vETH for transfer

    // Step 2: Whitelist the user
    address whitelistedUser = address(this);
    vm.prank(multiSigAdmin); // Simulate owner action
    vETH.addToWhiteList(whitelistedUser);

    vm.prank(whitelistedUser);
    vETH.cashIn(DEPOSIT_AMOUNT);

    // Assert no tokens minted
    uint256 mintedBalance = vETH.balanceOf(whitelistedUser);
    assertEq(mintedBalance, 0, "No tokens should be minted for the user");
}
```  

Additionally, in the `BaseTest.t.sol` file, create a `dummyToken` and assign it as the address of the `underlyingToken`:  

```solidity
address dummyToken = makeAddr("dummyToken");

vm.startPrank(multiSigAdmin);
vETH = new VirtualToken("vETH", "vETH", dummyToken);
```  

In this test, when the `cashIn` function is called with a non-native token, no virtual tokens are minted for the user because the minting logic incorrectly uses `msg.value`, which is zero for non-native tokens.  

## Recommended Mitigation Steps  

The bug can be resolved by modifying the minting logic to correctly handle both native and non-native token deposits. Specifically, the minting should use the `amount` parameter passed to the `cashIn` function, instead of `msg.value`. This ensures that the correct number of virtual tokens is minted for both native and non-native token deposits.  

The corrected function should look like this:  

```solidity
function cashIn(uint256 amount) external payable onlyWhiteListed {
    if (underlyingToken == LaunchPadUtils.NATIVE_TOKEN) {
        require(msg.value == amount, "Invalid ETH amount");
    } else {
        _transferAssetFromUser(amount);
    }
    _mint(msg.sender, amount); // Use the 'amount' parameter instead of msg.value
    emit CashIn(msg.sender, amount);
}
```  

By using `amount` instead of `msg.value`, the function will mint virtual tokens correctly for both native and non-native token deposits, ensuring consistent behavior and proper functionality.  

## Links to Affected Code  

[VirtualToken.sol#L72-L80](https://github.com/code-423n4/2024-12-lambowin/blob/874fafc7b27042c59bdd765073f5e412a3b79192/src/VirtualToken.sol#L72-L80)  
