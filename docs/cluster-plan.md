# cva6-multi — Design Decisions

**Status:** plan agreed, nothing built. Date: 2026-10-19.

Decision register for a stand-alone, cache-coherent quadcore RV64 cluster assembled from
open hardware. Gaps in the numbering are decisions not reproduced here.

---

## 1. Goal and scope

**Done means:** SMP Linux boots to a shell with all four harts online. This is chosen as a
forcing function rather than a feature — it is the cheapest test that catches integration bugs
which bare-metal tests silently tolerate. `nproc` returning 4 exercises coherence, IPIs,
per-hart PLIC contexts, the MMU, the timer and the device tree simultaneously.

**Phase 1 is Verilator only.** FPGA is a separate later phase. On FPGA the dominant work is the
DRAM controller, clocking and board plumbing, which is real but orthogonal to whether the cluster
functions. Simulation reaches a Linux shell with a behavioural AXI memory and none of it.

**In scope:** four coherent CVA6 harts, the Cheshire platform, L2, UART, interrupts, debug, boot.

**Not in scope:** multi-cluster topologies and any NoC, inter-cluster isolation or traffic
regulation, scratchpad partitioning, and any safety, redundancy or certification mechanism.
This is a single coherent cluster and nothing more.

---

## 2. Decisions

### Goal and target

- **D1 — Done = SMP Linux to a shell on 4 harts.** Not `riscv-tests` passing, not bare-metal AMP.
- **D2 — Verilator first, FPGA as phase 2.** Behavioural AXI memory in phase 1.
- **D3 — `NrHarts` parameterised throughout, default 4.** Cheap now; expensive to retrofit through
  the interconnect, PLIC contexts, CLINT `mtimecmp` array, debug module hart selection and the
  device tree generator. Also enables a 1-hart bring-up as an easier debug target.

### Architecture

- **D5 — Build the integration; adopt the coherence.** The SoC top, address map, AXI topology,
  peripheral wiring, interrupt routing, multi-hart debug plumbing, boot flow, device tree and
  testbench are ours. The Culsans MOESI/ACE CCU is adopted unmodified. Rationale: SoC integration
  and coherence-protocol design have wildly different cost curves — integration is tractable and
  debuggable with a clear success signal, a coherence protocol is a formal-methods problem.
  Four cores is also the top of Culsans' validated envelope, so staying at four means no
  coherence-directory work is required.
- **D6 — The integration being built is the Culsans-into-Cheshire join.** Cheshire is the platform;
  the coherent quadcore cluster is wired into it. Cheshire is single-core and Culsans is a
  multicore cluster on an older CVA6 — as far as we can determine, nobody has joined them.
- **D7 — RV64GC + Sv39 + M/S/U + Bitmanip.** No H, no V. Bitmanip arrives through the single `RVB`
  config bit, which covers Zba + Zbb + Zbc + Zbs with no finer granularity — **Zbc cannot be
  declined**, and its carry-less multiply XOR tree in the MUL pipe is the most likely source of FPGA
  timing pressure later. `ZKN` (Zbkb/Zbkx/Zkn) is available only on a current CVA6.
  - **The device tree and the hardware must agree about bitmanip, or Linux dies without a usable
    message.** Demonstrated on the virtual platform: with `zba`/`zbb`/`zbc`/`zbs` in
    `riscv,isa-extensions` but a core that does not implement them, Linux 6.18 patches in its
    Zbb-optimised `strcmp`/`strlen`/`strncmp` through `apply_boot_alternatives()` and then executes
    those instructions on a core that lacks them. The boot stops silently immediately after
    `riscv: ELF capabilities`, with no panic and no trace — the next expected line,
    `Kernel command line:`, simply never appears.
  - **This is a live risk, not a modelling curiosity.** Cheshire's default build targets a config
    package with `CVA6ConfigBExtEn = 0`, so a stock Cheshire plus a device tree written from D7
    reproduces exactly this failure. Whoever flips `CVA6ConfigBExtEn` and whoever writes the device
    tree must be the same decision.
  - Contrast with an *unimplemented CSR*, which is benign: a write to a missing `mhpmcounter` is
    ignored with a warning and the boot continues. A missing **instruction** is fatal. Advertising
    `zihpm` without the counters cost nothing; advertising `zbb` without the instructions cost the
    whole boot.
- **D8 — CLINT + PLIC.** Not AIA. See §4 for why this is not a simple swap. Reserve the IMSIC
  address ranges in the map now and leave the interconnect a spare master port for an MSI-mode
  APLIC — that is free today and converts part of any later swap from an architecture change into
  a module change.
- **D9 — Bring up on CVA6's native write-back `std_cache`; migrate to HPDcache later.** Culsans'
  ACE snoop extensions live in `std_cache` — that is the combination with a published paper behind
  it. HPDcache has no snoop support in Culsans today, so adopting it first would mean any hang has
  the integration, Culsans and an unproven cache pairing as simultaneous suspects.
