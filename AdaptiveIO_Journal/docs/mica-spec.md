# Mica Binary Format Specification v1.0

> **Status:** Draft — review before implementing
> **Author:** Agent (M5-T1)
> **Date:** 2026-04-13
> **Reference:** RESEARCH_PLAN.md §3, §3.5

---

## 1. Overview

Mica is a page-granular columnar format designed for selective analytical queries over NVMe storage with io_uring. Every data page is independently addressable, independently decompressible, and aligned for O_DIRECT reads.

**Design goals:**
1. Fixed-size pages enabling arithmetic page addressing (no page index needed)
2. 4096-byte alignment for O_DIRECT + io_uring
3. Per-page zone maps, Bloom filters, and bitmaps for fine-grained predicate pushdown
4. Lightweight per-page encoding (no heavyweight compression)
5. Single-pass metadata loading on file open

**Non-goals:**
- Storage efficiency (format prioritizes read performance over compactness)
- Streaming writes (writer buffers all data before writing)
- Nested/complex types (flat columns only)

---

## 2. Conventions

| Convention | Definition |
|---|---|
| Byte order | Little-endian throughout |
| Alignment | All section offsets are multiples of `page_size` (except footer) |
| `page_size` | File-level constant: 4096 or 8192 bytes |
| `rows_per_page` | File-level constant (default: 200). Uniform across all columns |
| `page_count` | `ceil(row_count / rows_per_page)` — same for all columns |
| Page addressing | Page `i` of column `j` is at `col_meta[j].data_offset + i * page_size` |
| Null sentinel | `0xFF..FF` (all-ones) in zone map min/max when page is all-null |
| String comparison | Zone map min/max use first 8 bytes, null-padded. Lexicographic on raw bytes |

---

## 3. File Layout

```
┌──────────────────────────────────────────────┐
│ File Header                                  │  page_size bytes (32B data + padding)
│   magic, version, row_count, col_count, ...  │
├──────────────────────────────────────────────┤  ← offset: page_size
│ Column Metadata Array                        │  ceil(col_count × 128 / page_size) × page_size
│   ColumnMeta[0], ColumnMeta[1], ...          │
├──────────────────────────────────────────────┤  ← page_size aligned
│ Zone Maps                                    │  All columns, contiguous per column
│   Col 0: [ZM_0, ZM_1, ..., ZM_{P-1}]        │  16 bytes/page/col. ALWAYS present.
│   Col 1: [ZM_0, ZM_1, ..., ZM_{P-1}]        │
│   ...                                        │  Padded to page_size boundary
├──────────────────────────────────────────────┤  ← page_size aligned
│ Bloom Filters (optional)                     │  Only columns with index_flags & BLOOM
│   Col A: [BloomHdr][Filter_0]...[Filter_P-1] │  Contiguous per column
│   Col B: [BloomHdr][Filter_0]...[Filter_P-1] │
│   ...                                        │  Padded to page_size boundary
├──────────────────────────────────────────────┤  ← page_size aligned
│ Per-Page Bitmaps (optional)                  │  Only columns with index_flags & BITMAP
│   Col X: [BitmapHdr][Dict][BV_0]...[BV_P-1]  │  Contiguous per column
│   Col Y: [BitmapHdr][Dict][BV_0]...[BV_P-1]  │
│   ...                                        │  Padded to page_size boundary
├──────────────────────────────────────────────┤  ← page_size aligned
│ Data Pages                                   │
│   Column 0: [Page_0][Page_1]...[Page_{P-1}]  │  page_count × page_size bytes per column
│   Column 1: [Page_0][Page_1]...[Page_{P-1}]  │
│   ...                                        │
├──────────────────────────────────────────────┤
│ Footer (32 bytes at EOF)                     │  NOT aligned — read via buffered I/O
└──────────────────────────────────────────────┘
```

**Section ordering is fixed.** The writer emits sections in this exact order.

---

## 4. File Header

**Size:** 32 bytes of data, occupying the first `page_size` bytes (zero-padded).

```
Offset  Size  Type      Field            Description
──────  ────  ────      ─────            ───────────
0       4     [u8; 4]   magic            b"MICA" = [0x4D, 0x49, 0x43, 0x41]
4       2     u16       version_major    Format major version (1)
6       2     u16       version_minor    Format minor version (0)
8       8     u64       row_count        Total rows in file
16      4     u32       column_count     Number of columns
20      4     u32       page_size        Page size in bytes (4096 or 8192)
24      4     u32       rows_per_page    Max rows per page (default: 200)
28      4     u32       page_count       ceil(row_count / rows_per_page)
```

