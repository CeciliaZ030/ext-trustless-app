# Trustless Work on Aomi

An [Aomi](https://aomi.dev) App that turns a service agreement into a single-release
escrow on Arc Testnet (chain `5042002`). The Trustless Work Arc API builds the unsigned
deploy; this App checks it, the user reviews it, and the Aomi host simulates it. The App
never signs or broadcasts.

## Flow

1. `trustless_work_review_escrow` takes the agreement (engagement id, title, description,
   amount, platform fee, milestones, role wallets). The signer is the connected wallet.
   It calls `POST /escrow/single-release/v2/deploy`, decodes the returned `createEscrow`
   calldata and refuses the build unless every field matches the agreement and
   `contractId` is the CREATE2 address that calldata creates. It returns the escrow
   address, amount, platform fee, the 0.3% Trustless Work fee, what the receiver gets,
   roles and milestones for review.
2. After the user confirms, `trustless_work_simulate_escrow` takes the reviewed
   `contract_id`, stages that exact build with `stage_tx` and runs `simulate_batch`
   (stops on failure). Only the latest reviewed build of the session can be simulated,
   once, within 15 minutes.

## Rules

| Rule | Behavior |
|---|---|
| Amount in normal USDC units (`10` = 10 USDC) | Sent as-is; the decoded on-chain amount must equal amount × 10^6 |
| At most 6 decimals | Otherwise stops with `AMOUNT_PRECISION_UNSUPPORTED` before any API call |
| `platformFee` is a percent (`2` = 2%) | Must decode to percent × 100 basis points |
| Dispute resolver holds no other role | Otherwise stops with `ESCROW_DISPUTE_RESOLVER_OVERLAPS_WITH_OTHER_ROLE` (HTTP 422) |
| Each `/deploy` returns a new `contractId` | A rebuild replaces the reviewed build and must be reviewed again |

API errors (`code`, status and `detail`) are shown to the user and stop the flow.

## Configuration

Secret `TRUSTLESS_WORK_API_KEY`, sent as `x-api-key`. Set it in the App's Environment
tab in Aomi Build. The API host, factory, implementation and USDC addresses are fixed
in `src/tool.rs`.

## Tests

```sh
cargo test --locked                                          # unit tests
cargo test --locked live_factory -- --ignored                # read-only Arc Testnet checks
TRUSTLESS_WORK_API_KEY=... cargo test --locked live_api -- --ignored --nocapture
```

`ARC_RPC_URL` overrides the RPC used by the live tests (default `https://arc-testnet.drpc.org`).

## Deploy

`.aomi/config.json` at the repository root points Aomi Build at `aomi.toml`. Connect this
repository through the Aomi GitHub App and choose Deploy. The crate is its own Cargo
workspace and pins `aomi-sdk = "=6.0.0"`, the version the Aomi host currently runs.
