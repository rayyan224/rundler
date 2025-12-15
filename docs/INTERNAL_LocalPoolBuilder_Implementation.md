# LocalPoolBuilder: Internal Implementation Notes

**Author:** Senior Ethereum Smart Contract Engineer
**Component:** `LocalPoolBuilder` - Rundler Pool Server
**Last Updated:** 2025-12-10
**File Locations:**
- Primary: `crates/pool/src/server/local.rs`
- Configuration: `crates/pool/src/mempool/mod.rs`
- Integration: `crates/pool/src/task.rs`, `bin/rundler/src/cli/pool.rs`

---

## Executive Summary

`LocalPoolBuilder` is an **actor-based message-passing pool server** that manages UserOperation (UO) mempools for ERC-4337 Account Abstraction infrastructure. It implements a builder pattern for initializing an asynchronous server that handles concurrent requests for multiple EntryPoint contracts (v0.6 and v0.7) while coordinating chain updates across distributed mempool instances.

**Key Characteristics:**
- **Architecture Pattern:** Actor model with unbounded MPSC channels
- **Concurrency Model:** Tokio async runtime with spawned tasks
- **Threading:** Single event loop with async task spawning for long-running operations
- **Memory Model:** Shared Arc-wrapped mempool instances
- **Communication:** Request-Response via oneshot channels, broadcasts for chain updates

---

## 1. Architecture & Design

### 1.1 Core Components

```
┌─────────────────────┐
│ LocalPoolBuilder    │
│  - req_sender       │────┐
│  - req_receiver     │    │
│  - block_sender     │    │
└─────────────────────┘    │
         │                 │
         │ .get_handle()   │
         ▼                 │
┌─────────────────────┐    │
│ LocalPoolHandle     │    │
│  - req_sender       │◄───┘
│  - metrics          │
└─────────────────────┘
         │
         │ .run()
         ▼
┌─────────────────────────────────┐
│ LocalPoolServerRunner           │
│  - req_receiver                 │
│  - block_sender                 │
│  - mempools: HashMap<Addr, MP>  │
│  - chain_subscriber             │
│  - task_spawner                 │
└─────────────────────────────────┘
```

### 1.2 Communication Flow

```
Client → LocalPoolHandle.send(request)
  ↓ (mpsc::unbounded_send)
req_sender → req_receiver
  ↓
LocalPoolServerRunner.run() event loop
  ↓ (tokio::select!)
  ├── Chain Update → Mempool.on_chain_update() → block_sender.send()
  │                                                      ↓
  │                                                Subscribers
  └── Request Processing
      ├── Sync operations → Immediate response
      └── Async operations → Spawn task → oneshot response
```

### 1.3 State Machine Lifecycle

**Phase 1: Construction**
```rust
let builder = LocalPoolBuilder::new(block_capacity: 1024);
// Creates:
// - Unbounded MPSC channel (req_sender, req_receiver)
// - Broadcast channel (block_sender, capacity=1024)
```

**Phase 2: Handle Distribution**
```rust
let handle = builder.get_handle();
// Returns cloneable LocalPoolHandle with:
// - Clone of req_sender
// - Fresh metrics instance
```

**Phase 3: Server Execution**
```rust
builder.run(task_spawner, mempools, chain_subscriber, shutdown)
// Consumes builder, starts event loop
// Runs until shutdown signal received
```

---

## 2. Initialization & Configuration

### 2.1 Builder Initialization

**Location:** `local.rs:64-74`

```rust
pub fn new(block_capacity: usize) -> Self {
    let (req_sender, req_receiver) = mpsc::unbounded_channel();
    let (block_sender, _) = broadcast::channel(block_capacity);
    Self { req_sender, req_receiver, block_sender }
}
```

**Default Block Capacity:** `BLOCK_CHANNEL_CAPACITY = 1024` (defined in `cli/pool.rs:33`)

