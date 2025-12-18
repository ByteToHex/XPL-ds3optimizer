# Optimization Algorithm Explanation

## Overview
This project implements an **exhaustive brute-force search algorithm** to find optimal armor combinations for Dark Souls 3. The algorithm evaluates all possible combinations of Head, Chest, Arms, and Legs armor pieces, filters them based on constraints, scores them using a weighted metric system, and maintains a sorted list of the top N best combinations.

---

## Core Algorithm: Exhaustive Search with Incremental Processing

### Main Implementation
**File:** `app/optimizer.ts`  
**Class:** `OptimizationEngine`  
**Method:** `ComputeOptimalsIncremental()` (lines 42-105)

The algorithm uses a nested loop structure to enumerate all possible armor combinations:

```typescript
ComputeOptimalsIncremental(curHeadIndex: number, Context: OptimizationEngine)
```

**Algorithm Steps:**

1. **Nested Loop Iteration:**
   - Outer loop iterates through Head pieces (`ih`)
   - Inner loops iterate through Chest (`ic`), Arms (`ia`), and Legs (`il`)
   - Each combination represents one complete armor set

2. **Early Filtering:**
   - Skip disabled armor pieces (lines 45-59)
   - Weight constraint check (line 61): Total weight must be ≤ `AvailableWeight`
   - If weight exceeds limit, skip to next combination

3. **Combination Creation:**
   - Creates `ArmorCombination` object via `ArmorCombinationFactory.Combine()` (line 64)
   - Calculates aggregate stats and scoring metric

4. **Constraint Validation:**
   - Validates minimum stat requirements (lines 66-84):
     - PhysicalAverage, Physical, Strike, Slash, Thrust
     - Magic, Fire, Lightning, Dark
     - Bleed, Poison, Frost, Curse
     - Poise

5. **Sorted List Insertion:**
   - If all constraints satisfied, attempts to add to sorted list via `List.TryToAdd()` (line 86)

6. **Incremental Processing:**
   - Uses `setTimeout` (line 96) to yield control after processing each head piece
   - Allows UI to update progress bar without blocking
   - Prevents browser from becoming unresponsive during long computations

**Entry Point:**
**File:** `app/optimizer.ts`  
**Method:** `ComputeOptimals()` (lines 23-33)
- Initializes progress tracking
- Calculates progress increment based on number of enabled head pieces
- Calls `ComputeOptimalsIncremental()` to start the search

---

## Data Structure: Doubly-Linked List for Top-K Maintenance

### Implementation
**File:** `app/doublylinkedlist.ts`  
**Class:** `DoublyLinkedList<T extends ISortable>`

The algorithm uses a custom doubly-linked list to maintain a sorted list of the top N best combinations efficiently.

**Key Properties:**
- `Head`: Pointer to the best (highest metric) combination
- `Tail`: Pointer to the worst (lowest metric) combination in the list
- `size`: Current number of elements
- `MaxSize`: Maximum number of elements to maintain (set via constructor)

**Interface:**
```typescript
export interface ISortable {
    Metric: number;
}
```

### Insertion Algorithm
**Method:** `TryToAdd(e: T)` (lines 12-126)

The insertion algorithm maintains the list in descending order by `Metric` value:

1. **Empty List Case** (lines 15-23):
   - If list is empty, insert element as both head and tail

2. **New Element Better Than Head** (lines 27-43):
   - If new element's metric > head's metric:
     - Insert at head position
     - Update pointers
     - If size exceeds `MaxSize`, drop the tail element

3. **Traversal to Find Insertion Point** (lines 46-123):
   - Start from `Head.next`
   - Traverse forward while `cur.element.Metric >= e.Metric` (line 63)
   - Three sub-cases:
     
     **a) Reached Tail** (lines 66-102):
     - If new element better than tail: insert before tail
     - If new element worse than tail but list not full: append to tail
     - If new element worse and list full: discard
     
     **b) Found Insertion Point** (lines 105-120):
     - Insert between two existing nodes
     - Update bidirectional pointers
     - If list exceeds `MaxSize`, drop tail

**Conversion to Array:**
**Method:** `ToArray(): T[]` (lines 128-136)
- Traverses the linked list from head to tail
- Returns array of elements in descending order (best first)

---

## Scoring System: Weighted Metric Calculation

