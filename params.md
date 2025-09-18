
## 1) Encoding & Capacity

> **Encoding model**: **K = 2¹² = 4096** original rows, **encoding factor 1:3** ⇒ parity **3K = 12288**, so **N = 16384** rows per blob. **Row size** is chosen per blob (multiple of 64 B) and capped to keep **max blob size = 128 MiB**.

| Parameter           |                  Default / Formula | Derived / Notes                                                                                      |
| ------------------- |-----------------------------------:|------------------------------------------------------------------------------------------------------|
| `original_rows (K)` |                           **4096** | Fixed **2¹²** originals.                                                                             |
| `encoding_factor`   |                            **1:3** | Parity is **3×** originals ⇒ **parity\_rows = 12288**.                                               |
| `total_rows (N)`    |                          **16384** | `N = 4K`.                                                                                            |
| `row_size`          |        **auto**; **64 B × n**, n∈ℕ | **Multiple of 64 B**. Chosen per blob: `row_size = ceil(original_length / K)` then round up to 64 B. |
| `row_size_min`      |                           **64 B** | Minimum legal row size.                                                                              |
| `row_size_max`      |                         **32 KiB** | To cap `max_blob_size` at **128 MiB**: `K * row_size_max = 4096 * 32 KiB`.                           |
| `max_blob_size`     |                        **128 MiB** | `K * row_size_max`.                                                                                  |
| `min_capacity`      |                        **256 KiB** | `K * row_size_min = 4096 * 64 B`. Small blobs are zero-padded to row bounds.                         |
| `val_set_size`      |                            **100** | Planning default; runtime uses actual (from `valset_height`).                                        |
| `rows_per_val`      | `≈ ceil(K / V * 1/3)` (even split) | With V=100 ⇒ **124 or 125** rows/validator (first `r=84` get 125; others 124).                       |
| `rlc_orig_size`     |                        **256 KiB** | `16 * N = 16 * 16384` bytes of GF128 coeffs.                                                         |
| `retention_ttl`                 |                               24 h |                                                                                                      |         |                               |
| **Throughput**                  |                         \~50 MiB/s | Configurable.                                                                                        |         |                               |

**Server limits**
* **Throughput**: `val_throughput ≈ 50 MiB/s` (configurable).

    * Per-FSP **request size ≈ rows\_assigned × row\_size**.
      Examples (V=100, \~163–164 rows/FSP):

        * `blob_size = 8 MiB`  → **\~0.5 MiB** per request ⇒ \~200 RPS at 50 MiB/s. Total network throughput ~1 GiB/s across 100 FSPs.
        * `blob_size = 32 MiB` → **\~1.2 MiB** per request ⇒ \~41.6 RPS at 50 MiB/s. Total network throughput ~1.3 GiB/s across 100 FSPs.
        * `blob_size = 128 MiB` → **\~4 MiB** per request ⇒ \~12.4 RPS at 50 MiB/s. Total network throughput ~1.6 GiB/s across 100 FSPs.

---

## 2) Gas Interface (summary; authoritative logic in `x/fibre`)

```
blob_gas = rows * row_size(blob_size) * gas_per_blob_byte
```

* `rows = N = 16384` (constant); `row_size(blob_size)` is the per‑blob chosen size after rounding.

**FIX:** Specify **max tolerated clock skew** for PP `creation_timestamp` checks (suggest ±3 minutes) and state it here for cross‑component consistency.

**FIX:** Add a constant set for permitted `row_size` values to make conformance tests trivial (e.g., generate from `ROW_SIZE_MIN`..`ROW_SIZE_MAX`).



**FIX:** Document an RPC for **fee estimation** given `blob_size` + `gas_per_blob_byte` to guide clients before constructing PPs.

