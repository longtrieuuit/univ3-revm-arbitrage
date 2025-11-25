# 🔧 SO SÁNH KỸ THUẬT CHI TIẾT: TIER 2 vs TIER 3

## 📌 OVERVIEW

```
TIER 2 (REVM)                    TIER 3 (Advanced Mock)
═════════════════════════════════════════════════════════════
Fetch state từ blockchain        Inject state manually
Real contracts + Real state      Mock contracts + Mocked state
99%+ chính xác                   90-98% chính xác (cần validate)
12.8x nhanh hơn RPC             30x nhanh hơn RPC
2-3 RPC calls                    0 RPC calls
Production-safe ✅              Cần validation ⚠️
```

---

## 🏗️ ARCHITECTURE CHI TIẾT

### **TIER 2: State Fetching Architecture**

```
┌──────────────────────────────────────────────────────────────┐
│                    TIER 2: REVM + AlloyDB                   │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  Application Code                                           │
│  ↓                                                          │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  REVM Context                                        │  │
│  │  ├─ EVM State Machine                              │  │
│  │  ├─ Transaction Execution                          │  │
│  │  └─ Contract Call Handling                         │  │
│  └──────────────────────────────────────────────────────┘  │
│                        ↓                                    │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  CacheDB (Wrapper)                                  │  │
│  │  ├─ In-Memory Cache (HashMap)                       │  │
│  │  ├─ Disk Cache (cacache library)                    │  │
│  │  └─ Account/Storage tracking                        │  │
│  └──────────────────────────────────────────────────────┘  │
│           ↓                                                 │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  AlloyDB (State Provider)                           │  │
│  │  ├─ Fetch account bytecode                          │  │
│  │  ├─ Fetch storage slots                             │  │
│  │  ├─ Fetch balances                                  │  │
│  │  └─ Fetch nonce                                     │  │
│  └──────────────────────────────────────────────────────┘  │
│           ↓                                                 │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  Alloy HTTP Client                                  │  │
│  │  (RPC Calls - Only when cache miss!)               │  │
│  └──────────────────────────────────────────────────────┘  │
│           ↓                                                 │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  Ethereum Node (RPC)                                │  │
│  │  ├─ eth_getCode (get bytecode)                      │  │
│  │  ├─ eth_getStorageAt (get storage)                  │  │
│  │  └─ eth_getBalance (get balance)                    │  │
│  └──────────────────────────────────────────────────────┘  │
│           ↓                                                 │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  Blockchain                                         │  │
│  │  Real state, real contracts, real data             │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                              │
│  KEY FLOW:                                                  │
│  1. First call to contract A:                             │
│     ├─ Check CacheDB: Not found                           │
│     ├─ Check disk cache: Not found                        │
│     ├─ RPC call: eth_getCode → Ethereum                   │
│     ├─ Store in memory & disk                             │
│     └─ Result to REVM                                     │
│                                                              │
│  2. Second call to contract A:                            │
│     ├─ Check CacheDB: Found! ✅                           │
│     ├─ Return from memory (instant)                       │
│     └─ No RPC call needed                                 │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

### **TIER 3: Manual Injection Architecture**

```
┌──────────────────────────────────────────────────────────────┐
│               TIER 3: REVM + Mocked State                    │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  Application Code                                           │
│  ↓                                                          │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  REVM Context                                        │  │
│  │  ├─ EVM State Machine (same as TIER 2)             │  │
│  │  ├─ Transaction Execution (same as TIER 2)         │  │
│  │  └─ Contract Call Handling (same as TIER 2)        │  │
│  └──────────────────────────────────────────────────────┘  │
│                        ↓                                    │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  CacheDB (Wrapper)                                  │  │
│  │  ├─ In-Memory Cache (HashMap)                       │  │
│  │  ├─ PRE-LOADED with mocked data                     │  │
│  │  └─ Account/Storage tracking                        │  │
│  └──────────────────────────────────────────────────────┘  │
│           ↓                                                 │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  Mocked Data (100% In-Memory)                       │  │
│  │  ├─ Custom quoter bytecode (from file)             │  │
│  │  ├─ Mock ERC20 bytecode (from file)                │  │
│  │  ├─ Mock pool bytecode (from file)                 │  │
│  │  ├─ Injected storage slots                         │  │
│  │  └─ Fake balances, liquidity, etc                  │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                              │
│  ⚠️ NO NETWORK CONNECTION NEEDED!                          │
│  🟢 100% OFFLINE EXECUTION                                 │
│                                                              │
│  KEY FLOW:                                                  │
│  1. Initialization (one time):                            │
│     ├─ Load custom quoter bytecode from file            │
│     ├─ Load ERC20 mock bytecode from file              │
│     ├─ Manually inject storage slots:                   │
│     │  ├─ User WETH balance: 1000 (0x...)             │
│     │  ├─ Pool USDC balance: 200k (0x...)             │
│     │  └─ Pool fee: 0.3% (0x...)                       │
│     └─ All in CacheDB (memory)                         │
│                                                              │
│  2. Each execution:                                        │
│     ├─ REVM reads from CacheDB (instant)               │
│     ├─ No RPC calls                                     │
│     ├─ No disk access                                   │
│     └─ Pure in-memory execution                         │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 💻 CODE COMPARISON CHI TIẾT