- **D10 — Peripheral set:** UART, CLINT, PLIC, debug module, boot ROM, `axi_llc`, iDMA. Off in
  phase 1: VGA, I2C, SPI, GPIO, serial link, CLIC. SPI returns for FPGA SD-card boot.
- **D11 — The address map is authored to be relocatable.** Internals position-independent, upper
  bits reserved for a cluster index, holes left for apertures phase 1 does not implement. Cheap
  to decide on day one, painful to retrofit once software starts hardcoding addresses.

### Dependencies

- **D12 — Every IP dependency is pinned at an explicit commit SHA**, bumped deliberately. Not a
  tracking branch. During integration, a break caused by an upstream push destroys the ability to
  attribute a regression to one's own last change — the thing most worth protecting when the
  integration itself is the deliverable.
- **D13 — The Culsans fork this project tracks is rebased onto a current CVA6.** This is
  load-bearing: it restores `misa.B` reporting (so Spike step-and-compare works), brings the
  upstream bitmanip decoder fixes, and makes `config_pkg.sv` and `ZKN` available.

### Boot and debug

- **D15 — Preload a single OpenSBI `fw_payload.elf`** (OpenSBI + kernel + DTB + initramfs) into the
  behavioural memory at reset. No boot ROM path, no SPI/SD model, no U-Boot, no filesystem in
  phase 1.
- **D16 — Hart 0 boots; harts 1–3 park in `wfi` awaiting OpenSBI HSM start.** Letting the other
  three free-run into the kernel produces corruption that looks random and is painful to diagnose.
  **The boot hart is not hart 0 by default** — observed on QEMU `virt` booting on hartid 3 in one
  configuration and hartid 0 in another, from identical images. Whichever hart arrives first wins
  unless something constrains it, so this is a platform obligation, not an assumption.
- **D17 — Instantiate the RISC-V debug module and JTAG DTM from the start; defer bring-up to FPGA.**
  Instantiating is nearly free and avoids re-plumbing hart selection later (another `NrHarts`
  consumer). Driving it under Verilator means modelling JTAG transport and running OpenOCD against
  the simulation — slow, and rarely worth it against full waveform visibility.

### Verification

- **D18 — Litmus tests on RTL are a gate before the first SMP Linux attempt.** A coherence bug found
  by Linux hanging is close to undiagnosable: the symptom appears thousands of instructions after
  the violating interleaving, in kernel code nobody here wrote, on four harts at once. The same bug
  found by a litmus test is a ten-line repro with an exact failing interleaving.
- **D19 — Three verification layers before Linux:** `riscv-tests` for the single-hart baseline,
  Spike step-and-compare for per-hart divergence, litmus for coherence.
- **D20 — Four milestones, each a working system.** See §5.

### Environment

- **D21 — Tooling is provisioned reproducibly** (nix or equivalent) and declared in-repo, so the
  environment is shared rather than reconstructed per machine. QEMU is included deliberately:
  validating the OpenSBI/kernel/initramfs stack there before subjecting it to Verilator avoids
  debugging software bugs at RTL simulation speed.

### Address map and decode

- **D22 — Everything the cluster owns lives inside Cheshire's external window
  `[0x2000_0000, 0x8000_0000)`.** Not a preference: Cheshire derives CVA6's PMA attributes in
  `gen_cva6_cfg()` from only EXTROM, BOOTROM, SPM, SPM_UNC, the LLC-out region, and the CIE/non-CIE
  split of that window. A region outside it gets **no** cacheability, executability or idempotency
  attribute from any rule. Carfield confirms this empirically — every one of its islands sits inside
  the same window.
- **D23 — The cluster ID is the AIA group index, `E = 25`, at `addr[27:25]`.** Cluster *N*'s global
  aperture base is `0x5000_0000 + (N << 25)`, 32 MiB each. The AIA group-index scheme
  (`(g << E) | A | (h << C)`, `riscv,group-index-shift` default 24, up to 7 group bits) is the only
  ratified RISC-V convention placing a cluster ID in a physical-address bit field, and its alignment
  rules on `A` exist specifically so `+` can be implemented as bitwise `|`. The spec names the use
  case directly: *"each group exists on a separate chip (or chiplet in a multi-chip module)."*
- **D24 — Two cluster-local aliases, one either side of the CIE split.** Cheshire provides exactly
  one CIE split point in the external window, so the cacheable alias (scratchpad, `0x2000_0000`) sits
  below it and the non-cacheable alias (per-cluster MMIO, `0x4000_0000`) above it, via
  `Cva6ExtCieOnTop = 0` and `Cva6ExtCieLength = 0x0400_0000`. Both stay position-independent, which is
  what lets one kernel image and device tree run on any cluster unchanged. Precedent: Snitch's
  `alias_region_enable` gives every internal subordinate two crossbar rules, one cluster-specific and
  one at a fixed cluster-independent base — and with it enabled `boot_addr` becomes unused, because
  every cluster boots from its own alias address.
