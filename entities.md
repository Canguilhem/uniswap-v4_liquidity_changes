Entities & relationships
Chain — one record per supported EVM chain. Holds the PoolManager and PositionManager contract addresses, finality configuration, and indexerHead (the last indexed block). The poller reads all Chain records on startup and spins up one polling loop per chain.
Token — ERC-20 tokens and native ETH sentinels, scoped to a chain. Each Pool references two Token rows via token0 and token1.
Pool — a Uniswap V4 pool identified by its PoolId (keccak256 of the PoolKey struct). One PoolManager can host thousands of pools; each pool belongs to one Chain.
PoolSnapshot — change-log of pool state. Written only on tick crossings and ModifyLiquidity events, not every block. The score worker reconstructs the pool tick at any historical block via a backwards-scan query: SELECT ... WHERE poolId = $1 AND blockNumber <= $target ORDER BY blockNumber DESC LIMIT 1.
Swap — one row per swap event. Records the post-swap tick, liquidity, and sqrtPriceX96.
Tick — current state of each initialised tick boundary. Updated on every tick crossing and on every ModifyLiquidity event that touches the tick. Used by the score worker to compute fee growth per position.
Position — an LP's liquidity position identified by {poolId}:{sender}:{tickLower}:{tickUpper}:{salt}. owner is the effective wallet that receives rewards — either the sender directly (ownerType=DIRECT) or resolved from a PositionManager Transfer event (ownerType=POSITION_MANAGER).
PositionToken — tracks the ERC-721 NFT minted by PositionManager for managed positions. Updated on every Transfer event. The current owner here is always the authoritative reward recipient for PositionManager-created positions.
PositionEvent — append-only log of every ModifyLiquidity event (ADD or REMOVE) for a position. Used by the score worker to compute in-range holding windows for the JIT consistency penalty.
Campaign — a reward program targeting a specific pool over a block range. Holds all formula parameters (rangeWeightK, lambdaJIT, etc.) that the score worker reads at job start. One Campaign has many Epochs.
Epoch — a fixed scoring window [startBlock, endBlock] within a Campaign. One Epoch has many EpochScores and one Allocation per LP wallet.
EpochScore — the computed score for one Position in one Epoch. Written by the score worker; upserted so the job is safe to re-run.
Allocation — the reward amount owed to one wallet for one Epoch, with its Merkle proof. cumulativeReward accumulates across all epochs of a campaign and is the value encoded in the Merkle tree. LPs claim on-chain using the proof.

Key conventions

sqrtPriceX96, liquidity, liquidityDelta are stored as String — they exceed JS Number precision (uint160/uint128). Always convert to BigInt before arithmetic.
PoolSnapshot uses a change-log pattern — row count should be ~1–5% of total blocks indexed, not 100%.
salt is always included in positionId — a single address can hold multiple positions at the same tick range using different salts.
blockHash on Swap and PoolSnapshot correlates events within a batch, not for reorg detection. The indexer only processes finalized blocks.
