# SlipPay Checkout — deployment & proof

The "Stripe of stablecoin" revenue primitive. One customer payment → contract
forces an atomic 97/3 split (net to merchant, fee to SlipPay). Fee recipient +
bps fixed at deploy (constructor) — INESCAPABLE; the merchant cannot route around it.

## Contract
- src: `contracts/checkout/src/lib.rs` — `pay(from, merchant, token, amount)`, `__constructor(fee_to, fee_bps)`, `fee_bps()`, `fee_to()`.
- wasm: 5256 bytes, 4 exported fns.

## Testnet deploy

Deploy is driven by `contracts/checkout/deploy-testnet.sh`. The fee recipient,
(rate in basis points) and the payment token are explicit arguments, validated
before `soroban contract deploy` is called. After deploy the script reads `fee_bps()`
and `fee_to()` back off the deployed contract and asserts they match the requested
values — the constructor arguments are immutable, so this is the only chance to
catch a wrong parameter.

### Prerequisites
- `soroban cli` on `PATH`
- a funded testnet identity (`sororan keys generate deployer`)
- the checkout wasm built: `cargo build --target wasm32-unknown-unknown --release`

### Secrets

Network and identity are read from `contracts/checkout/.testnet-deploy.env`,
which is git-ignored (`contracts/*/.testnet-deploy.env`). Never commit it.

```
SOROBAN_NETWORK=testnet
SOROBAN_IDENTITY=deployer
```

### Run

```
cd contracts/checkout
cargo build --target wasm32-unknown-unknown --release
./deploy-testnet.sh \
  --fee-to GBVMU64N5JK3NICY3ZRPXVW4RQIQFHLBPW5U4F55KVJPZJS3SIOWJF4 \
  --fee-bps 300 \
  --token CDLZFC3SYJDZT7K67VZ75HPJVIEUVNIXF47ZG2FB2RMQQVU2HHGCYSC
```

The script prints the deployed contract id and the verified `fee_to`/`fee_bps`,
and exits non-zero if the on-chain configuration does not match the request.

## Testnet deploy (2026-06-05)
- contract id: `CBO2CODZUTHH4II4JCQRZVO4RKDUIUH4MXZTAWOYVUZIVYI47UIDQCWQ`
- fee_to: `GBVMU64N5JK3NICY3ZRPXVW4RQIQFHLBPW5UI4F55KVJPZJS3SIOWJF4` (slippay-fee)
- fee_bps: 300 (3%)
- token used in proof: native SAC `CDLZFC3SYJDZT7K67VZ75HPJVIEUVNIXF47ZG2FB2RMQQVU2HHGCYSC`

## Proof
- tx `e142cd0bbaa19adfe1b74d3629ea7788f06abca8a0ba6ba121e6f41c67390941`
- buyer paid 10 → merchant +9.7 (97%), slippay-fee +0.3 (3%), one atomic call.

## Mainnet path
- same contract, deploy with mainnet slippay-fee + USDC SAC as token. Needs deployer XLM.
- frontend wrapper (TODO): merchant creates charge (amount) → QR encodes {checkout contract, merchant, amount} → customer pays via /pay (passkey → invoke checkout.pay) → split auto.