### **TIER 2: init_account() - Fetch từ blockchain**

```rust
// FILE: src/revm_cached.rs
// TIER 2: Fetch real contract bytecode từ Ethereum

#[tokio::main]
async fn main() -> Result<()> {
    let provider = ProviderBuilder::new()
        .on_http(std::env::var("ETH_RPC_URL").unwrap().parse()?);

    // ════════════════════════════════════════════════════════
    // TIER 2 STEP 1: Initialize empty cache database
    // ════════════════════════════════════════════════════════
    let mut cache_db = init_cache_db(provider.clone());
    // cache_db = CacheDB {
    //     memory: HashMap::new(),
    //     disk_cache: cacache::new(".evm_cache/"),
    //     alloy_db: AlloyDB(provider.clone()),
    // }

    // ════════════════════════════════════════════════════════
    // TIER 2 STEP 2: Load real contract bytecode
    // ════════════════════════════════════════════════════════

    // 2A. Load V3_QUOTER contract
    println!("⏳ Fetching V3_QUOTER bytecode from blockchain...");
    init_account(V3_QUOTER_ADDR, &mut cache_db, provider.clone()).await?;
    //
    // What happens inside init_account():
    // ─────────────────────────────────────
    // 1. Check if cache has V3_QUOTER_ADDR
    //    → First time? NO (empty cache)
    //
    // 2. Call AlloyDB.get_account(V3_QUOTER_ADDR)
    //    → AlloyDB calls RPC: eth_getCode
    //    → RPC Request → Ethereum node
    //    → Blockchain response: bytecode (8000 bytes)
    //
    // 3. Store in memory:
    //    cache_db.memory[V3_QUOTER_ADDR] = Account {
    //        code: 0x60806040...,
    //        balance: 0,
    //        nonce: 0,
    //    }
    //
    // 4. Store on disk:
    //    cacache.put("evm/V3_QUOTER_ADDR", bytecode)
    //
    // Time: ~500ms (network latency)

    // 2B. Load V3_POOL_500 contract
    println!("⏳ Fetching V3_POOL_500 bytecode from blockchain...");
    init_account(V3_POOL_500_ADDR, &mut cache_db, provider.clone()).await?;
    // Time: ~500ms (network latency)

    // 2C. Load V3_POOL_3000 contract
    println!("⏳ Fetching V3_POOL_3000 bytecode from blockchain...");
    init_account(V3_POOL_3000_ADDR, &mut cache_db, provider.clone()).await?;
    // Time: ~500ms (network latency)

    // TIER 2: Total RPC calls so far = 3
    // Total time: ~1500ms (3 RPC calls)

    // ════════════════════════════════════════════════════════
    // TIER 2 STEP 3: Setup storage (in-memory)
    // ════════════════════════════════════════════════════════
    println!("💾 Injecting storage slots...");

    // Give user 100 WETH
    insert_mapping_storage_slot(
        &mut cache_db,
        WETH_ADDR,
        ME,  // User address
        U256::from_dec_str("100000000000000000").unwrap(), // 100 WETH
    );
    // What happens:
    // ─────────────
    // cache_db.storage[WETH_ADDR][slot_index] = 100 WETH
    // This is INJECTED, not from blockchain
    // Time: instant (in-memory)

    // Give pool 200k USDC
    insert_mapping_storage_slot(
        &mut cache_db,
        USDC_ADDR,
        V3_POOL_500_ADDR,
        U256::from_dec_str("200000000000").unwrap(), // 200k USDC
    );
    // Time: instant (in-memory)

    // ════════════════════════════════════════════════════════
    // TIER 2 STEP 4: Execute 100 swaps in REVM (NO RPC!)
    // ════════════════════════════════════════════════════════
    let base_fee = provider.get_gas_price().await?;  // 1 RPC call
    let t1 = measure_start();

    for (i, volume) in volumes().iter().enumerate() {
        // Build transaction
        let calldata = quote_calldata(*volume);
        let tx = build_tx(V3_QUOTER_ADDR, ME, calldata, base_fee);

        // ✅ KEY DIFFERENCE FROM TIER 1:
        // Instead of provider.call(tx) which does RPC,
        // We use revm_call() which executes in-memory!
        let response = revm_call(ME, V3_QUOTER_ADDR, calldata, &mut cache_db)?;
        //
        // What happens inside revm_call():
        // ────────────────────────────────
        // 1. Check cache_db for V3_QUOTER_ADDR bytecode
        //    → Found in memory from STEP 2!
        // 2. Create REVM execution context
        // 3. Execute EVM bytecode in-memory
        // 4. Return result (instant)
        //
        // Time: ~35ms (NO network latency!)

        let (amount_out, _, _, _, _) = decode_quote_response(&response)?;

        if i % 20 == 0 {
            println!("Vol: {:?} WETH → {:?} USDC (REVM)", volume, amount_out);
        }
    }

    let elapsed = measure_end(t1);
    println!("⏱️  TIER 2 Total: {}ms", elapsed);
    // OUTPUT: ~3500ms for 100 calls
    // + 1500ms for fetching (only once!)
    // + 1ms for gas price (1 RPC)
    // = ~5 seconds total
    //
    // But if you run again:
    // Bytecode cached on disk → No RPC calls!
    // = 3.5 seconds next time!

    Ok(())
}

// ════════════════════════════════════════════════════════
// TIER 2 KEY CHARACTERISTICS:
// ════════════════════════════════════════════════════════
// ✅ Real contracts (from blockchain)
// ✅ Real state (cached from blockchain)
// ✅ Fetch once, cache forever
// ✅ Subsequent runs: 0 RPC calls
// ✅ 99%+ accurate (same EVM)
// ⚠️ First run slower (fetch from RPC)
// ⚠️ Disk cache takes space
```