**Channel Characteristics:**
- **Request Channel:** Unbounded MPSC
  - **Rationale:** Prevents request backpressure from blocking clients
  - **Risk:** Memory growth under extreme load (mitigated by upstream rate limiting)
- **Block Channel:** Bounded Broadcast
  - **Size:** 1024 blocks
  - **Behavior:** Lagged subscribers receive `RecvError::Lagged(n)` notification
  - **Rationale:** Balances memory vs. chain reorg depth (1024 blocks ≈ 3.4 hours on Ethereum)

### 2.2 PoolConfig Parameters

**Location:** `mempool/mod.rs:134-186`

#### Core Parameters

| Parameter | Type | Default | Description | Performance Impact |
|-----------|------|---------|-------------|-------------------|
| `entry_point` | `Address` | Chain-specific | EntryPoint contract address | None (routing key) |
| `entry_point_version` | `EntryPointVersion` | v0.6/v0.7 | ERC-4337 version | Validation logic selection |
| `same_sender_mempool_count` | `usize` | 4 | Max UOs per unstaked sender | ⬆️ = more UOs, higher memory |
| `min_replacement_fee_increase_percentage` | `u32` | 10 | Fee bump for replacement (%) | ⬇️ = more replacements, higher churn |
| `max_size_of_pool_bytes` | `usize` | 500MB | Total mempool size limit | ⬆️ = more UOs, linear memory cost |
| `throttled_entity_mempool_count` | `u64` | 4 | Max UOs for throttled entities | ⬇️ = stricter DoS protection |
| `throttled_entity_live_blocks` | `u64` | 10 | Blocks before throttled UO expires | ⬇️ = faster eviction |
| `drop_min_num_blocks` | `u64` | 10 | Minimum blocks before drop | ⬆️ = longer retention, higher memory |
| `max_time_in_pool` | `Option<Duration>` | None | Time-based expiration | Set to prevent stale UOs |

#### Gas Efficiency Thresholds

| Parameter | Type | Default | Description | EVM Impact |
|-----------|------|---------|-------------|-----------|
| `execution_gas_limit_efficiency_reject_threshold` | `f64` | N/A | Reject if `gasLimit/gasUsed` too low | Prevents gas griefing attacks |
| `verification_gas_limit_efficiency_reject_threshold` | `f64` | N/A | Verification gas efficiency threshold | Protects bundlers from verification waste |

**Tuning Guidance:**
- **High Throughput:** Increase `max_size_of_pool_bytes`, `same_sender_mempool_count`
- **DoS Protection:** Decrease `throttled_entity_mempool_count`, increase thresholds
- **Memory Constrained:** Decrease `max_size_of_pool_bytes`, `drop_min_num_blocks`

#### Tracking & Reputation

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `paymaster_tracking_enabled` | `bool` | true | Track paymaster balances |
| `paymaster_cache_length` | `u32` | 10000 | Paymaster balance cache size |
| `reputation_tracking_enabled` | `bool` | true | Enable reputation system |
| `da_gas_tracking_enabled` | `bool` | Chain-specific | Data availability gas tracking |

#### Access Control

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `blocklist` | `Option<HashSet<Address>>` | None | Always reject these addresses |
| `allowlist` | `Option<HashSet<Address>>` | None | Bypass reputation for these addresses |

**Security Note:** Blocklist/allowlist loaded from JSON files at startup (`--pool.blocklist_path`, `--pool.allowlist_path`)

#### Mempool Channels

| Parameter | Type | Description |
|-----------|------|-------------|
| `mempool_channel_configs` | `HashMap<B256, MempoolConfig>` | Per-channel mempool sharding configuration |

**Use Case:** Sharding for multiple bundle builders to prevent UO contention

---

## 3. EVM-Specific Considerations

### 3.1 EntryPoint Version Handling

**Location:** `local.rs:698-730`

The system supports two ERC-4337 EntryPoint versions:

```rust
match mempool.entry_point_version() {
    EntryPointVersion::V0_6 => {
        if !matches!(&op, UserOperationVariant::V0_6(_)) {
            return Err("Invalid UO version");
        }
    }
    EntryPointVersion::V0_7 => {
        if !matches!(&op, UserOperationVariant::V0_7(_)) {
            return Err("Invalid UO version");
        }
    }
}
```

**Critical Invariant:** UO version MUST match mempool EntryPoint version to prevent:
- ABI decoding failures
- Gas estimation errors
- Invalid signature verification

### 3.2 Gas Mechanics

#### Gas Limit Efficiency Validation

**Concept:** Reject UOs with inflated gas limits to prevent:
1. **Bundler Resource Waste:** Over-allocated gas reserves
2. **Block Space Griefing:** UOs claiming more gas than needed
3. **Fee Market Manipulation:** Artificial scarcity attacks

**Formula:**
```
efficiency = gasUsed / gasLimit
reject if efficiency < threshold
```

**Applied To:**
- `verificationGasLimit` - Gas for `validateUserOp` + `validatePaymasterUserOp`
- `callGasLimit` - Gas for actual execution

**EVM Dependency:** Requires simulation (`eth_estimateGas`) to determine `gasUsed`

#### Gas Tracking for Data Availability

**Location:** `PoolConfig.da_gas_tracking_enabled`

**Purpose:** Track L1 data availability gas costs for L2 rollups (Optimism, Arbitrum, Base, etc.)

**EVM Interaction:**
- Calldata size calculation: `|userOp.calldata| * 16 gas/byte` (non-zero) + `4 gas/byte` (zero)
- L1 fee oracle contract reads (e.g., `OVM_GasPriceOracle`)

**Performance:** Disabled on L1 chains, adds RPC overhead on L2s

### 3.3 Opcode-Level Dependencies

#### Storage Access (EIP-2930/7702)

**Location:** `PoolConfig.max_expected_storage_slots`

**Constraint:** UOs validated against expected storage access patterns

**EVM Opcodes Monitored:**
- `SLOAD` - Warm vs. cold storage reads (2100 vs 100 gas)
- `SSTORE` - Storage writes (20000 gas cold, 2900 gas warm)
- `TLOAD`/`TSTORE` (EIP-1153) - Transient storage

**Validation:** Simulation traces storage access, rejects UOs exceeding slot limit

#### CREATE2 Dependencies (Account Factories)

**EVM Behavior:** Account creation via `initCode` uses `CREATE2` opcode

**Gas Considerations:**
- `CREATE2` base: 32000 gas
- Code deployment: 200 gas/byte
- Initialization: Constructor execution cost

**EntryPoint Interaction:**
```solidity
// EntryPoint v0.6/v0.7
if (initCode.length > 0) {
    address sender = CREATE2(...);
    require(sender == userOp.sender);
}
```

#### DELEGATECALL Safety (Paymasters)

**Security:** Paymaster `validatePaymasterUserOp` uses `DELEGATECALL`

**Risk:** Malicious paymaster could:
- Drain EntryPoint balances
- Corrupt EntryPoint state

**Mitigation:** Reputation system tracks paymaster failures

### 3.4 Chain-Specific Behaviors

**Reorg Handling:**
- Chain updates propagate via `ChainSubscriber`
- Mempools updated BEFORE block broadcast (`local.rs:669-691`)
- Ensures bundle builders see post-reorg state

**Block Gas Limit:**
- Not explicitly enforced (assumed handled by bundler)
- Implicit via `max_ops` parameter in `get_ops()`

---

## 4. Request Processing Semantics

### 4.1 Synchronous Operations

**Processed in main event loop** (no task spawning):

