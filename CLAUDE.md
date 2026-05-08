# bch-rpc-explorer

Self-hosted Bitcoin Cash blockchain explorer built on Node.js/Express. No database — all data is fetched live from a Bitcoin Cash node via RPC, with optional caching layers. Displays blocks, transactions, addresses, mempool, mining info, and network statistics.

## Quick Start

```bash
npm install          # also compiles the C++ native addon (bch-diff.cpp) via node-gyp
npm start            # starts web server on http://127.0.0.1:3002
```

Configuration is loaded from environment variables. Copy `.env-sample` to `.env` in the working directory (or place it at `~/.config/bch-rpc-explorer.env`) and edit as needed. The app loads both files if both exist.

## Architecture: Three-Layer Data Flow

All data requests flow through three layers:

```
Browser/Route handler
       |
       v
  coreApi.js          -- abstraction layer; tiered cache + business logic
       |
       v
   rpcApi.js          -- direct RPC calls via bitcoin-core package; queue-controlled
       |
       v
Bitcoin Cash node     -- JSON-RPC over HTTP
```

**Layer 1 — `app/api/rpcApi.js`**
Thin wrapper around the `bitcoin-core` npm package. Every call goes through an `async.queue` with a configurable concurrency limit (default 10, set via `BTCEXP_RPC_CONCURRENCY`). Two global clients exist: `global.rpcClient` (with timeout) and `global.rpcClientNoTimeout` (for slow calls like `gettxoutsetinfo`). Tracks per-method call stats in `global.rpcStats`. Never import this file directly from route handlers.

**Layer 2 — `app/api/coreApi.js`**
The only file route handlers should import for blockchain data. Wraps every rpcApi call with the tiered cache via `tryCacheThenRpcApi(cache, cacheKey, maxAgeMs, rpcApiFn)`. Exports ~35 functions (see `module.exports` at line 1089). Also does data enrichment: blocks get `coinbaseTx`, `totalFees`, `subsidy`, and `miner` fields added before returning.

**Layer 3 — Routes**
`routes/baseActionsRouter.js` handles all page routes (HTML). `routes/apiRouter.js` handles `/api/*` JSON endpoints consumed by in-page AJAX. `routes/snippetRouter.js` handles `/snippet/*` for partial HTML renders. Routes set data on `res.locals` then call `res.render("template-name")`.

## Directory Structure

```
app.js                   Express app setup, middleware, startup sequence
bin/
  www                    HTTP server entry point; sets port/host, calls app.onStartup()
  cli.js                 CLI wrapper; maps CLI flags to BTCEXP_* env vars, then requires www
  refresh-mining-pool-configs.js  Utility script to update pool JSON files

app/
  config.js              Reads all BTCEXP_* env vars; exports single config object
  credentials.js         Parses RPC connection details (URI or individual vars or cookie file)
  coins.js               Registry mapping coin identifiers to coin config modules
  coins/bch.js           BCH-specific constants: genesis hashes, block reward schedule,
                         currency units, exchange rate URL, historical data
  coins/bab.js           BAB variant config
  auth.js                Basic HTTP auth middleware (checks BTCEXP_BASIC_AUTH_PASSWORD)
  utils.js               Currency formatting, error logging, miner identification,
                         exchange rate refresh, QR code generation, performance timing
  redisCache.js          Redis client wrapper; msgpack serialization; optional
  api/
    rpcApi.js            Raw RPC calls; async.queue; getRpcData() / getRpcDataWithParams()
    coreApi.js           Cached API surface; all route handlers use this
    addressApi.js        Dispatcher for address query backends
    electrumAddressApi.js  ElectrumX address queries
    blockchairAddressApi.js  Blockchair.com address queries
    blockchainAddressApi.js  blockchain.com address queries
    blockcypherAddressApi.js  BlockCypher address queries
    mockApi.js           Stub for testing (not wired in by default; swap in coreApi.js)

routes/
  baseActionsRouter.js   All HTML page routes (1661 lines); mounted at /
  apiRouter.js           JSON API routes; mounted at /api/
  snippetRouter.js       Partial HTML routes; mounted at /snippet/

views/                   Pug templates
  layout.pug             Base layout with nav, Bootstrap, theme switching
  index.pug              Homepage
  block.pug / blocks.pug / block-analysis.pug / block-stats.pug
  transaction.pug
  address.pug
  mempool-summary.pug / unconfirmed-transactions.pug
  mining-summary.pug / difficulty-history.pug
  node-status.pug / peers.pug
  terminal.pug / browser.pug   RPC tools (shown only when BTCEXP_UI_SHOW_RPC=true)
  decoder.pug
  admin.pug              Cache stats, RPC stats, error log
  error.pug

public/
  css/                   Bootstrap (dark + light), highlight.js, custom styling.css
  js/                    Bootstrap JS, jQuery, chart.js, highlight.js
  img/                   Coin logos, pool logos, UI icons
  txt/mining-pools-configs/BCH/  JSON files describing known mining pools
                                  (one file per pool, loaded at startup)

bch-diff.cpp             C++ N-API addon — exposes GetDifficulty(nBits) for real difficulty
binding.gyp              Node-GYP build config for the C++ addon
raw/                     SCSS source files for Bootstrap theme customization (not in build pipeline)
docs/                    Nginx config sample, server setup guide
```