**Bytes 32..page_size:** zero-filled.

**Validation on read:**
- `magic` must equal `b"MICA"`
- `version_major` must equal 1
- `page_size` must be 4096 or 8192
- `page_count` must equal `ceil(row_count / rows_per_page)` (or 0 if `row_count == 0`)

---

## 5. Column Metadata Array

**Starts at:** offset `page_size` (immediately after the header page).
**Entry size:** 128 bytes, fixed.
**Section size:** `ceil(column_count × 128 / page_size) × page_size` bytes.

### 5.1 ColumnMeta Entry (128 bytes)

```
Offset  Size  Type      Field              Description
──────  ────  ────      ─────              ───────────
0       64    [u8; 64]  name               Null-terminated UTF-8 column name
64      1     u8        data_type          DataType enum (§12)
65      1     u8        index_flags        Bitmask: ZONE_MAP|BLOOM|BITMAP (§14)
66      1     u8        encoding_hint      Suggested encoding (writer hint, not binding)
67      1     u8        _reserved_0        Must be 0
68      4     u32       page_count         Same as file header page_count (redundant for validation)
72      4     u32       null_count         Total nulls across all pages for this column
76      4     u32       distinct_count     Approximate distinct non-null values (for metadata)
80      8     u64       zone_map_offset    File offset to this column's zone map array
88      4     u32       zone_map_size      Byte size of this column's zone maps
92      4     u32       _reserved_1        Must be 0
96      8     u64       bloom_offset       File offset to this column's bloom section (0 if none)
104     4     u32       bloom_size         Byte size of this column's bloom section (0 if none)
108     4     u32       _reserved_2        Must be 0
112     8     u64       bitmap_offset      File offset to this column's bitmap section (0 if none)
120     4     u32       bitmap_size        Byte size of this column's bitmap section (0 if none)
124     4     u32       data_offset_lo     Lower 32 bits of data_offset (see §5.2)
```

Wait — that leaves no room for `data_offset` as u64. Let me restructure:

### 5.1 ColumnMeta Entry (128 bytes) — Revised

```
Offset  Size  Type      Field              Description
──────  ────  ────      ─────              ───────────
0       64    [u8; 64]  name               Null-terminated UTF-8 column name
64      1     u8        data_type          DataType enum (§12)
65      1     u8        index_flags        ZONE_MAP=0x01 | BLOOM=0x02 | BITMAP=0x04
66      2     u16       _reserved_0        Must be 0
68      4     u32       null_count         Total nulls across all pages
72      4     u32       distinct_count     Approximate distinct non-null values
76      4     u32       _reserved_1        Must be 0
80      8     u64       zone_map_offset    File offset → this column's zone map array
88      8     u64       bloom_offset       File offset → this column's bloom section (0 = absent)
96      8     u64       bitmap_offset      File offset → this column's bitmap section (0 = absent)
104     8     u64       data_offset        File offset → this column's first data page
112     4     u32       zone_map_size      Byte count of zone map data for this column
116     4     u32       bloom_size         Byte count of bloom data for this column (0 = absent)
120     4     u32       bitmap_size        Byte count of bitmap data for this column (0 = absent)
124     4     u32       _reserved_2        Must be 0
```

**Total: 128 bytes per column.**

### 5.2 Data Offset Calculation

With fixed `page_size` and uniform `page_count`, each column's data occupies exactly `page_count × page_size` bytes. The `data_offset` field provides the file offset to column `j`'s first page. Page `i` of column `j` is at:

```
offset = col_meta[j].data_offset + i * page_size
```

No page index is needed.

---

## 6. Zone Map Section

**Alignment:** Section starts at a `page_size`-aligned offset. Padded to `page_size` boundary at end.
**Layout:** Contiguous per column, columns in column-index order.

### 6.1 Zone Map Entry (16 bytes)

One entry per page per column.

```
Offset  Size  Type      Field       Description
──────  ────  ────      ─────       ───────────
0       8     [u8; 8]   min_value   Minimum non-null value in page (see §6.2)
8       8     [u8; 8]   max_value   Maximum non-null value in page (see §6.2)
```

**Per-column layout:**
```
zone_map_offset[j] → [Entry_0][Entry_1]...[Entry_{page_count-1}]
zone_map_size[j]   =  page_count × 16
```

### 6.2 Value Encoding in Zone Maps

Values are stored in their native representation, zero-extended to 8 bytes. Comparison must use the correct type interpretation from `data_type`.

