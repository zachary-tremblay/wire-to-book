# SPEC.md — Wire-to-Book

Technical specification. This document is the contract. If the RTL and this document disagree, one of them is a bug — decide which, then fix it. Do not let them drift.

---

## 0. Conventions

- **Language:** SystemVerilog-2012 synthesizable subset. `logic` everywhere; `always_ff` / `always_comb` only. No `reg`/`wire`, no latches, no initial blocks in RTL.
- **Reset:** synchronous, active-high, named `rst`. Single clock domain, named `clk`. No CDC anywhere in v1.
- **Endianness:** all multi-byte fields on the wire are **big-endian (network byte order)**.
- **Parameters:** defined once in `rtl/feed_pkg.sv`. Nothing hard-codes a width.
- **No backpressure.** Every module accepts one byte per cycle unconditionally. There is no `ready` signal. This is a simplification and it is documented as such.

---

## 1. Parameters (`rtl/feed_pkg.sv`)

```systemverilog
package feed_pkg;
  parameter int PRICE_W      = 16;   // price in integer ticks
  parameter int QTY_W        = 16;   // shares
  parameter int ORDER_ID_W   = 32;
  parameter int N_LEVELS     = 8;    // price levels per side
  parameter int ORDER_TBL_SZ = 256;  // direct-mapped order store entries
  parameter int ORDER_IDX_W  = 8;    // $clog2(ORDER_TBL_SZ)

  parameter logic [15:0] ETHERTYPE_IPV4 = 16'h0800;
  parameter logic [7:0]  IP_PROTO_UDP   = 8'd17;
  parameter logic [15:0] FEED_UDP_PORT  = 16'd31337;

  typedef enum logic [1:0] { SIDE_BID = 2'd0, SIDE_ASK = 2'd1 } side_e;
  typedef enum logic [1:0] { OP_ADD, OP_REDUCE, OP_DELETE, OP_NOP } op_e;

  typedef struct packed {
    op_e                    op;
    side_e                  side;
    logic [PRICE_W-1:0]     price;
    logic [QTY_W-1:0]       qty;      // for OP_REDUCE: the amount to remove
  } book_op_t;
endpackage
```

---

## 2. Byte-stream interface

Every stage-to-stage connection uses the same four signals. No `ready`.

| Signal | Width | Meaning |
|---|---|---|
| `data` | 8 | payload byte |
| `valid` | 1 | `data` is meaningful this cycle |
| `sof` | 1 | first byte of this unit (asserted with `valid`) |
| `eof` | 1 | last byte of this unit (asserted with `valid`) |

A one-byte unit asserts `sof` and `eof` in the same cycle. Downstream modules must tolerate this.

---

## 3. Network front end

### 3.1 `eth_rx.sv`

**In:** raw Ethernet frame byte stream (destination MAC first). Assume no preamble, no FCS — the pcap gives frames starting at the destination MAC.

Consumes 14 bytes: dst MAC (6), src MAC (6), ethertype (2).

