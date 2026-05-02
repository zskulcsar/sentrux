# Sentrux Quality Signals — Implementation Overview

This document describes how the five root cause quality signals are calculated in Sentrux.
Each signal measures a fundamental structural property of the codebase-as-graph.

---

## Terminology

| Term | Definition |
|------|------------|
| **DAG** | Directed Acyclic Graph — a graph with directed edges and no cycles |
| **SCC** | Strongly Connected Component — a subgraph where every node is reachable from every other node |
| **DFS** | Depth-First Search — a graph traversal algorithm that explores as far as possible along each branch before backtracking |
| **CC** | Cyclomatic Complexity — a software metric for code complexity based on decision paths |
| **ADP** | Acyclic Dependencies Principle — the principle that dependency graphs should have no cycles |

---

## Signal Definitions

| Signal | What it Measures | Theory | Raw Range | Normalized Score |
|--------|------------------|--------|-----------|------------------|
| **Modularity** | How well the dependency graph decomposes into independent clusters | Newman 2004 (Modularity Q) | [-0.5, 1.0] | `(Q + 0.5) / 1.5` → [0, 1] |
| **Acyclicity** | Absence of circular dependencies | Martin 2003 (ADP), Tarjan 1972 (SCC) | [0, ∞) cycle count | `1 / (1 + cycle_count)` → (0, 1] |
| **Depth** | Longest dependency chain in the DAG | Lakos 1996 (Levelization) | [0, ∞) levels | `1 / (1 + depth / 8)` → (0, 1] |
| **Equality** | How evenly complexity is distributed across functions | Gini 1912 | [0, 1] | `1 - G` → [0, 1] |
| **Redundancy** | Fraction of unnecessary code (dead + duplicate functions) | Kolmogorov complexity approximation | [0, 1] | `1 - R` → [0, 1] |

**Quality Signal** = Geometric mean of all 5 normalized scores:
```
quality_signal = (modularity × acyclicity × depth × equality × redundancy)^(1/5)
```

The geometric mean ensures that gaming one metric while tanking another cannot increase
the overall signal. All five dimensions must improve for the signal to rise significantly.

---

## Detailed Signal Breakdown

### 1. Modularity (Newman's Q)

**Purpose**: Measures how well the dependency graph clusters into independent modules.
Compares actual intra-module edge density against a random graph with the same degree sequence.

**Formula**:
```
Q = (1/m) * Σ_ij [A_ij - (k_out_i * k_in_j / m)] * δ(c_i, c_j)

Where:
- A_ij = 1 if edge from i to j, else 0
- k_out_i = out-degree of node i
- k_in_j = in-degree of node j  
- m = total edges
- δ(c_i, c_j) = 1 if i and j are in the same module, else 0
```

**Interpretation**:
- Q > 0.3: Significant modular structure
- Q > 0.6: Strong modular structure
- Q ≤ 0: Worse than random (anti-modular)

**Why ungameable**: Adding useless edges moves the graph closer to random, which *decreases* Q.
Only genuine modular restructuring improves Q.

**Language fairness**: Uses both import edges AND call edges. Projects with no imports
(e.g., Swift) still get meaningful Q from call graph alone.

---

### 2. Acyclicity

**Purpose**: Counts circular dependency cycles. Cycles make build order undefined,
change propagation unpredictable, and testing difficult.

**Formula**:
```
cycle_count = |{ scc ∈ SCC(G) | |scc| > 1 }|

Where:
- G = directed dependency graph
- SCC(G) = set of all strongly connected components in G
- |scc| = number of nodes in component scc
```

**Algorithm**: Tarjan's Strongly Connected Components (SCC) algorithm.
- Finds all SCCs in the dependency graph
- Only counts SCCs with >1 member (actual cycles, not single nodes)
- Uses iterative implementation to avoid stack overflow

**Normalization**: Sigmoid because cycle count is unbounded.
- 0 cycles → score = 1.0 (perfect)
- 1 cycle → score = 0.5
- 2 cycles → score = 0.33
- ...

**Why fundamental**: A cycle means A depends on B depends on A — neither can be understood
or tested independently. This is a structural impossibility, not a style preference.

---

### 3. Depth

**Purpose**: Measures the longest dependency chain. Deep chains mean a change at the
bottom propagates through many layers, making the system fragile.

**Formula**:
```
depth = max{ L(p) | p ∈ P(s, t) ∀ s ∈ S, t ∈ V }

Where:
- S = set of seed nodes (entry points or fan-in = 0)
- V = set of all nodes in the DAG
- P(s, t) = set of all paths from seed s to node t
- L(p) = length of path p (number of edges)
```

