# WOWMAX aggregator against real mainnet pools

An independent test. It is not affiliated with WOWMAX, Soroswap, Phoenix or
Aquarius, and their code is unmodified. The tests call the deployed WOWMAX
aggregator `CBMPYAEOGQUJ3LVMFXPN3X4GVNPPEI6FVG6YC7HYBYSN26KODOLUSNPF` in Stellar
mainnet state captured at protocol 29, ledgers 65395952 to 65395958.
[Kanatoko](https://github.com/roman-karpovich/kanatoko) replays that state in a
local Soroban host, so the aggregator runs against the real pool contracts and
reserves.

## What the tests show

Each test sells 300 XLM for USDC through these pools. The split and merge plans
send 100 XLM down each branch:

| Branch | Pools |
| --- | --- |
| Soroswap | XLM/USDC pair `CAM7DY53G63XA4AJRS24Z6VFYAFSSF76C3RZ45BE5YU3FQS5255OOABP` |
| Phoenix | XLM/USDC pool `CBHCRSVX3ZZ7EGTSYMKPEFGZNWRVCSESQR3UABET4MIW52N4EVU6BIZX` |
| Aquarius, two hops | XLM/AQUA pool `CCY2PXGMKNQHO7WNYXEWX76L2C5BH3JUW3RCATGUYKY7QQTRILBZIFWV`, then AQUA/USDC pool `CA6GAFOJCW4MGQQBUCQUSA3CLIH25G4SNKB2JHYKZCVWZTNW5VXMSC4O` |

| Test | Result |
| --- | --- |
| `split_pays_exactly_the_pool_quotes_and_the_minimum_is_inclusive` | `swap` with the three branches as strands pays exactly the sum of each pool's own quote, 43.8755003 USDC at the captured ledger. `amount_out_min` one stroop above that reverts; equal to it, the swap goes through. |
| `merge_of_the_same_pools_pays_the_same_with_no_deadline` | `swap_merge` over the same edges, as two stages that fan in at USDC, with `deadline = 0`. It pays the same amount, and the aggregator holds no XLM, USDC or AQUA afterwards. |
| `deadline_equal_to_now_passes_phoenix_and_aquarius_but_not_soroswap` | With `deadline` equal to the ledger timestamp, a Soroswap strand fails with the router's `RouterDeadlineExpired` (`Contract(503)`). Phoenix and Aquarius strands with the same deadline go through. |

The aggregator's own check rejects a deadline only once the ledger timestamp is
past it
([`lib.rs#L120-L124`](https://github.com/wowmax-exchange/wowmax-stellar-contracts/blob/2cdb2bfcd15e62fc68f55d72b7851902bad5aa99/contracts/router/src/lib.rs#L120-L124)),
then passes it on to Soroswap and Phoenix; the Aquarius call takes no deadline.
The Soroswap router rejects it when the timestamp is equal or later
([`ensure_deadline`](https://github.com/soroswap/core/blob/6eade008ee6794075ad941609e0fef0d1124d3eb/contracts/router/src/lib.rs#L32-L39)).
So at `deadline == now` a route with a Soroswap leg reverts, and a route
without one swaps. The upstream suite's mock venue records the deadline without
checking it
([`test.rs#L61-L69`](https://github.com/wowmax-exchange/wowmax-stellar-contracts/blob/2cdb2bfcd15e62fc68f55d72b7851902bad5aa99/contracts/router/src/test.rs#L61-L69)),
and its integration tests use `now + 900`.

Authorization is mocked. The tests cover `swap` and `swap_merge` only.

## Run it

```sh
cargo test --locked
```

The tests replay the committed bundles in `fixtures/` offline and make no RPC
calls.

## Pinned commit

The plan types and the two entry points in `tests/mainnet.rs` mirror
`contracts/router/src/lib.rs` at WOWMAX commit
`2cdb2bfcd15e62fc68f55d72b7851902bad5aa99`. The code that runs is the deployed
WASM in the bundles, SHA-256
`095ee35248f9076fb76d26d7d97e1308c35586df364ff0442b664c5fb3718883`, the hash
the repository's `public/mainnet.contracts.json` lists.

## Rebuild

This builds the aggregator the way the upstream CI does, with Rust 1.95.0 and
stellar-cli 26.0.0:

```sh
git clone https://github.com/wowmax-exchange/wowmax-stellar-contracts
cd wowmax-stellar-contracts/contracts
git checkout 2cdb2bfcd15e62fc68f55d72b7851902bad5aa99
RUSTUP_TOOLCHAIN=1.95.0 stellar contract build
sha256sum target/wasm32v1-none/release/wowmax_stellar_router.wasm
```

On Linux x86-64, as in the upstream CI, this produced
`095ee35248f9076fb76d26d7d97e1308c35586df364ff0442b664c5fb3718883`, the deployed
hash, with the network disabled during the build. On macOS arm64 the same
command and toolchain produced a different module of the same size.

## Re-capture

Delete the bundles and run the tests. With a bundle missing, its test captures a
new one from the public RPC `https://mainnet.sorobanrpc.com`. This needs mainnet
to be on protocol 29. Pool reserves move, so the quoted amounts change; the
assertions do not depend on them.

```sh
rm fixtures/*.capture.json
cargo test --locked
```

## License

Our code is licensed under the MIT License or the Apache License, Version 2.0,
at your option. See `LICENSE-MIT`, `LICENSE-APACHE` and `NOTICE`, which lists
the third-party code inside the bundles.
