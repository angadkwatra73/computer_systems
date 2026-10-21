# Product vs Subset Construction: What a DFA State Represents

Both constructions build a new DFA whose states are *snapshots* of other machines running in parallel. The difference is what the snapshot looks like.

## 1. Product construction (union of two DFAs)

- Input: two DFAs, M₁ and M₂.
- Each is deterministic, so at any moment each machine is in exactly **one** state.
- A state of the new DFA is a pair `(q, r)`: q is where M₁ is, r is where M₂ is.
- **Both machines are always active.** The only question is which state each one is in.
- The tuple length is always **2**, because Q = Q₁ × Q₂ (Cartesian product).
- Transition: advance both components together, `δ((q, r), a) = (δ₁(q, a), δ₂(r, a))`.
- Accepting: either component accepts, `F = (F₁ × Q₂) ∪ (Q₁ × F₂)`.
- Size: |Q₁| · |Q₂|.

## 2. Subset construction (NFA to DFA)

- Input: one NFA with states Q.
- Nondeterminism means the machine can split into several threads, so the number of live threads **changes** as input is read.
- A state of the new DFA is a **subset** of Q: exactly the NFA states that are alive after reading the input so far.
- The cardinality of the subset is the number of active NFA states, and it varies from step to step.
- Transition: advance every live thread on a and pool the results,

```
δ_DFA(S, a) = ⋃ over q in S of δ_NFA(q, a)
```

- Accepting: a subset is accepting if it contains an NFA accepting state ("any live thread accepts").
- Size: up to 2ⁿ, the power set. In practice only **reachable** subsets are built.

## 3. Side by side

| | Product (union) | Subset (NFA to DFA) |
|---|---|---|
| Starting point | Two DFAs | One NFA |
| State is | Pair (q, r) | Set S ⊆ Q |
| Shape | Fixed length 2 | Variable cardinality |
| What it records | The one state each machine is in | Every NFA state that is possible |
| Active threads | Always exactly 2 | Anywhere from 0 to n |
| Transition | Advance both components | Advance each member, then union |
| Accepting | Either component accepts | Any member accepts |
| Worst-case size | \|Q₁\| · \|Q₂\| | 2ⁿ |

## 4. The bit-vector view

A subset of an n-state NFA is an n-bit vector: bit i is 1 if state i is alive.

- The vector length is fixed at n.
- The number of 1s (the cardinality) is how many NFA threads are active.
- This is why the worst case is 2ⁿ: every bit pattern is a possible snapshot.
- The product is the same idea with a different shape: instead of n yes/no bits, it holds one *position* per machine.

## 5. How to read a DFA state name

- **Product DFA, state `(q, r)`:** M₁ is at q and M₂ is at r. The name tells you directly where each machine is.
- **Subset DFA, state `{P0, P1}`:** the NFA could be in P0 or in P1, and in no other state. The name lists all possible active states.
- In both cases the DFA is a deterministic summary of what is going on in the machines it simulates, so it never needs to guess.


## To formalize later

- ε-closure in the start state and in each transition step.
- Dead state ∅ and how it behaves.
