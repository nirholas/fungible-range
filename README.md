# FungibleRange

**A concentrated position with fungible shares, whose band widens every time it is forced to move.**

A production Uniswap v4 hook. It holds no funds and takes no fee for itself. No owner, no pause switch, no upgrade path.

- **Site:** https://fungible-range.pages.dev
- **Catalogue:** https://hookforge.pages.dev
- **Contract:** [`src/hooks/FungibleRangeHook.sol`](src/hooks/FungibleRangeHook.sol)
- **Licence:** Apache-2.0

## How it works

Fungible wrappers around concentrated liquidity are old: Gamma, Arrakis and the Alpha Vaults have run them for years. They all share a shape, and a defect that comes with it. The shape is a vault that sits outside the pool, holds a position, and is moved by a keeper.

The defect is that the moving is either discretionary, in which case depositors are trusting a manager, or mechanical, in which case it is predictable. A predictable rebalancer is a standing invitation. If liquidity re-centres whenever the price leaves a fixed band, anybody can compute where that happens, push the price there, and trade against a position they knew in advance was about to move.

The vault pays for the rebalance and the searcher takes the difference. Making the band larger helps once and then the same attack runs at the new boundary. This band is not fixed.

Every rebalance widens it, and quiet time narrows it back toward its base. So an attacker who forces a move has, by forcing it, made the next move harder to force and the position they wanted to pick off less concentrated. Run the attack repeatedly and it damps itself; stop, and the band tightens again on its own.

The width ends up tracking realised volatility, which is what a manager was being trusted to do by hand, computed from the pool's own ticks rather than from anybody's judgement. There is no keeper. Rebalancing happens inside whichever swap pushed the price out of band, which is also the swap that had a reason to.

There is no manager, no allowlist and no parameter anybody can change after deployment.

## Prior art

Gamma, Arrakis, Charm's Alpha Vaults and Steer wrap concentrated positions in fungible shares and move them with keepers, discretionary in some cases and on a fixed band in others. Uniswap v4 auto-rebalancing hooks remove the keeper by re-centring inside a swap. Volatility-sized ranges appear in academic market-making work and in some vault heuristics, always driven by an external estimate. Making the band's width a function of how often the position has been forced to move, so that attacking the rebalance widens the band that made the attack worth running, is the contribution here.

## Where it does not help

The position is a single band, so this is a market maker with one opinion rather than a strategy; a pool whose price gaps a long way in one move leaves the band behind and re-centres at the new level, realising the loss exactly as any concentrated position does. Widening protects the rebalance, not the inventory. Depositors also share one position, so a rebalance's cost falls on everybody holding shares at that moment, including somebody who deposited a block earlier. And a pool with no flow never rebalances at all, since the trigger is a swap; the band is only as current as the pool is busy.

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
| `AlreadyBound()` | This hook manages one pool, bound the first time one initializes with it. |
| `AmountTooSmall()` | The deposit was too small to mint a share, or the withdrawal too small to return anything. |
| `CallbackNotPoolManager()` | Only the `PoolManager` may drive the unlock callback. |
| `ERC20InsufficientAllowance(address,uint256,uint256)` | Indicates a failure with the `spender`’s `allowance`. Used in transfers. |
| `ERC20InsufficientBalance(address,uint256,uint256)` | Indicates an error related to the current `balance` of a `sender`. Used in transfers. |
| `ERC20InvalidApprover(address)` | Indicates a failure with the `approver` of a token to be approved. Used in approvals. |
| `ERC20InvalidReceiver(address)` | Indicates a failure with the token `receiver`. Used in transfers. |
| `ERC20InvalidSender(address)` | Indicates a failure with the token `sender`. Used in transfers. |
| `ERC20InvalidSpender(address)` | Indicates a failure with the `spender` to be approved. Used in approvals. |
| `InsufficientInitialLiquidity()` | The first deposit must exceed the permanently locked minimum. |
| `InvalidTrigger()` | A trigger at or inside the band's own edge would rebalance on every swap. |
| `InvalidWiden()` | Widening by nothing would leave the band fixed, which is the design this replaces. |
| `InvalidWidth()` | A width of zero, or a maximum below the base, is not a band. |
| `SafeCastOverflowedIntDowncast(uint8,int256)` | Value doesn't fit in an int of `bits` size. |
| `SafeCastOverflowedIntToUint(int256)` | An int value doesn't fit in a uint of `bits` size. |
| `SafeCastOverflowedUintDowncast(uint8,uint256)` | Value doesn't fit in a uint of `bits` size. |
| `SafeCastOverflowedUintToInt(uint256)` | A uint value doesn't fit in an int of `bits` size. |
| `SafeERC20FailedOperation(address)` | An operation with an ERC-20 token failed. |

## The callbacks it claims

Uniswap v4 reads a hook's permissions from the low fourteen bits of its own address, which is why deploying one means mining a CREATE2 salt. This hook claims 2 of the fourteen:

- `afterInitialize`
- `afterSwap`

Mask: `0x1040`, so every deployment of this hook has an address ending in those bits.

## It says what it is, on-chain

Every hook in this family implements `IHookMetadata`: four view functions that let an indexer, a wallet, a router or an agent identify a hook from its address alone, with no registry in the loop.

```bash
cast call $HOOK "hookName()(string)"    # FungibleRange
cast call $HOOK "hookVersion()(string)" # 1.0.0
cast call $HOOK "specURI()(string)"     # the machine-readable manifest
cast call $HOOK "hookTags()(string[])"  # liquidity, concentrated, rebalancing, keeper-free, no-admin
```

The manifest this repository ships as [`hook.json`](hook.json) is what `specURI()` points at.

## Build and test

```bash
git clone --recurse-submodules https://github.com/nirholas/fungible-range
cd fungible-range
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
