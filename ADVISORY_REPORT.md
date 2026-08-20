# Summary

The Nockchain bridge contracts use a single-owner UUPS upgrade pattern with no supply cap, allowing full bridge drain if the owner key is compromised.

# Vulnerability Details

## Root Cause

MessageInbox.sol uses `_authorizeUpgrade` with only `onlyOwner` modifier. The owner is a single EOA address (0x4356D2C1B6f7238dcEE0a6E90c9B5B1717648555), not a multisig or timelock. Nock.sol implements `cap()` returning 0, meaning no supply cap exists.

## Impact

If the owner private key is compromised:
1. Attacker calls `upgradeTo(maliciousImpl)` on MessageInbox
2. Malicious implementation mints unlimited tokens via `nock.mint()`
3. cap()=0 means no supply limit
4. Complete drain of all bridge tokens

## Affected Code

```solidity
// MessageInbox.sol
function _authorizeUpgrade(address newImplementation) internal override onlyOwner {}
function upgradeTo(address newImplementation) public onlyOwner { ... }

// Nock.sol  
function cap() public pure returns (uint256) { return 0; }
```

## Steps to Reproduce

1. Clone the repository and install Foundry
2. Deploy NockToken and MessageInbox on Base Sepolia testnet
3. Deposit tokens to simulate bridge activity
4. Deploy MaliciousImplementation
5. Call `upgradeTo(maliciousImpl)` as owner
6. Call `nock.mint(attacker, unlimited)` through malicious implementation
7. Verify unlimited minting occurred

```bash
# Verification on production contracts (Base Sepolia)
cast call 0xA9cd4087D9B050D8B35727AAf810296CA957c7B3 "cap()(uint256)" --rpc-url https://sepolia.base.org
# Returns: 0 (no supply cap)

cast call 0xA9cd4087D9B050D8B35727AAf810296CA957c7B3 "owner()(address)" --rpc-url https://sepolia.base.org  
# Returns: 0x4356D2C1B6f7238dcEE0a6E90c9B5B1717648555 (single EOA)
```

## Suggested Fix

1. Add OpenZeppelin TimelockController with 48h delay for upgrades
2. Use Gnosis Safe multisig for owner operations
3. Set a reasonable cap() value matching Nockchain supply model