## Configuration System

All configuration is through `BTCEXP_*` environment variables. Sources are loaded in this order (both applied if present):

1. `~/.config/bch-rpc-explorer.env`
2. `.env` in the working directory

`app/config.js` reads the environment and exports a plain config object. `app/credentials.js` handles RPC connection parsing separately. The config object is set as `global.config` at startup and also exposed to all templates as `res.locals.config`.

Key variables:

| Variable | Default | Effect |
|---|---|---|
| `BTCEXP_HOST` | `127.0.0.1` | Bind address |
| `BTCEXP_PORT` | `3002` | HTTP port |
| `BTCEXP_BITCOIND_URI` | — | Full RPC URI (overrides individual vars) |
| `BTCEXP_BITCOIND_HOST` | `127.0.0.1` | RPC host |
| `BTCEXP_BITCOIND_PORT` | `8332` | RPC port |
| `BTCEXP_BITCOIND_USER` | — | RPC username |
| `BTCEXP_BITCOIND_PASS` | — | RPC password |
| `BTCEXP_BITCOIND_COOKIE` | `~/.bitcoin/.cookie` | Cookie file (used if no user/pass) |
| `BTCEXP_ADDRESS_API` | none | `electrumx`, `blockchair.com` |
| `BTCEXP_ELECTRUMX_SERVERS` | — | Required when ADDRESS_API=electrumx |
| `BTCEXP_REDIS_URL` | — | Enables Redis caching tier |
| `BTCEXP_RPC_CONCURRENCY` | `10` | Max concurrent RPC calls |
| `BTCEXP_NO_INMEMORY_RPC_CACHE` | `false` | Disables LRU memory cache |
| `BTCEXP_SLOW_DEVICE_MODE` | `true` | Skips UTXO set and volume queries |
| `BTCEXP_PRIVACY_MODE` | `false` | Disables exchange rates and IP geolocation |
| `BTCEXP_NO_RATES` | `true` | Disables exchange rate queries |
| `BTCEXP_UI_SHOW_RPC` | `false` | Shows RPC terminal and browser tools |
| `BTCEXP_BASIC_AUTH_PASSWORD` | — | Enables HTTP basic auth |
| `BTCEXP_RPC_ALLOWALL` | `false` | Allows all RPC methods (overrides blacklist) |
| `BTCEXP_DEMO` | `false` | Demo-site mode |
| `BTCEXP_HIDE_IP` | `true` | Hides IP addresses in UI |
| `BTCEXP_OLD_SPACE_MAX_SIZE` | `1024` | V8 max old space in MB |
| `BTCEXP_HEADER_BY_HEIGHT_SUPPORT` | `false` | Node supports getblockheader by height |
| `BTCEXP_BLOCK_BY_HEIGHT_SUPPORT` | `false` | Node supports getblock by height |

## Caching System

Two optional cache tiers are stacked. If both are enabled, reads walk from fastest to slowest; writes go to all tiers.

**Memory (LRU)**
Three separate LRU caches, always active unless `BTCEXP_NO_INMEMORY_RPC_CACHE=true`:
- `miscCache` — 2000 items (blockchain info, network info, mempool info, etc.)
- `blockCache` — 2000 items (block data; stores only coinbase tx to save memory)
- `txCache` — 10000 items (only confirmed transactions with ≤9 inputs are cached)