---

### **TIER 3: init_account_with_bytecode() - Inject mocked**

```rust
// FILE: src/revm_quoter.rs
// TIER 3: Use compiled bytecode from file (not from blockchain)

#[tokio::main]
async fn main() -> Result<()> {
    let provider = ProviderBuilder::new()
        .on_http(std::env::var("ETH_RPC_URL").unwrap().parse()?);

    // ════════════════════════════════════════════════════════
    // TIER 3 STEP 1: Initialize empty cache database
    // ════════════════════════════════════════════════════════
    let mut cache_db = init_cache_db(provider.clone());
    // Same as TIER 2, but we won't use AlloyDB to fetch!

    // ════════════════════════════════════════════════════════
    // TIER 3 STEP 2: Load bytecode from FILE (not RPC!)
    // ════════════════════════════════════════════════════════

    println!("📦 Loading custom quoter from bytecode file...");

    // 2A. Load CUSTOM QUOTER (not from blockchain, from compiled file)
    init_account_with_bytecode(
        CUSTOM_QUOTER_ADDR,
        &mut cache_db,
        QUOTER_BYTECODE.to_vec(),  // This is compiled hex bytecode
    );
    //
    // What happens:
    // ─────────────
    // 1. QUOTER_BYTECODE = 0x60806040... (hex string from file)
    // 2. No RPC call! Bytecode is embedded in code
    // 3. Store directly in cache_db.memory[CUSTOM_QUOTER_ADDR]
    // 4. cache_db.storage[CUSTOM_QUOTER_ADDR] = Account {
    //        code: 0x60806040...,
    //        balance: 0,
    //        nonce: 0,
    //    }
    // Time: instant (file read, no network)

    // ⚠️ IMPORTANT: This is NOT the official Uniswap quoter!
    // This is custom contract we wrote ourselves.
    // More simplified, returns data via revert trick.

    // 2B. Load mock ERC20 token (not real WETH, mocked)
    println!("📦 Loading mock ERC20 from bytecode file...");
    init_account_with_bytecode(
        WETH_ADDR,
        &mut cache_db,
        ERC20_BYTECODE.to_vec(),  // Generic ERC20 bytecode
    );
    // This is NOT real WETH contract!
    // Just a generic ERC20 implementation
    // Time: instant

    // 2C. Load mock ERC20 for USDC
    init_account_with_bytecode(
        USDC_ADDR,
        &mut cache_db,
        ERC20_BYTECODE.to_vec(),  // Same generic ERC20
    );
    // Time: instant

    // 2D. Load mock pool (if needed)
    // In this example we're using custom quoter, so no pool load needed
    // But you could do:
    // init_account_with_bytecode(V3_POOL_ADDR, &mut cache_db, POOL_BYTECODE.to_vec());

    // ════════════════════════════════════════════════════════
    // TIER 3 STEP 3: Manually inject storage values
    // ════════════════════════════════════════════════════════
    println!("💉 Injecting storage slots with custom values...");

    // 3A. Inject user WETH balance: 1000 WETH
    insert_mapping_storage_slot(
        &mut cache_db,
        WETH_ADDR,
        ME,  // User address
        U256::from_dec_str("1000000000000000000").unwrap(), // 1000 WETH
    );
    //
    // What this does:
    // ───────────────
    // Find ERC20 storage slot for balances mapping:
    // balance_mapping_slot = keccak256("balances")  // or slot 0
    // user_slot = keccak256(ME || balance_mapping_slot)
    // cache_db.storage[WETH_ADDR][user_slot] = 1000 WETH
    //
    // This is NOT from blockchain!
    // We're creating fake scenario: "user has 1000 WETH"
    // Time: instant

    // 3B. Inject pool USDC liquidity: 200k USDC
    insert_mapping_storage_slot(
        &mut cache_db,
        USDC_ADDR,
        V3_POOL_500_ADDR,
        U256::from_dec_str("200000000000").unwrap(), // 200k USDC
    );
    // Fake scenario: "pool has 200k USDC"
    // Time: instant

    // 3C. Inject pool fee setting (example)
    // If we wanted to test different fee:
    // insert_mapping_storage_slot(
    //     &mut cache_db,
    //     V3_POOL_500_ADDR,
    //     fee_slot_address,
    //     U256::from(3000),  // 0.3% fee
    // );

    // ════════════════════════════════════════════════════════
    // TIER 3 STEP 4: Execute 100 swaps in REVM (0 RPC calls!)
    // ════════════════════════════════════════════════════════

    // Get gas price (1 RPC call, but not critical)
    let base_fee = provider.get_gas_price().await?;
    let t1 = measure_start();

    for volume in volumes().iter() {
        // Build transaction for custom quoter
        let calldata = get_amount_out_calldata(*volume);
        let tx = build_tx(CUSTOM_QUOTER_ADDR, ME, calldata, base_fee);

        // ✅ TIER 3 KEY POINT:
        // Use revm_revert() instead of revm_call()
        // Why? Because custom quoter returns data via revert!
        let result = revm_revert(
            ME,
            CUSTOM_QUOTER_ADDR,
            calldata,
            &mut cache_db,
        )?;
        //
        // What happens inside revm_revert():
        // ───────────────────────────────────
        // 1. Check cache_db for CUSTOM_QUOTER bytecode
        //    → Found in memory from STEP 2!
        // 2. Create REVM execution context
        // 3. Execute bytecode
        // 4. Quoter calls swap on pool
        //    → Pool calls back to quoter (callback)
        //    → Data returned via revert reason
        // 5. Extract revert reason bytes
        // 6. Return as "result"
        //
        // Time: ~15ms (NO network latency!)
        // Speed: 3x faster than TIER 2
        // Reason: No AlloyDB, pure in-memory

        // Decode from revert reason (special for custom quoter)
        let amount_out = decode_get_amount_out_response(&result)?;

        println!("Volume: {:?} WETH -> {:?} USDC (MOCKED)", volume, amount_out);
    }

    let elapsed = measure_end(t1);
    println!("⏱️  TIER 3 Total: {}ms", elapsed);
    // OUTPUT: ~1500ms for 100 calls
    // No RPC calls (+ just 1 for gas price)
    // 30x faster than TIER 1 RPC!
    // Every run: same speed (no variance)

    Ok(())
}

// ════════════════════════════════════════════════════════
// TIER 3 KEY CHARACTERISTICS:
// ════════════════════════════════════════════════════════
// ✅ Fast (in-memory only)
// ✅ Offline (0 RPC dependency)
// ✅ Consistent (no network variance)
// ❌ Not real (mocked contracts)
// ❌ Less accurate (simplified state)
// ⚠️ Needs validation before production
// 🎯 Perfect for: backtesting, optimization, HFT
```