### ArmorCombinationFactory
**File:** `app/armory.ts`  
**Class:** `ArmorCombinationFactory` (lines 570-604)

**Method:** `Combine(Head, Chest, Arms, Legs): ArmorCombination` (lines 575-602)

This factory class creates `ArmorCombination` objects and calculates their scoring metric.

**Metric Calculation Formula** (lines 579-595):
```typescript
Metric = 
    Weight_Physical × Physical +
    Weight_Strike × Strike +
    Weight_Slash × Slash +
    Weight_Thrust × Thrust +
    Weight_Magic × Magic +
    Weight_Fire × Fire +
    Weight_Lightning × Lightning +
    Weight_Dark × Dark +
    Weight_Bleed × Bleed +
    Weight_Poison × Poison +
    Weight_Frost × Frost +
    Weight_Curse × Curse +
    Weight_Poise × Poise
```

The weights are defined in `OptimizationParameters` and can be customized by the user.

### ArmorCombination Class
**File:** `app/armory.ts`  
**Class:** `ArmorCombination` (lines 511-568)

**Constructor** (lines 515-542) combines individual armor pieces into aggregate statistics:

1. **Weight Calculation** (line 516):
   ```typescript
   Weight = Head.Weight + Chest.Weight + Arms.Weight + Legs.Weight
   ```

2. **Damage Reduction Stats** (lines 518-528):
   - Uses multiplicative formula for damage reduction:
   ```typescript
   Physical = 1 - (1 - Head.Physical/100) × (1 - Chest.Physical/100) × 
              (1 - Arms.Physical/100) × (1 - Legs.Physical/100)
   ```
   - Applied to: Physical, Strike, Slash, Thrust, Magic, Fire, Lightning, Dark
   - PhysicalAverage = (Physical + Strike + Slash + Thrust) / 4 (line 523)

3. **Status Resistance Stats** (lines 530-533):
   - Uses additive formula:
   ```typescript
   Bleed = Head.Bleed + Chest.Bleed + Arms.Bleed + Legs.Bleed
   ```
   - Applied to: Bleed, Poison, Frost, Curse

4. **Poise Calculation** (lines 535-540):
   - Uses special formula via `PoiseFormula()` method:
   ```typescript
   PoiseFormula(p1, p2) = p1 + p2 - (p1 × p2 / 100)
   ```
   - Starts with `InnatePoise` value
   - Applies formula sequentially for each armor piece

---

## Constraint Filtering

### Weight Constraint
**File:** `app/optimizer.ts`  
**Location:** Line 61

```typescript
if(Head.Weight + Chest.Weight + Arms.Weight + Legs.Weight > AvailableWeight)
    continue;
```

**AvailableWeight Calculation:**
**File:** `app/armory.ts`  
**Method:** `UpdateTotalWeights()` (lines 114-135)

Calculates available weight based on:
- Character Vitality stat
- Equipped rings (Vitality modifiers and product modifiers)
- Weight fraction goal (e.g., 70% of max weight)
- Weight of equipped weapons
- Weight of equipped rings

### Minimum Stat Requirements
**File:** `app/optimizer.ts`  
**Location:** Lines 66-84

Validates that combination meets all minimum stat thresholds:
- `PhysicalAverage >= Minimums.PhysicalAverage`
- `Physical >= Minimums.Physical`
- `Strike >= Minimums.Strike`
- `Slash >= Minimums.Slash`
- `Thrust >= Minimums.Thrust`
- `Magic >= Minimums.Magic`
- `Fire >= Minimums.Fire`
- `Lightning >= Minimums.Lightning`
- `Dark >= Minimums.Dark`
- `Bleed >= Minimums.Bleed`
- `Poison >= Minimums.Poison`
- `Frost >= Minimums.Frost`
- `Curse >= Minimums.Curse`
- `Poise >= Minimums.Poise`

---

## Component Integration

### OptimizerComponent
**File:** `app/optimizer.component.ts`  
**Class:** `OptimizerComponent` (lines 16-138)

**Key Methods:**

1. **RunOptimization()** (line 57):
   ```typescript
   RunOptimization() {
       new OptimizationEngine(this as IOptimizerComponentVM, 
                             this._Armory as IOptimizertContext)
           .ComputeOptimals();
   }
   ```
   - Creates new `OptimizationEngine` instance
   - Starts the optimization process