| Operation | Location | Complexity | Notes |
|-----------|----------|-----------|-------|
| `GetSupportedEntryPoints` | `local.rs:747-751` | O(1) | Returns mempool HashMap keys |
| `GetOps` | `local.rs:478-490` | O(n log n) | Priority queue sort |
| `RemoveOps` | `local.rs:544-548` | O(n) | Batch removal |
| `DebugDumpMempool` | `local.rs:594-601` | O(n) | Full mempool clone |

**Rationale:** Fast operations (<1ms) don't benefit from spawning overhead

### 4.2 Asynchronous Operations

**Spawned to separate tasks** via `task_spawner`:

| Operation | Location | Reason for Async | I/O Pattern |
|-----------|----------|-----------------|-------------|
| `AddOp` | `local.rs:698-730` | Simulation required | `eth_call` (validation), `eth_getCode`, `eth_getBalance` |
| `GetStakeStatus` | `local.rs:731-743` | Blockchain reads | `eth_call` to EntryPoint `getDepositInfo()` |

**Pattern:**
```rust
let fut = |mempool, response| async move {
    let result = mempool.some_async_operation().await;
    response.send(result);
};
self.get_pool_and_spawn(entry_point, req.response, fut);
```

**Error Handling:** Errors sent via oneshot channel (no panics)

### 4.3 Chain Update Processing

**Location:** `local.rs:667-693`

**Flow:**
1. Receive chain update from `chain_subscriber`
2. Clone all mempool references
3. **Spawn task** to update mempools in parallel
4. After updates complete, broadcast `NewHead` (only for confirmed blocks)

**Concurrency:** Uses `future::join_all()` to update mempools simultaneously

**Critical Timing:** Mempools MUST update before `NewHead` broadcast to prevent race conditions in bundle builders

---

## 5. Error Handling & Edge Cases

### 5.1 Unknown EntryPoint

**Trigger:** Request targets non-configured EntryPoint

**Code:** `local.rs:472-476`
```rust
fn get_pool(&self, entry_point: Address) -> PoolResult<&Arc<dyn Mempool>> {
    self.mempools.get(&entry_point)
        .ok_or_else(|| PoolError::MempoolError(MempoolError::UnknownEntryPoint(entry_point)))
}
```

**Response:** `MempoolError::UnknownEntryPoint` returned to client

### 5.2 Version Mismatch

**Trigger:** v0.6 UO sent to v0.7 mempool (or vice versa)

**Code:** `local.rs:701-715`

**Response:** Custom error message with UO type information

**Impact:** Prevents invalid ABI encoding issues

### 5.3 Channel Closure

**Scenario 1: Request Sender Closed**
```rust
self.req_sender.send(...).map_err(|_| {
    error!("LocalPoolServer sender closed");
    PoolError::UnexpectedResponse
})
```
**Cause:** Server shutdown before client request processed

**Scenario 2: Response Receiver Dropped**
```rust
recv.await.map_err(|_| {
    error!("LocalPoolServer receiver closed");
    PoolError::UnexpectedResponse
})
```
**Cause:** Client dropped request future (timeout/cancellation)

### 5.4 Broadcast Lag

**Scenario:** Subscriber can't keep up with block rate

**Code:** `local.rs:418-420`
```rust
Err(broadcast::error::RecvError::Lagged(c)) => {
    error!("new_heads_receiver lagged {c} blocks");
}
```

**Behavior:** Log error, continue processing (subscriber loses lagged blocks)

**Mitigation:** Increase `BLOCK_CHANNEL_CAPACITY` or optimize subscriber

### 5.5 Mempool Size Overflow

**Protection:** `max_size_of_pool_bytes` enforced by underlying `UoPool`

**Behavior:** LRU eviction of lowest-priority UOs

**Gas Priority Formula:** `maxPriorityFeePerGas` (primary), `maxFeePerGas` (tiebreaker)

### 5.6 Replacement Underpricing

**Validation:** New UO must exceed old UO fees by `min_replacement_fee_increase_percentage`

**Example:**
```
Old UO: maxPriorityFeePerGas = 1 gwei
New UO: Must be >= 1.1 gwei (with 10% increase)
```