---

## 🔄 REVERT TRICK - TIER 3 ONLY

### **Tại sao cần revert trick?**

```
TIER 2: Official Quoter (Simple)
════════════════════════════════════════════════════════════

Function: quoteExactInputSingle(tokenIn, tokenOut, fee, amountIn)
┌─────────────────────────────────────────────────────────┐
│ Call quoter                                             │
├─────────────────────────────────────────────────────────┤
│ Input:   quoteExactInputSingle(WETH, USDC, 3000, 100)  │
│ Execute: Call pool swap, capture output                │
│ Return:  (amountOut, sqrtPrice, initializedTick, ...)  │
│ Output:  200 USDC                                       │
│                                                          │
│ Problem: Return value has LIMITED fields               │
│          What if we need more data?                    │
└─────────────────────────────────────────────────────────┘


TIER 3: Custom Quoter (Revert Trick)
════════════════════════════════════════════════════════════

Function: getAmountOut(amountIn, tokenIn, tokenOut)
┌─────────────────────────────────────────────────────────┐
│ Call custom quoter                                      │
├─────────────────────────────────────────────────────────┤
│ Input:   getAmountOut(100 WETH, WETH, USDC)            │
│                                                          │
│ Execute:                                                │
│ ├─ Setup swap parameters                              │
│ ├─ Call pool.swap()                                   │
│ ├─ Pool calls back: uniswapV3SwapCallback()           │
│ ├─ Callback returns data                              │
│ ├─ Quoter has result: 200 USDC                        │
│ └─ Quoter INTENTIONALLY REVERTS!                      │
│                                                          │
│ Revert Reason: 200 USDC (unlimited data!)             │
│                                                          │
│ Caller:                                                │
│ ├─ Catches revert                                     │
│ ├─ Extracts revert reason bytes                       │
│ ├─ Decodes: 200 USDC                                  │
│ └─ Problem solved! ✅                                 │
│                                                          │
│ Advantage: Can pass UNLIMITED data via revert!        │
│            vs limited return values                    │
└─────────────────────────────────────────────────────────┘


CODE EXAMPLE: Revert Trick
═════════════════════════════════════════════════════════

// Inside custom quoter contract (Solidity)
function getAmountOut(
    uint amountIn,
    address tokenIn,
    address tokenOut
) external {
    // Setup swap
    bytes memory payload = abi.encodeWithSelector(
        IUniswapV3Pool.swap.selector,
        // swap parameters...
    );

    // Execute swap (will callback)
    pool.swap(recipient, zeroForOne, amountIn, sqrtPriceLimit, payload);

    // If we reach here, swap succeeded
    // Now return data via REVERT!
    bytes memory result = abi.encode(
        amountOut,
        additionalData1,
        additionalData2,
        // ... unlimited fields!
    );

    // ⚠️ REVERT WITH DATA (not an error!)
    assembly {
        revert(add(result, 32), mload(result))
    }
}

// On caller side (Rust code)
let result = revm_revert(
    ME,
    CUSTOM_QUOTER_ADDR,
    calldata,
    &mut cache_db,
)?;  // ← Captures revert reason!

// Decode revert reason
let amount_out = decode_get_amount_out_response(&result)?;
// ✅ Data recovered from revert!
```