- **D25 — `mhartid = (cluster_id << HartIdShift) | core_index`, with `HartIdShift = 4`.** Four bits
  for the core field, not two: the coherence unit caps the cluster at four cores today, but a
  directory-based successor would allow up to sixteen, and a two-bit field forecloses it. Phase 1 is
  unaffected, since cluster 0 gives harts 0–3 under either width. This is also the decomposition the
  AIA uses — APLIC computes `g = (HartIndex >> LHXW) & (2^HHXW−1)` and `h = HartIndex & (2^LHXW−1)`
  with `LHXW = HartIdShift` — so the hart id, the AIA hart index and the cluster field of the address
  are one encoding. **`HartIdShift`, `riscv,hart-index-bits`, and the IMSIC reservation size are the
  same number**, and the reservation is the one that is cheap to oversize now, so the map reserves the
  sixteen-core geometry. Sparse hart ids are safe: Linux stores them as opaque values in
  `cpuid_to_hartid_map[]` with no density requirement, OpenSBI indexes through `hart_index2id`, and
  Carfield ships sparse ids under both. Note OpenSBI's `SBI_HARTMASK_MAX_BITS` defaults to 128, which
  8 x 16 harts exactly fills.

- **D26 — An 8 MiB `axi_llc` array in 8 ways of 1 MiB, partitionable up to 7 MiB scratchpad and
  1 MiB L2.** `axi_llc` partitions by way and selects SPM ways through a config register, so eight
  1 MiB ways give 1 MiB partition granularity with "at least one way stays cache" as the floor.
  Phase 1 configures 1 way scratchpad and 7 ways L2 — nothing consumes the scratchpad yet, but one
  SPM way exercises the way-partitioning path and the local-alias decode against a real region.
  Sizing: 1 MiB per way at 64-bit data width means `NumLines × NumBlocks = 131072`, e.g. 64-byte
  lines and `NumLines = 16384`. Note this is **64× Cheshire's default SPM** of 128 KiB.
  The cluster gets its own `axi_llc` and Cheshire's is kept as a platform LLC: Cheshire's instance
  sits in front of DRAM at a fixed address in the static region, which is the wrong slot for a
  per-cluster L2 and private scratchpad behind the CCU. That leaves two non-coherent layers below the
  CCU, so the window region must lie outside the cached range of **both** — `axi_llc` decides what to
  cache by address range rather than by AXI cache attributes, making this a configuration property.
  Note also that the SPM cached region's nominal size in `cheshire.rdl` is 64 KiB while its real
  backing is `get_llc_size(cfg)`; routing is correct but generated constants understate it. Free under
  Verilator; a serious constraint on FPGA, where it exceeds the total block RAM of many parts.
- **D27 — Structural decode inside and at the cluster boundary; programmable windows only at
  platform level.** Cluster-local aliases and cluster-ID routing are fixed structural decode —
  programmability there buys nothing and introduces a way for one cluster to differ from another.
  Base-and-size programmable windows (the PASW pattern, as found on various SoCs) are the right
  answer for DRAM extent and any PCIe or flash aperture, because those are boot-time discoveries.
  Consequence now: **every platform-level region is a single naturally-aligned power-of-two block**,
  so it can be converted to a window later without moving anything.
- **D28 — Write our own elaboration assertions.** `cheshire_soc.sv` lines 1783–1796 are all TODO
  comments; nothing checks external region overlap, legality, or consistency with the CVA6 PMA
  config. Assert on overlap, CIE boundary alignment, and aperture alignment against the AIA
  constraint.
- **D29 — Pin the Cheshire configuration and assert on `ext_base`.** External crossbar ports occupy
  the top indices after all *enabled* internal ports, so toggling `Dma` or `SerialLink` silently
  renumbers them. The caps are 15 external AXI subordinates and 15 rules.
- **D30 — Culsans' SoC top level is discarded; only the CCU and the coherent CVA6 wrapper are
  taken.** `culsans_top.sv` ships a full SoC with its own hardcoded CLINT at `0x0200_0000` and PLIC at
  `0x0C00_0000` that would collide with Cheshire's. D6 integrates `ace_ccu_top_intf`, not the Culsans
  platform.

### Scope of this repository

- **D31 — This repository is documentation only, permanently.** The implementation depends on IP
  hosted where an external reader cannot reach it, so nothing here could ever be built from a public
  clone, no matter how it were arranged. Rather than publish something that looks buildable and is
  not, the public copy carries the design record alone: the decisions, the address map, the AIA
  survey, the component list and the milestones, each of which stands on its own. There is no RTL
  here and no Bender manifest, and there will not be.

### Virtual platform

