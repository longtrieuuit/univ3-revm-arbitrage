# ⚡ QUICK REFERENCE - 3 TIER COMPARISON

## 🚀 Chạy Lệnh Nhanh

```bash
# TIER 1: RPC Direct
cargo run --bin eth_call_one --release    # 1 call, 500ms
cargo run --bin eth_call --release        # 100 calls, 45s
cargo run --bin anvil --release           # Local fork, 5s

# TIER 2: REVM
cargo run --bin revm --release            # Learn caching, 5s
cargo run --bin revm_cached --release     # Production, 3.5s ⚡
cargo run --bin revm_validate --release   # Verify accuracy, 500ms ✅

# TIER 3: Mock
cargo run --bin revm_quoter --release     # Learn mocking, 1.5s ⚡⚡
cargo run --bin revm_arbitrage --release  # Find MEV, 1.5s 🚀
```

---

## 📊 Tốc Độ So Sánh (100 gọi)

| TIER | Thời gian | Nhanh hơn RPC | Dùng RPC |
|------|-----------|--------------|---------|
| **1: eth_call** | **45s** | **1x** (baseline) | **100 gọi** ❌ CHẬM |
| **2: revm_cached** | **3.5s** | **12.8x** ⚡ | **2-3 gọi** ✅ |
| **3: revm_arbitrage** | **1.5s** | **30x** ⚡⚡ | **0 gọi** 🟢 |

---

## ✅/❌ Quick Checklist

| Tiêu chí | TIER 1 | TIER 2 | TIER 3 |
|---------|--------|--------|--------|
| Học contract? | ✅ | ❓ | ❌ |
| Production-safe? | ❌ | ✅ | ⚠️ |
| Offline? | ❌ | ❌ | ✅ |
| Thực tế? | ✅ | ✅ | ⚠️ |
| Nhanh? | ❌ | ✅ | ✅ |

---

## 🎯 Lựa Chọn Dùng

```
Bạn muốn?                  → Dùng TIER
─────────────────────────────────────
Học cách hoạt động         → TIER 1 (eth_call)
Kiểm chứng accuracy        → TIER 2 (revm_validate)
Production validation      → TIER 2 (revm_cached)
Tối ưu hóa thuật toán      → TIER 3 (revm_quoter)
Tìm MEV & trade            → TIER 3 (revm_arbitrage)
```

---

## 🔄 Workflow Khuyên

```
1. eth_call_one    → ABI format check
2. eth_call        → Batch test
3. revm_cached     → See speedup! (12.8x)
4. revm_validate   → Accuracy check ✅
5. revm_arbitrage  → Find MEV! 🎉
```

---

## ❓ Tại sao 8 binary không dùng config?

**Config file:** `cargo run -- --strategy=revm` ❌
- Binary to (load tất cả)
- Code lộn xộn (if-else)
- Khó bảo trì

**8 Binary:** `cargo run --bin revm_cached` ✅
- Binary nhỏ (focused)
- Code rõ ràng
- Dễ so sánh

---

## 📁 File Tài Liệu

```
TIER_COMPARISON_VI.md         ← Chi tiết hoàn chỉnh (tiếng Việt)
TIER_VISUAL_COMPARISON.txt    ← Diagrams & flowcharts
QUICK_REFERENCE.md            ← File này (tra cứu nhanh)
```

---

## 🎓 Mục Đích Mỗi Binary

### **TIER 1: Learn & Debug**
- `eth_call_one` - 1 call ABI test
- `eth_call` - 100 RPC calls baseline
- `anvil` - Local testing

### **TIER 2: Production & Validate**
- `revm` - Caching mechanism
- `revm_cached` - Optimized production ⭐
- `revm_validate` - Accuracy verification ✅

### **TIER 3: Optimize & Trade**
- `revm_quoter` - Mocking technique
- `revm_arbitrage` - MEV discovery 🚀

---

## 💡 Key Insights

```
1. TIER 1 = Reference (100% accurate, slow)
2. TIER 2 = Production (validated, fast)
3. TIER 3 = Optimized (offline, fastest)

Progression: Easy → Realistic → Fast
```

---

## 🚨 Important Notes

⚠️ **Never skip revm_validate!**
```
TIER 3 simulation = Fast but needs verification
TIER 2 validation = Confirm TIER 3 accuracy
Then production = Safe to deploy
```

⚠️ **RPC Dependency**
```
TIER 1: Always needs RPC
TIER 2: Needs RPC first time (cached after)
TIER 3: Never needs RPC (100% offline)
```

---

## 📞 Troubleshooting

**Q: Tại sao TIER 3 kết quả khác TIER 1?**
```
A: Mock ≠ Real. Dùng revm_validate (TIER 2) để verify.
   Nếu match → TIER 3 OK. Nếu khác → Fix mock.
```

**Q: TIER 2 vẫn cần RPC?**
```
A: Chỉ lần đầu (fetch bytecode). Sau đó cached.
   Lần sau chạy → 0 RPC calls (nếu cache đủ).
```

**Q: Nên dùng TIER nào cho production?**
```
A: TIER 3 (revm_arbitrage) để tìm MEV
   + TIER 2 (revm_validate) để verify nếu cần
   + On-chain execution khi có profit
```

---

**Last Updated:** 2025-11-25
**Branch:** `claude/project-flow-diagram-01RkyLGASqMWjhwYJG4xbihf`