---

## ⚡ PERFORMANCE COMPARISON

### **Execution Timeline: 100 Calls**

```
TIER 2: REVM (Cached)
═══════════════════════════════════════════════════════════

Time 0ms:    Start
├─ Connect provider
│  └─ Time: 0ms

Time 0-500ms: Fetch V3_QUOTER bytecode
├─ RPC call: eth_getCode
├─ Blockchain response
├─ Cache to memory & disk
└─ Time: ~500ms

Time 500-1000ms: Fetch V3_POOL_500 bytecode
└─ Time: ~500ms

Time 1000-1500ms: Fetch V3_POOL_3000 bytecode
└─ Time: ~500ms

Time 1500-2000ms: Inject storage (in-memory)
├─ User balance: 100 WETH
├─ Pool balance: 200k USDC
└─ Time: instant (in-memory)

Time 2000-5500ms: Execute 100 REVM calls
├─ Each call: 35ms
├─ 100 calls: ~3500ms
├─ No RPC (all cached!)
└─ Time: ~3500ms

Total: ~5500ms


TIER 3: REVM (Mocked)
═══════════════════════════════════════════════════════════

Time 0ms:    Start
├─ Connect provider (optional)
│  └─ Time: 0ms

Time 0-10ms: Load custom quoter bytecode
├─ Read from QUOTER_BYTECODE constant
├─ Store in cache_db memory
└─ Time: instant

Time 10-20ms: Load mock ERC20 bytecode
├─ Read from ERC20_BYTECODE constant
├─ Store in cache_db memory
└─ Time: instant

Time 20-30ms: Inject storage
├─ User balance: 1000 WETH
├─ Pool balance: 200k USDC
├─ All in-memory
└─ Time: instant

Time 30-1500ms: Execute 100 REVM calls
├─ Each call: 15ms
├─ 100 calls: ~1500ms
├─ No RPC at all!
└─ Time: ~1500ms

Total: ~1500ms


COMPARISON:
═══════════════════════════════════════════════════════════

TIER 2: 5500ms total
├─ 3 RPC calls: ~1500ms
└─ 100 executions: ~3500ms

TIER 3: 1500ms total
├─ 0 RPC calls: 0ms ✅
└─ 100 executions: ~1500ms

Speedup: 5500ms / 1500ms = 3.7x faster than TIER 2!
         45000ms / 1500ms = 30x faster than TIER 1!
```

