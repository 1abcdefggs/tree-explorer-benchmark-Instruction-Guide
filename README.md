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






## 1. Bash Script (`setup.sh`) — macOS / Linux / WSL / Git Bash

Run `bash setup.sh` in your terminal to initialize the entire directory layout and boilerplate configuration files in one command.

```bash
#!/usr/bin/env bash
set -e

echo "🚀 Creating monorepo directory structure..."

# Create directory tree
mkdir -p .github/workflows \
  crates/tree-engine-wasm/src \
  crates/tree-engine-wasm/benches \
  packages/core-js/src \
  packages/ui-components/src \
  packages/benchmark-suite/src \
  apps/benchmark-ui/src

echo "📝 Generating Root Configuration Files..."

# Root package.json
cat << 'EOF' > package.json
{
  "name": "tree-benchmark-monorepo",
  "private": true,
  "scripts": {
    "build": "turbo run build",
    "test:bench": "turbo run test:bench"
  },
  "devDependencies": {
    "turbo": "^1.13.0"
  }
}
EOF

# pnpm-workspace.yaml
cat << 'EOF' > pnpm-workspace.yaml
packages:
  - "packages/*"
  - "apps/*"
EOF

# Root Cargo.toml
cat << 'EOF' > Cargo.toml
[workspace]
members = ["crates/tree-engine-wasm"]
resolver = "2"

[profile.release]
opt-level = 3
lto = true
codegen-units = 1
panic = "abort"
EOF

# turbo.json
cat << 'EOF' > turbo.json
{
  "$schema": "https://turbo.build/schema.json",
  "pipeline": {
    "build": {
      "outputs": ["dist/**", "pkg/**"]
    },
    "test:bench": {}
  }
}
EOF

echo "🦀 Generating Rust Crate Files..."

# Rust Cargo.toml
cat << 'EOF' > crates/tree-engine-wasm/Cargo.toml
[package]
name = "tree-engine-wasm"
version = "0.1.0"
edition = "2021"

[lib]
crate-type = ["cdylib", "rlib"]

[dependencies]
wasm-bindgen = "0.2"

[dev-dependencies]
criterion = "0.5"

[[bench]]
name = "tree_bench"
harness = false
EOF

# Rust lib.rs placeholder
cat << 'EOF' > crates/tree-engine-wasm/src/lib.rs
use wasm_bindgen::prelude::*;

#[wasm_bindgen]
pub struct TreeEngine {
    node_count: usize,
}

#[wasm_bindgen]
impl TreeEngine {
    #[wasm_bindgen(constructor)]
    pub fn new(node_count: usize) -> Self {
        Self { node_count }
    }

    pub fn get_node_count(&self) -> usize {
        self.node_count
    }
}
EOF

echo "📦 Generating Package Files..."

# packages/core-js/package.json
cat << 'EOF' > packages/core-js/package.json
{
  "name": "@explorer/core-js",
  "version": "0.1.0",
  "main": "src/index.ts",
  "scripts": {
    "build": "tsc --noEmit"
  }
}
EOF

# packages/ui-components/package.json
cat << 'EOF' > packages/ui-components/package.json
{
  "name": "@explorer/ui-components",
  "version": "0.1.0",
  "main": "src/index.ts",
  "dependencies": {
    "lit": "^3.1.0"
  }
}
EOF

# packages/benchmark-suite/package.json
cat << 'EOF' > packages/benchmark-suite/package.json
{
  "name": "@explorer/benchmark-suite",
  "version": "0.1.0",
  "scripts": {
    "test:bench": "vitest bench"
  }
}
EOF

echo "🤖 Generating GitHub Actions Workflow..."

cat << 'EOF' > .github/workflows/benchmark.yml
name: Performance Benchmark Suite

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  rust-benchmark:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
      - run: cd crates/tree-engine-wasm && cargo bench

  browser-benchmark:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v3
        with:
          version: 9
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'pnpm'
      - run: pnpm install
EOF

echo "✅ Benchmark Monorepo layout generated successfully!"

```

---

### 2. PowerShell Script (`setup.ps1`) — Windows PowerShell

Run `.\setup.ps1` in PowerShell to generate the exact same structure natively on Windows.