- **D35 — The virtual platform is staged, and functional only.** Three vehicles, each with a
  different job:
  1. **QEMU `virt`** for the half of the software stack that does not depend on our map — kernel
     config, initramfs, the HSM multi-hart boot path, and D7's ISA as advertised in a device tree.
     Done: `sw/qemu-virt-check.sh`, 4/4 + 5/5 + 13/13.
  2. **Renode** for **our** map. Done: `sw/renode-check.sh`, 14/14 — OpenSBI 1.6 and Linux 6.18.7
     boot on the D22-D25 map with four harts online. Chosen because `.repl` platform descriptions
     are declarative, so a custom address map costs tens of lines rather than hundreds of lines of C.
  3. **A QEMU machine** for daily software work, when boot time starts to hurt. Not started. The
     shortcut is Ben Dooks' CVA6 machine series (v4, September 2025, reviewed but never merged):
     a working CVA6 + CLINT + PLIC + 16550 machine whose memmap table can be swapped for ours.
  - **Both vehicles are dynamic binary translators.** Renode's `CPU.RiscV64` is a C# wrapper over
    `translate-riscv64-le.so`, its tlib engine, which is derived from QEMU's TCG. Instruction
    semantics are modelled; pipeline, caches, bus arbitration and timing are not. `cpuType:
    "rv64gc_zba_zbb_zbc_zbs"` selects decode paths and is the *only* respect in which the core
    resembles CVA6 — no HPDcache, no `std_cache`, no ACE port, no coherence unit at all.
  - **Therefore it validates the map, the device tree and the software, and nothing about the
    hardware.** A functional model that boots cleanly while the RTL has an ordering bug is the
    expected outcome. D18's litmus gate stays on RTL.
