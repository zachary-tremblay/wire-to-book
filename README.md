# Wire-to-Book

An FPGA market-data feed handler: raw Ethernet frames in, live top-of-book out.

Written in SystemVerilog, verified against a cycle-accurate C++ golden model under Verilator, and synthesized for an Altera Cyclone V (DE1-SoC).

```
  Ethernet frame                                                    BBO
  ─────────────►┌────────┐  ┌─────────┐  ┌────────┐  ┌────────────┐ ───►
   (bytes, 1/clk)│ eth_rx │─►│ ipv4_rx │─►│ udp_rx │─►│ itch_decode│
                 └────────┘  └─────────┘  └────────┘  └─────┬──────┘
                  strip MAC   strip IP     strip UDP        │ 16-byte msg
                  ethertype   verify hdr   port filter      ▼
                  check       checksum                ┌─────────────┐
                                                      │ order_store │  256-entry
                                                      └──────┬──────┘  direct-mapped
                                                             │ book_op   (BRAM)
                                                             ▼
                                                      ┌─────────────┐
                                                      │     bbo     │  8 levels/side
                                                      └─────────────┘  parallel
                                                                       comparator tree
```

Full technical contract in [`SPEC.md`](SPEC.md).

---

## Results

*(Fill in from the Quartus fitter report and the testbench. Do not estimate — measure.)*

| Metric | Value |
|---|---|
| Target device | Cyclone V `5CSEMA5F31C6N` (DE1-SoC) |
| Fmax | `___` MHz |
| ALMs | `___` |
| Block RAM (M10K) | `___` |
| Registers | `___` |
| Throughput | 1 byte / clock cycle |
| **Wire-to-BBO latency** | **`___` cycles (`___` ns @ Fmax)** |
| Soak test | `___` messages, 0 BBO mismatches |
| Inferred latches | 0 |

Latency is measured from the cycle the final byte of a message is accepted at the input to the cycle `bbo_valid` pulses for that message. It is a fixed constant — there is no backpressure and no variable-length path.

---

## What it does

The design ingests a UDP market-data feed carrying a simplified ITCH-style binary protocol and maintains the best bid and best offer in hardware.

**Network front end.** Strips the Ethernet, IPv4, and UDP headers from a byte stream arriving one byte per clock. Checks the ethertype, verifies the IPv4 header checksum with a one's-complement adder tree, and filters on UDP destination port. Malformed frames are dropped and counted, not propagated.

**Message decoder.** Assembles fixed 16-byte messages from the UDP payload. Four message types: add, cancel, execute, delete.

**Order store.** Cancel, execute, and delete messages carry only an order ID and a share count — the side and price of the order they refer to must be *recovered by looking the order up*, exactly as in the real protocol. A 256-entry direct-mapped table in block RAM does this, keyed on the low bits of the order ID.

**Top-of-book engine.** Eight price levels per side held in registers. On each book update, the best bid (maximum price with non-zero quantity) and best ask (minimum price with non-zero quantity) are resolved combinationally by a three-stage parallel comparator tree — one cycle, regardless of book state.

---

## Verification

The RTL is checked against a C++ golden model — a deliberately simple `std::map`-based order book with no knowledge of the hardware. A constrained-random generator produces a 100,000-message feed as a `.pcap`; the testbench drives those bytes into the Verilated DUT one byte per cycle and, on **every** book update, asserts that the hardware's best bid and best ask match the model's exactly. A single mismatch dumps a waveform and fails the run.

The design also exposes five sticky error flags (order-ID collision, unknown order, over-cancel, price-level overflow, reduce-miss). A passing soak run requires all five to read zero — they exist to prove the stimulus constraints hold, not to be tolerated.

This is the same shape as production FPGA verification at low-latency trading firms: the hardware is not trusted, a software reference defines truth, and equivalence is asserted continuously under random stimulus.

---

## Build and run

**Prerequisites:** Verilator ≥ 5.0, a C++17 compiler, GTKWave, Python 3 with `scapy`, and Quartus Prime Lite (for synthesis only).

```bash
# generate stimulus
make stimulus              # writes data/smoke.pcap and data/soak.pcap

# simulate
make parser                # Weekend-1 target: network front end only
make book                  # full chain, smoke test
make soak                  # full chain, 100K messages  ← the headline run

# waveforms
make waves                 # opens the last dump in GTKWave

# synthesis (requires Quartus on PATH)
make syn                   # writes syn/results/
```

A passing `make soak` prints the message count, zero mismatches, all five error flags at zero, and the measured wire-to-BBO latency.

---

## Design notes and known limitations

These are deliberate. The point of the project was to build something correct and measured, not something complete.

| Limitation | What a production design does instead |
|---|---|
| **Direct-mapped order store**, no collision handling | A CAM, or a hash table with chained buckets. The stimulus generator constrains order IDs so collisions cannot occur, and a sticky flag proves it. |
| **8 price levels per side**, aggregate quantity only | Full depth-of-book in BRAM, typically a price ladder indexed by tick offset from a base. |
| **No order queue within a level** | A per-level FIFO preserving time priority — required to model fills correctly. |
| **No backpressure** | An AXI-Stream `ready`/`valid` handshake through the whole datapath. |
| **Single symbol** | A symbol ID in the message and either per-symbol book instances or a shared, indexed BRAM book. |
| **UDP checksum not verified** | Verified. |
| **1 byte per cycle** | A wide datapath — 8 or 64 bytes per cycle. This, not the book logic, is where the real latency win lives, and it is the hardest part of the redesign. |
| **No MAC / PCIe** | An XGMII front end and PCIe DMA to the host. |

The last row is the interesting one. At one byte per clock, header parsing dominates the latency budget and the book is nearly free; at 64 bytes per clock the parser becomes a wide combinational shift-and-select and the book becomes the critical path. The whole design would need to be reshaped around that.

---

## Repository layout

```
rtl/      SystemVerilog source
tb/       C++ testbenches and the golden model
gen/      stimulus generator (Python)
data/     generated .pcap and .msgs files
syn/      Quartus project, constraints, and committed fitter/timing reports
SPEC.md   the technical contract
```
