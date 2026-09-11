---
layout: default
title: FPGA Order Book
parent: Featured Projects
nav_order: 1
---

# FPGA NASDAQ ITCH Parser + Order Book

*Source code, SystemVerilog modules, and Cocotb testbenches are available on [GitHub](https://github.com/patrick96-chu/nasdaq-itch-parser-order-book).*

## Overview

A SystemVerilog project designing an FPGA design to parse the NASDAQ ITCH 5.0 protocol and construct an order book to return the best bid and offers (aka "top of the book").

<img src="/assets/Waveform 2.png" width="800" alt="Waveform">

*Output BBO & Shares available 6 clocks after last word of incoming message/packet*

### Core Specs

| Metric / Parameter | Specification |
| :--- | :--- |
| **Target Device** | AMD/Xilinx Artix UltraScale+ (xcau25p-sfvb784-2e) |
| **Clock Frequency ($F_{\text{max}}$)** | **312.5 MHz** (3.2 ns clock period), WNS = 0.000 ns |
| **Feed Protocols** | NASDAQ ITCH 5.0 (UDP) |
| **Ingress Bus Format** | 32-bit AXI4-Stream |
| **Tick-to-Signal Latency** | **6 Clock Cycles (19.2 ns)** |
| **L3 Order Book Capacity** | Hash Table: 2,048 Sets, 8 Ways (**16,384 slots**) |
| **L3 Collision Resolution** | 8-Entry Fully Associative Spillover CAM |
| **BBO Register Depth** | Depth-2 (Top-of-Book + Next Best) |
| **Verification Suite** | Cocotb (Python) + Verilator & Automated Scoreboard |

### Problem Context & Hardware Motivation

In high-frequency trading (HFT), firms must process incoming market data and calculate orders to send back to the exchange's matching engine. Latency is critical here; a firm that has a shorter tick-to-trade latency is able to capture a better price / queue position. A traditional CPU-based approach processes sequentially through an OS stack, leading to latency spikes on cache misses, context switches, etc. Meanwhile, a hardware-based approach with FPGAs can process data in parallel and directly on the silicon, cutting down latency from microseconds/milliseconds to nanoseconds. 

## Table of Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

## Stack

RTL: SystemVerilog<br>
Simulation: Verilator 5.032, Cocotb 2.0.1, GTKWave<br>
Timing: Vivado 2026.1<br>
Other: Python 3.14, Pytest<br>

## System Architecture

<img src="/assets/Block Diagram 2.png" width="800" alt="Block Diagram">

### MoldUDP Parser
- Asserts correct session ID, sequence number, and packet/message length
    - This ensures there are no missing messages, adding an extra layer of safety

### ITCH Parser
- Asserts valid message type and alpha fields, correct message length for message type
- Decodes raw bytestream into structured buses of enums and values:
    - dec_ctrl_s for system-wide messages such as: System Event, Directory, Trading Action, Operational Halt
    - dec_book_s for stock-specific order messages such as: Add (with/without MPID), Execute (with/without price), Cancel, Delete, Replace (split into a Delete and an Add dec_book_s)
- Begins holding oid_hash output to upcoming dec_book's value 2 cycles before dec_book is valid to begin reading from L3's BRAM (read latency = 2)
- Also only allows dec_book valid if stock_locate matches (stock_locate supplied by Control module upon Directory message)

### Control
- Receives initialized stock_symbol and market_code from top level input
- Receives dec_ctrl_s from ITCH Parser
- Asserts System Event "Start of Messages" is first message received, consistent stock_locate with desired stock_symbol
- Outputs stock_locate corresponding to stock_symbol, stock_active if "Start of System hours" and stock is trading/in quotation period

### L3 Order Table
- Receives dec_book_s from ITCH Parser
- Maintains record of each individual order ID's shares, buy/sell side.
- Asserts share count and order IDs are consistent with previous messages (no negative shares, ID does not already exist on Add, ID exists on Reduce)
- Outputs lookup_dec_book, which has all necessary fields populated (e.g. shares to subtract in a Delete order)

### L2 Price Table
- Receives initialized price_base from top level input
- Receives lookup_dec_book from L3 Order Table
- Maintains record of shares at each price tick in the range $\left[ \mathtt{price\\_base}, \mathtt{price\\_base} + \mathtt{tick\\_size} \cdot (\mathtt{L2\\_MEM\\_SIZE} - 1) \right]$
    - Maintains register of existence of shares at each price tick
    - This architecture is further discussed in the Tradeoffs & Design Choices section below
- Supplies BBO with next-best price level & shares upon depletion of a price level in the BBO

### BBO
- Receives lookup_dec_book from L3 Order Table
- Compares the new order against existing best 2 levels for 1-cycle determination of the best level immediately after new order
- Tracks number of total shares outside the best 2 levels to determine whether L2 can supply a valid next-best level

## Architectural Tradeoffs & Design Choices

### Parser 4-Byte Alignement

Since the individual ITCH messages are consecutively packed within the payload of the MoldUDP64 packets, there is effectively only a minimum gap of 2 bytes of irrelevant ITCH data (from the message length fields in the MoldUDP64 protocol, if those are discarded). Furthermore, ITCH messages could have any byte lengths modulo 4.

Therefore, a 4-byte wide window that enters the pipeline in 1 cycle could contain relevant ITCH data from two different ITCH messages, massively complicating the logic needed to parse this naively.

#### Shifted Sliding Window

To resolve the issue of byte offset, we keep a sliding window of the stream to the ITCH parser, such that the relevant fields may be 4-to-1 MUX selected from the window based on the byte offset. Sizing the sliding window to cover the whole message allows for all fields to be extracted at once.

We also allow the message length fields of the MoldUDP64 packets to pass to the ITCH parser, allowing the ITCH parser to keep track of bytes left in the message, determining the beginning of the next message length field and ITCH message accordingly. To avoid needing to parse content from multiple messages in a cycle, we still employ the sliding window from solution 1 by delaying the processing of the first few bits of the second message for a cycle, but not delaying later bytes of the message for no impact on the latency.

### L3 Order Book

Some kind of data structure is necessary to keep track of individual orders for fast lookup by order ID. During order execute, cancel, delete, and replace messages, the price and/or quantity of the order must be quickly accessible.

#### Total Capacity

A typical equity sees hundreds to thousands of resting orders (individual orders that must be tracked by the L3 book), and may spike to 10,000+ orders at certain times of the day (mega-cap). Thus, the total effective capacity of the L3 book for this project should be at least several thousand to handle all but the largest-cap stocks.

#### Order ID Hash Collision

It is almost certain that multiple order IDs simultaneously exist in the book with the same hash. Therefore, taking inspiration from cache structure, we implement an 8-way, $(N \geq 2000)$-set associative memory structure. Furthermore, we add a small content-addressable memory (CAM) to handle any overflows if more than 8 existing orders share the same hash. It was later determined that $N = 2048$ was the maximum power of 2 meeting timing constraints.

Note that actually reaching the total 8N capacity given by this structure is nearly impossible because too many hash collisions would occur; using a Poisson model, the maximum safe load is approximately 5000 (expected overflow = 2.6 orders).

#### Timing Constraint / Pipelining

Due to the high frequency required for this purpose, the selection logic for the large memory primitive macro (XPM_MEMORY_SDPRAM) itself has a 1-cycle read latency internally, followed by a custom 1 cycle latency for write-read bypassing and muxing from internal memory latches to the output. Then, the order ID comparison should be done the following cycle for a total L3 operation latency of 3. The order ID hash should be sent 2 cycles earlier from the ITCH decoder to facilitate pre-fetching of the hash table line.

### BBO Extraction

For fast retrieval of the top of the book, we store the top 2 aggregated price / shares orders in registers (note that storing the top 2 levels guarantees immediate output of the top price level after the order). The following algorithm outlines how to update these registers on a new order. WLOG, we consider only the bid / buy orders:

#### Add Order / Replace's Add

Compare the price of the new add order against the existing top 2 bids. If a match exists, add the shares to the same entry. Otherwise, choose the slot of the highest bid lower the price of the newest order to insert a new entry in (shift lower prices down 1 register).

#### Execute / Cancel / Delete Order

Search for any matches in the existing top 2 bids. If a match exists, subtract the shares from the same entry. If the resulting number of shares is 0 (the price level has been depleted), shift the lower bids up 1 register, and send a request to the L2 price ladder to fetch the new 2nd best bid.

### L2 Price Table

A memory structure containing aggregate orders (total # of shares at each price) is necessary to faciliate fast retrieval of the next best buy/sell price when the best price is depleted; searching through all orders is too slow and introduces latency spikes. To facilitate the lookup itself, a wide register indicating whether any shares exist at each particular price level can determine the next best price via masking and priority encoding. The larger memory structure serves to update this register array. Because each order requires modifying the number of shares at a particular price, we require constant time access by price.

### Priority-Encoding Existence Array

Since we wish to determine the best level after an already-known level, we can take in the already-known level as an input and mask out all undesired existences. Then, we use a priority encoder to find the index of the best existence within the masked levels.

After some timing testing, it was determined that the masking + priority encoding needed to be split into two cycles; masking + encoding within smaller blocks, then encoding across blocks + address decoding for memory lookup.

### Latency / Special Cases

The minimum interval between possible depletions of the BBO is 4 idle cycles (exclusive) between the arrival of consecutive delete orders (which have the shortest length at 19 bytes (21 including message length field)). Therefore, to avoid needing to delay processing of any orders, the L2 search and BBO population must be completed in these 4 idle cycles to avoid complicating logic for updating the BBO.

Note that the next best price level must always be less than (buy-side)/greater than (sell-side) the worst price level existing in the BBO, so we continuously calculate the next best price level even before the BBO receives the order that would cause a depletion.

#### Tick Size

Prices sent over the ITCH protocol are expressed as 32 bit integers equal to $10^4$ times the actual price, such that incrementing the price field corresponds to an increment of 0.01 cents. However, stocks priced above $1 per share have a minimum increment of 1 cent or 0.5 cents. It is more interesting/nuanced to design for a tick size of 0.5 cents (note that migrating to a tick size of 1 cent would be as simple as changing the price-ordering mapping, leading to a doubled price range, and should not impact timing).

Since it would cost an excessive amount of memory space to maintain tick sizes of 0.01 cents when at most only one slot every 50 ticks are actually used, we require a mapping of integers divisible by 50 to a continuous integer range (see address mapping below).

#### Tick Offset

Because most orders exist close to the top of the book $\pm$ a few dollars, and orders outside this range are highly unlikely to require BBO access, a typical architecture would feature a sliding window that only captures the smaller range at the top of the book, and store the remaining L2 orders elsewhere (e.g. a CAM or directly iterating through the L3 table in the unlikely event the window needs to shift). To keep the maximum latency low and to keep this project in scope, we only implement a static window with a initializable tick offset provided by the test bench, and assert an error if the BBO moves outside of this range.

#### Address Mapping

As previously mentioned, for stocks priced above 1 dollar per share, the tick size is 0.5 cents or 1 cent. To cover both of these cases, we design for a configuration with tick size of 0.5 cents. Then considering the tick offset, we require a bijective function to map the following sets: $\left{\mathtt{OFFSET\\_BASE} + 50 k\\right} \to {k}$ for integer $k \in \left[0, \mathtt{L2\\_SIZE} - 1\right]$ (not necessarily in this order). This serves to calculate the address to access the L2 memory structure with.

**Option 1**: The obvious solution is to subtract $\mathtt{OFFSET\\_BASE}$ and divide by 50. Division by 50 (which is not a power of 2) would cost significant logic, and have a latency of around 1-2 cycles. Since the latency of the L2 book affects the latency during BBO depletions (the L2 register array must be updated to the same state as what the BBO received before a next best bid/offer lookup may occur), this option may be too expensive latency-wise, especially since no other work is possible to be done in parallel.

**Option 2** (taken): Directly divide by 2 and truncate the upper bits. For a L2 price ladder sized as a power of 2, $50 / 2 = 25$ is coprime with the size, so the multiples of 25 in range cover all integers mod the size, although out of order. The benefit of this approach is that it costs 0 logic and only routing delay to calculate the address from the price.

However, the priority encoder to find the next best existing price will need take this different order into account. Since different $\mathtt{OFFSET\\_BASE}$ values mod $\mathtt{L2\\_SIZE}$ result in wildly different orderings of the address space, the best solution would still be to maintain the register array in order. Note that since the register array only needs to be written to as a result of a share count modification, which in turn requires a memory structure access, we may maintain a lookup table of the address space to the actual ordering at no cost to latency.

The reverse mappings of ordering to address space and price are necessary during a next best bid/offer lookup; the address space to access the aggregate shares at the selected price level, and the actual price to supply to the BBO module. While the calculation of price / address space from the ordering is simpler (a multiplication by 50 followed by adding the $\mathtt{OFFSET\\_BASE}$), it was later determined that it could not fit into a single cycle in addition to address decoding for a memory primitive. Thus, we also maintain the reverse mapping of ordering to address space AND ordering to shares at the corresponding price in another memory structure to cut one cycle from the latency.

## Verification & Testing

### Python Reference Model

A Python reference model was implemented to generated expected outputs of inputs. To test the Python reference model, scenario tests were implemented to trigger every possible error. Then, the model was tested on real data (Boston Exchange 2019-12-30) by reconstructing the L2 aggregate price table from the L3 order table, and constructing the BBO from the L2 table, asserting a consistent state across all modules.

### DUT Monitors

An error monitor was implemented to watch the overall DUT error flag as well as the intended error net, asserting that the intended error happens (if any was intentionally specified as a part of the test), otherwise that no error happened.

Output monitors for each module that was tested interpret the output signals and assert that the outputs matched the expected queue of outputs.

### Cocotb

Cocotb (a Python-based coroutine-based cosimulation testbench) was used with Verilator to simulate the DUT. Cocotb offers the advantage of ease-of-use.

## Vivado Implementation

At 32 bits per clock, a clock of 3.2 ns = 312.5 MHz corresponds to 10Gb/s max throughput.

The final design met timing constraints:

<img src="/assets/Timing Summary 2.png" width="800" alt="Post-implementation Timing Summary">

*Post-implementation Timing Summary*


<img src="/assets/Resource Utilization 2.png" width="400" alt="Resource Utilization">

*Resource Utilization*