---

## 🎯 STATE HANDLING COMPARISON

### **How state is managed**

```
SCENARIO: User executes swap with 0.1 WETH

═══════════════════════════════════════════════════════════
TIER 2: Fetch from blockchain
═══════════════════════════════════════════════════════════

Question: Does user have 0.1 WETH?
→ Where to check?
  ├─ Cache memory: No data yet
  ├─ Cache disk: Maybe from previous run
  └─ Blockchain (AlloyDB → RPC): Yes!

Data Flow:
1. Check cache_db.memory[WETH_ADDR][user_slot]
   → Not found (not loaded yet)
2. Check AlloyDB (which queries blockchain)
   → "User WETH balance = 12.5 ETH"
3. Load into cache_db.memory[WETH_ADDR][user_slot] = 12.5 ETH
4. Execute: 12.5 - 0.1 = 12.4 ETH after swap
5. Result: ✅ Swap valid (user had enough)

Accuracy: 100% (from blockchain)
Reality: Real state at execution time


═══════════════════════════════════════════════════════════
TIER 3: Injected manually
═══════════════════════════════════════════════════════════

Question: Does user have 0.1 WETH?
→ We already injected: 1000 WETH in setup!

Data Flow:
1. Cache initialization:
   insert_mapping_storage_slot(WETH_ADDR, ME, 1000 WETH)
2. cache_db.storage[WETH_ADDR][user_slot] = 1000 WETH
3. Execute: 1000 - 0.1 = 999.9 WETH after swap
4. Result: ✅ Swap valid (injected balance is enough)

Accuracy: ✅ (matched our injection)
Reality: ⚠️ Fake state (1000 WETH is our scenario, not real)

But we can test different scenarios:
├─ User has 0.1 WETH (minimum)
├─ User has 1000 WETH (large)
├─ User has 0.05 WETH (insufficient)
└─ Test all without RPC! ✅
```

---

## 🔍 VALIDATION COMPARISON

### **How accuracy is verified**

```
TIER 2: Already validated
═══════════════════════════════════════════════════════════

TIER 2 is used AS validation!

revm_validate binary:
├─ RPC call to official quoter → 200 USDC
├─ REVM call to official quoter → 200 USDC
├─ Compare: 200 == 200? YES! ✅
├─ Result: REVM matches blockchain!
└─ Confidence: 99%+ (same EVM bytecode)


TIER 3: Needs validation
═══════════════════════════════════════════════════════════

TIER 3 uses mocked contracts, so:
├─ Custom quoter ≠ Official quoter
├─ Result might differ
└─ Need verification!

Validation Process:
1. Run revm_arbitrage (TIER 3)
   → Find: 23 profitable volumes

2. Validate with revm_validate (TIER 2)
   ├─ Pick top profitable volumes
   ├─ Compare TIER 3 result vs RPC
   ├─ If match: ✅ TIER 3 is accurate
   └─ If differ: ❌ TIER 3 mock is wrong

3. Fix mock if needed
   ├─ Adjust injected storage
   ├─ Fix contract logic
   └─ Re-run

4. Only when validated: Deploy to production!
```

---

## 🚀 USE CASE DECISION

```
Use TIER 2 when:
═════════════════════════════════════════════════════════
├─ Need production-grade accuracy
├─ Want real blockchain state
├─ Can afford 1-2 RPC calls per session
├─ Validating results
├─ Medium batch (100-10000 calls)
└─ Example: revm_validate, revm_arbitrage validation


Use TIER 3 when:
═════════════════════════════════════════════════════════
├─ Need maximum speed
├─ Testing/backtesting scenarios
├─ Offline operation required
├─ Can verify separately with TIER 2
├─ Large batch (1000-1M calls)
├─ HFT/frequent operations
└─ Example: revm_arbitrage discovery, algorithm optimization
```

