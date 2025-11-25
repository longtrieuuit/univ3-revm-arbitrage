# Phân Tích Dự Án: Uniswap V3 REVM Arbitrage

## Tổng Quan Dự Án

Dự án này là một hệ thống tính toán cơ hội arbitrage MEV (Maximal Extractable Value) trên Uniswap V3 sử dụng REVM (Rust Ethereum Virtual Machine). Dự án so sánh hiệu suất giữa eth_call truyền thống và REVM simulation để tìm kiếm cơ hội arbitrage giữa các liquidity pools.

---

## 1. Kiến Trúc Tổng Thể

```mermaid
graph TB
    subgraph "External Layer"
        ETH[Ethereum RPC Node]
        POOLS[(Uniswap V3 Pools)]
    end

    subgraph "Application Layer"
        BINS[Binary Executables]

        subgraph "8 Binaries"
            EC1[eth_call_one]
            EC[eth_call]
            ANV[anvil]
            RVM[revm]
            RVMC[revm_cached]
            RVMQ[revm_quoter]
            RVMV[revm_validate]
            RVMA[revm_arbitrage]
        end
    end

    subgraph "Core Layer"
        SRC[Source Module]

        subgraph "Components"
            ABI[ABI Module]
            ACT[Actors Module]
            HLP[Helpers Module]
        end
    end

    subgraph "Data Layer"
        CACHE[CacheDB]
        DISK[Disk Cache]
    end

    ETH -->|RPC Calls| BINS
    POOLS -->|State Data| BINS
    BINS --> SRC
    SRC --> ABI
    SRC --> ACT
    SRC --> HLP
    HLP --> CACHE
    CACHE --> DISK

    style RVMA fill:#ff6b6b
    style RVMQ fill:#4ecdc4
    style CACHE fill:#ffe66d
```

---

## 2. Luồng Dữ Liệu Chính - Main Data Flow

```mermaid
sequenceDiagram
    participant User
    participant Binary
    participant Provider
    participant CacheDB
    participant REVM
    participant ETH as Ethereum Node

    User->>Binary: Run binary (cargo run)
    Binary->>Provider: Initialize HTTP provider
    Provider->>ETH: Connect to RPC

    Binary->>CacheDB: init_cache_db()
    CacheDB-->>Binary: CacheDB instance

    Binary->>Provider: Fetch contract bytecode
    Provider->>ETH: get_code_at()
    ETH-->>Provider: Bytecode
    Provider-->>Binary: Bytecode

    Binary->>CacheDB: Cache bytecode locally
    CacheDB->>Disk: Write to .evm_cache/

    Binary->>CacheDB: init_account()
    Binary->>CacheDB: Setup mocked balances

    loop For each volume
        Binary->>REVM: Execute simulation
        REVM->>CacheDB: Read state
        CacheDB-->>REVM: Return cached state
        REVM-->>Binary: Simulation result
        Binary->>User: Display profit/loss
    end
```

---

## 3. Luồng Chi Tiết: Arbitrage Detection Flow

```mermaid
flowchart TD
    START([Start]) --> INIT[Initialize Provider & CacheDB]
    INIT --> LOAD[Load Contract Bytecode]

    LOAD --> CACHE{Check Cache}
    CACHE -->|Hit| LOADCACHE[Load from .evm_cache/]
    CACHE -->|Miss| FETCHRPC[Fetch from RPC]
    FETCHRPC --> SAVECACHE[Save to cache]
    LOADCACHE --> SETUP
    SAVECACHE --> SETUP

    SETUP[Setup Accounts & Balances] --> MOCK[Mock ERC20 Tokens]
    MOCK --> DEPLOY[Deploy Custom Quoter]

    DEPLOY --> VOLS[Generate Volume Array]
    VOLS --> LOOP{For each volume}

    LOOP -->|Next| SWAP1[Simulate Swap 1:<br/>WETH -> USDC<br/>via Pool 500]
    SWAP1 --> DECODE1[Decode USDC Amount]

    DECODE1 --> SWAP2[Simulate Swap 2:<br/>USDC -> WETH<br/>via Pool 3000]
    SWAP2 --> DECODE2[Decode WETH Amount]

    DECODE2 --> CALC{WETH Out > WETH In?}
    CALC -->|Yes| PROFIT[Calculate Profit]
    CALC -->|No| NOPROFIT[No Arbitrage]

    PROFIT --> DISPLAY1[Display Profit]
    NOPROFIT --> DISPLAY2[Display Loss]

    DISPLAY1 --> LOOP
    DISPLAY2 --> LOOP

    LOOP -->|Done| END([End])

    style PROFIT fill:#90ee90
    style NOPROFIT fill:#ffcccb
    style DEPLOY fill:#87ceeb
```

