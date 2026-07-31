# bankrstonks-ledger

Public, unauthenticated mirror of the BankrStonks keeper ledger.

This repo exists so that bankrstonks.fun (and anyone else) can read the platform's
ledger state over plain HTTPS — no API key, no payment, no wallet.

Every push is a commit, so the git history is a timestamped, diffable,
tamper-evident record of the ledger. Entries cannot be silently edited.

## Endpoint

```
https://raw.githubusercontent.com/BankrStonks/bankrstonks-ledger/main/ledger.json
```

- Plain HTTPS, CORS-open (`Access-Control-Allow-Origin: *`), CDN-cached by GitHub
- GitHub's raw CDN caches for ~5 minutes, so a fresh commit takes a few minutes to propagate
- To bypass the cache during development, append a cache-buster: `?t=${Date.now()}`

## Update cadence

Written by the BankrStonks keeper:

- Daily at **06:15 UTC**, after the execution leg runs and the ecosystem snapshot refreshes
- Plus any manually triggered keeper run

This is a **snapshot**, not a live feed. Use `ledger.json` for ledger state
(reserve lots, cost basis, fee totals, burns, action log) and fetch live token
prices client-side from the on-chain pools.

## Schema

Top level:

| Field | Type | Description |
|---|---|---|
| `schemaVersion` | number | Bumped on any breaking shape change. Currently `1`. |
| `generatedAt` | ISO 8601 string | When this snapshot was produced. |
| `source` | string | Producing system. |
| `chain` | string | `robinhood` (Robinhood Chain). |
| `explorer` | string | Explorer base URL. |
| `updateCadence` | string | Human-readable cadence. |
| `summary` | object | Headline stats. |
| `control` | object | `{ paused, pausedAt }` — global keeper pause switch. |
| `config` | object | Hold days, keeper cadence, max slippage, fee splits. |
| `contracts` | object | Every address: treasury, keeper, burn address, $BNKRS, pool id, staking vault, stock tokens. |
| `vault` | object | Staking vault address + on-chain totals. |
| `bnkrsMarket` | object | $BNKRS market data from its WETH pool. |
| `coins` | array | One entry per registered coin (9 total, incl. $BNKRS). |
| `reserveLots` | array | Open and closed 30-day stock reserve lots. |
| `burns` | array | Burn events. |
| `poolInflows` | array | $BNKRS bought for the staking pool. |
| `alerts` | array | Open keeper alerts. Empty = healthy. |
| `trackedStocks` | array | Live tokenized-stock prices from their on-chain pools. |
| `branding` | object | `{ logoUrl, bannerUrl }`. |
| `actionLog` | array | Keeper action log, newest first, with tx hashes. |

### `summary`

```
totalValueReservedUsd    number   USD value of all open reserve lots
totalFeesCollectedUsd    number   cumulative claimed creator fees
totalBNKRSStaked         number   $BNKRS staked in the vault
totalCoinsBurntUsd       number   cumulative burn value
totalCoinsBurntUnits     number   cumulative burned coin units
rewardPoolUsd            number   $BNKRS earmarked/held for stakers
openLots                 number
dueForBurnLots           number
reservePricing           string   "on-chain stock pools"
apyTrailing30dPct        number|null   null until real distributions exist
updatedAt                ISO 8601 string
```

### `coins[]`

```
symbol                string    e.g. "bNVDA"
name                  string
isBnkrs               boolean   true only for BNKRS
tokenAddress          string    coin contract on Robinhood Chain
pairedStock           string    e.g. "NVDA" ("WETH" for BNKRS)
stockTokenAddress     string|null
stockPriceUsd         number
stockChange24Pct      number|null
stockPriceSource      string|null   "onchain"
priceUsd              number
change24Pct           number|null
mcapUsd               number
liquidityUsd          number
vol24Usd              number
totalSupply           number
burnedUnits           number    on-chain balance at the dead address
burnedPctOfSupply     number
reserveStockAmount    number
reserveUsd            number
totalFeeUsd           number
totalBurntUsd         number
lotCount              number
createdAt             ISO 8601 string
```

### `actionLog[]`

```
timestamp    ISO 8601 string
action       string    KEEPER_HEARTBEAT | COIN_REGISTERED | QUEUE_UPDATED |
                       FEE_CLAIM | FEE_SELL_TO_WETH | STOCK_BUY | LOT_OPENED |
                       LOT_MATURED | STOCK_SELL | COIN_BUYBACK | COIN_BURN |
                       BNKRS_POOL_BUY | VAULT_FUND | STEP_RETRY | STEP_FAILING
coinSymbol   string|null
amountUsd    number
details      string
txHash       string|null   link: https://robinhoodchain.blockscout.com/tx/<txHash>
```

### `reserveLots[]`

Empty until per-coin fees clear the $1 dust threshold and the keeper opens the
first lot. Each lot carries the coin, stock, amount, USD cost basis, open date,
maturity date, status, and buy/sell/burn tx hashes.

## Empty state

At the time of the first snapshot the platform is live but pre-volume: all 9
tokens are deployed at 100B supply, nothing is burned, no reserve lots are open,
and the staking vault holds zero rewards. Render empty states — do not
substitute placeholder numbers.

## Example

```js
const res = await fetch(
  'https://raw.githubusercontent.com/BankrStonks/bankrstonks-ledger/main/ledger.json'
);
const ledger = await res.json();

console.log(ledger.summary.totalValueReservedUsd);
console.log(ledger.coins.filter(c => !c.isBnkrs));
console.log(ledger.actionLog.slice(0, 10));
```

## Contracts

| What | Address |
|---|---|
| $BNKRS | `0x1504ED99c6369f8715449a3BcD2Dc5d5e37f6bA3` |
| Staking vault (verified) | `0x6643e356036EddF3661f063c2F40a85073Db9E25` |
| Treasury / fee recipient | `0xda692f9a6ef58b9a78a3fcbdf60efbae4ee7584c` |
| Burn address | `0x000000000000000000000000000000000000dEaD` |

All on Robinhood Chain. Explorer: https://robinhoodchain.blockscout.com
