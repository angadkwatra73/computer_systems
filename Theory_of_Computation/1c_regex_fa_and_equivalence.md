# Closure Proofs, Regex ↔ Finite Automata, and Non-Regular Languages

## 8. Closure Proofs via NFAs

We prove closure **by construction**: given NFAs for the input languages, build an NFA for the result. Union was easy with DFAs (product construction). Concatenation and star are much easier with NFAs, because ε-moves let us glue machines together.

Let N₁ recognize A₁ and N₂ recognize A₂.

- **Union (A₁ ∪ A₂):** add a new start state with ε-moves into the start states of N₁ and N₂. Accepting states are the accepting states of both.
- **Concatenation (A₁ ∘ A₂):** add ε-moves from every accepting state of N₁ to the start state of N₂. The start state is N₁'s start. The accepting states are N₂'s only (N₁'s are no longer accepting).
- **Star (A₁\*):** add a new start state that is also accepting, with an ε-move to the old start state. Add ε-moves from the old accepting states back to the old start state.
  - The new start state must be accepting so that ε ∈ A₁\*.
  - It must be a *new* state. Making the old start accepting could let extra strings through.

Result: regular languages are closed under union, concatenation and star.

## 9. Regular Expressions Describe Regular Languages

Where we stand:
- A language is **regular** if some finite automaton recognizes it (our definition).
- We have shown regular languages are closed under ∪, ∘ and \*.

Still to show: a language described by a regular expression is regular, and the converse. So the goal is **regex ⇔ finite automata**, in two directions.

### 9A. Regex → NFA

Induction on the structure of the regex.

**Base cases.** Each has a tiny NFA, so each is regular:
- `∅`: one start state, no accepting states.
- `ε`: one state that is both start and accepting.
- `a` (a symbol): start state, one transition on a to an accepting state.

**Inductive step.** If R₁ and R₂ already have NFAs, then (R₁ ∪ R₂), (R₁R₂) and (R₁\*) have NFAs by the constructions in section 8.

**Example: `a ∪ (ab)*`**
1. Build NFAs for `a` and `b` (base cases).
2. Concatenate them to get `ab`.
3. Apply the star construction to get `(ab)*`.
4. Union that with the NFA for `a`.

Every regex is built from the base cases using the three operations, so every regex has an NFA.

### 9B. NFA → Regex

Use a **generalized NFA (GNFA)**: an NFA whose transitions are labelled with *regular expressions* instead of single symbols.

**Setup.** Make the GNFA have a start state with no incoming arrows, a single accepting state with no outgoing arrows, and an arrow (possibly labelled ∅) between every pair of other states. Add a new start with an ε-move to the old start, and a new accept state with ε-moves from the old accepting states.

**Reduction (rip out one state).** With k states, remove a state q_rip. For every remaining pair (qᵢ, qⱼ), relabel the arrow:

```
R_new(qᵢ → qⱼ) = R₁ R₂* R₃ ∪ R₄
```

- R₁: label qᵢ → q_rip
- R₂: label on the loop at q_rip
- R₃: label q_rip → qⱼ
- R₄: the old label qᵢ → qⱼ

This keeps the language the same: every path through q_rip is now one arrow.

**Induction on k.** While k > 2, reduce a k-state GNFA to an equivalent (k−1)-state GNFA. At k = 2 only the start and accept states remain, and the single arrow's label is the regex for the language.

### Summary

For regular languages:
- **Generator:** regular expressions (they produce the strings).
- **Recognizer:** finite automata (they accept the strings).
- They are **equivalent**: regex → NFA (9A), NFA → regex (9B), and DFA ⇔ NFA from earlier.

## 10. Non-Regular Languages

Not every language is regular. Finite automata have finite memory, so they cannot count unboundedly. Two examples:

- `{ 0ⁿ1ⁿ : n ≥ 0 }`
- `{ 0ⁿ1ⁿ2ⁿ : n ≥ 0 }`

### Pumping lemma

**Using it to show non-regularity:** assume A is regular, pick a string s that cannot be split in a way that satisfies all three conditions, and conclude a contradiction.

**Example, `0ⁿ1ⁿ`:** take s = 0ᵖ1ᵖ. Since |xy| ≤ p, y lies entirely in the 0s. Pumping up (i = 2) gives more 0s than 1s, so xy²z ∉ A. Contradiction, so the language is not regular.

**Caveat:** the lemma can only prove a language is **non-regular**. A language that satisfies the pumping conditions is not necessarily regular.

## Interesting Questions

- Which other operators preserve regularity? Test candidates (intersection, complement, difference, reversal), and for each decide whether to prove closure or find a counterexample.
- Is the exponential blowup in NFA → DFA ever avoidable, and can we prove a lower bound for a specific language?
- The pumping lemma has no converse. What stronger tools can prove a language is regular or non-regular?
