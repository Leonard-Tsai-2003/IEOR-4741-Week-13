# HW13: Build Optimization (Compiler Flags Only)

## Environment & macOS Profiling Setup
* **System:** macOS (`arm64-apple-darwin25.6.0`, Apple Silicon), Apple Clang 16.0.0 (`g++ -std=c++20`).
* **Tools (`perf` substitute):** Used `/usr/bin/time -l` for `instructions retired` and `cycles elapsed`, plus `-Rpass=loop-vectorize` / `-Rpass-missed=loop-vectorize` for SIMD remarks.
* **Workload:** $3{,}000 \times 2^{16} = 196{,}608{,}000$ total inner-loop operations (`kernel.cpp` unmodified).

---

## Best Flag Sets & Speedup
* **Fastest Overall:** `g++ -std=c++20 -O3 -mcpu=native -funroll-loops -ffast-math kernel.cpp -o fast_math`
  * **`60.189 ms`** vs. **`1100.833 ms`** baseline (**`18.29x` speedup**; **`1.07` cycles/op** vs. `19.12` baseline).
* **Fastest Bit-Exact (IEEE-754):** `g++ -std=c++20 -O3 kernel.cpp -o test_o3`
  * **`189.369 ms`** vs. **`1113.993 ms`** baseline (**`5.88x` speedup**; exact checksum match).

---

## Timing Matrix & Checksums

| Configuration | Flags (`g++ -std=c++20 ...`) | `time_ms` | Checksum | Speedup |
| :--- | :--- | ---: | ---: | ---: |
| **Baseline** | `-O0` | `1113.993` | `151299320.088155` | `1.00x` |
| **O2** | `-O2` | `190.287` | `151299320.088155` | `5.85x` |
| **O3** | `-O3` | `189.369` | `151299320.088155` | `5.88x` |
| **Arch + Unroll** | `-O3 -march=native -funroll-loops` | `200.626` | `151299320.088155` | `5.55x` |
| **CPU + Unroll** | `-O3 -mcpu=native -funroll-loops` | `196.271` | `151299320.088155` | `5.68x` |
| **LTO** | `-O3 -mcpu=native -funroll-loops -flto` | `192.929` | `151299320.088155` | `5.77x` |
| **PGO** | `-O3 -mcpu=native -funroll-loops -fprofile-instr-use` | `192.356` | `151299320.088155` | `5.79x` |
| **Fast-Math (Best)** | `-O3 -mcpu=native -funroll-loops -ffast-math` | **`60.189`** | `151299381.089299` | **`18.29x`** |

*(Note: Timed runs for Baseline, PGO, and Fast-Math taken from `/usr/bin/time -l` measurements.)*

---

## Hardware Counters (`cycles/op` & Instructions)

| Metric | Baseline (`-O0`) | IEEE-Strict + PGO | Best (`-ffast-math`) |
| :--- | ---: | ---: | ---: |
| **Wall Time (`time_ms`)** | `1100.833 ms` | `192.356 ms` | **`60.189 ms`** |
| **Instructions Retired** | `16,768,322,070` | `2,569,900,133` | **`910,698,766`** |
| **Instructions / Op** | `85.29` | `13.07` | **`4.63`** |
| **Cycles Elapsed** | `3,759,887,177` | `671,125,529` | **`211,195,211`** |
| **Cycles / Op** | **`19.12`** | **`3.41`** | **`1.07`** |

---

## Vectorizer Remarks & Flag Analysis
* **SIMD Remarks (`-Rpass=loop-vectorize`):**
  * Without `-ffast-math`: `remark: loop not vectorized` (strict IEEE-754 prevents reordering `double acc` addition and `std::sqrt` `errno` semantics).
  * With `-ffast-math`: `remark: vectorized loop (vectorization width: 4, interleaved count: 4)` (16 elements processed per ARM NEON vector iteration).
* **`-O2` / `-O3` (`~5.88x`):** Eliminates `-O0` stack spills/reloads and inlines `std::vector` indexing, cutting instructions/op from `85.29` to `13.07`.
* **`-mcpu=native` & `-funroll-loops`:** Tunes scheduling for Apple Silicon `arm64`, though scalar `sqrt` latency bottlenecks throughput until SIMD is enabled.
* **`-flto` & PGO:** Minimal extra gain (`~192 ms`) because `kernel()` is already `static` in a single file, and Clang already lowers the ~50/50 `if (x > 0.0f)` branch to predicated `fcsel` instructions at `-O3`.
* **`-ffast-math` Trade-off:** Allows associative FP reassociation and drops `NaN`/`errno` checks. This changes parallel accumulation order, shifting the checksum from `151299320.088155` to `151299381.089299` (a relative difference of $4.03 \times 10^{-7}$, or `0.000040%`).

---

## Single Change That Helped Most
Enabling **`-ffast-math`** alongside `-O3 -mcpu=native -funroll-loops` delivered the largest performance breakthrough, cutting runtime from `196.271 ms` to `60.189 ms` (`3.26x` faster than the best scalar build and `18.29x` faster than `-O0`). Under strict IEEE-754 rules, Clang could not reorder the `double acc` reduction or vectorize `std::sqrt(x)`, leaving execution scalar at `3.41` cycles/op. Relaxing floating-point associativity and `errno` constraints allowed Clang to emit 4-wide ARM NEON SIMD instructions interleaved 4x—computing 16 iterations in parallel per step—which reduced retired instructions to `4.63` instructions/op and throughput to **`1.07` cycles/op**.