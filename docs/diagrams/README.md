<!-- SPDX-License-Identifier: Apache-2.0 -->
# Diagrams

Text source, rendered output, both committed. **Why this way and not a diagram editor is D41**;
this file is the practical part.

| | |
| --- | --- |
| `D1-cluster-internals.dot` | hand-written Graphviz |
| `D2-remote-store-path.dot` | hand-written Graphviz |
| `D3-address-map.svg` | **generated** by `sw/addrmap/gen.py` — has no `.dot`, do not edit it |

Each source produces a `.svg` and a `.pdf`. Both are committed, and both are checked.

## Adding or changing one

```sh
vi docs/diagrams/D4-something.dot
./build-diagrams.sh            # renders .svg and .pdf for anything older than its source
./build-diagrams.sh --check    # what CI runs: fails if any output is stale
```

Reference it from Markdown with the **`.svg`** path:

```markdown
![D4 — something](diagrams/D4-something.svg)
```

`render.sh` swaps the link to the `.pdf` when building the PDF, because pandoc cannot place an
SVG without `rsvg-convert` and fails the entire document rather than dropping the figure. The
HTML preview and GitHub keep the SVG. You do not need to think about this — write `.svg`.

## Two rules the build enforces

**Every node must be `shape=box`.** Graphviz defaults to `<ellipse>`, which `svg2pptx.py` cannot
convert into a PowerPoint shape. `build-diagrams.sh` validates its own output against that
converter's element set and fails immediately, naming the fix, rather than letting it surface in
a slide build days later.

**`D3-address-map.svg` is generated, so edit `sw/addrmap/cva6-multi-map.json` instead.** A
hand-drawn address map would be a sixth copy of numbers that already have one source, and the one
nobody re-checks. `sw/addrmap/gen.py --check` covers it alongside the device tree, the Renode
platform, the QEMU memmap and the VCML map.

## Keep them true

A diagram that disagrees with the decision record is worse than no diagram, because it looks
authoritative. Both hand-written sources carry a comment naming the decisions they depict, so a
change to D6, D26, D39 or D40 should send you here. Check the drawn behaviour against the decision
before trusting it: interrupts terminate at the harts rather than at the coherence unit, and a
coherent remote store completes whether or not another cache held the line dirty.