- **D36 — Renode hosting Verilated RTL is the candidate bridge to phase 1.** Renode ships a
  `CoSimulationPlugin`, and the idiom extends to the **CPU**, not only peripherals: Verilator
  compiles the RTL into a shared library and Renode loads it in place of a tlib core. Their
  `verilated_ibex.resc` shows it as one line — `cpu SimulationFilePathLinux <libVcpu_ibex.so>`.
  - **Why it is attractive.** The cluster could be Verilated as the CPU complex while Renode keeps
    supplying the peripherals and the address map that already boots. That is an incremental route
    from a validated map to real RTL, without first building a complete Verilator testbench and
    peripheral set — which is otherwise the long pole in D2's phase 1.
  - **Why it is a candidate and not a decision.** It needs the RTL to exist, so it is gated on the
    Culsans pin (D12, D13). It competes with a plain Verilator testbench, which has no C# in the
    loop. And the published example is Ibex, a small RV32 core: a four-hart coherent RV64 cluster is
    a far larger Verilated object, so throughput collapses toward plain Verilator and the hosting
    buys less.
  - **It does not relax D18.** Verilating the cluster inside Renode still leaves the surrounding
    model non-architectural about memory ordering, so litmus evidence must come from a flow where
    the whole path is RTL.
  - **VCML is the stronger SystemC route, and probably the better bridge overall.**
    [machineware-gmbh/vcml](https://github.com/machineware-gmbh/vcml) is an Apache-2.0 SystemC/TLM-2.0
    modelling library, actively maintained — pushed 2026-10-20, 205 stars, one open issue, CI across
    Linux, Windows and macOS with ASan, lint and coverage. That is a different standard of upkeep
    from the other SystemC options surveyed.
    - **Its RISC-V peripheral set reads like a shopping list for this project**: `models/riscv/`
      carries `clint.h` and `aclint.h` and `plic.h` for D8 today, `aplic.h` for the AIA
      reconvergence path, and an `iommu.h` for later. The surrounding platform would
      be largely off the shelf rather than written.
    - **Verilator emits SystemC modules natively** (`verilator --sc`), so a SystemC platform is the
      natural host for Verilated RTL — more natural than Renode's C# `CoSimulationPlugin`. This
      solves the missing-CPU problem and the RTL-bridge problem with one decision, and it is why
      VCML displaces Renode as the preferred route for this step.
    - **It also supplies timing annotation**, which is wanted here rather than feared: D26's
      scratchpad-versus-L2 split and the window sizing both lack any basis, and a model that can
      produce numbers is the only way to put one under them.
    - **Three gaps, none fatal but all real.** VCML is a library, not a platform, so the VP is built
      rather than configured. It contains **no CPU** — `core/processor.h` is abstract
      (`virtual void simulate(size_t) = 0`) and nothing in the organisation implements RISC-V, because
      MachineWare's business is SIM-V, their commercial ISS; a Verilated core is what fills that hole.
      There is **no 16550** among its serial models (`sifive`, `pl011`, `cdns`, `uartlite`, `nrf51`),
      and Cheshire's UART is 16550-compatible, so one would have to be written. And `aplic.h` ships
      without an `imsic.h`, so its AIA support covers direct delivery only.
    - **Open before this becomes load-bearing:** MachineWare's licensing. The library is Apache-2.0
      but they advertise a separate "VCML community" offering, which usually signals a dual-licence
      arrangement worth understanding before depending on it.
    - **Not a replacement for either working vehicle.** Renode boots the map at 14/14 and the QEMU
      machine at 15/15 in under 10 s. Building a VP from a library, writing a 16550 and sourcing a
      core, to reach a capability already in hand, is not the trade. This is recorded for when the
      RTL exists and is pinned (D12, D13).
  - Recorded because it was under-rated when the vehicles were first ranked: Renode's
    co-simulation was assumed to cover peripherals only.

- **D37 — The address map has one machine-readable source, and every view is generated from it.**
  `sw/addrmap/cva6-multi-map.json` is authoritative; `sw/addrmap/gen.py` emits the device tree, the
  Renode platform and the QEMU board's memmap header. `gen.py --check` prints a diff and fails if
  any generated file is stale, which belongs in CI next to the strip-list grep (D34).
  - **It exists because the map drifted and the drift was expensive.** The same map lived by hand in
    four places: prose, the device tree, the Renode platform and a C table. The device tree declared
    `reg-shift = <2>` on the UART while the QEMU model was byte-spaced, which cost several debugging
    rounds and produced a confidently wrong conclusion along the way. The failing access was a store
    fault at `base+0xC`; under Renode the same mismatch only produced warnings, so the louder model
    found it and the quieter one hid it.
  - **Properties belong in the source, not just addresses.** `reg_shift`, the 1 MHz timebase, the
    PLIC's source count and contexts-per-hart, the UART IRQ, and the ISA extension list are all
    things two consumers must agree on, and each is now stated once. The bitmanip list matters most:
    per D7, a model that does not implement what the tree advertises dies silently.
  - **Reserved span and populated size are distinct fields.** `size` is address space, `populated`
    is backing store, and they differ for the scratchpad: 8 MiB of array inside 64 MiB reserved
    (D26). Consumers that instantiate memory use `populated`; the device tree's `reserved-memory`
    node uses it too. This distinction came out of the generator getting it wrong first, which only
    the diff against known-good output caught.
  - **Stdlib only, JSON not YAML.** No pyyaml here and Python is 3.10 with no `tomllib`. A generator
    that needs a dependency installed is a generator that will not run on the machine that needs it.
  - **This is the lightweight precursor to D38's heavier flows**, not a competitor to them. It owns
    addresses and cross-consumer properties. It does not describe structure, connectivity or register
    fields, and is not trying to.
- **D38 — SystemRDL when the first register block is ours; IP-XACT when RTL assembly starts.**
  Neither yet, and the order matters.
  - **SystemRDL via PeakRDL, at the first self-defined register block.** Cheshire is already
    PeakRDL-based: its map lives in a generated `cheshire_addrmap_pkg.sv` built from `cheshire.rdl`.
    So the choice is not whether to adopt SystemRDL but whether to align with the platform's existing
    flow or maintain something parallel to it. The trigger has not arrived: every register block in
    D10's peripheral set belongs to adopted IP, and ours are all still reserved address space — the
    iDMA config, the cluster `axi_llc` config, and the QoS/regulator registers inside the
    `0x4000_0000` alias. Extend `cheshire.rdl`'s conventions rather than starting a second flow.
    PeakRDL is not installed; it is a one-line nix addition when wanted.
  - **IP-XACT at D6, not before.** IEEE 1685 describes structure and connectivity for IP
    integration, and its value appears when assembling real IP with real interfaces — which is
    exactly the Culsans-into-Cheshire join, and that work has not started. Ten regions and four harts
    do not need it; the ceremony would exceed the content. Adoption cost here is unusually low, given existing in-house IEEE 1685-2022 modelling of a comparable topology.

### Remote store programming

- **D39 — Coherent remote stores are reachable as an integration, not a coherence-unit
  modification. D5 stands.** Established from the RTL: the Culsans CCU already performs
  remote-store-invalidate for `WriteUnique`, and the path is exercised rather than dead code.
  - **The mechanism.** `ace_trs_dec.sv` decodes *negatively* — the only writes bypassing the CCU are
    `WriteBack` and `WriteNoSnoop`, so **everything else on AW enters it**, `WriteUnique` and
    `WriteLineUnique` included. `ccu_fsm` then goes `DECODE_W → SEND_INVALID_W` unconditionally,
    broadcasting `CleanInvalid` to every non-initiating master, absorbing any dirty copy via a
    writeback, and only then landing the initiator's write with AW/W forwarded verbatim — so byte
    strobes and sub-cacheline sizes survive, and the ordering is correct.
  - **CVA6 already emits it**: `miss_handler.sv` selects `WRITE_UNIQUE` for stores that are
    uncacheable but inside a shareable region.
  - **The integration cost.** Set `NoSlvPorts = cores + 1` and attach the inbound-write agent to the
    extra port. The CCU's slave-port and snoop-port arrays share one index space and `ccu_fsm` waits
    on `ac_ready != '1` / `cr_valid != '1` across all ports, so the new port needs a **trivial AC/CR
    snoop responder** — accept every AC, answer CR with `dataTransfer=0` — or the FSM deadlocks on
    the first shareable transaction from any core. There is no ACE-Lite port type in either CCU
    generation, and no precedent: every known integration sets slave ports equal to the core count.
    Adding a port also widens IDs downstream, since `ccu_fsm` recovers the initiator index from an ID
    prefix sized `$clog2(NoMstPorts)`.
  - **Invalidate rather than update is sufficient**: the destination takes one miss, refilled from
    its own L2 or scratchpad rather than across the fabric, so the store-latency-for-load-latency
    trade largely survives.
  - **Do not upgrade to the newer CCU expecting better.** The `devel` generation is a pipelined
    rewrite with a real snoop filter and a more complete ACE package, yet `ace_is_write_unique` is
    **never called** in it — a `WriteUnique` would reach memory un-snooped. Its AR path does decode
    `CleanInvalid` and `MakeInvalid`, and it carries a `hasUntrackedArId` field documented as "a port
    without a snoopable cache", so on that generation the viable shape is write plus an explicit
    AR-channel invalidate.
  - **Three latent hazards in the shipped CCU**, dormant only because CVA6 never emits the offending
    transactions, and live the moment a new master appears. **`Evict` would hang it** — not exempted
    by the decoder, so it enters the write path and waits for W beats that never come.
    `WriteLineUnique` gets `CleanInvalid` where the specification says `MakeInvalid`, costing a
    needless writeback. `WriteClean` and `WriteEvict` also take the invalidate-then-write path, wrong
    for `WriteClean`. A new agent must emit `WriteUnique` and nothing else.
  - **Consequence for D9: HPDcache has no coherence at all** — no AC/CR channel, no snoop responder,
    no shared/exclusive state — and evicts clean lines **silently**, so an external snoop filter in
    front of it is permanently conservative. The HPDcache migration would lose this capability as
    described. Its alternative hook is `mem_resp_read_inval_i`, a working external cacheline-
    invalidation input built for OpenPiton's L15 and currently hard-tied to zero on the AXI path with
    the comment *"AXI NoC doesn't support invalidations"*. That is in-band invalidation rather than
    ACE snooping, and its fan-out, back-pressure and refill-FIFO sizing are unsolved.
  - **No prior art.** No open-source RISC-V design implements remote store programming, and none does
    ACE IO-coherent DMA. The nearest precedents use other protocols: Rocket Chip's TileLink probes,
    ESP's MESI directory with ACE used only for L1 invalidation, and OpenPiton's L15 in-band
    invalidations. PULP's shared-L1 clusters sidestep it by making L1 a scratchpad, not a cache.

---

## 3. Component list

| Component | Source | Role | Notes |
| --- | --- | --- | --- |
| CVA6 ×4 | OpenHW Foundation | RV64GC + B, Sv39, M/S/U | Via a Culsans-compatible fork |
| L1I + L1D `std_cache` | CVA6 native | Private per hart | Write-back; ACE snoop lives here |
| Culsans CCU | pulp-platform | MOESI over AMBA ACE, broadcast snoop | Unmodified; 4 cores is the top of its validated envelope |
| Cheshire | pulp-platform | Platform, boot, peripherals | The SoC skeleton |
| `axi_llc` | pulp-platform, via Cheshire | L2 | Supports way-partitioning into scratchpad |
| CLINT | via Cheshire | `mtime`/`mtimecmp` per hart, `msip` IPIs | IPIs carry TLB shootdown and scheduler wakeups |
| PLIC | via Cheshire | External interrupts | 2 contexts per hart (M + S) = 8 contexts |
| `riscv-dbg` + JTAG DTM | via Cheshire | Debug | Instantiated, bring-up deferred |
| UART | via Cheshire | Console | |
| Boot ROM | via Cheshire | Zero-stage | Bypassed in phase 1 |
| iDMA | via Cheshire | DMA | |
| AXI4 interconnect | Cheshire | Fabric | |
| Behavioural AXI memory | Testbench | Main memory, phase 1 | Replaced by a DRAM controller in phase 2 |

Software: OpenSBI (≥ v1.0, generic platform), Linux ≥ 6.10, Buildroot or equivalent initramfs.

---

## 4. Why not AIA

AIA (APLIC + IMSIC) is the better interrupt architecture and its software support is mature. It is
nonetheless out of scope here, for reasons that are about RTL availability rather than preference.
Findings as of 2026-10-19:

- **Mainline CVA6 has no AIA at all.** Not partial — absent. No `miselect`/`mireg`, no
  `mtopei`/`stopei`, no `mtopi`/`stopi`, and no config parameter. The core's entire interrupt
  interface is `irq_i[1:0]`, `ipi_i`, `time_irq_i`, `debug_req_i` — a PLIC/CLINT contract.
- An IMSIC is architecturally **part of the hart**, reached through CSRs, not a bus peripheral.
  Without those CSRs the core physically cannot reach an IMSIC's interrupt files.
- The only CVA6 with AIA CSRs is the Zero-Day Labs research fork: **single-hart**, reaching its
  IMSIC through a bespoke non-standard sideband bundle, last touched January 2025, never proposed
  upstream.
- The AIA RTL itself (`zero-day-labs/riscv-aia`) carries its authors' own warning that it is
  "not formally verified" and "very likely to have bugs", with roughly one commit in the last 20
  months. The only alternative, `jhauser-us/jh_APLIC`, ships no IMSIC.
- **No multicore CVA6 running SMP Linux on AIA appears to exist publicly** — not in a paper, repo
  or thesis that we could find.

The software side is ready: Linux merged the APLIC and IMSIC drivers in **6.10** (July 2024) and
has maintained them since; OpenSBI has had AIA since v1.0 and enables it in the generic defconfig.
The bottleneck is entirely RTL.

Gotcha for anyone attempting it: OpenSBI detects Smaia by *reading* `mtopi`. On a CVA6 without it
the read traps, OpenSBI silently concludes "no AIA", and the failure presents as a confusing
fall-through rather than an error.

---

## 5. Bring-up milestones

Each stage isolates one new failure class. Going straight to four harts means every bug has every
possible cause.

1. **One hart, bare-metal** — `riscv-tests` passing. Validates the Cheshire integration, the address
   map and the toolchain.
2. **One hart, Linux to a shell** — adds MMU, traps, peripherals, boot flow, device tree.
3. **Two harts, SMP** — the first stage where coherence and IPIs matter at all. Litmus gate (D18)
   precedes this.
4. **Four harts, SMP** — scale. The D1 success condition.

**Open risk:** Culsans may not support a single-core configuration. It is a coherency unit built
for 2–4 cores and `NrHarts=1` may be untested or structurally unsupported. If so, stages 1 and 2
run on bare CVA6 without Culsans — still worth doing, since they validate the Cheshire integration
and address map independently, but the Culsans join then first gets exercised at stage 3.

---

## 6. Risks

**The CVA6 version triangle — the hardest practical problem in D6.** Three pins must become one:

| Consumer | Pins |
| --- | --- |
| Cheshire | `pulp-platform/cva6` at `pulp-v2.0.0` (2025-07-17) |
| Culsans | `planvtech/cva6` at `culsans_pulp` (`338d4c1e`, 2024-03-24) |
| This project | A current CVA6 (D13) |

Reconciling these is likely the bulk of the Culsans-into-Cheshire work. Note also that Cheshire's
default build targets a config package with `CVA6ConfigBExtEn = 0`, so **bitmanip is compiled out
of a stock Cheshire** — D7 requires explicitly turning it on.

**Other risks:**

- **HPDcache + Culsans has never been built.** Culsans' snoop extensions are in `std_cache`;
  HPDcache appears nowhere in the Culsans repo. Making that pairing work is a project, not a
  config switch.
- **Culsans upstream is dormant** — last public push 2024-05-04, with an unanswered
  "Some files are missing" issue open since March 2025.
- **Phase 2 has no board selected.** A quadcore CVA6 running Linux needs a large Xilinx or Altera
  part and the corresponding vendor toolchain; open flows (yosys/nextpnr targeting
  ECP5/iCE40/MachXO2/Nexus/Gowin) will not hold it.

---

## 7. Required tooling

| Tool | For |
| --- | --- |
| `qemu-system-riscv64` >= 11.1 | The generic `virt` pre-flight and the custom board (D35) |
| `dtc` | Compiling the generated device tree (D37) |
| Renode >= 1.17 | The declarative platform (D35) |
| Linux-target RISC-V toolchain | Kernel and userspace; Buildroot builds its own |
| Verilator >= 5 | D2 phase 1, and D36's co-simulation. Verilator 4.x is too old for current CVA6 flows |
| Spike | Tandem verification (D19) |
| `openocd` | Phase 2 debug bring-up |
| `bender` | Resolving Cheshire and Culsans dependencies. No distro package and no cargo toolchain by default |
| RISC-V GCC >= 12 | `riscv-tests` and bare-metal stages. GCC 11 and earlier reject the ratified `zba`/`zbb`/`zbs` names, which D7 makes a hard floor |
| PeakRDL | SystemRDL register blocks, when D38's trigger arrives |

Device tree note: the single `riscv,isa` string is deprecated in favour of `riscv,isa-base` plus a
`riscv,isa-extensions` array. Linux will not enable its Zbb-optimised string routines or report the
extensions through `hwprobe` unless the DT advertises them. Emit both forms.

---

## 8. Address map

Physical address width: CVA6's `PLEN` is **56** for RV64 and is not configurable (a pure function of
`XLEN`). Cheshire truncates to `AddrWidth`, default **48**. The map needs 31 bits, so neither binds.