**EVM Analogue:** Similar to Geth's replacement rules for transaction pool

---

## 6. Integration Instructions

### 6.1 Basic Setup

```rust
use rundler_pool::{LocalPoolBuilder, PoolTask, PoolTaskArgs, PoolConfig};
use rundler_task::TaskSpawner;
use tokio::sync::broadcast;

// 1. Create builder
let builder = LocalPoolBuilder::new(1024); // 1024 block capacity

// 2. Get handle BEFORE running server
let handle = builder.get_handle(); // Cloneable, shareable

// 3. Prepare mempools (one per EntryPoint)
let mempools = HashMap::from([
    (entry_point_v06, Arc::new(mempool_v06) as Arc<dyn Mempool>),
    (entry_point_v07, Arc::new(mempool_v07) as Arc<dyn Mempool>),
]);

// 4. Run server (consumes builder)
let server_task = builder.run(
    task_spawner,
    mempools,
    chain_subscriber,
    shutdown_signal,
);

// 5. Use handle for requests
let ops = handle.get_ops(entry_point, 100, None).await?;
```

### 6.2 Service Integration Pattern

**Location:** `task.rs` - PoolTask integration

```rust
pub struct PoolTask<P> {
    args: Args,
    event_sender: broadcast::Sender<WithEntryPoint<OpPoolEvent>>,
    pool_builder: LocalPoolBuilder,  // Stored until spawn()
    providers: P,
}

impl PoolTask {
    pub async fn spawn(self, task_spawner: T) -> Result<LocalPoolHandle> {
        // Get handle before consuming builder
        let handle = self.pool_builder.get_handle();

        // Start server
        task_spawner.spawn_critical_with_graceful_shutdown_signal(
            "op_pool_server",
            |shutdown| self.pool_builder.run(..., shutdown)
        );

        Ok(handle)
    }
}
```

**Key Pattern:** Builder stored in service struct, handle extracted during spawn

### 6.3 Multi-Service Architecture

```
┌─────────────┐
│   RPC API   │
└──────┬──────┘
       │
       ▼
┌────────────────────┐
│ LocalPoolHandle    │◄─────── Cloned to multiple services
└────────────────────┘
       │
       ▼
┌────────────────────────────┐
│ LocalPoolServerRunner      │
│  ┌──────────┐ ┌──────────┐│
│  │ Mempool  │ │ Mempool  ││
│  │  v0.6    │ │  v0.7    ││
│  └──────────┘ └──────────┘│
└────────────────────────────┘
       ▲
       │
┌──────┴──────────┐
│ ChainSubscriber │
└─────────────────┘
```

**Components:**
1. **RPC Server:** Holds `LocalPoolHandle`, serves `eth_sendUserOperation`, etc.
2. **Bundle Builder:** Subscribes to `NewHead`, calls `get_ops()`
3. **Monitoring:** Calls `debug_dump_mempool()`, `get_reputation_status()`

### 6.4 CLI Integration

**Location:** `cli/pool.rs:275-309`

```bash
# Start pool server
rundler pool \
  --pool.port 50051 \
  --pool.max_size_in_bytes 500000000 \
  --pool.same_sender_mempool_count 4 \
  --pool.min_replacement_fee_increase_percentage 10 \
  --pool.paymaster_tracking_enabled true \
  --pool.reputation_tracking_enabled true
```

**Environment Variables:**
- `POOL_PORT`: gRPC server port
- `POOL_MAX_SIZE_IN_BYTES`: Memory limit
- `POOL_BLOCKLIST_PATH`: JSON file with blocked addresses
- `POOL_ALLOWLIST_PATH`: JSON file with allowed addresses

---

## 7. Metrics & Observability

### 7.1 Built-in Metrics

**Location:** `local.rs:49-54`

