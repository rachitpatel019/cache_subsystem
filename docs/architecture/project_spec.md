# Reusable Multicore Cache Hierarchy IP — Project Specification and Phase Plan

**Document status:** Working project baseline  
**Version:** 1.0  
**Date:** 2026-10-31  
**Implementation language:** SystemVerilog  
**Verification methodology:** UVM (IEEE 1800.2-compatible implementation)  
**Primary focus:** RTL design and full-hierarchy simulation verification  

---

## 1. Project purpose

Design, implement, and thoroughly verify a reusable, parameterized cache hierarchy IP block. The IP sits between one or more processor request interfaces and an external memory-controller interface. The processor, external memory controller, and external RAM are **not** part of the synthesizable cache IP. They are represented by UVM agents/models during simulation.

The project has two equally important outcomes:

1. **RTL outcome:** a modular, synthesizable SystemVerilog cache hierarchy with clear interfaces, reusable cache structures, explicit control logic, and a path from a single-core implementation to a multicore implementation.
2. **Verification outcome:** a reusable UVM environment that drives realistic processor traffic, models backing memory, checks data and coherence behavior independently, measures functional coverage, runs assertions, and supports automated regressions.

The project should be technically ambitious, but each phase must have a well-defined completion gate. Later features must extend earlier interfaces and modules rather than require a complete redesign.

## 2. Scope and non-goals

### 2.1 In scope

- Private L1 instruction and data caches per core.
- One unified, private L2 cache per core.
- One unified L3 cache shared by all cores.
- Physical addressing; address translation is outside this IP.
- Byte-addressed memory semantics.
- Write-through propagation from L1D through L2 to L3.
- Write-back from L3 to external memory.
- Read allocation at all cache levels.
- No-write-allocate behavior for L1D and L2 writes.
- Write-allocate behavior for L3 writes.
- Pseudo-LRU replacement.
- Initially blocking cache operation.
- A centralized directory/coherence controller separate from the L3 data-cache module.
- Two-core coherence followed by parameterized N-core support.
- Custom valid-ready interfaces for the initial implementation.
- An AXI4 Full adapter at the external-memory boundary in a later phase.
- A full-hierarchy UVM environment, reference memory model, scoreboard, assertions, coverage, stress tests, and regressions.
- Technology-independent synthesizable RTL, with FPGA/ASIC implementation considerations kept in mind.

### 2.2 Explicitly outside the initial project scope

The following are not required for the initial hierarchy and must not be allowed to delay Phase 1:

- A real processor pipeline or CPU core.
- An external DDR/LPDDR controller, PHY, or physical memory device.
- Virtual-memory translation, page tables, or TLBs.
- Operating-system support.
- MMIO or uncacheable regions.
- Atomic read-modify-write operations, LR/SC, or AMOs.
- General unaligned accesses or accesses that cross a cache-line boundary.
- Multiple outstanding misses, MSHRs, out-of-order responses, or non-blocking cache operation.
- A full cache-maintenance architecture (flush/invalidate-all, clean-by-address, etc.).
- A standardized coherent interconnect such as ACE or CHI.
- Physical design, transistor-level design, SRAM compiler work, or signoff timing/power analysis.

Some of these may be considered as explicit later enhancements. They are not implicit requirements.

## 3. Top-level architecture

### 3.1 Required hierarchy

Only L1 is split into instruction and data caches. Every cache below L1 is unified.

```text
                          CACHE HIERARCHY IP

      Core 0 request ports                         Core N-1 request ports
       +-------------+                              +-------------+
       |             |                              |             |
      L1I           L1D                            L1I           L1D
       |             |                              |             |
       +------+------+                              +------+------+
              |                                            |
       L1I/L1D arbiter                              L1I/L1D arbiter
              |                                            |
          Private L2                                   Private L2
              |                                            |
              +-------------------+------------------------+
                                  |
                     Separate directory/coherence controller
                                  |
                           Shared unified L3
                                  |
                   External-memory line interface (Phase 1)
                                  |
                     AXI4 Full adapter (later phase)
                                  |
                   External memory controller (not in IP)
                                  |
                         External RAM (not in IP)
```

The diagram is logical: the directory controller communicates with the shared L3 and with each private L2. It is a separate RTL module, not code embedded inside the L3 cache data-array controller.

### 3.2 Per-core organization

Each core has:

- One private L1 instruction cache (L1I), read-only from the processor interface.
- One private L1 data cache (L1D), supporting reads and writes.
- One private unified L2 cache shared by that core's L1I and L1D.
- A two-input arbiter between L1I/L1D and the private L2.
- A coherence endpoint at the private L2 boundary. L2 is the directory's per-core coherence participant and cascades invalidations to both local L1 caches.

### 3.3 Shared organization

- One unified L3 is shared by all configured cores.
- One separate directory/coherence controller tracks which private L2 subtrees hold each L3 line.
- The L3 is the last on-chip cache and the only cache that writes dirty cache lines back to external memory.
- External memory is accessed using full cache-line transfers at the custom interface in Phase 1. The custom interface is later connected to AXI4 Full through a dedicated adapter.

## 4. Baseline architectural decisions

The following is the working baseline for implementation. Geometry values are concrete starting defaults so RTL and tests can be developed consistently. They may be changed during the Phase 0 design review, but they should not be changed casually after Phase 1 RTL and reference-model behavior are frozen.

### 4.1 Cache geometry defaults

All cache levels use a 64-byte cache line in Phases 1 and 2.

| Cache | Scope | Capacity | Associativity | Number of sets | Line size |
|---|---|---:|---:|---:|---:|
| L1I | Private, per core | 16 KiB | 4-way | 64 | 64 B |
| L1D | Private, per core | 16 KiB | 4-way | 64 | 64 B |
| L2 | Unified, private, per core | 256 KiB | 8-way | 512 | 64 B |
| L3 | Unified, shared | 2 MiB | 16-way | 2,048 | 64 B |

The number of sets is calculated as:

`NUM_SETS = CAPACITY_BYTES / (LINE_BYTES * NUM_WAYS)`

All set counts are powers of two. The initial replacement implementation is tree-based pseudo-LRU, which naturally fits the power-of-two associativities selected above.

### 4.2 Initial top-level parameters

Keep the initial public parameter list intentionally small:

| Parameter | Phase 1 default | Meaning |
|---|---:|---|
| `ADDR_WIDTH` | 32 | Physical byte-address width |
| `CPU_DATA_WIDTH` | 32 | CPU-side data path width in bits |
| `NUM_CORES` | 1 | Number of configured processor request-port pairs |

In Phase 1 only `NUM_CORES=1` is used for functional implementation, but the top-level parameter should already exist. Phase 2 elaborates with `NUM_CORES=2`. Phase 3 validates larger values.

The following remain fixed implementation constants in Phases 1 and 2 and are exposed as configuration parameters only in a later phase:

- `CACHE_LINE_BYTES = 64`
- L1I/L1D/L2/L3 capacities
- L1I/L1D/L2/L3 associativities
- Replacement policy = tree pseudo-LRU

### 4.3 Clock and reset

- One common clock is used by all cache and coherence RTL in the initial project.
- No clock-domain crossings are required.
- Use a single active-low reset (`rst_n`).
- Reset is synchronous to the active clock edge for the baseline implementation.
- Reset clears valid bits, replacement metadata, FSM state, pending-transaction state, coherence state, and directory entries.
- Data arrays do not need to be reset; invalid metadata makes their contents architecturally irrelevant.

### 4.4 Physical, byte-addressed memory

All cache request addresses are **physical byte addresses**. Address `A` refers to byte `A`; the address does not count words.

The CPU data width is independent of address width and independent of the eventual AXI data-bus width. A 32-bit CPU data path means a 4-byte CPU transfer lane, not word-addressed memory.

For the initial 64-byte line:

- Address bits `[5:0]` are the byte offset within a line.
- The set-index field follows the line offset and depends on the cache's number of sets.
- The remaining upper bits form the tag.
- Every internal line request uses a 64-byte-aligned line address.

The initial CPU access sizes are 1, 2, and 4 bytes for `CPU_DATA_WIDTH=32`. If/when a 64-bit CPU interface is enabled, 8-byte accesses may be added. Natural alignment is required in the initial implementation. Unsupported sizes, misaligned accesses, and accesses that cross a cache-line boundary must return a defined error response or be rejected by a documented protocol assertion; they must never silently corrupt neighboring bytes.

### 4.5 Write and allocation policies

| Level | Read miss | Write hit | Write miss | Dirty data relative to next level? |
|---|---|---|---|---|
| L1I | Read-allocate | No processor writes | Not applicable | No |
| L1D | Read-allocate | Write-through to L2 | No-write-allocate; forward to L2 | No |
| L2 | Read-allocate | Write-through to L3 | No-write-allocate; forward to L3 | No |
| L3 | Read-allocate | Update line and mark dirty | Write-allocate; fetch line, merge write, mark dirty | Yes, relative to RAM |

