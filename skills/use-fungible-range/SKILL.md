---
name: use-fungible-range
description: Build, test, integrate and deploy the FungibleRangeHook Uniswap v4 hook from the fungible-range repo. Use when you need to work with this hook's contract, wire it into a pool, or deploy it with a mined CREATE2 salt.
---

# FungibleRangeHook

A concentrated position with fungible shares, whose band widens every time it is forced to move.

Family: Liquidity management. Tags: liquidity, concentrated, rebalancing, keeper-free, no-admin. License: Apache-2.0. Source: `src/hooks/FungibleRangeHook.sol`. The machine-readable manifest is `hook.json` (ABI, errors, permissions, properties).

## Build and test

Foundry project (`foundry.toml`, solc 0.8.26, evm `cancun`). From a clone of the repo:

```bash
git clone https://github.com/nirholas/fungible-range && cd fungible-range
git submodule update --init --recursive   # dependencies live in lib/
forge build
forge test
```

Tests are in `test/FungibleRangeHook.t.sol`. Use `FOUNDRY_PROFILE=ci forge test` for the larger fuzz run (5000 runs).

## Hook permissions

Uniswap v4 reads a hook's permissions from the low bits of its address. This hook claims: `afterSwap`, `afterInitialize`. It is not upgradeable and has no custom swap data requirement (`hook.json` `properties`).

## Configuring

This hook has no `configure` call. Its terms are constructor arguments, fixed at deployment:

```solidity
constructor(IPoolManager, int24 baseHalfWidth, int24 maxHalfWidth, uint256 triggerBps, uint256 widenBps, uint64 calmSeconds, string shareName, string shareSymbol)
```

Edit the arguments in `script/Deploy.s.sol` before deploying, since the deployed address depends on them.

## Deploying

A v4 hook must be deployed at an address whose bits match its permissions, so `script/Deploy.s.sol` mines a CREATE2 salt first. The dry run sends nothing:

```bash
export RPC_URL=https://...        # a chain with a Uniswap v4 PoolManager
forge script script/Deploy.s.sol --rpc-url $RPC_URL
```

Broadcasting spends gas from the signer (`PRIVATE_KEY`, or `--account` / `--ledger`) and is irreversible, so show the user the chain, the predicted hook address and the signer, and get an explicit yes before running:

```bash
forge script script/Deploy.s.sol --rpc-url $RPC_URL --broadcast --verify
```

Re-running is safe: a hook already at its mined address is reported and skipped. If the miner fails with `EvmError: OutOfGas`, add `--gas-limit 30000000000`. `--verify` needs `ETHERSCAN_API_KEY`. The chain must be in the address book `src/libraries/Chains.sol`. `script/Deploy.s.sol` fails with "no Uniswap v4 PoolManager known for this chain" otherwise.

## Identifying a deployed hook

```solidity
IHookMetadata(hookAddress).hookName();
IHookMetadata(hookAddress).specURI();   // points at hook.json
```

## Errors you may meet

`AlreadyBound()`, `AmountTooSmall()`, `CallbackNotPoolManager()`, `InsufficientInitialLiquidity()`, `InvalidPool()`, `InvalidTrigger()`, `InvalidWiden()`, `InvalidWidth()`, `SafeCastOverflowedIntDowncast(uint8,int256)`, `SafeCastOverflowedIntToUint(int256)`, `SafeCastOverflowedUintDowncast(uint8,uint256)`, `SafeCastOverflowedUintToInt(uint256)`, `SafeERC20FailedOperation(address)`.

## Limitation

The position is a single band, so this is a market maker with one opinion rather than a strategy; a pool whose price gaps a long way in one move leaves the band behind and re-centres at the new level, realising the loss exactly as any concentrated position does. Widening protects the rebalance, not the inventory. Depositors also share one position, so a rebalance's cost falls on everybody holding shares at that moment, including somebody who deposited a block earlier. And a pool with no flow never rebalances at all, since the trigger is a swap; the band is only as current as the pool is busy.

More: `docs/integrating.md`, `docs/deploying.md`, and None.