**Redis**
Activated by setting `BTCEXP_REDIS_URL`. Uses msgpack serialization (with custom Decimal.js codec). Cache keys are namespaced by an 8-character MD5 prefix derived from RPC credentials, so multiple instances can share one Redis server without conflict. Version prefix `v1` in the key — bump this constant in `coreApi.js` if data formats change.

**Cache key pattern**
The cache key is a descriptive string, e.g. `"getBlockchainInfo"`, `"getBlock-<hash>"`, `"getRawTransaction-<txid>"`. Max ages vary: 1 second for mempool data, 10 seconds for blockchain info, 1 year for historical block data.

**Cache stats**
Available at `/admin`. Stored in `global.cacheStats` with `try`/`hit`/`miss` counters per tier.

## Code Conventions

- **Module system**: CommonJS exclusively (`require` / `module.exports`). No ES modules, no `import`/`export`.
- **Indentation**: Tabs (see `.editorconfig`).
- **Line endings**: LF.
- **Async style**: Callbacks inside `async.queue` (rpcApi layer); Promises everywhere else. Avoid mixing. No async/await in most files — the address route is a notable exception that uses `async`/`await`.
- **Variable declarations**: `var` throughout the codebase. Do not introduce `const`/`let` unless modifying code that already uses them.
- **Error handling**: Use `utils.logError(errorId, err, optionalData)`. The first argument is a unique opaque string ID (like `"3fehge9ee"` or `"239x7rhsd0gs"`). This populates `global.errorStats` and `global.errorLog`, visible in the `/admin` page. Do not use plain `console.error`.
- **No tests**: There is no test framework. `app/api/mockApi.js` exists but is commented out in `coreApi.js`.
- **Globals used at runtime**: `global.rpcClient`, `global.rpcClientNoTimeout`, `global.activeBlockchain`, `global.coinConfig`, `global.coinConfigs`, `global.config`, `global.exchangeRates`, `global.miningPoolsConfigs`, `global.specialAddresses`, `global.specialTransactions`, `global.specialBlocks`, `global.rpcStats`, `global.cacheStats`, `global.errorStats`, `global.errorLog`, `global.btcNodeSemver`.
- **Debug logging**: Use the `debug` package with `bchexp:` namespaces: `bchexp:app`, `bchexp:router`, `bchexp:core`, `bchexp:rpc`, `bchexp:utils`, `bchexp:error`, `bchexp:errorVerbose`, `bchexp:actionPerformace`. Enable with `DEBUG=bchexp:*` or specific namespaces.
- **Performance timing**: Call `utils.perfMeasure(req)` as the last statement in every route handler.

## Adding a New Page Route

1. Add a handler to `routes/baseActionsRouter.js`:
   ```js
   router.get("/my-feature", function(req, res, next) {
       coreApi.getSomeData().then(function(data) {
           res.locals.myData = data;
           res.render("my-feature");
           utils.perfMeasure(req);
       }).catch(function(err) {
           res.locals.userMessage = "Error: " + err;
           res.render("my-feature");
       });
   });
   ```
2. Create `views/my-feature.pug` extending `layout`:
   ```pug
   extends layout

   block headContent
       +title('My Feature')

   block content
       h1 My Feature
       // use res.locals variables directly
   ```
3. If new data is needed, add a function to `app/api/coreApi.js` following the `tryCacheThenRpcApi` pattern, then add the new RPC call to `app/api/rpcApi.js` if required.
4. To add it to the tools menu, add an entry to the `siteToolsJSON` array in `app/config.js` and update `subHeaderToolsList`/`prioritizedToolIdsList` indexes if desired.

## Adding a New JSON API Endpoint

Add a handler to `routes/apiRouter.js`. These are mounted at `/api/`. Return JSON, call `utils.perfMeasure(req)` at the end:

```js
router.get("/my-endpoint", function(req, res, next) {
    coreApi.getSomeData().then(function(data) {
        res.json(data);
        utils.perfMeasure(req);
    }).catch(function(err) {
        res.json({success: false, error: err});
    });
});
```

## Adding a New coreApi Function

Follow this pattern — always use `tryCacheThenRpcApi` for cacheable data:

```js
function getMyData(param) {
    return tryCacheThenRpcApi(miscCache, "getMyData-" + param, 30 * ONE_SEC, function() {
        return rpcApi.getMyData(param);
    });
}
```

Then add the corresponding rpcApi function:

```js
function getMyData(param) {
    return getRpcDataWithParams({method: "myrpcmethod", parameters: [param]});
}
```

Export both from their respective `module.exports` blocks.

