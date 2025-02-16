
## Unrestricted Access to Whitelist Functionality Allows Unauthorized Private Listing Purchases  
**Medium**  

## Finding Description and Impact  

## Root Cause  
The `whitelistAddress()` function in the smart contract allows any user to self-add themselves to the whitelist without any additional authorization or approval mechanism. This fundamentally breaks the intended access control for private listings.  

## Technical Details  
In the provided code snippet:  

```solidity
function whitelistAddress() external {
    require(totalWhitelist < maxWhitelist, "SS_Whitelist: Reached whitelist limit"); 
    require(userSettings[msg.sender] == false, "SS_Whitelist: User is whitelisted"); 

    userSettings[msg.sender] = true;
    totalWhitelist++;
    emit WhitelistedAddress(totalWhitelist, msg.sender);
}
```
### Key vulnerabilities:  
- No owner/administrator approval required  
- Any address can self-whitelist  
- No additional verification or permission checks  

## Potential Impact  
- Compromises the privacy and exclusivity of private listings  
- Allows unauthorized users to bypass intended access restrictions  
- Reduces the effectiveness of the whitelist mechanism  
- Potential financial risks if private sales are meant to be strictly controlled  

## Proof of Concept  
The provided test case demonstrates the vulnerability:  
- User2 can independently add themselves to the whitelist  
- No approval from the lot owner or administrator is required  
- The self-whitelist function succeeds without additional authorization  

## Proof of Concept Walkthrough  
1. Create a private listing with a whitelist  
2. Demonstrate that any user can add themselves to the whitelist  
3. Validate purchase by the self-added user  
4. Confirm bypassing of intended access controls  

## Code  
Paste the following code into the `Marketplace.test.ts` file:  

```typescript
it.only("allows unauthorized self-whitelisting and purchase of private listings", async function () {
    const { marketplace, token, user1, user2 } = await loadFixture(deployProxyFixture);

    const amount = parseEther("100");
    const cost = parseEther("75");
    const maxWhitelist = 1;
    const privateListing = true;

    // User1 creates the private listing
    await token.write.approve([marketplace.address, parseEther("1000")], { account: user1.account });
    const tx = await marketplace.write.listVesting(
        [vesting.address, amount, cost, BigInt(10), 0, 2, BigInt(maxWhitelist), token.address, parseEther("1"), privateListing],
        { account: user1.account }
    );

    const receipt = await hre.viem.getPublicClient().waitForTransactionReceipt({ hash: tx });
    const whitelistEvent = receipt.logs.find(log => log.topics[0] === keccak256(toHex("WhitelistCreated(...)")));
    const whitelistAddress = decodeEventLog({ abi: marketplace.abi, data: whitelistEvent.data, topics: whitelistEvent.topics }).args.whitelistAddress;

    const whitelistContract = await hre.viem.getContractAt("SS_Whitelist", whitelistAddress);

    // Unauthorized user2 adds themselves to the whitelist
    await whitelistContract.write.whitelistAddress({ account: user2.account });

    // Validate that user2 is now whitelisted
    expect(await whitelistContract.read.validateAddress([user2.account.address], { account: user2.account })).to.equal(true);

    // User2 purchases the private listing
    await token.write.mint([user2.account.address, parseEther("100000")]);
    await token.write.approve([marketplace.address, parseEther("100000")], { account: user2.account });
    await marketplace.write.spotPurchase([vesting.address, BigInt(0), parseEther("100"), whitelistContract.address], { account: user2.account });
});
```

## Recommended Mitigation Steps  

### Option 1: Owner-Based Approval  

```solidity
function whitelistAddress(address _userToWhitelist) external onlyOwner {
    require(totalWhitelist < maxWhitelist, "SS_Whitelist: Reached whitelist limit"); 
    require(userSettings[_userToWhitelist] == false, "SS_Whitelist: User is already whitelisted"); 

    userSettings[_userToWhitelist] = true;
    totalWhitelist++;
    emit WhitelistedAddress(totalWhitelist, _userToWhitelist);
}
```

### Option 2: Lot Owner Approval  

```solidity
function whitelistAddress(address _userToWhitelist, address _lotOwner) external {
    require(msg.sender == _lotOwner, "SS_Whitelist: Only lot owner can whitelist");
    require(totalWhitelist < maxWhitelist, "SS_Whitelist: Reached whitelist limit"); 
    require(userSettings[_userToWhitelist] == false, "SS_Whitelist: User is already whitelisted"); 

    userSettings[_userToWhitelist] = true;
    totalWhitelist++;
    emit WhitelistedAddress(totalWhitelist, _userToWhitelist);
}
```

## Links to Affected Code  
[SecondSwap_Whitelist.sol#L52-L65](https://github.com/code-423n4/2024-12-secondswap/blob/214849c3517eb26b31fe194bceae65cb0f52d2c0/contracts/SecondSwap_Whitelist.sol#L52-L65)  

---