Detailed behavior:

1. **L1D write hit:** obtain/confirm write permission as required by the coherence phase, send the byte update to L2, and update the L1D copy only when the lower-level write transaction has completed successfully.
2. **L1D write miss:** do not allocate an L1D line. Forward the write to L2.
3. **L2 write hit:** obtain/confirm write permission as required, update the L2 line, and forward the byte update to L3. L2 is write-through and never dirty relative to L3.
4. **L2 write miss:** do not allocate an L2 line. Forward the write to L3.
5. **L3 write hit:** merge the byte-enabled write into the resident line and set the L3 dirty bit.
6. **L3 write miss:** allocate the line. For a partial write, first fetch the old 64-byte line from external memory, then merge the write and mark the line dirty. A full-line write optimization may be added later but is not required initially.
7. **L3 dirty eviction:** write the complete 64-byte line to external memory before reusing the victim entry.
8. **L3 clean eviction:** no external write is required.

The policy intentionally puts the write-back boundary at L3: private L1/L2 caches propagate stores toward L3, while L3 absorbs writes and eventually writes complete dirty lines to RAM.

### 4.6 Replacement policy

Use tree-based pseudo-LRU for all four cache types.

- If a set has an invalid way, allocate into an invalid way before evicting a valid line.
- On a hit, update the pseudo-LRU metadata to record the accessed way as recently used.
- On a fill, update the metadata as though the newly filled way was just accessed.
- On a full-set miss, select the victim using the tree-PLRU bits.
- L1 and L2 victims are clean relative to their next level under the selected write-through policy. L2 eviction must still invalidate any inclusive L1 copies and notify the directory in multicore phases.
- L3 victims may be dirty and may have private L2 sharers. In Phase 2 onward, the hierarchy must complete required back-invalidations before the L3 entry is reused.

### 4.7 Inclusion policy

Use an **inclusive hierarchy**:

- L2 is inclusive of its corresponding L1I and L1D.
- L3 is inclusive of all private L2 caches.

Consequences:

- An L2 eviction must invalidate any matching L1I and L1D copies before the L2 way is reused.
- An L3 eviction must invalidate every private L2 copy recorded by the directory. Each L2 then invalidates matching local L1I/L1D copies and acknowledges the request.
- Directory tracking must never omit a real L2 sharer. Conservative false-positive sharer entries are safe but create unnecessary invalidations; false-negative entries are incorrect.
- Because L1/L2 are write-through, no private dirty-data writeback is needed on invalidation or eviction. L3 is responsible for preserving the most recent data.

The initial geometry gives L3 more capacity than one or two private L2 caches. For larger `NUM_CORES`, L3 capacity should be increased in the configurable-geometry phase to avoid excessive back-invalidations. A capacity ratio is a performance/design guideline, not a substitute for correct inclusion enforcement.

## 5. Cache coherence architecture

### 5.1 When coherence is introduced

- Phase 1: one core only; no inter-core coherence traffic is required.
- Phase 2: two cores; add centralized directory-based write-invalidate coherence.
- Phase 3: generalize the directory and coherence endpoints to `NUM_CORES`.

The architecture should preserve a dedicated coherence interface boundary in Phase 1 so Phase 2 does not require redesigning the data path.

### 5.2 Selected protocol

Use a project-defined, **centralized directory-based, write-invalidate protocol**. The private L2 is the coherence participant for each core's entire private cache subtree. The directory controller is a separate module associated with the shared L3.

Use three private coherence states at L2:

| State | Meaning |
|---|---|
| `I` — Invalid | The private core subtree does not hold a valid copy of this line. |
| `S` — Shared | The private core subtree holds a clean copy; one or more other cores may also hold the line. |
| `E` — Exclusive | This private core subtree is the only core allowed to hold a valid copy; writes are permitted, but are still written through to L3. |

There is no private `M` state in this baseline because L1D and L2 are write-through. L3 separately maintains a dirty bit relative to external RAM. `E` therefore means exclusive permission, not dirty ownership.

L1I and L1D maintain ordinary valid/tag/data metadata. L2 maintains the coherence state for the whole private subtree and is responsible for invalidating local L1 copies.

### 5.3 Directory metadata

For each valid L3 line, the separate directory stores:

- A `NUM_CORES`-bit sharer vector identifying private L2 caches that contain the line.
- Enough state to identify whether the line is exclusively held by one core or shared by multiple cores. This may be derived from the sharer vector or stored as a compact owner/state field.
- Any transient/busy metadata required to serialize an in-progress upgrade, downgrade, invalidation, or eviction.

A directory entry must be cleared when the corresponding L2 voluntarily evicts the line. If a directory entry is stale but conservatively indicates a possible sharer, an invalidation to a nonresident line must be acknowledged safely.

### 5.4 Required protocol behavior

**Read miss with no existing sharer**

1. Requesting L2 sends a shared-read request to the directory/L3.
2. L3 supplies the line.
3. Directory records the requesting core as the only sharer.
4. Requesting L2 receives `E` permission.

**Read miss when another core has `E`**

1. Requesting L2 requests the line.
2. Directory sends a downgrade request to the existing exclusive L2.
3. Existing L2 changes `E` to `S` and acknowledges.
4. L3 supplies the current line to the requester.
5. Directory records both sharers; both private L2 copies are `S`.

**Read miss when one or more cores have `S`**

1. L3 supplies the line.
2. Directory adds the requester to the sharer vector.
3. Requesting L2 receives `S`.

**Write to a line already held `E` by the requester**

1. The write is sent through L1D/L2 to L3.
2. L3 merges the byte update and marks its line dirty.
3. The requester remains the only sharer in `E`.
4. The write response is returned only after the update has committed at L3.

**Write to a line held `S` by the requester**

1. Requesting L2 asks for exclusive permission and supplies the write intent/data as part of the same logical transaction. A separate permission-grant then data-write implementation is also acceptable only if the directory marks the line transient/locked until the update commits, preventing another requester from observing an intermediate state.
2. Directory sends invalidations to every other core in the sharer vector.
3. Each target L2 invalidates the matching line and cascades the invalidation to its local L1I and L1D.
4. Each target acknowledges only after its local copies are invalid.
5. Directory establishes exclusive permission for the requester and updates the sharer vector to only that requester.
6. L3 applies the write update, marks the line dirty, and acknowledges completion. The store response is not returned before this commit point.

**Write miss at L2 (no-write-allocate)**

1. The requesting L2 does not become a sharer because it does not allocate the line.
2. It forwards the write to L3 with an indication that the requester does not retain a copy.
3. Directory invalidates all existing sharers, if any, and waits for acknowledgements.
4. L3 merges the update and marks the line dirty.
5. The directory sharer vector becomes empty for that line unless the write path explicitly allocated a private copy (not the baseline behavior for an L2 write miss).

**L3 eviction**

1. Directory identifies all private L2 sharers.
2. L3/directory issue back-invalidation requests to those L2s.
3. Each L2 invalidates the line and its local L1I/L1D copies, then acknowledges.
4. After all acknowledgements, L3 writes the victim line to RAM if dirty.
5. The L3 entry and its directory metadata can then be reused.

**L2 voluntary eviction**

1. L2 chooses a victim.
2. L2 invalidates matching L1I/L1D copies (inclusive requirement).
3. L2 informs the directory that this core no longer holds the line.
4. L2 reuses the victim entry. No private writeback is required because the line is clean relative to L3.

### 5.5 Coherence deadlock and ordering requirements

The directory must be able to invalidate a target L2 even if that L2 is waiting for a normal data-path response for a different cache line. Therefore:

- Coherence control traffic must use a control path independent of the ordinary L2-to-L3 data request/response path, or use an equivalent design that cannot deadlock.
- L2 must be able to accept and process an invalidation while its normal blocking FSM is waiting on a different address.
- The shared L3/directory serializes transactions for the same line in the blocking phases.
- A target that does not contain the line must still acknowledge the invalidation as a no-op.
- The directory cannot grant write permission until all required invalidation acknowledgements have arrived.
- The writer's store response cannot be returned until the L3 write-through update has committed.
- The design must prevent two cores from simultaneously holding `E` for the same line.

The protocol guarantees coherent per-line behavior. It does not attempt to specify a complete processor memory-consistency model for ordering arbitrary independent addresses. The testbench must distinguish per-line coherence requirements from global fence/order semantics.

### 5.6 Instruction/data cache interaction

L1I and L1D are separate private arrays but share a unified L2. Remote invalidation of a core's L2 line must invalidate both local L1I and L1D copies.

Self-modifying code and explicit instruction-cache maintenance semantics are not part of the first baseline. The project must document that a processor-level instruction synchronization/maintenance operation is needed before expecting newly written data to be fetched as instructions. A future phase may add an explicit I-cache invalidate/fence interface and verify it.

## 6. Interface architecture