```rust
#[derive(Metrics, Clone)]
#[metrics(scope = "op_pool_internal")]
struct LocalPoolMetrics {
    #[metric(describe = "the duration in milliseconds of send call")]
    send_duration: Histogram,
}
```

**Metric:** `op_pool_internal_send_duration`
- **Type:** Histogram
- **Unit:** Milliseconds
- **Meaning:** Round-trip time from handle.send() to response receipt
- **Use Case:** Detect slow mempool operations, I/O bottlenecks

### 7.2 Health Check

**Location:** `local.rs:433-453`

**Implementation:**
```rust
impl HealthCheck for LocalPoolHandle {
    async fn status(&self) -> ServerStatus {
        match timeout(Duration::from_secs(1),
                     self.get_supported_entry_points()).await {
            Ok(Ok(_)) => ServerStatus::Serving,
            _ => ServerStatus::NotServing,
        }
    }
}
```

**Behavior:** 1-second timeout on basic operation (entry point query)

**Integration:** Used by gRPC health check service, Kubernetes liveness probes

### 7.3 Logging

**Key Events:**
- `error!("LocalPoolServer sender closed")` - Critical: server shutdown
- `error!("new_heads_receiver lagged {c} blocks")` - Warning: subscriber slow
- `info!("new_heads_receiver closed, ending subscription")` - Info: clean shutdown

---

## 8. Testing Strategy

### 8.1 Unit Tests

**Location:** `local.rs:993-1162`

**Test Cases:**

1. **`test_add_op`** - Basic UO addition
   - Mock mempool expects `add_operation()`
   - Verifies hash returned correctly

2. **`test_chain_update`** - Block subscription
   - Subscribe to new heads
   - Send chain update
   - Verify received block hash/number

3. **`test_get_supported_entry_points`** - EntryPoint enumeration
   - Create mempools for 3 random addresses
   - Verify all addresses returned

4. **`test_multiple_entry_points`** - Multi-version handling
   - v0.6 and v0.7 mempools
   - Send v0.6 and v0.7 UOs
   - Verify correct routing and hashes

**Test Helper:**
```rust
fn setup(pools: HashMap<Address, Arc<dyn Mempool>>) -> State {
    let builder = LocalPoolBuilder::new(10);
    let handle = builder.get_handle();
    // ... spawn server with TaskManager
    State { handle, chain_update_tx, _task_manager }
}
```

### 8.2 Integration Test Considerations

**Not in unit tests (require full stack):**
- Actual blockchain simulation validation
- Real EntryPoint contract interaction
- Multi-threaded stress testing
- Reorg scenario validation

**Recommended Tools:**
- Anvil (local Ethereum node)
- ERC-4337 reference EntryPoint contracts
- Load testing with `tokio-test`

---

## 9. Performance Characteristics

### 9.1 Latency Profile

| Operation | Expected Latency | Bottleneck |
|-----------|------------------|------------|
| `get_supported_entry_points` | <1ms | HashMap key collection |
| `get_ops` | 1-10ms | Priority queue iteration |
| `add_op` | 50-500ms | **EVM simulation** (eth_call) |
| `get_stake_status` | 10-100ms | **RPC call** to EntryPoint |
| `subscribe_new_heads` | <1ms | Broadcast channel subscribe |

**Critical Path:** `add_op` dominated by simulation (external RPC I/O)

### 9.2 Throughput

**Request Channel:** Unbounded MPSC
- **Theoretical:** Limited by event loop processing speed
- **Practical:** Limited by mempool validation (simulation) parallelism

**Block Updates:** Serial per mempool, parallel across mempools
- **Optimization:** `future::join_all()` for concurrent mempool updates

### 9.3 Memory Usage

**Fixed Overhead:**
- Builder: ~200 bytes (3 channel endpoints)
- Handle: ~100 bytes (1 sender + metrics)
- Server: ~400 bytes + HashMap overhead