```powershell
$ErrorActionPreference = "Stop"

Write-Host "🚀 Creating monorepo directory structure..." -ForegroundColor Green

$dirs = @(
  ".github\workflows",
  "crates\tree-engine-wasm\src",
  "crates\tree-engine-wasm\benches",
  "packages\core-js\src",
  "packages\ui-components\src",
  "packages\benchmark-suite\src",
  "apps\benchmark-ui\src"
)

foreach ($d in $dirs) {
  New-Item -ItemType Directory -Force -Path $d | Out-Null
}

Write-Host "📝 Generating Root Configurations..." -ForegroundColor Cyan

@'
{
  "name": "tree-benchmark-monorepo",
  "private": true,
  "scripts": {
    "build": "turbo run build",
    "test:bench": "turbo run test:bench"
  },
  "devDependencies": {
    "turbo": "^1.13.0"
  }
}
'@ | Set-Content -Path "package.json"

@'
packages:
  - "packages/*"
  - "apps/*"
'@ | Set-Content -Path "pnpm-workspace.yaml"

@'
[workspace]
members = ["crates/tree-engine-wasm"]
resolver = "2"

[profile.release]
opt-level = 3
lto = true
codegen-units = 1
panic = "abort"
'@ | Set-Content -Path "Cargo.toml"

@'
{
  "$schema": "https://turbo.build/schema.json",
  "pipeline": {
    "build": {
      "outputs": ["dist/**", "pkg/**"]
    },
    "test:bench": {}
  }
}
'@ | Set-Content -Path "turbo.json"

Write-Host "🦀 Generating Rust Crate Files..." -ForegroundColor Cyan

@'
[package]
name = "tree-engine-wasm"
version = "0.1.0"
edition = "2021"

[lib]
crate-type = ["cdylib", "rlib"]

[dependencies]
wasm-bindgen = "0.2"

[dev-dependencies]
criterion = "0.5"

[[bench]]
name = "tree_bench"
harness = false
'@ | Set-Content -Path "crates\tree-engine-wasm\Cargo.toml"

@'
use wasm_bindgen::prelude::*;

#[wasm_bindgen]
pub struct TreeEngine {
    node_count: usize,
}

#[wasm_bindgen]
impl TreeEngine {
    #[wasm_bindgen(constructor)]
    pub fn new(node_count: usize) -> Self {
        Self { node_count }
    }

    pub fn get_node_count(&self) -> usize {
        self.node_count
    }
}
'@ | Set-Content -Path "crates\tree-engine-wasm\src\lib.rs"

Write-Host "✅ Setup completed successfully!" -ForegroundColor Green

```







---

## 1. Bash Script (`setup-src.sh`) — macOS / Linux / WSL / Git Bash