There is no single universal industry-standard interface for a CPU's internal connection to its L1 cache or for every private cache-to-cache connection. The initial project therefore uses explicitly documented custom valid-ready interfaces. AXI4 Full is added at the external memory boundary later. AXI4 is not itself a cache-coherence protocol; the separate directory controller remains responsible for coherence.

### 6.1 CPU request/response interface (CPU model to L1)

Each core has two independent request/response ports:

- Instruction port to L1I: reads only.
- Data port to L1D: reads and writes.

Minimum request fields:

| Field | Purpose |
|---|---|
| `req_valid` | Request is being presented. |
| `req_ready` | Cache accepts request on this clock edge when valid is also asserted. |
| `req_addr[ADDR_WIDTH-1:0]` | Physical byte address. |
| `req_write` | Read (`0`) or write (`1`). Must be `0` on L1I. |
| `req_size` | Number of bytes requested; initial legal values are 1, 2, or 4 for a 32-bit CPU port. |
| `req_wdata[CPU_DATA_WIDTH-1:0]` | Write data in CPU byte lanes. |
| `req_wstrb[CPU_DATA_WIDTH/8-1:0]` | Byte-lane write enables. |

Minimum response fields:

| Field | Purpose |
|---|---|
| `resp_valid` | Response is available. |
| `resp_ready` | Request source accepts response. |
| `resp_rdata[CPU_DATA_WIDTH-1:0]` | Returned bytes in the defined CPU data lanes. |
| `resp_error` | Request could not be completed, e.g. unsupported alignment/size or lower-level error. |

Handshake rules:

- A request is accepted only on a rising edge where `req_valid && req_ready` are both high.
- The requester must hold all request fields stable while `req_valid=1` and `req_ready=0`.
- A response is consumed only when `resp_valid && resp_ready` are both high.
- The responder must hold response fields stable while `resp_valid=1` and `resp_ready=0`.
- Each accepted request produces exactly one response, including error responses.
- L1I accepts reads only. A write presented to L1I is an error/protocol violation.
- Initial operation is blocking per L1 instance: no second request is accepted by that L1 until the prior request completes.

Byte-lane convention must be documented in the interface package and shared by RTL/UVM: the byte strobe bit `i` corresponds to byte lane `i` of the CPU-width-aligned data word containing the request address. The testbench must use the same convention. Read data is returned with requested bytes placed in their corresponding CPU data lanes; sign extension and load-type interpretation belong to the processor model, not the cache.

### 6.2 Cache-to-cache line interface

L1/L2/L3 communicate using a custom line transaction interface. It is line-oriented rather than CPU-word-oriented.

Minimum fields:

- Request valid/ready.
- Operation code.
- 64-byte-aligned physical line address.
- Full 512-bit line payload for line fills/writebacks where applicable.
- 64-bit byte mask for masked line updates.
- Response valid/ready.
- Full-line response payload for line reads.
- Response status/error.

Phase 1 operations:

- `READ_LINE`: request a complete line.
- `WRITE_BYTES`: update selected bytes in a resident lower-level line; if the target is a miss at L1D/L2, forward without allocating.
- `EVICT_NOTICE`/`DROP_LINE`: report a voluntary L2 eviction to the directory boundary once that exists in Phase 2.

Phase 2 coherence operations are carried on a logically separate control interface:

- `GET_SHARED` or equivalent read permission request.
- `GET_EXCLUSIVE`/`UPGRADE` or equivalent write-permission request.
- `INVALIDATE` and `INVALIDATE_ACK`.
- `DOWNGRADE` and acknowledgement.
- `DROP_LINE`/directory removal notification.

Exact enum names may be finalized in the interface specification, but transaction meaning and completion rules must not change without updating the UVM reference model and protocol assertions.

### 6.3 L3-to-external-memory interface for Phase 1

Use a custom full-line request/response interface so that cache logic can be verified before AXI4 is introduced.

Minimum operations:

- Read one 64-byte line from a line-aligned physical address.
- Write one complete 64-byte line to a line-aligned physical address.
- Valid-ready request handshake.
- Response valid-ready handshake for reads and write completion.
- Error/status response.

The UVM memory responder models external memory. It stores bytes in a byte-addressable backing store, accepts line writes, returns line reads, inserts configurable latency, and applies backpressure.

The Phase 1 line interface should be isolated behind a wrapper so Phase 4 can replace the backing interface with an AXI4 Full adapter without modifying the L3 cache policy logic.

### 6.4 AXI4 Full boundary in a later phase

At the external memory boundary, add a dedicated AXI4 Full master adapter. It converts one L3 line read/write into an AXI burst. The initial AXI configuration should use a 64-bit data bus, giving eight 64-bit beats per 64-byte cache line. The adapter must calculate `ARSIZE/AWSIZE`, `ARLEN/AWLEN`, aligned address, burst type, and `RLAST/WLAST` correctly.

Initially, it is acceptable to support one outstanding line transaction with one ID, provided all required AXI channel handshakes and backpressure cases are implemented correctly. Later concurrency work may add multiple IDs and out-of-order responses.

AXI channel requirements include:

- Read address channel (`AR`) and read data channel (`R`).
- Write address channel (`AW`), write data channel (`W`), and write response channel (`B`).
- Independent channel backpressure.
- Stable payload while `VALID=1` and `READY=0`.
- Correct burst length, size, address, incrementing burst behavior, and final-beat signaling.
- Correct response/error propagation.
- No burst may violate AXI boundary requirements.

Use the official Arm AMBA AXI protocol specification as the protocol authority; do not invent a simplified handshake that conflicts with AXI.

## 7. RTL microarchitecture and module boundaries

The design should use a common cache engine where practical, specialized wrappers for the cache roles, and separate hierarchy/coherence control. Avoid three unrelated cache implementations and avoid one monolithic module that combines arrays, replacement, hierarchy control, arbitration, and coherence.

### 7.1 Proposed RTL module inventory

Names are suggested and may be adjusted to match repository conventions.

| Module | Responsibility |
|---|---|
| `cache_hierarchy_top` | Top-level parameterized integration; instantiates per-core clusters, shared L3, directory controller, and external-memory adapter/wrapper. |
| `core_cache_cluster` | Instantiates one core's L1I, L1D, their L2-facing arbiter, and private L2. |
| `l1_icache` | L1 instruction-cache wrapper; read-only CPU port; read-allocate; 4-way baseline. |
| `l1_dcache` | L1 data-cache wrapper; read/write CPU port; WT/NWA write behavior; 4-way baseline. |
| `l2_cache` | Unified private L2 wrapper; arbitrates/serves L1I/L1D line requests; WT/NWA write behavior; 8-way baseline. |
| `l3_cache` | Unified shared data cache; 16-way baseline; read-allocate; write-allocate; write-back to memory. |
| `cache_engine` | Reusable tag/data/valid/replacement/lookup/fill/eviction mechanism parameterized internally for a cache instance. |
| `cache_tag_array` | Tag and valid storage and tag comparison interface. May be integrated into `cache_engine` if that is cleaner. |
| `cache_data_array` | Cache-line data storage and byte/line update logic. |
| `tree_plru` | Tree pseudo-LRU victim selection and update logic. |
| `l1_to_l2_arbiter` | Two-input round-robin arbitration between L1I and L1D requests to one private L2; records the selected source and routes the response back to that L1. |
| `l2_to_l3_arbiter` | Round-robin arbitration among private L2 requesters at the shared L3; records the requesting core and routes responses back to the correct L2. Added in Phase 2 and parameterized in Phase 3. |
| `directory_controller` | Separate centralized directory state, permission decisions, invalidation/downgrade sequencing, and L3 back-invalidation coordination. |
| `coherence_endpoint` | Per-core L2-side handling of directory control messages; probes/invalidates private L2 and cascades to L1I/L1D. |
| `line_memory_adapter` | Phase 1 custom line-memory wrapper; replaced/extended by AXI adapter in Phase 4. |
| `axi4_line_adapter` | Phase 4 line-to-AXI4 burst conversion. |

A module does not need to be created for every row if a clean implementation combines low-level array logic. The architectural boundaries (generic cache function, L1 role wrappers, private L2, shared L3, separate directory, memory adapter) must remain understandable and independently testable.

### 7.2 Cache-engine responsibilities

The generic cache engine should own:

- Tag lookup and hit/miss detection.
- Set and way selection.
- Valid-bit management.
- Data-line read and byte update.
- Invalid-way-first allocation.
- Pseudo-LRU victim selection/update.
- Refill and victim sequencing hooks.
- Per-instance blocking FSM.

It should not own:

- CPU instruction/data semantics.
- Inter-core directory state.
- AXI channel behavior.
- Global L1/L2/L3 routing.

Those responsibilities belong in wrappers/controllers/adapters.

### 7.3 Blocking behavior

Blocking is a deliberate first-phase simplification:

- Each L1I and L1D accepts at most one active CPU request.
- Each private L2 services at most one active lower-level line request at a time.
- The shared L3 services at most one active line transaction globally.
- While busy, a cache deasserts request-ready for requests it cannot accept.
- The two-input L1I/L1D arbiter uses round-robin selection to avoid fixed-priority starvation. It holds the selected source until the corresponding response is accepted and routes that response only to the selected L1.
- In Phase 2, a second round-robin arbiter selects among private L2 requesters for the single blocking L3 request port. It records the requesting core and routes the response to that L2. Phase 3 generalizes this arbiter to `NUM_CORES`.
- Response backpressure must be supported; response fields remain stable until accepted.
- Coherence control in Phase 2 must still be serviceable while a normal data request is waiting on a different line.

Do not add MSHRs or multiple outstanding misses to Phase 1 or Phase 2.

## 8. UVM verification architecture

The requested verification target is the **full hierarchy**. The main UVM environment drives and observes the integrated hierarchy rather than relying only on isolated unit tests. Small module-level tests may be used for debug, but they do not replace the full-system environment.

### 8.1 Testbench actors

**Processor-side agents**

- One instruction request agent per core, connected to L1I.
- One data request agent per core, connected to L1D.
- Each agent has a sequencer, driver, monitor, and transaction type.
- Sequences model instruction fetches, loads, and stores. The driver obeys request and response handshakes and may insert request gaps or response backpressure.

**External memory agent/model**

- Phase 1: active memory responder on the custom line interface.
- Phase 4: active AXI4 slave/memory-controller model behind the AXI adapter.
- Backing storage is byte-addressable and independent of DUT cache arrays.
- Supports deterministic and randomized response latency, request backpressure, and optional error injection.
- Initializes memory deterministically (e.g. zero or reproducible address-based pattern) so tests do not depend on X values.

**Monitors**

- CPU request/response monitors for every core and port.
- Cache-level line-interface monitors.
- Directory/coherence message monitor from Phase 2 onward.
- External-memory monitor.
- Monitors publish transactions through analysis ports; they do not drive DUT signals.

### 8.2 Scoreboard and reference models

Use independent models with distinct responsibilities.

1. **Architectural golden memory model:** byte-addressable expected memory contents. It is updated when a store becomes globally committed according to the externally observable protocol/linearization point. CPU read responses are compared against this model.
2. **Backing RAM model:** represents only external memory. It is updated when L3 writes back a dirty line. It is intentionally allowed to lag the golden architectural memory while L3 holds dirty data.
3. **Cache/coherence reference model:** tracks expected line residency and per-core S/E/I ownership at a transaction level. It predicts which cores should receive invalidations/downgrades and checks that directory-visible behavior is coherent. It must not inspect private DUT arrays to decide expected data.
4. **Protocol scoreboard:** pairs accepted requests with exactly one response, checks IDs/order where applicable, and detects lost, duplicated, or unexpected transactions.

For simultaneous multicore accesses to the same address, the reference model must use a documented serialization/linearization point, such as the accepted/committed transaction order visible at the shared L3/directory protocol boundary. Tests must not assume an arbitrary winner for truly concurrent writes; they must verify that the result is consistent with the DUT's observed serialization and that no stale read is returned after a completed write.

### 8.3 UVM environment layout

Suggested hierarchy:

```text
cache_hierarchy_tb
  cache_hierarchy_test
    cache_hierarchy_env
      cpu_env[NUM_CORES]
        icache_cpu_agent
        dcache_cpu_agent
      memory_agent
      cache_protocol_monitors
      coherence_monitor              // Phase 2+
      cache_scoreboard
      golden_memory_model
      backing_memory_model
      coherence_reference_model      // Phase 2+
      functional_coverage
      virtual_sequencer
```

Use a virtual sequencer to coordinate traffic across cores and across instruction/data ports. Keep agents reusable and configuration-driven.

### 8.4 Verification principles

- The scoreboard must not rely on the DUT's internal tag/data arrays to compute expected read data.
- Constrained-random traffic must use reproducible seeds and log the seed on failure.
- Directed tests must target exact corner cases and be retained after random testing is added.
- Assertions must check both safety (nothing incorrect happens) and liveness/progress (accepted work eventually completes under fair responder assumptions).
- Coverage must track both scenarios exercised and outcomes achieved; a test passing is not proof of coverage closure.
- Every bug fix must add a directed regression test or a constrained-random constraint/coverage improvement that would have exposed the bug.

## 9. Phase-by-phase implementation plan

Each phase has separate RTL tasks, UVM tasks, and an exit gate. Do not begin the next phase until the current phase's exit gate is met or any waived item is explicitly recorded.

---

# Phase 0 — Specification, repository, and verification foundation

## Goal

Turn this working document into a stable interface/behavior specification and create the project infrastructure before writing substantial cache RTL.

## RTL and architecture work

1. Freeze the Phase 1 cache geometry or record approved changes to the defaults in Section 4.1.
2. Define all request, response, line, and memory transaction structs/enums in a package.
3. Define the CPU request/response interface and the line-level valid-ready interface.
4. Define reset behavior, request acceptance, response completion, error behavior, and byte-lane conventions.
5. Define the address mapping for each baseline cache (line offset, set index, tag).
6. Define the exact pseudo-LRU update/victim algorithm and write directed examples for it.
7. Draw the RTL block diagram and data/control paths for L1I read, L1D read, L1D write hit, L1D write miss, L2 miss, L3 miss, and L3 dirty eviction.
8. Define module responsibilities and top-level ports.
9. Establish lint/elaboration rules and a compile script.
10. Define the Phase 1 and Phase 2 coherence interface boundary even though Phase 1 does not implement multicore coherence.

## UVM work

1. Choose a simulator and a UVM library compatible with IEEE 1800.2. Keep scripts/configuration documented so another simulator can be supported later.
2. Create the testbench top, clock/reset generation, basic UVM test, environment, and report configuration.
3. Create CPU transaction classes for instruction reads and data reads/writes.
4. Create a processor-side driver/monitor skeleton and custom line-memory responder skeleton.
5. Create the byte-addressable backing-memory model and establish deterministic initialization.
6. Create a basic scoreboard transaction flow, even before the cache RTL is complete.
7. Add seed logging and a regression entry point.

## Deliverables

- Architecture specification.
- Interface specification.
- Module/block diagram.
- Parameter/configuration document.
- UVM test plan and coverage plan.
- Repository with compile, lint, simulation, and regression scripts.
- Empty/skeleton UVM environment that runs and reports a passing reset/smoke test.

## Exit criteria

- RTL and UVM compile/elaborate with the selected simulator.
- Reset and handshake semantics are unambiguous.
- CPU and line-level transactions are defined in one shared package/interface contract.
- A trivial memory-model test passes.
- No Phase 1 feature remains dependent on an unresolved protocol ambiguity.

---

# Phase 1 — Single-core blocking inclusive L1/L2/L3 hierarchy

## Goal

Implement and verify a complete single-core hierarchy with separate L1I/L1D, unified private L2, and unified L3. All caches are physically addressed, inclusive, 64-byte-line, pseudo-LRU, and blocking. Writes propagate through L1D/L2 to L3; L3 writes dirty lines back to external memory.

## RTL scope

### 1. Topology

- Elaborate `cache_hierarchy_top` with `NUM_CORES=1`.
- Instantiate one L1I, one L1D, one private unified L2, and one shared L3.
- Connect L1I and L1D to L2 through a two-input round-robin arbiter.
- Connect L2 to L3 through the custom line-level interface.
- Connect L3 to the custom full-line memory interface.
- Instantiate a directory/coherence placeholder or keep the interface boundary stubbed; no inter-core protocol transactions are required in this phase.

### 2. Cache storage and lookup

Implement the baseline geometries:

- L1I: 16 KiB, 4-way, 64-byte lines.
- L1D: 16 KiB, 4-way, 64-byte lines.
- L2: 256 KiB, 8-way, 64-byte lines.
- L3: 2 MiB, 16-way, 64-byte lines.

For every cache:

- Tag lookup across all ways of the indexed set.
- Valid bits and tag storage.
- Full 64-byte data-line storage.
- Byte-selective update logic.
- Invalid-way-first allocation.
- Tree pseudo-LRU replacement.
- Blocking request/miss/refill/response FSM.

Metadata/data memories must be written in a synthesizable style. Do not reset the data arrays; clear valid bits and control metadata.

### 3. L1I behavior

- Read-only CPU interface.
- Read hit returns requested bytes from the resident line.
- Read miss requests a complete line from L2, allocates it, updates PLRU, and responds to the CPU.
- A write request on the instruction port is rejected as a protocol error.
- L1I never marks a line dirty and never sends write-through data.

### 4. L1D behavior

- Supports CPU reads and writes.
- Read hit returns requested bytes.
- Read miss fetches and allocates a full line from L2.
- Write hit sends the byte update downstream to L2 and completes only after the L2 transaction succeeds; the local line is updated consistently with the completed lower-level write.
- Write miss does not allocate in L1D; it forwards the masked byte write to L2.
- L1D has no dirty bit in the baseline.