| Range | Size | Class |
| --- | --- | --- |
| `0x0000_0000`–`0x1FFF_FFFF` | 512 MiB | **Cheshire, static.** Boot ROM, CLINT `0x0204_0000`, PLIC `0x0400_0000`, UART `0x0300_2000`, SPM, debug ROM. Generated from `cheshire.rdl` via PeakRDL into `cheshire_addrmap_pkg.sv` |
| `0x2000_0000`–`0x23FF_FFFF` | 64 MiB | **Local alias, cacheable (CIE).** Scratchpad, 8 MiB array. Identical in every cluster |
| `0x2400_0000`–`0x3FFF_FFFF` | 448 MiB | Reserved |
| `0x4000_0000`–`0x41FF_FFFF` | 32 MiB | **Local alias, non-cacheable.** Per-cluster MMIO. Identical in every cluster |
| `0x4200_0000`–`0x4FFF_FFFF` | 222 MiB | Reserved |
| `0x5000_0000`–`0x5FFF_FFFF` | 8 × 32 MiB | **Per-cluster global apertures.** Cluster *N* at `0x5000_0000 + (N << 25)` |
| `0x6000_0000`–`0x7FFF_FFFF` | 512 MiB | Reserved |
| `0x8000_0000`–… | configurable | DRAM, via `LlcOutRegionStart`/`LlcOutRegionEnd` (default 2 GiB) |