| Data Type | Storage | Comparison |
|---|---|---|
| Int8/16/32 | Sign-extended to i64, stored as little-endian i64 | Signed integer |
| Int64 | Little-endian i64 | Signed integer |
| UInt8/16/32 | Zero-extended to u64, stored as little-endian u64 | Unsigned integer |
| UInt64 | Little-endian u64 | Unsigned integer |
| Float32 | Widened to f64, stored as little-endian f64 | IEEE 754 float |
| Float64 | Little-endian f64 | IEEE 754 float |
| Boolean | 0 (false) or 1 (true), zero-extended to u64 | Unsigned integer |
| Utf8 | First 8 bytes of string, null-padded if shorter | Lexicographic bytes |
| Binary | First 8 bytes, null-padded if shorter | Lexicographic bytes |
| Date32 | i32 days since epoch, sign-extended to i64 | Signed integer |
| TimestampMicros | Little-endian i64 (µs since epoch) | Signed integer |

**All-null pages:** Both `min_value` and `max_value` are set to `0xFF_FF_FF_FF_FF_FF_FF_FF`. The reader treats this as "page may contain any value" (never pruned by zone maps). Readers MUST check for this sentinel before comparing.

**String zone maps:** Truncation to 8 bytes means zone maps provide approximate range pruning for strings. For exact equality pruning on strings, use Bloom filters or bitmaps.

### 6.3 Zone Map Pruning

For a range predicate `column op value`:

```
// Returns true if page MAY contain qualifying rows
fn zone_map_qualifies(entry: &ZoneMapEntry, op: Op, value: T) -> bool {
    if entry.is_all_null_sentinel() { return true; }
    match op {
        Eq      => entry.min <= value && value <= entry.max,
        Lt      => entry.min < value,
        Le      => entry.min <= value,
        Gt      => entry.max > value,
        Ge      => entry.max >= value,
        Between(lo, hi) => entry.max >= lo && entry.min <= hi,
        Ne      => !(entry.min == value && entry.max == value),  // skip only if entire page is that value
    }
}
```

---

## 7. Bloom Filter Section

**Present only for columns with `index_flags & BLOOM` (0x02).**
**Alignment:** Overall section starts at `page_size`-aligned offset. Padded to `page_size` boundary.
**Layout:** Contiguous per column. Columns in column-index order among bloom-enabled columns.

### 7.1 Per-Column Bloom Layout

```
bitmap_offset[j] →
  ┌───────────────────────────────────────┐
  │ BloomHeader (16 bytes)                │
  ├───────────────────────────────────────┤
  │ Filter[0]    (bytes_per_filter bytes) │
  │ Filter[1]    (bytes_per_filter bytes) │
  │ ...                                   │
  │ Filter[P-1]  (bytes_per_filter bytes) │
  └───────────────────────────────────────┘
```

### 7.2 BloomHeader (16 bytes)

```
Offset  Size  Type   Field              Description
──────  ────  ────   ─────              ───────────
0       4     u32    bits_per_element   Bits allocated per element (default: 10)
4       2     u16    num_hashes         Number of hash functions k (default: 7)
6       2     u16    _reserved          Must be 0
8       4     u32    bytes_per_filter   ceil(bits_per_element × rows_per_page / 8)
12      4     u32    _reserved2         Must be 0
```

### 7.3 Per-Filter Layout

Each filter is a bit array of `bits_per_element × rows_per_page` bits, stored as `bytes_per_filter` bytes in little-endian bit order (bit 0 of byte 0 is the lowest-index bit).

**Byte count:** `bytes_per_filter = ceil(bits_per_element × rows_per_page / 8)`

**Default:** 10 bits/element × 200 rows = 2000 bits = 250 bytes/filter.

### 7.4 Hash Functions

Bloom filters use the **double-hashing** scheme:

```
h_i(x) = (h1(x) + i × h2(x)) mod m       for i ∈ 0..num_hashes
```

Where:
- `h1(x)` = lower 32 bits of `xxhash64(x, seed=0)`
- `h2(x)` = upper 32 bits of `xxhash64(x, seed=0)`
- `m` = `bits_per_element × rows_per_page` (total bits in filter)

**Value hashing:** Values are hashed as their little-endian byte representation:
- Fixed-size types: `type_size` bytes, little-endian
- Utf8/Binary: raw bytes (no length prefix)
- Null values are NEVER inserted into Bloom filters

### 7.5 Space Budget

| rows_per_page | bits/elem | bytes/filter | FPR |
|---|---|---|---|
| 200 | 10 | 250 | ~0.82% |
| 200 | 12 | 300 | ~0.31% |
| 200 | 8 | 200 | ~2.16% |

Formula: `FPR ≈ (1 − e^(−k/m_ratio))^k` where `m_ratio = bits_per_element`.