### 5. L2 behavior

- Unified instruction/data line store.
- Accepts requests from L1I and L1D through a round-robin arbiter.
- Read hit services L1 from L2.
- Read miss requests a full line from L3, allocates it, and returns the line to the requesting L1.
- Write hit updates the matching bytes and forwards the update to L3; the write completes only after L3 acknowledges.
- Write miss does not allocate in L2 and forwards the write to L3.
- L2 does not hold dirty lines in this phase.
- On L2 eviction, invalidate any matching L1I/L1D copies before reusing the L2 entry. This is required for the inclusive property even in a single-core system.

### 6. L3 behavior

- Unified 2 MiB, 16-way cache with 64-byte lines.
- Read hit returns the requested line.
- Read miss requests a full line from external memory, allocates the line, and marks it clean.
- Write hit merges byte-enabled data and marks the line dirty.
- Write miss fetches the complete old line, allocates it, merges the write, and marks it dirty.
- On a clean victim, reuse without a RAM write.
- On a dirty victim, write the complete 64-byte victim line to external memory and wait for write completion before reusing the entry.
- Only one L3 line transaction is active at a time.

### 7. Control, errors, and reset

- All request/response valid-ready rules from Section 6 must be implemented.
- Hold valid response fields stable during response backpressure.
- Hold request payload stable while waiting for ready at every interface.
- Return one response per accepted CPU request.
- Propagate external-memory errors to the original CPU requester.
- Do not install a refill line as valid if the memory read fails.
- Do not silently commit a write if the required lower-level operation reports an error.
- Reset all valid bits, PLRU state, FSM state, response-valid state, and pending-operation state.

## UVM verification scope

### A. Basic functionality

- Reset and post-reset invalid state.
- Instruction fetch from an initially empty hierarchy.
- Data load from an initially empty hierarchy.
- Store to an empty hierarchy.
- Repeated reads to exercise L1I/L1D hits.
- Repeated reads after eviction/re-fill.
- L1I and L1D simultaneous request attempts to exercise the L2 arbiter.
- Request/response backpressure at CPU and memory interfaces.

### B. Hit/miss and allocation policy

- L1I read hit and read miss.
- L1D read hit and read miss.
- L1D write hit: verify write propagates to L2 and L3.
- L1D write miss: verify no L1D allocation and write reaches lower levels.
- L2 read hit and read miss.
- L2 write hit: verify update reaches L3.
- L2 write miss: verify no L2 allocation.
- L3 read hit and read miss.
- L3 write hit and write miss.
- Verify L3 partial write miss performs line fetch before merge.
- Verify instruction-side requests never generate writes.

### C. Replacement and eviction

- Fill all ways of one set.
- Access ways in controlled order and check tree-PLRU victim selection against an independent reference algorithm.
- Verify invalid ways are consumed before valid-way eviction.
- Force L1 eviction and verify L2 retains its inclusive copy.
- Force L2 eviction and verify local L1I/L1D copies are invalidated.
- Force clean L3 eviction and verify no external write occurs.
- Force dirty L3 eviction and verify exactly one full-line writeback with the correct address/data.
- Verify dirty L3 victim is not reused before writeback completes.

### D. Data correctness

- Byte, halfword, and word reads/writes.
- Every legal byte lane and every legal strobe pattern for the CPU data width.
- Partial update of an existing line without corrupting untouched bytes.
- Multiple writes to different byte lanes of the same line.
- Read-after-write and write-after-read behavior.
- Deterministic randomized address/data traffic.
- Same-line accesses and different-line accesses mapping to the same set.
- Requests at first and last legal byte positions in a line.
- Unsupported size, misalignment, and invalid L1I write handling.

### E. Memory model and error handling

- Fixed-latency line reads and writes.
- Randomized line-read and line-write latency.
- Memory request backpressure.
- CPU response backpressure.
- Read error propagation.
- Writeback error propagation.
- Confirm failed refill does not install a valid cache line.
- Confirm an error does not silently corrupt the golden memory model.

### F. Random and stress testing

- Constrained-random instruction reads and data reads/writes.
- Random hit/miss mix by selecting working sets of different sizes.
- Conflict-heavy traffic mapping many addresses to the same set.
- Streaming traffic larger than L1/L2/L3 capacity.
- Repeated hot-set traffic.
- Random response latency and backpressure.
- Long regression with multiple fixed seeds.

## Phase 1 assertions

At minimum, implement and exercise assertions for:

- Request payload stable while `req_valid && !req_ready`.
- Response payload stable while `resp_valid && !resp_ready`.
- No response without a previously accepted request.
- Exactly one response per accepted request.
- No second request accepted by a blocking cache while it is busy.
- A cache hit must not trigger an unnecessary lower-level line fetch.
- A read miss must not respond before the refill succeeds.
- A dirty L3 victim must not be overwritten before successful writeback.
- L1/L2 dirty bits are absent or remain zero in the selected write-through design.
- Invalid cache lines cannot generate architecturally visible hit responses.
- Reset clears valid/pending/response state.

## Phase 1 functional coverage

Cover and cross at least:

- Cache level × operation type (read/write where legal) × hit/miss.
- L1D write hit/miss × downstream L2 hit/miss.
- L2 read miss × L3 hit/miss.
- L3 miss × memory response latency bucket.
- L3 eviction × dirty/clean victim.
- Replacement × invalid-way/full-set case.
- Access size × byte-lane/strobe pattern.
- Request concurrency × arbiter winner (L1I/L1D).
- Backpressure location × transaction type.
- Reset timing relative to idle/busy/response states.

Coverage should be sampled from observed transactions and protocol events, not inferred from a test name.

## Phase 1 deliverables

- Synthesizable single-core L1I/L1D/L2/L3 RTL.
- Generic cache engine and tree-PLRU implementation.
- Custom line-memory interface and UVM memory responder.
- Full-hierarchy UVM environment and independent golden memory scoreboard.
- Directed tests, constrained-random tests, assertions, and functional coverage.
- Regression script with reproducible seeds and pass/fail summary.
- Block-level waveform/debug guide.

## Phase 1 exit criteria

- All directed tests pass.
- Constrained-random regressions pass across a documented seed set.
- No unexplained assertion failures.
- Functional coverage goals for all listed Phase 1 scenarios are met or have documented waivers.
- Scoreboard reports no data mismatches, missing responses, duplicate responses, or unexplained memory traffic.
- Dirty L3 writeback behavior is proven by forcing eviction and checking backing RAM.
- RTL elaborates for the baseline parameter set and passes the selected lint rules.

---

# Phase 2 — Two-core directory-based coherence

## Goal

Extend the verified single-core hierarchy to two cores. Each core retains private L1I/L1D/L2 caches. Both private L2s share L3 and communicate with a separate directory controller. Implement the I/S/E write-invalidate protocol described in Section 5.

## RTL scope

1. Elaborate the hierarchy with `NUM_CORES=2`.
2. Instantiate two independent private cache clusters.
3. Instantiate one shared L3 and one separate directory controller.
4. Add one coherence endpoint per private L2.
5. Add a two-input round-robin arbiter between the two private L2 request ports and the shared L3 data request port. Retain the winning core identity until response completion and route the response to the correct L2.
6. Add directory metadata per L3 line: sharer vector, exclusive/shared interpretation, and transient state.
7. Implement shared-read, exclusive/upgrade, downgrade, invalidate, invalidate acknowledgement, and voluntary-drop behavior.
8. Implement L3 back-invalidation on eviction.
9. Ensure the directory can process coherence messages independently of a private L2's ordinary data-path wait state.
10. Serialize transactions at the shared L3 in this blocking phase.
11. Keep private L1/L2 caches write-through; keep L3 write-back to RAM.

## Required state invariants

- A line may be held `E` by at most one core.
- If two or more cores hold a line, every such private L2 must be `S`.
- A directory sharer bit must be set for every core whose L2 contains the line.
- A directory sharer bit may be conservatively stale only if the protocol safely sends a no-op invalidation and clears it; false negatives are forbidden.
- A private L1I/L1D line must be backed by the same core's inclusive L2 line.
- Every private copy must contain data consistent with L3 after a completed write-through transaction.
- A writer cannot complete a store while another core still has a valid copy that should have been invalidated.
- L3 cannot evict a line until all recorded sharers have acknowledged invalidation.
- No private cache may retain a dirty line relative to L3.

## UVM verification scope

### A. Basic sharing

- Core 0 reads a line; verify it receives E.
- Core 1 reads the same line; verify Core 0 downgrades to S and Core 1 receives S.
- Multiple reads by both cores verify stable shared copies and no unnecessary invalidations.
- A read to a different line should not affect the first line's directory state.

### B. Write upgrades and invalidations

