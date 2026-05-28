# Flashbots Bundle Submission — Reference

## What a bundle is

A bundle is a sequence of transactions that block builders include atomically (all-or-nothing) at a specific block. You pay the builder via:
- Priority fee on the txs, OR
- A direct ETH transfer to `block.coinbase` inside the bundle (the "bribe")

If the bundle doesn't land, you pay nothing.

## Relay landscape (as of 2025-26)

| Relay | Notes |
|---|---|
| **Flashbots** | Original, most-used, broad builder support |
| **BloXroute** | Multi-tier (max-profit, ethical, regulated) |
| **Eden** | Strong on validator coverage |
| **Manifold** | Smaller, sometimes better landing rates |
| **Builder0x69, rsync-builder, beaverbuild, Titan** | Direct-to-builder, skip relay |

**Strategy:** submit to multiple relays in parallel. Same bundle, same builder ends up with it eventually.

## Bundle JSON-RPC

The main methods:

```
eth_sendBundle           # submit
eth_callBundle           # simulate (read-only)
eth_sendPrivateTransaction  # single tx, private
flashbots_getBundleStats # check landing status
```

Submission requires X-Flashbots-Signature header (signed by your reputation key, NOT your transaction signing key).

## Bundle structure

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "eth_sendBundle",
  "params": [{
    "txs": [
      "0xf86b...",  // signed RLP of tx 1
      "0xf86b..."   // signed RLP of tx 2
    ],
    "blockNumber": "0x1234abc",
    "minTimestamp": 0,
    "maxTimestamp": 0,
    "revertingTxHashes": []
  }]
}
```

Key fields:
- **`txs`** — signed transactions in RLP hex. Order matters; they execute in this order.
- **`blockNumber`** — target block. Must be future. Bundles for past blocks are rejected.
- **`revertingTxHashes`** — txs allowed to revert without killing the bundle. Useful for "try this, if it works great; if not, don't kill my arb tx."

## Go example (using go-ethereum + custom flashbots client)

```go
package mev

import (
    "context"
    "crypto/ecdsa"
    "encoding/hex"
    "encoding/json"
    "fmt"
    "math/big"
    "net/http"
    "strings"
    "time"

    "github.com/ethereum/go-ethereum/common"
    "github.com/ethereum/go-ethereum/core/types"
    "github.com/ethereum/go-ethereum/crypto"
    "github.com/ethereum/go-ethereum/rlp"
)

type BundleParams struct {
    Txs               []string `json:"txs"`
    BlockNumber       string   `json:"blockNumber"`
    RevertingTxHashes []string `json:"revertingTxHashes,omitempty"`
}

type BundleRequest struct {
    Jsonrpc string          `json:"jsonrpc"`
    ID      int             `json:"id"`
    Method  string          `json:"method"`
    Params  []BundleParams  `json:"params"`
}

type FlashbotsClient struct {
    relayURL   string
    reputationKey *ecdsa.PrivateKey
    httpClient *http.Client
}

func NewClient(relayURL string, reputationKey *ecdsa.PrivateKey) *FlashbotsClient {
    return &FlashbotsClient{
        relayURL:      relayURL,
        reputationKey: reputationKey,
        httpClient:    &http.Client{Timeout: 5 * time.Second},
    }
}

// SubmitBundle sends a bundle of signed txs targeting blockNumber.
func (c *FlashbotsClient) SubmitBundle(
    ctx context.Context,
    signedTxs []*types.Transaction,
    blockNumber uint64,
) error {
    // 1. Encode txs as hex RLP
    rawTxs := make([]string, len(signedTxs))
    for i, tx := range signedTxs {
        b, err := rlp.EncodeToBytes(tx)
        if err != nil {
            return fmt.Errorf("rlp encode tx %d: %w", i, err)
        }
        rawTxs[i] = "0x" + hex.EncodeToString(b)
    }

    // 2. Build the JSON-RPC request
    req := BundleRequest{
        Jsonrpc: "2.0",
        ID:      1,
        Method:  "eth_sendBundle",
        Params: []BundleParams{{
            Txs:         rawTxs,
            BlockNumber: fmt.Sprintf("0x%x", blockNumber),
        }},
    }

    body, err := json.Marshal(req)
    if err != nil {
        return err
    }

    // 3. Sign the body with reputation key (NOT tx signing key)
    hash := crypto.Keccak256Hash([]byte(fmt.Sprintf("0x%s", hex.EncodeToString(crypto.Keccak256(body)))))
    sig, err := crypto.Sign(hash.Bytes(), c.reputationKey)
    if err != nil {
        return err
    }
    sig[64] += 27  // Flashbots expects v in {27, 28}

    senderAddr := crypto.PubkeyToAddress(c.reputationKey.PublicKey).Hex()
    signature := fmt.Sprintf("%s:0x%s", senderAddr, hex.EncodeToString(sig))

    // 4. POST
    httpReq, err := http.NewRequestWithContext(ctx, "POST", c.relayURL, strings.NewReader(string(body)))
    if err != nil {
        return err
    }
    httpReq.Header.Set("Content-Type", "application/json")
    httpReq.Header.Set("X-Flashbots-Signature", signature)

    resp, err := c.httpClient.Do(httpReq)
    if err != nil {
        return fmt.Errorf("post: %w", err)
    }
    defer resp.Body.Close()

    if resp.StatusCode != 200 {
        return fmt.Errorf("relay returned %d", resp.StatusCode)
    }

    return nil
}

// PayCoinbase builds a tx that transfers ETH to block.coinbase as a bribe.
func PayCoinbase(
    chainID *big.Int,
    nonce uint64,
    gasLimit uint64,
    gasFeeCap *big.Int,
    gasTipCap *big.Int,
    bribeAmount *big.Int,
    fromKey *ecdsa.PrivateKey,
) (*types.Transaction, error) {
    // This requires a contract that does `block.coinbase.transfer(amount)`.
    // For pure-EOA bribing, you set high priority fee instead.
    // (Contract approach is more flexible because you only bribe on success.)
    panic("implement using a deployed bribe contract")
}
```

## Bribe sizing heuristics

```
bribe = max(
    min_profit_share,        // e.g., 50% of expected profit
    competitor_threshold,    // observed winning bribes for similar ops
    floor                    // e.g., 0.0001 ETH minimum
)
```

Track historical block winners to learn `competitor_threshold` per opportunity type.

## Common mistakes

1. **Using the same key for reputation and signing.** Don't. Reputation key = your identity to the relay. Signing key = controls funds. Separate them.
2. **Submitting to a single relay.** Always submit to ≥3 in parallel.
3. **Submitting only for `blockNumber = next`.** Also target `next+1` and `next+2` for opportunities that don't expire instantly.
4. **Forgetting the simulation step.** Always `eth_callBundle` before `eth_sendBundle`. Catches reverts cheaply.
5. **Bribing too much.** Profit goes to the builder, not you. Bribe just enough to win.
6. **Bribing too little.** No inclusion, no profit. Track win rate.
7. **Not handling reorgs.** A bundle included in block N can be reorged out. Track and recompute P&L.

## Monitoring metrics

- Bundles submitted per minute
- Bundles included per minute
- Win rate (included / submitted)
- Average bribe per bundle
- Average gross profit per included bundle
- Net profit (gross - bribe - gas - infra cost)
- Reorg-killed bundles
- Simulation success rate vs on-chain success rate (should be ~99%)
