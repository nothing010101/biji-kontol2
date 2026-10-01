# ApeStore Fee Claimer — Base only

Scan creator wallet addresses → connect **any** wallet (it only pays gas) → collect ApeStore pool fees on **Base (8453)**.
Fees are always paid by the router to the **token's creator address** — never to whoever presses the button.

Adapted from `nothing010101/biji-kontol`, stripped down to **Base chain only** and **ApeStore only**
(all other chains and all other launchpads — Noxa, Pons, Clanker, and their fee lockers — were removed).

## Repo layout

```
index.html            the whole app (single page, ethers v6 from CDN)
api/user/[address].js Vercel function — proxies GET /api/user/:address -> ape.store
api/config.js         Vercel function — proxies GET /api/config        -> ape.store
local/server.mjs      local dev server (not deployed) — serves the page + same proxy
package.json          engines/node hint for Vercel
```

## Why a proxy is needed

`ape.store`'s API returns **no CORS headers**, so the browser cannot call it cross-origin.
The `/api/*` functions call it server-side and return the JSON on the same origin as the page.

## Local run

```bash
node local/server.mjs        # http://localhost:8787
PORT=9000 node local/server.mjs
```

## Deploy to Vercel

1. Push this folder to GitHub (repo root = this folder, so `index.html` sits at the root).
2. Vercel → **Add New… → Project → Import** the repo.
3. Framework preset: **Other**. Build command: *(empty)*. Output directory: *(empty / `.`)*.
4. Deploy. `index.html` is served at `/`, and `api/*.js` become serverless functions.
5. Open the deployed URL in a browser with an injected wallet (MetaMask / Rabby / Coinbase Wallet).

No environment variables and no build step are required. `local/` is simply ignored by Vercel.

> The proxy is deliberately an allowlist: only `/api/config` and `/api/user/<0x-40-hex>` are forwarded
> to `ape.store`; anything else is rejected, so the deployment cannot be used as an open proxy.

## How it works

1. **Scan** — for each address in the textarea: `GET /api/user/<address>` returns that user's tokens
   (the same endpoint the official *My Apes* page uses). Each token carries
   `chain`, `protocol`, `router`, `address`.
2. **Base filter** — tokens whose `chain !== 8453` are ignored.
3. **Router resolution** — protocol `30` → `ApeV30Routers[router]`, `300` → `ApeV300Routers[router]`,
   `3` → `ApeV3Routers[router]`, `4` → `ApeV4Routers[router]`. Protocol `2` (V2) is not claimable and is skipped.
4. **Claim** — calls `collectFees(address token)` on that router from the connected wallet.
   The call is **permissionless**: the caller only pays gas, the payout goes to the stored creator.
   Verified on-chain: `eth_call` of `collectFees` from a random address succeeds on Base.

## Base config (snapshot of `ape.store/api/config`)

| Field | Value |
|---|---|
| Chain | Base — ID `8453`, ETH, RPC `https://mainnet.base.org`, explorer `https://basescan.org` |
| `ApeV30Routers` | `0xb3bEa12AFC263318C16DC16DbFD5AB3A0261DABf`, `0xB1900F41d78D330A2a35C6771b3A6088a1b51309`, `0x1Fc310f1ca381D42A5D23d93394BE9aA6C3675dB` |
| `ApeV300Routers` | `0x402A795Fc6ab0063726709500B4597e0873736C8` |
| `ApeV3Routers` | `0x2D0d55E8d4418ce6157Cf31d9ffafB88117C0040` |
| `ApeV4Routers` | `0xd299cbec046bb4a419db3b98e1d134f271108040` |

The app switches/adds the Base chain in the wallet automatically.

## Notes & limits

- The connected wallet does **not** have to be the creator — that is the point of "connect any wallet".
  It only needs native ETH on Base for gas.
- The API does not expose a pending-fee amount, so the app cannot show how much a token will pay.
  Claiming a token with 0 pending fees still costs a little gas.
- If ApeStore's API changes shape, update `apeRouterFor()` / the scan block in `index.html`.
