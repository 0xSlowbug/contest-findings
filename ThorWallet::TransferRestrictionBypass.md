# Bridged TITN Tokens Enable Indirect Trading Between Users Despite Transfer Restrictions

**Severity: High**

---

## 🧩 Issue Summary

The **Bridged TITN Tokens** are designed to restrict transfers — only allowing them to a predefined contract (`transferAllowedContract`) or the **LayerZero endpoint** (`lzEndpoint`) — until an admin explicitly unlocks transfers.

**However,** users can bypass these restrictions and trade TITN tokens **indirectly** by leveraging cross-chain bridging.

---

## 📄 Intended Restrictions (Per Documentation)

- Users **cannot** transfer TITN tokens to arbitrary addresses on the **same chain** unless explicitly enabled.
- Transfers are only allowed to:
  - `transferAllowedContract` (e.g., a staking contract).
  - `lzEndpoint` (for cross-chain bridging).
- Goal: Prevent trading **until** `isBridgedTokensTransferLocked` is disabled.

---

## 🔁 Bypassing the Restriction: Bridging as an Indirect Transfer

Despite the above restrictions, users can still trade TITN tokens using the following method:

1. **User A bridges** TITN tokens to **User B’s address** on another chain (e.g., Arbitrum → Base).
2. `_credit` function **mints** tokens to **User B** on Base.
3. **User B bridges** the tokens **back to Arbitrum** to their own address.
4. `_credit` again **mints** to User B on Arbitrum.

➡️ **Net Result**: User A has effectively transferred tokens to User B, **bypassing same-chain transfer restrictions**.

### ⚠️ Consequence

- Malicious users can **accumulate tokens cheaply** by setting low OTC prices.
- Once transfers are globally unlocked and liquidity pools (e.g., Uniswap) are enabled, they can **dump large volumes**, crashing the price.

---

## 🕵️ Root Cause

- `_validateTransfer()` correctly blocks same-chain transfers.
- `_credit()` does **not verify** that the recipient is `msg.sender`.
- Users can mint to **arbitrary recipients** on another chain and bridge back.

---

## ✅ Proof of Concept (PoC)

```js
it.only('should let a user bridge TITN to BASE and back to ARBITRUM to another user', async function () {
  // User1 gets ARB.TITN
  await tgt.connect(user1).approve(mergeTgt.address, ethers.utils.parseUnits('100', 18));
  await tgt.connect(user1).transferAndCall(mergeTgt.address, ethers.utils.parseUnits('100', 18), '0x');

  const claimableAmount = await mergeTgt.claimableTitnPerUser(user1.address);
  await mergeTgt.connect(user1).claimTitn(claimableAmount);

  const initialArbBalanceUser1 = await arbTITN.balanceOf(user1.address);
  const initialBaseBalanceUser2 = await baseTITN.balanceOf(user2.address);

  const options = Options.newOptions().addExecutorLzReceiveOption(200000, 0).toHex().toString();
  const sendParam = [
    eidA,
    ethers.utils.zeroPad(user2.address, 32),
    claimableAmount.toString(),
    claimableAmount.toString(),
    options,
    '0x',
    '0x',
  ];
  const [nativeFee] = await arbTITN.quoteSend(sendParam, false);
  await arbTITN.connect(user1).send(sendParam, [nativeFee, 0], user1.address, { value: nativeFee });

  const balanceArbUser1After = await arbTITN.balanceOf(user1.address);
  const balanceBaseUser2After = await baseTITN.balanceOf(user2.address);
  expect(balanceArbUser1After).to.be.eql(ethers.utils.parseUnits('0', 18));
  expect(balanceBaseUser2After).to.be.eql(claimableAmount);

  const sendParamBack = [
    eidB,
    ethers.utils.zeroPad(user2.address, 32),
    claimableAmount.toString(),
    claimableAmount.toString(),
    options,
    '0x',
    '0x',
  ];
  const [nativeFeeBack] = await baseTITN.quoteSend(sendParamBack, false);
  await baseTITN.connect(user2).send(sendParamBack, [nativeFeeBack, 0], user2.address, { value: nativeFeeBack });

  const finalBaseBalanceUser2 = await baseTITN.balanceOf(user2.address);
  const finalArbBalanceUser2 = await arbTITN.balanceOf(user2.address);
  expect(finalBaseBalanceUser2).to.be.eql(ethers.utils.parseUnits('0', 18));
  expect(finalArbBalanceUser2).to.be.eql(claimableAmount);
});
```

---

## 🛠 Recommendation

### 🔒 Enforce Self-Bridging in `_credit()` Function

Update the LayerZero receiver logic to **restrict bridging to self** only:

```solidity
function _credit(
    address _to,
    uint256 _amountLD,
    uint32 /*_srcEid*/
) internal virtual override returns (uint256 amountReceivedLD) {
    require(_to == msg.sender, "Bridging to another user is not allowed");

    if (_to == address(0x0)) _to = address(0xdead); // _mint(...) does not support address(0x0)
    _mint(_to, _amountLD);

    if (!isBridgedTokenHolder[_to]) {
        isBridgedTokenHolder[_to] = true;
    }

    return _amountLD;
}
```

This change ensures that **users can only bridge TITN tokens to themselves**, closing the indirect transfer loophole.


- [`Titn.sol` Lines 96–112](#) (https://github.com/code-423n4/2025-02-thorwallet/blob/98d7e936518ebd80e2029d782ffe763a3732a792/contracts/Titn.sol#L96-L112)

