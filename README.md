# tree-explorer-benchmark-Instruction-Guide
This document serves as a complete prompt and technical specification for an AI developer to generate a benchmark monorepo. The goal is to empirically measure, compare, and profile different architectural approaches for rendering and manipulating massive hierarchical trees ($N \ge 100,000$ nodes) in browser environments.

# High-Performance Tree Explorer Benchmark Monorepo Instruction Guide

This document serves as a complete prompt and technical specification for an AI developer to generate a benchmark monorepo. The goal is to empirically measure, compare, and profile different architectural approaches for rendering and manipulating massive hierarchical trees ($N \ge 100,000$ nodes) in browser environments.

---

## 1. Project Objective

Build an isolated benchmark monorepo that evaluates four distinct frontend architectures against standardized performance metrics derived from VS Code's [Lists and Trees](https://github.com/microsoft/vscode/wiki/Lists-And-Trees?utm_source=gemini) specification:

* **Initial Population Speed** (100,000 nodes)
* **Collapse All / Expand All Speed**
* **Filter / Fuzzy Search Latency** (Match All vs. Match Single)
* **Scroll Rendering FPS & Frame Drops** (60–120 fps target)
* **Memory Overhead & Garbage Collection (GC) Pauses**

---

## 2. Target Architectures to Compare

1. **`arch-js-main` (Baseline)**: Pure TypeScript/JS running on the Main Thread with standard DOM-based virtual scrolling.
2. **`arch-js-worker`**: Pure TypeScript/JS running in a Web Worker (Off-Main-Thread) communicating via `postMessage` with a Main Thread Virtual Scroll DOM renderer.
3. **`arch-wasm-main`**: Rust/Wasm compiled core running on the Main Thread + Web Components DOM with virtual recycling.
4. **`arch-wasm-worker` (Target Champion)**: Rust/Wasm core running inside a Web Worker + Web Components DOM with zero-copy array buffer transfer (`Transferable Objects`) and DOM node recycling.

---

## 3. Monorepo Directory Structure

Set up the project using **pnpm workspaces** and a **Cargo workspace**.

```text
tree-benchmark-monorepo/
├── .github/
│   └── workflows/
│       └── benchmark.yml             # GitHub Actions CI automated performance suite
├── Cargo.toml                        # Cargo Workspace root
├── pnpm-workspace.yaml               # pnpm Workspace root
├── package.json
├── turbo.json                        # Turborepo orchestration
├── crates/
│   └── tree-engine-wasm/             # Rust/Wasm Core Engine
│       ├── Cargo.toml
│       ├── src/
│       │   ├── lib.rs                # wasm-bindgen entry point
│       │   ├── tree.rs               # Flat B-Tree / Index Map implementation
│       │   └── search.rs             # Fuzzy search algorithm
│       └── benches/
│           └── tree_bench.rs         # Criterion.rs native Rust benchmarks
├── packages/
│   ├── core-js/                      # Pure TS Tree Engine (Reference Implementation)
│   │   ├── package.json
│   │   └── src/
│   │       └── tree.ts
│   ├── ui-components/                # Web Components (Lit/Vanilla) + Virtual Scroll
│   │   ├── package.json
│   │   └── src/
│   │       ├── virtual-scroll.ts
│   │       └── explorer-tree.ts
│   └── benchmark-suite/              # Standardized test runners & metrics capture
│       ├── package.json
│       └── src/
│           ├── runner.ts
│           └── metrics.ts
└── apps/
    └── benchmark-ui/                 # Interactive Web UI for live browser benchmarking
        ├── package.json
        ├── index.html
        └── src/
            ├── main.ts
            └── charts.ts             # Chart.js / Canvas visualization of results

```

---

## 4. Workspaces & Package Specifications

### 4.1 Cargo Workspace (`Cargo.toml`)

```toml
[workspace]
members = ["crates/tree-engine-wasm"]
resolver = "2"

[profile.release]
opt-level = 3
lto = true
codegen-units = 1
panic = "abort"

```

### 4.2 Rust Engine Specification (`crates/tree-engine-wasm/src/tree.rs`)

