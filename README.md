# cva6-multi

A stand-alone, cache-coherent **quadcore RV64 cluster** assembled from open hardware.

Four [CVA6](https://github.com/openhwfoundation/cva6) harts made coherent by
[Culsans](https://github.com/pulp-platform/culsans) (MOESI over AMBA ACE), integrated into the
[Cheshire](https://github.com/pulp-platform/cheshire) platform, targeting **SMP Linux booting to a
shell on all four harts**.

> **This repository is the design record, not the implementation.** It carries the agreed
> decisions, the physical address map, and the research behind them, in
> [`docs/cluster-plan.md`](docs/cluster-plan.md). There is no RTL here and there will not be: the
> implementation depends on IP hosted where an external reader cannot reach it, so a buildable
> public repository was never possible. Rather than ship something that looks buildable and is not,
> this repository carries the part that stands on its own.

## What this is

An integration project, not a new core or a new coherence protocol. The CPU, the coherence unit,
the caches and the peripherals are all existing open hardware used as-is. The work is the part that
does not exist yet: the SoC top, the address map, the AXI topology, interrupt routing, multi-hart
debug plumbing, the boot flow, the device tree, and the testbench — plus the join between Culsans
and Cheshire, which as far as we can tell nobody has made.

What is published here is the reasoning: why those choices, what the alternatives cost, and what
the current state of the ecosystem actually is. The address map and the survey of RISC-V AIA
support in the CVA6 ecosystem are both usable independently of this project.

## Components

| | |
| --- | --- |
| **Cores** | CVA6 ×4 — RV64GC + Zba/Zbb/Zbc/Zbs, Sv39, M/S/U |
| **Coherence** | Culsans CCU — MOESI over AMBA ACE, broadcast snoop |
| **L1D** | CVA6 native write-back `std_cache` (where the ACE snoop extensions live) |
| **L2** | `axi_llc` |
| **Interrupts** | CLINT (timer, IPIs) + PLIC (external, 8 contexts) |
| **Platform** | Cheshire — boot ROM, UART, debug, iDMA, AXI fabric |
| **Software** | OpenSBI ≥ v1.0, Linux ≥ 6.10 |

Phase 1 targets Verilator with a behavioural AXI memory. FPGA is a later phase.

## Bring-up milestones

1. One hart, bare-metal, `riscv-tests` passing
2. One hart, Linux to a shell
3. Two harts, SMP — litmus-tested coherence gate precedes this
4. Four harts, SMP

## Documentation

- [`docs/cluster-plan.md`](docs/cluster-plan.md) — the decision register, with rationale, the
  component list, bring-up milestones, known risks, and a section on **why not AIA** that collects
  the current state of RISC-V Advanced Interrupt Architecture support in the CVA6 ecosystem.

## Licence

[Solderpad Hardware License v2.1](https://solderpad.org/licenses/SHL-2.1/)
(`Apache-2.0 WITH SHL-2.1`) — see [LICENSE](LICENSE). Chosen to match the licensing of the
pulp-platform IP this project integrates.
