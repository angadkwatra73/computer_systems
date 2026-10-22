# Theory of Computation: Notes

Oct 10, 2026 · @Angad

## 1. Languages

A language is just a set of strings over a fixed alphabet.

- **Alphabet (Σ):** a finite set of symbols.
- **String:** a finite sequence of symbols from Σ.
- **Language:** a set of strings over Σ.

## 2. Finite Automata

A finite automaton M reads a string and either accepts or rejects it, so it defines a language.

- **Formal definition:** M = (Q, Σ, δ, q₀, F), meaning states, alphabet, transition function, start state, accepting states.
- **L(M):** the set of all strings M accepts. M *recognizes* L(M). A machine does not "contain" a language; it defines one.
- **Regular language:** a language that equals L(M) for some finite automaton M.

## 3. Regular Expressions

A regular expression R is a syntactic object, a string of symbols and operators, that describes a language. Its value is L(R).

- **Base cases:** ∅, ε, and a single symbol a ∈ Σ.
- **Regular operations** (the three that build languages): union A ∪ B, concatenation A ∘ B, star A\*.
- **Inductive construction:** if R₁ and R₂ are regular expressions, so are (R₁ ∪ R₂), (R₁R₂) and (R₁\*).
- **Each regex operator mirrors a language operation:**

```latex
L(R_1 \cup R_2) = L(R_1) \cup L(R_2)
\qquad
L(R_1 R_2) = L(R_1) \circ L(R_2)
\qquad
L(R^*) = \big(L(R)\big)^*
```
Note: We've mentioned regular ops, create regular expression, and combing reg ex with reg ex using ops also creates reg ex. 
While it's true, this is not to be confused with regular languages. we have not yet shown reg ex means a RL. 

Later, to show that, we will prove that reg exps can be realised as NFAs hence proving they are regular

Note: Below, we are proving the operations for languages and not regular expressions

## 4. Closure Under Union (via DFAs) [for languages]

**Theorem:** if langauges A₁ and A₂ are regular, so is A₁ ∪ A₂. The class of regular languages is closed under union.

**Proof idea (product construction):** run M₁ and M₂ in parallel on the same input, and accept if either one accepts. Let M₁ = (Q₁, Σ, δ₁, q₁, F₁) recognize A₁ and M₂ = (Q₂, Σ, δ₂, q₂, F₂) recognize A₂. Build M = (Q, Σ, δ, q₀, F) with:

```latex
Q = Q_1 \times Q_2 \qquad q_0 = (q_1, q_2)
\delta((q, r), a) = (\delta_1(q, a),\ \delta_2(r, a))
F = (F_1 \times Q_2) \cup (Q_1 \times F_2)
```

- A state of M is a pair that tracks where M₁ and M₂ each currently are.
- **F is "either side accepts".** The tempting F = F₁ × F₂ means "both accept", which gives the intersection, not the union.
- **Correctness:** by induction on |w|, after reading w, M is in state (δ₁\*(q₁, w), δ₂\*(q₂, w)). So M accepts w exactly when M₁ or M₂ does.
- **Cost:** |Q| = |Q₁| · |Q₂|.

## 5. Question Answered: What Does a Regex Take In?

**Q:** does a regular expression take in only symbols and return a language, or does it also take in languages?

**A:** a regex is built from symbols and other regexes, joined by the regular operations. It never takes a language as raw input. Its meaning, L(R), is a language, and each syntactic operator corresponds to an operation on languages. For example, (a ∪ b)\* is made of the symbols a and b plus operators, and its value is every string over {a, b}.

## 6. Non-Determinism (NFAs)

Non-determinism is a design convenience: it looks more powerful than a DFA, but it recognizes exactly the same languages.

- **Multiple paths:** on one symbol, an NFA can move to several states (or none).
- **ε-moves:** a transition that reads no input, so it is a free move. (The NFA does not read an empty string as a symbol; it moves without reading anything.)
- **Acceptance:** an NFA accepts w if *any* path through it ends in an accepting state. It rejects only if *every* path fails.
- **Formal change:** the transition function now returns a set of states.

$$ \delta : Q \times (\Sigma \cup \{\varepsilon\}) \to \mathcal{P}(Q) $$

**Three ways to picture it:**

1. **Parallel computation:** the machine is in many states at once, and each input symbol updates the whole set.
2. **A tree of paths:** each choice branches the computation, and w is accepted if some branch accepts.
3. **Guessing:** the machine guesses the right path and commits to it, as if it knew the answer in advance.

## 7. NFA and DFA Are Equivalent

**Theorem:** every NFA has an equivalent DFA, so NFAs and DFAs recognize exactly the regular languages.

- **DFA → NFA:** trivial, since every DFA is already an NFA.
- **NFA → DFA (subset construction):** each DFA state is a *set* of NFA states, namely the ones the NFA could be in after reading the input so far. Start from the ε-closure of the NFA's start state, and a DFA state is accepting if the set contains an NFA accepting state.
- **Cost:** an NFA with n states can need up to 2ⁿ DFA states. NFAs are easier to design but can be exponentially more compact.
- **Takeaway:** NFAs add convenience, not power. Any finite automaton gives a regular language, and an NFA is just an easier way to get there.

## 8. Up Next: Closure Proofs via NFAs

Union was easy with DFAs. Concatenation and star are much easier with NFAs.

- **Union (NFA way):** a new start state with ε-moves into both start states.
- **Concatenation:** ε-moves from the accepting states of the first NFA to the start of the second.
- **Star:** a new accepting start state, plus ε-moves from the old accepting states back to the old start.
- **Then:** use these to show regular expressions and finite automata describe exactly the same languages.

**Open questions:** none for now.