---

## 8. Per-Page Bitmap Section

**Present only for columns with `index_flags & BITMAP` (0x04).**
**Alignment:** Overall section starts at `page_size`-aligned offset. Padded to `page_size` boundary.
**Layout:** Contiguous per column. Columns in column-index order among bitmap-enabled columns.

### 8.1 Per-Column Bitmap Layout

```
bitmap_offset[j] →
  ┌──────────────────────────────────────┐
  │ BitmapHeader (8 bytes)               │
  ├──────────────────────────────────────┤
  │ Dictionary (variable size)           │  Sorted array of distinct non-null values
  ├──────────────────────────────────────┤
  │ BitVec[0]    (bitvec_size bytes)     │  Which dict values appear in page 0
  │ BitVec[1]    (bitvec_size bytes)     │
  │ ...                                  │
  │ BitVec[P-1]  (bitvec_size bytes)     │
  └──────────────────────────────────────┘
```

### 8.2 BitmapHeader (8 bytes)

```
Offset  Size  Type   Field         Description
──────  ────  ────   ─────         ───────────
0       4     u32    cardinality   Number of distinct non-null values in dictionary
4       4     u32    dict_size     Byte size of dictionary section
```

### 8.3 Dictionary

Sorted array of all distinct non-null values across the entire column. Sorted in natural order for the data type (ascending signed/unsigned integer, lexicographic for strings).

**Fixed-size types (Int8..UInt64, Float32/64, Boolean, Date32, TimestampMicros):**
```
[value_0 (type_size bytes)][value_1]...[value_{cardinality-1}]
dict_size = cardinality × type_size
```

**Variable-size types (Utf8, Binary):**
```
[u32 offsets[cardinality + 1]]   — offset array (arrow-style, offsets[i] = byte offset of string i)
[u8  data[offsets[cardinality]]] — concatenated string bytes
dict_size = (cardinality + 1) × 4 + offsets[cardinality]
```

### 8.4 Per-Page BitVec

Each bitvec has `cardinality` bits. Bit `k` is set (1) if dictionary value `k` appears in at least one non-null row of that page.

```
bitvec_size = ceil(cardinality / 8)
```

Little-endian bit order: bit 0 of byte 0 corresponds to dictionary index 0.

### 8.5 Bitmap Pruning

For an equality predicate `column = X`:
1. Binary search `X` in the sorted dictionary → index `k` (or "not found" → skip ALL pages)
2. For each page, check if `bitvec[page][k]` is set
3. Page qualifies only if bit is set

For `column IN (X, Y, Z)`: OR the bitvec checks for each value.

**Exact:** Zero false positives. If the bit is not set, the value is guaranteed absent from that page.

### 8.6 Space Budget

| Cardinality | bitvec_size | Per page | Notes |
|---|---|---|---|
| 2 (Boolean) | 1 byte | 1 B | Minimal |
| 10 | 2 bytes | 2 B | Very compact |
| 50 | 7 bytes | 7 B | Typical low-card |
| 100 | 13 bytes | 13 B | Threshold |

Dictionary overhead: `cardinality × type_size` (one-time, shared across pages).

---

## 9. Data Pages

Each data page occupies exactly `page_size` bytes.

### 9.1 Page Layout

```
┌──────────────────────────────────────────────┐
│ PageHeader (16 bytes)                        │
├──────────────────────────────────────────────┤
│ Null Bitmap (if flags & HAS_NULLS)           │  ceil(row_count / 8) bytes
├──────────────────────────────────────────────┤
│ Encoded Data (data_size bytes)               │  Format depends on encoding (§10)
├──────────────────────────────────────────────┤
│ Zero padding to page_size                    │
└──────────────────────────────────────────────┘
```

**Constraint:** `16 + null_bitmap_size + data_size ≤ page_size`. The writer MUST ensure this. If data would overflow, the writer MUST choose a more compact encoding or reduce `rows_per_page` for the entire file.

### 9.2 PageHeader (16 bytes)

```
Offset  Size  Type   Field              Description
──────  ────  ────   ─────              ───────────
0       2     u16    row_count          Rows in this page (≤ rows_per_page; last page may be smaller)
2       1     u8     encoding           Encoding enum: PLAIN=0, FOR=1, DELTA=2, DICT=3
3       1     u8     flags              Bit 0: HAS_NULLS (null bitmap present after header)
4       4     u32    data_size          Bytes of payload after header (null bitmap + encoded data)
8       4     u32    raw_value_size     Bytes needed after full decode (for buffer allocation)
12      4     u32    checksum           CRC32C of the data_size bytes following the header
```