---

## 4. Các Module Chính - Core Modules

### 4.1 Source Module Structure

```mermaid
graph LR
    subgraph "source/"
        MOD[mod.rs]

        subgraph "Sub-modules"
            ABI[abi.rs<br/>Smart Contract ABIs]
            ACT[actors.rs<br/>Contract Addresses]
            HLP[helpers.rs<br/>Utility Functions]
        end

        MOD --> ABI
        MOD --> ACT
        MOD --> HLP
    end

    subgraph "Functions in helpers.rs"
        HLP --> INIT[init_cache_db]
        HLP --> ACC[init_account]
        HLP --> CALL[revm_call]
        HLP --> REV[revm_revert]
        HLP --> SLOT[insert_mapping_storage_slot]
        HLP --> VOL[volumes]
        HLP --> TX[build_tx]
    end

    subgraph "Data in actors.rs"
        ACT --> WETH[WETH_ADDR]
        ACT --> USDC[USDC_ADDR]
        ACT --> POOL5[V3_POOL_500_ADDR]
        ACT --> POOL3[V3_POOL_3000_ADDR]
        ACT --> QUO[V3_QUOTER_ADDR]
        ACT --> CQUO[CUSTOM_QUOTER_ADDR]
    end

    subgraph "Functions in abi.rs"
        ABI --> ENC1[get_amount_out_calldata]
        ABI --> ENC2[quote_calldata]
        ABI --> DEC1[decode_get_amount_out_response]
        ABI --> DEC2[decode_quote_response]
    end
```

---

## 5. So Sánh Các Binary Executables

```mermaid
graph TB
    subgraph "Performance Comparison"
        direction TB

        ETH_CALL[eth_call<br/>---<br/>Uses: Ethereum RPC<br/>Speed: Slow<br/>Cost: Network calls]

        REVM_SIM[revm<br/>---<br/>Uses: REVM Simulation<br/>Speed: Fast<br/>Cost: No network]

        CACHED[revm_cached<br/>---<br/>Uses: REVM + Cache<br/>Speed: Fastest<br/>Cost: Disk I/O only]
    end

    subgraph "Testing & Validation"
        direction TB

        ETH_ONE[eth_call_one<br/>---<br/>Single test call]

        ANVIL[anvil<br/>---<br/>Local testnet<br/>integration]

        VALID[revm_validate<br/>---<br/>Compare eth_call<br/>vs REVM results]
    end

    subgraph "Advanced Features"
        direction TB

        QUOTER[revm_quoter<br/>---<br/>Custom quoter<br/>contract testing]

        ARB[revm_arbitrage<br/>---<br/>Full arbitrage<br/>detection]
    end

    style CACHED fill:#90ee90
    style ARB fill:#ff6b6b
    style VALID fill:#87ceeb
```

---

## 6. Luồng Xử Lý REVM Simulation

```mermaid
sequenceDiagram
    participant App as Application
    participant Cache as CacheDB
    participant REVM as REVM Engine
    participant State as EVM State

    App->>Cache: revm_call(from, to, calldata)
    Cache->>REVM: Build EVM Context

    REVM->>REVM: Setup Transaction
    Note over REVM: tx.caller = from<br/>tx.kind = Call(to)<br/>tx.data = calldata<br/>tx.value = 0

    REVM->>REVM: build_mainnet()
    REVM->>State: Load cached state
    State-->>REVM: Account info & bytecode

    REVM->>REVM: Execute transaction
    Note over REVM: Run EVM bytecode<br/>without network calls

    REVM-->>REVM: Get execution result

    alt Success
        REVM-->>Cache: Return output bytes
        Cache-->>App: Success result
    else Revert
        REVM-->>Cache: Return revert data
        Cache-->>App: Revert result
    else Error
        REVM-->>Cache: Return error
        Cache-->>App: Execution failed
    end
```

---

## 7. Chi Tiết: Arbitrage Calculation Logic