```bash
#!/usr/bin/env bash
set -e

echo "🦀 Populating Rust Crate Source Code..."

# Update Rust lib.rs & tree.rs
cat << 'EOF' > crates/tree-engine-wasm/src/lib.rs
pub mod tree;
pub use tree::FlatTree;
EOF

cat << 'EOF' > crates/tree-engine-wasm/src/tree.rs
use wasm_bindgen::prelude::*;

#[derive(Clone, Copy)]
pub struct Node {
    pub id: u32,
    pub parent_id: u32,
    pub depth: u16,
    pub is_directory: bool,
    pub is_expanded: bool,
    pub is_visible: bool,
}

#[wasm_bindgen]
pub struct FlatTree {
    nodes: Vec<Node>,
    visible_indices: Vec<u32>,
}

#[wasm_bindgen]
impl FlatTree {
    #[wasm_bindgen(constructor)]
    pub fn new(total_nodes: u32) -> Self {
        let mut nodes = Vec::with_capacity(total_nodes as usize);
        for i in 0..total_nodes {
            let is_directory = i % 10 == 0;
            let depth = if i == 0 { 0 } else { ((i % 5) + 1) as u16 };
            nodes.push(Node {
                id: i,
                parent_id: if depth == 0 { 0 } else { i - 1 },
                depth,
                is_directory,
                is_expanded: true,
                is_visible: true,
            });
        }
        let visible_indices: Vec<u32> = (0..total_nodes).collect();
        Self { nodes, visible_indices }
    }

    pub fn visible_count(&self) -> usize {
        self.visible_indices.len()
    }

    pub fn collapse_all(&mut self) {
        for node in self.nodes.iter_mut() {
            if node.is_directory {
                node.is_expanded = false;
            }
            if node.depth > 0 {
                node.is_visible = false;
            }
        }
        self.rebuild_visible();
    }

    pub fn expand_all(&mut self) {
        for node in self.nodes.iter_mut() {
            node.is_expanded = true;
            node.is_visible = true;
        }
        self.rebuild_visible();
    }

    fn rebuild_visible(&mut self) {
        self.visible_indices = self.nodes.iter()
            .enumerate()
            .filter(|(_, n)| n.is_visible)
            .map(|(i, _)| i as u32)
            .collect();
    }
}
EOF

# Rust Benchmark File
cat << 'EOF' > crates/tree-engine-wasm/benches/tree_bench.rs
use criterion::{criterion_group, criterion_main, Criterion};
use tree_engine_wasm::FlatTree;

fn bench_tree_ops(c: &mut Criterion) {
    let mut group = c.benchmark_group("Tree Operations 100k");

    group.bench_function("init_100k", |b| {
        b.iter(|| FlatTree::new(100_000))
    });

    group.bench_function("collapse_all_100k", |b| {
        let mut tree = FlatTree::new(100_000);
        b.iter(|| tree.collapse_all())
    });

    group.finish();
}

criterion_group!(benches, bench_tree_ops);
criterion_main!(benches);
EOF

echo "📦 Populating JS Baseline & UI Source Code..."

# JS Tree Baseline
cat << 'EOF' > packages/core-js/src/index.ts
export interface JSNode {
  id: number;
  parentId: number;
  depth: number;
  isDirectory: boolean;
  isExpanded: boolean;
  isVisible: boolean;
}

export class JSTree {
  nodes: JSNode[];
  visibleIndices: number[];

  constructor(totalNodes: number) {
    this.nodes = new Array(totalNodes);
    this.visibleIndices = [];

    for (let i = 0; i < totalNodes; i++) {
      const isDirectory = i % 10 === 0;
      const depth = i === 0 ? 0 : (i % 5) + 1;
      this.nodes[i] = {
        id: i,
        parentId: depth === 0 ? 0 : i - 1,
        depth,
        isDirectory,
        isExpanded: true,
        isVisible: true,
      };
      this.visibleIndices.push(i);
    }
  }

  collapseAll(): void {
    for (let i = 0; i < this.nodes.length; i++) {
      if (this.nodes[i].isDirectory) {
        this.nodes[i].isExpanded = false;
      }
      if (this.nodes[i].depth > 0) {
        this.nodes[i].isVisible = false;
      }
    }
    this.rebuildVisible();
  }

  expandAll(): void {
    for (let i = 0; i < this.nodes.length; i++) {
      this.nodes[i].isExpanded = true;
      this.nodes[i].isVisible = true;
    }
    this.rebuildVisible();
  }

  private rebuildVisible(): void {
    this.visibleIndices = [];
    for (let i = 0; i < this.nodes.length; i++) {
      if (this.nodes[i].isVisible) {
        this.visibleIndices.push(i);
      }
    }
  }
}
EOF

# UI Virtual Scroller Component
cat << 'EOF' > packages/ui-components/src/index.ts
export class VirtualScroller {
  private viewportHeight: number;
  private itemHeight: number;
  private visibleCount: number;

  constructor(viewportHeight: number = 600, itemHeight: number = 28) {
    this.viewportHeight = viewportHeight;
    this.itemHeight = itemHeight;
    this.visibleCount = Math.ceil(viewportHeight / itemHeight) + 5;
  }

  getRange(scrollTop: number, totalCount: number) {
    const startIndex = Math.max(0, Math.floor(scrollTop / this.itemHeight) - 2);
    const endIndex = Math.min(totalCount, startIndex + this.visibleCount);
    return {
      startIndex,
      endIndex,
      totalHeight: totalCount * this.itemHeight,
      offsetY: startIndex * this.itemHeight,
    };
  }
}
EOF

# Benchmark Runner
cat << 'EOF' > packages/benchmark-suite/src/index.ts
import { JSTree } from '@explorer/core-js';

export function runJSBenchmark(nodeCount: number = 100000) {
  const startInit = performance.now();
  const tree = new JSTree(nodeCount);
  const endInit = performance.now();

  const startCollapse = performance.now();
  tree.collapseAll();
  const endCollapse = performance.now();

  const startExpand = performance.now();
  tree.expandAll();
  const endExpand = performance.now();

  return {
    initializationMs: endInit - startInit,
    collapseAllMs: endCollapse - startCollapse,
    expandAllMs: endExpand - startExpand,
  };
}
EOF

echo "✨ Source files successfully populated!"

```

---

### 2. PowerShell Script (`setup-src.ps1`) — Windows PowerShell