### 9.3 Null Bitmap

Present only if `flags & 0x01` (`HAS_NULLS`).

- Size: `ceil(row_count / 8)` bytes
- Little-endian bit order: bit `k` of byte `k/8` represents row `k`
- Bit = 1 → value is NOT null (valid)
- Bit = 0 → value is null
- Encoded data contains only non-null values (in row order, skipping nulls)
- Non-null count = `popcount(null_bitmap)` = number of values in encoded data

### 9.4 Page Data Size Budget

For `page_size = 8192` and `rows_per_page = 200`:
- Available after header: `8192 − 16 = 8176` bytes
- With null bitmap: `8176 − ceil(200/8) = 8176 − 25 = 8151` bytes for encoded data
- PLAIN Int64: `200 × 8 = 1600` bytes → fits easily
- PLAIN Utf8 (avg 40 bytes): `200 × 40 + 201 × 4 = 8804` bytes → **exceeds 8176**

**Overflow handling:** If PLAIN encoding exceeds budget, the writer MUST either:
1. Use DICT encoding (which is smaller for repetitive data), or
2. Reduce `rows_per_page` globally to ensure all columns fit

The writer determines `rows_per_page` during a pre-scan of the data to guarantee no page overflows.

---

## 10. Encoding Formats

Each page independently selects an encoding. The writer MAY use different encodings for different pages of the same column.

### 10.1 PLAIN (encoding = 0)

Raw values in native little-endian representation.

**Fixed-size types (Boolean, Int8..UInt64, Float32/64, Date32, TimestampMicros):**
```
[value_0 (type_size bytes)][value_1]...[value_{N-1}]
```
Where `N` = number of non-null values.

**Boolean:** Packed bit array, `ceil(N / 8)` bytes. Bit `k` of byte `k/8` = value of row `k`.

**Variable-size types (Utf8, Binary):**
```
[u32 offsets[N + 1]]  — Arrow-style offset array
[u8  data[offsets[N]]] — Concatenated byte data
```
`offsets[0] = 0`. `offsets[i]` = byte offset where string `i` begins in the data buffer. `offsets[N]` = total data bytes.

### 10.2 FOR — Frame of Reference (encoding = 1)

**Valid for:** Integer types only (Int8..UInt64, Date32, TimestampMicros).

Stores values as fixed-width deltas from a base value.

```
[base_value (8 bytes, type-widened to i64/u64)]  — minimum value in page
[bit_width  (1 byte, u8)]                        — bits per delta (0..64)
[packed_deltas]                                   — bit-packed (value - base_value)
```

**Packed deltas:** `N` values, each `bit_width` bits, packed little-endian. Total bytes: `ceil(N × bit_width / 8)`.

**bit_width** = `ceil(log2(max_value - min_value + 1))`. If all values are identical, `bit_width = 0` and `packed_deltas` is empty (0 bytes).

**Decode:** `value[i] = base_value + unpack(packed_deltas, i, bit_width)`

### 10.3 DELTA (encoding = 2)

**Valid for:** Integer types only (Int8..UInt64, Date32, TimestampMicros). Most effective on sorted or monotonic data.

Stores zigzag-encoded deltas between consecutive values.

```
[first_value (8 bytes, type-widened to i64/u64)]  — first value
[bit_width   (1 byte, u8)]                        — bits per zigzag delta
[packed_deltas]                                    — bit-packed zigzag(value[i] - value[i-1])
```

**Zigzag encoding:** `zigzag(n) = (n << 1) ^ (n >> 63)` (for i64). Maps signed integers to unsigned: 0→0, -1→1, 1→2, -2→3, ...

**N-1 deltas** (first value is stored separately). Total packed bytes: `ceil((N-1) × bit_width / 8)`.

**bit_width** = `ceil(log2(max_zigzag_delta + 1))`, computed over all `N-1` deltas.

**Decode:**
```
values[0] = first_value
for i in 1..N:
    delta = zagzig(unpack(packed_deltas, i-1, bit_width))
    values[i] = values[i-1] + delta
```
Where `zagzig(z) = (z >> 1) ^ -(z & 1)` is the inverse.

### 10.4 DICT — Dictionary (encoding = 3)

**Valid for:** All types. Most effective on low-to-medium cardinality data.

```
[dict_entry_count (4 bytes, u32)]      — number of distinct values in page dictionary
[dictionary_data]                       — encoded dictionary entries
[index_bit_width  (1 byte, u8)]        — bits per index (ceil(log2(dict_entry_count)))
[packed_indices]                        — bit-packed indices into dictionary
```

**Dictionary data format:**