```mermaid
flowchart LR
    subgraph "Input"
        VOL[Volume Array<br/>0.01 ETH increments]
    end

    subgraph "Pool 500 (0.05% fee)"
        P5[WETH/USDC Pool]
        SWAP1[Swap WETH -> USDC]
    end

    subgraph "Pool 3000 (0.3% fee)"
        P3[USDC/WETH Pool]
        SWAP2[Swap USDC -> WETH]
    end

    subgraph "Calculation"
        CALC[Compare:<br/>WETH Out vs WETH In]
        PROFIT{Profit > 0?}
    end

    subgraph "Output"
        YES[Display Profit]
        NO[Display Loss]
    end

    VOL --> SWAP1
    SWAP1 -->|USDC Amount| SWAP2
    SWAP2 -->|WETH Amount| CALC
    CALC --> PROFIT
    PROFIT -->|Yes| YES
    PROFIT -->|No| NO

    style YES fill:#90ee90
    style NO fill:#ffcccb
    style P5 fill:#ffe66d
    style P3 fill:#87ceeb
```

---

## 8. Caching Strategy

```mermaid
graph TB
    subgraph "Cache Flow"
        REQ[Request Bytecode]

        CHECK{Check Local Cache}

        HIT[Cache Hit]
        MISS[Cache Miss]

        DISK[Read from<br/>.evm_cache/]
        RPC[Fetch from RPC]

        SAVE[Save to Cache]

        USE[Use Bytecode]
    end

    REQ --> CHECK
    CHECK -->|Found| HIT
    CHECK -->|Not Found| MISS

    HIT --> DISK
    MISS --> RPC

    RPC --> SAVE
    SAVE --> USE
    DISK --> USE

    style HIT fill:#90ee90
    style MISS fill:#ffcccb
    style SAVE fill:#87ceeb
```

**Cache Key Format**: `bytecode-{address}`

**Cache Location**: `.evm_cache/`

**Benefit**: Giảm số lượng RPC calls, tăng tốc độ thực thi đáng kể

---

## 9. State Mocking Strategy

```mermaid
graph LR
    subgraph "Real Contracts"
        REAL_WETH[Real WETH]
        REAL_USDC[Real USDC]
        REAL_POOL5[Real Pool 500]
        REAL_POOL3[Real Pool 3000]
    end

    subgraph "Mocked State"
        MOCK_ERC20[Generic ERC20<br/>Bytecode]
        MOCK_BAL[MAX Balance / 2]
        CUSTOM_Q[Custom Quoter]
    end

    subgraph "CacheDB"
        STATE[EVM State]
    end

    REAL_WETH -.->|Replaced by| MOCK_ERC20
    REAL_USDC -.->|Replaced by| MOCK_ERC20

    MOCK_ERC20 --> STATE
    MOCK_BAL --> STATE

    REAL_POOL5 --> STATE
    REAL_POOL3 --> STATE
    CUSTOM_Q --> STATE

    style MOCK_ERC20 fill:#ffe66d
    style MOCK_BAL fill:#ffe66d
    style CUSTOM_Q fill:#87ceeb
```

**Lý do Mock**:
- ERC20 tokens: Đảm bảo có đủ balance để test
- Pools: Giữ nguyên logic thực của Uniswap V3
- Custom Quoter: Deploy contract tùy chỉnh để tính toán hiệu quả hơn

---

## 10. Dependencies & Tech Stack

```mermaid
graph TB
    subgraph "Main Dependencies"
        ALLOY[Alloy 0.14<br/>Ethereum library]
        REVM_LIB[REVM 23.0.0<br/>EVM simulation]
        TOKIO[Tokio 1.40<br/>Async runtime]
        CACACHE[cacache 13.0<br/>Disk caching]
        ANYHOW[anyhow 1.0<br/>Error handling]
    end

    subgraph "Features"
        ALLOY --> FULL[full features]
        ALLOY --> NODE[node-bindings]
        REVM_LIB --> ALLOYDB[alloydb feature]
        TOKIO --> RT[tokio-runtime]
        CACACHE --> MMAP[mmap]
    end

    style REVM_LIB fill:#ff6b6b
    style ALLOY fill:#4ecdc4
    style TOKIO fill:#ffe66d
```

---

## 11. Performance Metrics

### Benchmark Comparison

| Method | Speed | Network Cost | Use Case |
|--------|-------|--------------|----------|
| **eth_call** | Slow (100-500ms/call) | High | Production validation |
| **revm** | Fast (1-5ms/call) | Low (initial fetch) | Development testing |
| **revm_cached** | Fastest (<1ms/call) | None (after cache) | High-frequency simulation |

### Volume Testing Strategy

```mermaid
graph LR
    START[0 ETH] --> V1[0.01 ETH]
    V1 --> V2[0.02 ETH]
    V2 --> V3[0.03 ETH]
    V3 --> DOTS[...]
    DOTS --> END[0.1 ETH]

    style START fill:#90ee90
    style END fill:#ff6b6b
```