- Core 0 and Core 1 both hold S; Core 0 writes; verify Core 1 receives invalidation and acknowledges before Core 0's store completes.
- Reverse the writer and repeat.
- Core 0 holds E and writes repeatedly; verify no unnecessary remote invalidation is generated.
- Core 0 holds E; Core 1 reads; verify downgrade E→S and both copies receive current data.
- Core 0 holds S; Core 1 writes; verify Core 0 is invalidated.
- Verify writer's next read returns its newly written value.
- Verify invalidated core's next read refetches and returns the newest value.

### C. No-write-allocate/coherence interaction

- Core 0 holds a line in S; Core 1 writes the line while it has an L2 miss; verify the directory invalidates Core 0 even though Core 1 does not allocate the line in L2.
- Verify the directory does not incorrectly mark a no-write-allocate requester as a sharer.
- Verify subsequent read from either core returns updated data.
- Core 0 has L1D hit but L2 hit; issue a write and verify upgrade/invalidations occur at the L2 coherence boundary.

### D. Inclusion and eviction

- L2 eviction invalidates local L1I and L1D copies and clears the core's directory sharer bit.
- L3 eviction sends invalidations to every sharer.
- L3 dirty eviction waits for all invalidation acknowledgements before writing the dirty line to RAM and reusing the way.
- A non-sharer receives no invalidation, unless a documented conservative directory false positive exists.
- A no-op invalidation to a nonresident L2 line is acknowledged safely.

### E. Concurrency and deadlock testing

- Both cores issue reads to the same line at nearly the same time.
- Both cores issue writes to the same line at nearly the same time.
- One core reads while the other writes the same line.
- One core writes a line while the other accesses a different line.
- Invalidation arrives while target L2 is blocked waiting on a different line's L3 response.
- Invalidation arrives while target L1I or L1D is backpressuring its response path.
- L3 is backpressured by the memory model during a dirty eviction.
- Randomized delays are inserted into invalidation and acknowledgement paths.
- Add a watchdog so any coherence transaction that fails to complete is reported as a deadlock/timeout, not an indefinite simulation hang.

### F. Coherence stress/random

- Random read/write traffic from both cores over a shared address pool.
- Separate private address regions to verify unrelated traffic does not create false sharing/invalidation.
- Shared hot-line contention.
- Alternating writer ownership.
- Producer/consumer pattern in which one core writes data and the other repeatedly reads it.
- Randomized request gaps and backpressure.
- Long-run invariants checked by the scoreboard after every completed operation.

## Phase 2 assertions and coverage

Assertions must check:

- No two cores have E for the same line.
- E-to-S transition occurs on a second core's read.
- Write permission is not granted before all required invalidation acknowledgements.
- A completed write is visible to subsequent coherent reads.
- L3 eviction is blocked until all sharers acknowledge.
- L2 invalidation clears matching L1 copies.
- Directory state and externally observed L2 presence remain consistent.
- Invalidation/acknowledgement messages are not duplicated or lost.
- Coherence message payload/address remains stable under backpressure.
- Progress occurs under fair L2/L3/memory response assumptions.

Coverage crosses must include:

- Requester core × current sharer count × requested operation.
- Previous state (I/S/E) × next state (I/S/E).
- Read share × E downgrade.
- Write upgrade × number of invalidated sharers.
- L3 eviction × sharer count × dirty/clean.
- Invalidation target busy/idle × target line hit/miss.
- Same-line race type (R/R, R/W, W/W).
- No-write-allocate store × existing sharers/no sharers.

## Phase 2 deliverables

- Two-core coherent RTL hierarchy.
- Separate directory controller and two coherence endpoints.
- Updated line/coherence interface specification.
- Updated UVM virtual sequences coordinating both cores.
- Independent coherence reference model and scoreboard checks.
- Coherence assertions and coverage report.
- Regression showing no stale-data, duplicate-exclusive-owner, directory, or deadlock failures.

## Phase 2 exit criteria

- All Phase 1 tests still pass.
- All directed coherence scenarios pass.
- Randomized two-core regressions pass over documented seeds.
- All coherence invariants hold throughout simulation.
- Every invalidation requiring an acknowledgement receives exactly one acknowledgement.
- No writer completes before required invalidations and L3 update completion.
- No deadlocks occur under the defined fair-response assumptions.
- Coherence functional coverage reaches the agreed closure targets.

---

# Phase 3 — Parameterized N-core hierarchy and configurable geometry

## Goal

Generalize the working two-core design so the number of cores is controlled by `NUM_CORES`. Expand configuration carefully without making the parameter space unmanageable.

## RTL scope

1. Remove hard-coded core-0/core-1 references from RTL.
2. Use generate loops and indexed arrays/interfaces for per-core L1I/L1D/L2 clusters and coherence endpoints.
3. Size the directory sharer vector from `NUM_CORES`.
4. Parameterize the L1I/L1D-to-L2 arbiter instances per core.
5. Generalize the shared-L3 arbitration/request selection for all private L2 ports.
6. Use a documented arbitration policy for multiple L2 requesters to shared L3. Begin with round-robin arbitration.
7. Preserve a separate coherence-control path so L3/directory can reach any private L2 independent of normal data-request arbitration.
8. Add elaboration-time checks for legal parameter combinations.
9. Expose selected cache geometry parameters: line bytes, capacity, associativity, and/or number of sets. Avoid adding policy knobs without a use case.
10. Keep pseudo-LRU as the default and initially required replacement policy.

Recommended configuration set for validation:

- `NUM_CORES=1` regression compatibility.
- `NUM_CORES=2` coherence regression compatibility.
- `NUM_CORES=4` multi-sharer/contending regression.
- `NUM_CORES=8` scalability/elaboration stress if the chosen simulator runtime permits.

L3 capacity should be configurable in this phase. A small L3 relative to the aggregate private L2 capacity is still functionally legal when back-invalidation is correct, but can cause heavy upper-level eviction traffic. Record this as a configuration/performance consideration.

## UVM verification scope

- Parameterized number of CPU agents and virtual-sequencer handles.
- Tests at 1, 2, and 4 cores, with 8-core elaboration/stress where practical.
- Verify a line shared by every core.
- Verify one writer invalidates all other sharers.
- Verify a second reader forces E→S downgrade.
- Verify directory sharer-vector bit mapping for first, middle, and last core indices.
- Verify L3 arbitration fairness among all cores.
- Verify simultaneous traffic from all cores to distinct lines.
- Verify simultaneous traffic to one hot line.
- Verify repeated ownership migration among cores.
- Verify back-invalidation to all and only recorded sharers.
- Verify one core's traffic does not corrupt another core's private lines.
- Verify correct behavior after parameter changes to capacity/ways/sets/line size if those parameters are exposed.
- Add negative elaboration tests for illegal geometries (e.g. nonintegral set count, unsupported non-power-of-two ways for tree-PLRU, line size not a power of two, or CPU data width not divisible by 8).

## Phase 3 deliverables

- N-core RTL with no hard-coded two-core logic.
- Parameter/configuration package and elaboration checks.
- N-core UVM agent generation and virtual sequences.
- N-core directory/coherence coverage.
- Regression matrix across supported parameter configurations.
- Configuration guide explaining legal parameter combinations and trade-offs.

## Phase 3 exit criteria

- The same RTL source elaborates and passes at `NUM_CORES=1`, `2`, and `4`.
- The directory correctly represents all configured cores.
- All two-core tests remain passing.
- N-core random regressions pass with multiple deterministic seeds.
- L3 arbitration is starvation-free under sustained requests, subject to fair response assumptions.
- Parameter checks reject unsupported configurations with clear messages.

---

# Phase 4 — AXI4 Full external memory interface

## Goal

Replace the Phase 1 custom line-memory boundary with a documented AXI4 Full master interface while keeping L3's cache-policy behavior independent of AXI channel details.

## RTL scope

1. Implement `axi4_line_adapter` between the L3 line request interface and AXI4 Full.
2. Initially use `AXI_DATA_WIDTH=64` bits and `ADDR_WIDTH=32`.
3. Convert one 64-byte read line to an eight-beat incrementing AXI read burst.
4. Convert one 64-byte writeback line to an eight-beat incrementing AXI write burst.
5. For the initial 64-bit AXI data width, use 8 bytes per beat, `ARSIZE/AWSIZE=3` (log2 of 8 bytes), `ARLEN/AWLEN=7` (eight beats minus one), and `INCR` burst type.
6. Generate and check burst address, size, length, burst type, and final-beat signaling.
7. Handle independent AR, R, AW, W, and B channel handshakes.
8. Hold VALID/payload stable while READY is low.
9. Support one outstanding line transaction initially, with a fixed/single transaction ID.
10. Propagate AXI read/write response errors to L3 and the original CPU requester as appropriate.
11. Keep the Phase 1 custom line-memory wrapper selectable for fast unit/regression runs if useful.

## UVM verification scope