**Out:**
- Byte stream of the ethertype payload (`sof` on the first payload byte, `eof` on the frame's last byte).
- `ethertype_ok` — high for the duration of the frame if ethertype == `0x0800`.

If ethertype ≠ IPv4, the frame's remaining bytes are consumed and **no payload is emitted**. Increment a `drop_non_ipv4` counter.

### 3.2 `ipv4_rx.sv`

**In:** IPv4 packet byte stream.

Consumes the 20-byte header. **Assume IHL == 5** (no options). If IHL ≠ 5, drop the packet and increment `drop_bad_ihl`. This is a documented limitation; the generator never produces options.

Verify the **header checksum**: 16-bit one's-complement sum of the ten header halfwords (with the checksum field itself treated as zero), then one's-complement of the result — must equal zero when summing all ten halfwords as-is. Implement as an adder tree with end-around carry, computed over the 20 buffered header bytes; the verdict is available before the first payload byte is emitted.

**Out:**
- Byte stream of the IP payload.
- `checksum_ok` (registered, valid from the first payload byte).
- `protocol` (8 bits).

If `protocol != 17` or `!checksum_ok`, drop the payload and increment `drop_bad_ip`.

### 3.3 `udp_rx.sv`

**In:** UDP datagram byte stream.

Consumes the 8-byte header: src port (2), dst port (2), length (2), checksum (2). **The UDP checksum is not verified** — it is legal for it to be zero in IPv4 and the generator emits zero. Documented limitation.

Drop the datagram unless `dst_port == FEED_UDP_PORT`; increment `drop_bad_port`.

**Out:** byte stream of the UDP payload, `sof` on the first payload byte, `eof` on the last.

### 3.4 `frame_parser_top.sv`

Chains `eth_rx → ipv4_rx → udp_rx`. Exposes the UDP payload byte stream plus the four drop counters.

**This module is Weekend 1's deliverable and must synthesize and pass `tb_parser` on its own.**

---

## 4. Message format (FROZEN — do not change after Aug 1)

A UDP payload contains **one or more 16-byte messages**, back to back. Payload length is always a multiple of 16.

| Offset | Size | Field | Notes |
|---|---|---|---|
| 0 | 1 | `msg_type` | ASCII: `'A'`=0x41, `'X'`=0x58, `'E'`=0x45, `'D'`=0x44 |
| 1 | 1 | `side` | ASCII: `'B'`=0x42 bid, `'S'`=0x53 ask |
| 2 | 4 | `order_id` | uint32, big-endian |
| 6 | 2 | `price` | uint16, big-endian, integer ticks |
| 8 | 2 | `qty` | uint16, big-endian, shares |
| 10 | 6 | reserved | must be zero |

### Field usage by message type

| Type | Meaning | Fields used | Fields ignored |
|---|---|---|---|
| `A` | Add order | `side`, `order_id`, `price`, `qty` | — |
| `X` | Cancel (partial reduce) | `order_id`, `qty` | `side`, `price` (set to 0 on the wire) |
| `E` | Execute (fill) | `order_id`, `qty` | `side`, `price` (set to 0 on the wire) |
| `D` | Delete order | `order_id` | `side`, `price`, `qty` (set to 0 on the wire) |

**This is deliberate and load-bearing.** Real ITCH cancel/execute messages carry only an order ID and a share count — the side and price must be recovered by *looking the order up*. That is what forces `order_store` to exist and to matter. Do not "simplify" by putting price and side on the wire for `X`/`E`/`D`.

`X` and `E` are treated identically for book maintenance in v1 (an execute is a reduce). They are kept as distinct types because a v2 with trade prints would need to tell them apart.

---

## 5. Book pipeline

### 5.1 `itch_decode.sv`

**In:** UDP payload byte stream.

Byte-counter FSM (0–15). Shifts bytes into a 16-byte register. On byte 15, emits the decoded message.

**Out:**
```systemverilog
logic                    msg_valid;
logic [7:0]              msg_type;
side_e                   msg_side;
logic [ORDER_ID_W-1:0]   msg_order_id;
logic [PRICE_W-1:0]      msg_price;
logic [QTY_W-1:0]        msg_qty;
```
Emits one message per 16 bytes. If a payload ends mid-message (`eof` on a byte where `count != 15`), discard the partial message and increment `drop_partial_msg`.

Unknown `msg_type` → increment `drop_bad_type`, emit nothing.

### 5.2 `order_store.sv`

A **direct-mapped** table, `ORDER_TBL_SZ` (256) entries, indexed by `order_id[ORDER_IDX_W-1:0]`. Inferred as a single-port BRAM (Cyclone V M10K).

**Entry:** `{ valid: 1, side: 1, price: PRICE_W, qty: QTY_W }` — 34 bits × 256.

**This is not a CAM.** It is a deliberate, documented tradeoff:
- **Collision:** two live orders whose IDs share the low 8 bits map to the same slot. The generator constrains order IDs so that never happens (≤256 live orders, IDs allocated from a free-list of distinct low bytes).
- On a would-be collision (an `A` targeting a slot that is already `valid`), assert a sticky `err_collision` flag and drop the message. **In a passing test run this flag must be zero.** It exists to prove the constraint holds, not to be tolerated.
- A real design would use a CAM or a hash table with chaining. Say so in the interview. Knowing the limitation is worth more than not having it.

**Behavior:**

| Message | Lookup | Table write | Emitted `book_op_t` |
|---|---|---|---|
| `A` | assert slot is free | write `{1, side, price, qty}` | `{OP_ADD, side, price, qty}` |
| `X` / `E` | read slot | `qty -= msg_qty`; if result ≤ 0, clear `valid` | `{OP_REDUCE, stored_side, stored_price, msg_qty}` |
| `D` | read slot | clear `valid` | `{OP_REDUCE, stored_side, stored_price, stored_qty}` |

`X`/`E`/`D` on an invalid slot → assert sticky `err_unknown_order`, drop, emit `OP_NOP`. **Must be zero in a passing run.**

`X`/`E` where `msg_qty > stored_qty` → clamp to `stored_qty` (removes the whole order) and assert sticky `err_overcancel`. The generator must never produce this. **Must be zero in a passing run.**

**Out:** `book_op_valid`, `book_op` (`book_op_t`).

### 5.3 `bbo.sv`

Two register arrays of `N_LEVELS` (8) entries each — one for bids, one for asks.

**Level:** `{ valid: 1, price: PRICE_W, qty: QTY_W }`.

**On `OP_ADD`:** parallel-match `price` against all 8 levels of the given side.
- Hit → `qty += op.qty`.
- Miss → allocate the first invalid level (priority encoder), set `{1, price, qty}`.
- Miss with no free level → assert sticky `err_level_overflow`, drop. **The generator constrains prices to ≤8 distinct values per side. This flag must be zero in a passing run.**

**On `OP_REDUCE`:** parallel-match `price` on the given side.
- Hit → `qty -= op.qty`; if `qty` reaches 0, clear `valid` (freeing the level for reallocation).
- Miss → assert sticky `err_reduce_miss`. **Must be zero in a passing run.**

**BBO resolution (combinational, single cycle):**
- `best_bid` = **maximum** `price` among bid levels with `valid && qty > 0`.
- `best_ask` = **minimum** `price` among ask levels with `valid && qty > 0`.
- Implement each as a **3-stage binary comparator tree** over 8 candidates (8 → 4 → 2 → 1). Invalid/zero-qty levels are neutralized by forcing their comparison key to `0` (bids) or `{PRICE_W{1'b1}}` (asks) before entering the tree.
- If no level qualifies on a side, assert `bid_empty` / `ask_empty` and leave the price output at its neutral value.

**This comparator tree is the interesting hardware in the whole project.** It is also where the entire latency story lives. Do not write it as a `for` loop with a sequential accumulator and hope the synthesizer figures it out — write the tree explicitly, and check the Quartus timing report to confirm it did not become the critical path.

**Out:**
```systemverilog
logic                 bbo_valid;     // pulses one cycle after any book op that changed the book
logic [PRICE_W-1:0]   best_bid_px;
logic [QTY_W-1:0]     best_bid_qty;  // aggregate qty at that level
logic                 bid_empty;
logic [PRICE_W-1:0]   best_ask_px;
logic [QTY_W-1:0]     best_ask_qty;
logic                 ask_empty;
```

`bbo_valid` pulses on **every** applied book op, not only on ops that move the top of book. This keeps the equivalence check in the testbench simple (compare on every op) rather than requiring the golden model to also model "did the top change."

### 5.4 `feed_top.sv`

Chains `frame_parser_top → itch_decode → order_store → bbo`. Exposes the BBO outputs, every drop counter, and every sticky error flag.

---

## 6. Latency definition

**Wire-to-BBO latency** = number of clock cycles from the cycle in which the **final byte of a message** is accepted at `feed_top`'s input (i.e. `valid` high on the 16th byte of that message's payload slot) to the cycle in which `bbo_valid` pulses for that message.

