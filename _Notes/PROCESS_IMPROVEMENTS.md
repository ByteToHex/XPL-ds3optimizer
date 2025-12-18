Reviewing the TypeScript files to understand the implementation and identify time complexity improvements.


[4 tools called]


Analyzing the code for time complexity improvements. Reviewing the implementation details:


[1 tool called]


Time complexity improvements:

## Time complexity improvement suggestions

### 1. Pre-filter enabled pieces (high impact)
Current: The algorithm iterates through all pieces and checks `Enabled` inside nested loops (lines 45-59 in `optimizer.ts`).

Improvement: Pre-filter enabled pieces into separate arrays before the nested loops.

Impact: Reduces iterations from O(Total × Total × Total × Total) to O(Enabled × Enabled × Enabled × Enabled). If 50% are disabled, this roughly halves the search space.

Implementation: Create filtered arrays at the start of `ComputeOptimals()`:
```typescript
let enabledHeads = this.Armory.Head.filter(h => h.Enabled);
let enabledChests = this.Armory.Chest.filter(c => c.Enabled);
// etc.
```

---

### 2. Early metric rejection (high impact)
Current: Creates the full `ArmorCombination` object (expensive) and calculates the metric for every valid combination, even if it won't make the top-N list.

Improvement: Before creating the combination, check if the list is full and estimate whether the combination could beat the worst element. If not, skip the expensive creation.

Impact: Avoids creating objects and computing metrics for combinations that won't be kept. With a full list of 10, this can skip many combinations.

Implementation: Maintain a `minMetric` threshold (worst element in list). Before creating combination, do a quick upper-bound estimate using max possible stats from remaining pieces.

---

### 3. Constraint checking order optimization (medium impact)
Current: Creates the full `ArmorCombination` object first, then checks all constraints (lines 66-84).

Improvement: Check constraints in order of computational cost and likelihood to fail. Some constraints can be checked with partial information.

Impact: Fails fast on invalid combinations without computing all stats.

Implementation:
- Check weight first (already done)
- For additive stats (Bleed, Poison, Frost, Curse), check max possible sum before creating object
- Check multiplicative stats only after object creation, but check easiest ones first

---

### 4. Pre-compute partial combinations (medium impact)
Current: For each combination, recalculates all multiplicative damage reduction formulas (8 stats × 4 pieces = 32 multiplications).

Improvement: Pre-compute partial combinations for Chest+Arms+Legs, then combine with Head pieces.

Impact: Reduces redundant calculations. Instead of H×C×A×L full calculations, do C×A×L pre-computations + H combinations.

Trade-off: Increases memory usage (C×A×L partial combinations) but reduces CPU time.

---

### 5. Optimize DoublyLinkedList insertion (low-medium impact)
Current: `TryToAdd()` has O(N) worst case where N = ResultListLength (typically 10).

Improvement: Use a min-heap (priority queue) for the worst element, or binary search for insertion point.

Impact: Reduces insertion from O(N) to O(log N) for small N (10), but the constant factor may matter more. For N=10, the improvement is modest.

---

### 6. Weight-based early termination (medium impact)
Current: Weight check happens after all loops are set up (line 61).

Improvement: Sort armor pieces by weight within each category, and use early termination in inner loops when weight already exceeds limit.

Impact: Can skip entire sub-trees of combinations. If Head + Chest already exceeds weight, skip all Arms/Legs combinations.

Implementation:
```typescript
// Sort by weight (ascending) before loops
let sortedHeads = enabledHeads.sort((a, b) => a.Weight - b.Weight);
// In loops, break early if partial weight exceeds limit
if (Head.Weight + Chest.Weight > AvailableWeight) break; // skip remaining chests
```

---

### 7. Parallel processing (high impact, architectural)
Current: Single-threaded exhaustive search.

Improvement: Use Web Workers to parallelize the search across multiple threads.

Impact: Near-linear speedup with number of cores (e.g., 4x on 4 cores).

Implementation: Split the Head pieces across workers, each worker searches its subset and returns top-N, then merge results.

---

### 8. Memoization for repeated calculations (low impact)
Current: Each combination recalculates multiplicative formulas from scratch.

Improvement: Cache intermediate results for common sub-combinations (e.g., Chest+Arms combinations).

Impact: Only beneficial if there's significant repetition, which is unlikely in exhaustive search. May not be worth the memory overhead.

---

### 9. Constraint pre-filtering (medium impact)
Current: All constraint checks happen after object creation.

Improvement: Pre-compute min/max bounds for each armor piece category. Before creating a combination, check if the theoretical maximum stats could satisfy constraints.

Impact: Can eliminate entire categories of combinations early.

Example: If max possible PhysicalAverage from all enabled pieces < Minimums.PhysicalAverage, skip all combinations.

---

### 10. Incremental metric calculation (low impact)
Current: Metric is calculated after all stats are computed.

Improvement: Calculate metric incrementally as stats are computed, allowing early termination if metric is clearly too low.

Impact: Minimal, since metric calculation is cheap compared to stat calculation.

---

## Priority ranking

1. Pre-filter enabled pieces — easy, high impact
2. Early metric rejection — medium effort, high impact
3. Weight-based early termination — easy, medium impact
4. Constraint pre-filtering — medium effort, medium impact
5. Parallel processing — high effort, high impact (architectural change)
6. Pre-compute partial combinations — medium effort, medium impact (memory trade-off)
7. Constraint checking order — easy, medium impact
8. Optimize DoublyLinkedList — low effort, low-medium impact
9. Incremental metric calculation — low effort, low impact
10. Memoization — low impact, may not help

## Estimated performance gains

- Pre-filtering: 2-4x speedup (if 50-75% pieces disabled)
- Early metric rejection: 2-5x speedup (depends on constraint strictness)
- Weight-based termination: 1.5-3x speedup (depends on weight distribution)
- Combined optimizations: potentially 10-20x overall speedup

The biggest wins are pre-filtering and early rejection, which are relatively straightforward to implement.