# StepCurve

<!-- three.ws:badges -->
[![GitHub stars](https://img.shields.io/github/stars/nirholas/step-curve?style=flat&logo=github)](https://github.com/nirholas/step-curve/stargazers) [![License](https://img.shields.io/github/license/nirholas/step-curve?style=flat)](https://github.com/nirholas/step-curve/blob/HEAD/LICENSE) [![Last commit](https://img.shields.io/github/last-commit/nirholas/step-curve?style=flat)](https://github.com/nirholas/step-curve/commits) [![PRs welcome](https://img.shields.io/badge/PRs-welcome-brightgreen?style=flat)](https://github.com/nirholas/step-curve/pulls) [![AI agent friendly](https://img.shields.io/badge/AI%20agents-AGENTS.md%20%2B%20llms.txt-6d5dfc?style=flat)](https://github.com/nirholas/step-curve/blob/HEAD/AGENTS.md)
<!-- /three.ws:badges -->


**A pool whose price moves in discrete steps instead of continuously, so there is no infinitesimal arbitrage to take and a quote holds still long enough to be worth quoting.**

A production Uniswap v4 hook. It holds no funds and takes no fee for itself. No owner, no pause switch, no upgrade path.

- **Site:** https://step-curve.pages.dev
- **Catalogue:** https://hookforge.pages.dev
- **Contract:** [`src/hooks/StepCurveHook.sol`](src/hooks/StepCurveHook.sol)
- **Licence:** Apache-2.0

## How it works

A constant-product pool changes its price on every swap, by any amount, however small. That is elegant and it has a cost that falls entirely on the people providing the liquidity: the price is always slightly wrong by an amount somebody can capture, and the smaller the increment the more often it is worth capturing. Continuous pricing is what makes an AMM permanently arbitrageable rather than occasionally.

Every other market prices in ticks. Equities trade in cents, bonds in thirty-seconds, futures in whatever the exchange decided, and the reason is not tradition: a minimum increment means a quote is worth something for a while, because moving the price at all costs a whole increment rather than a rounding error. This pool prices in increments.

The quote is constant across a band of inventory `stepSize` wide and falls by `stepX96` when the band is crossed, so: price(reserve0) = max(minPriceX96, startPriceX96 - (reserve0 / stepSize) * stepX96) Within a band the pool is a constant-sum market maker at a fixed price, which is to say it fills at exactly the quote with no slippage at all. A swap that would cross bands walks them, filling each at its own price. So a small trade sees a firm quote and no slippage, and a large trade sees exactly the depth the ladder was configured with.

The consequence for arbitrage is the point. An external price move smaller than one increment is not tradeable against this pool at all, because moving the price requires consuming a whole band. Providers are exposed to moves larger than the increment and immune to noise below it, which is the trade every quoting venue in the world makes.

Liquidity is fungible and proportional; see {ForgeCurveHook}. There are no ticks and no ranges, because the ladder is the range.

## Prior art

Constant-sum hooks (Uniswap's own constant-sum example, the StableSwap and Orbital submissions) replace the curve with a different continuous one. Bancor's Carbon quotes discrete asymmetric ladders, off v4. Discrete tick pricing is universal outside crypto. A v4 custom curve that is constant-sum inside a band and steps between bands, so a quote is firm below the increment and depth is exactly what was configured, is the contribution here.

## Where it does not help

A swap that crosses many bands walks them one at a time, so gas grows with the number of bands crossed and a swap larger than `MAX_STEPS` bands reverts rather than filling partially. Size `stepSize` against the trades the pool expects. The ladder is also fixed at deployment: a pool whose asset moves far outside the configured range runs out of ladder and stops quoting on that side, which is honest but is not the same as a curve that quotes everywhere.

## Using it

Uniswap v4 removed `hookData` from `initialize`, so per-pool parameters arrive out of band. Fix them for a pool key whose pool does not exist yet, then initialize. Nobody can change them afterwards, including you.

```solidity
// This hook needs no configuration.

poolManager.initialize(key, startingSqrtPriceX96);
```


### Parameters

This hook takes no per-pool configuration.

## What it reverts with

| Error | Meaning |
| --- | --- |
| `AlreadyInitialized()` | Hook was already initialized. |
| `AmountTooSmall()` | A deposit was too small to mint any shares, or a withdrawal too small to return anything. |
| `ERC20InsufficientAllowance(address,uint256,uint256)` | Indicates a failure with the `spender`’s `allowance`. Used in transfers. |
| `ERC20InsufficientBalance(address,uint256,uint256)` | Indicates an error related to the current `balance` of a `sender`. Used in transfers. |
| `ERC20InvalidApprover(address)` | Indicates a failure with the `approver` of a token to be approved. Used in approvals. |
| `ERC20InvalidReceiver(address)` | Indicates a failure with the token `receiver`. Used in transfers. |
| `ERC20InvalidSender(address)` | Indicates a failure with the token `sender`. Used in transfers. |
| `ERC20InvalidSpender(address)` | Indicates a failure with the `spender` to be approved. Used in approvals. |
| `ExpiredPastDeadline()` | A liquidity modification order was attempted to be executed after the deadline. |
| `InsufficientInitialLiquidity()` | The first deposit must exceed the permanently locked minimum. |
| `InsufficientReserves()` | The ladder cannot fill this swap: the pool has run out of the currency being bought. |
| `InvalidLadder()` | A constructor argument was zero or inconsistent. |
| `InvalidNativePayer(address)` | The native currency was settled on behalf of a `payer` other than the contract paying it. |
| `InvalidNativeValue()` | Native currency was not sent with the correct amount. |
| `LiquidityOnlyViaHook()` | Liquidity was attempted to be added or removed via the `PoolManager` instead of the hook. |
| `PoolNotInitialized()` | Pool was not initialized. |
| `SafeERC20FailedOperation(address)` | An operation with an ERC-20 token failed. |
| `SwapTooLarge()` | The swap would cross more than `MAX_STEPS` bands. Split it, or the pool needs a wider `stepSize`. |
| `TooMuchSlippage()` | Principal delta of liquidity modification resulted in too much slippage. |

## The callbacks it claims

Uniswap v4 reads a hook's permissions from the low fourteen bits of its own address, which is why deploying one means mining a CREATE2 salt. This hook claims 5 of the fourteen:

- `beforeInitialize`
- `beforeAddLiquidity`
- `beforeRemoveLiquidity`
- `beforeSwap`
- `beforeSwapReturnsDelta`

Mask: `0x2a88`, so every deployment of this hook has an address ending in those bits.

## It says what it is, on-chain

Every hook in this family implements `IHookMetadata`: four view functions that let an indexer, a wallet, a router or an agent identify a hook from its address alone, with no registry in the loop.

```bash
cast call $HOOK "hookName()(string)"    # StepCurve
cast call $HOOK "hookVersion()(string)" # 1.0.0
cast call $HOOK "specURI()(string)"     # the machine-readable manifest
cast call $HOOK "hookTags()(string[])"  # curve, custom-curve, discrete-pricing, mev, oracle-free
```

The manifest this repository ships as [`hook.json`](hook.json) is what `specURI()` points at.

## Build and test

```bash
git clone --recurse-submodules https://github.com/nirholas/step-curve
cd step-curve
forge build
forge test
```

Foundry 1.7 or newer, Solidity 0.8.26, EVM version `cancun` (Uniswap v4 requires transient storage).

## Deploy

```bash
# Dry run: mines the salt and prints the address without sending anything.
forge script script/Deploy.s.sol --rpc-url $RPC_URL

# For real.
forge script script/Deploy.s.sol --rpc-url $RPC_URL --broadcast --verify
```

Needs `PRIVATE_KEY` in the environment and a funded deployer on the target chain. See [`docs/deploying.md`](docs/deploying.md).

## Status

**Unaudited.** Built to an audited shape, on OpenZeppelin's audited hook bases, and tested against a real `PoolManager`. No third party has reviewed it. Read "where it does not help" above before putting money behind it.

Not affiliated with Uniswap Labs.

## Star History

[![Star History Chart](https://api.star-history.com/svg?repos=nirholas/step-curve&type=Date)](https://www.star-history.com/#nirholas/step-curve&Date)

<!-- three.ws:growth -->
## Support the project

If step-curve saves you time, **[star it on GitHub](https://github.com/nirholas/step-curve)**. Stars are how other developers and AI agents find the repositories worth trusting, and they cost you one click.

Know someone who would use it? [Post on X](https://twitter.com/intent/tweet?text=step-curve%3A%20A%20pool%20whose%20price%20moves%20in%20discrete%20steps%20instead%20of%20continuously%2C%20so%20there%20is%20no%20infinitesimal&url=https%3A%2F%2Fgithub.com%2Fnirholas%2Fstep-curve) · [Share on Bluesky](https://bsky.app/intent/compose?text=step-curve%3A%20A%20pool%20whose%20price%20moves%20in%20discrete%20steps%20instead%20of%20continuously%2C%20so%20there%20is%20no%20infinitesimal%20https%3A%2F%2Fgithub.com%2Fnirholas%2Fstep-curve) · [Share on LinkedIn](https://www.linkedin.com/sharing/share-offsite/?url=https%3A%2F%2Fgithub.com%2Fnirholas%2Fstep-curve) · [Submit to Hacker News](https://news.ycombinator.com/submitlink?u=https%3A%2F%2Fgithub.com%2Fnirholas%2Fstep-curve&t=step-curve%3A%20A%20pool%20whose%20price%20moves%20in%20discrete%20steps%20instead%20of%20continuously%2C%20so%20there%20is%20no%20infinitesimal) · [Share on Reddit](https://www.reddit.com/submit?url=https%3A%2F%2Fgithub.com%2Fnirholas%2Fstep-curve&title=step-curve%3A%20A%20pool%20whose%20price%20moves%20in%20discrete%20steps%20instead%20of%20continuously%2C%20so%20there%20is%20no%20infinitesimal)

## Built for AI agents too

Coding agents and LLM tooling can read this repo directly: [AGENTS.md](./AGENTS.md), [llms.txt](./llms.txt), [llms-full.txt](./llms-full.txt). Point an agent at `https://github.com/nirholas/step-curve` and it has the context it needs.

## More from the same author

- [All repositories by nirholas](https://github.com/nirholas/nirholas#readme): the full catalog, grouped by topic
- [three.ws](https://three.ws): the platform for 3D AI agents with Solana wallets, a skill marketplace and x402 payments
- Questions or ideas: [open an issue](https://github.com/nirholas/step-curve/issues) or [start a discussion](https://github.com/nirholas/step-curve/discussions)

## Contributors

[![Contributors](https://contrib.rocks/image?repo=nirholas/step-curve)](https://github.com/nirholas/step-curve/graphs/contributors)

<!-- /three.ws:growth -->