- AXI read of a complete line with no backpressure.
- AXI write of a complete line with no backpressure.
- Backpressure independently on AR, R, AW, W, and B channels.
- Randomized delay between address acceptance and data beats.
- Randomized R-channel beat gaps.
- Randomized W-channel beat acceptance.
- Verify exactly eight beats for a 64-byte line on a 64-bit bus.
- Verify `RLAST` and `WLAST` on the correct final beat.
- Verify line address alignment and burst increment behavior.
- Verify no illegal burst crossing; line-aligned 64-byte transfers should remain within legal boundaries.
- Verify L3 waits for B response before considering a dirty writeback complete.
- Verify L3 waits for the complete successful R burst before installing a refill line.
- Inject read and write response errors and verify propagation/no false line installation.
- End-to-end test: CPU store makes L3 dirty; force eviction; observe AXI writeback; later read fetches data from modeled RAM.
- Run full Phase 1 and Phase 2 regressions with the AXI-backed memory model.

## Phase 4 deliverables

- AXI4 Full line adapter RTL.
- AXI transaction documentation and parameter constraints.
- AXI UVM memory-controller/slave model.
- AXI protocol assertions and coverage.
- End-to-end dirty writeback/refill tests through AXI.

## Phase 4 exit criteria

- AXI adapter passes directed and constrained-random channel tests.
- No protocol assertion failures under randomized independent channel backpressure.
- Burst payload, address, size, length, and response behavior are correct.
- Full hierarchy passes with AXI as the only backing-memory path.

---

# Phase 5 — Non-blocking operation and advanced cache features

## Goal

Increase concurrency and performance only after blocking correctness and coherence are stable.

## Candidate features (implement incrementally)

1. Multiple outstanding misses using MSHRs in selected cache levels.
2. Merge multiple requests to the same outstanding miss (same-line miss merging).
3. Permit hits to other lines while a miss is outstanding.
4. Multiple outstanding L3-to-memory transactions.
5. AXI transaction IDs and out-of-order response handling.
6. Write buffers for write-through traffic and/or L3 dirty writebacks.
7. Refill/writeback overlap where ordering rules permit.
8. Better arbitration/QoS policies if starvation or latency requires them.
9. Explicit cache maintenance operations (invalidate, clean, flush) if needed for CPU integration.
10. Optional unaligned/split-line accesses and atomic operations only if they are deliberate project goals.

Do not implement all features simultaneously. Add one feature, update the reference model, add directed tests, close coverage, and only then continue.

## RTL scope for the first non-blocking increment

- Add a small configurable number of MSHRs to L1D and/or L2 (start with two).
- Each MSHR tracks line address, transaction type, destination way/set, waiting request(s), and refill status.
- Merge later requests to an already outstanding miss for the same line where semantics permit.
- Permit independent hits while a miss is pending.
- Preserve write-through ordering and coherence ownership rules.
- Do not allow a response to overtake another response where the CPU-side interface promises in-order behavior; otherwise define response tags/IDs explicitly.
- Extend L3 and memory adapter concurrency only after the upper-level MSHR model is stable.

## UVM verification scope

- Two different misses outstanding simultaneously.
- Hit to line B while miss to line A is pending.
- Two requests to the same missing line merge into one lower-level refill.
- Requests to different lines complete out of order internally but return in the interface's specified order or with correct IDs.
- MSHR exhaustion causes backpressure, not lost requests.
- MSHR freeing and reuse after completion.
- Error response frees/aborts the correct MSHR.
- Same-line read/write ordering under concurrent misses.
- Coherence invalidation of a line with a pending miss/refill.
- L3 eviction/writeback while other independent requests proceed.
- AXI IDs and out-of-order R/B response cases if multiple AXI transactions are enabled.
- Stress with randomized latency, arbitration, and backpressure.

## Phase 5 deliverables and exit criteria

- One or more advanced concurrency features implemented and documented.
- Updated transaction/state diagrams.
- Updated scoreboard/reference model supporting outstanding transactions.
- Assertions for MSHR allocation, matching, freeing, duplicate miss merging, and ordering.
- Coverage for MSHR occupancy, merge, hit-under-miss, reordering, and backpressure.
- No regressions in blocking-mode behavior (if blocking mode remains configurable).

---

# Phase 6 — Design-space exploration, portability, and implementation evidence

## Goal

Turn the verified hierarchy into a useful reusable IP and an evidence-backed architecture project. This phase emphasizes controlled experiments rather than adding features without measurable value.

## RTL/configuration work

- Expose and validate cache capacities, associativities, line size, and selected policy options.
- Keep a small set of known-good configurations for regression.
- Add clean storage wrappers that allow inferred FPGA RAM, behavioral simulation RAM, or ASIC SRAM-macro wrappers to be selected without changing cache control logic.
- Keep synthesizable RTL free of simulator-only constructs; place assertions/debug-only logic in bind modules or guarded verification code.
- Run lint, elaboration, and synthesis checks for supported configurations.
- If using an FPGA target, document inferred memory resources and any limitations caused by the chosen device. Do not claim ASIC physical implementation without an actual ASIC synthesis/PDK flow.

## Performance instrumentation

Collect counters or trace-derived metrics for:

- CPU instruction/data request count.
- L1I/L1D/L2/L3 hits and misses.
- Misses by cause where measurable (capacity/conflict/compulsory classification may be computed in the analysis model rather than RTL).
- L1D/L2 write-through transaction count and bytes.
- L3 dirty writeback count and bytes.
- Coherence invalidation/downgrade count.
- L3 back-invalidation count.
- Average and maximum miss latency.
- Cycles stalled due to each cache level.
- MSHR occupancy/merge metrics if Phase 5 is implemented.

## Workload experiments

Use reproducible synthetic workloads/traces:

- Repeated access to a small hot working set.
- Sequential streaming reads/writes.
- Strided accesses.
- Random accesses.
- Same-set conflict patterns.
- Producer/consumer shared-data pattern across cores.
- Read-mostly shared data.
- Write-sharing hotspot across cores.
- Working sets just below and just above each cache capacity.

For each experiment, record configuration, seed/trace, request count, hit/miss rates, memory traffic, latency, and any relevant timing/synthesis results.

## Phase 6 deliverables

- Configuration guide and supported-parameter matrix.
- Storage backend abstraction and synthesis evidence for at least one target flow.
- Performance counters or trace analyzer.
- Reproducible benchmark scripts and results.
- Architecture trade-off report explaining the measured effect of capacity, associativity, line size, replacement, coherence traffic, and advanced features.

## Phase 6 exit criteria

- All supported configurations elaborate and pass their designated regressions.
- Performance results are reproducible and tied to exact source/configuration revisions.
- Synthesis claims are supported by actual tool reports.
- The design remains modular and the UVM environment remains reusable across configurations.

---

## 10. Verification coverage plan by category

This section is the master verification checklist. The individual phase plans specify when each category becomes applicable.

### 10.1 Functional behavior

- Read hit/miss at every cache level.
- Write hit/miss at L1D, L2, and L3.
- Correct read/write allocation behavior.
- Correct write-through propagation.
- Correct L3 dirty tracking and RAM writeback.
- Correct line fill and byte merge.
- Correct tag/index/offset calculations.
- Correct invalid-way-first behavior and pseudo-LRU victim choice.
- Correct inclusive invalidation on L2/L3 eviction.

### 10.2 Protocol behavior

- Valid-ready handshakes.
- Stable payload under backpressure.
- Request acceptance and response pairing.
- No lost/duplicate transactions.
- Correct error propagation.
- Correct reset behavior during idle and busy states.
- Correct fairness for round-robin arbitration.

### 10.3 Coherence behavior

- E-to-S downgrade.
- S-to-exclusive upgrade.
- Invalidation fanout and acknowledgements.
- Directory update on fill, eviction, and invalidation.
- No duplicate E ownership.
- No stale read after completed write.
- L3 eviction with active sharers.
- Invalidation while target handles unrelated normal traffic.

### 10.4 Stress behavior

- High request rate.
- Same-set conflict pressure.
- Shared-line contention.
- Random backpressure at all interfaces.
- Random memory latency.
- Repeated resets at legal test boundaries.
- Long random sequences with deterministic replay.

### 10.5 Parameter behavior

- Address width legal configurations.
- CPU data width legal configurations.
- Core-count variations.
- Cache capacity/associativity/line-size variations when exposed.
- Illegal configurations rejected at elaboration.

## 11. Assertions and protocol checks

Use SystemVerilog Assertions where practical. Bind protocol assertions to interfaces or modules so the checks can be enabled in verification without polluting synthesizable logic.

The minimum assertion set should grow by phase:

| Phase | Required assertion focus |
|---|---|
| 0 | Request/response handshake contracts and reset |
| 1 | Blocking FSM, exactly-once response, refill/writeback safety, stable payload |
| 2 | Directory invariants, invalidation ordering, exclusive-owner uniqueness, coherence progress |
| 3 | Parameterized index ranges, sharer-vector correctness, arbitration fairness |
| 4 | AXI4 channel stability, burst shape, last-beat correctness, response sequencing |
| 5 | MSHR lifecycle, miss merging, ordering, ID matching, no resource leaks |
| 6 | Configuration invariants and storage-backend protocol equivalence |