**Algorithm**: Iterative longest-path DFS from seed nodes.

**Seed nodes** (in priority order):
1. Explicit entry points (from configuration/plugins)
2. Files with no incoming dependencies (fan-in = 0)

**Normalization**: Sigmoid with midpoint at 8.
- depth = 0 → score = 1.0
- depth = 8 → score = 0.5
- depth = 16 → score = 0.33

**Why independent from Q**: A graph can have perfect modularity (high Q) but still
have a chain of 20 modules depending sequentially. Depth measures this orthogonal property.

---

### 4. Equality (Gini Coefficient)

**Purpose**: Measures how evenly complexity (cyclomatic complexity) is distributed
across functions. God functions are the #1 source of AI agent confusion.

**Formula**:
```
G = Σ (2i - n - 1) * x_i / (n * Σ x_i)

Where:
- i = index of the value in the sorted list (1-based)
- x_i = complexity values sorted ascending
- n = number of functions
- G = 0: perfectly equal (every function has same CC)
- G = 1: perfectly unequal (one god function, rest trivial)
```

**Data source**: Per-function cyclomatic complexity from structural analysis.
Falls back to file line counts if no function data available.

**Normalization**: Direct invert of Gini.
- score = 1 - G

**Why not Shannon entropy**: Entropy of file sizes gives confusing direction (high = good
contradicts thermodynamic intuition). Gini is more intuitive and better at detecting outliers.

---

### 5. Redundancy

**Purpose**: Measures fraction of code that is unnecessary — structural waste that increases
the search space for AI agents without contributing to behavior.

**Formula**:
```
R = (dead_count + duplicate_count) / total_functions
```

