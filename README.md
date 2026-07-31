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
- Served as `text/plain`. `res.json()` parses it fine — do **not** gate on the content-type header
- GitHub's raw CDN caches for ~5 minutes; append `?t=${Date.now()}` to bypass during development

## Null policy (schemaVersion 3+)

**`null` means the value could not be computed for this snapshot. `0` always means a
real, measured zero.**

Render `null` as an em dash. Never coerce it to `0`. Every numeric field that depends
on an external read (market data, chain reads, vault reads) is nulled when that read
fails rather than defaulting to zero.

Check `dataQuality` before trusting a partially-populated snapshot:

```
vaultReadOk             boolean   vaultStats() read succeeded
bnkrsPriceOk            boolean   $BNKRS price resolved from its pool
coinsWithMarketData     number    coins with a resolved pool price
coinsTotal              number
coinsMissingMarket      string[]  symbols whose price could not be read
coinsMissingChainData   string[]  symbols whose supply/burn read failed
stocksPricedOnchain     number    of 8 tokenized stocks
marketFetchErrors       string[]  empty when all market reads succeeded
```

Each `coins[]` entry also carries `dataOk: { market, chain }` for per-coin gating.

## Update cadence

Written by the BankrStonks keeper:

- Daily at **06:15 UTC**, after the execution leg runs and the ecosystem snapshot refreshes
- Plus any manually triggered keeper run

This is a **snapshot**, not a live feed. Use `ledger.json` for ledger state
(reserve lots, cost basis, fee totals, burns, action log, vault stats) and fetch live
token prices client-side from the on-chain pools.

## Schema

Top level:

| Field | Type | Description |
|---|---|---|
| `schemaVersion` | number | Bumped on any breaking shape change. Currently `3`. |
| `generatedAt` | ISO 8601 string | When this snapshot was produced. |
| `source` | string | Producing system. |
| `chain` | string | `robinhood` (Robinhood Chain). |
| `explorer` | string | Explorer base URL. |
| `website` | string | bankrstonks.fun |
| `updateCadence` | string | Human-readable cadence. |
| `nullPolicy` | string | Machine-readable restatement of the null rule above. |
| `dataQuality` | object | Per-snapshot read health. See above. |
| `summary` | object | Headline stats. |
| `control` | object | `{ paused, pausedAt }` — global keeper pause switch. |
| `config` | object | Hold days, keeper cadence, max slippage, fee splits. |
| `contracts` | object | Every address: treasury, keeper, burn address, $BNKRS, pool id, staking vault, stock tokens. |
| `vault` | object | Staking vault address + on-chain totals, read live from `vaultStats()`. |
| `bnkrsMarket` | object\|null | $BNKRS market data from its WETH pool. `null` if the price could not be read. |
| `coins` | array | One entry per registered coin (9 total, incl. $BNKRS). |
| `reserveLots` | array | Open and closed 30-day stock reserve lots. |
| `burns` | array | Burn events. |
| `poolInflows` | array | $BNKRS bought for the staking pool. |
| `vaultFunding` | array | `fundRewards` pushes from treasury into the vault. |
| `alerts` | array | Open keeper alerts. Empty = healthy. |
| `trackedStocks` | array | Live tokenized-stock prices from their on-chain pools. |
| `actionLog` | array | Keeper action log, newest first. |
| `apyTrailing30dPct` | number\|null | `null` until real distributions exist. |
| `branding` | object | `{ logoUrl, bannerUrl }`. |

### `summary`

```
totalValueReservedUsd     number        USD value of all open reserve lots
totalFeesCollectedUsd     number        cumulative claimed creator fees
totalBNKRSStaked          number|null   $BNKRS staked in the vault (null if vault read failed)
totalBNKRSStakedUsd       number|null   null if vault read or $BNKRS price failed
stakerCount               number|null
vaultRewardReserve        number|null
vaultRewardsDistributed   number|null
vaultLive                 boolean       vaultStats() read succeeded
vaultDepositsPaused       boolean|null
totalCoinsBurntUsd        number
totalCoinsBurntUnits      number|null   null if no chain read succeeded
openLots                  number
dueForBurnLots            number
reservePricing            string        "on-chain stock pools" | "reference feed"
updatedAt                 ISO 8601 string
```