**Variable Overhead:**
- Request queue: Unbounded (risk under high load)
- Block broadcast: `1024 * sizeof(NewHead)` ≈ 64KB
- Mempools: Dominated by `max_size_of_pool_bytes` (default 500MB)

**Mitigation:** Set upstream rate limiting, monitor request queue depth

### 9.4 Concurrency Model

**Event Loop:** Single-threaded `tokio::select!` (no lock contention)

**Parallel Execution:**
- Async operations spawned to Tokio thread pool
- Mempool updates parallelized across entry points

**Lock-Free:** No mutexes in request path (channel-based synchronization)

---

## 10. Security Considerations

### 10.1 DoS Attack Vectors

**Attack 1: Unbounded Request Queue**
- **Mechanism:** Flood requests faster than processing
- **Impact:** Memory exhaustion, OOM kill
- **Mitigation:** Upstream rate limiting (not in LocalPoolBuilder)

**Attack 2: Expensive Simulation**
- **Mechanism:** Submit UOs with complex validation logic
- **Impact:** CPU exhaustion, slow request processing
- **Mitigation:** Simulation gas limits, timeout enforcement (in mempool layer)

**Attack 3: Broadcast Channel Overflow**
- **Mechanism:** Slow subscriber blocks channel
- **Impact:** Channel full → lagged errors
- **Mitigation:** Bounded channel (1024), lag detection

### 10.2 Reputation System Integration

**Purpose:** Prevent Sybil attacks via entity behavior tracking

**Entities Tracked:**
- Sender accounts
- Paymasters
- Factories
- Aggregators

**Reputation States:**
- OK: Normal operation
- Throttled: Limited UO count
- Banned: Zero UOs accepted

**Configuration:**
- `reputation_tracking_enabled`: Toggle system
- `throttled_entity_mempool_count`: Max UOs for throttled entities
- `throttled_entity_live_blocks`: Expiration for throttled UOs

### 10.3 Paymaster Balance Tracking

**Purpose:** Prevent paymaster insufficient funds DoS

**Mechanism:**
- Cache paymaster balances (LRU, size `paymaster_cache_length`)
- Reject UOs if cached balance insufficient
- Refresh on chain updates

**EVM Interaction:**
```solidity
// EntryPoint.balanceOf(paymaster)
function balanceOf(address account) external view returns (uint256);
```

---

## 11. Known Limitations & Future Work

### 11.1 Current Limitations

1. **Unbounded Request Queue**
   - No backpressure mechanism
   - Risk of memory exhaustion under extreme load

2. **No Request Prioritization**
   - FIFO processing regardless of UO fees
   - High-fee UOs may wait behind low-fee UOs

3. **Single Event Loop**
   - Single point of contention for all requests
   - Cannot scale vertically beyond single core

4. **No Persistent Storage**
   - All state in-memory
   - UOs lost on crash

### 11.2 Potential Optimizations

**Bounded Channels:**
```rust
// Replace unbounded with bounded + backpressure
let (req_sender, req_receiver) = mpsc::channel(1000);
```
**Pros:** Memory bound
**Cons:** Clients must handle `SendError::Full`

**Multi-Lane Processing:**
```rust
// Shard requests by entry point to parallel event loops
for ep in entry_points {
    spawn_server_for_entry_point(ep);
}
```
**Pros:** Parallel processing
**Cons:** Increased complexity, resource overhead

**Request Batching:**
```rust
// Batch requests every 10ms for amortized processing
while let Some(batch) = recv_batch(Duration::from_millis(10)) {
    process_batch(batch);
}
```
**Pros:** Higher throughput
**Cons:** Increased latency

---

## 12. Appendix: Code References

### 12.1 Key Type Definitions

