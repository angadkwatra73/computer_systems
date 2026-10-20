# Theory of Computation: Intro

## What is it really?

> "Computer science is no more about computers than astronomy is about telescopes."

Usually attributed to Edsger Dijkstra, though the attribution is shaky and the analogy may predate him. The point stands: the field is about **computation** (the process), not the machines that happen to carry it out.

## The baseline: problems, functions, algorithms

- A **problem** says what we want done.
- A **function** describes *what* is to be done: inputs in, answer out.
- An **algorithm** describes *how*: a precise procedure that runs on any valid input and reaches a conclusion on its own.
- If we can write such an algorithm, the problem is solved.

## What theory of computation asks

Given a problem, can a computing device solve it at all, and if so, how much resource does it cost? Many problems cannot be solved by any algorithm. These are **non-computable** (undecidable).

That splits the subject into two questions:

| Question | Branch | What it studies |
|---|---|---|
| What can we compute *in principle*? | **Computability theory** | Which problems are decidable. We reason with the Turing machine, an idealised machine, to see what it can and cannot solve. |
| What can we compute *in practice*? | **Complexity theory** | How much time and space (resources) a solvable problem needs. |

## Study plan

Three stages, each a more powerful model than the last.

1. **Regular languages**
   - Model: finite automata.
   - Related: Moore and Mealy machines (finite automata with output), and lexical analysis (regular expressions are the tool for tokenizing).
   - Limitation: regular languages are not expressive enough. Many useful languages, such as balanced parentheses, fall outside them.
   - Notes: [[regular languages and finite automata.md]]

2. **Context-free languages**
   - Model: pushdown automata (a finite automaton plus a stack).
   - Described by context-free grammars.
   - Used in parsing.

3. **Turing machines**
   - The full model of computation: decidability and what is computable, then complexity (what is feasible).

Each stage gets its own notes file, linked from here as it is written.

## Sources

- Ben Brubaker, ["Does Computer Science Need Computers?"](https://quantamagazine.org/does-computer-science-need-computers-20260828), Quanta Magazine (on the quote's attribution).
- Computational Complexity blog, ["What is a Computer?"](https://blog.computationalcomplexity.org/2026/09/) (calls the quote apocryphal).