**Volume Generation**: `volumes(U256::ZERO, ONE_ETHER / 10, 100)`
- From: 0 ETH
- To: 0.1 ETH
- Steps: 100 increments

---

## 12. Key Addresses (Ethereum Mainnet)

```mermaid
graph TB
    subgraph "Token Addresses"
        WETH["WETH<br/>0xc02aaa39b223fe8d0a0e5c4f27ead9083c756cc2"]
        USDC["USDC<br/>0xa0b86991c6218b36c1d19d4a2e9eb0ce3606eb48"]
    end

    subgraph "Uniswap V3 Contracts"
        POOL5["Pool 500 (0.05%)<br/>0x88e6A0c2dDD26FEEb64F039a2c41296FcB3f5640"]
        POOL3["Pool 3000 (0.3%)<br/>0x8ad599c3A0ff1De082011EFDDc58f1908eb6e6D8"]
        QUOTER["Official Quoter<br/>0x61fFE014bA17989E743c5F6cB21bF9697530B21e"]
    end

    subgraph "Custom Contracts"
        CQUOTER["Custom Quoter<br/>0xA5C381211A406b48A073E954e6949B0D49506bc0"]
        ME["Test Account<br/>0x0000000000000000000000000000000000000001"]
    end

    style WETH fill:#ff6b6b
    style USDC fill:#4ecdc4
    style CQUOTER fill:#ffe66d
```

---

## 13. Execution Examples

### Example 1: Run Arbitrage Detection

```bash
source .env && cargo run --bin revm_arbitrage --release
```

**Output**:
```
100000000000000000 WETH -> USDC 372506163498 -> WETH 99518267727624337
100000000000000000 WETH -> USDC 372506163498 -> WETH 99518267727624337
No profit.
```

### Example 2: Run Performance Comparison

```bash
source .env && cargo run --bin revm_validate --release
```

**Output**:
```
10000000000000000 WETH -> USDC REVM 37250616349 ETH_CALL 37250616349
20000000000000000 WETH -> USDC REVM 74501232698 ETH_CALL 74501232698
✓ All results match
```

---

## 14. Error Handling & Edge Cases

```mermaid
flowchart TD
    START([Execution]) --> TRY{Try Execute}

    TRY -->|Success| SUCCESS[Return Result]
    TRY -->|Revert| REVERT[Handle Revert]
    TRY -->|Error| ERROR[Handle Error]

    REVERT --> DECODE{Can Decode?}
    DECODE -->|Yes| EXTRACT[Extract Revert Data]
    DECODE -->|No| PANIC[Panic: Should never happen]

    ERROR --> LOG[Log Error]
    LOG --> RETURN[Return Err]

    EXTRACT --> SUCCESS
    SUCCESS --> END([End])
    PANIC --> END
    RETURN --> END

    style SUCCESS fill:#90ee90
    style ERROR fill:#ffcccb
    style PANIC fill:#ff6b6b
```

---

## 15. Future Improvements

```mermaid
mindmap
    root((Improvements))
        Performance
            Parallel execution
            Better caching strategy
            State pruning
        Features
            Multi-pool arbitrage
            Flash loan integration
            Gas optimization
        Monitoring
            Real-time alerts
            Profit tracking
            Success rate metrics
        Infrastructure
            Database persistence
            Web dashboard
            API endpoints
```

---

## Kết Luận

Dự án này là một ví dụ xuất sắc về việc sử dụng REVM để:

1. **Tăng tốc độ**: Giảm thời gian thực thi từ hàng trăm ms xuống còn <1ms
2. **Giảm chi phí**: Loại bỏ network calls sau lần đầu tiên
3. **Tăng độ chính xác**: So sánh kết quả giữa eth_call và REVM để đảm bảo tính đúng đắn
4. **Dễ dàng test**: Có thể mock state và test nhiều scenarios nhanh chóng

### Tech Stack Summary

- **Language**: Rust 1.83.0
- **EVM Simulation**: REVM 23.0.0
- **Ethereum Library**: Alloy 0.14
- **Async Runtime**: Tokio 1.40
- **Caching**: cacache 13.0

### Key Files

- `src/revm_arbitrage.rs` - Main arbitrage logic
- `src/source/helpers.rs` - Core utility functions
- `src/source/actors.rs` - Contract addresses
- `src/source/abi.rs` - ABI encoding/decoding
