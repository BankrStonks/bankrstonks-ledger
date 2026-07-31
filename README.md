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

## Price freshness — last-known-good (schemaVersion 4)

The upstream market source occasionally rate-limits or drops a request. As of
schemaVersion 4 a failed read no longer wipes a good price to `null`. Instead the
last successful reading is carried forward and labelled:

```
priceUsd         number|null   last known good price
priceAsOf        ISO 8601|null when that price was actually read
priceStale       boolean       true when priceAsOf is older than freshnessWindowSeconds
priceAgeSeconds  number|null   age of the reading in seconds (0 = read this snapshot)
priceSource      string|null   "live" (read this snapshot) | "cache" (carried forward)
```

Top-level knobs:

```
freshnessWindowSeconds   900      <= 15 min old counts as fresh (priceStale = false)
maxCacheAgeSeconds       604800   older than 7 days is discarded -> priceUsd becomes null
```

Recommended rendering:

- `priceStale === false` → show the price normally
- `priceStale === true` → show the price with a "as of {priceAsOf}" / greyed treatment
- `priceUsd === null` → em dash, no number

Tokenized-stock prices carry the same treatment via `stockPriceUsd` /
`stockPriceAsOf` / `stockPriceStale`, and `trackedStocks[]` entries carry
`priceAsOf` / `priceStale`. `bnkrsMarket` carries `priceAsOf`, `priceStale`,
`priceAgeSeconds` and `priceSource`.

`stockPriceSource` is `"onchain"` when read this snapshot, `"onchain-cached"` when
carried forward from the cache, `"reference"` when it came from the fallback feed.

## dataQuality

Check `dataQuality` before trusting a partially-populated snapshot:

```
vaultReadOk              boolean   vaultStats() read succeeded
bnkrsPriceOk             boolean   $BNKRS price resolved (live or cached)
bnkrsPriceStale          boolean   the $BNKRS price is a carried-forward reading
coinsWithMarketData      number    coins with any usable price (live or cached)
coinsWithLiveMarketData  number    coins priced fresh this snapshot
coinsTotal               number
coinsMissingMarket       string[]  symbols with no usable price at all
coinsFromCache           string[]  symbols served from the last-known-good cache
oldestPriceAgeSeconds    number|null  age of the oldest price in the snapshot
coinsMissingChainData    string[]  symbols whose supply/burn read failed
stocksPricedOnchain      number    stocks priced fresh from their pools
stocksPricedFromCache    number    stocks served from cache
marketFetchErrors        string[]  empty when all market reads succeeded
```

Each `coins[]` entry also carries `dataOk: { market, chain, marketLive }` for
per-coin gating — `market` means "a usable price exists", `marketLive` means
"that price was read this snapshot".

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
| `schemaVersion` | number | Bumped on any breaking shape change. Currently `4`. |
| `generatedAt` | ISO 8601 string | When this snapshot was produced. |
| `source` | string | Producing system. |
| `chain` | string | `robinhood` (Robinhood Chain). |
| `explorer` | string | Explorer base URL. |
| `website` | string | bankrstonks.fun |
| `updateCadence` | string | Human-readable cadence. |
| `nullPolicy` | string | Machine-readable restatement of the null + staleness rules. |
| `freshnessWindowSeconds` | number | Age at which a price is marked stale. |
| `maxCacheAgeSeconds` | number | Age at which a cached price is discarded to `null`. |
| `dataQuality` | object | Per-snapshot read health. See above. |
| `summary` | object | Headline stats. |
| `control` | object | `{ paused, pausedAt }` — global keeper pause switch. |
| `config` | object | Hold days, keeper cadence, max slippage, fee splits. |
| `contracts` | object | Every address: treasury, keeper, burn address, $BNKRS, pool id, staking vault, stock tokens. |
| `vault` | object | Staking vault address + on-chain totals, read live from `vaultStats()`. |
| `bnkrsMarket` | object\|null | $BNKRS market data from its WETH pool. `null` if never resolved. |
| `coins` | array | One entry per registered coin (9 total, incl. $BNKRS). |
| `reserveLots` | array | Open and closed 30-day stock reserve lots. |
| `burns` | array | Burn events. |
| `poolInflows` | array | $BNKRS bought for the staking pool. |
| `vaultFunding` | array | `fundRewards` pushes from treasury into the vault. |
| `alerts` | array | Open keeper alerts. Empty = healthy. |
| `trackedStocks` | array | Tokenized-stock prices with `priceAsOf` / `priceStale`. |
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

Vault values are **never** cached or carried forward — they are read fresh from the
contract every snapshot, and null out if the read fails. Chain is authoritative.

### `coins[]`

```
symbol                string        e.g. "bNVDA"
name                  string
isBnkrs               boolean       true only for BNKRS
tokenAddress          string        coin contract on Robinhood Chain
pairedStock           string        e.g. "NVDA" ("WETH" for BNKRS)
stockTokenAddress     string|null
stockPriceUsd         number|null
stockPriceAsOf        ISO 8601|null
stockPriceStale       boolean
stockChange24Pct      number|null
stockPriceSource      string|null   "onchain" | "onchain-cached" | "reference"
priceUsd              number|null   last known good pool price
priceAsOf             ISO 8601|null when that price was read
priceStale            boolean       older than freshnessWindowSeconds
priceAgeSeconds       number|null
priceSource           string|null   "live" | "cache"
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
dataOk                object        { market, chain, marketLive }
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
unrealized P&L, open date, maturity date, status, buy/sell/burn tx hashes, plus
`stockPriceAsOf` / `stockPriceStale` for the price used to value it.

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

function renderPrice(c) {
  if (c.priceUsd === null) return '—';
  return c.priceStale
    ? `${c.priceUsd} (as of ${c.priceAsOf})`
    : `${c.priceUsd}`;
}

console.log(fmt(ledger.vault.totalStaked));
console.log(ledger.coins.map(c => [c.symbol, renderPrice(c)]));
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