Measure it in the testbench by timestamping both events. It is a fixed constant (there is no backpressure and no variable-length path), so report the exact number, not a distribution. Report it in cycles **and** in nanoseconds at the achieved Fmax.

This single number is the headline result of the project. Do not fudge it, and do not quote a number the testbench did not measure.

---

## 7. Verification

### 7.1 Golden model — `tb/book_model.cpp`

Pure C++, no Verilator dependency. Deliberately boring:

```cpp
struct Order { uint8_t side; uint16_t price; uint16_t qty; };
std::unordered_map<uint32_t, Order> orders;
std::map<uint16_t, uint32_t> bids;   // price -> aggregate qty
std::map<uint16_t, uint32_t> asks;
```

`apply(msg)` mutates the maps per the semantics in §4. `best_bid()` = `bids.rbegin()` skipping zero-qty; `best_ask()` = `asks.begin()` skipping zero-qty. Erase levels that hit zero.

**Write this before any book RTL exists.** Get it printing correct BBOs against a hand-checked 20-message sequence first.

### 7.2 Stimulus — `gen/pcap_gen.py`

Emits a `.pcap` of Ethernet/IPv4/UDP frames. Constraints — these are what keep every sticky error flag at zero:

- **Order IDs** allocated from a free-list of 256 distinct low-byte values; an ID is returned to the free list only when the order is fully removed. (Guarantees no `err_collision`.)
- **Prices** drawn from exactly 8 distinct tick values per side, disjoint between sides, with all bids strictly below all asks. (Guarantees no `err_level_overflow`, and no crossed book.)
- **`X`/`E`/`D` only ever reference a live order ID**, and `X`/`E` qty never exceeds the order's remaining qty. (Guarantees no `err_unknown_order`, no `err_overcancel`.)
- **Messages per UDP payload:** vary between 1 and 8, to exercise the decoder's multi-message path.
- **Directed cases** that must appear at least once each: book empty on one side; a level driven to exactly zero and then re-added at the same price; a full-8-level book on both sides; a `D` on the only order at the best price (top-of-book moves); a `D` on an order behind the best price (top of book does *not* move).