Fixed-size types:
```
[value_0 (type_size bytes)][value_1]...[value_{dict_entry_count-1}]
```

Variable-size types (Utf8, Binary):
```
[u32 offsets[dict_entry_count + 1]]
[u8  data[offsets[dict_entry_count]]]
```

**Packed indices:** `N` values, each `index_bit_width` bits. Total bytes: `ceil(N × index_bit_width / 8)`.

**Decode:** `value[i] = dictionary[unpack(packed_indices, i, index_bit_width)]`

### 10.5 Encoding Selection Guidance

The writer selects encoding per page per column. Recommended heuristic:

| Condition | Encoding | Rationale |
|---|---|---|
| Boolean type | PLAIN (bit-packed) | Already 1 bit/value |
| `page_distinct_count ≤ 0.5 × row_count` | DICT | Dictionary pays off when repetition is high |
| Sorted/monotonic integers | DELTA | Deltas are small → narrow bit_width |
| `max - min` fits in few bits | FOR | Compact range → narrow bit_width |
| Fallback | PLAIN | No overhead, simplest decode |

Priority: DICT > DELTA > FOR > PLAIN (try in order, pick first that fits in page budget).

---

## 11. Footer

**Size:** 32 bytes, located at the last 32 bytes of the file.
**NOT aligned** to `page_size` — read via buffered I/O.

```
Offset  Size  Type      Field              Description
──────  ────  ────      ─────              ───────────
0       8     u64       column_meta_offset File offset to Column Metadata Array (§5)
8       8     u64       row_count          Total rows (must match header)
16      4     u32       column_count       Number of columns (must match header)
20      4     u32       page_size          Page size in bytes (must match header)
24      4     u32       checksum           CRC32C of footer bytes 0..24
28      4     [u8; 4]   magic              b"MICA" = [0x4D, 0x49, 0x43, 0x41]
```

**Read procedure:** Reader reads the last 32 bytes of the file. Validates `magic`, then `checksum`. Extracts `column_meta_offset` to locate metadata.

**Why duplicate header fields?** The footer is the entry point for reading. Duplicating `row_count`, `column_count`, and `page_size` allows the reader to allocate buffers and validate before reading the full header.

---

## 12. Data Types

```
Enum Value  Type              Native Size  Arrow Equivalent
──────────  ────              ───────────  ────────────────
0           Boolean           1 bit        Boolean
1           Int8              1 byte       Int8
2           Int16             2 bytes      Int16
3           Int32             4 bytes      Int32
4           Int64             8 bytes      Int64
5           UInt8             1 byte       UInt8
6           UInt16            2 bytes      UInt16
7           UInt32            4 bytes      UInt32
8           UInt64            8 bytes      UInt64
9           Float32           4 bytes      Float32
10          Float64           8 bytes      Float64
11          Utf8              variable     Utf8
12          Binary            variable     Binary
13          Date32            4 bytes      Date32
14          TimestampMicros   8 bytes      Timestamp(µs, None)
```

`type_size(dt)` for fixed-size types: `{Boolean: 1, Int8: 1, Int16: 2, Int32: 4, Int64: 8, UInt8: 1, UInt16: 2, UInt32: 4, UInt64: 8, Float32: 4, Float64: 8, Date32: 4, TimestampMicros: 8}`.

Note: Boolean `type_size = 1` for zone maps and dictionary entries. Within PLAIN-encoded pages, booleans are bit-packed.

---

## 13. Alignment & O_DIRECT Rules

1. **File header** starts at offset 0, occupies `page_size` bytes.
2. **Column metadata array** starts at offset `page_size`, padded to next `page_size` boundary.
3. **Zone map section** starts at next `page_size` boundary after column metadata.
4. **Bloom filter section** (if any bloom-enabled columns) starts at next `page_size` boundary.
5. **Bitmap section** (if any bitmap-enabled columns) starts at next `page_size` boundary.
6. **Data pages** start at next `page_size` boundary. Each page is exactly `page_size` bytes.
7. **Footer** is the last 32 bytes of the file (NOT aligned).

**O_DIRECT read requirements (Linux):**
- Buffer address must be aligned to logical block size (typically 512 bytes)
- Read offset must be aligned to logical block size
- Read size must be a multiple of logical block size

Since `page_size` (4096 or 8192) is a multiple of 512, all aligned sections satisfy O_DIRECT constraints. The reader allocates `page_size`-aligned buffers for O_DIRECT reads.

**Footer exception:** The footer is small (32 bytes) and read once via buffered I/O. It does not need O_DIRECT alignment.

---

## 14. Index Selection (`index_flags`)