* Maintain tree state as a contiguous dynamic array (`Vec<Node>`) using flat index offsets rather than nested object references.
* Define a `Node` payload with minimal memory footprint:
* `id: u32`
* `parent_id: u32`
* `depth: u16`
* `flags: u8` (bitfield for `is_directory`, `is_expanded`, `is_visible`)


* Expose methods:
* `build_tree(node_count: usize) -> Tree`
* `toggle_node(node_id: u32)` (recalculates visible bounds in $O(\log N)$)
* `collapse_all()`
* `expand_all()`
* `filter(query: &str) -> Vec<u32>`
* `get_visible_range(start: usize, limit: usize) -> Uint32Array`



### 4.3 Web Worker Bridge (`packages/ui-components/src/worker-bridge.ts`)

* Implement communication using `Comlink` for ergonomic calls, but support direct `Transferable Objects` (`SharedArrayBuffer` or `ArrayBuffer` transfer) for `get_visible_range` calls to achieve zero-copy slice delivery during high-frequency scrolling.

---

## 5. Standardized Benchmark Operations

All variants must implement the exact same test dataset ($N = 100,000$ hierarchical nodes, 10 levels of max depth).

| Metric | Target / Measurement Strategy |
| --- | --- |
| **Initial Population** | Measure time (ms) to construct 100k nodes in memory and render the initial 30 viewport items. |
| **Collapse All** | Trigger a full collapse of all expanded nodes. Record CPU time (ms) and GC overhead. |
| **Expand All** | Trigger a full expansion. Measure execution time and frame drops. |
| **Filter Match All** | Run search query matching all 100k nodes (highlight update test). |
| **Filter Match Single** | Run search query matching exactly 1 node out of 100k. |
| **Scroll Stress** | Simulate 5,000px continuous scroll via `requestAnimationFrame`. Record average FPS, 1% low FPS, and DOM mutation latency. |

---

## 6. GitHub Actions CI Pipeline (`.github/workflows/benchmark.yml`)

Set up automated, headless benchmark collection on every PR and commit to `main`.

```yaml
name: Performance Benchmark Suite

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  rust-benchmark:
    name: Rust Engine Benchmark (Criterion)
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install Rust Toolchain
        uses: dtolnay/rust-toolchain@stable

      - name: Run Cargo Benchmarks
        run: |
          cd crates/tree-engine-wasm
          cargo bench -- --output-format bencher | tee ../../rust-bench-output.txt

  browser-benchmark:
    name: Headless Browser Benchmark (Vitest / Playwright)
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js & pnpm
        uses: pnpm/action-setup@v3
        with:
          version: 9

      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'pnpm'

      - name: Install Dependencies
        run: pnpm install

      - name: Build Wasm Core
        run: |
          cargo install wasm-pack
          pnpm --filter tree-engine-wasm build

      - name: Run JS vs Wasm Benchmarks
        run: pnpm --filter benchmark-suite test:bench --json > benchmark-results.json

      - name: Store Benchmark Results
        uses: benchmark-action/github-action-benchmark@v1
        with:
          name: Tree Explorer Engine Benchmark
          tool: 'customSmallerIsBetter'
          output-file-path: benchmark-results.json
          github-token: ${{ secrets.GITHUB_TOKEN }}
          auto-push: ${{ github.ref == 'refs/heads/main' }}
          comment-on-alert: true
          alert-threshold: '120%'

```

---

## 7. Instructions for AI Code Generator

When generating this monorepo codebase:

1. **Monorepo Setup**: Generate root configuration files (`pnpm-workspace.yaml`, `Cargo.toml`, `turbo.json`, `package.json`).
2. **Rust Wasm Crate**: Implement `crates/tree-engine-wasm` using `wasm-bindgen`. Ensure algorithms avoid dynamic allocations in hot loops.
3. **JS Baseline Package**: Implement `packages/core-js` with identical tree signatures using standard JS `Array` and `Map` structures to serve as a fair baseline.
4. **UI Renderer Package**: Implement `packages/ui-components` using Vanilla JavaScript or `Lit` Web Components with a fixed pool of recycled DOM nodes (30–40 items max) positioned via `transform: translateY()`.
5. **Benchmark Web Application**: Build `apps/benchmark-ui` with a visual dashboard that displays side-by-side metric charts comparing all four architectures.