### `vault`

Read live from `vaultStats()` on `0x6643e356036EddF3661f063c2F40a85073Db9E25`.

```
contract                  string
verified                  boolean   source verified on Blockscout
readOk                    boolean   the on-chain read succeeded this snapshot
stakingToken              string    $BNKRS
totalStaked               number|null
totalStakedUsd            number|null
stakerCount               number|null
rewardReserve             number|null
totalRewardsDistributed   number|null
totalRewardsPaid          number|null
totalCompounded           number|null
depositsPaused            boolean|null
totalFundedBnkrs          number    cumulative fundRewards() pushed by the keeper
lastFundedAt              ISO 8601 string|null
lastFundTxHash            string|null
dist30d                   number    trailing-30d vault funding
```

### `coins[]`

```
symbol                string        e.g. "bNVDA"
name                  string
isBnkrs               boolean       true only for BNKRS
tokenAddress          string        coin contract on Robinhood Chain
pairedStock           string        e.g. "NVDA" ("WETH" for BNKRS)
stockTokenAddress     string|null
stockPriceUsd         number|null
stockChange24Pct      number|null
stockPriceSource      string|null   "onchain" | "reference"
priceUsd              number|null   null when the pool price could not be read
change24Pct           number|null
mcapUsd               number|null
liquidityUsd          number|null
vol24Usd              number|null
supply                number|null   alias of totalSupply
totalSupply           number|null   on-chain totalSupply()
burnedUnits           number|null   on-chain balance at the dead address
burnedPctOfSupply     number|null
reserveStockAmount    number
reserveUsd            number|null
totalFeeUsd           number
lotCount              number
createdAt             ISO 8601 string
dataOk                object        { market: boolean, chain: boolean }
```

### `actionLog[]`

```
ts           ISO 8601 string
timestamp    ISO 8601 string   (same value, kept for compatibility)
action       string    KEEPER_HEARTBEAT | COIN_REGISTERED | QUEUE_UPDATED |
                       FEE_CLAIM | FEE_SELL_TO_WETH | STOCK_BUY | LOT_OPENED |
                       LOT_MATURED | STOCK_SELL | COIN_BUYBACK | COIN_BURN |
                       BNKRS_POOL_BUY | VAULT_FUND | STEP_RETRY | STEP_FAILING
coinSymbol   string|null
amountUsd    number|null   null when the entry carries no USD amount
coinUnits    number|null
details      string
txHash       string|null   32-byte hex, or null for off-chain entries
explorerUrl  string|null   prebuilt Blockscout link, or null
```

On-chain actions are also independently verifiable on Blockscout. The mirror is most
useful for what the chain cannot show: heartbeats, planning-leg runs, cost basis, and
fee totals.

### `reserveLots[]`

Empty until per-coin fees clear the $1 dust threshold and the keeper opens the
first lot. Each lot carries the coin, stock, amount, USD cost basis, live value,
unrealized P&L, open date, maturity date, status, and buy/sell/burn tx hashes.

## Empty state

The platform is live but pre-volume: all 9 tokens are deployed at 100B supply,
nothing is burned, no reserve lots are open, and the staking vault holds zero
rewards. Render empty states — do not substitute placeholder numbers.

## Example

```js
const res = await fetch(
  'https://raw.githubusercontent.com/BankrStonks/bankrstonks-ledger/main/ledger.json'
);
const ledger = await res.json();

const fmt = v => (v === null || v === undefined ? '—' : v);

console.log(fmt(ledger.vault.totalStaked));
console.log(ledger.coins.filter(c => !c.isBnkrs).map(c => [c.symbol, fmt(c.priceUsd)]));
console.log(ledger.dataQuality);
```

## Contracts

| What | Address |
|---|---|
| $BNKRS | `0x1504ED99c6369f8715449a3BcD2Dc5d5e37f6bA3` |
| Staking vault (verified) | `0x6643e356036EddF3661f063c2F40a85073Db9E25` |
| Treasury / fee recipient | `0xda692f9a6ef58b9a78a3fcbdf60efbae4ee7584c` |
| Burn address | `0x000000000000000000000000000000000000dEaD` |

All on Robinhood Chain. Explorer: https://robinhoodchain.blockscout.com