### 14.1 Bit Definitions

```
Bit   Mask   Flag      Meaning
───   ────   ────      ───────
0     0x01   ZONE_MAP  Zone maps present (ALWAYS set)
1     0x02   BLOOM     Per-page Bloom filters present
2     0x04   BITMAP    Per-page bitmaps present
3-7   —      Reserved  Must be 0
```

ZONE_MAP is always set. BLOOM and BITMAP are mutually exclusive per column (a column has at most one of {BLOOM, BITMAP}).

### 14.2 Write-Time Selection Algorithm

```
per column:
    index_flags = ZONE_MAP                       // always

    if column.data_type not in {Utf8, Binary, Boolean}:
        // Bloom/bitmap only for types where equality makes sense and values are hashable
        if distinct_count(column) <= CARDINALITY_THRESHOLD:    // default: 100
            index_flags |= BITMAP
        else:
            index_flags |= BLOOM
    elif column.data_type in {Utf8, Binary}:
        if distinct_count(column) <= CARDINALITY_THRESHOLD:
            index_flags |= BITMAP
        else:
            index_flags |= BLOOM
    // Boolean: cardinality ≤ 2, so BITMAP is always selected if enabled
```

**`CARDINALITY_THRESHOLD`** default: 100. Configurable via writer config.

### 14.3 Read-Time Predicate Routing

```
fn evaluate_predicate(pred, col_meta) -> PageBitmap {
    match pred.op {
        // Range → zone maps only
        Gt | Lt | Ge | Le | Between =>
            evaluate_zone_maps(pred, col_meta),

        // Equality/IN → best available index
        Eq | In => {
            if col_meta.index_flags & BITMAP:
                evaluate_bitmap(pred, col_meta)     // exact, zero FPR
            else if col_meta.index_flags & BLOOM:
                evaluate_bloom(pred, col_meta)       // probabilistic
            else:
                evaluate_zone_maps(pred, col_meta)   // fallback
        }

        // NotEq: complement
        Ne => !evaluate_predicate(pred.as_eq(), col_meta),
    }
}

// Compound: AND-compose across columns
fn evaluate_compound(preds, metadata) -> PageBitmap {
    result = PageBitmap::all_set(page_count)
    for pred in preds:
        result &= evaluate_predicate(pred, metadata.columns[pred.col_idx])
    result
}
```

---

## 15. Space Overhead Analysis

### 15.1 Per-Component Overhead

For a dataset with `R` total rows, `C` columns, `page_size = P`, `rows_per_page = RPP`:
- `page_count = ceil(R / RPP)`

| Component | Size | Formula |
|---|---|---|
| File header | `P` bytes | Fixed |
| Column metadata | `ceil(C × 128 / P) × P` bytes | ~`128C` |
| Zone maps | `page_count × C × 16` bytes | Always present |
| Bloom filters | `Σ (page_count × bytes_per_filter)` per bloom column | ~`250 × page_count` per col |
| Bitmaps | `Σ (8 + dict_size + page_count × bitvec_size)` per bitmap column | Small |
| Data pages | `page_count × C × P` bytes | Dominant term |
| Footer | 32 bytes | Fixed |

### 15.2 ClickBench Example (100M rows, 105 columns)

Parameters: `page_size = 8192`, `rows_per_page = 200`, `page_count = 500,000`.

| Component | Size | % of data pages |
|---|---|---|
| File header | 8 KiB | ~0% |
| Column metadata | 16 KiB | ~0% |
| Zone maps (all 105 cols) | 840 MB | 0.20% |
| Bloom filters (est. 50 cols) | 6.25 GB | 1.47% |
| Bitmaps (est. 20 cols, avg card=50) | 70 MB | 0.02% |
| **Data pages (105 cols)** | **409 GB** | **100%** |
| Footer | 32 B | ~0% |

**Data page utilization:** Varies widely by column type.
- Int64 PLAIN: 1600 / 8192 = 19.5% utilization
- Int8 PLAIN: 200 / 8192 = 2.4% utilization (worst case)
- Int64 FOR (8-bit deltas): 209 / 8192 = 2.5% utilization
- String avg 30B: 6804 / 8192 = 83% utilization

### 15.3 Design Trade-off: Space vs. Read Performance

The fixed-page model trades storage efficiency for:
1. **Arithmetic page addressing** — no page index, O(1) random access
2. **O_DIRECT alignment** — every page read is a single aligned I/O
3. **Simple buffer management** — all buffers are `page_size` bytes

