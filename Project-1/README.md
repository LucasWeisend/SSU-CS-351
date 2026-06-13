# Memory Allocation Benchmark

A comparative performance analysis of four C++ memory allocation strategies — `alloca`, `malloc`, `new`, and `std::list` — measured across execution time, heap usage, and scalability.

*This project was overseen and graded by professor [Dave Shreiner](https://www.linkedin.com/in/daveshreiner/).*

---

## Overview

This project implements the same linked-list workload using four distinct allocation strategies, then benchmarks them to surface real differences in speed, heap pressure, and behavior under load. Each node stores a variable-length byte array (initialized with `std::iota`) and contributes to a running hash, ensuring the compiler can't optimize the work away.

| Program | Allocation Strategy | Data Storage |
|---|---|---|
| `alloca.out` | Stack (`alloca`) via recursion | Inline after node struct |
| `malloc.out` | Heap (`malloc`) + placement `new` | Inline after node struct |
| `new.out` | Heap (`operator new`) | `std::vector<Byte>` |
| `list.out` | `std::list<Node>` | `std::vector<Byte>` |

---

## Build & Run

**Requirements:** GCC or Clang, GNU Make, Linux (uses `strace` for heap analysis)

```bash
# Build all targets
make

# Run a quick correctness test
make test NUM_BLOCKS=10000

# Benchmark all programs (10 trials each)
make trials NUM_BLOCKS=10000 NUM_TRIALS=10

# Count heap expansions (brk syscalls) per program
make breaks NUM_BLOCKS=10000
```

**Configurable parameters** (pass as `make` variables):

| Variable | Default | Description |
|---|---|---|
| `NUM_BLOCKS` | `10000` | Number of nodes in each linked list |
| `NUM_TRIALS` | `10` | Benchmark repetitions per program |
| `MIN_BYTES` | `3` | Minimum bytes per node |
| `MAX_BYTES` | `100` | Maximum bytes per node |

---

## Results & Analysis

### Execution Speed

`alloca` is the fastest at small scales — stack allocation has near-zero overhead. At larger scales it triggers a stack overflow due to its recursive design, making it unsuitable for production use. Excluding `alloca`, **`malloc` is consistently the fastest**, outperforming both `new` and `list` at every scale tested.

`list` is the slowest, with `new` only marginally faster. Both allocate node metadata and data in separate heap operations — `new` via two allocations per node (the `Node` itself + the internal `std::vector` buffer), `list` via the same pattern managed by the STL container.

**Why `malloc` wins:** A single `malloc(sizeof(Node) + numBytes)` places the node header and its data in one contiguous allocation. This reduces allocator overhead, improves spatial locality, and cuts the number of heap expansions needed.

### Heap Pressure (`brk` syscalls)

```
NUM_BLOCKS=2,000,000  MIN_BYTES=60  MAX_BYTES=60
─────────────────────────────────────────────────
alloca.out    66     (exits early — stack overflow)
malloc.out   1,979
list.out     2,330
new.out      2,330
```

`malloc`'s contiguous allocation strategy requires significantly fewer heap expansions than `new` or `list` at high block counts, reinforcing why it's faster. `alloca` shows fewer `brk` calls only because it crashes before completing.

### Effect of Node Size

| Increasing... | Effect |
|---|---|
| `NUM_BLOCKS` | Execution time scales linearly — more allocations, more hashing |
| `MAX_BYTES` | Time increases, dominated by hashing and `iota` init (both O(n) in bytes) |
| `MAX_BYTES` (very large) | Allocation overhead becomes negligible relative to processing cost |

As node size grows, the fixed per-node allocation cost (one `malloc` call, one pointer) shrinks as a fraction of total runtime. Hashing and initialization dominate.

---

## Node Memory Layout

For `malloc.cpp` and `alloca.cpp`, each node is allocated as a single contiguous block: the `Node` struct header followed immediately by the raw byte data. The `bytes` pointer within the struct points into this same allocation rather than a separate heap buffer.

```
┌──────────────────────────────────────────┐
│  Node* next          (8 bytes)           │
│  Size  numBytes      (4 bytes)           │
│  Byte* bytes         (8 bytes) ─────┐    │
│  [padding]           (4 bytes)      │    │
├─────────────────────────────────────┤────┤
│  data[0] = 1  ◄─────────────────────┘    │
│  data[1] = 2                             │
│  ...                                     │
│  data[5] = 6   ← for a 6-byte node       │
└──────────────────────────────────────────┘
  Total: 24 bytes (struct) + 6 bytes (data) + padding
```

`head` points to the first node; `tail` tracks the last. Each node's `next` chains to the following node; the final node's `next` is `nullptr`.

---

## Key Takeaways

- **Contiguous allocation wins.** Combining the node header and its data into a single `malloc` call reduces allocator round-trips and improves cache locality — a meaningful real-world optimization pattern.
- **Stack allocation is fastest but fragile.** `alloca`-based recursion is extremely fast at small scales but is bounded by stack size, making it unsuitable for variable or large inputs.
- **STL containers add overhead.** `std::list` and `std::vector` provide safety and convenience but carry measurable cost from separate allocations and indirection.
- **Overhead amortizes with data size.** Per-node fixed costs (allocation, pointer setup) matter most at small node sizes; at large sizes, compute-bound work (hashing, init) dominates and allocation strategy matters less.