---

## 📋 DETAILED COMPARISON TABLE

```
┌─────────────────────┬──────────────────────┬──────────────────────┐
│ ASPECT              │ TIER 2 (Cached)      │ TIER 3 (Mocked)      │
├─────────────────────┼──────────────────────┼──────────────────────┤
│ ARCHITECTURE        │                      │                      │
│ State Source        │ Blockchain (via RPC) │ Injected manually    │
│ Fetch Method        │ AlloyDB + caching    │ In-memory injection  │
│ Network Calls       │ 2-3 (first time)     │ 0 (totally offline)  │
│                     │                      │                      │
├─────────────────────┼──────────────────────┼──────────────────────┤
│ EXECUTION           │                      │                      │
│ Speed               │ 35ms/call            │ 15ms/call            │
│ RPC per call        │ 0 (cached)           │ 0 (offline)          │
│ Consistency         │ ✅ High              │ ✅ Very high         │
│                     │ (same cache state)   │ (same injected)      │
│                     │                      │                      │
├─────────────────────┼──────────────────────┼──────────────────────┤
│ ACCURACY            │                      │                      │
│ vs Blockchain       │ ✅ 99%+              │ ⚠️ 90-98%            │
│ Real Contracts      │ ✅ YES               │ ❌ Custom/Mocked     │
│ Real State          │ ✅ YES (cached)      │ ❌ Injected          │
│ Validation Needed   │ ❌ NO                │ ✅ YES (with TIER 2) │
│                     │                      │                      │
├─────────────────────┼──────────────────────┼──────────────────────┤
│ SETUP               │                      │                      │
│ Complexity          │ Medium               │ Complex              │
│ Time                │ ~2s (fetch)          │ ~10ms (load)         │
│ Storage             │ Disk cache (persist) │ Memory only (temp)   │
│                     │                      │                      │
├─────────────────────┼──────────────────────┼──────────────────────┤
│ USE CASES           │                      │                      │
│ Learning            │ ✅ Good              │ ❌ Not realistic     │
│ Production          │ ✅ YES (validated)   │ ⚠️ After validation  │
│ Backtesting         │ ⚠️ Possible          │ ✅ PERFECT           │
│ Optimization        │ ⚠️ Possible          │ ✅ PERFECT           │
│ HFT                 │ ❌ Too slow          │ ✅ PERFECT           │
│ Single test         │ ✅ Good              │ ✅ Good              │
│ Batch 1000+         │ ⚠️ Possible          │ ✅ PERFECT           │
│                     │                      │                      │
├─────────────────────┼──────────────────────┼──────────────────────┤
│ ADVANTAGES          │                      │                      │
│                     │ • Real contracts     │ • Fastest (15ms)     │
│                     │ • Real state         │ • Offline (0 RPC)    │
│                     │ • High accuracy      │ • Scalable (1M+)     │
│                     │ • Cached (2nd run)   │ • Deterministic      │
│                     │ • Production-safe    │ • No network issues  │
│                     │                      │ • Full control       │
│                     │                      │                      │
├─────────────────────┼──────────────────────┼──────────────────────┤
│ DISADVANTAGES       │                      │                      │
│                     │ • RPC latency (1st)  │ • Not realistic      │
│                     │ • Slower than T3     │ • Needs validation   │
│                     │ • Network dependent  │ • Mocked contracts   │
│                     │ • RPC rate limit     │ • Less accurate      │
│                     │ • Disk space needed  │ • Complex setup      │
│                     │                      │                      │
├─────────────────────┼──────────────────────┼──────────────────────┤
│ TIER 1 COMPARISON   │                      │                      │
│ vs RPC (45s)        │ 12.8x faster ⚡     │ 30x faster ⚡⚡     │
│ Perfect For         │ Production ✅        │ Development ✅       │
│                     │ Validation ✅        │ Backtesting ✅       │
│                     │ Safe Trading ✅      │ Optimization ✅      │
│                     │                      │                      │
└─────────────────────┴──────────────────────┴──────────────────────┘
```

---

## 💡 KEY TECHNICAL INSIGHTS

### **1. Cache Effectiveness**

