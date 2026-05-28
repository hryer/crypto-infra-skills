# EIP-7702 — EOA Migration to Smart Accounts

## What 7702 does

EIP-7702 (live since Pectra, May 2025) lets an EOA *temporarily delegate* its code to a smart contract. The EOA address stays the same, but transactions sent from it execute the delegated contract's logic.

In plain terms: **your existing wallet address can now behave like a smart account, without migrating funds.**

## The authorization model

A 7702 transaction contains an `authorization_list`:

```
{
  chain_id,
  address,        // contract whose code the EOA delegates to
  nonce,
  y_parity, r, s  // signature from the EOA owner
}
```

The EOA owner signs this authorization. When included in a transaction (which can be sent by anyone), the EVM:
1. Recovers the signer from the auth signature
2. Sets the signer's code to `0xef0100 ++ address` (delegation indicator)
3. Subsequent calls to the EOA execute the delegated contract

## Critical pitfalls

### 1. Authorization is per-chain
The `chain_id` field is mandatory. An authorization signed for Ethereum mainnet does NOT work on Base.

You must re-authorize on every chain. This breaks the "one wallet, all chains" mental model.

### 2. Delegation can be replaced
The owner can sign a new authorization pointing at a different implementation. Your wallet's security entirely depends on what implementation you delegate to.

A malicious dApp could:
- Trick the user into signing an authorization pointing at a malicious implementation
- That implementation drains the wallet on the next transaction

**Mitigation:** wallets should display the delegation target clearly. Users should never sign 7702 authorizations from untrusted sources.

### 3. Delegation is sticky across sessions
Once delegated, the EOA stays delegated until:
- Owner signs a new authorization (changing target)
- Owner delegates to the zero address (clearing delegation)

There's no auto-expiry. A one-time delegation persists indefinitely.

### 4. Storage layout matters
Different smart-account implementations use different storage slots. Switching implementations without storage migration can corrupt state.

**Rule:** don't switch between implementations casually. Pick one and stick with it.

### 5. Reentrancy attack surface
The EOA can now be called by other contracts (since it has code). Implementations must guard against reentrancy in ways EOAs never had to.

### 6. Combining 7702 + 4337
You can delegate a 7702 EOA to a 4337-compatible account implementation, giving the EOA full smart-account features including UserOp validation.

**Subtlety:** the EOA can ALSO send normal transactions directly (bypassing the bundler). Your 4337 validator must handle this. ERC-7562 storage rules still apply to the validation phase.

## Migration patterns

### Pattern A: Opt-in upgrade (recommended for consumer wallets)

```
1. User clicks "Enable smart account features"
2. Wallet shows what the upgrade enables (gasless, batching, session keys)
3. Wallet shows the implementation contract address + audit info
4. User signs the authorization
5. First "upgraded" tx is sent (often a batch of setup ops)
```

### Pattern B: Lazy upgrade (recommended for power users)

```
1. User does a normal action (e.g., swap)
2. Wallet detects this action would benefit from batching/gasless
3. Wallet bundles the 7702 authorization INTO the action tx
4. One signature, action completes, EOA is now smart-account-enabled for future txs
```

### Pattern C: Mandatory upgrade (only for closed-ecosystem custodial wallets)

```
1. Backend signs authorization on behalf of custodial EOAs at issuance
2. All EOAs start life as smart accounts
3. No migration needed because all EOAs were "born" delegated
```

This only works when you control the EOA's keys (custodial).

## What 7702 does NOT solve

- Doesn't help on non-EVM chains
- Doesn't help on pre-Pectra chains (still many L2s)
- Doesn't replace ERC-4337 — they're complementary
- Doesn't give you sponsored gas by itself (you still need a paymaster / sponsor mechanism)
- Doesn't make EOAs secure (delegating to a buggy implementation = buggy wallet)

## Compatibility matrix

| Use case | EOA | EOA + 7702 | ERC-4337 SCA |
|---|---|---|---|
| Existing address | ✓ | ✓ | ✗ (new address) |
| Gasless txs | ✗ | ✓ (with paymaster) | ✓ |
| Batching | ✗ | ✓ | ✓ |
| Social recovery | ✗ | ✓ (impl-dependent) | ✓ |
| Session keys | ✗ | ✓ (impl-dependent) | ✓ |
| Works on Solana | ✗ | ✗ | ✗ |
| Works pre-Pectra | ✓ | ✗ | ✓ |

**Strategic implication:** for new products targeting EVM mainnet + major L2s in 2025+, 7702 + 4337 hybrid is the path. For multi-chain (Solana, Bitcoin, etc.), 4337 alone doesn't help and you need different infra per chain.