```powershell
$ErrorActionPreference = "Stop"

Write-Host "🦀 Populating Rust Source Code..." -ForegroundColor Green

@'
pub mod tree;
pub use tree::FlatTree;
'@ | Set-Content -Path "crates\tree-engine-wasm\src\lib.rs"

@'
use wasm_bindgen::prelude::*;

#[derive(Clone, Copy)]
pub struct Node {
    pub id: u32,
    pub parent_id: u32,
    pub depth: u16,
    pub is_directory: bool,
    pub is_expanded: bool,
    pub is_visible: bool,
}

#[wasm_bindgen]
pub struct FlatTree {
    nodes: Vec<Node>,
    visible_indices: Vec<u32>,
}

#[wasm_bindgen]
impl FlatTree {
    #[wasm_bindgen(constructor)]
    pub fn new(total_nodes: u32) -> Self {
        let mut nodes = Vec::with_capacity(total_nodes as usize);
        for i in 0..total_nodes {
            let is_directory = i % 10 == 0;
            let depth = if i == 0 { 0 } else { ((i % 5) + 1) as u16 };
            nodes.push(Node {
                id: i,
                parent_id: if depth == 0 { 0 } else { i - 1 },
                depth,
                is_directory,
                is_expanded: true,
                is_visible: true,
            });
        }
        let visible_indices: Vec<u32> = (0..total_nodes).collect();
        Self { nodes, visible_indices }
    }

    pub fn visible_count(&self) -> usize {
        self.visible_indices.len()
    }

    pub fn collapse_all(&mut self) {
        for node in self.nodes.iter_mut() {
            if node.is_directory {
                node.is_expanded = false;
            }
            if node.depth > 0 {
                node.is_visible = false;
            }
        }
        self.rebuild_visible();
    }

    pub fn expand_all(&mut self) {
        for node in self.nodes.iter_mut() {
            node.is_expanded = true;
            node.is_visible = true;
        }
        self.rebuild_visible();
    }

    fn rebuild_visible(&mut self) {
        self.visible_indices = self.nodes.iter()
            .enumerate()
            .filter(|(_, n)| n.is_visible)
            .map(|(i, _)| i as u32)
            .collect();
    }
}
'@ | Set-Content -Path "crates\tree-engine-wasm\src\tree.rs"

@'
use criterion::{criterion_group, criterion_main, Criterion};
use tree_engine_wasm::FlatTree;

fn bench_tree_ops(c: &mut Criterion) {
    let mut group = c.benchmark_group("Tree Operations 100k");

    group.bench_function("init_100k", |b| {
        b.iter(|| FlatTree::new(100_000))
    });

    group.bench_function("collapse_all_100k", |b| {
        let mut tree = FlatTree::new(100_000);
        b.iter(|| tree.collapse_all())
    });

    group.finish();
}

criterion_group!(benches, bench_tree_ops);
criterion_main!(benches);
'@ | Set-Content -Path "crates\tree-engine-wasm\benches\tree_bench.rs"

Write-Host "📦 Populating TypeScript Core & Benchmark Suite..." -ForegroundColor Cyan

@'
export interface JSNode {
  id: number;
  parentId: number;
  depth: number;
  isDirectory: boolean;
  isExpanded: boolean;
  isVisible: boolean;
}

export class JSTree {
  nodes: JSNode[];
  visibleIndices: number[];

  constructor(totalNodes: number) {
    this.nodes = new Array(totalNodes);
    this.visibleIndices = [];

    for (let i = 0; i < totalNodes; i++) {
      const isDirectory = i % 10 === 0;
      const depth = i === 0 ? 0 : (i % 5) + 1;
      this.nodes[i] = {
        id: i,
        parentId: depth === 0 ? 0 : i - 1,
        depth,
        isDirectory,
        isExpanded: true,
        isVisible: true,
      };
      this.visibleIndices.push(i);
    }
  }

  collapseAll(): void {
    for (let i = 0; i < this.nodes.length; i++) {
      if (this.nodes[i].isDirectory) {
        this.nodes[i].isExpanded = false;
      }
      if (this.nodes[i].depth > 0) {
        this.nodes[i].isVisible = false;
      }
    }
    this.rebuildVisible();
  }

  expandAll(): void {
    for (let i = 0; i < this.nodes.length; i++) {
      this.nodes[i].isExpanded = true;
      this.nodes[i].isVisible = true;
    }
    this.rebuildVisible();
  }

  private rebuildVisible(): void {
    this.visibleIndices = [];
    for (let i = 0; i < this.nodes.length; i++) {
      if (this.nodes[i].isVisible) {
        this.visibleIndices.push(i);
      }
    }
  }
}
'@ | Set-Content -Path "packages\core-js\src\index.ts"

@'
export class VirtualScroller {
  private viewportHeight: number;
  private itemHeight: number;
  private visibleCount: number;

  constructor(viewportHeight: number = 600, itemHeight: number = 28) {
    this.viewportHeight = viewportHeight;
    this.itemHeight = itemHeight;
    this.visibleCount = Math.ceil(viewportHeight / itemHeight) + 5;
  }

  getRange(scrollTop: number, totalCount: number) {
    const startIndex = Math.max(0, Math.floor(scrollTop / this.itemHeight) - 2);
    const endIndex = Math.min(totalCount, startIndex + this.visibleCount);
    return {
      startIndex,
      endIndex,
      totalHeight: totalCount * this.itemHeight,
      offsetY: startIndex * this.itemHeight,
    };
  }
}
'@ | Set-Content -Path "packages\ui-components\src\index.ts"

@'
import { JSTree } from '@explorer/core-js';

export function runJSBenchmark(nodeCount: number = 100000) {
  const startInit = performance.now();
  const tree = new JSTree(nodeCount);
  const endInit = performance.now();

  const startCollapse = performance.now();
  tree.collapseAll();
  const endCollapse = performance.now();

  const startExpand = performance.now();
  tree.expandAll();
  const endExpand = performance.now();

  return {
    initializationMs: endInit - startInit,
    collapseAllMs: endCollapse - startCollapse,
    expandAllMs: endExpand - startExpand,
  };
}
'@ | Set-Content -Path "packages\benchmark-suite\src\index.ts"

Write-Host "✨ Source files successfully generated!" -ForegroundColor Green

```