```
TIER 2: Multi-layer caching
═════════════════════════════════════════════════════════

First run:
  blockchain (RPC) → memory (instant) → disk (persistent)
  Time: 500ms (first call)
  Time: 35ms (subsequent calls in same session)

Second run (next day):
  disk cache (load) → memory → REVM execution
  Time: 5ms (load from disk) + 35ms (execution) = 40ms
  Result: 12.5x faster than first time!

Cache key: keccak256(contract_address)
Cache storage: ~/.evm_cache/ (uses cacache library)


TIER 3: No caching needed
═════════════════════════════════════════════════════════

Every run: Same performance
├─ Load from memory (embedded in binary)
├─ Inject storage (instant)
├─ Execute
└─ Time: always 15ms (consistent!)

No disk cache → No persistence between runs
But: Not needed! Bytecode never changes (compiled)
```

### **2. Storage Slot Injection**

```
TIER 2: Storage from blockchain
═════════════════════════════════════════════════════════

ERC20 balance structure:
├─ mapping(address => uint256) public balances
│  └─ Storage slot: keccak256(account, mapping_base_slot)
│     └─ Example: keccak256(0x123...abc || 0x0)
│        = 0x789...def (unique storage slot)

When we access balances[userA]:
1. Compute slot: keccak256(userA || mapping_slot)
2. Check cache_db.storage[token_addr][computed_slot]
3. If not cached: Fetch from blockchain via AlloyDB
4. Store in cache for next access
5. REVM reads from cache


TIER 3: Storage manually injected
═════════════════════════════════════════════════════════

insert_mapping_storage_slot() function:
├─ Takes: (contract_addr, key1, value)
├─ Computes: slot = keccak256(key1 || base_slot)
├─ Sets: cache_db.storage[contract_addr][slot] = value
└─ Result: Injected state available to REVM

Example:
insert_mapping_storage_slot(
    &mut cache_db,
    WETH_ADDR,
    ME,  // key1
    U256::from(1000),  // value: 1000 WETH
);
// Internally:
// slot = keccak256(ME || 0x0)
// cache_db.storage[WETH_ADDR][slot] = 1000 WETH
```

### **3. Callback Mechanism**

```
TIER 2: Official Quoter (Simple)
═════════════════════════════════════════════════════════

quoteExactInputSingle() flow:
1. Call pool.swap(quoter_addr, ...)
2. Pool calls: quoter.uniswapV3SwapCallback(...)
3. Quoter sends tokens
4. Pool executes swap
5. Quoter returns: (amountOut, sqrtPrice, ...)

Result: Direct return value with limited fields


TIER 3: Custom Quoter (Revert Trick)
═════════════════════════════════════════════════════════

getAmountOut() flow:
1. Call pool.swap(custom_quoter_addr, ...)
2. Pool calls: quoter.uniswapV3SwapCallback(...)
3. Quoter sends tokens
4. Pool executes swap
5. Quoter intentionally REVERTS with encoded data
6. revm_revert() catches revert
7. Decode revert reason to get result

Result: Unlimited data via revert reason

Advantage: Can pass complex data structures
via revert reason (no return value limit)
```

---

## 🎓 LEARNING PATH

```
Master TIER 2 & 3 Progression:
═════════════════════════════════════════════════════════

Level 1: Understanding TIER 2
├─ Run revm_cached
├─ Observe: "3.5 seconds vs 45 seconds"
├─ Read: "AlloyDB fetches from blockchain"
└─ Understand: "Caching makes it fast"

Level 2: Validation TIER 2
├─ Run revm_validate
├─ Compare: "RPC == REVM? YES ✅"
├─ Understand: "Same EVM = same results"
└─ Confidence: "REVM is production-safe"

Level 3: TIER 3 Basics
├─ Read revm_quoter.rs code
├─ Understand: "init_account_with_bytecode()"
├─ Understand: "insert_mapping_storage_slot()"
└─ See: "No RPC calls!"

Level 4: TIER 3 Advanced
├─ Understand: "Revert trick" (data encoding)
├─ Understand: "Custom contracts"
├─ Run revm_quoter
├─ Observe: "1.5 seconds! ⚡⚡"
└─ Confidence: "Offline is possible"

Level 5: Production
├─ Run revm_arbitrage
├─ Find: "23 profitable volumes"
├─ Validate with revm_validate
├─ Deploy to production
└─ Earn: MEV profit! 💰
```

---

**Summary**: TIER 2 adalah "production-safe & fast", TIER 3 adalah "fastest & offline". Pilih TIER 2 khi cần validation, pilih TIER 3 khi cần speed.
