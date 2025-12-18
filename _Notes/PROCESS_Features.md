Reviewing key files to identify the optimization algorithm and its implementation.


[4 tools called]


Reading the armory data structures to understand how combinations are created:


[1 tool called]


## Optimization Algorithm Explanation

### Overview
The optimizer uses an exhaustive brute-force search to find the top N armor combinations that meet constraints and maximize a weighted score.

---

### Core Algorithm: Exhaustive Search with Incremental Processing

**File: `app/optimizer.ts`**

The main algorithm is in the `OptimizationEngine` class, specifically the `ComputeOptimalsIncremental()` method (lines 42-105).

**Algorithm Steps:**
1. Nested loops iterate over all armor pieces:
   - Outer loop: Head pieces (`ih`)
   - Inner loops: Chest (`ic`), Arms (`ia`), Legs (`il`)
2. For each combination:
   - Skip disabled pieces (lines 45-59)
   - Weight constraint check (line 61): total weight must be ≤ `AvailableWeight`
   - Create an `ArmorCombination` via `ArmorCombinationFactory.Combine()` (line 64)
   - Validate minimum stat requirements (lines 66-84)
   - If valid, attempt to add to the sorted list via `List.TryToAdd()` (line 86)
3. Incremental processing: uses `setTimeout` (line 96) to yield control after each head piece, allowing UI updates

**Key Method:**
```42:105:app/optimizer.ts
ComputeOptimalsIncremental(curHeadIndex: number, Context: OptimizationEngine) {
    // Nested loops iterate through all combinations
    // Weight filtering
    // Stat requirement validation
    // Sorted list insertion
}
```

---

### Data Structure: Doubly-Linked List for Top-K Maintenance

**File: `app/doublylinkedlist.ts`**

The `DoublyLinkedList<T>` class maintains a sorted list of the top N combinations.

**Key Features:**
- Maintains descending order by `Metric` (best first)
- Fixed maximum size (`MaxSize`)
- Efficient insertion via `TryToAdd()` (lines 12-126)

**Insertion Algorithm (`TryToAdd` method):**
1. Empty list: insert as head/tail (lines 15-23)
2. New element better than head: insert at head, drop tail if over limit (lines 27-43)
3. Otherwise: traverse to find insertion point (line 63)
   - If better than tail: insert before tail (lines 70-88)
   - If at capacity and worse than tail: discard (line 102)
   - Otherwise: insert in sorted position (lines 105-120)

**Interface:**
```140:142:app/doublylinkedlist.ts
export interface ISortable {
    Metric: number;
}
```

---

### Scoring System: Weighted Metric Calculation

**File: `app/armory.ts`**

**1. ArmorCombinationFactory (lines 570-604)**
- `Combine()` creates an `ArmorCombination` and computes its `Metric`
- Metric formula (lines 579-595):
  ```
  Metric = Σ(Weight_i × Stat_i)
  ```
  Where stats include: Physical, Strike, Slash, Thrust, Magic, Fire, Lightning, Dark, Bleed, Poison, Frost, Curse, Poise

**2. ArmorCombination Constructor (lines 511-568)**
- Combines individual pieces into aggregate stats:
  - Damage reduction: multiplicative formula (lines 518-528)
    ```
    Final = 1 - (1 - Head/100) × (1 - Chest/100) × (1 - Arms/100) × (1 - Legs/100)
    ```
  - Status resistances: additive (lines 530-533)
  - Poise: special formula (lines 537-540, 544-546)
    ```
    Poise = PoiseFormula(Poise, Piece.Poise)
    PoiseFormula(p1, p2) = p1 + p2 - (p1 × p2 / 100)
    ```

---

### Constraint Filtering

**File: `app/optimizer.ts` (lines 61-84)**

Two constraint checks:

1. Weight constraint (line 61):
   ```typescript
   if(Head.Weight + Chest.Weight + Arms.Weight + Legs.Weight > AvailableWeight)
       continue;
   ```

2. Minimum stat requirements (lines 66-84):
   - PhysicalAverage, Physical, Strike, Slash, Thrust
   - Magic, Fire, Lightning, Dark
   - Bleed, Poison, Frost, Curse
   - Poise

---

### Component Integration

**File: `app/optimizer.component.ts`**

- `OptimizerComponent` (lines 16-138) triggers optimization via `RunOptimization()` (line 57)
- Calls `OptimizationEngine.ComputeOptimals()` (line 58)
- Receives results via `ReceiveResults()` (line 65)
- Updates progress via `UpdateProgress()` (line 61)

---

### Algorithm Characteristics

**Type:** Exhaustive search (brute force)

**Time Complexity:** O(H × C × A × L)
- H = number of enabled Head pieces
- C = number of enabled Chest pieces
- A = number of enabled Arms pieces
- L = number of enabled Legs pieces

**Space Complexity:** O(N) where N = `ResultListLength` (top combinations kept)

**Optimizations:**
1. Early termination: skip disabled pieces and overweight combinations
2. Top-K maintenance: only keep the best N combinations
3. Incremental processing: yields control to prevent UI blocking

---

### Summary

The algorithm:
1. Enumerates all valid armor combinations (4 nested loops)
2. Filters by weight and minimum stat requirements
3. Scores each valid combination using weighted metrics
4. Maintains a sorted top-K list using a doubly-linked list
5. Processes incrementally to keep the UI responsive

This is a constraint satisfaction problem solved via exhaustive search with efficient top-K maintenance.