### Inside the non-cacheable local alias (`0x4000_0000`, 32 MiB)

| Address | Size | Region | Phase 1 |
| --- | --- | --- | --- |
| `0x4000_0000` | 64 KiB | Per-cluster ACLINT reserve (MTIMER/MSWI/SSWI) | reserved — phase 1 uses Cheshire's platform CLINT |
| `0x4001_0000` | 64 KiB | IMSIC M-mode files, up to 16 × 4 KiB, aligned `0x10000` | reserved — D8 |
| `0x4002_0000` | 64 KiB | IMSIC S-mode files, up to 16 × 4 KiB, aligned `0x10000` | reserved — D8 |
| `0x4004_0000` | 32 KiB | APLIC (16 KiB minimum in MSI mode) | reserved — D8 |
| `0x4010_0000` | 1 MiB | iDMA configuration | ✓ |
| `0x4020_0000` | 1 MiB | `axi_llc` configuration / way partitioning | ✓ |
| `0x4030_0000` | 1 MiB | Quality-of-service / regulator registers | reserved |
| `0x4040_0000` | 28 MiB | Reserved | |

IMSIC geometry, sized for a sixteen-core ceiling rather than today's four: `riscv,hart-index-bits = 4`
at that ceiling and 2 while the cluster holds four cores, no virtualisation, one group per cluster,
`riscv,guest-index-bits = 0`, `riscv,group-index-bits = 3`, `riscv,group-index-shift = 25`. M-mode and
S-mode files are in separate regions — the AIA says "should" rather than "must", with the reason that
a single PMP entry can then grant supervisor access to all supervisor-level files, but the Linux
device-tree binding requires one IMSIC node per privilege level, making it de facto mandatory.