## RPC Version Gating

When an RPC method was introduced in a specific node version, gate it with semver:

```js
// In rpcApi.js — add to minRpcVersions:
var minRpcVersions = {getblockstats:"1.8.0", myrpcmethod:"2.0.0"};

// Then in the function:
if (semver.gte(global.btcNodeSemver, minRpcVersions.myrpcmethod)) {
    return getRpcDataWithParams({method:"myrpcmethod", parameters:[...]});
} else {
    return unsupportedPromise(minRpcVersions.myrpcmethod);
}
```

`global.btcNodeSemver` is set at startup by parsing the `subversion` field from `getnetworkinfo`. If parsing fails, it defaults to `"1000.1000.0"` (passes all version checks).

## Key Files for Common Tasks

| Task | Primary file(s) |
|---|---|
| Add or change a page | `routes/baseActionsRouter.js`, `views/<page>.pug` |
| Add a JSON API endpoint | `routes/apiRouter.js` |
| Change cached data or business logic | `app/api/coreApi.js` |
| Add a new RPC call | `app/api/rpcApi.js` |
| Change configuration options | `app/config.js`, `.env-sample` |
| Change UI layout/nav/theme | `views/layout.pug` |
| Add or change coin-specific data | `app/coins/bch.js` |
| Change address API behavior | `app/api/addressApi.js`, `app/api/electrumAddressApi.js` |
| Change RPC credential parsing | `app/credentials.js` |

## Address API

Address balance and transaction history require an external data source; the RPC node alone does not provide address indexing. Configure with `BTCEXP_ADDRESS_API`:

- `electrumx` — self-hosted ElectrumX server; also requires `BTCEXP_ELECTRUMX_SERVERS=tls://host:port,...`
- `blockchair.com` — external Blockchair API
- (unset) — address pages show no balance or tx history

The dispatcher in `app/api/addressApi.js` routes to the correct backend. Note: `blockchain.com` and `blockcypher.com` appear in CLI help but are not currently active in `addressApi.js`'s `getSupportedAddressApis()`.

## Rate Limiting

The `/address/:address` route has a rate limiter: 1 request per IP per minute. This is defined inline in `baseActionsRouter.js` using `express-rate-limit`. If you add other potentially expensive endpoints, apply a similar limiter.

## Native Addon

`bch-diff.cpp` is a C++ N-API addon that exposes `GetDifficulty(nBits)` — converts a compact bits value to a floating-point difficulty. It is compiled during `npm install` via `node-gyp`. It is imported in `baseActionsRouter.js` with `require('bindings')('bch')` and used to show real difficulty on the homepage. If the build fails (missing build tools), the server will not start.

Docker provides a build environment: `apt-get install build-essential python git` is needed.

## Startup Sequence

1. `bin/www` — sets V8 memory limit, binds HTTP server, calls `app.onStartup()`
2. `app.onStartup()` — sets globals, loads changelog, reads git commit hash, calls `continueStartup()`
3. `app.continueStartup()` — creates two `bitcoinCore` RPC clients, starts polling `verifyRpcConnection()` every 30s
4. On first successful RPC connection — loads historical block/tx/address data for the active chain, starts exchange rate polling, loads mining pool configs
5. If mainnet — starts UTXO set and network volume refresh timers (disabled in `BTCEXP_SLOW_DEVICE_MODE=true`)

## What NOT To Do

- Do not bypass `coreApi.js` and call `rpcApi.js` directly from route handlers. The caching layer only works if all calls go through `coreApi`.
- Do not use `console.log` or `console.error`. Use `debugLog(...)` (for info) or `utils.logError(id, err)` (for errors).
- Do not add ES module syntax (`import`/`export`). The entire codebase is CommonJS.
- Do not call `res.end()` or `res.send()` in page routes — use `res.render()`.
- Do not forget `utils.perfMeasure(req)` at the end of every route handler.
- Do not cache unconfirmed transactions. `shouldCacheTransaction()` in `coreApi.js` enforces this; do not work around it.
- Do not add new `BTCEXP_*` config vars without also documenting them in `.env-sample`.
- Do not store persistent state in the process between requests beyond what is explicitly managed in the globals listed above. The app has no database.
- Do not use `res.json()` in page routes or `res.render()` in API routes.
- Do not increment `cacheKeyVersion` in `coreApi.js` casually — it invalidates all Redis cache data for every instance sharing that Redis server.