Also emit a companion plaintext `.msgs` file listing every message in order — the C++ testbench reads this to drive the golden model, so the model is never re-parsing the pcap.

Two runs are required:
- `smoke.pcap` — ~50 messages, hand-checkable, used for bring-up.
- `soak.pcap` — **100,000 messages**, constrained-random, seeded, reproducible. This is the headline number.

### 7.3 Testbenches

**`tb/tb_parser.cpp`** (Weekend 1) — drives raw frame bytes into `frame_parser_top`, collects the emitted payload, and diffs it against a C++ reference parser that walks the same headers. Also drives deliberately malformed frames (bad IP checksum, wrong ethertype, wrong UDP port) and asserts the correct drop counter increments and that **no payload is emitted**.

**`tb/tb_book.cpp`** (Weekend 2) — the full chain. Drives `soak.pcap` one byte per cycle. On every `bbo_valid`, applies the corresponding message to the golden model and asserts:

```
rtl.best_bid_px  == model.best_bid().price
rtl.best_bid_qty == model.best_bid().qty
rtl.bid_empty    == model.bids_empty()
   ... and the same four for the ask side
```

A single mismatch → dump the waveform, print the message index and the last 10 messages, and fail non-zero.

---

## 8. Acceptance criteria

The project is done when **all eight** hold:

1. `tb_parser` passes on `smoke.pcap` and on the malformed-frame directed cases.
2. `tb_book` passes on `smoke.pcap`.
3. `tb_book` passes on `soak.pcap` — 100,000 messages, zero BBO mismatches.
4. Every sticky error flag (`err_collision`, `err_unknown_order`, `err_overcancel`, `err_level_overflow`, `err_reduce_miss`) reads **zero** after the soak run.
5. Every directed case in §7.2 is confirmed present in the soak stimulus (assert on a coverage counter, don't assume).
6. Quartus synthesizes `feed_top` for `5CSEMA5F31C6N` with **zero inferred latches** and zero timing violations at the reported Fmax.
7. Fmax, ALM count, and BRAM (M10K) count are recorded from the Quartus fitter report — not estimated.
8. Wire-to-BBO latency is measured by the testbench (§6) and recorded in cycles and in ns.

Anything less than all eight and the résumé bullets are not yet true. Do not write them until they are.

---

## 9. Directory layout

```
wire-to-book/
├── README.md
├── SPEC.md              ← this file
├── CLAUDE.md            ← implementation guardrails for Claude Code
├── Makefile
├── rtl/
│   ├── feed_pkg.sv
│   ├── eth_rx.sv
│   ├── ipv4_rx.sv
│   ├── udp_rx.sv
│   ├── frame_parser_top.sv
│   ├── itch_decode.sv
│   ├── order_store.sv
│   ├── bbo.sv
│   └── feed_top.sv
├── tb/
│   ├── tb_parser.cpp
│   ├── tb_book.cpp
│   ├── book_model.cpp / book_model.hpp
│   ├── pcap_reader.cpp / pcap_reader.hpp
│   └── ref_parser.cpp / ref_parser.hpp
├── gen/
│   └── pcap_gen.py
├── data/
│   ├── smoke.pcap / smoke.msgs
│   └── soak.pcap  / soak.msgs
└── syn/
    ├── quartus/        (feed_top.qsf, feed_top.sdc)
    └── results/        (fitter + timing reports, committed)
```

---

## 10. Known limitations (v2 backlog — keep this list, it is interview material)

| Limitation | What a real design does |
|---|---|
| Direct-mapped order store, no collision handling | CAM, or a hash table with chained buckets |
| 8 price levels per side | Full depth-of-book in BRAM; a price ladder indexed by tick offset |
| Aggregate qty per level, no order queue | Per-level FIFO preserving time priority |
| No backpressure | AXI-Stream `ready`/`valid` handshake throughout |
| Single symbol | Symbol ID in the message; per-symbol book instances or a shared BRAM book |
| No UDP checksum verify | Verify it |
| No 10G MAC / PCIe | XGMII front end; PCIe DMA to host |
| One byte per cycle | 8- or 64-byte-per-cycle datapath (the actual latency win) |

The last row is the one an interviewer will push on. Have an answer.