### Inside a per-cluster global aperture (32 MiB)

| Offset | Size | Region |
| --- | --- | --- |
| `+0x000_0000` | 1 MiB | Externally-visible per-cluster MMIO — global alias of the IMSIC/APLIC pages |
| `+0x010_0000` | 24 MiB | Window region, non-cacheable |
| `+0x190_0000` | 7 MiB | Reserved — mailbox, external debug |

**The scratchpad has no global aperture.** There is no address another cluster or external master can
emit that reaches it. That unreachability is a property of the map's shape rather than a permission
check that could be misconfigured.

**Self-window turnaround** falls out of D23: a cluster reaching its own window uses the global address
for its own ID, and the boundary compares `addr[27:25]` against its own cluster ID and services the
access locally instead of emitting it.

### Decode rules required

| | Rules | External AXI subordinate ports |
| --- | --- | --- |
| One cluster | 3 | 1 |
| Eight clusters | 3 | 2 |

Two local-alias rules plus one rule covering `0x5000_0000`–`0x5FFF_FFFF`, with self-turnaround decided
inside the cluster boundary. **Additional clusters cost no additional rules.**

### Open

- Whether phase 1 should decode cluster 0's own aperture and self-turnaround to a local
  non-cacheable region, rather than leaving the range unimplemented.
- `Cva6ExtCieLength` semantics: whether it is measured from the external window base, and what CIE
  actually asserts.
- Window region sizing, and the 8 MiB array, both need checking against a chosen FPGA part's
  on-chip memory; and the 1 MiB way granularity against `axi_llc`'s legal configurations.