Assertions should avoid depending on undocumented internal implementation details. Checks of internal state are acceptable when they directly verify a documented invariant, but end-to-end data correctness must remain independently checked by the scoreboard.

## 12. Suggested repository layout

```text
cache-hierarchy-ip/
  README.md
  docs/
    project_spec.md
    architecture.md
    interface_spec.md
    coherence_protocol.md
    verification_plan.md
    parameter_config.md
    decisions.md
    bug_log.md
  rtl/
    pkg/
      cache_types_pkg.sv
      cache_config_pkg.sv
    interfaces/
      cpu_cache_if.sv
      cache_line_if.sv
      coherence_if.sv
      memory_line_if.sv
      axi4_if.sv
    common/
      cache_engine.sv
      cache_tag_array.sv
      cache_data_array.sv
      tree_plru.sv
    cache_levels/
      l1_icache.sv
      l1_dcache.sv
      l2_cache.sv
      l3_cache.sv
    hierarchy/
      core_cache_cluster.sv
      l1_to_l2_arbiter.sv
      directory_controller.sv
      coherence_endpoint.sv
      cache_hierarchy_top.sv
    adapters/
      line_memory_adapter.sv
      axi4_line_adapter.sv
  tb/
    tb_top.sv
    env/
      cache_hierarchy_env.sv
      virtual_sequencer.sv
      cache_scoreboard.sv
      coherence_reference_model.sv
      functional_coverage.sv
    agents/
      cpu_agent/
      memory_agent/
      axi_memory_agent/
    sequences/
      base_sequence.sv
      instruction_sequences.sv
      data_sequences.sv
      coherence_sequences.sv
      stress_sequences.sv
    tests/
      reset_test.sv
      basic_functional_test.sv
      replacement_test.sv
      writeback_test.sv
      random_test.sv
      coherence_test.sv
      ncore_test.sv
      axi_test.sv
    assertions/
      cache_protocol_sva.sv
      coherence_sva.sv
      axi4_sva.sv
  scripts/
    compile.do
    run_test.sh
    run_regression.sh
    lint.sh
  regressions/
    phase1_smoke.list
    phase1_full.list
    phase2_coherence.list
    phase3_ncore.list
    phase4_axi.list
  results/
    .gitkeep
```

This is a suggested layout, not a requirement to create every file at the start. Add files when their phase begins.

## 13. Definition of done for every feature

A feature is not complete merely because its RTL simulates once. Mark a feature complete only when all applicable items are satisfied:

1. RTL implementation and interface behavior are documented.
2. Directed tests cover the normal path and important corner cases.
3. Scoreboard/reference model checks expected behavior independently.
4. Relevant assertions are enabled and passing.
5. Functional coverage points are sampled and the intended bins are hit.
6. At least one constrained-random scenario exercises the feature when randomization is applicable.
7. Regression is reproducible and passes with recorded seed/configuration.
8. A known bug found during implementation has a permanent regression test.
9. Lint/elaboration pass for the supported configuration.
10. Any known limitation is recorded rather than silently ignored.

## 14. Risks and design cautions

### 14.1 Write-through does not eliminate coherence

Even though L1D and L2 are write-through, other cores may hold stale copies. The directory must still invalidate or downgrade other sharers before a write is allowed to complete. Write-through simplifies dirty-data ownership in private caches; it does not remove permission management.

### 14.2 No-write-allocate writes still need invalidations

A write that misses in L1D and L2 may bypass both caches, but it must still invalidate any remote cached copies before L3 commits the new value. Do not equate “no allocation” with “no coherence transaction.”

### 14.3 Inclusive eviction must be ordered correctly

L2 cannot silently evict a line while a matching L1I/L1D copy remains valid. L3 cannot silently evict a line while any private L2 copy remains valid. The invalidation acknowledgement is part of the eviction completion condition.

### 14.4 Dirty L3 state is independent of private coherence state

L2 can be in `E` while L3 marks the line dirty relative to RAM. `E` means one private core subtree has exclusive permission. The L3 dirty bit means the L3 data differs from external RAM. These are separate concepts and must not share one bit/state encoding.

### 14.5 Reference-model independence

Do not make the scoreboard copy the DUT's FSM or inspect the DUT's tag/data arrays to determine expected values. Use the byte-addressable golden model and observed transaction ordering. The coherence reference model should represent protocol state abstractly.

### 14.6 Parameterization can expand the verification space quickly

Keep Phase 1 geometry fixed and well verified. Add a small, intentional configuration matrix later. Every exposed parameter multiplies elaboration and regression combinations.

### 14.7 AXI4 is not coherence

AXI4 Full is the external memory transaction protocol in this project. The directory controller remains responsible for cache-to-cache coherence. If the project later adopts a standardized coherent interconnect, evaluate ACE/CHI as a separate architectural extension rather than assuming AXI4 Full provides snooping or directory behavior.

## 15. Initial project decisions and deferred decisions

### Confirmed by project direction

- Reusable SystemVerilog cache-hierarchy IP.
- CPU and external memory controller/RAM are outside the RTL; UVM models them.
- Multiple cores supported through a parameter.
- L1I and L1D are separate; L2 and L3 are unified.
- L1/L2 are private per core; L3 is shared.
- Physical addresses.
- Write-through from L1D to L2 and L2 to L3; write-back from L3 to RAM.
- Pseudo-LRU replacement.
- Blocking cache for the initial implementation.
- Custom internal interfaces first; AXI4 Full at the external memory boundary later.
- Full-hierarchy UVM verification.
- Coherence controller/directory is separate from L3 cache data logic.
- Balanced RTL/DV portfolio with later research/performance extensions.

### Working defaults to freeze in Phase 0

- 32-bit physical address width.
- 32-bit CPU-side data width.
- 64-byte cache line.
- L1I/L1D: 16 KiB, 4-way.
- L2: 256 KiB, 8-way.
- L3: 2 MiB, 16-way.
- Tree pseudo-LRU.
- Inclusive L1-to-L2 and L2-to-L3 hierarchy.
- Read-allocate at every level.
- No-write-allocate at L1D and L2.
- Write-allocate at L3.
- Directory-based write-invalidate coherence with private L2 `I/S/E` states.
- Round-robin arbitration between L1I/L1D at private L2 and between private L2 requesters at shared L3.
- One outstanding transaction per blocking cache; one active L3 line transaction globally.
- One outstanding AXI line transaction in the first AXI adapter increment.

### Explicitly deferred choices

- Exact simulator/version, based on available licensed/tool-supported UVM implementation.
- Whether to add 64-bit CPU data width in the same project branch or as a later configuration.
- Exact cache-geometry parameter names and whether capacity or set count is the public parameter.
- Exact coherence opcode encodings and transient-state names.
- Whether self-modifying-code behavior gets an explicit I-cache invalidate/fence interface.
- MSHR count and ordering policy for non-blocking phases.
- Whether later coherent-interface work should study ACE or CHI.

## 16. Reference standards and useful documentation

These references are protocol/standard sources, not substitutes for the project-specific interface specification.

1. **Arm AMBA specifications:** overview of AMBA protocols, including AXI and CHI: <https://www.arm.com/architecture/system-architectures/amba/amba-specifications>
2. **Arm AMBA AXI and ACE protocol specification:** use the current Arm-hosted AXI protocol specification as the authority for the Phase 4 AXI adapter: <https://support.arm.com/documentation/ihi0022/latest/>
3. **IEEE 1800.2-2020:** Universal Verification Methodology language reference manual: <https://standards.ieee.org/ieee/62530-2/11429/1800.2/7567/>
4. **Accellera UVM resources:** UVM reference implementations and user documentation: <https://www.accellera.org/downloads/standards/uvm>
5. **RISC-V unprivileged ISA documentation:** example of a byte-addressable processor address space and byte/halfword/word terminology: <https://docs.riscv.org/reference/isa/>

---

## 17. First implementation milestone

The first coding milestone should not be the entire cache hierarchy. Start by completing Phase 0 and then implement this vertical slice in Phase 1:

1. CPU data request accepted by L1D.
2. L1D miss generates a line read to L2.
3. L2 miss generates a line read to L3.
4. L3 miss generates a 64-byte line read to the UVM memory responder.
5. The line is installed in L3, L2, and L1D.
6. The original CPU request receives the correct requested bytes.
7. A repeated request hits in L1D and causes no lower-level line read.

After this read path is proven, add L1I, byte writes, write-through propagation, L3 dirty tracking, and dirty eviction/writeback. This order creates a small, debuggable path through every level before the full behavior matrix is introduced.

**Phase 1 should be considered complete only after both read and write paths, replacement, inclusive eviction, L3 writeback, UVM scoreboarding, assertions, coverage, and regression criteria are satisfied.**
