# Molt EVM onlySpawnerToken Modifier Bypass — PoC

> **Educational Purpose Only** — This PoC is created for security research and education
> purposes only. It is a simplified simulation, not a fork replay against mainnet.

## Overview
- **Date:** 2026-03-07
- **Loss:** ~$127,000
- **Chain:** Base
- **Category:** Access Control / Protocol Logic
- **Technique:** onlySpawnerToken Modifier Bypass
- **Attacker:** [0xEB179B0179836c6B634056db60855234D6aF3338](https://basescan.org/address/0xEB179B0179836c6B634056db60855234D6aF3338)
- **First Exploit TX:** [0xdc7792...4fbf6](https://basescan.org/tx/0xdc7792c6f0372112190616e11dd733d17f5097f13b544552c83118b12ca4fbf6)
- **Swap/Drain TX:** [0x3c4fbf...21a9](https://basescan.org/tx/0x3c4fbfb650f5316456a60ad003ec7237aac0dc4c95c54af459684848dd9821a9)
- **Exploit Blocks:** 43084564–43084603 (mints), 43084941 (swap)
- **Token:** [0x225dA3D879D379FF6510C1CC27Ac8535353f501F](https://basescan.org/token/0x225da3d879d379ff6510c1cc27ac8535353f501f)

## Quick Start

```bash
git clone https://github.com/NomosLabs-Security/poc-molt-evm-2026
cd poc-molt-evm-2026
forge install foundry-rs/forge-std --no-git
forge test -vvvv
```

> No OpenZeppelin dependency needed -- ERC20 is inlined for zero-dependency setup.

## Vulnerability

The Molt EVM (mEVM) token contract on Base contained a critical access control flaw in
the `onlySpawnerToken` modifier. This modifier was intended to restrict sensitive token
minting/spawning operations to only authorized spawner contracts. However, the modifier's
validation logic could be bypassed, allowing an attacker to call protected functions
directly.

### Attack Flow

1. **Deploy Exploit Contracts (Blocks 43084564–43084603):** The attacker deployed 40
   separate exploit contracts in consecutive blocks over ~80 seconds.
2. **triggerMint:** Each contract called `triggerMint`, which internally overwrote the
   `spawnerToken` address via the unprotected `setSpawnerToken()` function, then minted
   ~787 quadrillion mEVM tokens per call.
3. **Accumulate:** All minted tokens (~31.5 quintillion mEVM) were sent to the attacker
   address `0xEB179B...3338`.
4. **Swap/Drain (Block 43084941):** The attacker swapped the minted mEVM for WETH via
   the Aerodrome mEVM/WETH liquidity pool, extracting ~$127K.

### Root Cause

The vulnerability stems from a common access control anti-pattern: the `onlySpawnerToken`
modifier correctly checked the caller identity, but the function to **set** the spawner
token address was either unprotected or had insufficient access control. This allowed
an attacker to modify the trusted address and bypass the modifier entirely.

```solidity
// Vulnerable pattern
modifier onlySpawnerToken() {
    require(msg.sender == spawnerToken, "Not spawner");  // Check is correct
    _;
}

// But the setter has no access control!
function setSpawnerToken(address _spawner) external {
    spawnerToken = _spawner;  // Anyone can call this
}
```

## Expected Output

Running `forge test -vvvv` demonstrates:
- How the unprotected setter allows an attacker to become the spawner
- How the attacker mints tokens using the compromised modifier
- How the attacker drains value from the liquidity pool
- How the fixed version (with proper `onlyOwner` on the setter) prevents the attack

## References

- [DefiLlama Hacks Database](https://defillama.com/hacks)
- [DefimonAlerts on X](https://x.com/DefimonAlerts/status/2030273132871741800)
- [First Exploit TX on BaseScan](https://basescan.org/tx/0xdc7792c6f0372112190616e11dd733d17f5097f13b544552c83118b12ca4fbf6)
- [Swap/Drain TX on BaseScan](https://basescan.org/tx/0x3c4fbfb650f5316456a60ad003ec7237aac0dc4c95c54af459684848dd9821a9)
- [Attacker Address on BaseScan](https://basescan.org/address/0xEB179B0179836c6B634056db60855234D6aF3338)
- [mEVM Token on BaseScan](https://basescan.org/token/0x225da3d879d379ff6510c1cc27ac8535353f501f)
- [Molt EVM on DEX Screener](https://dexscreener.com/base/0x064c9fbed2cce0fdc3600777492bc0413b2cf95e)

## Full Analysis

- [NomosLabs Security Intelligence Archive](https://nomoslabs.io/archive/molt-evm-2026)

## License

MIT — For educational use only.