**Components**:
- **Dead functions**: Functions not referenced by any call site, excluding:
  - Test files (detected via language profile or path patterns)
  - Public/exported functions (they're API surface)
  - Methods (called via object dispatch, can't trace statically)
  - Implicit entry points (main, init, run, etc.)
- **Duplicate functions**: Groups of functions with identical body hashes

**Normalization**: Direct invert.
- score = 1 - R

**Why fundamental**: Every line of dead or duplicate code increases the search space
for the AI agent without contributing to behavior. Removing it always improves the codebase.

---

## Code Location Overview

### Core Calculation (`sentrux-core/src/metrics/root_causes.rs`)

All five raw signal values are computed and normalized in this file:

```rust
// Raw values
pub struct RootCauseRaw {
    pub modularity_q: f64,      // Newman's Q ∈ [-0.5, 1.0]
    pub cycle_count: usize,     // Number of circular dependency cycles
    pub max_depth: u32,          // Longest dependency chain
    pub complexity_gini: f64,   // Gini coefficient ∈ [0, 1]
    pub redundancy_ratio: f64,  // Redundancy ratio ∈ [0, 1]
}

// Normalized scores
pub struct RootCauseScores {
    pub modularity: f64,   // [0, 1]
    pub acyclicity: f64,   // [0, 1]
    pub depth: f64,        // [0, 1]
    pub equality: f64,    // [0, 1]
    pub redundancy: f64,  // [0, 1]
}

// Computation functions
pub fn compute_modularity_q(...) -> f64       // Line 72
pub fn compute_complexity_gini(...) -> f64    // Line 232
pub fn compute_redundancy_ratio(...) -> f64  // Line 293
pub fn compute_root_cause_scores(...) -> (RootCauseScores, f64)  // Line 310
```

### Supporting Code (`sentrux-core/src/metrics/mod.rs`)

| Function | Line | Purpose |
|----------|------|---------|
| `tarjan_sccs()` | 718-765 | Iterative Tarjan's SCC for cycle detection |
| `compute_max_depth()` | 864-875 | Longest-path DFS for depth calculation |
| `collect_duplicate_groups()` | 309-317 | Groups functions by body hash |
| `collect_dead_functions()` | 441-461 | Finds unreferenced functions |
| `build_call_target_set()` | 340-350 | Builds set of all called functions |
| `build_body_hash_map()` | 295-305 | Maps body hashes to function instances |
| `compute_health()` | 561-618 | Main orchestration function |

### Module Detection (`sentrux-core/src/core/path_utils.rs`)

- `module_of(path: &str) -> &str` — Determines module boundary
- Uses adaptive depth-2/depth-3 heuristics:
  - Depth-3: For paths with ≥3 directory levels, uses fine-grained sub-modules
  - Depth-2: For paths with exactly 2 directory levels
  - Handles dominant directory detection for root-level scans

### API Surface (`sentrux-core/src/app/mcp_server/handlers.rs`)

The `health` tool exposes quality signals via MCP:

```rust
// Line 97-160
pub fn health_def() -> ToolDef { ... }
fn handle_health(...) -> Result<Value, String> { ... }
```

Response format:
```json
{
  "quality_signal": 7342,
  "bottleneck": "modularity",
  "root_causes": {
    "modularity":  {"score": 6700, "raw": 0.45},
    "acyclicity":  {"score": 10000, "raw": 0},
    "depth":       {"score": 10000, "raw": 4},
    "equality":    {"score": 6100, "raw": 0.35},
    "redundancy":  {"score": 8800, "raw": 0.12}
  }
}
```

Scores are multiplied by 10000 and rounded to integers for display.

### CLI Surface (`sentrux-bin/src/main_impl.rs`)

- `print_check_results()` (line 463) — Displays quality signal × 10000
- `gate_save()` / `gate_compare()` — Uses quality signal for regression detection

---

## Data Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                         CLI / MCP API                               │
└─────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────┐
│                    analysis::scanner::scan_directory()               │
│                    → produces Snapshot                                │
└─────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────┐
│                         metrics::compute_health()                    │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │ compute_module_metrics() → mm                                  ││
│  │   ├── compute_max_depth() → max_depth                         ││
│  │   └── detect_cycles() → tarjan_sccs() → circular_dep_count    ││
│  └─────────────────────────────────────────────────────────────┘│
│  ┌─────────────────────────────────────────────────────────────┐│
│  │ compute_file_metrics() → fm                                   ││
│  │   ├── collect_duplicate_groups() → duplicate_groups           ││
│  │   └── collect_dead_functions() → dead_functions               ││
│  └─────────────────────────────────────────────────────────────┘│
│  ┌─────────────────────────────────────────────────────────────┐│
│  │ root_causes::compute_modularity_q() → modularity_q           ││
│  │ root_causes::compute_complexity_gini() → complexity_gini      ││
│  │ root_causes::compute_redundancy_ratio() → redundancy_ratio   ││
│  └─────────────────────────────────────────────────────────────┘│
│  ┌─────────────────────────────────────────────────────────────┐│
│  │ root_causes::compute_root_cause_scores()                       ││
│  │   ├── modularity = (Q + 0.5) / 1.5                            ││
│  │   ├── acyclicity = 1 / (1 + cycle_count)                      ││
│  │   ├── depth = 1 / (1 + max_depth / 8)                         ││
│  │   ├── equality = 1 - complexity_gini                         ││
│  │   └── redundancy = 1 - redundancy_ratio                       ││
│  │   └── quality_signal = geometric_mean(scores)                ││
│  └─────────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────┐
│                      HealthReport with:                              │
│  - quality_signal: f64 [0, 1]                                       │
│  - root_cause_scores: RootCauseScores                              │
│  - root_cause_raw: RootCauseRaw                                    │
└─────────────────────────────────────────────────────────────────┘
```

---

## Key Design Principles

1. **Root cause, not proxy**: Each signal measures a fundamental graph-theoretic property,
   not a symptom that can be gamed.

2. **Monotone**: Genuine improvement always increases the signal.

3. **Smooth**: Small changes produce small signal changes (no discontinuities).

4. **Ungameable**: Improving the signal requires improving the actual system structure.

5. **Geometric mean aggregation**: Forces improvement across ALL dimensions, not just one.

6. **Language-fair**: Works on the dependency graph structure, not language syntax.

---

## Theoretical Foundation

| Signal | Theory | Year | Contribution |
|--------|--------|------|--------------|
| Modularity | Newman's Modularity Q | 2004 | Edge clustering quality |
| Acyclicity | Tarjan's SCC Algorithm | 1972 | Cycle detection |
| Depth | Lakos Levelization | 1996 | Dependency chain length |
| Equality | Gini Coefficient | 1912 | Distribution inequality |
| Redundancy | Kolmogorov Complexity | 1963 | Structural waste approximation |
| Aggregation | Nash Social Welfare | 1950 | Geometric mean as optimal aggregation |
| Overall | Cybernetics (Wiener) | 1948 | Feedback loop architecture |

---

## File Reference

- **`sentrux-core/src/metrics/root_causes.rs`** — Core signal computation
- **`sentrux-core/src/metrics/mod.rs`** — Supporting calculations (cycles, depth, dead/duplicate detection)
- **`sentrux-core/src/core/path_utils.rs`** — Module boundary detection
- **`sentrux-core/src/metrics/types.rs`** — Data structures (`HealthReport`, `RootCauseScores`, etc.)
- **`sentrux-core/src/app/mcp_server/handlers.rs`** — MCP API surface
- **`sentrux-bin/src/main_impl.rs`** — CLI surface
- **`docs/quality-signal-design.md`** — Design documentation