**Mitigation strategies:**
- Use `page_size = 4096` for datasets with many small-type columns (halves padding waste)
- Increase `rows_per_page` to amortize page overhead (degrades pruning; see §3.5 birthday analysis)
- Accept the trade-off: format targets selective reads where only 1-5% of pages are read

### 15.4 Comparison: page_size = 4096 vs. 8192

| Metric | 4096 | 8192 |
|---|---|---|
| Data pages total (ClickBench) | ~205 GB | ~409 GB |
| Avg utilization (Int64 PLAIN) | 39% | 19.5% |
| String overflow risk | Higher | Lower |
| io_uring sweet spot | ✓ (4K = NVMe block) | Slightly above |

**Recommendation:** `page_size = 4096` for most workloads. Use 8192 only when many columns have variable-length data that approaches 4K per page.

---

## 16. Read Procedure

1. **Read footer** (last 32 bytes, buffered I/O). Validate magic + checksum.
2. **Read column metadata** (`column_meta_offset`, size = `column_count × 128`, may require 1–2 aligned reads).
3. **Load zone maps** (one read per column: `zone_map_offset[j]`, `zone_map_size[j]` bytes).
4. **Load bloom filters** (one read per bloom column: `bloom_offset[j]`, `bloom_size[j]` bytes).
5. **Load bitmaps** (one read per bitmap column: `bitmap_offset[j]`, `bitmap_size[j]` bytes).
6. **Evaluate predicates** → `PageBitmap` of qualifying pages (§14.3).
7. **Read qualifying pages** via batched io_uring (O_DIRECT, `page_size`-aligned reads).
8. **Decode pages** independently per column.

Steps 3–5 can be batched: all metadata reads submitted as a single io_uring batch.

---

## 17. Write Procedure

1. **Pre-scan data:** Compute per-column statistics (distinct count, min/max, null count). Determine `rows_per_page` ensuring all columns fit within `page_size`. Determine index selection per column.
2. **Partition rows into pages** of `rows_per_page` rows each (last page may be smaller).
3. **Encode pages:** For each page of each column, select encoding and produce encoded bytes + null bitmap. Verify `header + null_bitmap + encoded_data ≤ page_size`.
4. **Compute zone maps:** Per page per column, extract min/max of non-null values.
5. **Compute Bloom filters:** Per page per bloom-enabled column, insert all non-null values.
6. **Compute bitmaps:** Build global dictionary per bitmap-enabled column. Per page, compute presence bitvec.
7. **Compute offsets:** Calculate file offsets for all sections based on sizes.
8. **Write file** in section order (§3):
   - File header (padded to `page_size`)
   - Column metadata array (padded to `page_size`)
   - Zone maps (padded to `page_size`)
   - Bloom filters (padded to `page_size`)
   - Bitmaps (padded to `page_size`)
   - Data pages (column by column, `page_count × page_size` each)
   - Footer (32 bytes, no padding)

---

## Appendix A: Bit Packing

All bit-packed data (FOR deltas, DELTA zigzag deltas, DICT indices, Bloom filters, bitmaps) uses **little-endian bit packing**:

- Values are packed starting from the least significant bit of the first byte.
- When a value crosses a byte boundary, the lower bits go in the current byte and the upper bits go in the next byte.

```
Example: 3-bit values [5, 3, 7, 1] packed into bytes:

Value 5 = 101 → bits 0-2 of byte 0
Value 3 = 011 → bits 3-5 of byte 0
Value 7 = 111 → bits 6-7 of byte 0, bit 0 of byte 1
Value 1 = 001 → bits 1-3 of byte 1

Byte 0: 11_011_101 = 0xDD
Byte 1: xxxx_001_1 = 0x03 (upper bits zero)
```

## Appendix B: CRC32C

All checksums use CRC32C (Castagnoli), matching the hardware-accelerated `crc32c` instruction on x86-64. Libraries: `crc32c` crate in Rust.

## Appendix C: Open Design Questions

> These items require review before finalizing the spec.

1. **`page_size` default:** 4096 recommended (§15.4) but RESEARCH_PLAN says 8192. Which?
2. **String overflow:** With `rows_per_page = 200` and `page_size = 4096`, long strings may not fit. Should the writer reduce `rows_per_page` per-file, or fall back to 8192?
3. **Variable-size pages (future):** A page offset index would eliminate padding waste but adds metadata and complexity. Worth pursuing?
4. **Compression layer:** Should the format support optional LZ4/Zstd on top of encoding? Adds decode latency but could dramatically reduce storage.
5. **Null zone map representation:** Current sentinel (`0xFF..FF`) is a valid value for unsigned types. Alternative: add a 1-byte flags field to ZoneMapEntry (20 bytes total) or use page header's null info.
