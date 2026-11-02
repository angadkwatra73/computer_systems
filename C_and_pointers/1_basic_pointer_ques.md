# C Pointers: Revision Questions

## Questions

1. Can you assign an integer variable to a pointer?
2. Can you add two pointers? Which arithmetic and other operations are allowed on pointers?
3. Can a non-const pointer (`int *p` or `int *const p`) hold the address of a const object? What is the correct way?
4. In `int *p`, does the `*` bind to `int` or to `p`? Where do the brackets go?
   - **4b (Bonus):** Using the same idea, what is the difference between an array of int pointers and a pointer to an array of ints?
5. 
   - **5a:** What is the difference between `static int *p = malloc(...);` and the two-line version (`static int *p;` then `p = malloc(...);`)?
   - **5b (Hint):** Which variables are initialized before the program runs, which after, and what does that require of the initializer?

---

## Answers

**1.** No. It gives a compiler warning or error, and dereferencing the result is undefined behavior (usually a crash).

```c
int num = 5;
int *p = num;   // wrong
int *p = &num;  // correct
```

**2.** No, adding two pointers is not allowed.

- Arithmetic allowed:
  - `p + n`, `p - n`
  - `p++`, `++p`, `p--`, `--p` (pre and post both work)
  - `p - q` gives the element count (`ptrdiff_t`), only within the same array
- Other operations allowed:
  - Assignment
  - Comparison: `==` and `!=` generally, `<` and `>` only within the same array
  - Dereference `*p`

```c
p = p + 2;
p++;
n = q - p;   // same array only
```

**3.** It compiles with a warning, because the `const` on the pointee is discarded. Modifying the object through it is undefined behavior. The `const` in `int *const p` applies to the pointer, not the object, so it doesn't help. The correct way is a pointer to const.

```c
const int num = 10;

int *p = &num;              // warning; *p = 20 is UB
int *const q = &num;        // same problem
const int *const r = &num;  // correct
```

**4.** The `*` binds to `p`, not to `int`. The declarator `*p` is what is declared; `int` is the base type. So `int *p` reads as `int (*p)`: "`*p` is an `int`", i.e. `p` is a pointer to `int`. Spacing doesn't matter.

Multiple declarations show it clearly: the `*` applies only to the name it sits next to.

```c
int *a, b;    // a is int*, b is plain int 
int (*a), b;  // same thing, brackets made visible
```

**4b.** `[]` and `()` bind tighter than `*`, so brackets decide the meaning.

```c
int *p[3];    // array of 3 int pointers: *(p[3]) - each entry is first read as a pointer
int (*p)[3];  // pointer to an array of 3 ints
```

**5b (answered first).** Global and static variables (unless `extern`, which is only a declaration) are initialized before the program runs, so their initializer must be a constant expression. They are always initialized: with the given constant, or with zero if none is given (`.data` / `.bss`). "Uninitialized" only exists for auto (non-static local) variables, which are initialized at runtime when execution reaches the declaration.

**5a.** The one-line form is an error, because `malloc` is a runtime call and the initializer is not a constant. The two-line form is fine: `p` is zero-initialized to `NULL` before the program runs, then assigned at runtime.

```c
static int *p = malloc(4);   // error: initializer element is not constant

static int *p;               // zero-initialized (NULL)
p = malloc(4);               // runtime assignment, fine
```

Note: C++ differs. It allows dynamic initialization of statics, so the one-line form compiles there.