---


---

### 1. Bash Script (`setup-app.sh`) — macOS / Linux / WSL / Git Bash

```bash
#!/usr/bin/env bash
set -e

echo "🎨 Creating Benchmark Web UI & Vite Configurations..."

# Vite Config
cat << 'EOF' > apps/benchmark-ui/vite.config.ts
import { defineConfig } from 'vite';

export default defineConfig({
  server: {
    port: 3000,
  },
  optimizeDeps: {
    exclude: ['@explorer/tree-engine-wasm'],
  },
});
EOF

# HTML Entry Point
cat << 'EOF' > apps/benchmark-ui/index.html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Tree Explorer Benchmark Suite</title>
  <style>
    body { font-family: system-ui, sans-serif; background: #0f172a; color: #f8fafc; margin: 0; padding: 20px; }
    .container { max-width: 900px; margin: 0 auto; }
    .card { background: #1e293b; padding: 20px; border-radius: 8px; margin-bottom: 20px; }
    button { background: #3b82f6; color: white; border: none; padding: 10px 20px; border-radius: 6px; cursor: pointer; font-weight: bold; }
    button:hover { background: #2563eb; }
    table { width: 100%; border-collapse: collapse; margin-top: 15px; }
    th, td { text-align: left; padding: 10px; border-bottom: 1px solid #334155; }
    th { color: #94a3b8; }
  </style>
</head>
<body>
  <div class="container">
    <h1>🌲 100k Node Tree Benchmark Dashboard</h1>
    <div class="card">
      <button id="run-bench">Run 100,000 Node Benchmark</button>
      <div id="status">Ready</div>
    </div>
    <div class="card">
      <h2>Results Comparison</h2>
      <table>
        <thead>
          <tr>
            <th>Engine / Architecture</th>
            <th>Init (100k)</th>
            <th>Collapse All</th>
            <th>Expand All</th>
          </tr>
        </thead>
        <tbody id="results-body">
          <tr><td colspan="4" style="color: #64748b;">No benchmark runs yet.</td></tr>
        </tbody>
      </table>
    </div>
  </div>
  <script type="module" src="/src/main.ts"></script>
</body>
</html>
EOF

# UI Main Logic
cat << 'EOF' > apps/benchmark-ui/src/main.ts
import { runJSBenchmark } from '@explorer/benchmark-suite';

const runBtn = document.getElementById('run-bench') as HTMLButtonElement;
const statusEl = document.getElementById('status') as HTMLDivElement;
const resultsBody = document.getElementById('results-body') as HTMLTableSectionElement;

runBtn.addEventListener('click', async () => {
  statusEl.innerText = 'Running JavaScript Baseline Benchmark...';
  runBtn.disabled = true;

  // Allow UI to render status
  await new Promise((r) => setTimeout(r, 50));

  const jsResults = runJSBenchmark(100000);

  resultsBody.innerHTML = `
    <tr>
      <td><strong>JS Main Thread (Baseline)</strong></td>
      <td>${jsResults.initializationMs.toFixed(2)} ms</td>
      <td>${jsResults.collapseAllMs.toFixed(2)} ms</td>
      <td>${jsResults.expandAllMs.toFixed(2)} ms</td>
    </tr>
  `;

  statusEl.innerText = 'Benchmark completed!';
  runBtn.disabled = false;
});
EOF

echo "✨ Benchmark Application Files successfully generated!"

```

