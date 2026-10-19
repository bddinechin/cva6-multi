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
- **D7 — RV64GC + Sv39 + M/S/U + Bitmanip.** No H, no V. Bitmanip via the single `RVB` config bit,
  which covers Zba + Zbb + Zbc + Zbs with no finer granularity — **Zbc cannot be declined**, and its
  carry-less multiply XOR tree in the MUL pipe is the most likely source of FPGA timing pressure
  later. `ZKN` (Zbkb/Zbkx/Zkn) requires a current CVA6.
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

| Tool | Why |
| --- | --- |
| Verilator ≥ 5 | Phase 1 simulation. Verilator 4.x is too old for current CVA6 flows |
| `bender` | Dependency resolution for Cheshire and Culsans |
| `dtc` | Device tree compilation; Linux will not boot without it |
| RISC-V GCC ≥ 12 | GCC 11 and earlier reject the ratified `zba`/`zbb`/`zbs` names, so D7 makes this a hard floor |
| Linux-target RISC-V toolchain | Kernel and userspace |
| `qemu-system-riscv64` | Software pre-flight before RTL simulation |
| Spike | Step-and-compare (D19) |
| `openocd` | Phase 2 debug bring-up |

Device tree note: the single `riscv,isa` string is deprecated in favour of `riscv,isa-base` plus a
`riscv,isa-extensions` array. Linux will not enable its Zbb-optimised string routines or report the
extensions through `hwprobe` unless the DT advertises them. Emit both forms.