**ServerRequestKind (21 variants):**
```rust
enum ServerRequestKind {
    GetSupportedEntryPoints,
    AddOp { entry_point, op, perms, origin },
    GetOps { entry_point, max_ops, filter_id },
    GetOpsSummaries { entry_point, max_ops, filter_id },
    GetOpsByHashes { entry_point, hashes },
    GetOpByHash { hash },
    GetOpById { id },
    RemoveOps { entry_point, ops },
    RemoveOpById { entry_point, id },
    UpdateEntities { entry_point, entity_updates },
    DebugClearState { clear_mempool, clear_reputation, clear_paymaster },
    AdminSetTracking { entry_point, paymaster, reputation },
    DebugDumpMempool { entry_point },
    DebugSetReputations { entry_point, reputations },
    DebugDumpReputation { entry_point },
    DebugDumpPaymasterBalances { entry_point },
    GetReputationStatus { entry_point, address },
    GetStakeStatus { entry_point, address },
    SubscribeNewHeads { to_track },
}
```

**ServerResponse (18 variants):** Mirror of requests, omitted for brevity

### 12.2 Dependency Graph

```
LocalPoolBuilder
├── tokio::sync::{mpsc, broadcast, oneshot}
├── rundler_task::{TaskSpawner, GracefulShutdown}
├── rundler_types::pool::{Pool, PoolOperation, ...}
├── mempool::Mempool (trait)
└── chain::ChainSubscriber
```

### 12.3 File Map

```
crates/pool/src/
├── server/
│   ├── local.rs            # LocalPoolBuilder (this file)
│   └── mod.rs              # Re-exports
├── mempool/
│   ├── mod.rs              # PoolConfig, Mempool trait
│   ├── pool.rs             # UoPool implementation
│   ├── reputation.rs       # Reputation tracker
│   └── paymaster.rs        # Paymaster tracker
├── task.rs                 # PoolTask integration
└── lib.rs                  # Public API

bin/rundler/src/cli/
├── pool.rs                 # CLI arguments, spawn_tasks()
└── node/mod.rs             # Node mode integration
```

---

## 13. Quick Reference

### 13.1 Common Operations

**Initialize:**
```rust
let builder = LocalPoolBuilder::new(1024);
let handle = builder.get_handle();
```

**Add UserOperation:**
```rust
let hash = handle.add_op(user_operation, permissions).await?;
```

**Get Operations for Bundling:**
```rust
let ops = handle.get_ops(entry_point, 100, Some("shard_0".into())).await?;
```

**Subscribe to Blocks:**
```rust
let mut stream = handle.subscribe_new_heads(vec![tracked_address]).await?;
while let Some(block) = stream.next().await {
    println!("New block: {:?}", block.block_hash);
}
```

**Health Check:**
```rust
let status = handle.status().await;
assert_eq!(status, ServerStatus::Serving);
```

### 13.2 Configuration Checklist

- [ ] Set `max_size_of_pool_bytes` based on available memory
- [ ] Configure `same_sender_mempool_count` for expected sender behavior
- [ ] Set `min_replacement_fee_increase_percentage` (10% standard)
- [ ] Enable `paymaster_tracking_enabled` for paymaster support
- [ ] Enable `reputation_tracking_enabled` for DoS protection
- [ ] Set `drop_min_num_blocks` to balance retention vs. memory
- [ ] Configure `blocklist`/`allowlist` if needed
- [ ] Set `max_time_in_pool` to prevent stale UOs
- [ ] Tune `BLOCK_CHANNEL_CAPACITY` for expected subscriber lag

---

## 14. Glossary

**UO / UserOperation:** ERC-4337 transaction representation
**EntryPoint:** Smart contract handling UO validation/execution
**Mempool:** In-memory pool of pending UOs
**Bundler:** Entity aggregating UOs into transactions
**Paymaster:** Contract sponsoring gas fees for UOs
**Reputation:** Score tracking entity reliability
**Simulation:** EVM execution preview (eth_call) for validation
**Preconf:** Pre-confirmation commitment for UO inclusion

---

**Document Version:** 1.0
**Reviewed By:** Internal
**Next Review:** On major version bump or architectural change
