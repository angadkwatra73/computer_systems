# How Function Calls Work

Oct 12, 2026 · @Angad

## The setting

This doc describes function calls on a **register machine** (x86-64 here) that keeps its call frames on a **stack**. Two things are true at once:

- **In hardware (the ISA):** `push`, `pop`, `call` and `ret` are real instructions, and `rsp` (stack pointer) and `rip` (program counter) are real registers. Intel and AMD implement these directly. ARM works differently: `bl` puts the return address in a link register instead of pushing it.
- **By convention (compiler + ABI):** the stack is just ordinary memory plus an agreement to treat `rsp` as its top. Frame layout, the frame base pointer `rbp`, and which registers a function must preserve are all choices made by the compiler and calling convention, not by the CPU.

Other designs exist (heap-allocated frames, link registers, static locals), but a stack is the default because allocating a frame is one instruction and frames die in last-in, first-out order.

## The mental model: code and data are separate

**The code segment decides what runs next. The stack holds each call's private data.** Instructions are never copied onto the stack, and a function call never copies a function's code.

- **Code:** one fixed copy of each function in the code segment. The program counter walks through it sequentially, and a call or return only changes the address the PC holds.
- **Data:** each call gets its own frame on the stack, holding its arguments, locals, the saved caller context, and the return address.
- **Shared data:** `static` and global variables (like `count` in the recursion example) live in the data segment, not in any frame, so every call sees the same copy.

## The cast

| Name | x86-64 | What it holds | Lives in |
| --- | --- | --- | --- |
| Program counter (PC) | `rip` | Address of the next instruction to fetch | Points into the code segment, always |
| Stack pointer (SP) | `rsp` | Address of the top of the stack; moves down on push, up on pop | Points into the stack |
| Base pointer (BP) | `rbp` | Fixed anchor for the current frame, so locals sit at constant offsets like `[rbp-4]` | Points into the stack |
| Argument register | `edi` | First integer argument | Register |
| Return register | `eax` | The return value | Register |
| Code segment |  | Instructions only | Memory |
| Stack |  | Return addresses, saved `rbp`, saved registers, locals; data only | Memory |

The stack grows toward lower addresses, so "top" means the lowest address in use.

## What a call does

A call has two parts: the single `call` instruction (hardware semantics) and the callee's opening instructions (compiler convention).

| Step | Who | Code side (PC) | Data side (stack) |
| --- | --- | --- | --- |
| 1. `mov edi, 3` | Caller | Next instruction | Argument goes in a register, not on the stack |
| 2. `call f` | One CPU instruction | PC = address of `f` in the code segment | Pushes the address of the instruction after the call (the resume point) |
| 3. `push rbp` | Callee prologue | Next instruction | Saves the caller's frame base |
| 4. `mov rbp, rsp` | Callee prologue | Next instruction | New frame base; `rbp` and `rsp` both point at the saved-`rbp` slot |
| 5. `sub rsp, N` | Callee prologue | Next instruction | Reserves room for locals; `rsp` moves below `rbp` |

`call` pushes the address of the **next** instruction, not the current PC, because the PC has already moved past the call by the time it executes. The body of `f` then runs by the PC walking through `f`'s code.

## What a return does, and what must be preserved

The function puts its result in `eax`, then undoes the call in reverse:

1. **`leave`:** `mov rsp, rbp` drops the locals, then `pop rbp` restores the caller's frame base.
2. **`ret`:** pops the return address into the PC, so fetching resumes right after the caller's `call`.

Afterward, `rsp` and `rbp` hold exactly what they held before the call, and the PC is back in the caller's code. Nothing is erased: the old frame's bytes stay in memory below `rsp` until the next call overwrites them.

**Registers are shared, so they need rules.** There is one physical register set, so the calling convention splits the work:

- **Caller-saved** (`rax`, `rcx`, `rdx`, `rsi`, `rdi`, `r8`-`r11`): the callee may overwrite them. A caller that needs one across a call saves it in its own frame first.
- **Callee-saved** (`rbx`, `rbp`, `r12`-`r15`): the callee must restore them before returning. If it uses one, it pushes the old value in its prologue and pops it before `ret`.

The compiler saves only what it actually uses, which is why the unoptimized `rf` saved only `rbp`.

## Recursion: same code, a new frame each time

A recursive call is an ordinary call whose target happens to be the function already running. Every active call runs the **same instructions at the same addresses**. What differs is only the frame: each has its own `n`, its own saved `rbp`, and its own return address.

Example: `rf(n)` does `count++` (a static), calls `rf(n-1)`, and returns what that returns; `rf(0)` returns `count`. Starting from `main` calling `rf(2)`:

| Event | Code side (PC) | Data side (stack) |
| --- | --- | --- |
| `main` calls `rf(2)` | Jumps to start of `rf` | Frame A (`n=2`), return address back into `main` |
| `rf(2)`: `count` becomes 1, calls `rf(1)` | Jumps to start of `rf` again; A is paused at its call | Frame B (`n=1`) pushed on top of A |
| `rf(1)`: `count` becomes 2, calls `rf(0)` | Same jump; B is paused at its call | Frame C (`n=0`) on top of B |
| `rf(0)`: base case, `eax = count = 2` | Takes the base-case path, no call | `leave; ret` pops C; PC returns into `rf(1)` |
| `rf(1)` resumes | Runs `leave; ret` | Pops B; PC returns into `rf(2)` |
| `rf(2)` resumes | Runs the same `leave; ret` | Pops A; PC returns into `main`, which stores `eax` (2) as the result |

Two facts explain the whole pattern:

- **No returns until the bottom.** Each frame's PC is stuck at its own `call`. The first `ret` happens only in `rf(0)`, and each `ret` unblocks exactly the frame beneath it. The stack's last-in, first-out order forces returns to unwind one function at a time.
- **The same `ret` runs repeatedly with different context.** Every frame's saved return address is the same code address, so the identical `leave; ret` executes at each level, but with a different `rbp` and `rsp` each time. Only the bottom frame returns to `main`.

The value passes through unchanged because nothing touches `eax` on the way back up. At higher optimization levels, a compiler can turn `return rf(n-1)` into a plain jump (a tail call), and the frames disappear.

## In one paragraph

The code segment supplies the single sequence of instructions, and the PC walks through it. The stack supplies private data for each active call. `call` saves where to resume, the callee's prologue saves which frame to return to, and `leave; ret` undo both, restoring `rsp`, `rbp` and the PC. Recursion adds nothing new: it reuses the same code with a fresh frame each time, and unwinds in last-in, first-out order.