2. **UpdateProgress(progress: number)** (line 61):
   - Called by `OptimizationEngine` to update UI progress bar
   - Updates `Progress` property bound to view

3. **ReceiveResults(result: ArmorCombination[])** (line 65):
   - Called when optimization completes
   - Receives sorted array of top combinations
   - Updates `OptimalArmorCombinations` property for display

**Interfaces:**
- `IOptimizerComponentVM` (lines 188-193): Defines contract for view model
- `IOptimizertContext` (lines 195-211): Defines contract for optimization context

---

## Algorithm Characteristics

### Type
**Exhaustive Search (Brute Force)**

The algorithm evaluates every possible combination of armor pieces, making it guaranteed to find the optimal solutions within the constraint space.

### Time Complexity
**O(H × C × A × L)**
- H = number of enabled Head pieces
- C = number of enabled Chest pieces  
- A = number of enabled Arms pieces
- L = number of enabled Legs pieces

For example, with 50 head pieces, 50 chest pieces, 50 arms pieces, and 50 legs pieces:
- Total combinations: 50 × 50 × 50 × 50 = 6,250,000 combinations

### Space Complexity
**O(N)** where N = `ResultListLength`
- Only maintains the top N combinations in memory
- Default is typically 10 combinations (line 92 in `armory.ts`)

### Optimizations

1. **Early Termination:**
   - Skips disabled armor pieces immediately
   - Skips combinations that exceed weight limit before creating objects

2. **Top-K Maintenance:**
   - Only keeps best N combinations in memory
   - Efficient insertion using sorted doubly-linked list
   - O(1) to O(N) insertion time depending on position

3. **Incremental Processing:**
   - Uses `setTimeout` to yield control to browser
   - Prevents UI blocking during long computations
   - Allows real-time progress updates

4. **Lazy Evaluation:**
   - Only creates `ArmorCombination` objects for valid combinations
   - Skips expensive calculations for invalid combinations

---

## Data Flow

```
User Input (Weights, Minimums, Vitality, etc.)
    ↓
OptimizerComponent.RunOptimization()
    ↓
OptimizationEngine.ComputeOptimals()
    ↓
OptimizationEngine.ComputeOptimalsIncremental()
    ↓
For each Head piece:
    For each Chest piece:
        For each Arms piece:
            For each Legs piece:
                ↓
            Weight Check → Skip if overweight
                ↓
            ArmorCombinationFactory.Combine()
                ↓
            Stat Calculation (multiplicative/additive formulas)
                ↓
            Metric Calculation (weighted sum)
                ↓
            Constraint Validation → Skip if doesn't meet minimums
                ↓
            DoublyLinkedList.TryToAdd()
                ↓
            Maintain sorted top-K list
    ↓
Update Progress (via setTimeout)
    ↓
Return Results (sorted array)
    ↓
OptimizerComponent.ReceiveResults()
    ↓
Display in UI
```

---

## Key Classes and Files Summary

| File | Class/Interface | Purpose |
|------|----------------|---------|
| `app/optimizer.ts` | `OptimizationEngine` | Main optimization algorithm |
| `app/optimizer.ts` | `IOptimizerComponentVM` | View model interface |
| `app/optimizer.ts` | `IOptimizertContext` | Optimization context interface |
| `app/optimizer.component.ts` | `OptimizerComponent` | Angular component controller |
| `app/doublylinkedlist.ts` | `DoublyLinkedList<T>` | Top-K sorted list data structure |
| `app/doublylinkedlist.ts` | `ISortable` | Interface for sortable objects |
| `app/armory.ts` | `ArmorCombination` | Represents a complete armor set |
| `app/armory.ts` | `ArmorCombinationFactory` | Creates and scores combinations |
| `app/armory.ts` | `ArmorPiece` | Individual armor piece data |
| `app/armory.ts` | `OptimizationParameters` | Weights and minimums configuration |
| `app/armory.ts` | `Armory` | Main data container and context |

---

## Algorithm Classification

This is a **Constraint Satisfaction Problem (CSP)** solved using:
- **Exhaustive Search** (complete search strategy)
- **Top-K Selection** (maintains best N solutions)
- **Weighted Objective Function** (multi-criteria optimization)

The algorithm is deterministic and guaranteed to find the optimal solutions within the defined constraint space, but has exponential time complexity relative to the number of armor pieces.