---

### 2. PowerShell Script (`setup-app.ps1`) — Windows PowerShell

```powershell
$ErrorActionPreference = "Stop"

Write-Host "🎨 Creating Benchmark Web UI & Vite Configurations..." -ForegroundColor Green

@'
import { defineConfig } from 'vite';

export default defineConfig({
  server: {
    port: 3000,
  },
  optimizeDeps: {
    exclude: ['@explorer/tree-engine-wasm'],
  },
});
'@ | Set-Content -Path "apps\benchmark-ui\vite.config.ts"

@'
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Tree Explorer Benchmark Suite</title>
  <style>
    body { font-family: system-ui, sans-serif; background: #0f172a; color: #f8fafc; margin: 0; padding: 20px; }
    .container { max-width: 900px; margin: 0 auto; }
    .card { background: #1e293b; padding: 20px; border-radius: 8px; margin-bottom: 20px; }
    button { background: #3b82f6; color: white; border: none; padding: 10px 20px; border-radius: 6px; cursor: pointer; font-weight: bold; }
    button:hover { background: #2563eb; }
    table { width: 100%; border-collapse: collapse; margin-top: 15px; }
    th, td { text-align: left; padding: 10px; border-bottom: 1px solid #334155; }
    th { color: #94a3b8; }
  </style>
</head>
<body>
  <div class="container">
    <h1>🌲 100k Node Tree Benchmark Dashboard</h1>
    <div class="card">
      <button id="run-bench">Run 100,000 Node Benchmark</button>
      <div id="status">Ready</div>
    </div>
    <div class="card">
      <h2>Results Comparison</h2>
      <table>
        <thead>
          <tr>
            <th>Engine / Architecture</th>
            <th>Init (100k)</th>
            <th>Collapse All</th>
            <th>Expand All</th>
          </tr>
        </thead>
        <tbody id="results-body">
          <tr><td colspan="4" style="color: #64748b;">No benchmark runs yet.</td></tr>
        </tbody>
      </table>
    </div>
  </div>
  <script type="module" src="/src/main.ts"></script>
</body>
</html>
'@ | Set-Content -Path "apps\benchmark-ui\index.html"

@'
import { runJSBenchmark } from '@explorer/benchmark-suite';

const runBtn = document.getElementById('run-bench') as HTMLButtonElement;
const statusEl = document.getElementById('status') as HTMLDivElement;
const resultsBody = document.getElementById('results-body') as HTMLTableSectionElement;

runBtn.addEventListener('click', async () => {
  statusEl.innerText = 'Running JavaScript Baseline Benchmark...';
  runBtn.disabled = true;

  await new Promise((r) => setTimeout(r, 50));

  const jsResults = runJSBenchmark(100000);

  resultsBody.innerHTML = `
    <tr>
      <td><strong>JS Main Thread (Baseline)</strong></td>
      <td>${jsResults.initializationMs.toFixed(2)} ms</td>
      <td>${jsResults.collapseAllMs.toFixed(2)} ms</td>
      <td>${jsResults.expandAllMs.toFixed(2)} ms</td>
    </tr>
  `;

  statusEl.innerText = 'Benchmark completed!';
  runBtn.disabled = false;
});
'@ | Set-Content -Path "apps\benchmark-ui\src\main.ts"

Write-Host "✨ Application generated successfully!" -ForegroundColor Green

```

---


1. **`setup.sh` / `setup.ps1**`: monorepo
2. **`setup-src.sh` / `setup-src.ps1**`: Rust/Wasm Engine, JS Baseline, UI Engine gen
3. **`setup-app.sh` / `setup-app.ps1**`: Web view

## command

```bash
pnpm install
pnpm --filter benchmark-ui dev

```

`http://localhost:3000` Run 100,000 Node Benchmark
